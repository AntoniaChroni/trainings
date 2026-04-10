# 📌 Big picture: LSF vs Interactive Nodes

### LSF batch jobs

- Run inside strict scheduler‑controlled cgroups
- Hard limits on memory, processes, and sometimes threads
- If limits are exceeded → job can be throttled or killed
- Metrics are tracked and reported (what you’re seeing)

### Interactive nodes

- Shared, permissive Linux environment
- No per‑job enforcement (beyond OS/user limits)
- Same bad behavior may not crash, but will slow everyone
- Metrics are usually not tracked per job

So the same code can behave very differently.

- **LSF batch jobs**: resources are requested up front; memory is enforced with a hard peak limit (rusage[mem]). Exceed it and LSF can terminate the job. 
- **Interactive nodes**: shared environment with more permissive Linux behavior — short spikes or borderline memory behavior may “survive,” but that doesn’t mean it will survive On LSF. 

Practical consequence

- Interactive runs are great for debugging logic, but LSF metrics are the ones you should trust for capacity planning (especially memory).


## Understanding `max_swap`, `max_processes`, and `max_threads` On LSF
### *(and how this differs from interactive nodes)*

When running jobs on an HPC system, **LSF batch jobs** and **interactive nodes** behave very differently. The metrics `max_swap`, `max_processes`, and `max_threads` help diagnose performance, stability, and resource efficiency—especially for R, Seurat, and containerized workflows.

---

## High‑level difference: LSF vs Interactive Nodes

| Aspect | LSF Batch Jobs | Interactive Nodes |
|------|---------------|-------------------|
| Resource enforcement | **Strict (cgroups)** | **Loose (OS‑level)** |
| Memory limits | Hard enforced | Soft / permissive |
| Swap behavior | Minimal or job killed | Allowed |
| Process/thread limits | Tracked (sometimes capped) | Rarely enforced |
| Failure mode | Job terminated | Job slows |
| Metrics available | ✅ Yes | ❌ No |

This is why code may **work interactively but fail On LSF**.

---

## 1. `max_swap`

### What it measures
The **maximum amount of swap memory** used by your job.

> Swap is disk‑backed memory used when RAM is exhausted -> Swap = memory pages pushed from RAM to disk

### How to interpret it

| `max_swap` | Interpretation |
|-----------|----------------|
| 0–5 MB | ✅ Excellent |
| 100–500 MB | ⚠️ Memory pressure |
| >1 GB | ❌ Oversubscription |

### On LSF

- Swap is **discouraged**
- If memory is exceeded, LSF may:
  - allow minimal swap briefly
  - or kill the job (`TERM_MEMLIMIT`)
- Swap usage is tightly coupled to memory enforcement


### On interactive nodes

- Linux allows heavy swapping
- Job keeps running but becomes **very slow**
- Performance degrades instead of failing
- No automatic failure: This is why code may “work interactively but fail On LSF”.


---

## 2. `max_processes`

### What it measures
The **maximum number of OS processes** spawned by your job, including:
- main R process
- `future` workers
- forked workers
- external commands
- background helpers


### Why this matters

Each process:

- consumes memory
- has its own overhead


Too many processes → memory fragmentation, scheduler stress

### On LSF

- Process count is tracked
- High values may indicate:
  - uncontrolled fork()
  - nested parallelism
  - future / parallel / system calls spawning workers


Some clusters enforce process limits (ulimit -u)

### On interactive nodes

- Usually not enforced
- You can spawn hundreds of processes
- But you risk:
  - slowing the node
  - annoying other users 😄

✅ **Best practice:** `~30–100 processes `→ reasonable for complex R + Singularity workflows

>100 consistently → worth inspecting


---

## 3. `max_threads`

### What it is

- Total number of threads used across all processes
- Threads ≠ cores
- Many libraries spawn threads automatically:
  - OpenBLAS
  - MKL
  - OpenMP
  - future / parallel


### Why this is critical

This is the #1 silent performance killer.

Example:

  - You request 24 cores
  - Each BLAS‑using process spawns 64 threads
  - You suddenly have >1000 threads

LSF does not stop this, but:

  - CPU is oversubscribed
  - Cache thrashing
  - Performance collapses

### On LSF

- Threads are counted but not capped
- Scheduler allocates cores, but threads can exceed them
- High max_threads = oversubscription

### On interactive nodes

