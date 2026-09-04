---
title: Illumina short-read quality check with FastQC
description: "Run FastQC on Illumina short reads, inspect the major report modules, and decide which warnings matter before trimming."
svg: /genomics.svg
author: Aleksandra Badaczewska
tags: [quality control, short-read, Illumina, FastQC]
language: Bash

header:
  overlay_image: 07-wrangling/assets/img/07_data_acquisition_banner.png
index: 
order: 2
wg: Bioinformatics
type: interactive tutorial

related:
  - "[Sequencing Reads Quality Control](/bioinformatics/reads-qc/)"
  - "[Short-read QC and trimming](/bioinformatics/reads-qc/short-read/)"
  - "[Workspace setup for bioinformatics](/bioinformatics/resources/workspace_setup)"

references:
  - "[FastQC project page](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)"

tools:
  fastqc:
    name: FastQC
    exec: fastqc
    module: fastqc
    version: 0.12.1

objectives:
  - Run `FastQC` on single-end or paired-end Illumina `FASTQ` files in a reproducible workspace.
  - Interpret the most informative FastQC modules before making any trimming decisions.
  - Distinguish routine warning patterns from signals that justify preprocessing.
  - Compare the QC needs of read 1 and read 2 before choosing downstream cleanup steps.

applications:
  - Checking raw RNA-seq, variant-calling, or metagenomic short-read inputs before alignment or quantification.
  - Identifying low-quality tails, adapter carryover, GC shifts, or duplication patterns that need follow-up.
  - Documenting baseline read quality before any trimming workflow changes the data.
  - Deciding whether to continue with `fastp`, `Cutadapt`, `BBDUK`, or `Trimmomatic (legacy)` after QC review.

terms:
  - quality control
  - FASTQ
  - base quality
  - adapter contamination
  - overrepresented sequence
  - read layout

materials:
  - "Example paired-end Arabidopsis `FASTQ` files for this tutorial are provided under `/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/`."
  - "Create and use a practice workspace under `/90daydata/shared/$USER/` before running the commands."
  - "Reference QC outputs for comparison can be organized in parallel dataset folders such as `/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/reads_qc/fastqc/` and `/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/reads_qc/multiqc/`."
  - "If you need a refresher on workspace conventions, review [Workspace setup for bioinformatics](/bioinformatics/resources/workspace_setup)."

overview: [objectives, applications, terminology, materials]
---

## Overview

This tutorial shows how to inspect raw **Illumina short reads** with `FastQC` before any trimming or filtering is applied.
You will use an **Arabidopsis** short-read dataset to identify which QC modules provide actionable evidence and which warnings should be interpreted in context rather than corrected automatically.

{% include overviews %}

## Getting Started

{% include segment/getting_started time="02:00:00" cores="2" %}

