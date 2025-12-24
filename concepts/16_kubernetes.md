# 16 Kubernetes integration

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

TBA
