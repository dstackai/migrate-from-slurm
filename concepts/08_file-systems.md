# 8 Filesystems and data access

## Slurm

### Host file system visibility

Standard Slurm jobs run as native processes on the compute node. They see the host's operating system directly, unlike virtualized environments.

- **Full visibility**: Slurm jobs can read system paths. Visibility is restricted only by standard Linux user permissions. Jobs can read system files but usually can only **write** to user-specific directories.
- **No local sync**: The local hard drives of compute nodes are physically separate. A file created on Node A does **not** exist on Node B. Slurm does not automatically synchronize local storage across nodes.

### The shared filesystem (Global Namespace)

Slurm assumes a **Global Namespace** provided by networked storage (NFS, Lustre, GPFS). This is essential for distributed jobs.

- **Slurm's assumption**: Slurm expects paths to be identical across the cluster. If a script is located at a specific path, it must be accessible at that exact path on all nodes. Slurm does not provide filesystem abstraction or path translation.
- **Consistency**: This is the primary communication channel for distributed Slurm jobs. Slurm relies on the filesystem's consistency guarantees. The consistency model (immediate, eventual, or eventual with strong consistency) depends on the underlying filesystem, not Slurm.

### Local storage (`$SLURM_TMPDIR`)

While Slurm sees local disks as just another path, it manages them differently to ensure job isolation.

