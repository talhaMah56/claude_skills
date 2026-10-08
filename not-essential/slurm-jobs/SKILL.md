---
name: slurm-jobs
description: Write, submit, debug, and tune SLURM batch jobs for deep learning training on an HPC cluster. Covers sbatch script directives (--gres/--gpus-per-node, --cpus-per-task, --mem, --time, --array, --signal), single-node multi-GPU and multi-node torchrun/DDP launch under srun, hyperparameter sweeps with job arrays, atomic checkpointing with requeue-on-preemption, and post-mortem diagnosis with sacct/seff/squeue. Use this when the user mentions sbatch, srun, salloc, squeue, scancel, sacct, seff, sinfo, scontrol, SLURM_ARRAY_TASK_ID, SLURM_JOB_NODELIST, MASTER_ADDR, "job pending", "job got killed at the time limit", "OOM on the cluster", "requeue", "node-local scratch", "$SLURM_TMPDIR", "module load", or asks to turn a local training script into a cluster job, run a sweep on the cluster, or figure out why a job failed/pended/wasted its allocation. For the Qizhi platform specifically, use the qzcli skill instead — this skill is for generic SLURM.
---

# SLURM for deep learning training

Assumes Linux compute nodes and a PyTorch stack. SLURM has no Windows client — drive it over
ssh (from WSL or Windows Terminal); write scripts with LF endings or `sbatch` fails with
`/bin/bash^M: bad interpreter`.

## Cheat sheet: the commands you actually use

```bash
sbatch train.sh                      # submit; prints "Submitted batch job 12345"
sbatch --parsable train.sh           # prints just "12345" — use in scripts
sbatch --test-only train.sh          # validate directives + estimated start, submit nothing

squeue -u $USER -o "%.10i %.12P %.24j %.2t %.10M %.10l %.4D %R"   # %R = REASON, the useful column
squeue --start -j 12345              # backfill's estimated start time (only if backfill sched)
squeue -u $USER -t PD                # only pending

scancel 12345                        # kill job;  12345_7 kills one array task
scancel -u $USER -t PENDING          # kill all your pending jobs, leave running ones alone
scancel -b --signal=USR1 12345       # hand-trigger your checkpoint-and-requeue handler.
                                     # -b/--batch is REQUIRED: without it the signal goes to the
                                     # job steps, not the batch shell where your trap lives.

scontrol show job 12345              # live truth: Reason, TRES, StartTime, NodeList, StdOut
scontrol show hostnames "$SLURM_JOB_NODELIST"   # expands "gpu[03-06]" -> one host per line
scontrol requeue 12345               # put a running job back in the queue (needs --requeue)
scontrol show config | grep -E "MaxArraySize|MaxJobCount|DefMemPerCPU"

sinfo -o "%.20P %.5a %.12l %.6D %.6t %N"        # partitions, time limits, node states
sinfo -p gpu --Node -o "%N %t %C %m %G"         # per-node: state, CPU A/I/O/T, mem, GRES
sinfo -R                                        # why nodes are drained

seff 12345                           # after the fact: CPU% and Mem% of what you requested
sacct -j 12345 --units=G -o JobID,JobName%20,State,ExitCode,Elapsed,Timelimit,MaxRSS,ReqMem,AllocTRES%40
sacct -X -u $USER -S 2026-08-01 -o JobID,JobName%24,State,Elapsed,NodeList   # -X = job rows only
```

`sacct` reads the accounting DB, so it is the only tool that works after a job ends.
`squeue`/`scontrol show job` only know about live and very recently finished jobs.

### Interactive / debug sessions

```bash
# One-shot interactive shell on a GPU node (dies when you disconnect):
srun --partition=gpu --gres=gpu:1 --cpus-per-task=8 --mem=32G --time=01:00:00 --pty bash -l

# Allocation you can attach multiple steps to:
salloc --partition=gpu --gres=gpu:2 --cpus-per-task=16 --mem=64G --time=02:00:00
# ...then, inside the allocation shell:
srun --pty bash -l          # shell ON the compute node (salloc's shell may be on the login node)
```

Use `bash -l` or your module/conda hooks are not loaded. Run `srun --pty` inside `tmux` on the
login node — the moment your ssh drops, an untmuxed interactive job dies.

## The resource directives, and the CPU trap

