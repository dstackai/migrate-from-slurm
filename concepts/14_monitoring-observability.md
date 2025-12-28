# 14 Monitoring and observability

## Slurm

### Slurm commands

Slurm provides command-line tools for monitoring cluster state, jobs, and history.

- **`sinfo`**: Displays partition and node information. Output shows node states (`IDLE`, `ALLOCATED`, `MIXED`, `DOWN`), partition names, and node counts.
- **`squeue`**: Shows the job queue. The output shows job ID, partition, name, user, state, time, nodes, and reason code.
- **`sacct`**: Queries job accounting data from the database. The `State` field shows `COMPLETED`, `FAILED`, `CANCELLED`, etc. The `ExitCode` field shows the script's exit code (0:0 means success, non-zero means failure).
- **`sstat`**: Shows resource usage statistics for **running** jobs only. Note: `sstat` does not work for completed jobs; use `sacct` for historical data.
- **`scontrol`**: Administrative command for viewing and modifying cluster configuration. Can show node details, job information, partition configuration, and `slurm.conf` settings.

### sstat usage and limitations

`sstat` provides real-time resource usage statistics for running jobs, but has important limitations.

**Usage** (invoked from login nodes):
```bash
# Show resource usage for a running job
sstat --job=12345

# Show specific fields
sstat --job=12345 --format=JobID,MaxRSS,MaxVMSize,NTasks

# Show usage for all your running jobs
sstat --user=$USER

# Show usage for specific job steps
sstat --job=12345.0,12345.1
```

**Example output** (from login node):
```bash
$ sstat --job=12345 --format=JobID,MaxRSS,MaxVMSize,NTasks,CPUUtil
       JobID     MaxRSS  MaxVMSize   NTasks   CPUUtil
------------ ---------- ---------- -------- --------
12345.0        2048M     4096M          16    95.2%
12345.1        1024M     2048M           8    87.5%
```

**Limitations:**
- **Only works for running jobs**: `sstat` cannot query completed jobs. Once a job finishes, `sstat` returns no data.
- **Requires job to be running**: The job must be in `RUNNING` state. Jobs in `PENDING`, `COMPLETED`, `FAILED`, or `CANCELLED` states cannot be queried.
- **Step-level granularity**: Statistics are reported per job step, not per task. Use `--format` to see per-step breakdown.
- **Real-time data only**: `sstat` shows current resource usage, not historical trends.
- **No GPU usage monitoring**: `sstat` does not track GPU utilization or memory usage. It only shows CPU and memory statistics. To monitor GPU usage, use vendor tools like `nvidia-smi` (NVIDIA) or `rocm-smi` (AMD) directly on compute nodes, or access the node via `srun --jobid=<JOB_ID> --pty bash` to run monitoring commands.

**Alternatives for completed jobs:**
- **`sacct`**: Use `sacct` to query resource usage for completed jobs from the accounting database
- **Example**:
  ```bash
  # Query completed job resource usage
  sacct --job=12345 --format=JobID,MaxRSS,MaxVMSize,Elapsed,State
  
  # Query with more detail
  sacct --job=12345 --format=JobID,JobName,MaxRSS,MaxVMSize,TotalCPU,Elapsed,State
  
  # Query GPU allocation (what was allocated, not usage)
  sacct --job=12345 --format=JobID,AllocTRES,Elapsed,State
  # AllocTRES shows allocated resources like "cpu=32,mem=64G,gpu=4"
  ```

**GPU monitoring:**
- **GPU allocation**: `sacct` can show GPU allocation (via `AllocTRES` field) - what GPUs were allocated to the job, but not actual GPU utilization.
- **GPU usage/utilization**: Slurm does not track GPU utilization metrics. To monitor GPU usage during job execution, use vendor tools like `nvidia-smi` (NVIDIA) or `rocm-smi` (AMD) directly on compute nodes (access via `srun --jobid=<JOB_ID> --pty bash`).

**When to use each:**
- **`sstat`**: Monitor resource usage of currently running jobs in real-time
- **`sacct`**: Query historical resource usage for completed jobs, analyze job performance after completion

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

