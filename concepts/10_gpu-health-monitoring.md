# 10 GPU health monitoring and device constraints

## Slurm

### GPU failure modes

Unlike CPUs, GPUs are peripheral devices that can fail "partially" while the node remains online. Slurm must detect these specific states to prevent black-hole scheduling (sending jobs to a node with a broken GPU).

- **Xid errors**: The driver reports specific error codes (Xid errors) indicating hardware degradation.
- **Zombie processes**: Previous jobs may leave processes attached to the GPU memory, preventing new jobs from allocating the device even if the GPU itself is healthy.
- **Throttling**: GPUs can get stuck in a low-power clock state due to thermal violations or power management bugs, severely impacting performance for the next user.

### GPU configuration and scheduling

GPUs are configured in `gres.conf` and allocated based on job requests.

- **gres.conf structure**: Define GPUs in `gres.conf` using node names, GPU names, and device files, or use auto-detection for automatic discovery. Each GPU type must be explicitly defined.
- **Example: gres.conf with auto-detection** (located on compute nodes, or controller if configless):
  ```bash
  # Auto-detect GPUs on all nodes
  AutoDetect=nvml
  ```
- **Example: gres.conf manual configuration** (located on compute nodes):
  ```bash
  # Node gpu-node01 has 4 A100 GPUs
  Name=gpu Type=a100 File=/dev/nvidia[0-3]
  ```
- **GPU scheduling policies**: When multiple jobs request GPUs, Slurm allocates them based on the scheduling policy. By default, GPUs are allocated exclusively to one job. Some clusters configure GPU sharing, but this requires additional setup.
- **GPU allocation**: GPUs are requested via job submission. The scheduler ensures GPUs are allocated from nodes that have them available.

### NVIDIA integration (DCGM and NVML)

For NVIDIA hardware, Slurm uses both NVML and DCGM for GPU management and health monitoring, but they serve different purposes.

**NVML (NVIDIA Management Library):**
- **Purpose**: GPU discovery and basic queries
- **Usage**: Slurm uses NVML internally to discover GPUs at startup and query basic information (model, memory, temperature)
- **Location**: NVML library calls are made by `slurmd` on compute nodes during GPU discovery
- **Limitations**: NVML is lightweight and suitable for discovery, but cannot detect complex hardware failures or run intrusive diagnostics

**DCGM (Data Center GPU Manager):**
- **Purpose**: Deep health monitoring and diagnostics
- **Usage**: DCGM provides both passive health monitoring (via health watches) and active diagnostic tests to validate GPU hardware integrity
- **Location**: DCGM runs on compute nodes via health check scripts configured by administrators
- **Health monitoring**: DCGM health watches passively monitor for errors (ECC errors, PCIe errors, thermal violations) in real-time without interrupting workloads
- **Diagnostics**: DCGM diagnostics run active, intrusive tests (math operations, memory tests, PCIe bandwidth tests) to validate hardware integrity
- **Integration**: Administrators configure the `HealthCheckProgram` parameter in `slurm.conf` (on controller node) to specify a script that runs DCGM diagnostics on compute nodes

**When each is used:**
- **NVML**: Used automatically by Slurm for GPU discovery during `slurmd` startup on compute nodes. No user configuration needed.
- **DCGM**: Used by health check scripts (NHC) that run periodically or after jobs on compute nodes. Requires DCGM installation and health check script configuration.

**Example: DCGM health check script** (script located on compute nodes, configured in `slurm.conf` on controller):
```bash
#!/bin/bash
# Health check script on compute node
# Configured via HealthCheckProgram in slurm.conf on controller

# Run DCGM diagnostic
dcgmi diag -r 1  # Run level 1 diagnostics (quick check)

if [ $? -ne 0 ]; then
    echo "DCGM diagnostic failed" >&2
    exit 1  # Non-zero exit drains node
fi

# Check ECC errors
dcgmi health -c
if [ $? -ne 0 ]; then
    echo "ECC error count exceeded threshold" >&2
    exit 1
fi
```

**Epilog checks**: It is best practice to run a quick DCGM check in the `Epilog` script (located on shared filesystem, executed on compute nodes) immediately after every GPU job. If the check fails, the node is immediately drained to protect future jobs.

**IGID (Instance Group ID)**: For MIG (Multi-Instance GPU), DCGM can monitor specific instances to ensure one corrupted slice doesn't take down the whole physical card.

**Note**: Detailed MIG configuration in Slurm is an advanced topic. See out-of-scope document for MIG setup details.

### AMD integration (ROCm & RVS)

For AMD Instinct (MI200/MI300) series, the toolset differs but the integration logic remains the same.

- **ROCm SMI**: Used for basic queries like temperature and utilization. Similar to NVML for NVIDIA, ROCm SMI provides basic GPU information queries.
- **RVS (ROCm Validation Suite)**: The equivalent of DCGM for AMD GPUs. It runs targeted tests to validate hardware integrity.
- **Integration**: The script specified by `HealthCheckProgram` in `slurm.conf` (on controller node) calls RVS to validate the GPU on compute nodes. If RVS reports a failure, the script outputs an error message that triggers Slurm to drain the node.

