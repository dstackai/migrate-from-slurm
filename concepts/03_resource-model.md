# 3 Resource model, generic resources (TRES/GRES), and enforcement

## Slurm

### Resource model & topology

Slurm models hardware topology to enable precise resource allocation and uses **cgroups** to strictly enforce those limits.

- **Consumable resources**: Large clusters must use `SelectType=select/cons_tres`. This enables the scheduler to track and allocate individual cores and memory segments (consumables) rather than allocating whole nodes to a single job.
- **Hardware topology**: Slurm detects the physical hierarchy (Sockets, Cores, Threads) to optimize task placement. This allows for NUMA-aware scheduling, ensuring tasks run on CPUs physically close to their allocated memory.
- **Trackable resources (TRES)**: This is the internal billing currency of Slurm. Beyond CPU and RAM, it tracks **Energy** (Joules) and **Billing** (synthetic weights). Billing weights allow admins to charge users more for premium hardware.

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
#SBATCH --job-name=gpu-training
#SBATCH --nodes=1
#SBATCH --gres=gpu:2  # Request 2 GPUs of any type
#SBATCH --mem=100G
#SBATCH --time=12:00:00

# Slurm sets CUDA_VISIBLE_DEVICES automatically
# PyTorch will use both GPUs via DataParallel or DistributedDataParallel
srun python train.py --model=transformer --batch-size=32
```
- **Example: requesting specific GPU type** (script on login node, executes on compute nodes):
```bash
#!/bin/bash
#SBATCH --job-name=a100-training
#SBATCH --nodes=1
#SBATCH --gres=gpu:a100:2  # Request 2 A100 GPUs specifically (80GB each)
#SBATCH --mem=200G
#SBATCH --time=24:00:00

# Job will only run on nodes with A100 GPUs available
# Large model training requiring high-memory GPUs
srun python train_large_model.py --model=llama-7b --precision=bf16
```
- **Example: checking GPU allocation** (from login node):
  ```bash
  # Show GPU allocation for running job
  scontrol show job 12345 | grep -i gres
  # Gres=gpu:2(IDX:0,1)
  ```
- **Multi-Instance GPU (MIG)**: On NVIDIA A100/H100 hardware, Slurm can manage MIG "slices" (hardware partitions) as distinct GRES types. This allows multiple small jobs to safely share a single physical GPU without memory interference.

### Resource enforcement

Slurm uses Linux Control Groups (**cgroups**) to strictly confine jobs and enforce resource limits. This prevents one user's job from crashing a node or stealing resources from another.

#### Cgroups mechanism

- **Cgroup version**: Modern clusters use **Cgroup v2** (unified hierarchy), though some older clusters may still use Cgroup v1. Slurm uses `task/cgroup` plugin type.
- **Enforcement scope**: When you request resources (CPU, memory, GPUs), Slurm creates a cgroup for your job and enforces those exact limits at the kernel level.

#### Memory enforcement

Memory can be requested in two ways, and limits are enforced per your specification:

- **`--mem=M`**: Requests M memory **per node**. If `--nodes=4 --mem=200G`, each of the 4 nodes gets 200GB, totaling 800GB across all nodes. This is the typical approach for ML workloads.
- **`--mem-per-cpu=M`**: Requests M memory **per CPU**. If `--ntasks=8 --cpus-per-task=2 --mem-per-cpu=4G`, total memory = 8 tasks × 2 CPUs × 4GB = 64GB per node. Useful for multi-process data loading scenarios.

**Enforcement behavior**: Memory limits are enforced at the cgroup level on each compute node. The kernel memory controller creates a hard limit (RAM + Swap). If a job exceeds its requested memory on any node, the kernel OOM Killer immediately terminates the process inside that cgroup, preventing it from crashing the physical node and protecting other jobs.

**Example: per-node memory** (job script on login node, memory enforced on compute node):
```bash
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --ntasks=8
#SBATCH --gres=gpu:4
#SBATCH --mem=200G  # 200GB per node (400GB total across 2 nodes)
#SBATCH --time=24:00:00

