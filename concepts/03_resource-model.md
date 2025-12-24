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
- **Example: gres.conf with auto-detection** (located on compute nodes, or controller if configless):
  ```bash
  # Auto-detect NVIDIA GPUs using NVML
  AutoDetect=nvml
  ```
- **Example: gres.conf manual configuration** (located on compute nodes):
  ```bash
  # Node gpu-node01 has 4 A100 GPUs
  NodeName=gpu-node01 Name=gpu Type=a100 File=/dev/nvidia[0-3]
  ```
- **GPU binding & locality**: Slurm ensures tasks talk to the GPU physically closest to their allocated CPU core (PCIe locality). This optimizes bandwidth.
- **Isolation**: Slurm sets environment variables so the user's application only "sees" the specific GPUs allocated to it, preventing conflicts.
- **Requesting specific GPU types**: Users can request specific GPU models or types. The exact syntax depends on cluster configuration. If the cluster defines typed GRES in `gres.conf` (e.g., `Name=gpu Type=a100`), users can request specific types using `--gres=gpu:type:count` (e.g., `--gres=gpu:a100:2`). Some clusters may require using `--constraint` instead. If no type is specified in `--gres=gpu:count`, any available GPU is allocated. GPU types typically correspond to specific memory sizes (e.g., A100 has 40GB or 80GB variants), so selecting by type effectively selects by model and memory capacity.
- **Example: requesting any GPU** (script on login node, executes on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=gpu-job
  #SBATCH --nodes=1
  #SBATCH --gres=gpu:2  # Request 2 GPUs of any type
  
  # Slurm sets CUDA_VISIBLE_DEVICES automatically
  srun python train.py
  ```
- **Example: requesting specific GPU type** (script on login node, executes on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=a100-job
  #SBATCH --nodes=1
  #SBATCH --gres=gpu:a100:2  # Request 2 A100 GPUs specifically
  
  # Job will only run on nodes with A100 GPUs available
  srun python train.py
  ```
- **Example: checking GPU allocation** (from login node):
  ```bash
  # Show GPU allocation for running job
  scontrol show job 12345 | grep -i gres
  # Gres=gpu:2(IDX:0,1)
  ```
- **Multi-Instance GPU (MIG)**: On NVIDIA A100/H100 hardware, Slurm can manage MIG "slices" (hardware partitions) as distinct GRES types. This allows multiple small jobs to safely share a single physical GPU without memory interference.

## dstack

TBA