# 📌 How to Diagnose **Memory‑Related** Issues in Snap and Other Pipelines

This document summarizes common memory‑related failure modes observed in Snap and similar pipelines on HPC systems, and provides guidance on how to correctly interpret and diagnose them.

---

## 1. Understanding “Bus Error” Failures

A **`bus error`** typically indicates a **low‑level system crash**, not a standard R error.

- In LSF batch jobs, memory limits are **strictly enforced** via a hard peak RSS cap (`rusage[mem]`).
- Batch environments are also more sensitive to **filesystem or library access issues**, especially under memory pressure.
- As a result, a job may fail due to:
  - a small transient memory overshoot (e.g. ~1–2GB), or
  - a temporary shared filesystem or library read failure.

Importantly, these failures **do not necessarily indicate insufficient memory allocation**, even though they may surface during memory‑intensive steps.

---


## 2. Framework‑Level Limits (`future.globals.maxSize`)

Using future package with multisession means multiple independent R processes are launched, each with its own memory copy of large objects. This improves safety and control but increases peak memory usage and makes correct memory configuration critical on HPC.

When running an R script that uses the future package, the multisession plan means that:

  - Parallel execution is implemented using multiple independent R processes, not threads.
  - Each worker runs as a separate R session with its own memory space.
  - Data and objects needed for computation are copied (serialized) and sent from the main R process to each worker.

In practical terms, this has several important implications on HPC:

1. Memory is replicated across processes. Because each worker is a separate R process:

    - Large objects (e.g. Seurat objects, metadata, functions with environments) are duplicated in memory.
    - Total memory usage can be significantly higher than the size of the original object.
    - Memory spikes often occur during object export and deserialization, not just during computation.

2. LSF sees multiple processes sharing the same memory limit. Although multisession creates multiple R processes:

    - LSF enforces a single, shared peak memory limit for the entire job.
    - If the combined memory usage of all worker processes exceeds the requested rusage[mem], the job is terminated—even if each process individually appears reasonable.
    - This explains why jobs may fail with TERM_MEMLIMIT or low‑level SIGBUS errors during parallel steps.

3. Failures often surface during object export, not computation. With multisession, failures frequently occur when:

    - large globals are exported to workers,
    - objects are merged or serialized,
    - downstream steps trigger additional memory duplication.
    - As a result, the error location may appear misleading (e.g. during plotting or file output), even though the root cause is memory pressure from parallelization.

4. Why this matters for Snap and similar pipelines. Snap uses future intentionally to: control parallel execution, safely process large objects, and fall back to sequential behavior when needed. However, this means users must: account for memory replication when requesting resources, configure `future.globals.maxSize` appropriately, and limit internal library threading to avoid compounding memory pressure.



In several cases, job failures maybe caused by how **`future` handles global objects**, rather than by total memory availability.

- During parallelized steps (e.g. object merging or downstream computations in `Process_Seurat`), large objects such as:
  - Seurat objects,
  - metadata,
  - functions with captured environments  
  are exported as globals to worker processes.

- By default, `future` enforces a **conservative size limit** on exported globals.
- If this limit is exceeded, the job can fail **even when sufficient memory is available at the LSF level**.

Evidence supporting this diagnosis:

- Increasing LSF cores and memory alone does **not** resolve the failure.
- The same step completed successfully once `future.globals.maxSize` is explicitly increased (e.g. to ~400 GB).
- Similar‑ or larger‑scale cohorts have previously completed successfully with comparable or smaller LSF allocations.

✅ Example:

   > One run required ~34 cores × 64 GB (~2.1 TB total) on an interactive node, with `future.globals.maxSize = 400GB`, confirming the issue was not total RAM availability.


---

## 3. Secondary Failures: Filesystem and Library Access

After resolving the initial `future.globals.maxSize` limitation, a **second failure** maybe observed:

