<p align="center">
  <img src="img/DNB-BINF-Core-logo.png" alt="DNB-BINF-Core-logo" width="560px" />
</p>
<p align="center">
  <a href="https://www.repostatus.org/#active"><img src="https://www.repostatus.org/badges/latest/active.svg?style=for-the-badge" alt="The project has reached a stable, usable state and is being actively developed." /></a>

</p>


# scRNA-seq Snap seq pipeline: Hands-on workshop

A brief overview of single cell RNA-seq is followed by a guided, hands-on run of the [Single cell RNA Seq Snap workflow (ScRNASeqSnap)](https://github.com/stjude-dnb-binfcore/trainings/tree/main/courses/sc-rna-seq-snap-repo/). Using a small training dataset, the workflow is executed end-to-end: QC, alignment & quantification, integration, clustering, cell-type annotation, and reporting via RShiny—all within a reproducible, containerized HPC environment. Upon completion, participants receive a project folder containing standard figures, reports, an RShiny app, and comprehensive Snap pipeline documentation for further exploration.

This workshop is an initiative of our team at the [Bioinformatics Core at the Department of Developmental Neurobiology at the St. Jude Children's Research Hospital](https://www.stjude.org/research/departments/developmental-neurobiology/shared-resources/bioinformatic-core.html)





| Audience | Computational Skills | Prerequisites | Duration |
:----------|:----------|:----------|:----------|
| Biologists/Bioinformaticians | Beginner | None | 1 h|


## Learning Objectives

1. Prepare the required inputs: a properly formatted `project_metadata.tsv` and `project_parameters.Config.yaml`.
2. Launch the container on the HPC and submit or monitor your jobs.
3. Run the ScRNASeqSnap modules sequentially.
4. Review the generated results, plots, and reports.


## Target Audience

The course is intended for wet lab biologists or bioinformaticians that are interested in exploring the capabilities of the `ScRNASeqSnap` pipeline. 


> These materials were developed for a trainer-led workshop, but are also amenable to self-guided learning.

## Lessons and Materials 

* `./resources`: supplemental materials and learning resources 
* `./lectures`: [sc-rna-seq-snap-workshop-2025-10-29](https://github.com/stjude-dnb-binfcore/trainings/tree/main/courses/sc-rna-seq-snap-workshop/lectures). _Currently under construction—stay tuned for updates!_ 🚧
* St. Jude employees can access a recording from the workshop [here](https://sjcrh.sharepoint.com/:v:/r/sites/Team-DNB_BINF_Core/Shared%20Documents/Learning/Training_Recordings/snap-hands-on-workshop-2025-10-29.mp4?csf=1&web=1&e=btsaMU)

 
## Below is the main directory structure listing the analyses and data files used in this repository

```
├── img
|   └── DNB-BINF-Core-logo.png
├── lectures
├── README.md
└── resources
|   ├── snap_quickstart_guide.html
|   ├── snap_quickstart_guide.md
|__ └── links-to-resources.md
```


## Course Authors

Asha Jacob Jannu, MS ([@ashajacob29](https://github.com/ashajacob29))

## Contact

Contributions, issues, and feature requests are welcome! Please feel free to check [issues](https://github.com/stjude-dnb-binfcore/trainings/issues).

---

*These materials have been developed by the [DNB Bioinformatics core team](https://www.stjude.org/research/departments/developmental-neurobiology/shared-resources/bioinformatic-core.html) at the [St. Jude Children's Research Hospital](https://www.stjude.org/). These are open access materials distributed under the terms of the [BSD 2-Clause License](https://opensource.org/license/bsd-2-clause), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*
