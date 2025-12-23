# 15 Monitoring and observability

## Slurm

### Slurm commands

Slurm provides command-line tools for monitoring cluster state, jobs, and history.

- **`sinfo`**: Displays partition and node information. Output shows node states (`IDLE`, `ALLOCATED`, `MIXED`, `DOWN`), partition names, and node counts.
- **`squeue`**: Shows the job queue. The output shows job ID, partition, name, user, state, time, nodes, and reason code.
- **`sacct`**: Queries job accounting data from the database. The `State` field shows `COMPLETED`, `FAILED`, `CANCELLED`, etc. The `ExitCode` field shows the script's exit code (0:0 means success, non-zero means failure).
- **`sstat`**: Shows resource usage statistics for **running** jobs only. Note: `sstat` does not work for completed jobs; use `sacct` for historical data.
- **`scontrol`**: Administrative command for viewing and modifying cluster configuration. Can show node details, job information, partition configuration, and `slurm.conf` settings.

### Logging

Slurm daemons write logs that are critical for debugging and monitoring.

- **Log locations**: Slurm daemons write logs including `slurmctld.log` (controller), `slurmd.log` (compute nodes), and `slurmdbd.log` (database).
- **Log rotation**: Configure log rotation using `logrotate` or Slurm's built-in rotation. Debug levels can be set to control verbosity.
- **Log analysis**: Search logs for errors, job failures, or authentication issues. Use timestamps to correlate events across daemons.

### Metrics and integration

Slurm can export metrics for integration with monitoring systems.

- **Slurm metrics API**: The REST API (via `slurmrestd`) provides endpoints for job statistics, node states, and cluster utilization. Access metrics via HTTP requests to `slurmrestd`. The API version may vary by Slurm version.
- **Available metrics**: Common metrics include job counts (pending, running, completed), node states (idle, allocated, down), queue depth, resource utilization (CPU, memory, GPU), and throughput statistics.

### Debugging techniques

- **Job pending reasons**: Jobs can be pending for various reasons. Common reasons include `Priority` (waiting for higher priority jobs), `Resources` (cluster full), or `AssocGrpCPULimit` (user hit limit).
- **Node state investigation**: Use `scontrol show node` to see node state, reason codes, and configuration.
- **Scheduler debugging**: Increase debug levels for verbose scheduler logs.
- **Database connectivity**: If `sacct` fails, check database daemon logs for database connection errors.
- **Network issues**: If nodes appear `DOWN`, check network connectivity and MUNGE authentication.

## dstack

TBA

