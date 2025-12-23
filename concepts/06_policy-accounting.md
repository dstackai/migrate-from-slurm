# 6 Accounts, QOS, and accounting pipeline

## Slurm

### Accounts and associations

Slurm uses a hierarchical tree to track resource usage ("Fairshare") and enforce limits.

- **Account hierarchy**: Accounts form a tree (Parent -> Child -> Grandchild). Usage "bubbles up" the tree, meaning a parent account accumulates the usage of all its children for reporting.
- **The Association**: This is the most critical internal concept. It is the distinct intersection of a **User**, an **Account**, a **Partition**, and a **Cluster**. Limits (like `MaxJobs` or `MaxTRES`) are applied to the *Association*, not just the raw user.
- **Fairshare calculation**: Resource usage flows into the MultiFactor priority plugin. Slurm uses a **Half-Life Decay** mechanism. Recent usage reduces a job's priority significantly, while usage from months ago has little effect.
- **Limit enforcement**: Before a job enters the queue, Slurm checks the Association limits. Common limits include `MaxJobs` (anti-spam), `MaxSubmitJobs`, and `GrpTRES` (total CPUs/GPUs allowed for a group at once).

### QOS (Quality of Service)

While Accounts track "who" is running jobs, QOS defines "how" they are allowed to run. It serves as the primary policy engine.

- **Priority & Preemption**: QOS is the primary mechanism for preemption. A high-priority QOS (e.g., `critical`) can be configured to preempt jobs in a low-priority QOS (e.g., `normal`) to free up resources immediately.
- **Usage Factors**: QOS can accelerate usage charging. For example, a "borrowed" QOS might have a `UsageFactor=2.0`, causing users to burn through their fairshare score twice as fast in exchange for access to extra resources.
- **TRES Limits**: QOS allows for granular hardware limits, such as `MaxTRESPerUser=gpu:2`. This prevents a single user from monopolizing specific hardware types, regardless of their account settings.
- **Grace Time**: Defines how much time a preempted job has to save its state (checkpoint) before being killed or requeued.

### Accounting pipeline (`slurmdbd`)

The `slurmdbd` daemon acts as the secure interface between the compute cluster and the SQL backend.

- **Data flow**: `slurmctld` caches usage data in RAM and asynchronously sends it to `slurmdbd`. If the database goes down, the cluster can typically continue scheduling for a short time (if configured), but limits cannot be updated.
- **The `sacct` command**: This is the user interface for history.
    - **Performance warning**: On large clusters, running `sacct` without a start time (`-S`) or specific job ID queries the entire table scan. This can lock the database. Users should always filter by time or job ID.

## dstack

TBA