```bash
#SBATCH --job-name=segformer-pretrain
#SBATCH --partition=gpu                 # sinfo to see names; often gpu/a100/short/debug
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1             # ONE task = one torchrun launcher per node
#SBATCH --gres=gpu:a100:4               # older syntax, still the most portable
##SBATCH --gpus-per-node=4              # newer equivalent (Slurm >= 19.05); do not set both
#SBATCH --cpus-per-task=32              # 32 CPUs for THAT one task
#SBATCH --mem=240G                      # HOST RAM. Nothing to do with GPU memory.
#SBATCH --time=24:00:00                 # D-HH:MM:SS or HH:MM:SS
#SBATCH --output=logs/%x-%j.out         # %x=job name, %j=job id, %A_%a for arrays
#SBATCH --error=logs/%x-%j.err          # omit to merge stderr into --output
#SBATCH --open-mode=append              # CRITICAL if you requeue: default truncates the log
#SBATCH --requeue                       # allow requeue on preemption/node failure
#SBATCH --signal=B:USR1@120             # SIGUSR1 to the batch shell 120s before the wall clock
```

`logs/` must exist before you submit. SLURM does not create it; the job starts, fails to open
the output file, and dies with no log telling you why.

**The trap that silently halves throughput:** `--cpus-per-task` is per *task*; you get
`--ntasks` × `--cpus-per-task`.

| Directives | You get | Effect on a 4-GPU DDP run |
|---|---|---|
| `--ntasks=1 --cpus-per-task=32` | 1 process, 32 cores | Correct for `torchrun` (it forks 4 children that share the 32-core cgroup) |
| `--ntasks=4 --cpus-per-task=8` | 4 processes, 8 cores each | Correct for `srun`-as-launcher (one rank per task) |
| `--ntasks=4 --cpus-per-task=1` | 4 processes, 1 core each | Dataloaders thrash one core; GPUs sit at 15%; **no error message** |
| `--ntasks-per-node=4 --gres=gpu:4` + `torchrun` inside | 4 torchruns × 4 ranks = 16 ranks on 4 GPUs | Hang or CUDA OOM |

Two more CPU landmines:

1. **Slurm 22.05+ stopped propagating `--cpus-per-task` from `sbatch` to `srun`.** Your `srun`
   step silently gets 1 CPU per task. Always do this in the script:
   ```bash
   export SRUN_CPUS_PER_TASK="${SLURM_CPUS_PER_TASK:-1}"   # or pass --cpus-per-task to srun
   ```
2. **`os.cpu_count()` and `multiprocessing.cpu_count()` report the whole physical node**, not
   your cgroup. On a 128-core node with `--cpus-per-task=8`, `num_workers=os.cpu_count()//2`
   spawns 64 workers into 8 cores. Use the affinity mask:
   ```python
   import os
   n_cpu = len(os.sched_getaffinity(0))          # Linux: respects the cgroup / cpuset
   num_workers = max(1, n_cpu // int(os.environ.get("SLURM_GPUS_ON_NODE", 1)))
   ```

`--mem` is host RAM enforced by cgroups; exceed it and the kernel OOM killer SIGKILLs you. GPU
memory is governed only by `--gres=gpu:N` — no SLURM flag limits or reserves VRAM.

## Single-node multi-GPU script

Directives as above (`--nodes=1 --ntasks-per-node=1 --gres=gpu:4 --cpus-per-task=32 --mem=200G`),
then the body:

```bash
set -euo pipefail                 # fail loudly; without this a bad `module load` is ignored

# --- environment (never rely on your interactive shell's env; see Environment section) ---
module purge
module load cuda/12.1 cudnn/8.9
source "$(conda info --base)/etc/profile.d/conda.sh"    # required: `conda activate` is a shell fn
conda activate dl

export SRUN_CPUS_PER_TASK="$SLURM_CPUS_PER_TASK"
export OMP_NUM_THREADS=$(( SLURM_CPUS_PER_TASK / SLURM_GPUS_ON_NODE ))   # else each rank grabs all cores
export PYTHONUNBUFFERED=1         # otherwise your log stays empty until the job dies
export TOKENIZERS_PARALLELISM=false

# --- stage data to node-local NVMe (see Data section) ---
LOCAL="${SLURM_TMPDIR:-${TMPDIR:-/tmp/$SLURM_JOB_ID}}"
mkdir -p "$LOCAL/data"
time tar -xf "/scratch/$USER/datasets/brats-shards.tar" -C "$LOCAL/data"

CKPT_DIR="/scratch/$USER/runs/$SLURM_JOB_NAME"; mkdir -p "$CKPT_DIR"

srun torchrun --standalone --nnodes=1 --nproc_per_node="$SLURM_GPUS_ON_NODE" \
  train.py --data "$LOCAL/data" --ckpt-dir "$CKPT_DIR" --num-workers 8 --resume auto &

wait      # required for the SIGUSR1 trap to fire; see Checkpointing section
```

