# Out of scope

## Slurm

This document lists topics that are not covered in the current concept guides. These topics may be covered in separate operations guides, advanced documentation, or specialized resources.

### Plugin system architecture

- **Plugin types**: Scheduler plugins (`sched/backfill`, `sched/builtin`), accounting plugins, authentication plugins, job submission plugins, etc.
- **Plugin configuration**: How to configure and enable plugins in `slurm.conf`
- **Custom plugin development**: Writing custom plugins (advanced topic requiring C programming)
- **Plugin ecosystem**: Common third-party plugins and their use cases

**Note**: Basic plugin usage (e.g., `SchedulerType=sched/backfill`) is mentioned in relevant guides, but detailed plugin architecture and development are not covered.

### Performance tuning

- **Scheduler tuning**: Optimizing `SchedulerParameters` for large clusters, tuning backfill parameters, adjusting priority weights
- **Database tuning**: MySQL/MariaDB optimization for `slurmdbd`, buffer pool sizing, query optimization, archiving strategies
- **Network tuning**: Optimizing message passing performance, tuning `TreeWidth`, adjusting timeout values
- **Filesystem tuning**: Impact of filesystem performance on Slurm, optimizing shared storage for job I/O

**Note**: Basic tuning parameters are mentioned where relevant, but comprehensive performance optimization is an operational topic.

### Multi-cluster and federation

- **Federated clusters**: Connecting multiple separate Slurm clusters (different clusters, possibly different locations/organizations)
- **Multi-cluster setup**: Cluster naming, federation configuration, cross-cluster authentication
- **Cross-cluster job submission**: How jobs are routed between clusters, cluster selection policies

**Note**: This is distinct from multi-node/parallel jobs (covered in guide 2), which run across multiple nodes within a single cluster.

### Cloud and hybrid deployments

- **Hybrid clusters**: Mixing on-premise and cloud nodes in a single cluster, managing heterogeneous infrastructure across environments, cross-cloud deployments (managing nodes across multiple cloud providers simultaneously)
- **Cloud API specifics**: Detailed cloud API configuration and authentication for specific providers (AWS, Azure, GCP, neo-clouds), provider-specific instance types and pricing models
- **Network connectivity for hybrid**: VPN configuration, firewall rules, MUNGE key distribution across network boundaries for hybrid deployments

**Note**: Basic dynamic provisioning mechanisms (Resume/Suspend programs) are covered in guide 16, but hybrid cluster architectures and provider-specific API details are operational topics.

### Advanced operational topics

- **Upgrade procedures**: Detailed steps for upgrading Slurm versions, rolling upgrades, compatibility considerations
- **Disaster recovery**: Backup strategies for `StateSaveLocation`, database backups, recovery procedures
- **Backup strategies**: Regular backup schedules, backup validation, restore procedures
- **Advanced troubleshooting**: Deep debugging techniques, performance profiling, custom monitoring setups

**Note**: Basic operational concepts (health checks, node management) are covered, but detailed operational procedures are outside the scope of concept guides.

### Advanced scheduling features

- **Gang scheduling (time-slicing)**: Scheduling mechanism that suspends and resumes jobs at regular intervals to time-share CPUs. Rarely used in production clusters as job packing with `OverSubscribe=YES` is more efficient. Enabled via `SchedulerType=sched/gang` in `slurm.conf`.

### Advanced GRES configuration

- **MIG (Multi-Instance GPU) configuration**: Detailed configuration of NVIDIA MIG slices in Slurm, including how to define MIG instances as distinct GRES types in `gres.conf` and manage MIG slices for multiple jobs sharing a single physical GPU.
- **FPGA GRES configuration**: Detailed configuration of FPGA devices as GRES, including device file mapping, FPGA-specific constraints, and management of FPGA resources.

### Advanced partition features

- **Routing queues (Job Submit Plugin)**: Implementation of routing partitions using custom Job Submit Plugins (Lua scripts) to automatically route jobs to specific partitions based on resource requests. Requires custom plugin development and is an advanced configuration topic.

### External checkpoint libraries

- **SCR (Scalable Checkpoint/Restart)**: Third-party checkpoint library that caches checkpoints in local RAM/NVMe and flushes to shared filesystem asynchronously. Not a Slurm feature but may be used in conjunction with Slurm for distributed job checkpointing.

### Command-line syntax and usage

- **Command syntax examples**: Detailed command-line syntax, flags, and usage examples for Slurm commands (`sbatch`, `srun`, `salloc`, `squeue`, `sinfo`, `sacct`, `scontrol`, etc.)
- **Parameter values and tuning**: Specific parameter values, tuning details, and configuration examples
- **Operational procedures**: Firewall configuration, log rotation setup, upgrade procedures, and other operational tasks

### Third-party tools and integrations

- **Monitoring tools**: Third-party monitoring tools (Ganglia, Grafana, Slurm Web, Open OnDemand) and their specific configurations
- **Container runtime specifics**: Container runtime syntax and configuration details (Enroot, Pyxis, Apptainer)
- **Database backend specifics**: Database backend tuning, buffer pool sizing, and optimization (MySQL/MariaDB specific configurations)

## dstack

TBA

