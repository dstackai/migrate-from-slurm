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

### dstack architecture

In dstack, the architecture is divided into the **control plane** (management via `dstack-server`) and the **compute plane** (execution via fleets and instances).

dstack is designed for AI workloads—including development, training, and inference—and is cloud-native from day one. While Slurm targets AI/HPC researchers and academia, often on on-premises clusters, dstack targets AI researchers and ML engineers, often using GPU cloud capacity.

**Key approach differences:**

- **Cloud-native design**: dstack is cloud-native from day zero, with native integrations for top GPU cloud providers. Slurm focuses on HPC and job scheduling; it can be cloud-native only when run on top of Kubernetes, and this integration isn't fully native.
- **Container-native**: dstack is container-native by design, while Slurm supports containers as one option among many.
- **Out-of-the-box support**: dstack aims for out-of-the-box support for clouds and GPU vendors, simplifying setup and configuration.

**Commonalities:**

Both dstack and Slurm are open-source and aim to be vendor-agnostic standards. However, since Slurm was acquired by NVIDIA, it is no longer fully vendor-agnostic.

**Complete infrastructure:**

The dstack platform consists of:

- **`dstack-server`**: The central control plane that combines the functionality of Slurm's `slurmctld`, `slurmdbd`, and `slurmrestd` into a single service. It provides an HTTP API for submitting runs and managing users, projects, backends, fleets, secrets, and gateways.
- **CLI**: The command-line interface that communicates with `dstack-server`. Unlike Slurm's login nodes, the CLI can be used from anywhere (user's laptop, CI/CD systems, etc.) and must have network access to the server. It can attach to running jobs to forward ports, stream logs, and interact with containers.

<!-- TODO: Explain CLI configuration (pointing to server URL, authentication token), projects concept, and how these relate to authentication and cluster management. This will be expanded in authentication and security section and cluster node management section. -->
- **Fleets**: Pools of instances (VMs or containers) that run actual workloads. Fleets can be provisioned via backends (native cloud integrations or Kubernetes) or created from existing on-premises servers via SSH.
- **Instances**: Individual compute nodes (VMs or containers) within fleets that execute jobs.

<!-- TODO: Add explicit terminology mapping between dstack concepts (fleet, instance) and Slurm concepts (compute nodes, partitions) to help Slurm admins understand the relationship. -->

**Setup and management:**

1. Run `dstack-server` on any machine with access to your cloud providers or clusters.
2. Configure backends (if using clouds or Kubernetes) via `~/.dstack/server/config.yml` or the UI.
3. Create fleets using `dstack apply` with fleet configuration files.
4. Submit runs via CLI using `dstack apply` with run configuration files.

### State persistence

dstack stores its state in a database, supporting both SQLite and PostgreSQL.

- **Default (SQLite)**: By default, dstack uses SQLite stored locally in `~/.dstack/server`. With SQLite, you can run at most one server replica. SQLite is suitable for single-server deployments.
- **PostgreSQL**: For production deployments requiring horizontal scaling, dstack can use PostgreSQL. Multiple server replicas can share the same PostgreSQL database, enabling high availability and load distribution. Set the `DSTACK_DATABASE_URL` environment variable to use PostgreSQL.

  **Example: Starting server with PostgreSQL**:
  ```bash
  export DSTACK_DATABASE_URL=postgresql+asyncpg://user:password@db-host:5432/dstack
  dstack server
  ```

**Job logs storage:**

- **Default**: Job logs are stored locally in `~/.dstack/server/projects/<project_name>/logs`.
- **External services**: For multi-replica server deployments, logs must be stored externally. dstack supports AWS CloudWatch and GCP Logging for centralized log storage. This is required when running multiple server replicas, as local file storage isn't shared across replicas.

  **Example: Starting server with AWS CloudWatch**:
  ```bash
  export DSTACK_SERVER_CLOUDWATCH_LOG_GROUP=dstack-logs
  export DSTACK_SERVER_CLOUDWATCH_LOG_REGION=us-east-1
  dstack server
  ```

  **Example: Starting server with GCP Logging**:
  ```bash
  export DSTACK_SERVER_GCP_LOGGING_PROJECT=my-gcp-project
  dstack server
  ```

**File storage:**