- R lost access to the shared package library (`/usr/local/lib/R/site-library`).
- This may result in a low‑level `bus error` during future parallelization.
- The failure occurs while exporting large globals and accessing package metadata.

This behavior points to **filesystem or mount instability under memory pressure**, not to an incorrect memory request.

Mitigations applied:

- Increasing `future.globals.maxSize` reduced pressure on the system.
- Removing and rebuilding Singularity (`.sif`) images and cache artifacts ensured no stale state persisted.
- Adding `OPENBLAS_CORETYPE` library before executing the container, e.g.,

```
singularity exec --cleanenv --env OPENBLAS_CORETYPE=Haswell ../../rstudio_4.4.0_seurat_4.4.0_latest.sif bash <my_fav_script>.sh
```

---


## 4. Threading Considerations

At present, explicit thread control is **not consistently enforced** on the St. Jude HPC system. However, the following practices help:

###  ✅ Container execution with a fixed CPU target

- This command does several important things: `--cleanenv`

  - Strips host environment variables before entering the container.
  - Prevents unintended inheritance of host‑level thread or BLAS settings.
  - Improves reproducibility and reduces environment‑specific failures.

- `OPENBLAS_CORETYPE=Haswell`:

  - Forces OpenBLAS to use a baseline, widely supported CPU instruction set.
  - Prevents OpenBLAS from selecting aggressive, architecture‑specific optimizations at runtime.
  - Avoids subtle numerical differences and low‑level crashes that can occur when the same container runs on heterogeneous CPU architectures.

This setting has been shown to:

  - Improve reproducibility across different HPC nodes.
  - Reduce the likelihood of low‑level SIGBUS or library access errors under memory pressure.

```
singularity exec --cleanenv --env OPENBLAS_CORETYPE=Haswell ../../rstudio_4.4.0_seurat_4.4.0_latest.sif bash <my_fav_script>.sh
```

###  ✅ Thread limiting (preventing oversubscription)

These environment variables force single‑threaded execution for common numerical libraries used by R and its dependencies (OpenMP, OpenBLAS, MKL, NumExpr).

Why this matters:

- Many R packages internally spawn threads per process.
- When combined with parallelization frameworks (e.g. future), this can lead to massive thread oversubscription.
- Oversubscription increases memory usage, context switching, and filesystem pressure, often causing jobs to fail even when memory appears sufficient.

By explicitly setting all thread counts to  1,  before executing  the  container:

- You ensure that LSF controls parallelism, not the libraries.
- Memory usage becomes more predictable.
- Stability improves, especially during large object serialization and parallel execution.

```
export OMP_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
export MKL_NUM_THREADS=1
export NUMEXPR_NUM_THREADS=1
```

---


## 5. Best Practices: Memory Validation and Resource Requests

- Always review **post‑run peak memory usage** after a successful job.
- Use this information as a concrete benchmark for future runs.
- This helps avoid both under‑ and over‑requesting resources.

### Choosing Cores vs Memory

When selecting between configurations such as:

- 34 cores with 64GB vs
- 25 cores with 100GB  

keep in mind:

