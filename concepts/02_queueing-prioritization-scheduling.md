# 2 Queueing, prioritization, and scheduling mechanics

## Slurm

### Centralized scheduling loop

`slurmctld` runs a single, global scheduling thread that periodically evaluates the queue and decides which jobs to start.

- **The plugin (`sched/backfill`)**: While Slurm offers a simple FIFO scheduler (`sched/builtin`), virtually all production clusters use `sched/backfill`. This plugin enables high utilization by fitting small jobs into gaps left by large, high-priority jobs waiting for resources.
- **Scheduling cycle**: The scheduler runs at intervals (default ~30-60s for backfill). During a cycle, it first attempts to place the highest priority jobs (Main Scheduler). If those are blocked, it triggers the Backfill Cycle to find smaller jobs that can fit without delaying the top jobs.
- **Job state transitions**: Jobs move from `PENDING` → `RUNNING` when resources are allocated, then to `COMPLETED`, `FAILED`, or `CANCELLED` upon finish.

### Queueing and eligibility

For a job to move from `PENDING` to `RUNNING`, it must meet both **policy** and **resource** constraints.

- **Reason codes**: Jobs can be pending for various reasons. Common reasons include `Priority` (waiting for higher ranked jobs), `Resources` (cluster full), `Dependency` (waiting for another job), or `AssocGrpCPULimit` (user hit a running limit).
- **Eligibility**: The scheduler evaluates these constraints every cycle. A job is "eligible" only when it satisfies all policies (Time, QOS, Partition limits) and is simply waiting for idle resources.

### Priority calculation

Slurm uses the `priority/multifactor` plugin to assign a single integer value to every job. The higher the value, the sooner the scheduler attempts to place it.

- **Fairshare**: The most common factor. It prioritizes users who have been "under-using" their allocated share of the cluster compared to heavy users.
- **Age**: Slightly increases priority the longer a job sits in the queue to prevent starvation.
- **Job size**: Can be tuned to prioritize large jobs (to encourage parallelism) or small jobs (for throughput).
- **QOS**: A static boost based on Quality of Service (e.g., a "debug" QOS might have high priority).
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
- **Allocation**: The scheduler must satisfy all component requirements simultaneously. All components start together and are managed as a single job.

#### Exclusive vs shared access

Jobs can request exclusive node access or allow sharing with other jobs.

- **Exclusive (`--exclusive`)**: The job gets the entire node(s) to itself. No other jobs can run on those nodes while the job is active. This is the default for multi-node jobs.
- **Shared access**: In partitions with `OverSubscribe=YES`, multiple jobs can share nodes automatically. Useful for high-throughput workloads where many small jobs can efficiently share resources.
- **Partition-level control**: The partition's `OverSubscribe` setting controls whether jobs can share nodes. Individual jobs can request exclusive access with `--exclusive` even in oversubscribed partitions.

## dstack

TBA