`--standalone` sets up single-node rendezvous on a random free port — use it whenever
`--nodes=1`, and you never touch `MASTER_ADDR`/`MASTER_PORT`.

## Multi-node distributed training

Pick one launcher and do not mix them.

| Approach | When it wins | Cost |
|---|---|---|
| `srun --ntasks-per-node=1` wrapping `torchrun` | Default. Keeps torchrun's restart logic; one SLURM task per node so `--cpus-per-task` maps cleanly | Two layers of process management to reason about |
| `srun --ntasks-per-node=$NGPU` launching `python` directly, ranks from `SLURM_PROCID` | Simplest process tree; SLURM binds each rank to its own core set (best NUMA behaviour) | You wire up `RANK`/`LOCAL_RANK`/`WORLD_SIZE` yourself; no elastic restart |
| `accelerate launch` / Lightning `SLURMEnvironment` | You already use those frameworks; they read `SLURM_*` for you | Hidden defaults; still needs correct sbatch directives, which is where the bugs are |

Static rendezvous (`--node_rank`) is deterministic and fine for fixed allocations;
`--rdzv_backend=c10d` is needed for elastic runs but still requires node 0 reachable at the
endpoint, so it does not remove the `MASTER_ADDR` work.

```bash
#!/bin/bash
#SBATCH --job-name=vitl-pretrain
#SBATCH --partition=gpu
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=1          # ONE torchrun per node
#SBATCH --gres=gpu:4
#SBATCH --cpus-per-task=32
#SBATCH --mem=400G
#SBATCH --time=2-00:00:00
#SBATCH --output=logs/%x-%j.out
#SBATCH --open-mode=append
#SBATCH --requeue
#SBATCH --signal=B:USR1@180
set -euo pipefail

module purge && module load cuda/12.1
source "$(conda info --base)/etc/profile.d/conda.sh" && conda activate dl

# --- rendezvous ---
MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n1)
# Port derived from the job id: unique per job, stable across requeue, above 1024 and below
# the 32768-60999 ephemeral range so it can't collide with a socket the OS picked.
MASTER_PORT=$(( 20000 + SLURM_JOB_ID % 10000 ))
export MASTER_ADDR MASTER_PORT
echo "rdzv $MASTER_ADDR:$MASTER_PORT  nodes=$SLURM_NNODES  gpus/node=$SLURM_GPUS_ON_NODE"

# --- interconnect: pick the fast NIC, or NCCL may fall back to 1GbE and you lose 20x ---
export NCCL_SOCKET_IFNAME=^lo,docker0,virbr0    # or the explicit device, e.g. ib0
export NCCL_DEBUG=WARN                          # INFO when debugging a hang, it is very verbose
export TORCH_NCCL_ASYNC_ERROR_HANDLING=1        # torch>=2.2 (was NCCL_ASYNC_ERROR_HANDLING)
export SRUN_CPUS_PER_TASK="$SLURM_CPUS_PER_TASK"
export OMP_NUM_THREADS=$(( SLURM_CPUS_PER_TASK / SLURM_GPUS_ON_NODE ))
export PYTHONUNBUFFERED=1

# --- stage data ONCE PER NODE, before any torch process starts (see NCCL timeout note) ---
export LOCAL="${SLURM_TMPDIR:-/tmp/$SLURM_JOB_ID}"
export CKPT_DIR="/scratch/$USER/runs/$SLURM_JOB_NAME"; mkdir -p "$CKPT_DIR"
srun --ntasks-per-node=1 bash -c \
  'mkdir -p "$LOCAL/data" && tar -xf '"/scratch/$USER"'/datasets/shards.tar -C "$LOCAL/data"'

# Single quotes are load-bearing: SLURM_NODEID is set PER TASK by srun inside the step, and
# exported vars (MASTER_ADDR/PORT, LOCAL, CKPT_DIR) are propagated into it by srun.
srun --ntasks-per-node=1 --cpus-per-task="$SLURM_CPUS_PER_TASK" bash -c '
  torchrun --nnodes="$SLURM_NNODES" --nproc_per_node="$SLURM_GPUS_ON_NODE" \
           --node_rank="$SLURM_NODEID" \
           --master_addr="$MASTER_ADDR" --master_port="$MASTER_PORT" \
           train.py --data "$LOCAL/data" --ckpt-dir "$CKPT_DIR" --resume auto' &

wait
```