When using files or repos, dstack uploads local files and diffs to the server. By default, files are stored in the database with a 2MB limit per upload. You can configure S3 or GCS for file storage to support larger uploads and multi-replica deployments.

  **Example: Starting server with S3 file storage**:
  ```bash
  export DSTACK_SERVER_S3_BUCKET=my-dstack-files
  export DSTACK_SERVER_S3_BUCKET_REGION=us-east-1
  dstack server
  ```

  **Example: Starting server with GCS file storage**:
  ```bash
  export DSTACK_SERVER_GCS_BUCKET=my-dstack-files
  dstack server
  ```

### High availability & scaling

dstack supports horizontal scaling of the server through multiple replicas.

- **Single replica (SQLite)**: The default SQLite setup supports one server replica. Suitable for small deployments.
- **Multiple replicas (PostgreSQL)**: With PostgreSQL, you can run multiple `dstack-server` replicas behind a load balancer. All replicas share the same database, enabling horizontal scaling and high availability.
- **Scheduling loop scaling**: The scheduling loop can be scaled by increasing the `DSTACK_SERVER_BACKGROUND_PROCESSING_FACTOR` environment variable, which increases the number of concurrent background workers processing jobs. This must be balanced with database connection pool settings (`DSTACK_DB_POOL_SIZE` and `DSTACK_DB_MAX_OVERFLOW`). Scaling constraints are discussed in more detail in the job execution lifecycle section.

**Server limits:**

A single `dstack-server` replica can support:
- Up to 150 active runs
- Up to 150 active jobs
- Up to 150 active instances

Exceeding these limits may affect performance. For larger deployments, use PostgreSQL with multiple replicas.

### Network communication and SSH tunnels

dstack uses SSH tunnels for secure communication between components.

- **Server-to-instance communication**: Both `dstack-server` and the CLI must have network access to where fleets are provisioned, even if it's a private VPC. The server establishes SSH connections to fleet instances (hosts) to communicate with agents running on them.
- **CLI-to-container communication**: When the CLI attaches to a run (explicitly with `dstack attach` or implicitly with `dstack apply`), it establishes an SSH connection inside the container (`dstack-runner`). This may use a jump host (the instance host or a Kubernetes host). The CLI can forward ports, stream logs, and interact with the running container.
- **SSH server requirement**: Each fleet instance must have an SSH server running. The SSH server provides a secure channel for communication with runner APIs and port forwarding without listening on `0.0.0.0`.

### Agents: dstack-shim and dstack-runner

dstack uses two types of agents to manage job execution:

**`dstack-shim`** (on fleet instances/hosts):

- **Purpose**: `dstack-shim` is an agent that runs on every fleet instance (host) when using VM-based backends or bare-metal servers. It emulates a container-only environment on VMs.
- **Responsibilities**: 
  - Pulling Docker images (public or private)
  - Configuring Docker containers (mounts, GPU forwarding, entrypoint, etc.)
  - Running Docker containers
  - Detecting hardware (GPUs, CPUs, memory)
  - Monitoring container health
  - Terminating containers on signal from `dstack-server`
- **Communication**: All communication between `dstack-server` and `dstack-shim` happens via REST API through an SSH tunnel. `dstack-shim` doesn't collect logs itself.
- **Deployment**: `dstack-shim` is typically started via `cloud-init` user-data scripts when VMs are provisioned.

**`dstack-runner`** (inside containers):

- **Purpose**: `dstack-runner` is a cloud-agnostic component that runs as the entrypoint of Docker containers. It manages the actual job execution.
- **Responsibilities**:
  - Setting environment variables and secrets
  - Executing user commands
  - Collecting and reporting logs
  - Reporting job status
  - Terminating the job on signal from `dstack-server`
- **Lifecycle**: 
  1. Wait for job spec submission
  2. Wait for code (tarball or git diff)
  3. Prepare the repo (clone git repo and apply diff, or extract archive)
  4. Run commands from the job spec
  5. Wait until all logs are read by server and CLI, or exit after timeout
- **Communication**: Communication with `dstack-server` happens via REST API through an SSH tunnel. `dstack-runner` collects job logs and serves them via HTTP to the server and via WebSocket to the CLI.

**Container-based backends:**

For container-based backends (e.g., Kubernetes or cloud providers that only provide container interfaces), `dstack-runner` runs directly inside the container without `dstack-shim`. In these cases, `dstack-server` communicates directly with `dstack-runner` through the container interface.

### Authentication

dstack has built-in authentication using tokens. Each user is issued a token upon creation, which must be used for authentication when:
- Logging into the control plane UI
- Using the CLI
- Using the API

Token-based authentication is discussed in more detail in the authentication and security section.