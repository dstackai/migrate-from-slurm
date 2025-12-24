# 12 Partition design, node grouping, and queue layout

## Slurm

### Partition fundamentals (Queues)

In Slurm, a **Partition** is the mechanism used to group nodes and jobs. It functions as what other schedulers call a "Queue."

- **The logical layer**: A partition is not a physical object; it is a logical tag applied to a list of nodes. Users submit jobs to a partition (`-p gpu`), not directly to specific nodes.
- **Node-Partition mapping**: A single partition can contain one node, all nodes, or a specific subset.
- **Overlapping partitions**: A single compute node can belong to multiple partitions simultaneously. A node might be in the `gpu` partition (for long jobs) and the `debug` partition (for short, high-priority tests) at the same time.
- **Default partition**: One partition must be designated as the default. If a user submits a job without specifying a partition, it lands here automatically.
- **Example: partition configuration** (in `slurm.conf` on controller node):
  ```bash
  # GPU partition
  PartitionName=gpu Nodes=gpu-node[01-10] Default=NO MaxTime=24:00:00
  
  # CPU partition (default)
  PartitionName=cpu Nodes=cpu-node[01-50] Default=YES MaxTime=72:00:00
  
  # Debug partition (overlaps with gpu partition)
  PartitionName=debug Nodes=gpu-node[01-10] Default=NO MaxTime=1:00:00
  ```
- **Example: submitting to partition** (from login node):
  ```bash
  # Submit to GPU partition
  sbatch --partition=gpu gpu_job.sh
  
  # Or in job script
  #SBATCH --partition=gpu
  ```
- **Example: checking partition status** (from login node):
  ```bash
  # Show partition information
  sinfo
  # PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
  # gpu          up 1-00:00:00     10   idle gpu-node[01-10]
  # cpu          up 3-00:00:00     50   idle cpu-node[01-50]
  ```

### Partition design patterns

Partitions are usually designed around either hardware boundaries or time limits.

- **Hardware-based**: Grouping by physical capability (e.g., GPU nodes, high-memory nodes).
- **Time-based**: Grouping by walltime limits.
- **Overlap strategy**: The same nodes can belong to multiple partitions simultaneously, allowing short jobs to backfill into gaps left by long jobs.

### Access control & constraints

Partitions act as the first line of defense for cluster security and policy.

- **AllowGroups / AllowAccounts**: Restrict access to specific Unix groups or Slurm Accounts.
- **Root-only**: Hidden partitions can be created that only admins can submit to.
- **Resource Limits**: Partitions enforce hard limits like `MaxTime` (maximum walltime) and `MaxNodes` (preventing a single job from requesting the entire partition).
- **Shared access**: The `OverSubscribe` setting controls if jobs can share nodes. `NO` grants exclusive access, while `YES` allows multiple small jobs to run on one node.

### Partition vs. QOS

A common confusion is "When do I use a Partition vs. a QOS?"

- **Partition = Hardware/Physical**: Use Partitions to define *where* the job runs (e.g., "GPU Nodes" or "Big Memory Nodes").
- **QOS = Policy/Service**: Use QOS to define *how* it runs (e.g., "High Priority," "Preemptible," "Double Charge").
- **The Matrix**: Typically, you mix them. A user submits to the `gpu` **Partition** (Hardware) with the `critical` **QOS** (Policy).

## dstack

TBA