If you let the outer shell expand `$SLURM_NODEID` (double quotes, or a heredoc without quoting),
it is baked in once by the batch shell — which runs on the *first* allocated node, where
`SLURM_NODEID` is `0` (or unset) — so all four nodes launch with `node_rank=0`. The job then
hangs at rendezvous until the store/NCCL timeout, with no error message pointing at it. Same
trap for `SLURM_PROCID`/`SLURM_LOCALID`: per-task, meaningful only inside an `srun` step.

In `train.py`:

```python
import os, torch, datetime, torch.distributed as dist

local_rank = int(os.environ["LOCAL_RANK"])       # set by torchrun
torch.cuda.set_device(local_rank)                # BEFORE init_process_group, always
dist.init_process_group("nccl", timeout=datetime.timedelta(minutes=60))  # default ~10 min
model = torch.nn.parallel.DistributedDataParallel(model.cuda(), device_ids=[local_rank])
```

**NCCL's collective watchdog aborts the job if any rank is more than the timeout behind.** The
classic trigger is slow one-time work (untarring data, building an index, downloading weights)
on rank 0 while the others wait at a barrier. Do that *before* `init_process_group`, or in the
sbatch script via `srun --ntasks-per-node=1`, or raise `timeout`.

Debugging a multi-node hang, in order: (1) `NCCL_DEBUG=INFO`, check every rank printed a ring
topology; (2) `TORCH_DISTRIBUTED_DEBUG=DETAIL` to catch mismatched collective shapes/order;
(3) confirm all ranks have the same number of batches — use `drop_last=True` with
`DistributedSampler` or a short rank desyncs the whole job.

## Job arrays for sweeps

```bash
#SBATCH --array=0-23%4        # 24 tasks, at most 4 running concurrently
#SBATCH --output=logs/%x-%A_%a.out    # %A = array job id, %a = task index
#SBATCH --gres=gpu:1                  # resources are PER TASK, not for the whole array
```

The `%4` throttle caps how much of the partition you occupy, so the scheduler still backfills
your other work and you do not trip `QOSMaxJobsPerUser`. Without it, 23 tasks pend behind your
own first task's priority drop.

`--array=0-99:5` steps by 5. `--array=3,7,11` reruns just the failed ones (find them with
`sacct -X -j <arrayjobid> -s FAILED,TIMEOUT -o JobID`). The hard cap is `MaxArraySize` (often
1001, and it caps the *index*, not the count) — check `scontrol show config`.

```bash
# configs.txt, one run per line:  --lr 1e-4 --wd 0.05 --model vit_base
CFG=$(sed -n "$((SLURM_ARRAY_TASK_ID + 1))p" configs.txt)   # sed is 1-indexed, array is 0-indexed
[ -z "$CFG" ] && { echo "no config for task $SLURM_ARRAY_TASK_ID"; exit 1; }

OUT=/scratch/$USER/sweeps/$SLURM_JOB_NAME/${SLURM_ARRAY_JOB_ID}_${SLURM_ARRAY_TASK_ID}
mkdir -p "$OUT"
echo "$CFG" > "$OUT/config.txt"          # so you can reconstruct the run months later
srun python train.py $CFG --out-dir "$OUT" --seed "$SLURM_ARRAY_TASK_ID"
```

Two things that ruin sweeps: (a) every task writing to the same `--out-dir` or the same wandb
run name, so results silently interleave; (b) `$SLURM_ARRAY_TASK_ID` unset when you resubmit
without `--array`, making `sed -n "1p"` rerun config 0. The `[ -z "$CFG" ]` guard plus `set -u`
catches both. `sbatch --array=3,7,11 sweep.sh` on the command line overrides the directive —
handy for rerunning a subset without editing the file.

## Checkpointing, atomicity, and requeue

Assume the job **will** be killed mid-write — time limit, preemption, node failure, full disk.

1. Checkpoint on wall clock (every 20-30 min), not only per epoch — one epoch on a large
   dataset can exceed the partition's time limit.
2. Write to a temp file on the **same filesystem**, fsync, then `os.replace` (atomic on POSIX).
   A kill mid-`torch.save` otherwise leaves a truncated file that fails to load *and* has
   already destroyed your last good one.
