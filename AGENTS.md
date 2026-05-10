</extremely important>

- **Any command that executes user code, tests, binaries, or scripts must run through Slurm.**

- **Local shell use is allowed for lightweight actions such as reading, editing, or writing files, obtaining data file header information (e.g., ncdump -h, etc.), downloading and installing environments/packages (e.g., pip install, conda create/install, module load, etc.). Any testing, calculations, or other heavy-load tasks must go through srun or sbatch.**

- Forbidden outside Slurm: `python`, `pytest`, `python -m pytest`, `make test`, `ctest`, `npm test`, `uv run`, `poetry run`, and any other direct script/test/program execution.

- Before any execution, always inspect resources with:
  `sinfo -h -O Partition,Available,StateLong,CPUs,Memory,Gres,Time`

- Default to CPU. Use GPU only when the task uses CUDA/GPU (e.g., torch.cuda, TensorFlow GPU, JAX GPU, CUDA-dependent code).

- Allocate compute resources to optimize time-to-result while preserving reliability, efficiency, and fair cluster usage. Base the request on the workload’s demonstrated or expected CPU/GPU scalability, memory needs, I/O behavior, and queue/start-time tradeoffs. Do not under-request resources in a way that risks failure or excessive runtime, and do not over-request resources the job cannot efficiently use.

- If suitable compute nodes are idle or likely to start quickly, consider increasing the requested CPU, memory, node, or GPU resources to a reasonably high level that the workload is expected to use efficiently, so the task can complete faster without clearly wasteful over-allocation or undue impact on other users.

- If multiple independent, non-conflicting tasks exist and can run concurrently, prefer submitting them as separate Slurm jobs or as a Slurm job array so they execute in parallel rather than sequentially.

- Print the full Slurm command immediately before executing it.

- When writing and optimizing computational scripts, prioritize efficient algorithms and parallel computing tools suited to **HPC environments** such as **dask** and other parallel computing frameworks to improve **computational efficiency**, **resource utilization**, and **scalability**.

- Never fall back to local execution because Slurm is inconvenient.

- Use `srun` for small tasks: smoke tests, unit tests, quick validation, short debugging, single-command runs.

  Template:
  `srun -p <PARTITION> -n 1 -c <CPUS> --mem=<MEM> -t <TIME> <command>`

- Use `sbatch job.sh` for large tasks: expected runtime >15 minutes, multi-step pipelines, full benchmarks, training, data generation, or heavy full-suite tests.

  If queue/start-time matters, probe candidate configs first with:
  `srun --test-only ...`
  or
  `sbatch --test-only job.sh`

- `job.sh` template:

  ```bash
  #!/bin/bash
  #SBATCH --job-name=<JOB_NAME>
  #SBATCH --account=<ACCOUNT>
  #SBATCH --partition=<PARTITION>
  #SBATCH --nodes=1
  #SBATCH --ntasks-per-node=<NTASKS>
  #SBATCH --error=%j.err
  #SBATCH --time=<TIME>
  
  cd "$SLURM_SUBMIT_DIR"
  
  <run command>
  # e.g.
  # mpirun -np <NTASKS> ../bld/cesm.exe >& run.log
  ```

- Adjust `job.sh` to the actual workload. Do not leave placeholders in executed commands.

</extremely important>

## Slurm workflow

- First run:
  `sinfo -h -O Partition,Available,StateLong,CPUs,Memory,Gres,Time`
- Then classify the task:
  - Small task -> `srun`
  - Large task -> `sbatch`
- Treat a task as large if it is expected to run >15 minutes, needs multi-step orchestration, or requires heavy CPU/GPU/memory.
- Prefer CPU unless the workload requires GPU/CUDA.
- Use the most suitable allocation.
- If multiple configs are plausible and queue/start-time matters, test them with `--test-only` before real submission.
- Never execute the workload directly outside Slurm.