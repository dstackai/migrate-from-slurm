# Migration Guide: Slurm to dstack

This guide provides a comprehensive comparison between Slurm and dstack workload orchestration systems, enabling Slurm users and administrators to successfully transition GPU-accelerated workloads to dstack. This document covers architectural similarities, key differences, and practical examples demonstrating feature implementations in both systems.

## Overview: Similarities and differences

### System purpose

Both Slurm and dstack are open-source workload orchestration systems designed to manage compute resources and schedule jobs. However, they target different deployment models:

- **Slurm**: Optimized for HPC and AI workloads on static on-premises clusters. Focuses on job scheduling, queuing, and resource management in pre-configured environments. Widely deployed in academic and research institutions for large-scale distributed training.
- **dstack**: Designed for AI workloads (development, training, inference) with cloud-native architecture. Supports large-scale distributed training, day-to-day development workflows, and production-grade inference. Optimized for ML engineers and researchers leveraging GPU cloud capacity with dynamic provisioning and container-based execution.

### Shared capabilities

Both systems provide the following core capabilities:

- **Workload orchestration**: Management of compute resources and job scheduling across multiple nodes
- **Multi-node distributed training**: Support for multi-node GPU workloads with proper networking and coordination
- **Resource allocation and scheduling**: Allocation of CPU, memory, and GPU resources based on job requirements
- **GPU workload optimization**: Optimized for ML/AI workloads requiring GPU acceleration
- **Authentication and authorization**: Mechanisms to control user access and resource usage
- **Interactive sessions**: Resource allocation for interactive development and experimentation
- **Monitoring capabilities**: Tools to track job status, resource usage, and system health

### Architectural differences

The systems differ in the following key areas:

- **Use-case scope**: Slurm targets HPC and AI workloads on on-premises clusters, while dstack is use-case agnostic and supports any containerized workload (development, training, inference, services)
- **Architecture**: Slurm uses native process execution on static nodes (workloads have direct access to the host), while dstack uses a container-first architecture where all workloads run in Docker containers. Slurm supports containers as an optional feature, while dstack is designed with containers as a core architectural principle
- **Infrastructure model**: Slurm assumes static, pre-configured nodes, while dstack provides native cloud provisioning support, dynamically provisioning clusters and instances as needed. dstack natively integrates with leading GPU cloud providers (Lambda, Nebius, RunPod, Vast.ai, Verda, AWS, GCP) for GPU instance provisioning
- **Backend flexibility**: Slurm requires dedicated compute nodes, while dstack supports Kubernetes (as a container-based backend) or VM-based backends and SSH fleets, providing flexibility to match your infrastructure requirements
- **Resource model**: Slurm manages static nodes with fixed resources, while dstack uses dynamic instance provisioning that scales based on demand

### Migration recommendations

**Consider migrating to dstack when:**

- Native cloud provisioning support is required (dstack supports both cloud and on-premises infrastructure natively, with native integration to leading GPU cloud providers)
- Workloads are primarily ML/AI training jobs that benefit from dynamic GPU provisioning
- A use-case agnostic tool is needed that supports day-to-day development and production-grade inference
- Container-based execution is preferred for consistent environments and dependency isolation without managing container runtimes
- Cost optimization through spot instances is important
- Simplified configuration and workflow management is preferred

### Architecture and concept mapping

Both Slurm and dstack follow a client-server architecture with a centralized control plane and distributed compute agents. The following table maps Slurm concepts to their dstack equivalents:

| Slurm concept | dstack equivalent | Notes |
|---------------|-------------------|-------|
| `slurmctld` (controller) | `dstack-server` | Central control plane managing jobs and resources |
| `slurmdbd` (database) | `dstack-server` (SQLite/PostgreSQL) | State persistence (dstack combines controller and database) |
| `slurmrestd` (REST API) | `dstack-server` (HTTP API) | API access (dstack combines all three) |
| `slurmd` (compute agent) | `dstack-shim` (VM-based) or `dstack-runner` (container-based) | Agent running on compute nodes/instances |
| Login nodes | CLI (runs from anywhere) | Job submission interface (dstack CLI can run from laptop, CI/CD, etc.) |
| Partitions | Fleets | Instance/cluster pools and provisioning templates (Slurm: logical grouping, dstack: physical pools) |
| Shared filesystem (NFS/Lustre/GPFS) | Network volumes or SSH fleet filesystems | Data access (dstack uses explicit volume mounting) |
| MUNGE authentication | Token-based authentication | Inter-component security |
| Static compute nodes | Dynamic instances | Resource model (Slurm: fixed nodes, dstack: on-demand provisioning) |

**Architectural implementation:**

**Slurm:**
- **Control plane**: Separate services (`slurmctld`, `slurmdbd`, `slurmrestd`) that can be distributed across nodes
- **Compute agents**: `slurmd` daemons execute jobs as native processes on the host
- **Job submission**: Requires SSH access to login nodes where users run `sbatch`, `srun`, or `salloc`
- **Filesystem**: Assumes shared filesystem (NFS, Lustre, GPFS) with global namespace
- **Authentication**: Uses MUNGE for inter-daemon authentication
- **High availability**: Active-passive high availability (typically 2 controller nodes)

**dstack:**
- **Control plane**: Single `dstack-server` service combining controller, database, and API (can scale horizontally with PostgreSQL)
- **Compute agents**: 
  - VM-based backends: `dstack-shim` agent manages the VM and starts Docker containers for user jobs
  - Container-based backends: `dstack-runner` agent accepts and executes user jobs inside containers
- **Job submission**: CLI can run from anywhere (laptop, CI/CD pipelines, developer machines) and communicates with server via HTTP (no login nodes required)
- **Filesystem**: Supports shared filesystems via SSH fleets (using instance volumes with pre-mounted network storage). For backend fleets, mounting shared storage via instance volumes may require dedicated endpoints (e.g., init scripts), which is not currently supported
- **Authentication**: Token-based authentication for CLI/API access, project-based access control with user roles (Admin, Manager, User)
- **High availability**: Supports horizontal scaling with multiple server replicas (requires PostgreSQL for state persistence)

## Feature comparison and implementation

### Job submission and configuration

Job submission is the primary interface for user interaction in both systems. Slurm uses shell scripts with embedded directives, while dstack uses declarative YAML configurations. Slurm jobs are submitted from login nodes, while dstack jobs can be submitted from anywhere (laptop, CI/CD) via the CLI.

#### Slurm job script

Slurm uses shell scripts with `#SBATCH` directives embedded in the script:

```bash
#!/bin/bash
#SBATCH --job-name=train-model
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --gres=gpu:1
#SBATCH --mem=32G
#SBATCH --time=2:00:00
#SBATCH --partition=gpu
#SBATCH --output=train-%j.out
#SBATCH --error=train-%j.err

export HF_TOKEN
export LEARNING_RATE=0.001

module load python/3.9
srun python train.py --batch-size=64
```

#### dstack task configuration

dstack uses declarative YAML configuration files. The following example demonstrates a single-node training configuration equivalent to the Slurm example above:

```yaml
type: task
name: train-model

python: 3.9
repos:
  - .

env:
  - HF_TOKEN
  - LEARNING_RATE=0.001

commands:
  - python train.py --batch-size=64

resources:
  gpu: 1
  memory: 32GB

max_duration: 2h
```

