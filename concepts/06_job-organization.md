# 7 Job arrays & dependencies

## Slurm

### Job arrays

Job arrays provide the mechanism to submit massive numbers of similar tasks (e.g., hyperparameter sweeps across 1,000 configurations) without overloading the scheduler database with 1,000 separate job scripts.

- **Efficiency & Limits**: Arrays reduce overhead by sharing a single job script and parameters. Large clusters typically enforce a `MaxArraySize` limit in `slurm.conf`. To submit more, users must split their arrays.
- **Throttling**: Array jobs can be throttled to run only a limited number of tasks simultaneously.
- **Identification**: The system assigns a single **Array Job ID** (e.g., `1001`) and individual **Task IDs** (e.g., `1001_1`, `1001_2`).
- **Environment variables**: The script uses `SLURM_ARRAY_TASK_ID` to determine which slice of data to process. Other variables like `SLURM_ARRAY_TASK_COUNT` help calculate offsets.
- **Example: array job script** (script on login node, each task executes on compute nodes):
  ```bash
  #!/bin/bash
  #SBATCH --job-name=hyperparameter-sweep
  #SBATCH --array=1-100
  #SBATCH --nodes=1
  #SBATCH --gres=gpu:1
  #SBATCH --time=4:00:00
  
  # Each task runs training with different hyperparameters
  config_file="configs/hp_config_${SLURM_ARRAY_TASK_ID}.yaml"
  srun python train.py --config $config_file
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
- **`aftercorr` (Array-to-Array)**: This is essential for ML pipelines. If Array A generates model checkpoints and Array B evaluates them, using `aftercorr` ensures that **Task 1 of B starts as soon as Task 1 of A finishes**. It does not wait for the entire Array A to complete.
- **`singleton`**: This dependency is based on the **Job Name** and **User**, not IDs. It ensures that only one job with the name "ModelTraining" runs at a time for that user. It is useful for serializing access to a shared model registry or dataset without tracking Job IDs.
- **Example: job dependencies** (from login node):
  ```bash
  # Submit training job
  JOB_TRAIN=$(sbatch train.sh | awk '{print $4}')
  
  # Submit evaluation job that depends on training (afterok = only if training succeeds)
  sbatch --dependency=afterok:$JOB_TRAIN evaluate.sh
  
  # Submit cleanup job that runs after training regardless of success/failure
  sbatch --dependency=afterany:$JOB_TRAIN cleanup.sh
  
  # Submit job with singleton dependency (only one training job with this name runs)
  sbatch --job-name=ModelTraining --dependency=singleton train.sh
  ```

## dstack

dstack does not provide native job arrays or job dependencies. Each run is independent and must be submitted separately. For workflows requiring arrays or dependencies, use external workflow orchestration frameworks.

### Job arrays

dstack does not support job arrays. Each run is a separate, independent workload. To run multiple similar tasks, submit multiple runs programmatically.

- **No native arrays**: Unlike Slurm's array jobs, dstack has no concept of array job IDs or task IDs. Each run has its own unique run name and ID.
- **Multiple runs**: Submit multiple runs using scripts or the Python API. Each run can use the same configuration file but with different parameters.
- **Example: submitting multiple similar tasks** (using bash script):
  ```bash
  # Submit 100 hyperparameter tuning runs
  for i in {1..100}; do
    dstack apply -f train.dstack.yml --name "hp-sweep-${i}" \
      --env CONFIG_ID=${i}
  done
  ```
- **Throttling**: dstack does not provide built-in throttling for multiple runs. Use external tools (e.g., job queues, workflow frameworks) to control concurrency.
- **Management**: Each run must be managed individually. Use `dstack list` to view all runs and `dstack stop` to cancel specific runs.

### Job dependencies

dstack does not support job dependencies. The scheduler does not hold runs in the queue based on other runs' completion status. Use external workflow orchestration frameworks to implement dependency logic.

- **No native dependencies**: dstack has no equivalent to Slurm's `--dependency` flag. Runs are scheduled independently based on resource availability and priority.
- **External workflow frameworks**: Use workflow orchestration tools (e.g., Airflow, Prefect, Nextflow) that submit dstack runs based on dependency conditions. These frameworks monitor run completion and submit dependent runs accordingly.
- **Example: using Airflow to orchestrate dstack runs with dependencies** (based on [dstack Airflow example](https://github.com/dstackai/dstack/tree/master/examples/misc/airflow)):
  ```python
  from airflow.decorators import dag, task
  from datetime import datetime, timedelta
  
  DSTACK_VENV_PATH = "/path/to/dstack-venv"
  DSTACK_REPO_PATH = "/path/to/dstack-repo"
  
  @dag(
      schedule_interval=timedelta(days=1),
      start_date=datetime(2024, 1, 1),
      catchup=False,
  )
  def dstack_pipeline():
      @task.bash
      def train() -> str:
          return (
              f". {DSTACK_VENV_PATH}/bin/activate"
              f" && cd {DSTACK_REPO_PATH}"
              " && dstack apply -y -f train.dstack.yml"
          )
      
      @task.bash
      def evaluate() -> str:
          return (
              f". {DSTACK_VENV_PATH}/bin/activate"
              f" && cd {DSTACK_REPO_PATH}"
              " && dstack apply -y -f evaluate.dstack.yml"
          )
      
      # evaluate depends on train succeeding (afterok pattern)
      train() >> evaluate()
  
  dstack_pipeline()
  ```
- **Polling for completion**: By default, `dstack apply` runs in attached mode and waits for completion while streaming logs to stdout/stderr. Use `--detach` to submit runs without waiting. In detached mode, use the Python API to poll run status and determine when dependencies are satisfied.
- **Singleton pattern**: dstack does not provide a `singleton` dependency type. Implement this pattern in external workflow frameworks by checking for existing runs with the same name before submitting new ones.