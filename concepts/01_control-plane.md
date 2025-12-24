# 1 Control plane, state, and scaling

## Slurm

### Slurm architecture

In a large-scale deployment, the architecture is divided into the **control plane** (management) and the **compute plane** (execution).

**Complete cluster infrastructure:**

A typical HPC cluster consists of three types of servers, managed by cluster administrators:

- **Controller nodes**: Run Slurm control plane daemons (`slurmctld`, `slurmdbd`, `slurmrestd`). Set up and managed by cluster admins. Not part of the compute pool. **Count**: Typically 2 nodes (1 primary + 1 backup) for high availability, regardless of cluster size. See "High availability & scaling" section below for details.
- **Login nodes**: User access servers where users SSH to submit jobs. Set up by cluster admins, separate from Slurm. Have Slurm client tools installed (`sbatch`, `srun`, `squeue`, etc.) to communicate with the controller. **Not listed in `slurm.conf`** as compute nodes - they're outside the Slurm-managed compute pool. Shared by all users, typically with resource limits enforced by the OS. **Count**: Determined by user load (concurrent SSH sessions, job submission rate). Small clusters may have 1-2 login nodes; large clusters with hundreds of users may have 10-20+ for load balancing and redundancy.
- **Compute nodes**: Servers that execute jobs. Listed in `slurm.conf` with their resources. Run `slurmd` daemon. Managed by Slurm scheduler. **Count**: Varies from tens to thousands depending on cluster size and workload requirements.

**Setup and management:**
- Cluster administrators provision and configure all three types of servers
- Login nodes require Slurm client tools and network access to the controller
- Login nodes are not managed by Slurm (not in `slurm.conf`), but they must have Slurm client tools to submit jobs
- Compute nodes are managed by Slurm (listed in `slurm.conf`) and run `slurmd` to receive job assignments

The **control plane** consists of:
- `slurmctld`: The central controller (orchestrator). It manages the job queue, monitors node states, and performs scheduling calculations.
- `slurmdbd`: The database daemon. It manages the interface to the SQL backend (MySQL/MariaDB). It stores **critical policy data** (User Associations, QOS limits, Fairshare usage) alongside historical job records.
- `slurmrestd` (Optional but standard): The REST API daemon, used for integrating web portals, CI/CD pipelines, or external workflow managers.

> **Note**: `slurmd` runs on the compute nodes (the compute plane). It acts as a remote execution agent, receiving instructions from the control plane to launch tasks.

### State persistence

Slurm separates its "hot" scheduler state from its "cold" accounting data.

- **Scheduler state (`slurmctld`)**: The controller persists active job queues, reservations, and node states to a **flat-file directory** specified by `StateSaveLocation`. It does *not* use the SQL database for this. Even in massive clusters, scheduler recovery relies on these files.
- **Example: slurm.conf configuration** (located on controller node):
  ```bash
  # StateSaveLocation must be on shared filesystem accessible by both controllers
  StateSaveLocation=/shared/slurm/state
  ```
- **Accounting & policy (`slurmdbd`)**: The SQL database is the source of truth for limits, users, and history. If the DB is down, you cannot enforce new limits or add users, but the active scheduler can typically continue running jobs using cached policies.
- **Job output**: Job standard output and error logs (`stdout`/`stderr`) are written to files. Slurm routes job output to files specified via `--output` and `--error` directives or default paths. The controller tracks job output file locations but does not store the output content itself.

### High availability & scaling

Slurm uses an **active-passive** architecture for the controller.

- **Controller count**: Regardless of cluster size (400 or 4,000 nodes), the standard deployment is **2 nodes** (1 Primary + 1 Backup). Only one instance manages the cluster state at a time.
- **Storage requirements**: The `StateSaveLocation` requires a shared filesystem accessible by both controllers.
- **Vertical scaling**: Since the scheduler is single-threaded, performance relies on **CPU frequency (GHz)** rather than core count. Run `slurmctld` on a dedicated server to avoid resource contention.
- **Topology & fan-out**: Configure `TreeWidth` in `slurm.conf` to create a communication hierarchy. This forces `slurmd` daemons to act as **message relays**, forwarding instructions to their peers to prevent the primary controller from choking on thousands of simultaneous connections.
- **Example: slurm.conf with TreeWidth** (located on controller node):
  ```bash
  # TreeWidth=32 means each node can relay to up to 32 peers
  # Reduces direct connections to controller from 4000 to ~125
  TreeWidth=32
  ```

### Network communication and message passing

Slurm daemons communicate over the network using a message-based protocol with authentication.

- **Message passing**: `slurmctld` sends instructions to `slurmd` daemons to launch jobs, update node state, or perform administrative tasks. Messages are sent asynchronously and daemons respond with status updates.
- **Port requirements**: Slurm daemons communicate over specific network ports that must be open between controller and compute nodes, and login nodes need access to the controller port.
- **MUNGE authentication**: All inter-daemon communication is authenticated using MUNGE tokens. This prevents unauthorized nodes from joining the cluster or spoofing messages. MUNGE tokens have a limited lifetime and are regenerated periodically.
- **Network topology**: The `TreeWidth` parameter creates a hierarchical message routing structure. Instead of all `slurmd` daemons connecting directly to `slurmctld`, nodes form a tree where intermediate nodes relay messages. This reduces connection overhead on the controller.
- **Message timeout tuning**: Configure `MessageTimeout` to handle network latency. If a node doesn't respond within this time, it may be marked `DOWN`. For clusters with high network latency or slow filesystems, increase this value. `SlurmdTimeout` controls how long the controller waits for node heartbeats before marking nodes as unresponsive.
- **Example: slurm.conf timeout configuration** (located on controller node):
  ```bash
  # Increase timeouts for high-latency networks
  MessageTimeout=60
  SlurmdTimeout=30
  ```
- **Example: checking controller state** (from login node):
  ```bash
  # Check if controller is running
  scontrol ping
  # SLURMCTLD: UP
  
  # Show controller status
  scontrol show config | grep StateSaveLocation
  ```

## dstack

TBA