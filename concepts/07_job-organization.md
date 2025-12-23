# 7 Job arrays & dependencies

## Slurm

### Job arrays

Job arrays provide the mechanism to submit massive numbers of similar tasks (e.g., 10,000 simulations) without overloading the scheduler database with 10,000 separate job scripts.

- **Efficiency & Limits**: Arrays reduce overhead by sharing a single job script and parameters. Large clusters typically enforce a `MaxArraySize` limit in `slurm.conf`. To submit more, users must split their arrays.
- **Throttling**: Array jobs can be throttled to run only a limited number of tasks simultaneously.
- **Identification**: The system assigns a single **Array Job ID** (e.g., `1001`) and individual **Task IDs** (e.g., `1001_1`, `1001_2`).
- **Environment variables**: The script uses `SLURM_ARRAY_TASK_ID` to determine which slice of data to process. Other variables like `SLURM_ARRAY_TASK_COUNT` help calculate offsets.
- **Management**: You can modify or cancel specific parts of an array, leaving the rest running.

### Job dependencies

Dependencies allow Slurm to act as a basic workflow manager, holding jobs in the queue until specific conditions are met.

- **Dependency types**: Dependencies can be specified with different types.
- **`afterok` vs `afterany`**: `afterok` runs only if the dependency job finishes with Exit Code 0. `afterany` runs regardless of success or failure (useful for cleanup jobs).
- **`aftercorr` (Array-to-Array)**: This is essential for pipelines. If Array A generates data and Array B processes it, using `aftercorr` ensures that **Task 1 of B starts as soon as Task 1 of A finishes**. It does not wait for the entire Array A to complete.
- **`singleton`**: This dependency is based on the **Job Name** and **User**, not IDs. It ensures that only one job with the name "MyDataPipeline" runs at a time for that user. It is useful for serializing access to a specific database or resource without tracking Job IDs.

## dstack

TBA