# Distributed training with PyTorch DDP
# If job exceeds 200GB on any node, OOM killer terminates it
srun python distributed_training.py --batch-size=64 --model=resnet50
```

**Example: per-CPU memory** (job script on login node, memory enforced on compute node):
```bash
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --cpus-per-task=2
#SBATCH --gres=gpu:1
#SBATCH --mem-per-cpu=4G  # 4GB per CPU = 4 tasks × 2 CPUs × 4GB = 32GB total

# Multi-process data loading with GPU training
srun python train_model.py --num-workers=4
```

#### CPU affinity & pinning

Beyond simple enforcement, Slurm manages **CPU Affinity** to maximize performance.

- **Task affinity**: Slurm pins (binds) tasks to specific CPU cores. Tasks can be bound to specific cores or sockets (allowing movement between cores on that socket).
- **NUMA awareness**: Slurm prioritizes allocating memory from the RAM bank closest to the pinned CPU core to minimize latency.

#### GPU device isolation

The cgroup device controller creates an allow-list for GPU access. If you request `--gres=gpu:1`, the cgroup ensures your process can literally only see `/dev/nvidia0` and is denied access to other GPUs. This physical isolation complements the logical isolation provided by environment variables like `CUDA_VISIBLE_DEVICES`.

#### Process containment & tracking

Slurm ensures that when a job ends, *everything* ends.

- **Proctrack plugin**: Slurm uses `ProctrackType=proctrack/cgroup`. This tracks processes based on their cgroup membership rather than Parent Process ID (PPID).
- **Benefit**: Even if a user's job "daemonizes" (double-forks) to detach from its parent, it remains trapped inside the cgroup.
- **Cleanup**: When a job is cancelled or finishes, `slurmstepd` simply destroys the cgroup. The kernel then guarantees that every process inside that container is killed instantly. This eliminates "zombie processes" on compute nodes.

#### Walltime enforcement

Time limits are strictly enforced to keep the queue moving.

- **Tracking**: `slurmstepd` monitors the elapsed time of the allocation.
- **Grace period**: When the limit is reached, Slurm sends `SIGTERM` first. The application has a configured grace period (default 30-60s) to save checkpoints and exit cleanly.
- **Hard kill**: If the process is still running after the grace period, Slurm sends `SIGKILL` (immediate termination).

#### Additional resource limits

Slurm can optionally enforce CPU time limits via cgroups (in addition to walltime), though this is not enabled on all clusters.

#### Container integration

Slurm supports running jobs inside containers for isolation and reproducibility. Container runtime is configured on compute nodes via `slurm.conf`, while users specify container images in job scripts from login nodes.

- **Container runtimes**: Slurm integrates with container runtimes such as Singularity/Apptainer, Docker (via plugins), Enroot, or Pyxis. The runtime is configured via the `ContainerType` parameter in `slurm.conf` on the controller node.
- **Container configuration**: Use `--container` to specify the container image path in job scripts. The exact syntax depends on the container runtime configured. The container runs with the user's UID/GID.
- **GPU passthrough**: GPUs are passed through to containers via the device controller. The container sees only the GPUs allocated to the job (via `--gres=gpu`). Environment variables (e.g., `CUDA_VISIBLE_DEVICES`) are set automatically.
- **Filesystem access**: Containers have restricted filesystem access compared to native jobs. Host system paths are hidden unless explicitly mounted. Shared filesystems (home directories, project storage) are typically mounted automatically by the container runtime. See "Container filesystem isolation" in the Filesystems guide for details.

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

**Example: Singularity/Apptainer** (image file path):
```bash
#!/bin/bash
#SBATCH --job-name=container-training
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --mem=100G
#SBATCH --time=12:00:00
#SBATCH --container=/shared/images/pytorch-2.0-cuda11.8.sif

