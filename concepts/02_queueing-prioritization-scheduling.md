# 2 Queueing, prioritization, and scheduling mechanics

## Slurm

### Centralized scheduling loop

`slurmctld` runs a single, global scheduling thread that periodically evaluates the queue and decides which jobs to start.

- **The plugin (`sched/backfill`)**: While Slurm offers a simple FIFO scheduler (`sched/builtin`), virtually all production clusters use `sched/backfill`. This plugin enables high utilization by fitting small jobs into gaps left by large, high-priority jobs waiting for resources.
- **Scheduling cycle**: The scheduler runs at intervals (default ~30-60s for backfill). During a cycle, it first attempts to place the highest priority jobs (Main Scheduler). If those are blocked, it triggers the Backfill Cycle to find smaller jobs that can fit without delaying the top jobs.
- **Job state transitions**: Jobs move from `PENDING` → `RUNNING` when resources are allocated, then to `COMPLETED`, `FAILED`, or `CANCELLED` upon finish.

### Queueing and eligibility

For a job to move from `PENDING` to `RUNNING`, it must meet both **policy** and **resource** constraints.

- **Partition context**: Scheduling occurs within partitions. Each partition has its own queue, and jobs are scheduled only against nodes in the specified partition.
- **Reason codes**: Jobs can be pending for various reasons. Common reasons include `Priority` (waiting for higher ranked jobs), `Resources` (cluster full), `Dependency` (waiting for another job), or `AssocGrpCPULimit` (user hit a running limit).
- **Associations and limits**: Slurm uses associations to define resource limits per user, account, or group. Common limits include `MaxJobs`, `MaxCPUs`, `MaxNodes`, and `MaxWall`. When a user hits these limits, jobs remain pending with reason codes like `AssocGrpCPULimit`.
- **Example: checking job status and reason** (from login node):
  ```bash
  squeue -j 12345
  # JOBID PARTITION     NAME     USER ST  TIME  NODES REASON
  # 12345     gpu    training   user1 PD  0:00      4 Priority
  ```
- **Eligibility**: The scheduler evaluates these constraints every cycle. A job is "eligible" only when it satisfies all policies (Time, QOS, Partition limits) and is simply waiting for idle resources.

### Priority calculation

Slurm uses the `priority/multifactor` plugin to assign a single integer value to every job. The higher the value, the sooner the scheduler attempts to place it.

