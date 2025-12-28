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
- [x] [3 Resource model, generic resources (TRES/GRES), and enforcement](concepts/03_resource-model.md)
- [x] [4 Job submission, allocation, and execution](concepts/04_job-execution-lifecycle.md)
- [x] [5 Accounts, QOS, and accounting pipeline](concepts/05_policy-accounting.md)
- [ ] [6 Job arrays & dependencies](concepts/06_job-organization.md)
- [ ] [7 Cluster node management, health, and lifecycle](concepts/07_cluster-node-management.md)
- [ ] [8 Filesystems and data access](concepts/08_file-systems.md)
- [ ] [9 Fault tolerance, checkpointing, and job recovery](concepts/09_fault-tolerance.md)
- [ ] [10 GPU health monitoring and device constraints](concepts/10_gpu-health-monitoring.md)
- [ ] [11 Partition design, node grouping, and queue layout](concepts/11_partition-design.md)
- [ ] [12 Authentication and security](concepts/12_authentication-security.md)
- [ ] [13 Reservations](concepts/13_reservations.md)
- [ ] [14 Monitoring and observability](concepts/14_monitoring-observability.md)
- [ ] [15 Kubernetes integration](concepts/15_kubernetes.md)

## Project structure

- `concepts/` - Documentation mapping Slurm concepts to their `dstack` equivalents
