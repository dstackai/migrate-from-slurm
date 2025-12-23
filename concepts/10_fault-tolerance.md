# 10 Fault tolerance, checkpointing, and job recovery

## Slurm

### Job failure types

In a Slurm environment, jobs generally fail in one of two ways, requiring different recovery strategies.

- **System failure (Hardware crash)**: The compute node loses power or connectivity. Slurm detects the timeout and marks the node `DOWN`.
- **Application failure (Software crash)**: The user's code produces a segmentation fault or exits with a non-zero error code.
- **Preemption**: A higher-priority job forces a running job to stop immediately to free up resources.

### Automatic requeueing

Slurm can automatically restart a job if the hardware fails, but this behavior must be requested.

- **Requeueing**: By default, if a node crashes, the job fails. If requeueing is enabled (via job submission or admin configuration), Slurm puts the job back into `PENDING` state to run again on a healthy node.
- **Distributed (MPI) behavior**: In a multi-node job, if *one* node fails, the entire job allocation is usually killed. If requeueing is active, the entire parallel job restarts from scratch on a new set of nodes.

### Preemption & grace periods

In large clusters, low-priority jobs (e.g., "Guest" or "Backfill" QOS) may be killed to make room for high-priority owners.

- **Grace period**: When Slurm decides to preempt a job, it does not kill it instantly. It enters a **Grace Period** (configured in `slurm.conf`).
- **Soft kill**: Slurm sends `SIGTERM` to the job steps. The application has this window to save its state (checkpoint) before `SIGKILL` is sent.
- **Requeue mechanism**: Preempted jobs are typically requeued automatically. They return to the queue in a held state or pending state, depending on configuration.

### Checkpointing & Signaling

Slurm does not inherently save the memory state of a job (no "system hibernation"). "Checkpointing" relies on the application saving its own progress to disk.

- **Application support**: The user's code (or ML framework like PyTorch/TensorFlow) must support saving state to disk periodically.
- **Signaling**: Users can instruct Slurm to send a specific signal to the application *before* the time limit expires or during preemption. The application catches this signal, writes a checkpoint file, and exits cleanly.

### Distributed job recovery (MPI)

Recovering distributed jobs is strictly harder than single-node jobs.

- **All-or-Nothing**: Standard MPI is not fault-tolerant. If Rank 0 fails, the MPI communicator breaks, and Ranks 1-1000 also die.

## dstack

TBA