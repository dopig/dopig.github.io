---
title: "Nextflow for Overexpression Barseq"
---
## Project Overview

This project demonstrates a modular Nextflow DSL2 pipeline for simulating and analyzing barcoded overexpression libraries using barcode amplicon sequencing (barseq). It covers workflows from synthetic library design to barcode mapping, experiment simulation, and fitness analysis. All steps are containerized using Docker for portability and reproducibility, and the workflows are capable of running locally or on AWS Batch.

[GitHub Repository](https://github.com/dopig/nf-overexpression-barseq)

---

## Functional Genomics Screening

Functional genomics screening in bacteria often relies on libraries of plasmids carrying genomic fragments. Traditional cosmid libraries (~40 kb inserts) are highly effective, but barcoded plasmid overexpression libraries have recently been employed under the name [Boba-seq](https://www.nature.com/articles/s41467-024-50124-3). Screening with barcoded plasmids offers several advantages (and a few drawbacks) relative to cosmid-based screening:

| Feature                  | Winner  | Reason |
|---------------------------|---------|--------|
| **Library Construction**  | Cosmids | Barseq requires more work and cost to map libraries. Getting the appropriate genome fragment sizes for cosmids can be challenging. |
| **Resolution**            | Plasmids | Plasmids can pinpoint the responsible gene (as demonstrated below), while cosmids include a dozen or more genes per insert. |
| **Phenotype Sensitivity** | Plasmids | Barseq detects much smaller phenotype differences. |
| **Promoter Strength**     | Plasmids | Overexpression plasmids can use strong, inducible promoters; cosmids rely on native promoters, which may be weak. |
| **Genetic Context**       | Tied    | Plasmids may miss neighboring partner genes; cosmids may include antagonistic neighbors. |

See the diagram [Boba-seq workflow for high-throughput functional screens](https://pmc.ncbi.nlm.nih.gov/articles/PMC11300592/figure/Fig1/), or the paper it comes from, for an excellent depiction of the process of building and using barcoded plasmid overexpression libraries.

Barseq libraries require two distinct phases of sequencing:
- **Mapping**: Link random ~20 bp barcodes to ~3 kb genomic inserts. Only needs to occur once for each library.
- **Experimentation**: Sequence only the barcodes to quantify plasmid abundances across conditions. Can be performed many times on many different occasions.

This setup allows many conditions to be tested cheaply once the library is built and mapped.

---

## My Background and Motivation
My background includes extensive experience working with both cosmid libraries and barseq. My work includes cosmid screens in bacteria[^1] and yeast[^2], [RB-TnSeq](https://journals.asm.org/doi/full/10.1128/mbio.00306-15) on plant-colonizing microbes (unpublished), and in non-transgenic barseq in cereal crop field trials[^3].

I wanted to create a demonstration Nextflow DSL2 pipeline for processing barseq data. Since real Bobaseq mapping or barseq datasets were not publicly available, I also built synthetic data generation workflows.

---

## Methods

### Synthetic Library Design

I selected the medium/low copy number plasmid [pBbA1k](https://www.addgene.org/35336/), featuring a strong IPTG-inducible Trc promoter, as the backbone. Barcodes (~20 bp) were introduced near the multiple-cloning site, enabling PacBio amplicon sequencing to link barcodes to inserts. In the diagram below, the genomic inserts are cloned into the PmlI site (salmon-colored arrow) at 3484 bp, and the barcode is inserted at 3584 bp (between the yellow arrows).

![Schematic of pBbA1k-bc barcode and insert sites](plasmid-region.png)

---

### Workflow Breakdown

Each bullet point below corresponds to a Nextflow process that runs in its own Docker container.

#### Library Simulation
- **ncbi_download.sh**: Downloads genome assembly files.
- **simulate_library.py**: Simulates random barcode insertion and genomic fragment cloning.
- **[PBSIM3](https://github.com/yukiteruono/pbsim3)**: Simulates a BAM file of PacBio HiFi reads of the library sequences.

#### Library Analysis (Mapping)
- **[CCS](https://ccs.how/)**: Generates consensus FASTQ reads.
- **[Boba-seq](https://github.com/OGalOz/Boba-seq)**: Links barcodes to genomic inserts.

#### Barseq Simulation
- **simulate_barseq_reads.py**: Simulates Illumina barcode sequencing reads for experimental samples based on user parameters (e.g., number of winners, count variability).

#### Barseq Analysis
- **[MultiCodes](https://bitbucket.org/berkeleylab/feba)**: Quantifies barcode read counts.
- **[BobaseqFitness](https://github.com/morgannprice/BobaseqFitness)**: Computes relative fitness of plasmid inserts and select gene winners.
- **R Scripts**: Generate fitness plots of winning genes.

---

## Testing and Results

### Test Genomes and Libraries

- ***Escherichia* phage Mu** (36.7 kb), chosen for rapid initial testing with a 40-plasmid simulated library (see [mu.yaml](https://github.com/dopig/nf-overexpression-barseq/blob/nfsim/shared/example/config/mu.yaml)).
- ***Candidatus* Pelagibacter ubique** (1.3 Mb), chosen for its minimal genome with a 10,000-plasmid simulated library (see [pelagibacter-large.yaml](https://github.com/dopig/nf-overexpression-barseq/blob/nfsim/shared/example/config/pelagibacter-large.yaml)).

### Workflow Performance

- **Local testing**: Running *Pelagibacter* workflows locally took ~100 minutes on my laptop, with ~87% of the time spent on CCS.
- **AWS Batch**:
  - Default settings (2 CPUs/process): ~9 hours for *Pelagibacter*.
  - Default except 36 CPUs for CCS: Entire workflow completed in ~50 minutes. The CCS spot instance only cost $0.34, and CCS is no longer the slowest part of the workflow (see plot below).

![Plot of workflow timing per process with optimized CCS](optimized-ccs.jpg)

### Exploring Library Size Effects
When simulated libraries are generated proportional to their genome sizes (e.g. [pelagibacter-large.yaml](https://github.com/dopig/nf-overexpression-barseq/blob/nfsim/shared/example/config/pelagibacter-large.yaml)), BobaseqFitness perfectly selects the simulated winners. However, when the simulated library size is 10x reduced (e.g. [pelagibacter-small.yaml](https://github.com/dopig/nf-overexpression-barseq/blob/nfsim/shared/example/config/pelagibacter-small.yaml)) fitness detection is significantly degraded. Instead of precisely identifying 10 winners, BobaseqFitness only confidently identified 3. Furthermore, each of these "winners" was flawed, having drifted by one or two open reading frames from the true simulated winner.

Example:
- Real winner: **SAR11_RS00985**
- Misidentified winner: **SAR11_RS00995**

![Plot showing misassigned winner region](SAR11_RS00995.svg)

Simulations help design proper experimental scales.

---

## Challenges and Lessons Learned

- **Synthetic Data Generation**: Required simulation of both PacBio and Illumina reads.
- **Resource Optimization**: Custom fork limits in AWS Batch would avoid time wasted initiating instances for quick steps that are run many times (e.g. MultiCodes).
- **Custom AMI for AWS Batch**: New ECS-Optimized AMIs now contain awscli, but Nextflow still required a custom awscli (re)install for proper operation.
- **Docker Images**: Initial slim Docker builds needed revision to support Nextflow and AWS compatibility.

---

## Summary

This project described construction of a fully containerized, scalable workflow for simulating and analyzing barseq experiments, with a focus on reproducibility and cloud compatibility. It provides a foundation for real experimental analyses, as well as a framework for synthetic dataset generation to explore design choices in barseq screening experiments.

**Check out the full project on [GitHub](https://github.com/dopig/nf-overexpression-barseq)!**

---

*Questions or ideas? Feel free to [reach out](/contact)!*

---

## References
[^1]: **DA Higgins**, JM Gladden, JA Kimbrel, BA Simmons, SW Singer, & MP Thelen. Guanidine Riboswitch-Regulated Efflux Transporters Protect Bacteria against Ionic Liquid Toxicity. *Journal of Bacteriology*. 201, e00069-19 (2019). [https://doi.org/10.1128/jb.00069-19](https://doi.org/10.1128/jb.00069-19)

[^2]: **DA Higgins**, MKM Young, M Tremaine, M Sardi, JM Fletcher, M Agnew, L Liu, Q Dickinson, D Peris, RL Wrobel, CT Hittinger, AP Gasch, SW Singer, BA Simmons, R Landick, MP Thelen, & TK Sato. Natural Variation in the Multidrug Efflux Pump SGE1 Underlies Ionic Liquid Tolerance in Yeast. *Genetics*. 210, 219-234 (2018). [https://doi.org/10.1534/genetics.118.301161](https://doi.org/10.1534/genetics.118.301161)

[^3]: **Douglas Higgins** & Andrew Prior. "Plant colonization assays using natural microbial barcodes." *Patent application* WO2020146372A1. [https://patents.google.com/patent/WO2020146372A1/en](https://patents.google.com/patent/WO2020146372A1/en)
