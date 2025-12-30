# 15 Kubernetes integration

## Slurm

### Two distinct integration modes

“Slurm + Kubernetes” usually means one (or both) of these. They work differently and the mechanics are not interchangeable.

1) **Slurm deployed on Kubernetes (Slurm daemons are pods)**  
   Kubernetes runs `slurmctld`, `slurmdbd`, login nodes, and `slurmd` as pods.  
   **Jobs are still launched by Slurm as processes**: `slurmctld` allocates resources, then RPCs `slurmd`, which spawns `slurmstepd` and the job’s tasks. Kubernetes does not create a new “job pod” per Slurm job in this mode (this matches Slinky Operator and Soperator).

2) **Kubernetes pods scheduled by Slurm (Slurm-backed Kubernetes scheduler)**  
   Users submit normal Kubernetes objects with `spec.schedulerName: ...`. A Slurm-aware scheduler converts pod resource requests into a **placeholder/external Slurm job** to reserve capacity, waits for Slurm to allocate nodes, then **binds** the pod to a node. The kubelet then launches the pod as usual.

Some stacks can run both modes at once (Slurm jobs and Slurm-scheduled Kubernetes pods), but the execution path is different for each workload type.

### Architecture differences vs traditional Slurm

**Traditional Slurm**
- Controllers are system services on dedicated hosts.
- Compute nodes run `slurmd` on the host; jobs run as host processes.
- Users SSH to login nodes and use `sbatch/srun/salloc`.
- A shared filesystem provides identical paths across nodes.

**Slurm deployed on Kubernetes**
- `slurmctld`/`slurmdbd`/login run as pods.
- Slurm workers are either:
  - **Worker pods as Slurm nodes**: each Slurm node is a long-lived pod running `slurmd`, or
  - **DaemonSet `slurmd`**: one `slurmd` pod per Kubernetes Node (Slurm node ≈ Kubernetes Node).
- Job execution remains Slurm-native:
  - `slurmctld` decides placement/priority.
  - `slurmd` + `slurmstepd` launch tasks as processes in the worker environment.
- User workflow and tooling is unchanged: `sbatch/srun/squeue/sacct`, job scripts, and Slurm environment variables behave the same.

**Kubernetes pods scheduled by Slurm**
- Workloads are Kubernetes pods; Slurm controls *when/where* they are allowed to run.
- Kubernetes still owns pod lifecycle (image pulls, container runtime, restarts, logs, etc.).

### Control plane deployment on Kubernetes

**Controller state**
- `StateSaveLocation` must survive pod restarts (use persistent storage).
- Slurm controller HA is still active/standby at the Slurm level. Kubernetes can restart/relocate controller pods, but you still need a single active controller at a time and shared state for failover.

**Accounting**
- `slurmdbd` connects to MySQL/MariaDB (in-cluster StatefulSet or external).
- Treat the DB like any other production DB: persistence, backups, and HA are separate from Slurm.

**Configuration**
- `slurm.conf` is commonly injected via ConfigMap (or configless mode, depending on the project).
- Updating the mounted file does not automatically apply changes:
  - Apply via `scontrol reconfigure` for controller-side changes, and
  - Restart/roll worker pods for changes that require `slurmd` restart (many operators do this automatically).

**Authentication / identity**
- MUNGE requires the same key on controller, login, and workers (mounted from a Secret).
- UID/GID mapping must be consistent across login and worker environments (directory service / SSSD / centralized user management). If IDs differ, file ownership and MUNGE-authenticated launches break in non-obvious ways.

### Compute node provisioning models

#### A) Slurm nodes are Kubernetes Pods
- One long-lived pod runs `slurmd` and represents one Slurm node (stable identity/hostname is typically important for MPI and for Slurm’s node bookkeeping).
- Scaling up/down adds/removes Slurm nodes by scaling the worker set.
- Scale-down must coordinate with Slurm:
  - mark nodes `DRAIN`,
  - wait for jobs to finish,
  - then terminate pods.

