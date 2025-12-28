# 4 Job submission, allocation, and execution

## Slurm

### Job submission and definition

Jobs are submitted using `sbatch` (batch), `srun` (step), or `salloc` (allocation). All three commands are invoked from **login nodes** (user access servers where users SSH to interact with the cluster; see [Control plane, state, and scaling](01_control-plane.md) for details on cluster infrastructure), but they differ in execution model and use cases. Jobs execute on **compute nodes**, not login nodes.

- **sbatch (Batch script)**: Submits a script file containing `#SBATCH` directives to the queue. The script executes on compute nodes when resources become available. Use for production jobs that don't require interaction.
- **srun (Process launcher)**: Launches application tasks across allocated nodes. As a standalone command, it blocks until resources are allocated and the command completes. Can also be used inside `sbatch` scripts. Use for quick tests or immediate execution.
- **salloc (Interactive allocation)**: Allocates resources and returns a shell on the login node with environment variables set. You must use `srun` or `ssh` to access compute nodes. Use for interactive sessions where you need to run multiple commands.

**Examples:**

**sbatch** (from login node):
```bash
# Create and submit job script
cat > train.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=train
#SBATCH --nodes=2
#SBATCH --ntasks=16
#SBATCH --time=2:00:00
#SBATCH --output=train-%j.out

srun python train.py
EOF

sbatch train.sh  # Job queued, returns immediately
```

**srun** (from login node):
```bash
# Runs immediately, blocks until completion
srun --nodes=1 --ntasks=4 --gres=gpu:1 --time=10:00 python test_model.py
```

**salloc** (from login node):
```bash
# Allocates resources, returns shell on login node
salloc --nodes=1 --ntasks=8 --gres=gpu:2 --time=1:00:00

# Run commands in allocation
srun hostname  # Executes on compute node
srun python train.py
srun python evaluate.py

# Or SSH to allocated node
ssh $SLURM_NODELIST

exit  # Releases allocation
```

### Interactive jobs and sessions

Interactive jobs allow users to run commands directly on compute nodes with real-time interaction.

- **`salloc` mechanics**: `salloc` creates an allocation and returns a shell on the login node with Slurm environment variables set. The allocation is active, but you must use `srun` or SSH to access the compute nodes.
- **`srun --pty`**: The `--pty` flag allocates a pseudo-terminal, allowing interactive commands. This provides an interactive shell on the first allocated node.
- **X11 forwarding**: X11 forwarding can be enabled for GUI applications, forwarding X11 to the first allocated node or all allocated nodes. Requires X11 forwarding configured in SSH.
- **SSH access**: From an `salloc` session, you can SSH directly to allocated nodes using the hostnames in `SLURM_NODELIST`.

### Job allocation mechanics

An "allocation" is the reservation of specific hardware (Nodes, CPUs, Memory) for a job.

- **Creation**: An allocation is created when a job transitions from `PENDING` to `RUNNING`.
- **Scope**: The allocation acts as a container. Within one allocation (e.g., 4 nodes for 2 hours), a user can run multiple "Job Steps" (e.g., run a data prep step on 1 node, then an MPI run on 4 nodes).
- **Structure**: The `slurmctld` records the node list and resource limits. The nodes are marked as `ALLOCATED` or `MIXED` state.

### Task launch (`slurmstepd`)

While `slurmctld` manages the cluster, `slurmstepd` is the daemon that manages the *process*.

- **Role**: A `slurmstepd` instance spawns on every allocated compute node. It is the parent process of the user's application.
- **Duties**: It launches the task, connects standard I/O (stdout/stderr) back to the user, and enforces resource limits.
- **Signal handling**: `slurmstepd` acts as the signal proxy. If a user cancels a job (`scancel`), `slurmstepd` receives the instruction and sends `SIGTERM`/`SIGKILL` to the application processes locally.
- **Step completion**: When the application exits, `slurmstepd` ensures all child processes are dead before reporting "Step Complete" to the controller.

### Job steps and task distribution

A single job allocation can contain multiple job steps, each running different commands or applications.

