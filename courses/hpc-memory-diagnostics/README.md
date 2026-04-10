# HPC Memory Diagnostics & Troubleshooting

This folder contains documentation and practical guidance for **diagnosing, understanding, and resolving memory-related issues on HPC systems**, with a focus on LSF-based environments and containerized workflows commonly used in single‑cell and multi‑omic analyses.

The goal of this documentation is to help users:
- distinguish between true memory exhaustion and framework/environment-related failures,
- interpret LSF resource usage summaries correctly,
- apply best practices for memory allocation, testing, and reproducible re‑runs.

---

## Scope

This documentation covers:
- Identifying common **memory-related failure modes** on HPC (e.g., OOM kills, silent crashes, framework limits).
- Interpreting **LSF resource usage summaries** (Max Memory, Delta Memory, swap, runtime vs CPU time).
- Differentiating between:
  - insufficient memory requests,
  - framework-level limits (e.g., `future.globals.maxSize`),
  - threading/oversubscription issues,
  - filesystem or I/O–related crashes.
- Recommended **debugging workflows** using interactive and batch jobs.
- Best practices for **stable, reproducible re‑runs**.

This guide is intended for **practitioners running production pipelines**, not as an exhaustive HPC systems manual.

---

## Intended Audience

- Bioinformaticians and data scientists running large-scale analyses on HPC.
- Pipeline developers maintaining production workflows.
- Users troubleshooting failed or unstable HPC jobs.
- Core facility staff supporting single‑cell and multi‑omic analyses.

---

### Below is the main directory structure listing the files used in this directory

```
├── md-files
├── README.md
└── resources
|  ├── diagnosing-hpc-memory-issues.html
|_ └── lsf-vs-interactive-nodes-memory-behavior.html
```


---

#### Authors

Antonia Chroni, PhD ([@AntoniaChroni](https://github.com/AntoniaChroni))


### Contact

Contributions, issues, and feature requests are welcome! Please feel free to check [issues](https://github.com/stjude-dnb-binfcore/trainings/issues).

---

*These tools and pipelines have been developed by the Bioinformatic core team at the [St. Jude Children's Research Hospital](https://www.stjude.org/). These are open access materials distributed under the terms of the [BSD 2-Clause License](https://opensource.org/license/bsd-2-clause), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*
