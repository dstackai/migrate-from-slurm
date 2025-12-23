# 11 GPU health monitoring and device constraints

## Slurm

### GPU failure modes

Unlike CPUs, GPUs are peripheral devices that can fail "partially" while the node remains online. Slurm must detect these specific states to prevent black-hole scheduling (sending jobs to a node with a broken GPU).

- **Xid errors**: The driver reports specific error codes (Xid errors) indicating hardware degradation.
- **Zombie processes**: Previous jobs may leave processes attached to the GPU memory, preventing new jobs from allocating the device even if the GPU itself is healthy.
- **Throttling**: GPUs can get stuck in a low-power clock state due to thermal violations or power management bugs, severely impacting performance for the next user.

### GPU configuration and scheduling

GPUs are configured in `gres.conf` and allocated based on job requests.

- **gres.conf structure**: Define GPUs in `gres.conf` using node names, GPU names, and device files, or use auto-detection for automatic discovery. Each GPU type must be explicitly defined.
- **GPU scheduling policies**: When multiple jobs request GPUs, Slurm allocates them based on the scheduling policy. By default, GPUs are allocated exclusively to one job. Some clusters configure GPU sharing, but this requires additional setup.
- **GPU allocation**: GPUs are requested via job submission. The scheduler ensures GPUs are allocated from nodes that have them available.

### NVIDIA integration (DCGM)

For NVIDIA hardware, the **Data Center GPU Manager (DCGM)** is the standard tool for deep health monitoring, going beyond basic driver queries.

- **DCGM vs NVML**: While Slurm uses NVML to *discover* GPUs, NVML is often too lightweight to detect complex failures. DCGM provides active, intrusive health checks.
- **Diagnostics**: DCGM runs intrusive tests (math, memory, PCIe bandwidth) to validate hardware integrity.
- **Slurm Health Check integration**: Administrators configure the `HealthCheckProgram` parameter in `slurm.conf` to specify a script (NHC) that runs DCGM diagnostics.
- **Epilog checks**: It is best practice to run a quick DCGM check in the `Epilog` script immediately after every GPU job. If the check fails, the node is immediately drained to protect future jobs.
- **IGID (Instance Group ID)**: For MIG (Multi-Instance GPU), DCGM can monitor specific instances to ensure one corrupted slice doesn't take down the whole physical card.

**Note**: Detailed MIG configuration in Slurm is an advanced topic. See out-of-scope document for MIG setup details.

### AMD integration (ROCm & RVS)

For AMD Instinct (MI200/MI300) series, the toolset differs but the integration logic remains the same.

- **ROCm SMI**: Used for basic queries like temperature and utilization.
- **RVS (ROCm Validation Suite)**: The equivalent of DCGM. It runs targeted tests (e.g., PEQT for PCIe, GST for stress testing) to validate the hardware.
- **Integration**: The script specified by `HealthCheckProgram` calls RVS to validate the GPU. If RVS reports a failure, the script outputs an error message that triggers Slurm to drain the node.

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