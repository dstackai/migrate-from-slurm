# 7 Cluster node management, health, and lifecycle

## Slurm

### Node states & lifecycle

Nodes act as the reusable vessels for jobs. Slurm tracks their state to ensure scheduling integrity.

- **Standard states**: Common states include `IDLE` (Empty), `ALLOCATED` (Busy), `MIXED` (Some cores busy), and `DOWN` (Offline).
- **Example: checking node states** (from login node):
  ```bash
  # Show all nodes and their states
  sinfo
  # PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
  # gpu          up   infinite      4  idle gpu-node[01-04]
  # cpu          up   infinite      2 alloc cpu-node[01-02]
  
  # Show detailed node information
  scontrol show node gpu-node01
  ```
- **DRAIN state**: The administrative state used for maintenance. A draining node allows currently running jobs to finish but refuses new allocations.
- **Example: draining a node** (from controller or login node, admin only):
  ```bash
  # Drain node for maintenance
  scontrol update NodeName=gpu-node01 State=DRAIN Reason="Maintenance"
  
  # Undrain node after maintenance
  scontrol update NodeName=gpu-node01 State=RESUME
  ```
- **FAIL state**: Set automatically when a health check detects a hardware fault, removing the node from the pool.
- **FUTURE state**: Used in large clusters. The node is defined in the config but hasn't "joined" yet, keeping database overhead low until the node actually boots.
- **Lifecycle flow**: Nodes typically cycle from `IDLE` → `ALLOCATED` (Job Starts) → `IDLE` (Job Ends). `slurmd` generally does *not* reboot nodes between jobs unless explicitly configured.

### Health checks & failure handling

In large clusters, reactive failure handling (waiting for a node to crash) is insufficient. Proactive health checking is mandatory.

- **Heartbeats**: `slurmctld` expects a "ping" from every node every few seconds (`SlurmdTimeout`). If missed, the node is marked `DOWN`, and jobs are requeued (if `--requeue` is set).
- **Proactive health checks (NHC)**: Large clusters configure the `HealthCheckProgram` parameter in `slurm.conf` to specify a script that runs periodically to detect soft failures.
- **Example: health check script configuration** (in `slurm.conf` on controller node):
  ```bash
  # Health check script located on shared filesystem
  HealthCheckProgram=/shared/scripts/health_check.sh
  HealthCheckInterval=300  # Run every 5 minutes
  ```
- **Example: health check script** (located on shared filesystem, executed on compute nodes):
  ```bash
  #!/bin/bash
  # Health check script on compute node
  # Checks disk space, memory, GPU health
  
  # Check disk space
  if [ $(df /tmp | tail -1 | awk '{print $5}' | sed 's/%//') -gt 90 ]; then
      echo "Disk space critical" >&2
      exit 1
  fi
  
  # Check GPU (if node has GPUs)
  if command -v nvidia-smi &> /dev/null; then
      nvidia-smi --query-gpu=health.status --format=csv,noheader | grep -q "Healthy"
      if [ $? -ne 0 ]; then
          echo "GPU health check failed" >&2
          exit 1
      fi
  fi
  ```
- **Soft failure detection**: The script checks for non-fatal issues that could affect job execution.
- **Automated action**: If the script returns an error, Slurm automatically moves the node to `DRAIN`. This prevents *new* jobs from starting on bad hardware while allowing running jobs to finish.

### Power saving & elastic scaling

Slurm manages dynamic clusters (Cloud Bursting) or Green computing via its Power Saving subsystem.

- **Resume mechanism**: If pending jobs require more resources than available, Slurm calls the resume program (configured via `ResumeProgram` in `slurm.conf`) to boot idle nodes (Wake-on-LAN or Cloud API).
- **Suspend mechanism**: If a node remains `IDLE` for a configured time period, Slurm calls the suspend program (configured via `SuspendProgram` in `slurm.conf`) to power it off.

### Configuration & configless mode

Keeping `slurm.conf` synchronized across 4,000 nodes is a major operational challenge.

- **Synchronization rule**: `slurmd` (compute) and `slurmctld` (controller) must agree on the cluster configuration. Mismatches lead to "split-brain" scheduling behavior.
- **Configless Slurm**: The modern standard for large clusters.
- **Mechanism**: Compute nodes do *not* store a local `slurm.conf`. On startup, they connect to the controller, download the configuration into memory, and start. This ensures 100% consistency with zero manual file copying.
- **Example: enabling configless mode** (in `slurm.conf` on controller node):
  ```bash
  # Enable configless mode
  SlurmdParameters=configless
  ```

### Upgrade constraints

*Note: Detailed upgrade steps belong in a separate Operations Playbook. This section defines the architectural constraints.*

- **Compatibility**: The Slurm Controller is backward compatible (can talk to old nodes), but old Controllers cannot talk to new nodes.
- **Strict Order of Operations**:
    1.  **Database (`slurmdbd`)**: Must be upgraded first.
    2.  **Controller (`slurmctld`)**: Must be upgraded second.
    3.  **Nodes (`slurmd`)**: Must be upgraded last.
- **Rolling Upgrades**: Because the controller can manage older nodes, you can upgrade the compute nodes in batches (Rolling Upgrade) while the cluster is running, minimizing downtime.