- Same oversubscription happens
- Linux time‑slices threads
- Job appears “alive” but is slow and unstable

✅ **Best practice:** `max_threads ~40–100 `→ healthy

> max_threads ~600–1300 → clear oversubscription


## 🔄 Why behavior differs so much

### LSF Batch Jobs vs Interactive Nodes

| Aspect | LSF Batch Jobs | Interactive Nodes |
|------|---------------|-------------------|
| Memory | Hard enforced | Soft / forgiving |
| Swap | Minimal or job killed | Allowed |
| Processes | Tracked, sometimes limited | Mostly unlimited |
| Threads | Tracked, not capped | Not capped |
| Failure mode | Job killed | Job slows |
| Debugging | Logs & metrics | Manual inspection |

---

## ✅ How to use these metrics in practice

### ✅ Good signs
- `max_swap ≈ 0`
- `max_threads ≈ requested cores` (or a small multiple)
- `max_processes` stable and explainable

### ⚠️ Warning signs
- Swap > **0.5–1 GB**
- Threads ≫ cores (e.g., **1200 threads on 24 cores**)
- Large variability between runs

---

## ✅ Best practices on LSF

Set these explicitly in your LSF script (adjust cores/memory/queue accordingly):

```
#BSUB -P project
#BSUB -J run-upstream-analysis
#BSUB -oo job.out -eo job.err
#BSUB -n 6
#BSUB -R "rusage[mem=280GB] span[hosts=1]"
#BSUB -q superdome
#BSUB -cwd "."

export OMP_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
export MKL_NUM_THREADS=1
export NUMEXPR_NUM_THREADS=1

singularity exec --cleanenv --env OPENBLAS_CORETYPE=Haswell ../../<your_fav_container>.sif bash <your_fav_bash_script>.sh
```


---

## 📌 Summary (Key Takeaways)

 - max_swap → Did I exceed RAM? (You didn’t ✅)
 - max_processes → How many OS workers did I spawn?
 - max_threads → Did I accidentally oversubscribe CPUs? (Often yes ⚠️)

**LSF is strict and honest. Interactive nodes are forgiving but misleading.**


---

# 📌 Other metrics to look at

## 1. `cpu_time`

- **What it is:** Total CPU seconds consumed (sum across all cores).
- **Why it matters:** It tells you whether you’re using parallel resources efficiently.

  - **On LSF:** useful because it’s collected per job; helps detect “requested 24 cores but used 1 core worth of CPU.” 
  - **On interactive nodes:** you typically don’t get a clean per-job summary; OS is shared so interpretation is noisier.

Use it to ask: Is my job CPU-bound (good parallel use) or mostly waiting/serial?

---

## 2. `max_memory`

- **What it is:** Peak memory usage at any moment.
- **Why it matters:** Most important metric for survival on LSF.

  - **On LSF:** directly tied to job success because LSF enforces a hard per-job peak memory limit. 
  - **On interactive nodes:** a job may temporarily spike and survive; that does not mean it would survive in batch. 

Use it to: set future `rusage[mem]` based on real peaks.

---

## 3. `average_memory`

- **What it is:** Average memory over runtime.
- **Why it matters:** Capacity planning and efficiency, but not a kill trigger.

  - **On LSF:** helpful for understanding whether your job is “memory-flat” (steady usage) vs “spiky” (large peaks).
  - **On interactive nodes:** still meaningful, but harder to capture unless you monitor manually.

Use it to: decide whether you can safely reduce requested memory or whether peaks are the real driver.

---

## 4. `total_requested_memory`

- **What it is:** Total memory you requested for the job (often expressed as an aggregate).
- **Why it matters:** Queue scheduling and wait times — requesting too much memory can significantly slow job dispatch.

  - **On LSF:** Directly influences scheduling. Batch jobs wait until *all* requested resources are available, so over‑requesting memory can increase queue time.
  - **On interactive nodes:** You usually don’t explicitly request memory; you simply consume what is available on the shared node.

Use it to: reduce wasted allocation and improve turnaround time.

---

## 5. `memory_delta`

- **What it is:** Conceptual headroom = *requested memory − maximum memory used*.
- **Why it matters:** Indicates whether you are:
  - **Under‑requesting** memory (negative or near‑zero delta)
  - **Over‑requesting** memory (large positive delta)

