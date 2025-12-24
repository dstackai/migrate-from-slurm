# 5 Resource enforcement, containment, and limits

## Slurm

### Resource isolation (Cgroups)

Slurm uses Linux Control Groups (**cgroups**) to strictly confine jobs. This prevents one user's job from crashing a node or stealing resources from another.

- **Mechanism**: Modern clusters use **Cgroup v2** (unified hierarchy).
- **Memory specification**: Memory can be requested in two ways:
  - **`--mem=M`**: Requests M memory **per node**. If `--nodes=4 --mem=16G`, each of the 4 nodes gets 16GB, totaling 64GB across all nodes.
  - **`--mem-per-cpu=M`**: Requests M memory **per CPU**. If `--ntasks=8 --cpus-per-task=2 --mem-per-cpu=2G`, total memory = 8 tasks × 2 CPUs × 2GB = 32GB per node.
- **Memory enforcement**: Memory limits are enforced at the cgroup level on each compute node. The kernel memory controller creates a hard limit (RAM + Swap). If a job exceeds its requested memory on any node, the kernel OOM Killer immediately terminates the process inside that cgroup, preventing it from crashing the physical node and protecting other jobs.
- **Example: per-node memory** (job script on login node, memory enforced on compute node):
  ```bash
  #!/bin/bash
  #SBATCH --nodes=2
  #SBATCH --ntasks=8
  #SBATCH --gres=gpu:4
  #SBATCH --mem=16G  # 16GB per node (32GB total across 2 nodes)
  
  # If job exceeds 16GB on any node, OOM killer terminates it
  srun python distributed_training.py
  ```
- **Example: per-CPU memory** (job script on login node, memory enforced on compute node):
  ```bash
  #!/bin/bash
  #SBATCH --nodes=1
  #SBATCH --ntasks=4
  #SBATCH --cpus-per-task=2
  #SBATCH --mem-per-cpu=4G  # 4GB per CPU = 4 tasks × 2 CPUs × 4GB = 32GB total
  
  srun python simulation.py
  ```
- **Device isolation**: The device controller creates an allow-list. If a user requests `--gres=gpu:1`, the cgroup ensures the process can literally only see `/dev/nvidia0` and is denied access to other GPUs.

### CPU affinity & pinning

Beyond simple enforcement, Slurm manages **CPU Affinity** to maximize performance.

- **Task affinity**: Slurm pins (binds) tasks to specific CPU cores. Tasks can be bound to specific cores or sockets (allowing movement between cores on that socket).
- **Numa awareness**: Slurm prioritizes allocating memory from the RAM bank closest to the pinned CPU core to minimize latency.

### Process containment & tracking

Slurm ensures that when a job ends, *everything* ends.

- **Proctrack plugin**: Slurm uses `ProctrackType=proctrack/cgroup`. This tracks processes based on their cgroup membership rather than Parent Process ID (PPID).
- **Benefit**: Even if a user's job "daemonizes" (double-forks) to detach from its parent, it remains trapped inside the cgroup.
- **Cleanup**: When a job is cancelled or finishes, `slurmstepd` simply destroys the cgroup. The kernel then guarantees that every process inside that container is killed instantly. This eliminates "zombie processes" on compute nodes.

### Walltime enforcement

Time limits are strictly enforced to keep the queue moving.

- **Tracking**: `slurmstepd` monitors the elapsed time of the allocation.
- **Grace period**: When the limit is reached, Slurm sends `SIGTERM` first. The application has a configured grace period (default 30-60s) to save checkpoints and exit cleanly.
- **Hard kill**: If the process is still running after the grace period, Slurm sends `SIGKILL` (immediate termination).
- **Pre-warnings**: Users can request a warning signal before the walltime hits, allowing for proactive checkpointing.

### Resource limits and constraints

Beyond basic enforcement, Slurm provides additional resource limits and constraints.

- **CPU affinity details**: The `SLURM_CPU_BIND` environment variable shows the binding mask.
- **CPU time limits**: In addition to walltime, Slurm can enforce CPU time limits via cgroups. The `cpu` controller limits CPU usage to a percentage of available CPUs, preventing jobs from consuming more CPU time than allocated. This is configured via the `CgroupPlugin` parameter in `slurm.conf`.
- **File descriptor limits**: Cgroups can enforce file descriptor limits per job via the `CgroupPlugin` parameter in `slurm.conf`. This prevents jobs from exhausting system resources.

### Container integration

Slurm supports running jobs inside containers for isolation and reproducibility. Container runtime is configured on compute nodes via `slurm.conf`, while users specify container images in job scripts from login nodes.

- **Container runtimes**: Slurm integrates with container runtimes such as Singularity/Apptainer, Docker (via plugins), Enroot, or Pyxis. The runtime is configured via the `ContainerType` parameter in `slurm.conf` on the controller node.
- **Container configuration**: Use `--container` to specify the container image path in job scripts. The exact syntax depends on the container runtime configured. The container runs with the user's UID/GID.
- **GPU passthrough**: GPUs are passed through to containers via the device controller. The container sees only the GPUs allocated to the job (via `--gres=gpu`). Environment variables (e.g., `CUDA_VISIBLE_DEVICES`) are set automatically.
- **Filesystem access**: Containers have restricted filesystem access compared to native jobs. Host system paths are hidden unless explicitly mounted. Shared filesystems (home directories, project storage) are typically mounted automatically by the container runtime. See "Container filesystem isolation" in the Filesystems guide for details.

### Container runtime integration specifics

Container runtime integration requires configuration on compute nodes and specification in job scripts.

**Configuration (on controller node, in `slurm.conf`):**
```bash
# Example: Apptainer/Singularity
ContainerType=singularity

# Example: Enroot
ContainerType=enroot

# Example: Docker (requires plugin)
ContainerType=docker
```

**Job script usage (from login node):**
```bash
#!/bin/bash
#SBATCH --job-name=container-job
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --container=/path/to/image.sif

# For Singularity/Apptainer: image file path
#SBATCH --container=/shared/images/pytorch.sif

# For Enroot: image name or path
#SBATCH --container=docker://pytorch/pytorch:latest

# Container runs with user's UID/GID
# GPUs automatically available inside container
srun python train.py
```

**Container runtime behavior:**
- **Image location**: Container images must be accessible from compute nodes (shared filesystem or container registry)
- **User mapping**: Container runs with the submitting user's UID/GID (no root access)
- **Resource allocation**: Container inherits all Slurm resource allocations (CPUs, memory, GPUs)
- **Environment variables**: Slurm environment variables (`SLURM_JOB_ID`, `SLURM_NODELIST`, etc.) are available inside container
- **Filesystem access**: Host filesystems are mounted into container based on runtime configuration (typically home directory and shared storage)

**Example: Singularity/Apptainer job** (script on login node, container executes on compute node):
```bash
#!/bin/bash
#SBATCH --job-name=gpu-training
#SBATCH --nodes=2
#SBATCH --ntasks=16
#SBATCH --gres=gpu:2
#SBATCH --container=/shared/images/pytorch-2.0.sif

# Container image must exist on shared filesystem accessible from compute nodes
# Job runs inside container with user's permissions
srun python distributed_train.py
```

**Example: Enroot job** (script on login node, pulls from registry on compute node):
```bash
#!/bin/bash
#SBATCH --job-name=tensorflow-job
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --container=docker://tensorflow/tensorflow:latest-gpu

# Enroot pulls image from registry on compute node
# Image cached locally on compute node after first pull
srun python train.py
```

## dstack

TBA