# Container runs with user's UID/GID
# GPUs automatically available inside container
srun python train.py --model=resnet50 --epochs=100
```

**Example: Enroot** (pulls from registry):
```bash
#!/bin/bash
#SBATCH --job-name=container-training
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --mem=100G
#SBATCH --time=12:00:00
#SBATCH --container=docker://pytorch/pytorch:2.0.0-cuda11.8-cudnn8-runtime

# Enroot pulls image from registry on compute node
# Container runs with user's UID/GID
# GPUs automatically available inside container
srun python train.py --model=resnet50 --epochs=100
```

**Example: Singularity/Apptainer job** (script on login node, container executes on compute node):
```bash
#!/bin/bash
#SBATCH --job-name=gpu-training
#SBATCH --nodes=2
#SBATCH --ntasks=16
#SBATCH --gres=gpu:2
#SBATCH --mem=200G
#SBATCH --time=24:00:00
#SBATCH --container=/shared/images/pytorch-2.0.sif

# Container image must exist on shared filesystem accessible from compute nodes
# Job runs inside container with user's permissions
# Distributed training with PyTorch DDP
srun python distributed_train.py --batch-size=64
```

## dstack

### Resource model & containers

dstack uses Docker containers to run jobs and relies on vendor-specific container drivers for GPU access: NVIDIA Container Toolkit for NVIDIA GPUs, AMD ROCm runtime for AMD GPUs, Tenstorrent container runtime, and Google Cloud TPU runtime for TPUs.

- **Container-based execution**: All jobs run inside Docker containers. dstack automatically configures GPU access by setting appropriate device requests (NVIDIA), device mappings (AMD), or runtime options based on the detected GPU vendor. The container runtime handles GPU device filtering and isolation.

### Blocks & resource sharing

dstack supports splitting a host into multiple blocks to allow concurrent jobs on the same instance. This enables better GPU utilization when jobs don't need the full host.

- **Blocks concept**: The `blocks` property divides a host into a specified number of blocks. The number must be an integer not greater than the number of GPUs. When blocks are enabled, dstack splits GPU, CPU, and memory resources proportionally across blocks, while disk is shared. A job may use one or more blocks at once. For example, with 8 GPUs, 128 CPUs, and 2TB RAM, setting `blocks: 8` gives each block 1 GPU, 16 CPUs, and 256 GB RAM.
- **Distributed tasks**: For distributed tasks (multi-node workloads), blocks must be disabled. The job container uses all host resources, including access to high-speed interconnects like InfiniBand, EFA, or GPUDirect-TCPX for inter-node communication. See [cluster node management](07_cluster-node-management.md) for details on cluster placement.
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
- **Shared memory (`shm_size`)**: Docker containers have a default shared memory size of 64MB (`/dev/shm`), which is often insufficient for multi-GPU and multi-node workloads. The `shm_size` property configures the shared memory size. For distributed training with PyTorch or other frameworks that use shared memory for inter-process communication, set `shm_size` to at least `16GB` or `24GB` depending on the workload. This is especially critical for multi-node distributed tasks where processes communicate via shared memory.

**Example: task with GPU resources for multi-GPU training** (`.dstack.yml`):
```yaml
type: task
name: train-transformer

# Clone repo containing training script
repos:
  - .:train

resources:
  # 4 GPUs from 40GB to 80GB (for large model training)
  gpu: 40GB..80GB:4
  # 200GB or more RAM (for large batch sizes and model weights)
  memory: 200GB..
  # Shared memory (required for multi-GPU PyTorch DDP)
  shm_size: 24GB
  # Disk size for datasets and checkpoints
  disk: 500GB

commands:
  - pip install torch torchvision transformers
  # Use torchrun for multi-GPU training on single node
  - torchrun --nproc-per-node=$DSTACK_GPUS_PER_NODE train/train.py --model transformer --batch-size 32