In the Intermediate HPC training, memory delta is explicitly used to estimate how much memory should be requested for future runs based on observed peak usage.

  - **On LSF:** Very useful for tuning future resource requests.
  - **On interactive nodes:** Not available as a clean, per‑job metric.

Use it to: right‑size memory requests and avoid both OOM kills and long queue waits.

---


## 6. `run_time`

- **What it is:** Actual execution time of the job (in seconds).
- **Why it matters:** Throughput, cost, and walltime estimation.

  - **On LSF:** Definitive per‑job measurement.
  - **On interactive nodes:** Still meaningful, but may be distorted by contention on shared resources.

Use it to: compare performance across queues, core counts, and memory configurations.

---

## 7. `turnaround_time`

- **What it is:** Typically queue wait + run_time (end-to-end time from submission to completion).
- **Why it matters:** This is the user experience metric (how long you waited).

  - **On LSF:** Critical. Large memory or core requests can significantly increase queue wait; misconfigured jobs may remain pending indefinitely.
  - **On interactive nodes:** Largely irrelevant, since there is no queue wait once a session is allocated.

Use it to: decide whether reducing requested memory or cores would improve scheduling.

---

## 8. `max_swap`, `max_processes`, `max_threads`

These metrics provide context on memory pressure and parallelism behavior.

### `max_swap`
- Indicates whether the job experienced memory pressure.
- **On LSF:** Swap is discouraged; jobs under memory pressure are more likely to be killed.
- **On interactive nodes:** Jobs may swap and continue running, but with degraded performance.

### `max_processes` / `max_threads`
- Reflect the degree of parallelism and potential oversubscription.
- Especially relevant for BLAS/OpenMP‑backed libraries (e.g., R, Seurat).

- **On LSF:** Oversubscription can lead to unstable or inefficient runs.
- **On interactive nodes:** Often tolerated, but performance may collapse silently.

Use them to: detect runaway parallelism and nested threading.

---

## ✅ How to prioritize metrics (what to check first)

### Tier 1 — Job survival & correctness
- `max_memory` — Did the job exceed the hard memory limit?
- `memory_delta` — Was memory under‑ or over‑requested?
- `max_swap` — Was the job under memory pressure?

If these look bad, nothing else matters.

---

### Tier 2 — Efficiency & performance
- `cpu_time` — Did the job actually use the cores requested?
- `max_threads` / `max_processes` — Was the job oversubscribed?
- `average_memory` — Was memory usage steady or spiky?

---

### Tier 3 — Scheduling & throughput
- `turnaround_time` — Was queue wait the bottleneck?
- `total_requested_memory` — Could requesting less improve dispatch?
- `run_time` — Optimize only after the above are stable.

---

## ✅ Practical patterns & actions

### Pattern A: High `max_memory` + negative or near‑zero `memory_delta`
Action: Increase memory request or switch queues — otherwise the job risks being killed due to the hard peak memory limit.

---

### Pattern B: Large `memory_delta` + long `turnaround_time`
Action: Reduce requested memory. Over‑requesting slows scheduling and increases wait time.

---

### Pattern C: `max_threads` ≫ cores + `cpu_time` not improving
Action: Cap BLAS/OpenMP threads in the job environment (especially for R/Seurat and containers).  
This is a common cause of “slow but not failing” interactive runs that behave poorly under LSF.

---

### Use `bjobs -l job_id`

Run `bjobs -l job_id` (replacing job_id with the job number) while the job is running or soon after it ends to see the CPU utilization. The number of threads is relevant, but not the full story (a lot of the threads may be sleeping at any one time).


---

## 📌 Bottom line

- **LSF metrics** are the most reliable for capacity planning because they reflect enforced limits and real scheduling behavior.
- **Interactive runs** are valuable for debugging and iteration but can be misleading with respect to memory spikes and oversubscription.


---

#### Authors

Antonia Chroni, PhD ([@AntoniaChroni](https://github.com/AntoniaChroni))


## Contact

Contributions, issues, and feature requests are welcome! Please feel free to check [issues](https://github.com/stjude-dnb-binfcore/trainings/issues).

---

*These materials have been developed by the [DNB Bioinformatics core team](https://www.stjude.org/research/departments/developmental-neurobiology/shared-resources/bioinformatic-core.html) at the [St. Jude Children's Research Hospital](https://www.stjude.org/). These are open access materials distributed under the terms of the [BSD 2-Clause License](https://opensource.org/license/bsd-2-clause), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*