**Tutorial Steps:**
1. [Prepare the practice workspace](#prepare-the-practice-workspace) for raw-read quality control (QC).
1. [Get the dataset](#get-the-dataset) and confirm the input files you will inspect.
1. [Run FastQC on raw reads](#run-fastqc-on-raw-reads) and generate per-sample reports.
1. [Review the report modules](#review-the-report-modules) for quality, adapter, duplication, and sequence-composition patterns.
1. [Compare read pairs and sample patterns](#compare-read-pairs-and-sample-patterns) to identify mate-specific issues before trimming.
1. [Decide on the next preprocessing step](#decide-on-the-next-preprocessing-step) and document whether trimming is justified.

## Tutorial Steps

<div class="process-list" markdown='1'>

### Prepare the practice workspace

{% include setup/practice_workspace workdir="tutorials/short_reads_qc/fastqc" %}

### Get the dataset

{% include setup/get_dataset
  dataset_path="/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/"
  dataset="arabidopsis_prjna348194"
  mode="reference"
  list="true" %}

<pre><small>total 28G
-rw-r-----. 1 scinet-workbook 1.6G SRR27003605_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.6G SRR27003605_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.7G SRR27003606_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.7G SRR27003606_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.6G SRR27003607_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.6G SRR27003607_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.8G SRR27003608_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.8G SRR27003608_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.5G SRR27003609_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.5G SRR27003609_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.5G SRR27003610_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.5G SRR27003610_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.2G SRR27003611_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.1G SRR27003611_2.fastq.gz
-rw-r-----. 1 scinet-workbook 2.0G SRR27003612_1.fastq.gz
-rw-r-----. 1 scinet-workbook 2.0G SRR27003612_2.fastq.gz
-rw-r-----. 1 scinet-workbook 1.5G SRR27003613_1.fastq.gz
-rw-r-----. 1 scinet-workbook 1.5G SRR27003613_2.fastq.gz
</small></pre>

In this QC tutorial, you are using the raw paired-end reads directly so you can inspect the original quality profile before any trimming or filtering changes the data.

### Run FastQC on raw reads

{% include tool_summary tool_key="fastqc" %}

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="Find and load or install the tool" controls="fastqc-tool-setup" class="outline" icon=false %}
<div id="fastqc-tool-setup" class="accordion_content" markdown='1' hidden>

{% include segment/find_tool env_root="bioinformatics" %}

{% include setup/module tool_key="fastqc" known="true" %}
{% include setup/tool_verify tool_key="fastqc" %}
</div>

{% include accordion title="Run one sample interactively first to learn the syntax and validate paths" controls="fastqc-live-first" class="outline" icon=false %}
<div id="fastqc-live-first" class="accordion_content" markdown='1' hidden>

Start with **one paired-end sample** (`R1` + `R2`) in the interactive session. `FastQC` is usually quick enough that you can watch it finish, confirm that the `.html` and `.zip` outputs are created, and check that they are written to the expected directory before repeating the same pattern for the rest of the files.

For `FastQC`, the main choices are which input files you want to inspect together, how many CPU threads you want to use, and which output directory should collect the reports.

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| `--threads` | Number of CPU threads used by `FastQC` | `--threads 2` |
| `--outdir` | Directory where the `.html` and `.zip` reports will be written | `--outdir fastqc_SRR27003605` |
| `mkdir -p` | Create the output directory before writing reports into it | `mkdir -p fastqc_SRR27003605` |
| input files | One or more `FASTQ` files to summarize | `SRR27003605_1.fastq.gz SRR27003605_2.fastq.gz` |

</div>

### Run a command

Use one paired-end sample directly from the [dataset location](#get-the-dataset). For a first QC pass, keep the output directory tied to the sample name so the reports stay easy to match to the input reads:

```bash
cd /90daydata/shared/$USER/tutorials/short_reads_qc/fastqc

R1=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/SRR27003605_1.fastq.gz
R2=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/SRR27003605_2.fastq.gz
OUTDIR=fastqc_SRR27003605

mkdir -p ${OUTDIR}
fastqc --threads 2 --outdir ${OUTDIR} ${R1} ${R2}
```

<pre><small>real    4m25.391s
user    8m33.432s
sys     0m6.516s
</small></pre>

### Validate the output before scaling up

First, confirm that the expected `FastQC` reports were created in your practice workspace:

```bash
ls -lh ${OUTDIR}
```

*You should see one `.html` report and one `.zip` archive for each mate.*

<pre><small>534K SRR27003605_1_fastqc.html
525K SRR27003605_1_fastqc.zip
531K SRR27003605_2_fastqc.html
525K SRR27003605_2_fastqc.zip
</small></pre>

Next, confirm that both mate reports exist:

```bash
ls -lh ${OUTDIR}/*_fastqc.html
```

<pre><small>534K fastqc_SRR27003605/SRR27003605_1_fastqc.html
531K fastqc_SRR27003605/SRR27003605_2_fastqc.html
</small></pre>

Then, inspect the module summary inside one of the report archives:

```bash
unzip -p ${OUTDIR}/SRR27003605_1_fastqc.zip SRR27003605_1_fastqc/summary.txt
```

*Use this summary to see if all modules were assessed.*

<pre><small>PASS    Basic Statistics                SRR27003605_1.fastq.gz
PASS    Per base sequence quality       SRR27003605_1.fastq.gz
PASS    Per tile sequence quality       SRR27003605_1.fastq.gz
PASS    Per sequence quality scores     SRR27003605_1.fastq.gz
FAIL    Per base sequence content       SRR27003605_1.fastq.gz
PASS    Per sequence GC content         SRR27003605_1.fastq.gz
PASS    Per base N content              SRR27003605_1.fastq.gz
WARN    Sequence Length Distribution    SRR27003605_1.fastq.gz
FAIL    Sequence Duplication Levels     SRR27003605_1.fastq.gz
PASS    Overrepresented sequences       SRR27003605_1.fastq.gz
FAIL    Adapter Content                 SRR27003605_1.fastq.gz
</small></pre>

</div>
{% include accordion title="Scale up with SLURM to automate the task for all samples" controls="fastqc-slurm-all" class="outline" icon=false %}
<div id="fastqc-slurm-all" class="accordion_content" markdown='1' hidden>

{% include segment/cli_to_slurm tool="FastQC" %}

1. In your practice workspace, create a SLURM script file and open it for editing:
```bash
cd /90daydata/shared/$USER/tutorials/short_reads_qc/fastqc
touch fastqc_batch.sh
nano fastqc_batch.sh
```

1. Copy and paste the following script body. Before saving it, inspect the path settings carefully. Use `pwd` in your current workspace if you want to confirm the absolute working-directory path.
```bash
   #!/bin/bash
   #SBATCH --job-name=fastqc_arabidopsis
   #SBATCH --partition=<partition>                  # EDIT PARTITION; Ceres: ceres; Atlas: atlas
   #SBATCH --nodes=1
   #SBATCH --ntasks=2
   #SBATCH --account=<account>                      # EDIT ACCOUNT, provide your SCINet project account
   #SBATCH --mem=8G
   #SBATCH --time=02:00:00
   #SBATCH --output=fastqc_batch_%j.log

   module load fastqc

   INPUT_DIR=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data
   WORKDIR=/90daydata/shared/$USER/tutorials/short_reads_qc/fastqc
   OUTDIR=${WORKDIR}/fastqc_all

   mkdir -p "${OUTDIR}"
   fastqc --threads 2 --outdir "${OUTDIR}" "${INPUT_DIR}"/*.fastq.gz
```

    - <small>`FastQC` accepts multiple input files in one command, so this batch version writes all per-file reports into one output directory for later comparison and `MultiQC` aggregation.</small>
    - <small>`FastQC` enables multithreading with option `--threads 2`, that must match the number of requested cores `#SBATCH --ntasks=2`. This sets how many input `FASTQ` files are processed at once, up to one core per file; extra files wait in queue, and more cores than files usually give no benefit.</small>
    - <small>`--mem=8G` and `--time=02:00:00` are starter values for report generation on this dataset; adjust them only if your practice run shows the job needs more resources.</small>

1. Submit it with:
```bash
sbatch fastqc_batch.sh
```

1. Then check whether the job is queued or running and inspect the batch log file for any immediate errors:
```bash
squeue -u $USER
```
   
   <pre><small>   JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
   21880008     ceres fastqc_a alex.bad  R       1:49      1 ceres20-mem-2
   </small></pre>

   ```bash
   less fastqc_batch_*.log
   ```

   <pre><small>Started analysis of SRR27003605_1.fastq.gz
   Started analysis of SRR27003605_2.fastq.gz
   Approx 5% complete for SRR27003605_1.fastq.gz
   ...
   </small></pre>

1. Optionally, wait a while to confirm that the expected report files are being produced in your practice workspace.
```bash
ls -lh fastqc_all/
```

</div>
</div>

### Review the report modules

{% include segment/ref_outputs
  reference_results_path="/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/reads_qc/fastqc/" %}

*Use this summary to see which modules passed, warned, or failed before you decide whether trimming is justified.*
*If read 2 shows systematically worse quality or stronger adapter signal than read 1, note that before choosing the next cleanup tool.*
### Compare read pairs and sample patterns
### Decide on the next preprocessing step
</div>
## Summary