#### B) Slurm nodes map to Kubernetes Nodes
- A DaemonSet runs `slurmd` on every Kubernetes Node.
- Slurm capacity tracks Kubernetes Node capacity (node autoscaling adds/removes Slurm nodes implicitly).
- This is operationally closest to “Slurm installed on every machine,” except the installation vehicle is Kubernetes.

### Job execution mechanics

#### Slurm jobs (sbatch/srun) in Slurm-deployed-on-Kubernetes clusters
Execution path is the same as traditional Slurm, just inside pods/namespaces:

1) User submits a job from a login node (`sbatch`, or interactive `salloc` + `srun`).
2) `slurmctld` schedules the job and allocates nodes/CPUs/memory/GPUs.
3) `slurmctld` contacts `slurmd` on each allocated node.
4) `slurmd` starts `slurmstepd` for the allocation/step.
5) `slurmstepd` launches the job tasks as processes (sets `SLURM_*` env vars, sets up stdin/stdout, runs prolog/epilog if configured, etc.).

**Multi-node jobs**
- Still work as Slurm multi-node jobs: one allocation spans multiple Slurm nodes; `srun` steps can start tasks across those nodes.
- Worker-to-worker connectivity and name resolution must support your runtime (MPI, NCCL, etc.). In pod-based nodes, this usually means stable hostnames and direct pod-to-pod reachability.

**Resource enforcement**
- Decide whether Slurm or Kubernetes is authoritative for cgroups/limits:
  - If Slurm enforces cgroups, worker pods often need host cgroup access (and typically privileged settings).
  - If Kubernetes enforces limits, align Slurm allocations with pod limits/requests so Slurm doesn’t “grant” resources that the container runtime will throttle.

**Filesystem assumptions**
- Job scripts and output paths assume a global namespace. Mount the same shared paths into login and worker pods (or provide an equivalent mechanism such as a shared jail root).

#### Kubernetes pods scheduled by Slurm (Slurm-backed scheduler)
Execution path for Kubernetes-native workloads:

1) User submits a pod/workload with `spec.schedulerName` pointing to the Slurm-aware scheduler.
2) The scheduler reads pod resource requests (CPU/memory/GPUs and any placement constraints it supports).
3) It creates a placeholder/external Slurm job representing that pod’s resources.
4) When Slurm allocates nodes, the scheduler binds the pod to the selected node (assigns the pod to that node).
5) kubelet starts the pod; Kubernetes manages container lifecycle.

**Constraints (implementation-dependent; document what you run)**
- Some implementations schedule pods as whole-node allocations.
- Mixed sharing on the same node (especially GPUs) may be restricted: Kubernetes device plugins and Slurm GRES accounting can conflict, so “Slurm job + Kubernetes GPU pod on the same node” is often not supported.
- In SUNK-mode deployments, GPUs on a node are typically not shareable between Kubernetes pods and Slurm jobs due to allocation/device-plugin conflicts.

### Compute node autoscaling (adding/removing Slurm compute nodes)

Compute autoscaling means changing the number of Slurm compute nodes available to `slurmctld`. The mechanics depend on how compute nodes are represented.

#### Model A: Slurm compute nodes are long-lived `slurmd` Pods
- **Scale up:** increase the worker set replicas (e.g., a NodeSet/StatefulSet replica count). Each new worker pod runs `slurmd` and appears in Slurm as a new node.
  - If the new worker pods cannot be scheduled (cluster has no free CPU/GPU/memory), you must also scale the underlying Kubernetes Nodes (Cluster Autoscaler/Karpenter/platform equivalent); Slurm capacity will not increase until the `slurmd` pods are actually running.
- **Scale down:** drain in Slurm first, then delete/scale down the worker pods. If you delete a worker pod without draining, Slurm will see an abrupt node loss (`DOWN`) and allocations on that node can be interrupted.

**Implemented behavior**
- **Slinky Operator:** exposes a scalable compute NodeSet and documents autoscaling it (including scale-to-zero) via Kubernetes autoscalers (e.g., KEDA/HPA), and drains nodes before terminating pods for scale-in/upgrade.