3. Save optimizer, LR scheduler, AMP scaler, epoch **and** global step, and RNG state. Restoring
   only model weights restarts the LR schedule and reuses the same data order — you get a
   visible discontinuity in the loss curve and a run that is not the one you claim in the paper.
4. Resume automatically at startup. A resume you have to remember to type is one you will forget
   at 3am.

```python
import os, random, tempfile, numpy as np, torch, torch.distributed as dist

def save_checkpoint(path, model, optimizer, scheduler, scaler, epoch, step):
    if dist.is_initialized() and dist.get_rank() != 0:
        dist.barrier()                    # non-zero ranks wait; keeps ranks in lockstep
        return
    state = {
        "model": (model.module if hasattr(model, "module") else model).state_dict(),
        "optimizer": optimizer.state_dict(),
        "scheduler": scheduler.state_dict(),
        "scaler": scaler.state_dict(),        # AMP loss scale; dropping it spikes loss on resume
        "epoch": epoch, "step": step,
        "rng": {
            "python": random.getstate(),
            "numpy": np.random.get_state(),
            "torch": torch.get_rng_state(),                 # CPU generator
            "cuda": torch.cuda.get_rng_state_all(),         # list, one per visible device
        },
    }
    d = os.path.dirname(path) or "."
    os.makedirs(d, exist_ok=True)
    fd, tmp = tempfile.mkstemp(dir=d, suffix=".tmp")        # same dir => same filesystem
    try:
        with os.fdopen(fd, "wb") as f:
            torch.save(state, f)
            f.flush()
            os.fsync(f.fileno())                            # force to disk before the rename
        os.replace(tmp, path)                               # atomic
        link_tmp = path + ".link.tmp"                        # point "latest" at it, atomically
        if os.path.lexists(link_tmp): os.remove(link_tmp)
        os.symlink(os.path.basename(path), link_tmp)
        os.replace(link_tmp, os.path.join(d, "latest.pt"))
    except BaseException:
        if os.path.exists(tmp): os.remove(tmp)
        raise
    if dist.is_initialized():
        dist.barrier()

def load_checkpoint(path, model, optimizer, scheduler, scaler, device):
    ck = torch.load(path, map_location=device, weights_only=False)  # your own file: trusted
    (model.module if hasattr(model, "module") else model).load_state_dict(ck["model"])
    optimizer.load_state_dict(ck["optimizer"]); scheduler.load_state_dict(ck["scheduler"])
    scaler.load_state_dict(ck["scaler"])
    random.setstate(ck["rng"]["python"]); np.random.set_state(ck["rng"]["numpy"])
    torch.set_rng_state(ck["rng"]["torch"].cpu())            # must be a CPU ByteTensor
    torch.cuda.set_rng_state_all([s.cpu() for s in ck["rng"]["cuda"]])
    return ck["epoch"], ck["step"]
```

Notes that bite: `weights_only=True` is the safe default in torch 2.6+ but **rejects** your own
checkpoint, because the RNG dict holds non-tensor Python objects — use `weights_only=False` for
files you wrote, never for downloaded ones. `torch.cuda.get_rng_state_all()` length follows
`CUDA_VISIBLE_DEVICES`, so a 4-GPU checkpoint will not restore onto 2 GPUs; guard the length.
Keep the last 2-3 checkpoints, not one: atomic rename protects you from truncation, not from
having saved a NaN model.

**This rank-0-only pattern is correct for DDP only.** Under FSDP or DeepSpeed ZeRO-2/3 each rank
holds a *shard* of the params/optimizer state, so a rank-0 `state_dict()` is either wrong or a
collective that every rank must enter. Call `save_checkpoint` on all ranks and let the framework
do the gathering: FSDP via `torch.distributed.checkpoint.save` (one file per rank, resharded on
load) or `FSDP.state_dict_type(model, StateDictType.FULL_STATE_DICT, FullStateDictConfig(offload_to_cpu=True, rank0_only=True))`
before the rank-0 branch; DeepSpeed via `engine.save_checkpoint(dir, tag)` on every rank. Full
consolidated state for a 7B model is ~80 GB with optimizer state — sharded saves are also 10-50x
faster to the shared filesystem, which is what makes the `@120` signal window survivable.

### Preemption / time-limit handler

```bash
#SBATCH --signal=B:USR1@120     # B: = signal the BATCH SHELL, not the job steps
#SBATCH --requeue
```

Without `B:`, SIGUSR1 goes to the srun-launched processes and your bash trap never runs. `@120`
is seconds before the wall clock; make it comfortably longer than one checkpoint write — a 7B
optimizer state to a shared filesystem can take minutes. Measure it, do not guess.