- **Job steps**: Within one allocation, you can run multiple steps sequentially or in parallel. Each step is identified by `SLURM_STEP_ID`. Example: Step 0 might prepare data, Step 1 runs the main computation, Step 2 post-processes results.
- **Allocation vs step resources**: The `#SBATCH` directives define the **allocation** (total resources reserved). Within that allocation, `srun` commands can use all or a subset of those resources. Each `srun` creates a new job step that consumes resources from the allocation.
- **Step resource specification**: `srun` can override allocation parameters for that specific step. If the allocation has 4 nodes but a step only needs 1 node, specify `srun --nodes=1` to use only 1 node for that step.
- **Example: multiple steps in one job** (script on login node, all steps execute on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --nodes=4          # Allocation: 4 nodes reserved
  #SBATCH --ntasks=32        # Allocation: 32 tasks available
  
  # Step 0: Data preparation (uses 1 node from allocation)
  srun --ntasks=1 --nodes=1 python prepare_data.py
  
  # Step 1: Main computation (uses all 4 nodes from allocation)
  srun --ntasks=32 --nodes=4 python train.py
  
  # Step 2: Post-processing (uses 1 node from allocation)
  srun --ntasks=1 --nodes=1 python postprocess.py
  ```
- **Resource inheritance**: If `srun` doesn't specify resource parameters, it inherits from the allocation. For example, if allocation has `--ntasks=32`, an `srun` without `--ntasks` will use all 32 tasks.
- **Task distribution**: Tasks are distributed evenly across allocated nodes. If `--ntasks=32` and `--nodes=4`, Slurm distributes 8 tasks per node. Use `--ntasks-per-node` to override this distribution.
- **MPI integration**: Slurm's `srun` is the recommended MPI launcher (replaces `mpirun`). It sets `SLURM_PROCID` to match MPI rank. Use `srun mpirun` only if your MPI implementation requires it. Slurm handles rank mapping automatically.
- **OpenMP/threading**: For threaded applications, use `--cpus-per-task` to allocate CPUs per task. CPU binding can be configured to bind threads to specific CPU cores. The `OMP_NUM_THREADS` environment variable should match `--cpus-per-task`.
- **Task binding**: CPU affinity is controlled by `--cpu-bind` with options to bind to cores, threads, sockets, or no binding.

### Job scripts and environment

#### Job script structure

A job script is a shell script (typically Bash) that contains both Slurm directives and executable commands.

- **Script format**: Job scripts start with a shebang line (e.g., `#!/bin/bash`) followed by `#SBATCH` directives, then the actual commands to execute.
- **Directive placement**: `#SBATCH` directives must appear at the top of the script, before any executable commands. They can be interspersed with comments, but must come before the first non-comment, non-directive line.
- **Script execution**: When the job starts, the script executes on the first allocated node (the "batch" node). The script runs with the user's permissions and environment.
- **Example structure** (script created on login node, executed on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=myjob
  #SBATCH --nodes=2
  #SBATCH --ntasks=32
  #SBATCH --time=1:00:00
  
  # Your commands here
  module load python/3.9
  srun python training_script.py
  ```
  
- **Submission** (from login node):
  ```bash
  # On login node
  sbatch myjob.sh
  # Submitted batch job 12345
  ```

#### Resource specification fundamentals

Understanding how Slurm allocates resources requires understanding the relationship between nodes, tasks, CPUs, and memory.

**Core concepts:**
- **Node**: A physical or virtual compute machine in the cluster. Each node has CPUs, memory, and optionally GPUs.
- **Task**: A process (program instance) that Slurm launches. For MPI jobs, each task is one MPI rank. For multi-threaded jobs, one task may spawn multiple threads.
- **CPU**: A CPU core (or hardware thread) that can execute instructions. Tasks consume CPUs.
- **Memory**: RAM allocated to the job, enforced via cgroups.

**Resource specification directives:**
- **`--nodes=N`**: Requests N compute nodes. The job will run on exactly N nodes (or fail if not available).
- **`--ntasks=M`**: Requests M tasks (processes) total across all allocated nodes. Tasks are distributed evenly across nodes unless `--ntasks-per-node` is specified.
- **`--cpus-per-task=C`**: Allocates C CPUs per task. If you request 32 tasks with 4 CPUs each, you need 128 CPUs total.
- **`--mem=M`**: Requests M memory per node (e.g., `--mem=16G` requests 16GB per node). Alternative: `--mem-per-cpu=M` requests memory per CPU.

**How they interact:**
- **Total CPUs needed**: If `--cpus-per-task` is specified, Total CPUs = `--ntasks` × `--cpus-per-task`. If `--cpus-per-task` is not specified, each task gets 1 CPU by default, so Total CPUs = `--ntasks`.
- **CPUs per node** = Total CPUs ÷ `--nodes` (must not exceed node capacity)
- **Memory per node** = `--mem` value (or `--mem-per-cpu` × CPUs per node)

**What `--nodes=2 --ntasks=16` means in HPC/AI/GPU context:**

This creates a **distributed parallel job** across 2 compute nodes with 16 processes total (8 per node).

- **MPI context**: 16 MPI ranks total. Rank 0-7 on node 1, ranks 8-15 on node 2. The MPI communicator spans both nodes, enabling inter-node communication.
- **Distributed ML training**: 16 training processes (e.g., PyTorch DDP, Horovod). Processes on the same node can use shared memory; inter-node communication uses the network.
- **GPU context**: If `--gres=gpu:4` is specified, each node gets 4 GPUs. Slurm sets `CUDA_VISIBLE_DEVICES` so processes only see allocated GPUs. The application framework (PyTorch DDP, etc.) handles process-to-GPU mapping.

**Example: Distributed training with `--nodes=2 --ntasks=16`** (script on login node, executes on compute nodes):
```bash
#!/bin/bash
#SBATCH --nodes=2          # 2 compute nodes
#SBATCH --ntasks=16        # 16 training processes (8 per node)
#SBATCH --gres=gpu:4       # 4 GPUs per node (8 GPUs total)
#SBATCH --cpus-per-task=4  # 4 CPUs per process

# This creates:
# - Node 1: 8 processes, 4 GPUs, 32 CPUs
# - Node 2: 8 processes, 4 GPUs, 32 CPUs
# - Total: 16 processes, 8 GPUs, 64 CPUs
# Slurm allocates 4 GPUs per node, sets CUDA_VISIBLE_DEVICES
# Application framework handles process-to-GPU mapping

srun python distributed_train.py
```

**Example: MPI job** (script on login node, executes on compute nodes):
```bash
#!/bin/bash
#SBATCH --nodes=4          # 4 nodes
#SBATCH --ntasks=32        # 32 MPI processes (ranks) total, 8 per node
#SBATCH --cpus-per-task=2  # 2 CPUs per process
#SBATCH --mem=8G           # 8GB RAM per node

# 32 MPI ranks total, 8 per node
# MPI_COMM_WORLD spans all 4 nodes

srun python mpi_program.py
```

**Example: threaded job** (script on login node, executes on compute nodes):
```bash
#!/bin/bash
#SBATCH --nodes=1           # 1 node
#SBATCH --ntasks=1          # 1 task (one process)
#SBATCH --cpus-per-task=16  # 16 CPUs for that process
#SBATCH --mem=32G           # 32GB RAM

export OMP_NUM_THREADS=16
srun python threaded_program.py
```

**Common patterns:**
- **MPI jobs**: `--nodes=4 --ntasks=32` (32 processes across 4 nodes, 8 per node)
- **Threaded jobs**: `--nodes=1 --ntasks=1 --cpus-per-task=16` (1 process, 16 threads)
- **Hybrid MPI+OpenMP**: `--nodes=4 --ntasks=8 --cpus-per-task=4` (8 MPI processes, each with 4 threads = 32 total threads)
- **Single-node multi-core**: `--nodes=1 --ntasks=8` (8 processes on 1 node)

**Location context**: These directives are specified in job scripts created on login nodes. Slurm interprets them during job submission and allocates resources on compute nodes accordingly.

#### #SBATCH directives

Job scripts use `#SBATCH` directives to specify resource requirements and job options.

- **Common directives**: `--nodes`, `--ntasks`, `--cpus-per-task`, `--mem`, `--time`, `--partition`, `--job-name`, `--output`, `--error`.
- **Resource specification**: See "Resource specification fundamentals" above for detailed explanation of how `--nodes`, `--ntasks`, `--cpus-per-task`, and `--mem` interact.
- **Time limits**: `--time=2:00:00` sets a 2-hour walltime limit. Use `--time-min` for minimum time guarantees.
- **Output control**: `--output=job-%j.out` sets stdout filename, `--error=job-%j.err` sets stderr filename. Use `%j` for job ID, `%A` for array job ID, `%a` for array task ID.
- **Example with output files** (script on login node, output files written to submission directory on shared filesystem):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=training
  #SBATCH --nodes=4
  #SBATCH --ntasks=32
  #SBATCH --output=/home/user/jobs/train-%j.out
  #SBATCH --error=/home/user/jobs/train-%j.err
  
  srun python train.py
  ```

#### Slurm environment variables

Slurm injects environment variables into the job's environment.

- **Identity**: `SLURM_JOB_ID` (job ID), `SLURM_JOB_USER` (submitting user), `SLURM_JOB_NAME` (job name).
- **Topology**: `SLURM_NODELIST` (comma-separated node list), `SLURM_JOB_NUM_NODES` (node count), `SLURM_NODEID` (node index in allocation).
- **Task geometry**: `SLURM_NTASKS` (total tasks), `SLURM_CPUS_PER_TASK` (CPUs per task), `SLURM_NPROCS` (total processes).
- **Process identity**: `SLURM_PROCID` (MPI rank/global task ID), `SLURM_LOCALID` (local task ID on node), `SLURM_STEP_ID` (step ID within job).
- **Array jobs**: `SLURM_ARRAY_JOB_ID`, `SLURM_ARRAY_TASK_ID`, `SLURM_ARRAY_TASK_COUNT`, `SLURM_ARRAY_TASK_MAX`, `SLURM_ARRAY_TASK_MIN`.
- **Environment inheritance**: By default, `sbatch` copies the user's current environment (loaded modules, paths) into the job. Environment inheritance can be controlled to start with a minimal environment (only Slurm variables) or to explicitly copy all variables.


#### Job prolog and epilog

Administrators can configure scripts that run before and after every job.

- **Prolog**: Scripts configured via the `Prolog` parameter in `slurm.conf` run on each allocated node before the job starts. Useful for setting up node-specific configurations or validating node state.
- **Epilog**: Scripts configured via the `Epilog` parameter in `slurm.conf` run on each allocated node after the job completes. Useful for cleanup, health checks, or logging.
- **User prolog/epilog**: Users can specify `--prolog` and `--epilog` scripts in job scripts, but these require administrator configuration to enable.

### Job script error handling and exit codes

Job scripts can fail for various reasons, and Slurm interprets exit codes to determine job state.

- **Exit code interpretation**: If a job script exits with code 0, the job state is `COMPLETED`. If it exits with a non-zero code, the job state is `FAILED`. The exit code of the last command in the script determines the job's final state.
- **Script failure handling**: If a command in the script fails (non-zero exit), subsequent commands may still execute unless the script uses `set -e` (exit on error). Use `set -e` at the start of Bash scripts to ensure the job fails immediately on any command error.
- **Signal handling in scripts**: Job scripts can trap signals to perform cleanup. Use `trap 'cleanup_function' EXIT TERM INT` to run cleanup on job termination. Custom signals can be sent before walltime expiration.
- **Job state transitions**: Jobs transition `PENDING` → `RUNNING` → `COMPLETED` (exit 0) or `FAILED` (non-zero exit) or `CANCELLED` (user/admin cancellation). Preempted jobs may be `REQUEUED` back to `PENDING`.

### Job output and logging

Slurm captures job output and provides mechanisms for log management and notifications.

- **stdout/stderr handling**: By default, job output goes to files in the submission directory. Custom paths can be specified for output and error files.
- **Log file naming**: Log file naming patterns support placeholders for job ID, array job ID, array task ID, node name, and username.

## dstack

### Run submission and definition

A **run** is the primary unit of workload in dstack, analogous to a Slurm job. A run can spawn one or multiple **jobs**, depending on the configuration. For distributed tasks with multiple nodes, each node gets its own job. Each job runs on a single instance. Runs are created from run configurations defined in YAML files (ending with `.dstack.yml`). There are three types of run configurations:

1. **`dev-environment`**: Provisions an instance and provides access via a desktop IDE (typically VS Code). The instance remains running until explicitly stopped, allowing interactive development and debugging.
2. **`task`**: Executes a bash script or series of commands until completion. Tasks are batch-oriented and terminate when the commands finish, similar to Slurm batch jobs.
3. **`service`**: Runs a long-lived application that exposes a port for inference or web services. Services are out of scope for this guide, as they implement inference workloads that Slurm does not handle.

**Submission methods**: While runs can be submitted via the API, the primary method is using `dstack apply` with a configuration file. The command reads the YAML configuration, provisions resources, and starts the run. By default, `dstack apply` attaches the CLI to the run, which enables port forwarding if the task defines ports. You can also attach the CLI to a running run later using `dstack attach <run-name>`.

**Port forwarding**: Tasks can define ports in their configuration. When ports are specified, `dstack apply` (or `dstack attach`) automatically forwards these ports from the remote instance to your local machine, enabling secure access to applications running on those ports.

**Examples:**

**Dev environment** (interactive development):
```yaml
type: dev-environment
name: vscode

python: 3.12
ide: vscode

resources:
  gpu: 24GB
```

**Task** (single-node batch job):
```yaml
type: task
name: train

python: 3.12
commands:
  - uv pip install -r requirements.txt
  - python train.py

resources:
  gpu: H100:1
```

**Task with port forwarding** (web application):
```yaml
type: task
name: streamlit-app

python: 3.12
commands:
  - uv pip install streamlit
  - streamlit hello

ports:
  - 8501

resources:
  gpu: 24GB
```

When running this task, `dstack apply` forwards port `8501` from the remote instance to `localhost:8501`, allowing you to access the Streamlit application from your local machine.

**Distributed task** (multi-node batch job):
```yaml
type: task
name: train-distrib

nodes: 2

python: 3.12
env:
  - NCCL_DEBUG=INFO
commands:
  - git clone https://github.com/pytorch/examples.git pytorch-examples
  - cd pytorch-examples/distributed/ddp-tutorial-series
  - uv pip install -r requirements.txt
  - |
    torchrun \
      --nproc-per-node=$DSTACK_GPUS_PER_NODE \
      --node-rank=$DSTACK_NODE_RANK \
      --nnodes=$DSTACK_NODES_NUM \
      --master-addr=$DSTACK_MASTER_NODE_IP \
      --master-port=12345 \
      multinode.py 50 10

resources:
  gpu: 24GB:1..2
  shm_size: 24GB
```

**Fleet requirement for distributed tasks:** Distributed tasks require a fleet with `placement: cluster` configured. This ensures instances are provisioned with optimal inter-node connectivity (e.g., InfiniBand, EFA, GPUDirect) for distributed workloads. See [Fleet-based instance provisioning](#fleet-based-instance-provisioning) below and [Cluster node management](08_cluster-node-management.md) for details on creating and configuring fleets.

**Job startup and termination order**: For distributed tasks, the `startup_order` property controls when jobs start. With `workers-first`, worker nodes start before the master node. The `stop_criteria` property controls when jobs terminate. With `master-done`, worker jobs terminate when the master job completes, even if they are still running (e.g., waiting in `sleep infinity`). See [MPI example](#mpi-example-with-startup-order-and-stop-criteria) in the appendix for a complete example using these properties.

### Container and process execution

dstack runs user configurations as Docker containers. The execution involves two components:

- **`dstack-shim`**: Runs on the host (for VM-based backends and SSH fleets) and is responsible for:
  - Pulling Docker images (public or private)
  - Configuring Docker containers (mounts, GPU forwarding, entrypoint, network mode)
  - Starting and stopping containers
  - Managing GPU resource allocation and volume mounting/unmounting
  - Communicating with the dstack server via REST API through an SSH tunnel

- **`dstack-runner`**: Runs inside the Docker container as the entrypoint and is responsible for:
  - Setting environment variables and secrets
  - Executing user commands from the job specification
  - Collecting and serving logs to the dstack server (via HTTP) and CLI (via WebSocket)
  - Reporting job status and exit codes
  - Handling termination signals from the dstack server

**Container lifecycle**: The container entrypoint installs `openssh-server`, configures SSH keys, and starts both `sshd` and `dstack-runner`. All communication between the dstack server and `dstack-runner` happens via REST API through an SSH tunnel.

**Exit code handling**: When a task's commands complete, `dstack-runner` reports the exit code to the server. If the exit code is 0, the job state becomes `DONE`. If the exit code is non-zero, the job state becomes `FAILED`. For distributed tasks, if any job fails, the run's state depends on the retry policy (see below).

### Fleet-based instance provisioning

dstack uses **fleets** to manage compute resources. You must create a fleet before submitting runs. When you submit a run, dstack reuses idle instances from a matching fleet or provisions new ones based on the fleet's configuration.

For distributed tasks requiring multiple nodes, the fleet must have `placement: cluster` configured to ensure optimal inter-node connectivity (e.g., InfiniBand, EFA, GPUDirect). See [Cluster node management](08_cluster-node-management.md) for details on creating and configuring fleets.

### Run and job state management

A run can spawn one or multiple jobs (one per node for distributed tasks). The run's state aggregates the states of its jobs: if any job is `RUNNING`, the run is `RUNNING`; if all jobs are `DONE`, the run becomes `DONE`; if any job fails, the run becomes `FAILED` (unless retry is enabled).

Jobs progress through states: `SUBMITTED` → `PROVISIONING` → `PULLING` → `RUNNING` → `TERMINATING` → `DONE`/`FAILED`/`TERMINATED`/`ABORTED`.

### Retry policy

By default, runs fail on errors, capacity issues, or instance interruptions. You can configure automatic retry using the `retry` property in the run configuration. See [Queueing, prioritization, and scheduling](02_queueing-prioritization-scheduling.md) for details on retry behavior and configuration.

### Environment variables

dstack automatically injects environment variables into runs that correspond to the run and job context. These are available in dev environments, tasks, and services:

**Run identity**:
- `DSTACK_RUN_NAME`: The name of the run
- `DSTACK_RUN_ID`: The UUID of the run
- `DSTACK_REPO_ID`: The ID of the repo

**Job identity**:
- `DSTACK_JOB_ID`: The UUID of the job submission

**Resource topology**:
- `DSTACK_GPUS_NUM`: The total number of GPUs in the run
- `DSTACK_NODES_NUM`: The number of nodes in the run
- `DSTACK_GPUS_PER_NODE`: The number of GPUs per node

**Distributed task coordination**:
- `DSTACK_NODE_RANK`: The rank of the node (0 for master, 1, 2, ... for workers)
- `DSTACK_MASTER_NODE_IP`: The internal IP address of the master node
- `DSTACK_NODES_IPS`: Newline-delimited list of internal IP addresses of all nodes
- `DSTACK_MPI_HOSTFILE`: Path to a pre-populated MPI hostfile

**Working directories**:
- `DSTACK_WORKING_DIR`: The working directory of the run
- `DSTACK_REPO_DIR`: The directory where the repo is mounted (if any)

These variables enable distributed frameworks (PyTorch DDP, Horovod, MPI) to discover nodes and coordinate communication. For example, `torchrun` uses `DSTACK_MASTER_NODE_IP` and `DSTACK_NODE_RANK` to establish the distributed training cluster.

## Appendix

### MPI example with startup_order and stop_criteria

For MPI workloads that require specific job startup and termination behavior, dstack provides `startup_order` and `stop_criteria` properties. This example demonstrates running NCCL tests using MPI:

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

**How it works**:
- `startup_order: workers-first` ensures worker nodes (rank > 0) start before the master node (rank 0), allowing the master to connect to already-running workers.
- `stop_criteria: master-done` causes all worker jobs to terminate when the master job completes, even if they are still executing (e.g., the `sleep infinity` command on workers).
- The master node (rank 0) runs the MPI command using `mpirun` with the pre-populated `DSTACK_MPI_HOSTFILE`.
- Worker nodes wait indefinitely until the master completes, at which point they are automatically terminated.