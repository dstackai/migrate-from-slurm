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


### CLI

- **`dstack ps`**: Lists runs and their status. By default, shows unfinished runs or the last finished run. Use `--all` (or `-a`) to show all runs, `--verbose` (or `-v`) for additional details, `--watch` (or `-w`) to monitor status in real-time, and `--json` (or `--format json`) for JSON output. The output shows run names, backends, resources (GPU, CPU, memory), prices, status, and submission time.

**Example: `dstack ps` output**:
```bash
$ dstack ps

 NAME          BACKEND  GPU             PRICE    STATUS       SUBMITTED
 training-job  aws      H100:1 (spot)   $4.50    running      5 mins ago
 dev-env       gcp      L4:24GB         $0.16    provisioning 2 mins ago
```

**Example: `dstack ps --json` output**:
```bash
$ dstack ps --json
{
  "runs": [
    {
      "run_name": "training-job",
      "status": "running",
      "submitted_at": "2024-01-15T10:30:00Z",
      "jobs": [
        {
          "job_name": "training-job-0-0",
          "status": "running",
          "backend": "aws",
          "region": "us-east-1",
          "resources": {
            "gpus": [{"name": "H100", "memory_mib": 81920}],
            "cpus": 32,
            "memory_mib": 262144
          }
        }
      ]
    }
  ]
}
```

- **`dstack fleet`**: Lists fleet instances and their status. Shows fleet names, instance numbers, backends, resources (GPU), prices, status (including GPU health warnings), and creation time. Use `--watch` (or `-w`) to monitor fleet status in real-time, and `--verbose` (or `-v`) for additional details.

**Example: `dstack fleet` output with GPU health status**:
```bash
$ dstack fleet

 FLEET     INSTANCE  BACKEND          RESOURCES  STATUS          PRICE   CREATED
 my-fleet  0         aws (us-east-1)  T4:16GB:1  idle            $0.526  11 mins ago
           1         aws (us-east-1)  T4:16GB:1  idle (warning)  $0.526  11 mins ago
           2         aws (us-east-1)  T4:16GB:1  idle (failure)  $0.526  11 mins ago
```

The status field shows GPU health status detected by DCGM health checks: `idle` (healthy), `idle (warning)` (non-fatal issues like correctable ECC errors), or `idle (failure)` (fatal issues like uncorrectable ECC errors or PCIe failures). Instances with `idle (failure)` are automatically excluded from scheduling. For details on DCGM health monitoring, see the Metrics section below.

### REST API

The `dstack` server provides REST API endpoints for all entities, including runs/jobs, fleets/instances, and metrics.

### Metrics

#### Essential metrics

`dstack-runner` automatically collects essential per-job metrics using `nvidia-smi` for NVIDIA GPUs, `amd-smi` for AMD GPUs, `tt-smi` for Tenstorrent accelerators, etc. These metrics include CPU usage, memory usage, GPU utilization, and GPU memory usage. Metrics are collected periodically during job execution and stored by the `dstack` server.

The `dstack metrics` CLI command shows hardware metrics (CPU, memory, GPU utilization) for a specific run. Displays the most recently tracked metrics per job. Use `--watch` (or `-w`) to monitor metrics in real-time.

**Example: `dstack metrics` output**:
```bash
$ dstack metrics gentle-mayfly-1

 NAME             STATUS  CPU  MEMORY          GPU
 gentle-mayfly-1  done    0%   16.27GB/2000GB  gpu=0 mem=72.48GB/80GB util=0%
                                               gpu=1 mem=64.99GB/80GB util=0%
                                               gpu=2 mem=580MB/80GB util=0%
                                               gpu=3 mem=4MB/80GB util=0%
```

The `dstack` UI also provides a visual interface for viewing these metrics.

#### DCGM integration

For NVIDIA GPUs, `dstack-shim` integrates with DCGM (Data Center GPU Manager) to provide passive GPU health monitoring and advanced telemetry. DCGM is automatically available on VM-based fleets for AWS, Azure, GCP, and OCI backends. For SSH fleets, ensure the `datacenter-gpu-manager-4-core`, `datacenter-gpu-manager-4-proprietary`, and `datacenter-gpu-manager-exporter` packages are installed on the hosts.

DCGM telemetry includes GPU utilization, memory usage, temperature, power consumption, ECC error counts, PCIe replay counters, NVLink errors, and many other hardware telemetry metrics. However, these advanced DCGM telemetry metrics are only accessible via Prometheus (see the Prometheus section below). The CLI and UI only display essential metrics (CPU, memory, GPU utilization, GPU memory usage) collected via `nvidia-smi` and similar vendor utilities.

The DCGM metrics are only available via the Prometheus integration (see below).

##### Prometheus

To enable Prometheus metrics export, set the `DSTACK_ENABLE_PROMETHEUS_METRICS` environment variable on the `dstack` server and configure Prometheus to scrape metrics from `<dstack server URL>/metrics`. When enabled, `dstack-shim` can export DCGM metrics to Prometheus using `dcgm-exporter` (if available on the host). The `dstack` server collects these metrics from running jobs only (not from idle instances) and exposes them via the `/metrics` endpoint. In addition to essential metrics available via CLI and UI, `dstack` exports additional metrics to Prometheus, including data on fleets, runs, jobs, and DCGM telemetry.

Prometheus exports include:
- **Fleet**: Instance duration, price, GPU count (with labels for project, fleet, instance, backend, GPU)
- **Run**: Run counters (total, terminated, failed, done) per user and project
- **Job**: Job duration, price, CPU/memory/GPU usage, and DCGM telemetry (with labels for project, user, run, job, backend, GPU)
- **Server health**: HTTP request counts and durations

### Health monitoring

- **Active health checks (NCCL and RCCL tests)**: Unlike Slurm's automated health check scripts, `dstack` does not automatically run active diagnostic tests. However, administrators can manually run active checks such as NCCL (NVIDIA) or RCCL (AMD) tests as distributed tasks to validate GPU-to-GPU communication and interconnect bandwidth across a fleet. For detailed examples, see the [GPU health monitoring](10_gpu-health-monitoring.md) guide.