```bash
handler() {
  echo "$(date -Is) caught USR1, requeueing job $SLURM_JOB_ID"
  # Ask the training process to checkpoint. Either forward the signal to python
  # (which installs its own signal.SIGUSR1 handler and saves), or drop a sentinel
  # file that the training loop polls once per step.
  touch "$CKPT_DIR/PLEASE_CHECKPOINT"
  sleep 90                      # give it time to finish the atomic write
  scontrol requeue "$SLURM_JOB_ID"
  exit 0
}
trap handler USR1

srun ... train.py ... &         # MUST be backgrounded
wait                            # bash only runs traps between foreground commands;
                                # `wait` is interruptible, a foreground srun is not.
```

Requeued jobs keep the **same job id**, so `--output=logs/%x-%j.out` reopens the same file —
this is exactly why `--open-mode=append` matters; the default `truncate` erases the first run's
log. `SLURM_RESTART_COUNT` tells you which attempt you are on; log it and bail out above ~10 to
avoid an infinite requeue loop against a genuinely broken node.

The sentinel file is more robust than forwarding signals through `srun` → `torchrun` → N python
children, where propagation is version-dependent. Poll it in the training loop:

```python
if step % 50 == 0 and os.path.exists(os.path.join(ckpt_dir, "PLEASE_CHECKPOINT")):
    save_checkpoint(...); dist.barrier(); sys.exit(0)
```

Auto-resume at startup: load `ckpt_dir/latest.pt` if it exists, else start fresh. That single
branch makes requeue transparent.

## Diagnosing failures

### Out of memory — decide which memory first

They are unrelated systems and the fix is different:

| Symptom | Which memory | Check | Fix |
|---|---|---|---|
| `torch.cuda.OutOfMemoryError: Tried to allocate ...` in the log; job state `FAILED` | **GPU VRAM** | the traceback; `nvidia-smi` during the run | smaller batch, grad accumulation, AMP/bf16, gradient checkpointing, `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` for fragmentation |
| State `OUT_OF_MEMORY`, or `sacct` ExitCode `0:125` / `0:9` (SIGKILL, which a wrapping shell reports as 137), log ends abruptly with no traceback | **host RAM** (cgroup OOM kill) | `sacct -j ID --units=G -o JobID,State,ExitCode,MaxRSS,ReqMem` | raise `--mem`; reduce `num_workers`/`prefetch_factor`; stop caching the whole dataset in a `Dataset.__init__` |

Reading `MaxRSS` correctly: it is recorded on the **step rows** (`12345.batch`, `12345.0`), not
the job row, so `sacct -X -j 12345 -o MaxRSS` shows blank — drop `-X`. It is also sampled every
`JobAcctGatherFrequency` seconds (commonly 30), so a fast spike is missed entirely: a job that
OOM-kills while `MaxRSS` looks fine is normal, not a contradiction.

Host-RAM math people get wrong: each DataLoader worker is a fork with its own copy-on-write
memory, and decoded batches sit in pinned host memory. `num_workers=16`, `prefetch_factor=4`,
batch 32 of 512×512×3 float32 ≈ 16×4×32×3 MB ≈ 6 GB of buffers alone.

### Job pending forever

`squeue -u $USER -o "%.10i %.9P %.2t %R"` — the last column is the reason.

| Reason | Meaning | What to do |
|---|---|---|
| `Priority` | Higher-priority jobs are ahead; nothing is wrong | wait, or request less/shorter so backfill can slot you in |
| `Resources` | Your request is valid but nothing free yet | `squeue --start -j ID` for the estimate |
| `QOSMaxJobsPerUserLimit`, `AssocMaxJobsLimit`, `QOSGrpGRES` | You hit a per-user cap | throttle arrays with `%N`; check `sacctmgr show assoc user=$USER format=Account,QOS,MaxJobs,GrpTRES` |
| `PartitionTimeLimit` | `--time` exceeds the partition's max | `sinfo -o "%P %l"` and split into requeueing chunks |
| `ReqNodeNotAvail`, `Reservation` | Nodes drained, or a maintenance reservation starts before your job would finish (the reason often reads `ReqNodeNotAvail, Reserved for maintenance`) | `sinfo -R`; shorten `--time` to fit before the reservation |
| `Dependency` | Waiting on `--dependency=afterok:...` | check the parent; `DependencyNeverSatisfied` means the parent failed — your job will pend forever, cancel it |
| `AssocGrpBillingMinutes` | Allocation/credits exhausted | talk to your admin |

