# 5 Accounts, QOS, and accounting pipeline

## Slurm

### Accounts and associations

Slurm uses a hierarchical tree to track resource usage ("Fairshare") and enforce limits.

- **Account hierarchy**: Accounts form a tree (Parent -> Child -> Grandchild). Usage "bubbles up" the tree, meaning a parent account accumulates the usage of all its children for reporting.
- **The Association**: The distinct intersection of a **User**, an **Account**, a **Partition**, and a **Cluster**. Limits (like `MaxJobs` or `MaxTRES`) are applied to the *Association*, not just the raw user.
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

- **Data flow**: `slurmctld` caches usage data in RAM and asynchronously sends it to `slurmdbd`. If the database goes down, the cluster can continue scheduling (if configured), but limits cannot be updated.
- **The `sacct` command**: This is the user interface for history.
    - **Performance warning**: On large clusters, running `sacct` without a start time (`-S`) or specific job ID queries the entire table scan. This can lock the database. Users should always filter by time or job ID.
- **Example: querying job history** (from login node):
  ```bash
  # Query specific job
  sacct --job=12345
  
  # Query with time filter (required for large clusters)
  sacct --starttime=2024-01-01 --endtime=2024-01-02
  
  # Query with specific fields
  sacct --job=12345 --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
  ```
- **Example: requesting account in job script** (script on login node):
  ```bash
  #!/bin/bash
  #SBATCH --account=research-group
  #SBATCH --job-name=training
  #SBATCH --nodes=1
  #SBATCH --gres=gpu:1
  
  srun python train.py
  ```

## dstack

### Projects and access control

dstack uses **projects** as the equivalent of Slurm's accounts, but without hierarchical structure. Projects are flat and independent.

- **Projects**: Projects are entirely isolated from each other. Each project can configure its own backends and fleets, and resources are isolated at the project level (projects do not share compute resources). Priority is evaluated within each project. All users within a project have equal access to all resources configured for that project—there are no per-user quotas or resource limits, and users share the same fleet capacity.
- **User roles**: Projects have three roles: **Admin**, **Manager**, and **User**. Users can also be assigned a global admin role. For details on authentication and user management, see the [Authentication and security](12_authentication-security.md) guide.
- **QOS and policy**: dstack uses a simple **priority** system (0-100 integer, set via `priority: 50` in run configuration) instead of Slurm's QOS-based policy engine. The priority system replaces only QOS's priority component. dstack lacks QOS features: no preemption, no usage factors, no TRES limits (hardware limits exist only at fleet level), and no grace time. For detailed information on priority and scheduling mechanics, see the [Queueing, prioritization, and scheduling](02_queueing-prioritization-scheduling.md) guide.
- **No quotas, fairshare, or reservations**: dstack does not support manageable quotas or per-user resource limits (unlike Slurm's association-based limits such as `MaxJobs`, `MaxCPUs`, `GrpTRES`), does not implement fairshare, and does not support reservations. Resource constraints are defined at the fleet level. Most resource enforcement mechanisms in dstack are described in the [Resource model](03_resource-model.md) guide.

### Accounting and run history

dstack maintains run history similar to Slurm's `slurmdbd` accounting database, but accessed via the dstack server's REST API and CLI rather than a dedicated accounting daemon.

- **CLI and REST API**: The dstack server stores run history in its database. Historical information is available via REST API endpoints and the `dstack ps --json` CLI command, providing run details, status, submission time, and resource allocation.
- **Prometheus**: When enabled, dstack can export run and job metrics to Prometheus, providing another way to access historical run information and accounting data. For details on enabling Prometheus integration and available metrics, see the [Monitoring and observability](14_monitoring-observability.md) guide.