### RVS test types and configuration

RVS provides multiple test types for validating AMD GPU hardware. Health check scripts on compute nodes run these tests, configured via `HealthCheckProgram` in `slurm.conf` on the controller.

**RVS test types:**
- **PEQT (PCIe End-to-End Test)**: Validates PCIe bandwidth and connectivity between CPU and GPU
- **GST (GPU Stress Test)**: Stress tests GPU compute units and memory
- **SMQT (System Memory Qualification Test)**: Tests GPU memory integrity
- **P3QT (P2P Qualification Test)**: Tests peer-to-peer communication between GPUs
- **FET (Frequency Endurance Test)**: Validates GPU frequency stability

**Example: RVS health check script** (script located on compute nodes, configured in `slurm.conf` on controller):
```bash
#!/bin/bash
# Health check script on compute node for AMD GPUs
# Configured via HealthCheckProgram in slurm.conf on controller

# Run PEQT (PCIe test) - quick check
rvs -c rvs.conf -t peqt

if [ $? -ne 0 ]; then
    echo "RVS PEQT test failed" >&2
    exit 1  # Non-zero exit drains node
fi

# Run GST (stress test) - more thorough
rvs -c rvs.conf -t gst

if [ $? -ne 0 ]; then
    echo "RVS GST test failed" >&2
    exit 1
fi
```

**RVS configuration file** (located on compute nodes):
```json
{
  "test": "peqt",
  "device": "all",
  "duration": 60
}
```

**Integration with Slurm**: RVS tests are run by health check scripts on compute nodes. The script exits with non-zero code on failure, which triggers Slurm to drain the node automatically.

### Automated draining mechanism

Slurm does not natively "speak" DCGM or RVS protocols. It relies on the **Node Health Check (NHC)** script acting as the translator.

- **The Trigger**: `slurmd` runs the script specified by `HealthCheckProgram` in `slurm.conf` periodically or after jobs.
- **The Check**: The script executes the vendor tool (DCGM or RVS).
- **The Failure**: If the tool reports "Fail" (e.g., ECC count exceeded), the script prints an error message to `stderr` and exits with a non-zero code.
- **The Action**: Slurm detects the non-zero exit code, automatically sets the node state to `DRAIN`, and sets the "Reason" field to the error message (e.g., "Reason: DCGM_ECC_ERROR").

### GPU resetting and isolation

Recovering a GPU often requires resetting the device rather than rebooting the whole node.

- **MPS / MIG Cleanup**: If using NVIDIA MPS (Multi-Process Service) or MIG, the `Epilog` script must explicitly tear down these services. Leaving an MPS daemon running can cause the next job to unknowingly share the context.
- **Reset commands**: Administrators can reset GPUs to clear GPU state without a reboot. Note that this only works if no processes are attached.

## dstack

### Hardware detection and metrics collection

`dstack-shim` detects GPU hardware when provisioning instances using vendor-specific utilities: `nvidia-smi` for NVIDIA GPUs, `amd-smi` for AMD GPUs, `tt-smi` for Tenstorrent, etc. `dstack-runner` uses the same utilities for some vendors (including NVIDIA and AMD) to collect per-job metrics such as GPU utilization and GPU memory usage during job execution. For detailed information on metrics collection and monitoring, see the [monitoring and observability](14_monitoring-observability.md) guide.

### NVIDIA integration (DCGM and dstack-shim)

For NVIDIA hardware, `dstack` integrates with DCGM (Data Center GPU Manager) through `dstack-shim` to perform passive GPU health monitoring. Unlike Slurm's active diagnostic approach, `dstack-shim` continuously monitors GPU hardware for reliability issues without interrupting running workloads. `dstack-shim` enables DCGM health watches and periodically queries GPU health status. The health status is reported to the `dstack` server, which uses it to determine instance scheduling eligibility.

**DCGM integration via dstack-shim:**
- **Purpose**: Passive health monitoring that detects GPU hardware issues (ECC errors, thermal violations, PCIe errors) in real-time
- **Implementation**: `dstack-shim` runs on fleet instances (hosts) and queries DCGM via the NVIDIA go-dcgm library. DCGM must be present on the host. For VM-based fleets, DCGM is automatically shipped with the default VM image for certain backends (AWS, Azure, GCP, OCI). For SSH fleets, ensure the `datacenter-gpu-manager-4-core`, `datacenter-gpu-manager-4-proprietary`, and `datacenter-gpu-manager-exporter` packages are installed on the hosts
- **Health statuses**: Instances can have three health statuses: `HEALTHY` (no issues), `WARNING` (non-fatal issues like correctable ECC errors), or `FAILURE` (fatal issues like uncorrectable ECC errors or PCIe failures). The health status is displayed in fleet listings via `dstack fleet`:

