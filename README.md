# Migrate from Slurm

This repo documents key Slurm's concepts and how they map to `dstack`'s equvivalents. Once the mapping is finalized, this documentation will be integrated into the `dstack` [docs](https://dstack.ai/docs/).

## Feedback

We welcome feedback on the Slurm documentation and mappings. You can contribute in the following ways:

- **Create an issue**: Open a [GitHub issue](https://github.com/dstackai/migrate-from-slurm/issues) to report errors, suggest improvements, or ask questions about the documentation
- **Submit a PR**: Submit a [pull request](https://github.com/dstackai/migrate-from-slurm/pulls) with corrections, additions, or improvements to the documentation
- **Contact directly**: Reach out to [@peterschmidt85](https://github.com/peterschmidt85) on GitHub for direct feedback or questions

## Work in progress

- [x] [1 Control plane, state, and scaling](concepts/01_control-plane.md)
- [x] [2 Queueing, prioritization, and scheduling mechanics](concepts/02_queueing-prioritization-scheduling.md)
- [x] [3 Resource model & generic resources (TRES/GRES)](concepts/03_resource-model.md)
- [x] [4 Job submission, allocation, and execution](concepts/04_job-execution-lifecycle.md)
- [ ] [5 Resource enforcement, containment, and limits](concepts/05_resource-enforcement.md)
- [ ] [6 Accounts, QOS, and accounting pipeline](concepts/06_policy-accounting.md)
- [ ] [7 Job arrays & dependencies](concepts/07_job-organization.md)
- [ ] [8 Cluster node management, health, and lifecycle](concepts/08_cluster-node-management.md)
- [ ] [9 Filesystems and data access](concepts/09_file-systems.md)
- [ ] [10 Fault tolerance, checkpointing, and job recovery](concepts/10_fault-tolerance.md)
- [ ] [11 GPU health monitoring and device constraints](concepts/11_gpu-health-monitoring.md)
- [ ] [12 Partition design, node grouping, and queue layout](concepts/12-partition-design.md)
- [ ] [13 Authentication and security](concepts/13_authentication-security.md)
- [ ] [14 Reservations](concepts/14_reservations.md)
- [ ] [15 Monitoring and observability](concepts/15_monitoring-observability.md)
- [ ] [16 Kubernetes integration](concepts/16_kubernetes.md)

## Project structure

- `concepts/` - Documentation mapping Slurm concepts to their `dstack` equivalents