For detailed multi-node distributed training examples, refer to the [Multi-node distributed training](#multi-node-distributed-training) section.

#### Configuration comparison

| Slurm | dstack | Explanation |
|-------|--------|-------------|
| Shell script with `#SBATCH` directives | YAML configuration file (`.dstack.yml`) | Slurm uses executable scripts; dstack uses declarative YAML |
| `--gres=gpu:1` | `gpu: 1` | Both specify GPU count (dstack can also specify GPU type and memory range) |
| `--mem=32G` | `memory: 32GB` | Slurm exact value; dstack supports ranges (minimum requirement) |
| `--time=2:00:00` | `max_duration: 2h` | Slurm enforces walltime; dstack uses for cost control |
| `--partition=gpu` | Fleet selection (automatic or explicit) | Slurm requires partition; dstack auto-selects or uses `fleets` property |
| `--output=train-%j.out` | Logs via `dstack logs` or UI | Slurm writes files; dstack streams logs via API |
| `export VAR` or `--export=ALL,VAR=value` | `env: - VAR` or `--env VAR=value` | Environment variables in Slurm; dstack supports both definition in YAML and CLI override |

**CLI commands for job submission:**

The `dstack apply` command submits runs from YAML configuration files. It automatically attaches to the run unless `--detach` is used. The CLI supports various options (environment variables, fleet selection, naming) that are covered in relevant sections below.

**Slurm:**
```bash
# Submit job
sbatch train.sh
# Submitted batch job 12345

# Submit with environment variables (overrides script defaults)
sbatch --export=ALL,LEARNING_RATE=0.002 train.sh
```

**dstack:**
```bash
# Submit run (uses env vars from YAML config)
dstack apply -f train.dstack.yml

# Submit with environment variables (overrides YAML config)
dstack apply -f train.dstack.yml --env LEARNING_RATE=0.002
```

### Container images

Both systems support containerized execution, with different implementation approaches.

#### Slurm container execution

Slurm supports container execution using Singularity/Apptainer or Enroot. The container image must exist on a shared filesystem or be pulled from a registry:

```bash
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --mem=32G
#SBATCH --time=2:00:00
#SBATCH --container=/shared/images/pytorch-2.0-cuda11.8.sif

# Container image must exist on shared filesystem
# GPUs automatically available inside container
srun python train.py --batch-size=64
```

**Using Enroot (pulls from registry):**
```bash
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --mem=32G
#SBATCH --time=2:00:00
#SBATCH --container=docker://pytorch/pytorch:2.0.0-cuda11.8-cudnn8-runtime

srun python train.py --batch-size=64
```

#### dstack container images

dstack uses container images by default. You can specify a pre-built image or use a base image with Python and other dependencies:

**Using pre-built image:**
```yaml
type: task
name: train-with-image

image: pytorch/pytorch:2.0.0-cuda11.8-cudnn8-runtime

repos:
  - .

commands:
  - python train.py --batch-size=64

resources:
  gpu: 1
  memory: 32GB
```

**Using NVIDIA NGC image (with authentication):**
```yaml
type: task
name: train-ngc

image: nvcr.io/nvidia/pytorch:24.01-py3

registry_auth:
  username: $oauthtoken
  password: ${{ secrets.nvidia_ngc_api_key }}

repos:
  - .

commands:
  - python train.py --batch-size=64

resources:
  gpu: 1
  memory: 32GB
```

### Resource specification

Resource specification determines what compute resources (CPU, memory, GPUs) are allocated to jobs. In Slurm, resources are specified via `#SBATCH` directives in job scripts. In dstack, resources are specified in the YAML configuration's `resources` section. Slurm requires exact resource matching on static nodes, while dstack can dynamically provision instances that match resource requirements (with ranges and flexibility).

#### Slurm resource directives

In Slurm, resources are specified via `#SBATCH` directives in job scripts. Resources must match exactly what's available on static nodes:

```bash
#!/bin/bash
#SBATCH --nodes=1          # 1 compute node
#SBATCH --ntasks=1         # 1 process
#SBATCH --cpus-per-task=8  # 8 CPUs
#SBATCH --gres=gpu:1       # 1 GPU
#SBATCH --mem=32G          # 32GB RAM
```

#### dstack resource specification

In dstack, resources are specified in the YAML configuration's `resources` section. dstack can dynamically provision instances that match resource requirements, supporting ranges and flexibility:

```yaml
type: task
name: training-job

python: 3.12
repos:
  - .

commands:
  - python train.py --batch-size=64

resources:
  gpu: 1                  # 1 GPU
  memory: 32GB            # 32GB memory (minimum)
  cpu: 8                  # 8 CPUs
  shm_size: 8GB           # Shared memory

max_duration: 2h
```

**Resource ranges:** dstack supports flexible resource ranges (e.g., `gpu: 40GB..80GB:1..8`). When you specify a range, dstack selects any instance type that matches the criteria. For example, `gpu: 40GB..80GB:1..8` matches instances with 1-8 GPUs where each GPU has 40-80GB memory. dstack automatically selects the best match based on availability and cost optimization.

**Note**: For multi-node distributed training examples, refer to the [Multi-node distributed training](#multi-node-distributed-training) section.

#### Resource model comparison

| Concept | Slurm | dstack |
|---------|-------|--------|
| **GPU specification** | `--gres=gpu:N` or `--gres=gpu:type:N` | `gpu: A100:80GB:4` or `gpu: 40GB..80GB:2..8` |
| **Memory** | `--mem=M` (per node) or `--mem-per-cpu=M` | `memory: 200GB..` (range, per node) |
| **CPU** | `--cpus-per-task=C` or `--ntasks` | `cpu: 32` (per node) |
| **Shared memory** | Configured on host | `shm_size: 24GB` (explicit) |
| **Resource enforcement** | cgroups (v2) | Docker container constraints (cgroups) |

### Multi-node distributed training

Multi-node distributed training is essential for training large models that exceed single GPU or node capacity. Both Slurm and dstack support multi-node GPU workloads, with different coordination and topology handling mechanisms.

**Key aspects:**
- **Process coordination**: Slurm provides environment variables (`SLURM_NODELIST`, `SLURM_PROCID`) for process coordination, while dstack provides `DSTACK_*` environment variables
- **Cluster placement**: Distributed tasks in dstack require fleets with `placement: cluster` configured. This ensures instances are provisioned with optimal inter-node connectivity (InfiniBand, EFA, GPUDirect) for high-bandwidth GPU communication. Cluster placement is separate from topology-aware scheduling (which dstack does not support automatically except in specific backend cases)
- **Topology-aware scheduling**: dstack does not support automatic topology-aware scheduling (placing jobs on physically close nodes) except in specific cases: AWS fleets with cluster placement automatically configure EFA networking when supported, and GCP fleets automatically configure GPUDirect-TCPXO/GPUDirect-TCPX or RoCE networking for certain instance types. For other cases, manually configure network topology in scripts

#### Slurm PyTorch DDP training

Slurm provides environment variables for distributed coordination. You must manually set up the master address and port:

```bash
#!/bin/bash
#SBATCH --job-name=distributed-train
#SBATCH --nodes=4
#SBATCH --ntasks=32          # 32 GPUs total (8 per node)
#SBATCH --gres=gpu:8         # 8 GPUs per node
#SBATCH --mem=200G
#SBATCH --time=24:00:00
#SBATCH --partition=gpu

# Set up distributed training environment
export MASTER_ADDR=$(scontrol show hostnames $SLURM_NODELIST | head -n1)
export MASTER_PORT=12345
export WORLD_SIZE=$SLURM_NTASKS
export RANK=$SLURM_PROCID

# Launch training with torchrun
srun python -m torch.distributed.launch \
  --nproc_per_node=8 \
  --nnodes=$SLURM_JOB_NUM_NODES \
  --node_rank=$SLURM_NODEID \
  --master_addr=$MASTER_ADDR \
  --master_port=$MASTER_PORT \
  train.py \
  --model llama-7b \
  --batch-size=32 \
  --epochs=10
```

#### dstack PyTorch DDP training

dstack automatically sets up environment variables for distributed coordination. Distributed tasks require a fleet with `placement: cluster` configured to ensure optimal inter-node connectivity. On AWS, EFA networking is automatically configured when supported. On GCP, GPUDirect-TCPXO/GPUDirect-TCPX or RoCE networking is automatically configured for certain instance types (A3 Mega, A3 High, A4). On other backends, networking must be manually configured.

dstack automatically injects the following environment variables into each container:
- `DSTACK_NODES_NUM`: Total number of nodes
- `DSTACK_NODE_RANK`: Rank of the current node (0-based)
- `DSTACK_GPUS_PER_NODE`: Number of GPUs per node
- `DSTACK_GPUS_NUM`: Total number of GPUs across all nodes
- `DSTACK_MASTER_NODE_IP`: IP address of the master node (rank 0)
- `DSTACK_NODES_IPS`: Newline-delimited list of internal IP addresses of all nodes

These variables allow distributed frameworks (PyTorch DDP, Horovod, etc.) to discover and coordinate with other nodes.

**Example configuration:**

```yaml
type: task
name: distributed-train-pytorch

nodes: 4

python: 3.12
repos:
  - .

env:
  - NCCL_DEBUG=INFO
  - NCCL_IB_DISABLE=0
  - NCCL_SOCKET_IFNAME=eth0

commands:
  - |
    torchrun \
      --nproc-per-node=$DSTACK_GPUS_PER_NODE \
      --node-rank=$DSTACK_NODE_RANK \
      --nnodes=$DSTACK_NODES_NUM \
      --master-addr=$DSTACK_MASTER_NODE_IP \
      --master-port=12345 \
      train.py \
      --model llama-7b \
      --batch-size=32 \
      --epochs=10

resources:
  gpu: A100:80GB:8
  memory: 200GB..
  shm_size: 24GB

max_duration: 24h
```

**Important**: This task must run on a fleet with `placement: cluster` configured. Refer to the [Fleets section](#partitions-and-fleets) for configuration details.

#### Slurm MPI (NCCL tests)

This example demonstrates running NCCL tests using MPI, which is useful for validating multi-node GPU communication. Slurm handles MPI coordination automatically:

```bash
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --ntasks=16
#SBATCH --gres=gpu:8
#SBATCH --mem=200G
#SBATCH --time=24:00:00

export MASTER_ADDR=$(scontrol show hostnames $SLURM_NODELIST | head -n1)
export MASTER_PORT=12345

# MPI with NCCL tests or custom MPI application
srun mpirun \
  --allow-run-as-root \
  --hostfile $SLURM_JOB_NODELIST \
  -n $SLURM_NTASKS \
  -N $SLURM_NTASKS_PER_NODE \
  --bind-to none \
  /opt/nccl-tests/build/all_reduce_perf -b 8 -e 8G -f 2 -g 1
```

#### dstack MPI (NCCL tests)

For MPI workloads that require specific job startup and termination behavior, dstack provides `startup_order` and `stop_criteria` properties. The master node (rank 0) runs the MPI command, while worker nodes wait for the master to complete. This example also demonstrates how to verify cluster placement and test inter-node connectivity:

```yaml
type: task
name: nccl-tests

nodes: 2
startup_order: workers-first
stop_criteria: master-done

env:
  - NCCL_DEBUG=INFO

commands:
  - |
    if [ $DSTACK_NODE_RANK -eq 0 ]; then
      mpirun \
        --allow-run-as-root \
        --hostfile $DSTACK_MPI_HOSTFILE \
        -n $DSTACK_GPUS_NUM \
        -N $DSTACK_GPUS_PER_NODE \
        --bind-to none \
        /opt/nccl-tests/build/all_reduce_perf -b 8 -e 8G -f 2 -g 1
    else
      sleep infinity
    fi

resources:
  gpu: nvidia:1..8
  shm_size: 16GB
```

**Implementation details:**
- `startup_order: workers-first` ensures worker nodes start before the master node
- `stop_criteria: master-done` causes all worker jobs to terminate when the master completes
- The master node runs the MPI command using `mpirun` with the pre-populated `DSTACK_MPI_HOSTFILE`
- **Verification**: NCCL tests validate cluster placement and inter-node connectivity. Use `dstack logs` to verify all nodes are in the same availability zone/region

**Environment variables comparison:**

| Slurm | dstack | Purpose |
|-------|--------|---------|
| `SLURM_NODELIST` | `DSTACK_NODES_IPS` | Newline-delimited list of node IPs |
| `SLURM_PROCID` | `DSTACK_NODE_RANK` | Process/node rank |
| `SLURM_NTASKS` | `DSTACK_GPUS_NUM` | Total number of processes/GPUs |
| `SLURM_JOB_NUM_NODES` | `DSTACK_NODES_NUM` | Number of nodes |
| Manual master address | `DSTACK_MASTER_NODE_IP` | Master node IP (automatically set) |
| N/A | `DSTACK_MPI_HOSTFILE` | Pre-populated MPI hostfile |

### Queueing and scheduling

Queueing and scheduling determine when and how jobs are executed. Both systems support scheduling large jobs and efficient resource utilization, with different implementation approaches.

**Slurm scheduling:**
- Multi-factor priority system with fairshare, age, and QOS-based scheduling
- Supports backfill scheduling (scheduling smaller jobs in gaps left by larger jobs)
- Supports job preemption (higher priority jobs can preempt lower priority ones)

**dstack scheduling:**
- Integer priority (0-100) with FIFO ordering within the same priority
- Automatic idle instance reuse
- No backfill or preemption (jobs run until completion)
- No fairshare, QOS, or usage quotas (priority is the only scheduling mechanism)
- Retry policy: Jobs can be automatically retried on no-capacity, failure, or interruption (see retry policy below). The no-capacity event is important for queueing on limited capacity.
- Topology-aware scheduling: dstack does not support automatic topology-aware scheduling (placing jobs on physically close nodes) except in specific cases: AWS fleets with cluster placement automatically configure EFA networking when supported, and GCP fleets automatically configure GPUDirect-TCPXO/GPUDirect-TCPX or RoCE networking for certain instance types. For other cases, manually configure network topology in scripts
- Heterogeneous jobs: dstack does not support heterogeneous jobs. All jobs within a run must use the same resource specification

#### Slurm queueing

Slurm uses a multi-factor priority system with fairshare, age, and QOS-based scheduling. Jobs are queued and scheduled based on these factors:

```bash
# Submit job
sbatch train.sh
# Submitted batch job 12345

# Check queue status
squeue -u $USER
# JOBID PARTITION     NAME     USER ST  TIME  NODES REASON
# 12345     gpu    training   user1 PD  0:00      2 Priority

# Check job details
scontrol show job 12345
# JobId=12345 JobName=training
# UserId=user1(1001) GroupId=users(100)
# Priority=4294 Reason=Priority (Resources)
```

**Slurm priority factors:**
- Fairshare (most common): prioritizes users who have been under-using resources
- Age: increases priority the longer a job waits
- Job size: can prioritize large or small jobs
- QOS: static boost based on Quality of Service

#### dstack queueing and retry policy

dstack uses an integer priority system (0-100) with FIFO ordering within the same priority. Jobs can be automatically retried on various failure conditions:

```bash
# Submit run
dstack apply -f train.dstack.yml
# Run submitted: training-job

# Check run status
dstack ps
# NAME          BACKEND  RESOURCES       PRICE    STATUS       SUBMITTED
# training-job  aws      H100:1 (spot)  $4.50    provisioning 2 mins ago
```

**dstack priority:**
- Integer 0-100 (set via `priority: 50` in configuration)
- FIFO ordering within same priority level

**dstack retry policy:**

dstack automatically retries runs when certain events occur, which affects how jobs are scheduled and how long they wait. The `on_events` parameter specifies which events trigger a retry:
- `no-capacity`: No instances available matching resource requirements (important for queueing on limited capacity). Use this when you want jobs to wait in queue until resources become available.
- `error`: Job failed with non-zero exit code. Use this for automatic retry on application failures (e.g., transient errors, OOM).
- `interruption`: Spot instance was terminated or instance was interrupted. Use this for fault tolerance with spot instances.

The `duration` parameter specifies the maximum time to keep retrying (e.g., `48h` means retry for up to 48 hours total). When a run is retried, it's resubmitted as a new run, which may affect wait times if resources are constrained.

This is similar to Slurm's `--requeue` behavior, but dstack provides more granular control over which events trigger retries and how long to keep retrying.

**Example with retry policy:**

```yaml
type: task
name: train-with-retry

python: 3.12
repos:
  - .

commands:
  - python train.py --batch-size=64

resources:
  gpu: 1
  memory: 32GB

retry:
  on_events: [no-capacity]
  duration: 48h

max_duration: 2h
```

### Partitions and fleets

Partitions in Slurm and fleets in dstack serve similar organizational purposes but operate differently:

- **What they are**: Slurm partitions are logical groupings of static nodes, while dstack fleets are physical pools of instances
- **How they differ**: Slurm partitions can overlap (a node can belong to multiple partitions), while dstack fleets are exclusive (each instance belongs to exactly one fleet)
- **dstack fleet types**: dstack supports both fixed pools/clusters (SSH fleets) and elastic ones with dynamic provisioning via GPU cloud or Kubernetes. Refer to [Backend configuration](#backend-configuration) for details
- **Function**: Partitions enforce limits and access control, while fleets define resource requirements and provisioning behavior

#### Slurm partition configuration

Slurm partitions are logical groupings of static nodes defined in `slurm.conf`. Nodes can belong to multiple partitions:

```bash
# In slurm.conf on controller node
PartitionName=gpu Nodes=gpu-node[01-10] Default=NO MaxTime=24:00:00
PartitionName=cpu Nodes=cpu-node[01-50] Default=YES MaxTime=72:00:00
PartitionName=debug Nodes=gpu-node[01-10] Default=NO MaxTime=1:00:00
```

**Submit to specific partition:**
```bash
sbatch --partition=gpu train.sh
```

**Submit to specific fleet in dstack:**
```bash
dstack apply -f train.dstack.yml --fleet gpu-fleet
```

#### dstack fleet configuration

dstack fleets are pools of instances (VMs or containers) that serve as both the organization unit and the provisioning template. Unlike Slurm's partitions (logical groupings of static nodes), fleets are dynamic pools that can span cloud providers or on-premises infrastructure. Each fleet defines resource requirements, and dstack automatically provisions matching instances when jobs need them.

dstack supports two types of fleets:

- **Backend fleets**: Dynamically provisioned through configured backends (cloud providers like AWS, GCP, Azure, or Kubernetes). You specify resource requirements and a `nodes` range; dstack provisions matching instances automatically.
- **SSH fleets**: Use existing on-premises servers. You list hostnames/IPs; dstack connects via SSH, installs dependencies, and registers them. SSH fleets are useful when you have existing infrastructure you want to use with dstack.

**Basic GPU fleet (backend fleet):**
```yaml
type: fleet
name: gpu-fleet

nodes: 0..10

resources:
  gpu: 40GB..80GB:1..8

backends: [aws, gcp]
```

**Fleet with cluster placement (for distributed training):**
```yaml
type: fleet
name: distributed-training-fleet

# Cluster placement ensures optimal inter-node connectivity
placement: cluster

nodes: 0..8

resources:
  gpu: A100:80GB:8

backends: [aws]

# Spot instances for cost savings
spot_policy: auto
```

**Fleet with blocks (for multi-tenant GPU sharing):**

Blocks allow you to split instance GPUs into smaller units for multi-tenant sharing. For example, an instance with 8 GPUs can be split into 8 blocks (1 GPU per block), allowing 8 concurrent single-GPU jobs to run on the same instance.

**Important limitations:**
- Blocks do NOT work with distributed tasks. Distributed tasks require full access to host resources, ports, InfiniBand networking, and direct GPU-to-GPU communication. When you specify `nodes: 2` or more in a task, dstack automatically uses the full instance without blocks.
- Blocks are only suitable for single-node, single-GPU or multi-GPU jobs that don't require inter-node communication.

```yaml
type: fleet
name: shared-gpu-fleet

nodes: 0..4

resources:
  gpu: A100:80GB:8

# Split 8 GPUs into 8 blocks (1 GPU per block)
# Allows 8 concurrent single-GPU jobs
# Note: Distributed tasks (nodes > 1) cannot use blocks
blocks: 8

backends: [aws]
```

**Fleet management commands:**
```bash
# Create/update fleet
dstack apply -f fleet.dstack.yml

# List fleets
dstack fleet

# List fleet instances with details
dstack fleet -v

# Delete fleet
dstack delete -f fleet.dstack.yml
```

**Fleet operation:**
- **Instance exclusivity**: Each instance belongs to exactly one fleet (unlike Slurm partitions where nodes can belong to multiple partitions). This ensures clear resource ownership and simplifies provisioning logic.
- **Automatic selection**: There is no default fleet requirement. dstack automatically selects a fleet based on resource requirements and cost optimization. You can explicitly specify a fleet using the `fleets` property in your task configuration.
- **Resource definition**: Fleets define both resource requirements (GPU type, memory, etc.) and provisioning behavior (spot policy, placement, blocks). When a job needs resources, dstack provisions instances from matching fleets.
- **Distributed training requirement**: Tasks with `nodes: 2` or more require a fleet with `placement: cluster` configured. This ensures instances are provisioned with optimal inter-node connectivity for high-bandwidth GPU communication. On AWS, EFA networking is automatically configured when supported. On GCP, GPUDirect-TCPXO/GPUDirect-TCPX or RoCE networking is automatically configured for certain instance types. On other backends, networking must be manually configured. Without `placement: cluster`, distributed training performance will be severely degraded.
- **Project isolation**: Fleets belong to projects and are not shared across projects. Sharing fleets across projects and managing quotas is not currently supported in dstack.

### Backend configuration

Backends enable provisioning of clusters and instances via integrations, including GPU clouds or Kubernetes, as opposed to static clusters (SSH fleets). Backends in dstack are configured at the server level (in `~/.dstack/server/config.yml` or via the UI), not in run configurations.

**VM-based backends:**
- Major cloud providers: AWS, GCP, Azure, OCI
- GPU cloud providers: Lambda, Nebius AI Cloud, Verda Cloud, dstack Sky
- Other providers: Vultr, Cudo, DigitalOcean, CloudRift, HotAisle, AMDDevCloud
- dstack provisions VMs and manages the entire lifecycle
- Supports blocks, instance volumes, privileged containers, and reusable instances

**Container-based backends:**
- Kubernetes: Orchestrates containers as Kubernetes pods, delegates node lifecycle to Kubernetes. Requires a proxy jump host for SSH access to containers.
- Marketplace providers: RunPod, Vast.ai, TensorDock
- Limitations: Blocks, instance volumes, and network volumes are not available for Kubernetes backend (Kubernetes native volumes can still be used). For other container-based backends, blocks and instance volumes are not available since dstack doesn't control the underlying nodes

**SSH fleets:**
- For on-premises servers or existing infrastructure
- Create SSH fleets that list hostnames/IPs; dstack connects via SSH, installs dependencies, and registers them
- Can use existing shared filesystems (NFS, Lustre, GPFS) already configured on the servers via instance volumes
- **When to use**: SSH fleets are ideal when you have existing on-premises infrastructure, want to use pre-mounted shared filesystems, or need to integrate with existing cluster management. Backend fleets are better for cloud-native deployments with dynamic provisioning.

**Hybrid deployments**: dstack makes it easy to use both SSH fleets and cloud backends together. Simply configure both in your project—dstack automatically selects the best fleet based on resource requirements and availability. This allows gradual migration from on-premises to cloud or running workloads across both environments.

**SSH fleet example:**
```yaml
type: fleet
name: on-prem-gpu-fleet

ssh_config:
  user: dstack
  identity_file: ~/.ssh/id_rsa
  hosts:
    - gpu-node01.example.com
    - gpu-node02.example.com
```

SSH fleets automatically detect resources on each host. Resources are not specified in the fleet configuration. SSH fleets support `proxy_jump` for accessing nodes via login/bastion hosts (similar to Slurm's login nodes).

**Example backend configuration** (`~/.dstack/server/config.yml`):
```yaml
projects:
  - name: main
    backends:
      # VM-based backends
      - type: aws
        creds:
          type: default  # Uses ~/.aws/credentials
        region: us-east-1
      - type: gcp
        creds:
          type: default  # Uses gcloud auth
        region: us-central1
      - type: lambda
        creds:
          type: api_key
          api_key: <lambda-api-key>
      # Container-based backend
      - type: kubernetes
        kubeconfig:
          filename: ~/.kube/config
        proxy_jump:
          hostname: <jump-host-ip>
          port: 32000
```

Backends are configured per project, allowing different projects to use different cloud providers or infrastructure. The server loads this configuration on startup, or you can configure backends via the project settings page in the UI.

### Kubernetes backend

The Kubernetes backend allows dstack to orchestrate containers as Kubernetes pods, delegating node lifecycle management to Kubernetes.

**Configuration requirements:**
- Requires `kubeconfig` file or in-cluster configuration
- Must specify `proxy_jump` port for SSH access (hostname can be auto-detected in most cases). The port must be accessible on the jump host - dstack will use it to proxy SSH traffic
- GPU access requires nodes with GPU support and proper device plugin configuration
- For distributed training, use `placement: cluster` in fleet configuration to ensure pods are scheduled on nodes with proper networking

**Example configuration:**
```yaml
type: kubernetes
kubeconfig:
  filename: ~/.kube/config
proxy_jump:
  hostname: <jump-host-ip>  # Node accessible to both server and CLI for SSH tunneling
  port: 32000
```

### Filesystems and data access

Data access patterns differ between Slurm and dstack:

- **Slurm**: Relies on shared filesystems (NFS, Lustre, GPFS) with a global namespace, meaning the same path exists on all nodes
- **dstack**: Uses explicit volume mounting—you must specify which volumes to mount and where. dstack supports shared filesystems via SSH fleets (using instance volumes with pre-mounted network storage). For backend fleets, mounting shared storage via instance volumes is not straightforward and may require dedicated endpoints. Volumes are the main interface
- **Slurm**: Provides `$SLURM_TMPDIR` for local scratch space (auto-cleaned)
- **dstack**: Uses instance volumes that persist on the instance. Instance volumes can leverage shared filesystems (NFS, SMB) when using SSH fleets where users control the instances and can pre-mount network storage. For backend fleets, mounting shared storage via instance volumes is not straightforward and may require dedicated endpoints

#### Slurm shared filesystem

Slurm assumes a shared filesystem (NFS, Lustre, GPFS) with a global namespace. The same path exists on all nodes, and `$SLURM_TMPDIR` provides local scratch space that is automatically cleaned:

```bash
#!/bin/bash
#SBATCH --nodes=4
#SBATCH --gres=gpu:8
#SBATCH --time=24:00:00

# Global namespace - same path on all nodes
# Dataset accessible at same path on all nodes
DATASET_PATH=/shared/datasets/imagenet

# Local scratch (faster I/O, auto-cleaned)
# Copy dataset to local SSD for faster access
cp -r $DATASET_PATH $SLURM_TMPDIR/dataset

# Training with local dataset
python train.py \
  --data=$SLURM_TMPDIR/dataset \
  --checkpoint-dir=/shared/checkpoints \
  --epochs=100

# $SLURM_TMPDIR automatically cleaned when job ends
# Checkpoints saved to shared filesystem persist
```

#### dstack volumes for datasets and checkpoints

dstack uses explicit volume mounting—you must specify which volumes to mount and where. dstack supports two types of volumes:

**Network volumes**: Persistent storage (AWS EBS, GCP persistent disks, RunPod volumes) that persists across runs. Network volumes are shared across all nodes in a multi-node job, but have limitations:
- Network latency for I/O operations
- May not be available in all regions or zones
- Can use interpolation to select from multiple volumes: `- name: dataset-${{ dstack.node_rank }}` (selects volume based on node rank)
- **Backend support**: Network volumes are currently supported for the `aws`, `gcp`, and `runpod` backends

**Instance volumes**: Local storage on the instance that persists for the lifetime of the instance. Instance volumes are faster than network volumes but are tied to the instance lifecycle.

**Shared filesystems**: 

- **SSH fleets**: Support shared filesystems (NFS, Lustre, GPFS) via instance volumes with pre-mounted network storage.
- **Backend fleets**: Currently, there is no way to mount shared filesystems with backend fleets.
- **Network volumes**: Cannot be used as shared filesystems (they are single-attach per backend, except RunPod).
- **Data sharing**: To share data between multiple runs, use network volumes with interpolation (`- name: dataset-${{ dstack.node_rank }}`) or instance volumes with manually mounted shared filesystems on SSH fleets.

**Create network volume for datasets:**

Create `dataset-volume.dstack.yml`:
```yaml
type: volume
name: imagenet-dataset
backend: aws
region: us-east-1
size: 500GB
```

Then apply:
```bash
dstack apply -f dataset-volume.dstack.yml
```

**Network volumes example:**

```yaml
type: task
name: train-with-network-volumes

python: 3.12
repos:
  - .

volumes:
  - name: imagenet-dataset
    path: /data/imagenet

commands:
  - python train.py --data=/data/imagenet --epochs=100

resources:
  gpu: 1
  memory: 32GB
```

**Instance volumes example (backed by instance's local storage):**

```yaml
type: task
name: train-with-instance-volumes

python: 3.12
repos:
  - .

volumes:
  # Instance volume (faster I/O, backed by instance's local storage)
  - name: checkpoint-cache
    path: /checkpoints
    instance: true

commands:
  - python train.py --checkpoint-dir=/checkpoints --epochs=100

resources:
  gpu: 1
  memory: 32GB
```

**Multi-node example with network and instance volumes:**

```yaml
type: task
name: distributed-train-with-volumes

nodes: 4

python: 3.12
repos:
  - .

volumes:
  # Network volume (shared across nodes, persistent)
  - name: imagenet-dataset
    path: /data/imagenet
  # Instance volume (per-node, faster I/O)
  - name: checkpoint-cache
    path: /checkpoints
    instance: true

commands:
  - |
    torchrun \
      --nproc-per-node=$DSTACK_GPUS_PER_NODE \
      --node-rank=$DSTACK_NODE_RANK \
      --nnodes=$DSTACK_NODES_NUM \
      --master-addr=$DSTACK_MASTER_NODE_IP \
      --master-port=12345 \
      train.py \
      --data=/data/imagenet \
      --checkpoint-dir=/checkpoints \
      --epochs=100

resources:
  gpu: A100:80GB:8
  memory: 200GB..
  shm_size: 24GB
```

**Volume management commands:**
```bash
# List volumes
dstack volume list

# Create volume
dstack apply -f dataset-volume.dstack.yml

# Delete volume
dstack delete -f dataset-volume.dstack.yml
```

**dstack repos and files:**

In addition to volumes, dstack provides two mechanisms for bringing code and data into containers:

- **Repos**: The `repos` property allows cloning Git repositories into the container. dstack clones the repo on the instance, applies local changes, and mounts it into the container. This is useful for code that needs to be version-controlled and synced.
- **Files**: The `files` property allows mounting local files or directories into the container. Each entry maps a local path to a container path. Files are uploaded to the instance and mounted into the container, but are not persisted across runs (2MB limit per file, configurable).

**Example with repos and files:**
```yaml
type: task
name: train-with-code

python: 3.12

# Clone Git repository
repos:
  - .  # Clone current directory repo

# Mount local files
files:
  - ../configs:~/configs
  - ~/.ssh/id_rsa:~/ssh/id_rsa

commands:
  - python train.py --config ~/configs/model.yaml

resources:
  gpu: A100:80GB:1
```

### Interactive sessions and development

Interactive sessions enable users to allocate resources for experimentation and development:

- **Slurm approach**: Uses `salloc` to allocate resources with a time limit, after which resources are automatically released
- **dstack approach**: Uses `dev-environment` configurations that run until explicitly stopped, with optional inactivity-based termination
- **Access methods**: Slurm requires SSH access to allocated nodes, while dstack provides VS Code in the browser or SSH access

#### Slurm interactive allocation

Slurm uses `salloc` to allocate resources with a time limit. After the time limit expires, resources are automatically released:

```bash
# Allocate resources and get shell on login node
salloc --nodes=1 --gres=gpu:1 --time=4:00:00

# Run commands on allocated nodes
srun hostname
srun python experiment.py

# Or SSH to allocated node
ssh $SLURM_NODELIST

# Work interactively
python train.py --epochs=1  # Quick test
python evaluate.py --checkpoint=best.pt
```

#### dstack dev environment

dstack uses `dev-environment` configurations that run until explicitly stopped, with optional inactivity-based termination. Access is provided via VS Code in the browser or SSH:

**Basic dev environment:**
```yaml
type: dev-environment
name: ml-dev

python: 3.12
ide: vscode

resources:
  gpu: A100:80GB:1
  memory: 200GB

# Instance stays running until explicitly stopped
# Access via VS Code in browser or SSH
```

**Dev environment commands:**
```bash
# Start dev environment (automatically attaches unless --detach is used)
dstack apply -f dev.dstack.yml

 #  BACKEND  REGION    RESOURCES                SPOT  PRICE
 1  runpod   CA-MTL-1  9xCPU, 48GB, A5000:24GB  yes   $0.11

Submit the run ml-dev? [y/n]: y

Launching `ml-dev`...
---> 100%

To open in VS Code Desktop, use this link:
  vscode://vscode-remote/ssh-remote+ml-dev/workflow
```

### Job arrays and hyperparameter sweeps

Hyperparameter sweeps enable testing multiple configurations in parallel. Slurm provides native job arrays (`--array=1-100`) that create multiple job tasks from a single submission, while dstack requires submitting multiple runs programmatically or using external workflow orchestration tools. Slurm job arrays provide built-in task ID management and throttling, while dstack runs are independent and must be managed externally.

#### Slurm job array for hyperparameter sweep

Slurm provides native job arrays (`--array=1-100`) that create multiple job tasks from a single submission. Each task uses `$SLURM_ARRAY_TASK_ID` to determine its configuration:

```bash
#!/bin/bash
#SBATCH --job-name=hyperparameter-sweep
#SBATCH --array=1-100
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --mem=32G
#SBATCH --time=4:00:00
#SBATCH --partition=gpu

# Each task uses different hyperparameters
TASK_ID=$SLURM_ARRAY_TASK_ID

# Generate hyperparameters based on task ID
LEARNING_RATE=$(python -c "print(0.001 * (1 + $TASK_ID * 0.1))")
BATCH_SIZE=$((32 + $TASK_ID % 64))
WEIGHT_DECAY=$(python -c "print(0.0001 * (1 + $TASK_ID * 0.01))")

srun python train.py \
  --learning-rate=$LEARNING_RATE \
  --batch-size=$BATCH_SIZE \
  --weight-decay=$WEIGHT_DECAY \
  --output-dir=/shared/results/sweep_${TASK_ID}
```

**Submit and monitor:**
```bash
sbatch array_job.sh
# Submitted batch job 1001

squeue -j 1001
# Shows 1001_1, 1001_2, ..., 1001_100

# Cancel specific array tasks
scancel 1001_[50-100]  # Cancel tasks 50-100
```

#### dstack hyperparameter sweep

dstack does not support native job arrays. Submit multiple runs programmatically or use external workflow orchestration tools (Airflow, Prefect, W&B sweeps, etc.):

```bash
# Submit multiple runs programmatically (use --detach to submit without waiting)
for i in {1..100}; do
  dstack apply -f train.dstack.yml \
    --name "hp-sweep-${i}" \
    --env CONFIG_ID=${i} \
    --detach
done
```

### Job dependencies

Job dependencies enable chaining tasks together, ensuring that downstream jobs only run after upstream jobs complete. Slurm provides native dependency support via `--dependency` flags (`afterok`, `afterany`, `singleton`), while dstack requires using external workflow orchestration tools (Airflow, Prefect, etc.) to implement dependencies. Slurm dependencies are managed by the scheduler, while dstack dependencies must be implemented in the orchestration layer.

#### Slurm dependencies

Slurm provides native dependency support via `--dependency` flags. Dependencies are managed by the scheduler:

```bash
# Submit training job
JOB_TRAIN=$(sbatch train.sh | awk '{print $4}')

# Submit evaluation that depends on training (only if training succeeds)
sbatch --dependency=afterok:$JOB_TRAIN evaluate.sh

# Singleton dependency (only one job with this name runs at a time)
sbatch --job-name=ModelTraining --dependency=singleton train.sh
```

#### dstack external workflows

dstack does not support native job dependencies. Use external workflow orchestration tools (Airflow, Prefect, etc.) to implement dependencies. Since `dstack apply` waits for completion by default, tasks will automatically wait for each run to finish. The following example uses Prefect:

```python
# Using Prefect to orchestrate dstack runs
from prefect import flow, task
import subprocess

@task
def train_model():
    """Submit training job and wait for completion"""
    subprocess.run(
        ["dstack", "apply", "-f", "train.dstack.yml", "--name", "train-run"],
        check=True  # Raises exception if training fails
    )
    return "train-run"

@task
def evaluate_model(run_name):
    """Submit evaluation job after training succeeds"""
    subprocess.run(
        ["dstack", "apply", "-f", "evaluate.dstack.yml", "--name", f"eval-{run_name}"],
        check=True
    )

@flow
def ml_pipeline():
    train_run = train_model()
    evaluate_model(train_run)
```

**Using Airflow to chain tasks:**

Airflow provides a declarative way to define task dependencies, similar to Slurm's `--dependency` flag. Since `dstack apply` waits for completion by default, tasks will automatically wait for each run to finish:

```python
from airflow.decorators import dag, task
from datetime import datetime
import subprocess

@dag(schedule=None, start_date=datetime(2024, 1, 1), catchup=False)
def ml_training_pipeline():
    @task
    def train(context):
        """Submit training job and wait for completion"""
        run_name = f"train-{context['ds']}"
        subprocess.run(
            ["dstack", "apply", "-f", "train.dstack.yml", "--name", run_name],
            check=True  # Raises exception if training fails
        )
        return run_name
    
    @task
    def evaluate(run_name, context):
        """Submit evaluation job after training succeeds"""
        eval_name = f"eval-{run_name}"
        subprocess.run(
            ["dstack", "apply", "-f", "evaluate.dstack.yml", "--name", eval_name],
            check=True
        )
    
    # Define task dependencies - train() completes before evaluate() starts
    train_run = train()
    evaluate(train_run)

ml_training_pipeline()
```

### Environment variables and secrets

Both systems need to handle sensitive data (API keys, tokens, passwords) for ML workloads. Slurm uses environment variables or files for secrets, while dstack provides a secrets management system with encryption that securely stores and injects secrets into jobs.

#### Slurm environment variables

Slurm uses OS-level authentication and environment variables. Jobs run with the user's UID/GID and inherit the environment from the login node:

```bash
# Slurm uses OS-level authentication
# Users authenticate via SSH to login nodes
# Jobs run with user's UID/GID

# No built-in secrets management
# Users manage credentials in their environment or shared files

# Example: Using environment variables from user's shell
export HF_TOKEN=$(cat ~/.hf_token)
sbatch train.sh  # Job inherits environment

# Or in job script
#!/bin/bash
#SBATCH --job-name=train
source ~/.secrets  # Load secrets from file
python train.py --token=$HF_TOKEN
```

#### dstack secrets management

dstack provides a secrets management system with encryption that securely stores and injects secrets into jobs. Secrets are referenced in configuration using `${{ secrets.name }}` syntax:

**Set secrets:**
```bash
# Set Hugging Face token
dstack secret set huggingface_token <token>

# Set Weights & Biases API key
dstack secret set wandb_api_key <key>

# Set AWS credentials
dstack secret set aws_access_key_id <key>
dstack secret set aws_secret_access_key <secret>
```

**Use secrets in configuration:**
```yaml
type: task
name: train-with-secrets

python: 3.12
repos:
  - .

env:
  # Reference secrets using ${{ secrets.name }} syntax
  - HF_TOKEN=${{ secrets.huggingface_token }}
  - WANDB_API_KEY=${{ secrets.wandb_api_key }}
  - AWS_ACCESS_KEY_ID=${{ secrets.aws_access_key_id }}
  - AWS_SECRET_ACCESS_KEY=${{ secrets.aws_secret_access_key }}

commands:
  - |
    # Download private model from Hugging Face
    huggingface-cli download meta-llama/Llama-2-7b-hf
    
    # Initialize Weights & Biases
    wandb login
    
    # Train with logging
    python train.py \
      --model-path=./meta-llama/Llama-2-7b-hf \
      --wandb-project=llama-training

resources:
  gpu: A100:80GB:8
  memory: 200GB..
```

**Private Docker registry authentication:**
```yaml
type: task
name: train-with-private-image

image: nvcr.io/nvidia/pytorch:24.01-py3

registry_auth:
  username: $oauthtoken
  password: ${{ secrets.nvidia_ngc_api_key }}

commands:
  - python train.py

resources:
  gpu: A100:80GB:8
```

### Authentication

Both systems handle user authentication differently.

#### Slurm authentication

Slurm uses OS-level authentication for users. Users authenticate via SSH to login nodes using their Unix accounts, and jobs run with the user's UID/GID. MUNGE is used for inter-daemon authentication (between `slurmctld`, `slurmd`, `slurmdbd`), not for user authentication.

#### dstack authentication

dstack uses token-based authentication for CLI/API access. Users authenticate with tokens issued by the server, and access is controlled at the project level with user roles (Admin, Manager, User). No SSH access to login nodes is required.

### Monitoring and observability

Monitoring provides visibility into job performance, resource usage, and system health:

- **Slurm**: Provides command-line tools (`squeue`, `sstat`, `sacct`) and requires manual SSH access to nodes for detailed metrics. GPU utilization must be checked manually via `nvidia-smi` on each node
- **dstack**: Provides automatic GPU metrics collection via DCGM, real-time monitoring via CLI, and can export metrics to Prometheus. GPU metrics are collected automatically for all GPUs
- **Logs**: Slurm logs are written to files on shared filesystems (specified via `--output` and `--error`), while dstack streams logs via API and stores them locally or in external services (CloudWatch, GCP Logging)
- **Multi-node monitoring**: Slurm requires checking each node separately via SSH, while dstack provides a unified view across all nodes with a single command

#### Slurm monitoring

Slurm provides command-line tools for monitoring jobs and nodes. GPU utilization must be checked manually via `nvidia-smi` on each node:

```bash
# Check node status
sinfo
# PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
# gpu          up  1-00:00:00     10   idle gpu-node[01-10]

# Check job queue
squeue -u $USER
# JOBID PARTITION     NAME     USER ST  TIME  NODES
# 12345     gpu    training   user1  R  2:30      2

# Check job details
scontrol show job 12345
# JobId=12345 JobName=training
# UserId=user1(1001) GroupId=users(100)
# NumNodes=2 NumCPUs=64 NumTasks=32
# Gres=gpu:8(IDX:0,1,2,3,4,5,6,7)

# Check resource usage (running jobs only)
sstat --job=12345 --format=JobID,MaxRSS,MaxVMSize,CPUUtil
#       JobID     MaxRSS  MaxVMSize   CPUUtil
# 12345.0        2048M     4096M      95.2%

# Check GPU usage (requires SSH to node)
srun --jobid=12345 --pty nvidia-smi
# GPU 0: 95% utilization, 72GB/80GB memory

# Check job history
sacct --job=12345 --format=JobID,Elapsed,MaxRSS,State,ExitCode
#       JobID    Elapsed     MaxRSS      State ExitCode
# 12345     2:30:00     2048M  COMPLETED      0:0
```

#### dstack monitoring

dstack provides automatic GPU metrics collection via DCGM, real-time monitoring via CLI, and can export metrics to Prometheus. GPU metrics are collected automatically for all GPUs:

```bash
# List runs
dstack ps
# NAME          BACKEND  RESOURCES       PRICE    STATUS       SUBMITTED
# training-job  aws      H100:1 (spot)  $4.50    running      5 mins ago

# List fleets and instances
dstack fleet
# FLEET     INSTANCE  BACKEND          RESOURCES  STATUS          PRICE
# my-fleet  0         aws (us-east-1)  T4:16GB:1  idle            $0.526

# Check real-time metrics
dstack metrics training-job
# NAME             STATUS  CPU  MEMORY          GPU
# training-job     running 45%  16.27GB/200GB   gpu=0 mem=72.48GB/80GB util=95%

# Stream logs
dstack logs training-job
```

**Prometheus integration:**

dstack can export metrics to Prometheus when enabled via the `DSTACK_ENABLE_PROMETHEUS_METRICS` environment variable. This provides access to advanced DCGM telemetry (GPU utilization, memory, temperature, ECC errors, PCIe replay counters, NVLink errors) in addition to the essential metrics available via CLI. Configure Prometheus to scrape metrics from `<dstack server URL>/metrics`.

### Fault tolerance, checkpointing, and retry

Fault tolerance is essential for long-running training jobs that may be interrupted by hardware failures, spot instance terminations, or other issues. Slurm provides `--requeue` to automatically requeue jobs on failure and signal-based checkpointing. dstack provides automatic retry policies that can restart jobs on various failure conditions (no-capacity, error, interruption) and supports checkpointing via volumes.

#### Slurm checkpointing and requeue

Slurm provides `--requeue` to automatically requeue jobs on failure and signal-based checkpointing. Applications can trap signals to save checkpoints before preemption:

```bash
#!/bin/bash
#SBATCH --job-name=train-with-checkpoint
#SBATCH --nodes=4
#SBATCH --gres=gpu:8
#SBATCH --mem=200G
#SBATCH --time=48:00:00
#SBATCH --requeue  # Enable automatic requeue on node failure
#SBATCH --signal=B:USR1@300  # Send USR1 signal 5 minutes before time limit

# Trap signal for checkpointing before preemption
trap 'echo "Checkpointing..."; python save_checkpoint.py --checkpoint-dir=/shared/checkpoints' USR1

# Resume from checkpoint if exists
if [ -f /shared/checkpoints/latest.pt ]; then
    RESUME_FLAG="--resume /shared/checkpoints/latest.pt"
fi

srun python train.py \
  --checkpoint-dir=/shared/checkpoints \
  --checkpoint-interval=1000 \
  $RESUME_FLAG
```

#### dstack checkpointing and retry

dstack provides automatic retry policies that can restart jobs on various failure conditions (no-capacity, error, interruption). Checkpoints are saved to volumes and can be resumed automatically:

```yaml
type: task
name: train-with-checkpoint-retry

nodes: 4

python: 3.12
repos:
  - .

volumes:
  # Persistent checkpoint storage
  - name: checkpoint-volume
    path: /checkpoints

env:
  - WANDB_API_KEY=${{ secrets.wandb_api_key }}

commands:
  - |
    # Resume from checkpoint if exists
    if [ -f /checkpoints/latest.pt ]; then
      RESUME_FLAG="--resume /checkpoints/latest.pt"
    fi
    
    python train.py \
      --checkpoint-dir=/checkpoints \
      --checkpoint-interval=1000 \
      $RESUME_FLAG

resources:
  gpu: A100:80GB:8
  memory: 200GB..
  shm_size: 24GB

spot_policy: auto  # Prefer spot instances for cost savings

retry:
  on_events: [no-capacity, error, interruption]
  duration: 48h  # Retry for up to 48 hours

max_duration: 48h
```

Training scripts should check for existing checkpoints on startup and resume from the last checkpoint when retried after failure or interruption.

### GPU health monitoring

GPU health monitoring is critical for maintaining reliable infrastructure, especially when using spot instances or long-running training jobs.

#### Slurm GPU health checks

Slurm requires custom scripts (using DCGM or RVS) that periodically check nodes and drain them if issues are detected. Health checks are configured in `slurm.conf`:

```bash
# Health check script (configured in slurm.conf)
#!/bin/bash
# HealthCheckProgram=/shared/scripts/gpu_health_check.sh

# Run DCGM diagnostic
dcgmi diag -r 1

if [ $? -ne 0 ]; then
    echo "DCGM diagnostic failed" >&2
    exit 1  # Non-zero exit drains node
fi
```

#### dstack GPU health monitoring

dstack automatically monitors GPU health using DCGM (Data Center GPU Manager) on instances with NVIDIA GPUs. The monitoring is continuous and automatic, detecting:
- **Correctable ECC errors**: Memory errors that are automatically corrected but may indicate degrading hardware
- **Uncorrectable ECC errors**: Fatal memory errors that can cause data corruption
- **PCIe errors**: Communication errors between GPU and CPU
- **Thermal issues**: Overheating that can cause throttling or shutdown
- **NVLink errors**: Inter-GPU communication failures (needed for multi-GPU workloads)

**Health status and actions:**

```bash
# GPU health automatically detected via DCGM
dstack fleet
# FLEET     INSTANCE  BACKEND          RESOURCES  STATUS          PRICE
# my-fleet  0         aws (us-east-1)  T4:16GB:1  idle            $0.526
#           1         aws (us-east-1)  T4:16GB:1  idle (warning)  $0.526  # Correctable ECC errors
#           2         aws (us-east-1)  T4:16GB:1  idle (failure)  $0.526  # Fatal issues, excluded
```

**Health status definitions:**
- **`idle`**: Healthy GPU, ready for workloads. No issues detected.
- **`idle (warning)`**: Non-fatal issues detected (e.g., correctable ECC errors). The instance is still usable but should be monitored. Jobs can still be scheduled on it.
- **`idle (failure)`**: Fatal issues detected (uncorrectable ECC, PCIe failures, etc.). The instance is automatically excluded from scheduling. No new jobs will be placed on it until the issue is resolved.

**DCGM for SSH fleets**: Install DCGM packages on SSH fleet hosts for automatic GPU health monitoring.

## Detailed guides and comparisons

For more detailed information on specific topics, see the comprehensive guides in the `concepts/` folder:

- **[Control plane, state, and scaling](concepts/01_control-plane.md)**: Architecture, state persistence, high availability, and scaling
- **[Queueing, prioritization, and scheduling](concepts/02_queueing-prioritization-scheduling.md)**: Scheduling mechanics, priority systems, and queueing behavior
- **[Resource model, generic resources, and enforcement](concepts/03_resource-model.md)**: Resource specification, GPU allocation, and enforcement mechanisms
- **[Job submission, allocation, and execution](concepts/04_job-execution-lifecycle.md)**: Job lifecycle, environment variables, and execution mechanics
- **[Accounts, QOS, and accounting](concepts/05_policy-accounting.md)**: User accounts, quality of service, and accounting systems
- **[Job arrays and dependencies](concepts/06_job-organization.md)**: Job arrays, dependencies, and workflow orchestration
- **[Cluster node management, health, and lifecycle](concepts/07_cluster-node-management.md)**: Fleet management, instance states, and health monitoring
- **[Filesystems and data access](concepts/08_file-systems.md)**: Volume management, shared filesystems, and data access patterns
- **[Fault tolerance, checkpointing, and job recovery](concepts/09_fault-tolerance.md)**: Retry policies, checkpointing, and failure handling
- **[GPU health monitoring and device constraints](concepts/10_gpu-health-monitoring.md)**: DCGM integration, health checks, and GPU diagnostics
- **[Partition design, node grouping, and queue layout](concepts/11_partition-design.md)**: Partition concepts and fleet organization
- **[Authentication and security](concepts/12_authentication-security.md)**: User authentication, secrets management, and security mechanisms
- **[Reservations](concepts/13_reservations.md)**: Resource reservations and capacity management
- **[Monitoring and observability](concepts/14_monitoring-observability.md)**: Metrics, logging, and monitoring tools
- **[Kubernetes integration](concepts/15_kubernetes.md)**: Kubernetes backend configuration and limitations

