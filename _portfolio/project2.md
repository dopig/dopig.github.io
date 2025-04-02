---
title: "Project 2: Dummy Analysis Cont'd"
tabs:
  - id: introduction
    title: Introduction
    content: |
      ## Overview

      This project focuses on analyzing different coverage amounts when creating a library using the Bobaseq workflow. We've converted the original workflow to Nextflow to improve scalability and reproducibility.

      ### Key Objectives

      - Evaluate the impact of sequencing depth on library quality
      - Determine optimal coverage thresholds for reliable results
      - Implement efficient data processing in Nextflow

  - id: library-outcomes
    title: Library Outcomes
    content: |
      ## Library Mapping Results

      Our library mapping analysis revealed several key insights about coverage requirements:

      ### Coverage Distribution

      We tested coverage depths ranging from 10x to 100x and found that:

      - **Low coverage (10-30x)**: Sufficient for detecting major variants but misses rare alleles
      - **Medium coverage (40-60x)**: Optimal balance between cost and detection sensitivity
      - **High coverage (70-100x)**: Diminishing returns except for detecting very rare variants

      ### Mapping Quality

      The Nextflow implementation maintained consistent mapping quality across all coverage levels, with an average mapping quality score of 38.7.

  - id: experimental-outcomes
    title: Experimental Outcomes
    content: |
      ## Experimental Results

      The experimental validation of our Nextflow-based Bobaseq workflow demonstrated significant improvements:

      ### Performance Metrics

      | Metric | Original Workflow | Nextflow Implementation | Improvement |
      |--------|-------------------|-------------------------|-------------|
      | Runtime | 4.2 hours | 1.8 hours | 57% faster |
      | Resource Usage | 24GB RAM peak | 16GB RAM peak | 33% reduction |
      | Reproducibility | Manual steps | Fully automated | Eliminated human error |

      ### Biological Findings

      Our optimized workflow enabled us to identify several previously undetected variants that may have functional significance in our target organism.
---

This is another dummy project.

<!-- This project explores the impact of different coverage depths on the quality and reliability of Bobaseq libraries. By converting the workflow to Nextflow, we've improved both performance and reproducibility.

Select a tab above to view detailed information about this project. -->