```bash
$ dstack fleet

 FLEET     INSTANCE  BACKEND          RESOURCES  STATUS          PRICE   CREATED
 my-fleet  0         aws (us-east-1)  T4:16GB:1  idle            $0.526  11 mins ago
           1         aws (us-east-1)  T4:16GB:1  idle (warning)  $0.526  11 mins ago
           2         aws (us-east-1)  T4:16GB:1  idle (failure) $0.526  11 mins ago
```

  - **`idle`**: No issues detected, instance is healthy and ready for workloads
  - **`idle (warning)`**: Non-fatal issues detected (e.g., correctable ECC errors). The instance remains usable but should be monitored
  - **`idle (failure)`**: Fatal issues detected. The instance is automatically excluded from scheduling

**DCGM metrics export to Prometheus:**
When Prometheus metrics are enabled (via `DSTACK_ENABLE_PROMETHEUS_METRICS` environment variable), `dstack-shim` can export DCGM metrics to Prometheus. `dstack-shim` uses `dcgm-exporter` (if available on the host) to expose DCGM metrics in Prometheus format. The `dstack` server collects these metrics from running jobs only (not from idle instances) and exposes them via the `/metrics` endpoint.

DCGM metrics exported include GPU utilization, memory usage, temperature, power consumption, ECC error counts, PCIe replay counters, NVLink errors, and many other hardware telemetry metrics. These metrics are available alongside job-level metrics (CPU, memory, GPU usage) and can be scraped by Prometheus for monitoring and alerting.

### Active health checks (NCCL and RCCL tests)

Unlike Slurm's automated health check scripts, `dstack` does not automatically run active diagnostic tests. However, administrators can manually run active checks such as NCCL (NVIDIA) or RCCL (AMD) tests as distributed tasks to validate GPU-to-GPU communication and interconnect bandwidth across a fleet.

**Example: NCCL tests as a distributed task** (`.dstack.yml`):
```yaml
type: task
name: nccl-tests

nodes: 2
startup_order: workers-first
stop_criteria: master-done

env:
  - NCCL_DEBUG=INFO
commands:
  - |
    if [ $DSTACK_NODE_RANK -eq 0 ]; then
      mpirun \
        --allow-run-as-root \
        --hostfile $DSTACK_MPI_HOSTFILE \
        -n $DSTACK_GPUS_NUM \
        -N $DSTACK_GPUS_PER_NODE \
        --bind-to none \
        /opt/nccl-tests/build/all_reduce_perf -b 8 -e 8G -f 2 -g 1
    else
      sleep infinity
    fi

resources:
  gpu: nvidia:1..8
  shm_size: 16GB
```

**Example: RCCL tests as a distributed task** (`.dstack.yml`):
```yaml
type: task
name: rccl-tests

nodes: 2
startup_order: workers-first
stop_criteria: master-done

# Mount the system libraries folder from the host
volumes:
  - /usr/local/lib:/mnt/lib

image: rocm/dev-ubuntu-22.04:6.4-complete
env:
  - NCCL_DEBUG=INFO
  - OPEN_MPI_HOME=/usr/lib/x86_64-linux-gnu/openmpi
commands:
  # Setup MPI and build RCCL tests
  - apt-get install -y git libopenmpi-dev openmpi-bin
  - git clone https://github.com/ROCm/rccl-tests.git
  - cd rccl-tests
  - make MPI=1 MPI_HOME=$OPEN_MPI_HOME

  # Preload the RoCE driver library from the host (for Broadcom driver compatibility)
  - export LD_PRELOAD=/mnt/lib/libbnxt_re-rdmav34.so

  # Run RCCL tests via MPI
  - |
    if [ $DSTACK_NODE_RANK -eq 0 ]; then
      mpirun --allow-run-as-root \
        --hostfile $DSTACK_MPI_HOSTFILE \
        -n $DSTACK_GPUS_NUM \
        -N $DSTACK_GPUS_PER_NODE \
        --mca btl_tcp_if_include ens41np0 \
        -x LD_PRELOAD \
        -x NCCL_IB_HCA=mlx5_0/1,bnxt_re0,bnxt_re1,bnxt_re2,bnxt_re3,bnxt_re4,bnxt_re5,bnxt_re6,bnxt_re7 \
        -x NCCL_IB_GID_INDEX=3 \
        -x NCCL_IB_DISABLE=0 \
        ./build/all_reduce_perf -b 8M -e 8G -f 2 -g 1 -w 5 --iters 20 -c 0
    else
      sleep infinity
    fi

resources:
  gpu: MI300X:8
```

**Note**: The RCCL example includes InfiniBand/RoCE configuration for high-performance interconnects. For simpler setups, some MPI options and volume mounts may be omitted, but the full configuration is shown here for production use cases.

These distributed tasks use MPI to coordinate across multiple nodes and test inter-GPU communication. They can reveal network or interconnect issues that passive monitoring might miss.

### AMD and other GPU vendors

`dstack` does not have DCGM-like GPU health monitoring integration for AMD GPUs or other accelerator vendors. For AMD Instinct GPUs, administrators can manually run RCCL tests (as shown above) or use ROCm SMI and RVS tools directly. For other GPU vendors (Intel Gaudi, Tenstorrent, etc.), there is currently no GPU health monitoring integration.