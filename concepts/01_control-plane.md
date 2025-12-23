# 1 Control plane, state, and scaling

## Slurm

### Slurm architecture

In a large-scale deployment, the architecture is divided into the **control plane** (management) and the **compute plane** (execution).

The **control plane** consists of:
- `slurmctld`: The central controller (orchestrator). It manages the job queue, monitors node states, and performs scheduling calculations.
- `slurmdbd`: The database daemon. It manages the interface to the SQL backend (MySQL/MariaDB). It stores **critical policy data** (User Associations, QOS limits, Fairshare usage) alongside historical job records.
- `slurmrestd` (Optional but standard): The REST API daemon, used for integrating web portals, CI/CD pipelines, or external workflow managers.

> **Note**: `slurmd` runs on the compute nodes (the compute plane). It acts as a remote execution agent, receiving instructions from the control plane to launch tasks.

### State persistence

Slurm separates its "hot" scheduler state from its "cold" accounting data.

- **Scheduler state (`slurmctld`)**: The controller persists active job queues, reservations, and node states to a **flat-file directory** specified by `StateSaveLocation`. It does *not* use the SQL database for this. Even in massive clusters, scheduler recovery relies on these files.
- **Accounting & policy (`slurmdbd`)**: The SQL database is the source of truth for limits, users, and history. If the DB is down, you cannot enforce new limits or add users, but the active scheduler can typically continue running jobs using cached policies.
- **Job output**: Job standard output and error logs (`stdout`/`stderr`) are written to files. Slurm routes job output to files specified via `--output` and `--error` directives or default paths. The controller tracks job output file locations but does not store the output content itself.

### High availability & scaling

Slurm uses an **active-passive** architecture for the controller.

- **Controller count**: Regardless of cluster size (400 or 4,000 nodes), the standard deployment is **2 nodes** (1 Primary + 1 Backup). Only one instance manages the cluster state at a time.
- **Storage requirements**: The `StateSaveLocation` requires a shared filesystem accessible by both controllers.
- **Vertical scaling**: Since the scheduler is single-threaded, performance relies on **CPU frequency (GHz)** rather than core count. Run `slurmctld` on a dedicated server to avoid resource contention.
- **Topology & fan-out**: Configure `TreeWidth` in `slurm.conf` to create a communication hierarchy. This forces `slurmd` daemons to act as **message relays**, forwarding instructions to their peers to prevent the primary controller from choking on thousands of simultaneous connections.

### Network communication and message passing

Slurm daemons communicate over the network using a message-based protocol with authentication.

- **Message passing**: `slurmctld` sends instructions to `slurmd` daemons to launch jobs, update node state, or perform administrative tasks. Messages are sent asynchronously and daemons respond with status updates.
- **Port requirements**: Slurm daemons communicate over specific network ports that must be open between controller and compute nodes, and login nodes need access to the controller port.
- **MUNGE authentication**: All inter-daemon communication is authenticated using MUNGE tokens. This prevents unauthorized nodes from joining the cluster or spoofing messages. MUNGE tokens have a limited lifetime and are regenerated periodically.
- **Network topology**: The `TreeWidth` parameter creates a hierarchical message routing structure. Instead of all `slurmd` daemons connecting directly to `slurmctld`, nodes form a tree where intermediate nodes relay messages. This reduces connection overhead on the controller.
- **Message timeout tuning**: Configure `MessageTimeout` to handle network latency. If a node doesn't respond within this time, it may be marked `DOWN`. For clusters with high network latency or slow filesystems, increase this value. `SlurmdTimeout` controls how long the controller waits for node heartbeats before marking nodes as unresponsive.

## dstack

TBA