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
- **Size limits**: `$SLURM_TMPDIR` is limited by available local disk space.

### Container filesystem isolation

When Slurm runs jobs inside containers, the filesystem behavior changes from standard Slurm jobs.

- **File system hiding**: Containers create a sealed environment and may hide the host's system directories. Slurm jobs running in containers cannot access host system paths that are not explicitly mounted.
- **Bind mounts in Slurm**: Container bind mounts are managed by the container runtime (configured via `ContainerType` in `slurm.conf`). To access host files from within a container, they must be explicitly mapped. Administrators configure default mounts via container runtime settings.
- **Shared filesystem access**: Container runtimes typically mount shared storage (like home directories) into containers automatically so users retain access to their data. The shared filesystem paths remain consistent between container and host, allowing Slurm jobs in containers to access the same shared paths as non-containerized jobs.
- **`$SLURM_TMPDIR` in containers**: The `$SLURM_TMPDIR` variable is set by Slurm and points to temporary storage. In containers, this typically points to host local storage that is mounted into the container, maintaining the same isolation and cleanup behavior as non-containerized jobs.


### File broadcasting

Slurm provides file broadcasting to distribute files efficiently using its internal network topology.

- **Mechanism**: File broadcasting uses Slurm's message tree to push files from the login node to all allocated compute nodes, avoiding filesystem contention when many nodes need the same file.
- **Usage**: File broadcasting works only within an active allocation. Use it in job scripts after resources are allocated, not during submission. This is particularly useful for distributing executables or input files to multiple nodes in a multi-node job.

## dstack

TBA