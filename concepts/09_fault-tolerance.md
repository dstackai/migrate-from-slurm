# 9 Fault tolerance, checkpointing, and job recovery

## Slurm

### Job failure types

In a Slurm environment, jobs generally fail in one of two ways, requiring different recovery strategies.

- **System failure (Hardware crash)**: The compute node loses power or connectivity. Slurm detects the timeout and marks the node `DOWN`.
- **Application failure (Software crash)**: The user's code produces a segmentation fault or exits with a non-zero error code.
- **Preemption**: A higher-priority job forces a running job to stop immediately to free up resources.

### Automatic requeueing

Slurm can automatically restart a job if the hardware fails, but this behavior must be requested.

- **Requeueing**: By default, if a node crashes, the job fails. If requeueing is enabled (via job submission or admin configuration), Slurm puts the job back into `PENDING` state to run again on a healthy node.
- **Example: enabling requeue** (script on login node):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=requeue-job
  #SBATCH --nodes=4
  #SBATCH --requeue  # Enable automatic requeue on node failure
  
  srun python train.py
  ```
- **Distributed (MPI) behavior**: In a multi-node job, if *one* node fails, the entire job allocation is usually killed. If requeueing is active, the entire parallel job restarts from scratch on a new set of nodes.

### Preemption & grace periods

In large clusters, low-priority jobs (e.g., "Guest" or "Backfill" QOS) may be killed to make room for high-priority owners.

- **Grace period**: When Slurm decides to preempt a job, it does not kill it instantly. It enters a **Grace Period** (configured in `slurm.conf`).
- **Soft kill**: Slurm sends `SIGTERM` to the job steps. The application has this window to save its state (checkpoint) before `SIGKILL` is sent.
- **Requeue mechanism**: Preempted jobs are typically requeued automatically. They return to the queue in a held state or pending state, depending on configuration.

### Checkpointing & Signaling

Slurm does not inherently save the memory state of a job (no "system hibernation"). "Checkpointing" relies on the application saving its own progress to disk.

- **Application support**: The user's code (or ML framework like PyTorch/TensorFlow) must support saving state to disk periodically.
- **Signaling**: Users can instruct Slurm to send a specific signal to the application *before* the time limit expires or during preemption. The application catches this signal, writes a checkpoint file, and exits cleanly.
- **Example: checkpoint signal handling** (script on login node, executes on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=checkpoint-job
  #SBATCH --nodes=1
  #SBATCH --time=2:00:00
  #SBATCH --signal=B:USR1@60  # Send USR1 signal 60 seconds before time limit
  
  # Trap signal for checkpointing
  trap 'echo "Checkpointing..."; python save_checkpoint.py' USR1
  
  srun python long_training.py
  ```

### Distributed job recovery (MPI)

Recovering distributed jobs is strictly harder than single-node jobs.

- **All-or-Nothing**: Standard MPI is not fault-tolerant. If Rank 0 fails, the MPI communicator breaks, and Ranks 1-1000 also die.

## dstack

### Instance failure detection

dstack continuously monitors instance health and connectivity to detect failures.

- **Unreachable instances**: dstack tracks instance reachability via periodic health checks. If an instance becomes unreachable (e.g., due to network issues or hardware failure), dstack marks it with an `unreachable` flag. If an instance remains unreachable beyond a termination deadline, dstack marks it as `TERMINATING` with termination reason `UNREACHABLE`.
- **Instance health status**: Instances can have health statuses: `HEALTHY`, `WARNING`, or `FAILURE`. Unhealthy instances are automatically excluded from scheduling to prevent jobs from running on degraded hardware. For GPU instances, dstack integrates with NVIDIA DCGM (Data Center GPU Manager) via `dstack-shim` to perform passive GPU health checks, monitoring for reliability issues (e.g., ECC errors, thermal issues) without interrupting workloads. Health checks work automatically on supported cloud backends (AWS, Azure, GCP, OCI) and SSH fleets where DCGM is installed. Health status is displayed in fleet listings:
  ```bash
  $ dstack fleet
  
   FLEET     INSTANCE  BACKEND          RESOURCES  STATUS          PRICE   CREATED
   my-fleet  0         aws (us-east-1)  T4:16GB:1  idle            $0.526  11 mins ago
             1         aws (us-east-1)  T4:16GB:1  idle (warning)  $0.526  11 mins ago
             2         aws (us-east-1)  T4:16GB:1  idle (failure)  $0.526  11 mins ago
  ```
  Instances with `idle (warning)` remain usable but should be monitored, while instances with `idle (failure)` are automatically excluded from scheduling. For detailed information on GPU health monitoring, see the [GPU health monitoring](10_gpu-health-monitoring.md) and [monitoring and observability](14_monitoring-observability.md) guides.

