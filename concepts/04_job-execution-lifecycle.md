# 4 Job submission, allocation, and execution

## Slurm

### Job submission and definition

Jobs are submitted using `sbatch` (batch), `srun` (step), or `salloc` (allocation).

- **sbatch (Batch script)**: The standard for production work. It submits a script file containing `#SBATCH` directives (e.g., `#SBATCH --nodes=2`) to the queue. The script executes on the first allocated node only when resources become available.
- **srun (Process launcher)**: This is Slurm's parallel process launcher (better than `mpirun`). It launches the actual application tasks across the allocated nodes. It can be used as a standalone command (creates an interactive allocation and runs immediately) or inside an `sbatch` script.
- **salloc (Interactive allocation)**: Requests resources for an interactive session. **Crucial**: By default, `salloc` grants a shell on the *login node* with environment variables set; it does not automatically SSH you to the compute node. You must run `srun` or `ssh` to reach the allocated resources.

### Interactive jobs and sessions

Interactive jobs allow users to run commands directly on compute nodes with real-time interaction.

- **`salloc` mechanics**: `salloc` creates an allocation and returns a shell on the login node with Slurm environment variables set. The allocation is active, but you must use `srun` or SSH to access the compute nodes.
- **`srun --pty`**: The `--pty` flag allocates a pseudo-terminal, allowing interactive commands. This provides an interactive shell on the first allocated node.
- **X11 forwarding**: X11 forwarding can be enabled for GUI applications, forwarding X11 to the first allocated node or all allocated nodes. Requires X11 forwarding configured in SSH.
- **SSH access**: From an `salloc` session, you can SSH directly to allocated nodes using the hostnames in `SLURM_NODELIST`.

### Job allocation mechanics

An "allocation" is the reservation of specific hardware (Nodes, CPUs, Memory) for a job.

- **Creation**: An allocation is created when a job transitions from `PENDING` to `RUNNING`.
- **Scope**: The allocation acts as a container. Within one allocation (e.g., 4 nodes for 2 hours), a user can run multiple "Job Steps" (e.g., run a data prep step on 1 node, then an MPI run on 4 nodes).
- **Structure**: The `slurmctld` records the node list and resource limits. The nodes are marked as `ALLOCATED` or `MIXED` state.

### Task launch (`slurmstepd`)

While `slurmctld` manages the cluster, `slurmstepd` is the daemon that manages the *process*.

- **Role**: A `slurmstepd` instance spawns on every allocated compute node. It is the parent process of the user's application.
- **Duties**: It launches the task, connects standard I/O (stdout/stderr) back to the user, and enforces resource limits.
- **Signal handling**: `slurmstepd` acts as the signal proxy. If a user cancels a job (`scancel`), `slurmstepd` receives the instruction and sends `SIGTERM`/`SIGKILL` to the application processes locally.
- **Step completion**: When the application exits, `slurmstepd` ensures all child processes are dead before reporting "Step Complete" to the controller.

### Job steps and task distribution

A single job allocation can contain multiple job steps, each running different commands or applications.

- **Job steps**: Within one allocation, you can run multiple steps sequentially or in parallel. Each step is identified by `SLURM_STEP_ID`. Example: Step 0 might prepare data, Step 1 runs the main computation, Step 2 post-processes results.
- **Task distribution**: Tasks are distributed across allocated nodes based on the `--ntasks` and `--ntasks-per-node` parameters. Tasks are distributed evenly across nodes according to these specifications.
- **MPI integration**: Slurm's `srun` is the recommended MPI launcher (replaces `mpirun`). It sets `SLURM_PROCID` to match MPI rank. Use `srun mpirun` only if your MPI implementation requires it. Slurm handles rank mapping automatically.
- **OpenMP/threading**: For threaded applications, use `--cpus-per-task` to allocate CPUs per task. CPU binding can be configured to bind threads to specific CPU cores. The `OMP_NUM_THREADS` environment variable should match `--cpus-per-task`.
- **Task binding**: CPU affinity is controlled by `--cpu-bind` with options to bind to cores, threads, sockets, or no binding.

### Job scripts and environment

#### Job script structure

A job script is a shell script (typically Bash) that contains both Slurm directives and executable commands.