A request no node can ever satisfy (e.g. `--gres=gpu:8` on a partition of 4-GPU nodes) pends
forever rather than erroring. `sinfo -p <part> --Node -o "%N %G"` shows what actually exists;
`sbatch --test-only` catches many of these at submit time.

### Time limit hit / node failure

`TIMEOUT` means the log just stops; `sacct -o Elapsed,Timelimit` confirms. Only a disaster if
you were not checkpointing. Structurally, prefer many short requeueing jobs over one long one —
short jobs backfill and start far sooner than a 48h request.

`NODE_FAIL` (or `srun: error: ... Killed by node failure`) is retried automatically with
`--requeue`. If the same node kills you repeatedly, get it from `sacct -j ID -o NodeList`,
resubmit with `--exclude=gpu07`, and report it.

### "Works interactively, fails in batch"

Almost always environment, and almost always one of these:

- **`~/.bashrc` is not sourced.** `#!/bin/bash` in a batch script is a non-interactive,
  non-login shell. Your conda init block and PATH edits do not run. Fix: explicitly
  `source "$(conda info --base)/etc/profile.d/conda.sh"` then `conda activate`. Do **not** use
  `#!/bin/bash -l` as the fix — it works, but it drags in whatever is in your login files, which
  is the next bullet.
- **`--export=ALL` is the default**, so your login shell's `PYTHONPATH`, `LD_LIBRARY_PATH`,
  `CUDA_VISIBLE_DEVICES` and half-activated conda env leak into the job. It works for you and
  breaks for your collaborator. For reproducibility: `#SBATCH --export=NONE` and set up the
  environment explicitly in the script. (`SLURM_*` variables are still provided.)
- **Different modules on login vs compute nodes**, or a login node with a newer CPU. Always
  `module purge` first, then load pinned versions (`module load cuda/12.1`, not `cuda`).
- **No `set -euo pipefail`**, so a failed `module load` or `conda activate` prints a warning and
  the job runs against the system Python, producing a confusing `ImportError` 40 lines later.
- **Relative paths.** The job starts in `$SLURM_SUBMIT_DIR`, which is where you *submitted* from,
  not where the script lives. Use absolute paths or `cd "$SLURM_SUBMIT_DIR"`.
- **`CUDA_VISIBLE_DEVICES`.** SLURM already restricts you to your allocated GPUs; if you also
  export it yourself you will index into the wrong set. Never set it in an sbatch script.

Reproduce the batch environment on purpose:
`srun --export=NONE --pty bash --noprofile --norc`, then run your setup lines by hand.

Containers (Singularity/Apptainer) remove this whole class of problem and are the right answer
for anything you will publish:

```bash
srun singularity exec --nv \
  --bind "$LOCAL/data":/data,/scratch/$USER/runs:/runs \
  /scratch/$USER/images/torch24.sif \
  python /workspace/train.py --data /data --ckpt-dir /runs
```

`--nv` is what exposes the NVIDIA driver — forget it and you get "no CUDA-capable device". Bind
what you need explicitly rather than assuming `$HOME` and `/scratch` are visible. Build the
`.sif` where you have root (or via a remote builder); you generally cannot build one on the
cluster.

### Compute nodes usually have no outbound internet

Login nodes reach the internet; compute nodes commonly do not. The failure is not an immediate
error — HTTP calls hang until a long timeout, so the job burns its first 10 minutes of GPU time
and then dies (or worse, W&B silently drops your metrics). Anything that downloads must be done
on the login node first, into a shared cache, and then pinned offline in the job:

```bash
# ONCE, on the login node (never in the job):
export HF_HOME=/scratch/$USER/hf                  # replaces the old TRANSFORMERS_CACHE
huggingface-cli download facebook/dinov2-large    # weights, tokenizer, config into HF_HOME
python -c "import timm; timm.create_model('vit_large_patch14_dinov2', pretrained=True)"

# in the sbatch script:
export HF_HOME=/scratch/$USER/hf
export HF_HUB_OFFLINE=1 HF_DATASETS_OFFLINE=1     # fail fast instead of hanging on a 30s retry
export WANDB_MODE=offline                         # writes to ./wandb/offline-run-*
export WANDB_DIR="$CKPT_DIR"                      # keep runs with the checkpoints, not in $HOME
export TORCH_HOME=/scratch/$USER/torch            # torchvision / torch.hub weights
```

