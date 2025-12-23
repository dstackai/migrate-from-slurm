# 3 Resource model & generic resources (TRES/GRES)

## Slurm

### Resource model & topology

Slurm models hardware topology to enable precise resource allocation and uses **cgroups** to strictly enforce those limits.

- **Consumable resources**: Large clusters must use `SelectType=select/cons_tres`. This enables the scheduler to track and allocate individual cores and memory segments (consumables) rather than allocating whole nodes to a single job.
- **Hardware topology**: Slurm detects the physical hierarchy (Sockets, Cores, Threads) to optimize task placement. This allows for NUMA-aware scheduling, ensuring tasks run on CPUs physically close to their allocated memory.
- **Trackable resources (TRES)**: This is the internal billing currency of Slurm. Beyond CPU and RAM, it tracks **Energy** (Joules) and **Billing** (synthetic weights). Billing weights allow admins to charge users more for premium hardware.
- **Enforcement (Cgroups)**: Slurm uses Linux Control Groups (`task/cgroup`) to strictly confine jobs. If a job requests 4GB RAM, the cgroup kernel mechanism ensures the process is killed (OOM) if it exceeds that limit, preventing it from crashing the physical node.

### Generic resources (GRES) & GPUs

Accelerators (GPUs) are managed via the GRES system, which handles both the logical allocation and the physical hardware binding.

- **GRES types**: This system manages arbitrary hardware, most commonly **GPUs** (NVIDIA/AMD) or **FPGAs**.
- **Configuration (Auto-detect)**: Modern large clusters use auto-detection in `gres.conf`. Instead of manually mapping device files in config text (prone to error), Slurm queries the GPU driver at startup to automatically discover GPU counts, models, and links.
- **GPU binding & locality**: Slurm ensures tasks talk to the GPU physically closest to their allocated CPU core (PCIe locality). This optimizes bandwidth.
- **Isolation**: Slurm sets environment variables so the user's application only "sees" the specific GPUs allocated to it, preventing conflicts.
- **Multi-Instance GPU (MIG)**: On NVIDIA A100/H100 hardware, Slurm can manage MIG "slices" (hardware partitions) as distinct GRES types. This allows multiple small jobs to safely share a single physical GPU without memory interference.

## dstack

TBA