- **Script format**: Job scripts start with a shebang line (e.g., `#!/bin/bash`) followed by `#SBATCH` directives, then the actual commands to execute.
- **Directive placement**: `#SBATCH` directives must appear at the top of the script, before any executable commands. They can be interspersed with comments, but must come before the first non-comment, non-directive line.
- **Script execution**: When the job starts, the script executes on the first allocated node (the "batch" node). The script runs with the user's permissions and environment.
- **Example structure**:
  ```bash
  #!/bin/bash
  #SBATCH --job-name=myjob
  #SBATCH --nodes=2
  #SBATCH --ntasks=32
  #SBATCH --time=1:00:00
  
  # Your commands here
  module load python/3.9
  srun python my_script.py
  ```

#### #SBATCH directives

Job scripts use `#SBATCH` directives to specify resource requirements and job options.

- **Common directives**: `--nodes`, `--ntasks`, `--cpus-per-task`, `--mem`, `--time`, `--partition`, `--job-name`, `--output`, `--error`.
- **Resource specification**: `--nodes=4` requests 4 nodes, `--ntasks=32` requests 32 tasks, `--cpus-per-task=4` requests 4 CPUs per task.
- **Time limits**: `--time=2:00:00` sets a 2-hour walltime limit. Use `--time-min` for minimum time guarantees.
- **Output control**: `--output=job-%j.out` sets stdout filename, `--error=job-%j.err` sets stderr filename. Use `%j` for job ID, `%A` for array job ID, `%a` for array task ID.

#### Slurm environment variables

Slurm injects environment variables into the job's environment.

- **Identity**: `SLURM_JOB_ID` (job ID), `SLURM_JOB_USER` (submitting user), `SLURM_JOB_NAME` (job name).
- **Topology**: `SLURM_NODELIST` (comma-separated node list), `SLURM_JOB_NUM_NODES` (node count), `SLURM_NODEID` (node index in allocation).
- **Task geometry**: `SLURM_NTASKS` (total tasks), `SLURM_CPUS_PER_TASK` (CPUs per task), `SLURM_NPROCS` (total processes).
- **Process identity**: `SLURM_PROCID` (MPI rank/global task ID), `SLURM_LOCALID` (local task ID on node), `SLURM_STEP_ID` (step ID within job).
- **Array jobs**: `SLURM_ARRAY_JOB_ID`, `SLURM_ARRAY_TASK_ID`, `SLURM_ARRAY_TASK_COUNT`, `SLURM_ARRAY_TASK_MAX`, `SLURM_ARRAY_TASK_MIN`.
- **Environment inheritance**: By default, `sbatch` copies the user's current environment (loaded modules, paths) into the job. Environment inheritance can be controlled to start with a minimal environment (only Slurm variables) or to explicitly copy all variables.


#### Job prolog and epilog

Administrators can configure scripts that run before and after every job.

- **Prolog**: Scripts configured via the `Prolog` parameter in `slurm.conf` run on each allocated node before the job starts. Useful for setting up node-specific configurations or validating node state.
- **Epilog**: Scripts configured via the `Epilog` parameter in `slurm.conf` run on each allocated node after the job completes. Useful for cleanup, health checks, or logging.
- **User prolog/epilog**: Users can specify `--prolog` and `--epilog` scripts in job scripts, but these require administrator configuration to enable.

### Job script error handling and exit codes

Job scripts can fail for various reasons, and Slurm interprets exit codes to determine job state.

- **Exit code interpretation**: If a job script exits with code 0, the job state is `COMPLETED`. If it exits with a non-zero code, the job state is `FAILED`. The exit code of the last command in the script determines the job's final state.
- **Script failure handling**: If a command in the script fails (non-zero exit), subsequent commands may still execute unless the script uses `set -e` (exit on error). Use `set -e` at the start of Bash scripts to ensure the job fails immediately on any command error.
- **Signal handling in scripts**: Job scripts can trap signals to perform cleanup. Use `trap 'cleanup_function' EXIT TERM INT` to run cleanup on job termination. Custom signals can be sent before walltime expiration.
- **Job state transitions**: Jobs transition `PENDING` → `RUNNING` → `COMPLETED` (exit 0) or `FAILED` (non-zero exit) or `CANCELLED` (user/admin cancellation). Preempted jobs may be `REQUEUED` back to `PENDING`.

### Job output and logging

Slurm captures job output and provides mechanisms for log management and notifications.

- **stdout/stderr handling**: By default, job output goes to files in the submission directory. Custom paths can be specified for output and error files.
- **Log file naming**: Log file naming patterns support placeholders for job ID, array job ID, array task ID, node name, and username.

## dstack

TBA