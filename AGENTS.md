# Slurm HPC Execution Policy
You are operating on a Slurm-managed HPC cluster. Follow this policy for ALL execution. Goal: complete tasks as fast as possible while preserving reliability, efficiency, and fair cluster usage.

---

## 0. Hard rules — non-negotiable

1. **Every execution of user code, tests, binaries, or scripts MUST go through Slurm** (`srun` / `sbatch`).
   Forbidden on the login node: `python`, `pytest`, `python -m pytest`, `make test`, `ctest`, `npm test`, `uv run`, `poetry run`, `./binary`, any `bash script.sh` that performs computation, and any other direct execution.
2. **The login node (local shell) is only for lightweight, non-compute work**:
   - reading / editing / writing files; metadata peeks (`ls`, `head`, `wc -l`, `ncdump -h`, `h5ls`);
   - **all network work**: `pip install`, `conda create/install`, `git clone`, `wget`/`curl`, dataset downloads, `module load`. Compute nodes may have no internet — finish every download and install on the login node **before** submitting jobs.
3. **Print the full Slurm command immediately before executing it.**
4. **Never fall back to local execution** because Slurm is inconvenient or a job failed. Diagnose (`squeue`, `sacct`, `.err` files) and resubmit.

## 1. Standard workflow

1. **Inspect the cluster** (at session start; refresh before any large submission):
   ```bash
   sinfo -h -O 'Partition:18,Available:6,StateLong:12,CPUs:6,Memory:10,Gres:14,Time:12'
   sinfo -N -h -O 'NodeList:12,Partition:16,StateLong:12,CPUsState:14,Memory:10'
   ```
   `CPUsState` prints `alloc/idle/other/total` per node — use idle counts to size requests and to confirm node03/node04 availability.
2. **Classify the task**: small → `srun` (§3); large → `sbatch` (§4).
   Large = expected runtime > 15 min, multi-step orchestration, or heavy CPU/GPU/memory needs.
3. **Size resources** per §2. If queue/start time matters, probe candidate configs first:
   `srun --test-only ...` or `sbatch --test-only job.sh`
4. **Submit → monitor → verify** (§5). Iterate fast (§6).

## 2. Resource sizing — optimize time-to-result

- **CPU by default.** Request GPU (`--gres=gpu:1`) only when the code actually uses it (`torch.cuda`, TensorFlow/JAX GPU, CUDA kernels).
- **Idle-cluster baseline = 32 cores.** When idle CPU nodes exist and the workload can scale, request **at least 32 cores** so tasks finish quickly — and make the workload actually use them:
  - tests: `python -m pytest -n <CPUS>` (pytest-xdist; match the `-c` value; install it on the login node)
  - Python: dask / joblib / multiprocessing with workers = allocated cores
  - numeric libs: `export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK` (already in the §4 template)
  Only drop below 32 cores for inherently serial tasks that cannot be parallelized quickly.
- **Upper bound**: never request more than the code can efficiently use. **Lower bound**: never so little that OOM or timeout becomes likely.
- **Prefer node03 / node04 for CPU-only jobs when idle**: `-w node03` (srun) or `#SBATCH --nodelist=node03`.
- **Walltime = shortest realistic estimate + ~50% buffer.** Tight limits start sooner via backfill and cap runaway jobs.
- **Memory**: estimate from input size + working set, add a buffer; take whole-node memory only when truly needed.

## 3. Small tasks → `srun`

Smoke tests, unit tests, quick validation, short debugging, single commands (expected < 15 min):

```bash
srun -p <PARTITION> -n 1 -c <CPUS> --mem=<MEM> -t <TIME> [-w node03] <command>
# e.g.
srun -p cpu -n 1 -c 32 --mem=32G -t 00:10:00 -w node03 python -m pytest -n 32 tests/
```

## 4. Large tasks → `sbatch`

Expected > 15 min, multi-step pipelines, full benchmarks, training, data generation, heavy full-suite tests.

```bash
#!/bin/bash
#SBATCH --job-name=<JOB_NAME>
#SBATCH --partition=<PARTITION>
#SBATCH --nodes=1
#SBATCH --ntasks=<NTASKS>              # 1 unless MPI
#SBATCH --cpus-per-task=<CPUS>
#SBATCH --mem=<MEM>
#SBATCH --time=<TIME>
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err
##SBATCH --account=<ACCOUNT>           # uncomment if the cluster requires accounting
##SBATCH --nodelist=node03             # idle node03/node04 preferred for CPU work
##SBATCH --gres=gpu:1                  # GPU jobs only
##SBATCH --array=0-9%4                 # N independent shards, <=4 concurrent; then use --output=%x-%A_%a.out

set -euo pipefail
cd "$SLURM_SUBMIT_DIR"

# Environment (all installs/downloads were already done on the login node)
# source ~/miniconda3/etc/profile.d/conda.sh && conda activate <ENV>
# module load <MODULES>
export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

<run command>
# MPI:       srun -n "$SLURM_NTASKS" ../bld/cesm.exe >& run.log
# Job array: python process.py --shard "$SLURM_ARRAY_TASK_ID"
```

Adapt the script to the actual workload and **fill every `<...>` placeholder — never execute a command or script that still contains one.**

## 5. Monitor, verify, iterate

- Record the job ID from `sbatch` output. Poll gently — `sleep` 20–60 s between checks, never a tight loop:
  `squeue -j <JOBID> -h -o '%T %M %R'`
- Peek at early output: `tail -n 50 <JOB_NAME>-<JOBID>.out <JOB_NAME>-<JOBID>.err`. If the job is clearly failing, `scancel <JOBID>` immediately — do not let a doomed job burn its walltime.
- On completion:
  `sacct -j <JOBID> -o JobID,State,ExitCode,Elapsed,MaxRSS,ReqCPUS`
  Success = `COMPLETED` + exit code 0 **and** the output files contain the expected results. Use `MaxRSS` / `Elapsed` to right-size the next submission.
- On failure: read the `.err` file, fix, resubmit. Debug by rerunning the failing command through a short `srun` if needed.

## 6. Go faster

- **Smoke-test before the big run**: validate the logic on a tiny subset via a 1–2 min `srun` before queueing a long `sbatch`. A one-minute test saves an hour-long failed job.
- **Fan out independent work**: multiple independent, non-conflicting tasks go out as separate parallel `sbatch` jobs or one job array (`--array`) — never run them sequentially.
- **Write parallel code**: when authoring or optimizing computational scripts, prefer HPC-suited parallel frameworks (dask, joblib, multiprocessing, mpi4py, numba, pytest-xdist) so the allocated cores are fully used and the solution scales.
- **Match the queue**: idle cluster → scale requests up to the workload's efficient limit; busy cluster → smaller core counts + tighter walltime slot into backfill gaps and start sooner. Compare start times with `--test-only` when unsure.