#### Model B: Slurm compute nodes map 1:1 to Kubernetes Nodes (DaemonSet `slurmd`)
- **Scale up:** add Kubernetes Nodes. The DaemonSet starts a `slurmd` pod on each new node; Slurm gains a compute node.
- **Scale down:** drain the Slurm node, then drain/remove the Kubernetes Node. If the Kubernetes Node is removed first, `slurmd` disappears and Slurm marks the node `DOWN`.

**Implemented behavior**
- **Soperator:** does not provide a built-in “Slurm queue → provision Kubernetes Nodes” loop (“on-demand nodes” is an explicit future improvement). Scaling down removes workers from Kubernetes, but the deleted nodes can remain in the Slurm controller’s memory (visible in `sinfo`) until you remove them using `scontrol`.

#### Slurm-backed Kubernetes scheduler mode (Slurm Bridge / SUNK scheduling pods)
- This mode does not autoscale Slurm compute nodes by itself. It reserves/binds Kubernetes pods using Slurm, but capacity must come from Kubernetes Node scaling and/or (if you also run Slurm-on-Kubernetes) scaling `slurmd` workers as in Model A/B.

**SUNK note**
- SUNK can scale Slurm nodes (as `slurmd` pods), but on CoreWeave CKS the platform’s Node Pool autoscaling explicitly has no SUNK integration, so underlying Kubernetes Node autoscaling is a separate concern.


### Storage and filesystem integration

Slurm expects the same paths on controller, login, and compute.

Minimum requirements:
- Shared paths for user homes and job I/O (e.g., `/home`, `/scratch`, project directories) mounted identically into login and worker pods.
- Persistent storage for controller state (`StateSaveLocation`).
- Per-node `slurmd` spool must exist and be writable (often `emptyDir` is fine unless you require persistence across worker restarts).

Soperator commonly uses a shared “jail” filesystem to provide a consistent root environment for jobs; regardless of approach, keep the global namespace requirement explicit and test it (`pwd`, `which`, shared libs, home dirs).

### Network and communication

**Daemon connectivity**
- Workers must reach the controller endpoint reliably (usually exposed via a Kubernetes Service/DNS name).
- Ensure required ports are reachable (commonly: `slurmctld` 6817, `slurmd` 6818, `slurmdbd` 6819, `slurmrestd` often 6820; all configurable).

**Multi-node workloads**
- Worker-to-worker traffic must be allowed (MPI/NCCL ports, SSH or PMI/PMIx paths if used, etc.).
- If you use NetworkPolicies, explicitly allow:
  - login → controller,
  - worker ↔ controller,
  - worker ↔ worker.

**Low-latency interconnects**
- If you need InfiniBand/RDMA-class performance, plan for host networking and/or specialized CNI/device integration (SR-IOV/Multus). Generic overlay networking may add unacceptable latency.

### Integration implementations

#### Slinky (SchedMD)
- **Slurm Operator**: deploys/manages Slurm on Kubernetes. Typical compute model is worker pods as Slurm nodes (NodeSets) running `slurmd`; Slurm runs jobs natively as processes (Kubernetes does not schedule Slurm jobs).
- **Slurm Bridge**: Slurm-backed scheduling for Kubernetes pods via placeholder/external jobs and node binding (often whole-node oriented).

#### Soperator (Nebius)
- Operator-managed Slurm on Kubernetes that emphasizes a consistent runtime environment across nodes.
- Uses a shared “jail” filesystem mounted into workers/login to present a stable root environment for user workloads.
- Includes operational automation such as GPU health checks and automated node handling.

#### SUNK (CoreWeave)
- Deploys Slurm on Kubernetes and also supports scheduling Kubernetes workloads through Slurm using placeholder/external jobs.
- Commonly documented with restrictions on mixing Slurm and Kubernetes GPU workloads on the same node.

## dstack

### Kubernetes as a backend

