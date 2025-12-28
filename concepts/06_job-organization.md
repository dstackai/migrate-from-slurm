# 6 Job arrays & dependencies

## Slurm

### Job arrays

Job arrays provide the mechanism to submit massive numbers of similar tasks (e.g., 10,000 simulations) without overloading the scheduler database with 10,000 separate job scripts.

- **Efficiency & Limits**: Arrays reduce overhead by sharing a single job script and parameters. Large clusters typically enforce a `MaxArraySize` limit in `slurm.conf`. To submit more, users must split their arrays.
- **Throttling**: Array jobs can be throttled to run only a limited number of tasks simultaneously.
- **Identification**: The system assigns a single **Array Job ID** (e.g., `1001`) and individual **Task IDs** (e.g., `1001_1`, `1001_2`).
- **Environment variables**: The script uses `SLURM_ARRAY_TASK_ID` to determine which slice of data to process. Other variables like `SLURM_ARRAY_TASK_COUNT` help calculate offsets.
- **Example: array job script** (script on login node, each task executes on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=array-job
  #SBATCH --array=1-100
  #SBATCH --nodes=1
  #SBATCH --time=1:00:00
  
  # Each task processes different input file
  input_file="simulation_${SLURM_ARRAY_TASK_ID}.h5"
  srun python run_simulation.py $input_file
  ```
- **Example: submitting array job** (from login node):
  ```bash
  sbatch array_job.sh
  # Submitted batch job 1001
  
  # Check array status
  squeue -j 1001
  # Shows 1001_1, 1001_2, ..., 1001_100
  ```
- **Management**: You can modify or cancel specific parts of an array, leaving the rest running.

### Job dependencies

Dependencies allow Slurm to act as a basic workflow manager, holding jobs in the queue until specific conditions are met.

- **Dependency types**: Dependencies can be specified with different types.
- **`afterok` vs `afterany`**: `afterok` runs only if the dependency job finishes with Exit Code 0. `afterany` runs regardless of success or failure (useful for cleanup jobs).
- **`aftercorr` (Array-to-Array)**: This is essential for pipelines. If Array A generates data and Array B processes it, using `aftercorr` ensures that **Task 1 of B starts as soon as Task 1 of A finishes**. It does not wait for the entire Array A to complete.
- **`singleton`**: This dependency is based on the **Job Name** and **User**, not IDs. It ensures that only one job with the name "MyDataPipeline" runs at a time for that user. It is useful for serializing access to a specific database or resource without tracking Job IDs.
- **Example: job dependencies** (from login node):
  ```bash
  # Submit job A
  JOB_A=$(sbatch job_a.sh | awk '{print $4}')
  
  # Submit job B that depends on A (afterok = only if A succeeds)
  sbatch --dependency=afterok:$JOB_A job_b.sh
  
  # Submit cleanup job that runs after A regardless of success/failure
  sbatch --dependency=afterany:$JOB_A cleanup.sh
  
  # Submit job with singleton dependency (only one job with this name runs)
  sbatch --job-name=MyPipeline --dependency=singleton pipeline.sh
  ```

## dstack

TBA