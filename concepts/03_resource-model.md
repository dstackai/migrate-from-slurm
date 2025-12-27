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

### Resource model & containers

dstack uses Docker containers to run jobs and relies on vendor-specific container drivers for GPU access: NVIDIA Container Toolkit for NVIDIA GPUs, AMD ROCm runtime for AMD GPUs, Tenstorrent container runtime, and Google Cloud TPU runtime for TPUs.

- **Container-based execution**: All jobs run inside Docker containers. dstack automatically configures GPU access by setting appropriate device requests (NVIDIA), device mappings (AMD), or runtime options based on the detected GPU vendor.
- **Resource enforcement**: On VM-based backends, the host enforces memory limits through container constraints. If a container exceeds its memory limit, the host's OOM killer terminates it. If containers collectively exceed the host's total resources, the host may become unresponsive. Container-only backends (Kubernetes) rely on the underlying orchestration layer for enforcement.

### Blocks & resource sharing

dstack supports splitting a host into multiple blocks to allow concurrent jobs on the same instance. This enables better GPU utilization when jobs don't need the full host.

- **Blocks concept**: The `blocks` property divides a host into a specified number of blocks. The number must be an integer not greater than the number of GPUs. When blocks are enabled, dstack splits GPU, CPU, and memory resources proportionally across blocks, while disk is shared. A job may use one or more blocks at once. For example, with 8 GPUs, 128 CPUs, and 2TB RAM, setting `blocks: 8` gives each block 1 GPU, 16 CPUs, and 256 GB RAM.
- **Distributed tasks**: For distributed tasks (multi-node workloads), blocks must be disabled. The job container uses all host resources, including access to high-speed interconnects like InfiniBand, EFA, or GPUDirect-TCPX for inter-node communication. See [cluster node management](08_cluster-node-management.md) for details on cluster placement.
- **Network mode**: When blocks are enabled, containers use `bridge` network mode to avoid port conflicts. When blocks are disabled (single job per instance), containers use `host` network mode for optimal performance.

**Example: fleet configuration with blocks** (`.dstack.yml`):
```yaml
type: fleet
name: my-fleet

# Allow provisioning 0-4 nodes on-demand
nodes: 0..4

resources:
  # Flexible GPU specification with ranges
  gpu: 40GB..80GB:4..8

# Split into 4 blocks, each with 2 GPUs (when 8 GPUs are available)
blocks: 4
```

### Resource configuration

Resources are specified in run configurations (tasks, dev environments, services) via the `resources` property. dstack supports flexible GPU specifications including vendor names, model names, memory ranges, and count ranges.

- **GPU specification format**: The `gpu` property accepts vendor, model names, memory ranges, and count ranges. The general format is `<vendor>:<comma-separated names>:<memory range>:<quantity range>`, where each component is optional. Ranges can be closed (`24GB..80GB`), open (`24GB..`), or single values (`24GB`).
- **GPU vendor support**: dstack supports `nvidia`, `amd`, `tpu` (Google Cloud TPU), `intel` (Gaudi), and `tenstorrent` vendors.
- **Resource examples**: `1` (any GPU), `nvidia:2` (two NVIDIA GPUs), `A100` (one A100), `A10G,A100` (either A10G or A100), `24GB..` (any GPU with at least 24GB), `24GB..40GB:2` (two GPUs between 24GB and 40GB), `A100:80GB` (one 80GB A100), `A100:2` (two A100s), `A100:40GB:2` (two 40GB A100s).

**Example: task with GPU resources** (`.dstack.yml`):
```yaml
type: task
name: train

# Clone repo containing train.py
repos:
  - .:train

resources:
  # 4 GPUs from 40GB to 80GB
  gpu: 40GB..80GB:4
  # 200GB or more RAM
  memory: 200GB..
  # Disk size
  disk: 500GB

commands:
  - pip install torch
  - python train/train.py
```

### Cloud provisioning vs SSH fleets

dstack supports two fleet types:

- **Backend fleets** (cloud provisioning): dstack provisions instances based on fleet configuration by selecting matching instance offers from configured backends. It handles the entire provisioning lifecycle (VMs, drivers, networking, containers). Use `dstack offer` to list available offers that match fleet requirements. When applying a fleet with `dstack apply`, the output shows the selected offers.
- **SSH fleets** (on-premises): dstack automatically detects resources on pre-provisioned machines via SSH and configures them but does not provision new instances.

**Example: listing offers** (from terminal):
```shell
$ dstack offer --gpu 40GB..80GB:4..8

 #   BACKEND  REGION          INSTANCE TYPE    RESOURCES              SPOT  PRICE   
 1   aws      us-east-1       g5.12xlarge     NVIDIA A10G:4 (40GB)   yes   $2.03
 2   gcp      us-central1     a2-highgpu-4g   NVIDIA A100:4 (40GB)   yes   $3.15
```

<!-- TODO: May need to also show idle/busy instances as a part of `dstack offer` -->

### GPU slices

dstack does not support GPU slices (NVIDIA MIG or AMD partitions) out of the box. Each GPU is allocated as a whole unit.

### Exit codes & job termination

dstack tracks exit codes from job execution: exit code 0 marks the job as `done`, non-zero codes mark it as `failed`. When a job is killed (OOM killer or user termination), dstack reports the termination reason and exit code when available.

### Metrics & monitoring

dstack tracks essential metrics (CPU, memory, GPU utilization) accessible via UI and CLI (`dstack metrics`). For advanced metrics, dstack integrates with NVIDIA DCGM on VM-based backends and SSH fleets, providing detailed GPU telemetry. DCGM metrics are exported to Prometheus when enabled. See [GPU health monitoring](11_gpu-health-monitoring.md) and [monitoring observability](15_monitoring-observability.md) for details.