Then `wandb sync "$CKPT_DIR"/wandb/offline-run-*` from the login node afterwards. For a sweep,
set `WANDB_RUN_GROUP="$SLURM_ARRAY_JOB_ID"` and `WANDB_NAME="task-$SLURM_ARRAY_TASK_ID"` so the
24 offline runs land as one comparable group instead of 24 identically named runs.

## Data: the shared filesystem is the bottleneck

Lustre/GPFS/NFS are optimized for large sequential I/O from few clients. A dataset of 2 million
loose PNGs hammers the metadata server, and every other user on the cluster notices. Symptoms:
GPU utilization sawtoothing between 0 and 90%, `ls` taking 30 seconds, epoch times that vary
wildly and get worse when the cluster is busy.

Order of operations at job start:

1. Keep the dataset on shared storage as a **small number of large archives** — WebDataset `.tar`
   shards (~0.5-2 GB each), or LMDB, or HDF5. 500 shards, not 2M files.
2. Copy/extract to node-local scratch **once per node**:
   `srun --ntasks-per-node=1 tar -xf ... -C "$SLURM_TMPDIR"`. Doing it per-rank means 4 processes
   untarring the same archive onto each other.
3. Train from node-local scratch. Write checkpoints to shared `/scratch` (node-local is wiped
   when the job ends — anything left there is gone, including your last checkpoint).

Which variable: `$SLURM_TMPDIR` exists on Alliance-style clusters; elsewhere it may be `$TMPDIR`,
`/local/$SLURM_JOB_ID`, or nothing at all. Use
`LOCAL="${SLURM_TMPDIR:-${TMPDIR:-/tmp/$SLURM_JOB_ID}}"` and check `df -h "$LOCAL"` in a debug
job before assuming you have 2 TB of NVMe.

| Format | Random access | Small-file friendly | Multiprocess gotcha |
|---|---|---|---|
| Loose files | yes | no — worst case on Lustre | none |
| WebDataset `.tar` shards | sequential only (shuffle buffer) | yes | shard count must be ≥ world size × workers or some ranks starve |
| LMDB | yes | yes | open the env lazily **inside the worker**, with `lock=False, readahead=False` |
| HDF5 / h5py | yes | yes | **h5py is not fork-safe** — open the file in `__getitem__`/`worker_init_fn`, never in `Dataset.__init__`, or workers share a handle and you get silent garbage or a hard crash |

Also: `num_workers>0` + `persistent_workers=True` + `pin_memory=True` is the right default once
data is local; on a slow shared FS more workers just multiplies metadata pressure.

## Etiquette and not wasting the allocation

- **Debug first.** Every cluster has a short/debug partition or QOS with near-instant start.
  Run the exact script with `--time=00:15:00` and a `--max-steps 20` flag before submitting 48h.
- **Request honestly.** Over-requesting delays *you*: the scheduler backfills small, short jobs
  into gaps, so a 4h/1-GPU job often starts hours before a 24h/4-GPU one. Right-size from
  `seff`/`sacct` of a previous run rather than guessing.
- **Check `seff <jobid>` after every substantial run.**
  ```
  CPU Efficiency: 8.2% of 32-00:00:00 core-walltime     <- asked for 32 cores, used ~2.6
  Memory Efficiency: 11.4% of 200.00 GB                 <- drop --mem to 32G and start sooner
  ```
  `seff` does **not** report GPU utilization, the number that matters most. Log it yourself on
  the first run of a new config:
  ```bash
  nvidia-smi --query-gpu=index,utilization.gpu,memory.used --format=csv -l 60 \
      > "$CKPT_DIR/gpustats-$SLURM_JOB_ID.csv" &
  NVSMI_PID=$!;  trap 'kill $NVSMI_PID 2>/dev/null' EXIT
  ```
  Sustained GPU util below ~80% on a data-heavy vision job means you are input-bound: check
  `num_workers`, whether data is on node-local scratch, and whether decode/augmentation is the
  hot path.
- Chain long runs instead of asking for a week:
  `JID=$(sbatch --parsable train.sh); sbatch --dependency=afterany:$JID train.sh` — with
  auto-resume from `latest.pt` this gets 5 days out of a 24h limit.
- Never run training, tarring, or large `rsync` on the **login node**.

## Related

The `qzcli` skill covers the Qizhi platform's own submission tooling; use it there. This skill
is for vanilla SLURM (`sbatch`/`srun`/`sacct`) on a generic HPC cluster.