### Filesystem dependencies

Slurm relies on the filesystem for basic operational stability.

- **Global namespace**: Slurm assumes a "Single System Image." Paths must be identical across the controller and all compute nodes.
- **Latency & Timeouts**: On large clusters, `slurmd` can become unresponsive if the shared filesystem stalls on I/O. Tune `MessageTimeout` in `slurm.conf` to be higher than your filesystem's maximum expected latency spike. If the timeout is too low, Slurm will mistakenly mark healthy nodes as `DOWN` simply because they are waiting on disk I/O.

## dstack

### Fleets

Fleets in `dstack` act as pools of instances and templates for how those instances are provisioned. Unlike Slurm's static node configuration, fleets provide a dynamic abstraction that can span cloud providers or on-premises infrastructure.

`dstack` supports two types of fleets:

- **Backend fleets**: Dynamically provisioned through configured backends (cloud providers or Kubernetes). You specify resource requirements and a `nodes` range; `dstack` provisions matching instances.
  ```yaml
  type: fleet
  name: my-fleet
  nodes: 0..2  # min..max instances
  resources:
    gpu: A100:80GB:2
  ```

- **SSH fleets**: Use existing on-premises servers. List hostnames/IPs; `dstack` connects, installs dependencies, and registers them.
  ```yaml
  type: fleet
  name: my-fleet
  ssh_config:
    user: ubuntu
    identity_file: ~/.ssh/id_rsa
    hosts:
      - 192.168.1.10
      - 192.168.1.11
  ```

- **Node ranges and scaling**: Specify `nodes: min..max` to control fleet size. Instances scale up when jobs need resources and scale down when idle. The `idle_duration` parameter controls how long an instance must remain idle before auto-termination (default: 3 days). If `min > 0`, that many instances are pre-provisioned.

### Fleet management

- Create/update fleet: `dstack apply -f fleet.dstack.yml`
- List fleets: `dstack fleet` or `dstack fleet list`
- Delete fleet: `dstack delete -f fleet.dstack.yml` or `dstack fleet delete <name>`

### Fleet instance states

Fleet instances have several states:

- **`idle`**: Available and ready for jobs
- **`busy`**: Running one or more jobs
- **`unreachable`**: Cannot be contacted; excluded from scheduling

### Placement: cluster

For distributed workloads that require fast inter-node communication (such as distributed training), `dstack` supports fleets with `placement: cluster`. This configuration ensures that all instances within the fleet are provisioned with optimal inter-node connectivity.

- **Backend fleets**: `dstack` auto-configures high-performance networking (e.g., EFA on AWS, GPUDirect/RoCE on GCP). Supported on major cloud backends and [Kubernetes](15_kubernetes.md).

- **SSH fleets**: Indicates hosts are interconnected. Requires pre-configured interconnect drivers (e.g., InfiniBand).

### Blocks

Blocks divide instances into resource slices to enable concurrent jobs. Resources (GPU, CPU, memory) split evenly; disk is shared. Set `blocks: auto` to match GPU count.

- **Backend fleets**: Configure at fleet level
- **SSH fleets**: Configure per host
- **Distributed tasks**: Require all blocks on each node

### Auto-update

`dstack-shim` and `dstack-runner` support automatic updates. The `dstack` server periodically checks the installed versions of these components on fleet instances and automatically updates them to match the server's expected version. This ensures that fleet instances remain compatible with the server without manual intervention. The update process happens in the background and does not interrupt running jobs.

### Provisioning and hardware detection

- **Backend fleets**: Instances provisioned to match specified requirements. Hardware known in advance from cloud instance types.

- **SSH fleets**: Existing servers discovered and registered. Hardware detected from what's present on each host.

### Maintenance

`dstack` has no maintenance concept (no equivalent to Slurm's `DRAIN` state that allows running jobs to finish while refusing new allocations).

- **Backend fleets**: Delete/terminate the instance; `dstack` provisions a replacement if needed.
- **SSH fleets**: Remove host from fleet config, perform maintenance, then re-add.

### Health monitoring

`dstack` monitors instance health through multiple mechanisms:

- **Instance connectivity**: Heartbeats to the `dstack` server. Instances that cannot be contacted are marked as `unreachable` and automatically excluded from scheduling.

**GPU health**: `dstack` integrates with NVIDIA DCGM (Data Center GPU Manager) via `dstack-shim` to detect GPU health. This provides passive health checks that continuously monitor GPU hardware for reliability issues. The health status is displayed in fleet listings:

- **`idle`**: No issues detected, instance is healthy and ready for workloads.
- **`idle (warning)`**: Non-fatal issues detected (e.g., correctable ECC errors). The instance remains usable but should be monitored.
- **`idle (failure)`**: Fatal issues detected. The instance is automatically excluded from scheduling.

GPU health checks work on supported cloud backends and SSH fleets where DCGM is installed.

**Metrics**: `dstack` automatically collects metrics such as CPU usage, GPU utilization, memory consumption, and GPU memory usage. For detailed information on GPU health monitoring and general metrics, see the [GPU health monitoring](10_gpu-health-monitoring.md) and [monitoring and observability](14_monitoring-observability.md) guides.