```

**Example: task with GPU resources for multi-node distributed training** (`.dstack.yml`):
```yaml
type: task
name: train-transformer-distributed

# Clone repo containing training script
repos:
  - .:train

# Multi-node configuration (requires fleet with placement: cluster)
nodes: 2..4

resources:
  # 4 GPUs per node (for large-scale distributed training)
  gpu: 40GB..80GB:4
  # 200GB or more RAM per node
  memory: 200GB..
  # Shared memory (required for multi-node PyTorch DDP)
  shm_size: 24GB
  # Disk size for datasets and checkpoints
  disk: 500GB

commands:
  - pip install torch torchvision transformers
  # Use torchrun for multi-node multi-GPU training
  # See job execution lifecycle guide for distributed training details
  - |
    torchrun \
      --nproc-per-node=$DSTACK_GPUS_PER_NODE \
      --node-rank=$DSTACK_NODE_RANK \
      --nnodes=$DSTACK_NODES_NUM \
      --master-addr=$DSTACK_MASTER_NODE_IP \
      --master-port=12345 \
      train/train.py --model transformer --batch-size 32
```

For details on distributed training setup and execution, see [Job execution lifecycle](04_job-execution-lifecycle.md).

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

- **GPU slices**: dstack does not support GPU slices (NVIDIA MIG or AMD partitions) out of the box. Each GPU is allocated as a whole unit.

### Resource enforcement

In dstack, resource enforcement is tightly coupled with resource allocation. When you specify resources (possibly with ranges), dstack selects an instance offer that matches your requirements, and enforcement limits are based on the **actual offer capacity**, which may exceed your minimum requested values.

#### Resource allocation and enforcement

- **Resource ranges**: When you specify a range (e.g., `memory: 200GB..`), dstack selects an offer that meets the minimum requirement. The enforcement limit is based on the offer's maximum capacity, not your minimum request.
- **Example**: If you request `memory: 200GB..` and dstack selects an offer with 256GB capacity, your container will be limited to 256GB, not 200GB.
- **From user's perspective**: The resource model and enforcement are the same thing - you request resources, dstack allocates them, and you're limited to what was allocated.

#### Container-based enforcement

- **VM-based backends**: The host enforces memory and CPU limits through Docker container constraints (`hostConfig.Memory`, `hostConfig.NanoCPUs`). If a container exceeds its memory limit, the host's OOM killer terminates it. If containers collectively exceed the host's total resources, the host may become unresponsive. Docker uses cgroups (on Linux hosts) to enforce these limits at the kernel level.
- **Kubernetes/other container-based backends**: Container-only backends rely on the orchestration layer (Kubernetes, Nomad, etc.) for resource limits and requests. These systems enforce limits via cgroups (similar to Slurm) but managed by the orchestration layer. They typically set both `requests` (minimum guaranteed) and `limits` (maximum allowed) for each container.
- **Enforcement mechanism**: Docker containers are constrained at the kernel level using cgroups (on Linux hosts). Memory limits are hard limits - exceeding them triggers the OOM killer. CPU limits are enforced through CPU shares/quota mechanisms.
- **GPU isolation**: When you request GPUs (e.g., `gpu: 40GB..80GB:4`), dstack selects specific GPUs and configures the container to only access those GPUs. For NVIDIA GPUs, the NVIDIA Container Toolkit filters GPU access via Docker's `--gpus` flag. For AMD GPUs, the ROCm runtime maps only the allocated GPUs to the container.
- **Blocks enforcement**: When blocks are enabled, resource limits are enforced per block. Each block's container is independently constrained to its allocated share of resources (GPUs, CPU, memory).

#### Job termination

When a container exceeds its memory limit, the host's OOM killer terminates it and dstack reports the termination reason. Unlike Slurm's walltime enforcement, dstack does not enforce time limits by default.