- **The Variable**: Slurm sets the environment variable `$SLURM_TMPDIR` to point to the temporary storage on the specific compute node running the job.
- **Isolation**: On large clusters, Slurm is often configured (via plugins) to create a private directory for each job. This ensures that Job A cannot read files from Job B, even if they are on the same node.
- **Automatic Cleanup**: Unlike the shared filesystem, Slurm automatically deletes the contents of `$SLURM_TMPDIR` when the job ends (via `slurmstepd` cleanup). This prevents nodes from filling up with old data.
- **Example: using $SLURM_TMPDIR** (script on login node, executes on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --nodes=1
  #SBATCH --time=1:00:00
  
  # Copy input data to local scratch (faster I/O)
  cp /shared/data/training_dataset.h5 $SLURM_TMPDIR/
  
  # List files in local scratch
  srun ls $SLURM_TMPDIR/
  
  # Copy results back to shared filesystem (if any were created)
  # cp $SLURM_TMPDIR/results.h5 /shared/results/
  
  # $SLURM_TMPDIR automatically cleaned up when job ends
  ```
- **Size limits**: `$SLURM_TMPDIR` is limited by available local disk space.

### Container filesystem isolation

When Slurm runs jobs inside containers, the filesystem behavior changes from standard Slurm jobs.

- **File system hiding**: Containers create a sealed environment and hide the host's system directories. Slurm jobs running in containers cannot access host system paths (e.g., `/usr`, `/etc`, `/opt`) that are not explicitly mounted. This isolation prevents jobs from seeing or modifying host system files.
- **Disk access restrictions**: Unlike native Slurm jobs that can read any system path (subject to permissions), containerized jobs only see:
  - Paths explicitly mounted via bind mounts (configured by administrators)
  - The container's internal filesystem (from the container image)
  - Shared filesystems that are mounted into the container (typically home directories and shared storage)
- **Bind mounts**: Container bind mounts are managed by the container runtime (configured via `ContainerType` in `slurm.conf` on the controller node). Administrators configure which host directories are automatically mounted into containers. Common defaults include home directories (`/home/$USER`) and shared storage paths (`/shared`, `/scratch`).
- **Shared filesystem access**: Container runtimes typically mount shared storage (like home directories and project directories) into containers automatically. The shared filesystem paths remain consistent between container and host (e.g., `/home/user/data` is the same path inside and outside the container), allowing Slurm jobs in containers to access the same shared paths as non-containerized jobs.
- **`$SLURM_TMPDIR` in containers**: The `$SLURM_TMPDIR` variable is set by Slurm and points to temporary storage. In containers, this typically points to host local storage (e.g., `/tmp` or node-local scratch) that is mounted into the container. The same isolation and automatic cleanup behavior applies - files in `$SLURM_TMPDIR` are deleted when the job ends.
- **Example: container filesystem access** (script on login node, executes in container on compute node):
  ```bash
  #!/bin/bash
  #SBATCH --nodes=1
  #SBATCH --gres=gpu:1
  #SBATCH --container=/shared/images/pytorch.sif
  
  # Shared filesystem accessible (bind-mounted by container runtime from host)
  # Container runtime automatically mounts home directories and shared storage
  # Path /home/user/data on host is accessible at same path inside container
  srun ls /home/user/data
  
  # Local scratch accessible via $SLURM_TMPDIR (host local storage mounted into container)
  cp /home/user/data/input.h5 $SLURM_TMPDIR/
  srun ls $SLURM_TMPDIR/
  
  # Host system paths NOT accessible (not bind-mounted into container)
  # ls /usr/local  # Fails - host /usr/local not mounted into container
  ```


### File broadcasting

Slurm provides file broadcasting to distribute files efficiently using its internal network topology.

- **Mechanism**: File broadcasting uses Slurm's message tree to push files from the login node to all allocated compute nodes, avoiding filesystem contention when many nodes need the same file.
- **Usage**: File broadcasting works only within an active allocation. Use it in job scripts after resources are allocated, not during submission. This is particularly useful for distributing executables or input files to multiple nodes in a multi-node job.
- **Example: file broadcasting** (script on login node, files broadcast to compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --nodes=4
  #SBATCH --ntasks=32
  
  # Broadcast executable to all allocated nodes
  srun --ntasks=1 --nodes=1 sbcast /shared/bin/my_app /tmp/my_app
  
  # Broadcast input data
  srun --ntasks=1 --nodes=1 sbcast /shared/data/input.txt /tmp/input.txt
  
  # Verify files are available on all nodes
  srun ls /tmp/
  ```

## dstack

### Container filesystem isolation

Unlike Slurm, which can run jobs as native processes with access to system paths (subject to permissions), dstack exclusively runs workloads inside Docker containers. dstack jobs only see:
- Paths explicitly mounted via volumes (network or instance volumes)
- The container's internal filesystem (from the container image)
- Files and repos uploaded by dstack (temporary, not persisted across runs)

- **User permissions**: Containers always run as root by default, but user processes within containers may run as different users depending on the container image configuration or explicit user settings. This differs from Slurm where jobs run as the submitting user.

<!-- TODO: Elaborate on user permissions - how dstack handles user mapping, how this affects volume permissions, and how it compares to Slurm's user-based execution model. -->

### Instance volumes

Instance volumes allow mapping any directory on the host instance to any path inside the container. This provides a way to access host storage or manually-mounted network storage.

- **Mechanism**: Instance volumes bind mount directories from the host instance into the container. The data persists only if the run executes on the same instance.
- **Shared filesystem support**: Users can use shared filesystems (e.g., NFS, SMB) with instance volumes, but are responsible for mounting them on the host. This is straightforward with SSH fleets where users control the instances. For backend fleets, mounting shared storage may require a dedicated endpoint (e.g., support for `init` scripts with fleets), which is not currently supported.
- **Backend support**: Instance volumes are supported for all backends except `runpod`, `vastai`, and `kubernetes`. They can also be used with SSH fleets.
- **Use cases**: Instance volumes are suitable for caching (e.g., pip cache, model downloads) or persistent storage when using SSH fleets with pre-mounted network storage.
- **Example: instance volume for caching**:
  ```yaml
  type: task
  
  commands:
    - ls /root/.cache/pip
  
  volumes:
    - /dstack-cache/pip:/root/.cache/pip
  ```
- **Example: instance volume with NFS** (SSH fleet with NFS pre-mounted on host, using short syntax `instance_path:container_path`):
  ```yaml
  type: task
  
  commands:
    - ls /storage
  
  volumes:
    - /mnt/nfs-storage:/storage
  ```

### Network volumes

Network volumes are cloud-managed persistent storage volumes provisioned by dstack backends and mounted to container directories. They provide persistent storage that survives across runs.

- **Backend support**: Network volumes are currently supported for the `aws`, `gcp`, and `runpod` backends.
- **Region and backend binding**: Network volumes are bound to a specific backend and region. A volume created for AWS in `eu-central-1` cannot be used with GCP or in a different AWS region.
- **Multi-attach support**: The ability to attach a single network volume to multiple runs or instances simultaneously is currently supported only by the `runpod` backend. Other backends (AWS EBS, GCP persistent disks) support single-attach volumes only.
- **Volume creation**: Volumes are created using a separate configuration file and the `dstack apply` command:
  ```yaml
  type: volume
  name: my-volume
  backend: aws
  region: eu-central-1
  size: 100GB
  ```
- **Attaching volumes**: Volumes are attached to runs via the `volumes` property:
  ```yaml
  type: task
  
  commands:
    - ls /volume_data
  
  volumes:
    - name: my-volume
      path: /volume_data
  ```
- **Short syntax**: Network volumes can use the short syntax `volume-name:container-path`:
  ```yaml
  type: task
  
  commands:
    - ls /volume_data
  
  volumes:
    - my-volume:/volume_data
  ```
- **Multiple volumes for different regions/backends**: If you're unsure which region or backend will be used, you can specify multiple volumes for the same mount point. dstack will select one based on the run's region and backend:
  ```yaml
  type: task
  
  commands:
    - ls /volume_data
  
  volumes:
    - name: [my-aws-eu-west-1-volume, my-aws-us-east-1-volume]
      path: /volume_data
  ```
- **Volume interpolation for distributed tasks**: When using single-attach volumes with distributed tasks, you can attach different volumes to different nodes using dstack variable interpolation:
  ```yaml
  type: task
  nodes: 8
  
  commands:
    - ls /volume_data
  
  volumes:
    - name: data-volume-${{ dstack.node_rank }}
      path: /volume_data
  ```
  This ensures each node uses its own volume. The `dstack.node_rank` variable (or `dstack.job_num`) is interpolated per job/node.
- **Automatic cleanup**: Automatic cleanup of volumes is supported via the `auto_cleanup_duration` configuration option in the volume configuration. This feature only works for volumes created by dstack (not for external volumes registered with `volume_id`). Volumes are automatically deleted after the specified idle duration when they're no longer attached to any runs. If not configured, volumes must be manually deleted when no longer needed.
- **Volume access**: Volumes are only accessible from runs. They are automatically attached when a run starts and detached when the run stops. Volumes cannot be accessed outside of active runs.

### Files and repos

Beyond volumes, dstack provides two mechanisms for bringing code and data into containers: files and repos. Both are temporary and not persisted across runs unless mounted into a volume directory.

- **Files**: The `files` property allows mounting local files or directories into the container. Each entry maps a local path to a container path (supports `~` for home directory):
  ```yaml
  type: task
  
  commands:
    - ls ~/workspace
  
  files:
    - ../examples:~/workspace/examples
    - ~/.ssh/id_rsa:~/ssh/id_rsa
  ```
  - **Size limit**: Each file or directory entry is limited to 2MB (configurable via `DSTACK_SERVER_CODE_UPLOAD_LIMIT`).
  - **Persistence**: Files are uploaded to the instance and mounted into the container, but are not persisted across runs. To persist files, mount them into a volume directory.
  - **Short syntax**: Files can use the short syntax `local_path[:container_path]` (supports `~` for home directory):
    ```yaml
    type: task
    
    commands:
      - ls
    
    files:
      - ../examples  # Auto-calculated container path
      - ~/.ssh/id_rsa:~/ssh/id_rsa
    ```

- **Repos**: The `repos` property allows cloning Git repositories into the container:
  ```yaml
  type: task
  
  commands:
    - ls
  
  repos:
    - ..  # Clone parent directory repo to working directory
  ```
  - **Local repos**: Can specify a local Git repository path (relative or absolute). dstack clones the repo on the instance, applies local changes, and mounts it into the container.
  - **Remote repos**: Can specify a Git URL to clone directly:
    ```yaml
    type: task
    
    commands:
      - ls
    
    repos:
      - https://github.com/dstackai/dstack
    ```
  - **Custom directory**: Can specify where to clone the repo (supports `~` for home directory):
    ```yaml
    type: task
    
    commands:
      - ls ~/my-repo
    
    repos:
      - ..:~/my-repo
    ```
  - **Size limit**: Repo size is not limited, but local changes are limited to 2MB (configurable).
  - **Persistence**: Repos are cloned on the instance and mounted into the container, but are not persisted across runs. To persist repos, mount them into a volume directory.
  - **Limit**: Currently, a maximum of one repo is supported per run configuration.

### SSH file transfer

As an alternative to the `files` property, you can transfer files directly via SSH using the run name alias while the CLI is attached to the run. To attach the CLI to a run, use `dstack attach <run name>`. Note that `dstack apply` attaches to the run by default unless you use the `--detach` flag. This method is useful for larger files that exceed the 2MB limit, but transfer speed depends on the network bandwidth between the CLI and the instance.

Example: Transfer a file to a running task using the run name alias:

```bash
# While CLI is attached to the run
scp large-dataset.h5 <run name>:/path/inside/container/
```

You can also use `rsync` for more efficient transfer of directories:

```bash
rsync -avz ./data/ <run name>:/path/inside/container/data/
```