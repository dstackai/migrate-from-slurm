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
- **Usage**: DCGM runs active, intrusive health checks to validate GPU hardware integrity
- **Location**: DCGM runs on compute nodes via health check scripts configured by administrators
- **Diagnostics**: DCGM runs intrusive tests (math operations, memory tests, PCIe bandwidth tests) to validate hardware integrity
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

TBA