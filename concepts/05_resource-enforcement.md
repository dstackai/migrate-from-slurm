# 5 Resource enforcement, containment, and limits

## Slurm

### Resource isolation (Cgroups)

Slurm uses Linux Control Groups (**cgroups**) to strictly confine jobs. This prevents one user's job from crashing a node or stealing resources from another.

- **Mechanism**: Modern clusters use **Cgroup v2** (unified hierarchy).
- **Memory enforcement**: The kernel memory controller creates a hard limit (RAM + Swap). If a job exceeds its requested `--mem`, the kernel OOM Killer immediately terminates the process inside that cgroup. This protects the OS and other jobs.
- **Memory specification**: Use `--mem` to request total memory per node, or `--mem-per-cpu` to request memory per CPU. Memory limits are enforced at the cgroup level, preventing jobs from exceeding their allocation.
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

Slurm supports running jobs inside containers for isolation and reproducibility.

- **Container runtimes**: Slurm integrates with container runtimes. The runtime is configured via the `ContainerType` parameter in `slurm.conf`.
- **Container configuration**: Use `--container` to specify the container image path. The exact syntax depends on the container runtime configured. The container runs with the user's UID/GID.
- **GPU passthrough**: GPUs are passed through to containers via the device controller. The container sees only the GPUs allocated to the job (via `--gres=gpu`). Environment variables are set automatically.
- **Filesystem bind mounts**: Containers can access host filesystems via bind mounts configured by the container runtime.

## dstack

TBA