- **Fairshare**: The most common factor. It prioritizes users who have been "under-using" their allocated share of the cluster compared to heavy users.
- **Age**: Slightly increases priority the longer a job sits in the queue to prevent starvation.
- **Job size**: Can be tuned to prioritize large jobs (to encourage parallelism) or small jobs (for throughput).
- **QOS**: A static boost based on Quality of Service (e.g., a "debug" QOS might have high priority).
- **Example: requesting QOS in job script** (script on login node):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=high-priority
  #SBATCH --qos=debug  # Request high-priority QOS
  #SBATCH --nodes=1
  #SBATCH --gres=gpu:1
  #SBATCH --time=1:00:00
  
  srun python test_model.py
  ```
- **Recalculation**: Priorities are dynamic. As a job waits, its "Age" factor grows, potentially moving it up the list.

### Multi-node & parallel allocation

For parallel jobs requesting multiple nodes (e.g., `-N 4`), Slurm utilizes **Space Sharing** with strict co-allocation rules.

- **All-or-nothing constraint**: The scheduler enforces that all requested nodes must be available *simultaneously*. If a job needs 4 nodes and only 3 are free, the job remains `PENDING` until all 4 are `IDLE` at the exact same moment.
- **Topology awareness**: If configured, Slurm attempts to place the job on nodes that are physically close (same switch/rack) to minimize network latency.
- **Fragmentation**: The all-or-nothing requirement creates "holes" (fragmentation) in the cluster where nodes sit idle waiting for their peers to free up. This is the primary driver for using **Backfill Scheduling**.

### Backfill scheduling mechanics

The backfill scheduler is the engine of cluster efficiency. It allows lower-priority jobs to start *now* if they do not delay the start time of the highest-priority jobs.

- **The mechanism**: The scheduler looks at the top priority job and calculates a "Shadow Reservation" based on when resources will become available. It then scans the rest of the queue for smaller/shorter jobs that fit into the current empty slots *and* finish before that reservation begins.
- **The requirement**: Backfill strictly relies on the job's **Wall Time Limit** (`--time`). Users must set accurate time limits. If a user requests 24 hours for a 10-minute job, the backfill scheduler assumes it will block resources for 24 hours and will likely not schedule it.

### Throughput limits & tuning

In large clusters (10,000+ jobs), the single-threaded scheduler cannot check every job in every cycle without "freezing" the controller.

- **Cycle limits**: Use `SchedulerParameters` to limit the depth of the schedule scan. This sets the maximum number of jobs to test per cycle to keep the controller responsive.
- **Continuous scheduling**: The scheduler can resume scanning where it left off in the previous cycle, ensuring deep-queue jobs eventually get checked.
- **Latency**: Because of these limits, a job submitted to a busy cluster may take a few scheduling cycles (minutes) to be evaluated and started, even if resources are free.

### Advanced scheduling features

#### Job packing and oversubscription

Job packing allows multiple jobs to run simultaneously on the same node, sharing CPU and memory resources.

- **OverSubscribe**: Set `OverSubscribe=YES` in a partition to allow multiple jobs per node. The scheduler can pack multiple small jobs onto a single node.
- **Exclusive access**: By default, jobs request exclusive node access. Use `--exclusive` to explicitly request exclusive access. In partitions with `OverSubscribe=YES`, jobs can share nodes automatically without a special flag.
- **Resource allocation**: When packing jobs, Slurm ensures the sum of requested resources (CPUs, memory) does not exceed node capacity. Jobs may receive fewer resources than requested if the node is shared.
- **Burstable resources**: Some clusters allow oversubscription where the sum of requested resources can exceed physical capacity, relying on the fact that not all jobs use peak resources simultaneously.

#### Heterogeneous jobs

Heterogeneous jobs allow a single job to request different resource requirements for different components.

- **Component specification**: Use `--het-group` to define job components with different requirements. Submit multiple job specifications separated by `:` where each `--het-group` defines one component with its own resource requirements.
- **Example: heterogeneous job submission** (from login node):
  ```bash
  # Submit heterogeneous job with two components
  sbatch --het-group=0 --nodes=2 --gres=gpu:1 : \
         --het-group=1 --nodes=4 --gres=gpu:2 job.sh
  ```
- **Allocation**: The scheduler must satisfy all component requirements simultaneously. All components start together and are managed as a single job.

#### Exclusive vs shared access

Jobs can request exclusive node access or allow sharing with other jobs.

- **Exclusive (`--exclusive`)**: The job gets the entire node(s) to itself. No other jobs can run on those nodes while the job is active. This is the default for multi-node jobs.
- **Shared access**: In partitions with `OverSubscribe=YES`, multiple jobs can share nodes automatically. Useful for high-throughput workloads where many small jobs can efficiently share resources.
- **Partition-level control**: The partition's `OverSubscribe` setting controls whether jobs can share nodes. Individual jobs can request exclusive access with `--exclusive` even in oversubscribed partitions.

#### Preemption

Slurm supports job preemption, where higher-priority jobs can preempt (suspend or cancel) lower-priority running jobs. This is typically configured via QOS settings (`PreemptMode`, `PreemptType`). Preempted jobs may be requeued automatically or require manual resubmission depending on configuration.

## dstack

### Background scheduling workers

dstack uses background workers that periodically process runs and jobs to handle scheduling, provisioning, and lifecycle management.

- **Scheduling interval**: The `process_submitted_jobs` worker runs every 4 seconds to process jobs waiting for provisioning. The `process_runs` worker runs every 2 seconds to manage run state transitions and handle retries. These intervals include random variation to prevent synchronization when multiple dstack server replicas are running.
- **Batch processing**: Each scheduling cycle processes multiple jobs (default batch size of 5) to improve throughput. The effective batch size is automatically reduced to 1 if no jobs were processed in the last 2 minutes, to avoid overwhelming the system when many jobs are submitted simultaneously via API without getting a run plan first.
- **Throughput tuning**: The `SERVER_BACKGROUND_PROCESSING_FACTOR` setting allows scaling the number of background worker replicas to handle higher loads. By default, one server replica can handle approximately 150 active jobs with 2 minutes processing latency. Increasing this factor requires corresponding increases in database connection limits. Multiple dstack server replicas can run in parallel for high availability and horizontal scaling, with each replica running its own set of background workers.
- **Multiple replicas**: For services with multiple replicas (configured via `replicas`), each replica is scheduled independently. The scheduler processes all replicas concurrently, allowing them to be provisioned in parallel when resources are available.

### Queueing and job ordering

Jobs are queued and processed based on priority and submission order.

- **Priority-based ordering**: Jobs are processed in FIFO order sorted by run priority (descending), then by `last_processed_at` (ascending) within the same priority level. This means higher priority runs are provisioned first, and among runs with the same priority, older submissions are processed first.
- **Project isolation**: Projects are entirely isolated. Priority is evaluated within each project, so runs from different projects do not affect each other's scheduling.
- **Default priority**: If not specified, runs have a default priority of 0. Priority can be set in the run configuration using the `priority` property (integer value between 0 and 100).
- **Fleet selection**: The scheduler selects fleets based on idle instance availability, pricing, and placement constraints. It prefers fleets with idle instances that can accommodate the run, then selects the cheapest offer. For distributed tasks, only fleets with `placement: cluster` are considered, and all required instances must be provisioned before any job starts. This ensures optimal inter-node connectivity and coordinated startup for cluster workloads.

### Run and job states

A run is the primary unit of workload in dstack (similar to a Slurm job). A run can consist of one or multiple jobs—for example, a distributed task with `nodes: 4` creates one job per node. Each job represents a specific container workload, while the run aggregates the status of all its jobs.

- **Job states**: Jobs have states: `SUBMITTED`, `PROVISIONING`, `PULLING` (Docker image being pulled), `RUNNING`, `TERMINATING`, and terminal states `DONE`, `FAILED`, `TERMINATED`, or `ABORTED`.
- **Run states**: A run's state is an aggregate of its jobs' states. For example, a run becomes `RUNNING` when any job enters `RUNNING`, and `PROVISIONING` when any job is `PROVISIONING` or `PULLING`.
- **Distributed tasks**: For distributed tasks, all jobs must be provisioned before execution begins. Each job is scheduled independently, but the run waits for all instances to be ready before starting any job.

> Note: The detailed job execution lifecycle, including state transitions and provisioning mechanics, is covered in the [job execution lifecycle](04_job-execution-lifecycle.md) guide.

### Retry policy

By default, dstack does not retry failed jobs or runs. If a job fails to provision due to capacity constraints, exits with an error, or is interrupted, the run will fail immediately.

- **Retry configuration**: Retry can be enabled via the `retry` property, specifying which events trigger retries (`on_events`: `no-capacity`, `error`, or `interruption`) and the maximum retry duration. Duration is calculated from run age for `no-capacity`, or from the last event occurrence for `interruption` and `error`.
- **Distributed task retry**: If any job of a distributed task fails and retry is enabled, dstack stops all jobs and resubmits the entire run to ensure all nodes are provisioned together.
- **Example: task configuration with priority and retry** (`.dstack.yml`):
  ```yaml
  type: task
  name: training-job
  
  priority: 50
  
  retry:
    on_events: [no-capacity]
    duration: 1h
  
  repos:
    - .
  
  python: 3.12
  commands:
    - python train.py
  
  resources:
    gpu: H100:1
  ```
- **Example: monitoring runs and jobs** (from terminal):
  ```bash
  $ dstack ps
  NAME          BACKEND  RESOURCES       PRICE    STATUS       SUBMITTED
  training-job  aws      H100:1 (spot)  $4.50    running      5 mins ago
  dev-env       gcp      L4:24GB         $0.16    provisioning 2 mins ago
  ```
  The output shows both runs and their constituent jobs.

### Monitoring queues

You can monitor queued and running jobs using the dstack CLI or the web dashboard.

- **List runs**: Use `dstack ps` to view all runs and their current status, including queued (`SUBMITTED` or `PROVISIONING`) and running jobs. The output shows run names, backends, resources, prices, and status.
- **Queue position and waiting reasons**: Unlike Slurm's `squeue` which shows queue position and reason codes, dstack doesn't display explicit queue position or detailed waiting reasons. The status (`SUBMITTED`, `PROVISIONING`) indicates the current state but not the specific reason for waiting.
- **Verbose output**: Use `dstack ps --verbose` (or `-v`) to see additional details such as inactivity duration for dev environments.
- **UI dashboard**: The dstack UI dashboard provides a visual view of all runs, their status, and resource usage.

### Scheduled tasks

Tasks can be scheduled to run periodically using cron syntax.

- **Scheduled tasks**: Specify `schedule` with a cron expression to start a task periodically at specific UTC times. The scheduler automatically submits runs according to the schedule.
- **Example: scheduled task** (`.dstack.yml`):
  ```yaml
  type: task
  name: daily-training
  
  schedule:
    cron: "15 23 * * *"  # Every day at 23:15 UTC
  
  repos:
    - .
  
  python: 3.12
  commands:
    - python train.py
  
  resources:
    gpu: H100:1
  ```

The detailed mechanics of execution and cluster node management are covered in the [job execution lifecycle](04_job-execution-lifecycle.md) and [cluster node management](08_cluster-node-management.md) guides.

### Scheduling considerations

The scheduler considers `max_duration`, `spot` policy, and `resources` when selecting fleets and provisioning instances.

- **Backfill vs. utilization**: Unlike Slurm's backfill scheduler that requires accurate wall time estimates to fit jobs into gaps, dstack achieves utilization through idle instance reuse and blocks. dstack cannot make "shadow reservation" calculations like Slurm, so it cannot backfill small jobs around large jobs.
- **Cost control**: The `max_duration` setting terminates runs after a maximum runtime for cost control, not for scheduling optimization. The `utilization_policy` can auto-terminate runs if GPU utilization falls below a threshold. For dev environments, `inactivity_duration` automatically stops idle instances.

### Blocks and network modes

A fleet can be configured with `blocks` to divide an instance's resources (GPUs, CPUs, memory) into multiple units, allowing multiple jobs to share the same instance. Resources are split evenly across blocks, while disk is shared.

- **Network modes**: When a job uses all blocks (exclusive access), it uses host network mode for direct network access. When a job shares an instance (uses only some blocks), it uses bridge network mode to isolate traffic between jobs.
- **Distributed tasks**: Distributed tasks always require exclusive access (blocks are not enabled) and use host network mode, enabling direct inter-node communication via private IP addresses.

### Limitations

dstack has some limitations compared to Slurm's scheduling capabilities.

- **Fairshare**: dstack does not implement fairshare. Priority is the only mechanism for controlling job ordering.
- **Preemption**: dstack does not support job preemption. Once a job is running, it cannot be preempted by a higher-priority job.
- **Job dependencies**: dstack does not support job dependencies. Use external workflow frameworks (e.g., Airflow, Prefect) that submit dstack runs based on dependency logic.
- **Heterogeneous jobs**: dstack does not support heterogeneous jobs. All jobs within a run must use the same resource specification.
- **Topology-aware scheduling**: dstack does not support automatic topology-aware scheduling except in specific cases: AWS fleets with cluster placement automatically configure EFA networking when supported, and GCP fleets automatically configure GPUDirect-TCPXO/GPUDirect-TCPX or RoCE networking for certain instance types. For other cases, manually configure network topology in scripts.