### Backend fleet instance management

For backend fleets, dstack automatically maintains the configured minimum number of instances.

- **Minimum instance guarantee**: When a fleet is configured with a minimum number of nodes (e.g., `nodes: 2..8`), dstack continuously ensures at least that many instances are active. If an instance fails or is terminated, dstack automatically provisions a replacement to maintain the minimum.
- **Automatic replacement**: Failed instances are detected and replaced by the fleet maintenance process, which runs periodically to check the fleet state and provision missing instances.

### Retry policy

By default, dstack does not retry failed runs. If a job fails to provision, exits with an error, or is interrupted, the run fails immediately.

- **Retry configuration**: Retry can be enabled via the `retry` property in the run configuration, specifying which events trigger retries (`on_events`: `no-capacity`, `error`, or `interruption`) and the maximum retry duration. Duration is calculated from run age for `no-capacity` events, or from the last event occurrence for `interruption` and `error` events.
- **Example: task configuration with retry** (`.dstack.yml`):
  ```yaml
  type: task
  name: train
  
  python: 3.12
  repos:
    - .  # Clone current directory repo
  commands:
    - python train.py
  
  retry:
    on_events: [no-capacity, error, interruption]
    duration: 1h
  ```

### Distributed task retry

For distributed tasks (multi-node runs), retry behavior ensures all nodes are provisioned together.

- **All-or-nothing retry**: If any job of a distributed task fails and retry is enabled, dstack stops all jobs and resubmits the entire run. This ensures all nodes are provisioned together and maintains cluster connectivity requirements (e.g., InfiniBand, EFA).
- **Scheduling coordination**: When retrying a distributed task, dstack waits for all required instances to be available before starting any job, ensuring the cluster is fully provisioned before execution begins.

### Persistence via volumes

Unlike Slurm's shared filesystem model, dstack uses volumes for persistent storage across runs.

- **Network volumes**: Cloud-managed persistent storage volumes that survive across runs. Volumes are bound to a specific backend and region, and can be attached to runs to persist data.
- **Instance volumes**: Host directories mounted into containers, useful for caching or when using SSH fleets with pre-mounted network storage.
- **Checkpointing**: Applications can save checkpoints to volume-mounted directories, which persist even if the run fails or is interrupted. For detailed information on volumes and filesystem access, see the [filesystems guide](08_file-systems.md).

### Spot instances support

dstack supports spot instances (preemptible/transient instances) via the `spot_policy` configuration.

- **Spot policies**: Configure `spot_policy` as `spot` (only spot instances), `on-demand` (only on-demand instances), or `auto` (prefer spot, fall back to on-demand). Spot instances are typically cheaper but can be interrupted by the cloud provider.
- **Spot interruptions**: When a spot instance is interrupted, the job termination reason is set to `INTERRUPTED_BY_NO_CAPACITY`. By default, dstack does not retry interrupted runs. If retry is enabled with `interruption` in `on_events`, dstack automatically resubmits the run.
- **Example: task configuration with spot policy and retry** (`.dstack.yml`):
  ```yaml
  type: task
  name: train
  
  python: 3.12
  repos:
    - .  # Clone current directory repo
  commands:
    - python train.py
  
  spot_policy: auto  # Prefer spot instances, fall back to on-demand
  retry:
    on_events: [interruption]
    duration: 1h
  ```

### Exit code handling

dstack captures container exit codes, container errors, and job errors to determine job success or failure.

- **Exit code interpretation**: Exit code `0` marks the job as `DONE`; any non-zero exit code marks it as `FAILED`. Commands execute sequentially, and execution stops if any command exits with a non-zero code.
- **Inspecting exit codes**: Exit codes can be viewed via `dstack ps` (use `dstack ps -v` for verbose output), `dstack attach`, or the API.
- **Distributed tasks**: If any job fails (non-zero exit code), the run becomes `FAILED` unless retry is enabled, in which case the run is resubmitted.