In `dstack`, Kubernetes is a **container-based backend**—one option alongside native cloud backends (VM-based) and SSH fleets. `dstack` treats Kubernetes as another compute target for orchestrating dev environments, tasks, and services.

**Backend types in dstack:**

- **VM-based backends** (e.g., AWS, GCP, Azure, Lambda): `dstack` provisions VMs and launches the `dstack-shim` agent inside each VM. The shim controls the VM and starts Docker containers for user jobs. This provides fine-grained control and supports features like blocks, instance volumes, privileged containers, and reusable instances.

- **Container-based backends** (e.g., Kubernetes, RunPod, Vast.ai): `dstack` orchestrates container-based runs directly. With Kubernetes, `dstack` delegates provisioning to Kubernetes and runs containers as pods. Since `dstack` doesn't control the underlying nodes, container-based backends don't support some features available with VM-based backends (see limitations below).

- **SSH fleets**: For on-prem servers or existing infrastructure, `dstack` can create fleets from SSH-accessible hosts without requiring a backend configuration.

### Kubernetes backend configuration

Configure the backend in `~/.dstack/server/config.yml`:

```yaml
projects:
  - name: main
    backends:
      - type: kubernetes
        kubeconfig:
          filename: ~/.kube/config
        proxy_jump:
          hostname: 204.12.171.137
          port: 32000
```

**Proxy jump:** `dstack` requires a node to act as a jump host to proxy SSH traffic into containers. Both the `dstack` server and CLI must be able to reach this node (GPU or CPU-only). `dstack` configures and manages the proxy automatically.

**GPU detection:** For `dstack` to detect GPUs, the cluster must have the appropriate GPU operator pre-installed (e.g., [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html) for NVIDIA GPUs).

<!-- TODO: Add instructions for AMD GPU operator/device plugin setup -->

**Minimum node requirement:** `dstack` requires at least one node to always be provisioned in Kubernetes clusters for two reasons: (1) `dstack` loads offers based on labels available on existing nodes, so if the cluster supports specific GPUs, at least one node with such GPU must be running; (2) at least one node must be available to host the jump pod that proxies SSH traffic into containers (this node can be CPU-only). If using managed Kubernetes with autoscaling, set the node pool's desired count to at least 1.

### Fleet configuration

Create fleets with `placement: cluster` for Kubernetes backends. Since Kubernetes clusters are interconnected by default, you can always set `placement: cluster` for all workload types:

```yaml
type: fleet
name: my-k8s-fleet

placement: cluster
# For Kubernetes, min must be 0 since dstack can't pre-provision VMs
nodes: 0..

backends: [kubernetes]

resources:
  gpu: 1..8
```

### Limitations and unsupported features

Container-based backends (including Kubernetes) delegate node lifecycle management to Kubernetes, so some VM-based features aren't available:

- **Pre-provisioning**: Setting `nodes` range to start above `0` is not supported (only VM-based backends support this)
- **Reusable instances**: Unlike VM-based backends where instances can remain idle and reusable, Kubernetes instances are immediately terminated when idle and returned to the cluster
- **Instance volumes**: Mounting host directories into containers via dstack's instance volumes is not supported
- **Network volumes**: dstack's network volumes feature is not supported (Kubernetes native volumes can still be used)
- **Blocks**: Resource sharing via dstack's blocks abstraction is not supported; Kubernetes handles resource sharing through its native mechanisms (resource requests/limits, node selectors, etc.)
- **Idle duration**: The `idle_duration` setting doesn't apply since `dstack` doesn't control the underlying node lifecycle
- **Auto-scaling**: `dstack` can only run on pre-provisioned Kubernetes nodes. Support for auto-scalable clusters (scale to zero) is coming soon

**Note:** Unlike other container-based backends, Kubernetes supports privileged containers (often required for InfiniBand access).

### When to use Kubernetes backend

Use the `kubernetes` backend if your GPUs already run on Kubernetes and your team depends on Kubernetes tooling. Otherwise, consider VM-based backends for better provisioning control or SSH fleets for on-prem simplicity.