- Runtime is often driven more by **available memory** than by core count.
- Using more cores with less memory can be acceptable for safety, but queue selection must match the **total memory requested** to ensure proper scheduling. For example, if you require more than 1 TB of memory, use the `large_mem` queue to ensure proper resource allocation. For more information, see [Introduction to the HPCF cluster](https://wiki.stjude.org/display/HPCF/Introduction+to+the+HPCF+cluster).
- However, increasing the number of cores is not always beneficial. Many steps in single‑cell pipelines are memory‑bound rather than CPU‑bound, meaning performance is limited by how fast large objects can be allocated, copied, and accessed in memory—not by how many cores are available. In addition, parallel frameworks such as future often create multiple independent R processes, each with its own memory footprint. Increasing the core count therefore increases the number of worker processes and replicates large objects across processes, which can significantly raise peak memory usage without providing meaningful runtime improvements. In these cases, requesting more cores can actually increase the likelihood of memory pressure, filesystem stress, and job failure, rather than speeding up execution.


---


## 6. LSF vs Interactive Nodes

When a cohort size and configuration have worked previously but failures persist:

✅  **Run the job interactively and compare behavior.**

Key differences:

- Interactive sessions run on shared nodes with **more permissive Linux behavior**.
- Temporary memory spikes or brief filesystem hiccups may be tolerated.
- LSF batch jobs enforce:
  - strict per‑job peak memory limits,
  - tighter I/O and mount handling.

Under LSF:

- Exceeding the memory limit even briefly can trigger immediate termination (e.g. `TERM_MEMLIMIT`, exit 143).
- Combined memory pressure and transient filesystem issues (e.g. “Transport endpoint is not connected”) can surface as fatal `SIGBUS` errors.

In interactive sessions, these issues may recover silently; under LSF, they are **visible and fatal**.


---

### ✅ Resource Usage Summary for Representative Snap Cohort

| no_samples | no_cells | object_size_GB | pipeline | module                  | node | queue    | memory_requested        |  future_globals_GB (default: 200) |
|-----------:|---------:|---------------:|----------|--------------------------|------|----------|------------------------|----------------------------------|
| 8          | 55,424   | 13             | snap     | upstream-analysis        | LSF  | standard | 16 cores, 30GB        |  200                             |
| 8          | 55,424   | 13             | snap     | integrative-analysis     | LSF  | standard | 10 cores, 96GB        | 200                              |
| 8          | 55,424   | 13             | snap     | cluster-cell-calling     | LSF  | standard | 4 cores, 48GB         |  200/400                          |
| 8          | 55,424   | 13             | snap     | cell-types-annotation   | LSF  | standard | 4 cores, 64GB          | 200                              |
| 8          | 55,424   | 13             | snap     | rshiny-app               | LSF  | standard | 4 cores, 30GB         |  N/A                             |


**Notes:**

- These values reflect observed resource usage for a representative cohort size and module configuration.
- `future.globals.maxSize` defaults to 200/400GB unless explicitly overridden; adjustments may be required for large object exports during parallel steps.
- Actual requirements may vary depending on dataset complexity, parameters, and execution context.


---

## 📌 Summary (Key Takeaways)

- The observed failures are **not always caused by insufficient LSF memory**.
- The primary issue can be an underestimated `future.globals.maxSize`.
- A secondary failure may point to **filesystem or shared library access instability**.
- Understanding how Snap and similar pipelines are constructed is essential for distinguishing:
  - resource allocation issues,
  - framework‑level behavior,
  - environmental or HPC instability.
- Checking peak memory usage after successful runs is a **standard and critical best practice**.


---

## Additional Resources

- [HPC Queues](https://wiki.stjude.org/display/HPCF/HPC+Queues)
- [Open OnDemand – HPC Research Cluster](https://wiki.stjude.org/pages/viewpage.action?spaceKey=RC&title=Open+OnDemand+-+HPC+Research+Cluster)
- [Parallel Processing using the future package in R](https://computing.stat.berkeley.edu/tutorial-dask-future/R-future.html)
- [A Future for R: A Comprehensive Overview](https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html)

---

#### Authors

Antonia Chroni, PhD ([@AntoniaChroni](https://github.com/AntoniaChroni))


---

## Contact

Contributions, issues, and feature requests are welcome! Please feel free to check [issues](https://github.com/stjude-dnb-binfcore/trainings/issues).

---

*These materials have been developed by the [DNB Bioinformatics core team](https://www.stjude.org/research/departments/developmental-neurobiology/shared-resources/bioinformatic-core.html) at the [St. Jude Children's Research Hospital](https://www.stjude.org/). These are open access materials distributed under the terms of the [BSD 2-Clause License](https://opensource.org/license/bsd-2-clause), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*


