# 9 Filesystems and data access

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
  
  # Process data from local scratch
  srun python preprocess_data.py $SLURM_TMPDIR/training_dataset.h5
  
  # Copy results back to shared filesystem
  cp $SLURM_TMPDIR/preprocessed.h5 /shared/results/
  
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
  ls /home/user/data
  
  # Local scratch accessible via $SLURM_TMPDIR (host local storage mounted into container)
  cp /home/user/data/input.h5 $SLURM_TMPDIR/
  srun python train.py $SLURM_TMPDIR/input.h5
  
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
  
  # Now run on all nodes (files available at /tmp/)
  srun /tmp/my_app /tmp/input.txt
  ```

## dstack

TBA