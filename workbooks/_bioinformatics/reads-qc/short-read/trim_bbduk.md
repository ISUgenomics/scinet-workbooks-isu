---
title: K-mer-based adapter and contaminant trimming/filtering with BBDUK
description: "Use BBDUK to trim adapters and remove contaminant sequence from short reads when k-mer-based filtering is more appropriate than quality trimming alone."
svg: /genomics.svg
author: Aleksandra Badaczewska
tags: [quality control, short-read, BBDUK, contaminant filtering]
language: Bash

header:
  overlay_image: 07-wrangling/assets/img/07_data_acquisition_banner.png
index: 
order: 3
wg: Bioinformatics
type: interactive tutorial

related:
  - "[Sequencing Reads Quality Control](/bioinformatics/reads-qc/)"
  - "[Short-read QC and trimming](/bioinformatics/reads-qc/short-read/)"
  - "[Illumina short-read quality check with FastQC](/bioinformatics/reads-qc/short-read/qc_fastqc)"

references:
  - "[BBDUK documentation](https://bbmap.org/tools/bbduk)"

tools:
  bbduk:
    name: BBDUK
    exec: bbduk.sh
    help_cmd: bbduk.sh

environment:
  name: bbmap_40.00
  path: /reference/workbook/bioinformatics/env/conda

objectives:
  - Use `BBDUK` to remove adapter sequence and unwanted k-mer-matched content from Illumina reads.
  - Choose parameters that match the actual QC problem instead of applying aggressive defaults.
  - Combine trimming and filtering decisions without losing more read length than necessary.
  - Validate whether k-mer-based cleanup improved the read set for downstream analysis.

applications:
  - Removing standard adapters when contamination is accompanied by additional unwanted sequence content.
  - Filtering technical or biological contaminants identified through overrepresented-sequence or matching-based review.
  - Cleaning read sets before alignment, assembly, or taxonomic profiling when simple tail trimming is not enough.
  - Comparing k-mer-based cleanup against `Cutadapt` or `fastp` for datasets with more complex contamination profiles.

terms:
  - quality control
  - read preprocessing
  - read trimming
  - adapter contamination
  - term: k-mer matching
    definition: Sequence matching based on fixed-length substrings used to detect adapters, contaminants, or unwanted sequence content without full alignment.
  - term: contaminant filtering
    definition: Removal of reads or read segments that match unwanted technical or biological sequence content.

materials:
  - "Example Arabidopsis short-read inputs for this tutorial are provided under `/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/`."
  - "Perform the exercises from `/90daydata/shared/$USER/` so trial outputs stay in temporary user space."
  - "Reference filtered outputs can be organized under dataset-specific result paths such as `/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/reads_qc/bbduk/`."
  - "Complete a raw-read QC check first so the filtering targets reflect the actual problem in the data."

overview: [objectives, applications, terminology, materials]
---

## Overview

This tutorial focuses on `BBDUK` as a **k-mer-based cleanup tool** for short reads that need more than generic end trimming.
Using the **Arabidopsis** dataset from the [short-read QC tutorial](/bioinformatics/reads-qc/short-read/qc_fastqc), you will decide whether the QC signal points to adapter carryover alone or to a broader contaminant-filtering problem that justifies matching-based removal.

{% include overviews %}

## Getting Started

{% include segment/getting_started time="04:00:00" cores="8" %}

**Tutorial Steps:**
1. [Prepare the practice workspace](#prepare-the-practice-workspace) and inspect the input set for the cleanup task.
1. [Get the dataset](#get-the-dataset) and confirm how you will access the raw reads.
1. [Decide what should be removed](#decide-what-should-be-removed) from prior QC evidence.
1. [Run BBDUK on the Arabidopsis reads](#run-bbduk-on-the-arabidopsis-reads) with appropriate k-mer settings for adapters or contaminants.
1. [Inspect the filtered outputs](#inspect-the-filtered-outputs) and logs to see what changed.
1. [Validate the cleanup with follow-up QC](#validate-the-cleanup-with-follow-up-qc) and decide whether the dataset improved enough for downstream use.

## Tutorial Steps

<div class="process-list" markdown='1'>

### Prepare the practice workspace

{% include setup/practice_workspace
  workdir="tutorials/short_reads_qc/bbduk" %}

### Get the dataset

{% include setup/get_dataset
  dataset_path="/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/"
  dataset="arabidopsis_prjna348194"
  mode="reference"
  prior_tutorial_title="Illumina short-read quality check with FastQC"
  prior_tutorial_url="/bioinformatics/reads-qc/short-read/qc_fastqc"
  list="true" %}

<pre><small>total 28G
SRR27003605_1.fastq.gz  SRR27003608_1.fastq.gz  SRR27003611_1.fastq.gz
SRR27003605_2.fastq.gz  SRR27003608_2.fastq.gz  SRR27003611_2.fastq.gz
SRR27003606_1.fastq.gz  SRR27003609_1.fastq.gz  SRR27003612_1.fastq.gz
SRR27003606_2.fastq.gz  SRR27003609_2.fastq.gz  SRR27003612_2.fastq.gz
SRR27003607_1.fastq.gz  SRR27003610_1.fastq.gz  SRR27003613_1.fastq.gz
SRR27003607_2.fastq.gz  SRR27003610_2.fastq.gz  SRR27003613_2.fastq.gz
</small></pre>

In this tutorial, those reads are used as a realistic case for deciding whether adapter-only cleanup is enough or whether k-mer-based contaminant filtering is justified.

### Decide what should be removed

### Run BBDUK on the Arabidopsis reads

{% include tool_summary tool_key="bbduk" %}

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="Find and load or install the tool" controls="bbduk-tool-setup" class="outline" icon=false %}
<div id="bbduk-tool-setup" class="accordion_content" markdown='1' hidden>

{% include segment/find_tool env_root="bioinformatics" %}

{% include setup/conda known_conda="true" activate="true" tool_key="bbduk" tool_context="from the BBMap suite" %}

{% include setup/tool_verify tool_key="bbduk" %}

</div>

{% include accordion title="Run one sample interactively first to learn the syntax and validate paths" controls="bbduk-live-first" class="outline" icon=false %}
<div id="bbduk-live-first" class="accordion_content" markdown='1' hidden>

Start with **one paired-end sample** in the interactive session. `BBDUK` is flexible and can remove very different kinds of sequence content, so the first run should confirm that the selected k-mer strategy creates non-empty outputs and removes what you intended rather than more than you intended.

For `BBDUK`, you need both the paired-end read files and a reference adapter file for k-mer matching. The main choices are the reference sequences you want to match against, the k-mer sensitivity settings, the minimum read length to retain after trimming, and how many CPU threads to use.

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| `ref=` | Reference adapter or contaminant sequences to match | `ref=adapters.fa` |
| `in1=`, `in2=` | Input paired-end `FASTQ` files | `in1=reads_1.fastq.gz`<br>`in2=reads_2.fastq.gz` |
| `out1=`, `out2=` | Output files for cleaned read 1 and read 2 | `out1=reads_1.clean.fastq.gz`<br>`out2=reads_2.clean.fastq.gz` |
| `ktrim=` | Trim mode for matched k-mers | `ktrim=r` |
| `k=` | K-mer length used for matching | `k=23` |
| `mink=` | Smaller k-mer size allowed near read ends | `mink=11` |
| `hdist=` | Allowed mismatches in k-mer matching | `hdist=1` |
| `tpe`, `tbo` | Keep paired-end trimming synchronized and use pair overlap to refine adapter trimming | `tpe tbo` |
| `qtrim=`, `trimq=` | Quality-trim read ends before length filtering | `qtrim=rl trimq=20` |
| `minlen=` | Drop reads shorter than this after cleanup | `minlen=36` |
| `threads=` | Number of CPU threads used by `BBDUK` | `threads=4` |

</div>

### Run a command

Use one paired-end sample directly from the [dataset location](#get-the-dataset). For a first adapter-focused cleanup trial, use a local adapter reference file, trim matches from the right end of reads, and retain reads at least 36 bases long.

Before running the command, make sure you know the path to the adapter reference file you want `BBDUK` to use. `BBDUK` requires reference adapter sequences for matching and trimming, and in this example they are provided under `/reference` with the tutorial dataset. For your own project, use the adapter file supplied with your sequencing kit or the adapter sequences published by the platform or library-preparation vendor:

```bash
cd /90daydata/shared/$USER/tutorials/short_reads_qc/bbduk

R1=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/SRR27003605_1.fastq.gz
R2=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data/SRR27003605_2.fastq.gz
ADAPTERS=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/adapters/adapters.fa

bbduk.sh \
  in1=${R1} in2=${R2} \
  out1=SRR27003605_1.bbduk.fastq.gz out2=SRR27003605_2.bbduk.fastq.gz \
  ref=${ADAPTERS} \
  ktrim=r k=23 mink=11 hdist=1 tpe tbo \
  qtrim=rl trimq=20 minlen=36 threads=4
```

### Validate the output before scaling up

First, confirm that the filtered paired-end files were created:

```bash
ls -lh SRR27003605_1.bbduk.fastq.gz SRR27003605_2.bbduk.fastq.gz
```

*Both mate files should be present and non-empty.*

<pre><small></small></pre>

Next, confirm that the compressed filtered reads are readable and not truncated:

```bash
gzip -t SRR27003605_1.bbduk.fastq.gz SRR27003605_2.bbduk.fastq.gz && echo "gzip check passed"
```

<pre><small></small></pre>

Then compare the raw inputs and cleaned outputs with `seqkit stats`:

```bash
module load seqkit
seqkit version  # save version in the README or docs for your project
seqkit stats ${R1} ${R2} SRR27003605_1.bbduk.fastq.gz SRR27003605_2.bbduk.fastq.gz
```

*Confirm that the retained reads still look usable and that the cleanup level matches the adapter or contaminant problem you set out to address.*

<pre><small></small></pre>

</div>

{% include accordion title="Scale up with SLURM to automate the task for all samples" controls="bbduk-slurm-all" class="outline" icon=false %}
<div id="bbduk-slurm-all" class="accordion_content" markdown='1' hidden>

{% include segment/cli_to_slurm tool="BBDUK" %}

1. In your practice workspace, create a SLURM script file and open it for editing:
```bash
cd /90daydata/shared/$USER/tutorials/short_reads_qc/bbduk
touch bbduk_batch.sh
nano bbduk_batch.sh
```

1. Copy and paste the following script body. Before saving it, inspect the path settings carefully. Use `pwd` in your current workspace if you want to confirm the absolute working-directory path.
```bash
    #!/bin/bash
    #SBATCH --job-name=bbduk_arabidopsis
    #SBATCH --partition=<value>                      # EDIT partition; Ceres: ceres, scavenger; Atlas: atlas
    #SBATCH --nodes=1
    #SBATCH --ntasks=4
    #SBATCH --account=<account>                      # EDIT ACCOUNT, provide your SCINet project account
    #SBATCH --mem=16G
    #SBATCH --time=04:00:00
    #SBATCH --output=bbduk_batch_%j.out

    source activate /reference/workbook/bioinformatics/env/conda/bbmap_40.00

    INPUT_DIR=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/00_raw_data
    WORKDIR=/90daydata/shared/$USER/tutorials/short_reads_qc/bbduk
    ADAPTERS=/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/adapters/adapters.fa

    for R1 in "${INPUT_DIR}"/*_1.fastq.gz; do
        SAMPLE=$(basename "${R1}" _1.fastq.gz)
        R2="${INPUT_DIR}/${SAMPLE}_2.fastq.gz"
        bbduk.sh \
          in1="${R1}" in2="${R2}" \
          out1="${WORKDIR}/${SAMPLE}_1.bbduk.fastq.gz" out2="${WORKDIR}/${SAMPLE}_2.bbduk.fastq.gz" \
          ref="${ADAPTERS}" \
          ktrim=r k=23 mink=11 hdist=1 tpe tbo \
          qtrim=rl trimq=20 minlen=36 threads=4
    done
```

    - <small>`BBDUK` requires a reference adapter file for k-mer matching.</small>
    - <small>`BBDUK` enables multithreading with option `threads=4`, that must match the number of requested cores `#SBATCH --ntasks=4`. Here, all 4 cores are used by one trimming command on the current read pair, so extra samples wait for the next loop iteration.</small>
    - <small>The batch settings here stay intentionally close to the interactive adapter-focused trial so you can compare outputs consistently across samples.</small>

1. Submit it with:
```bash
sbatch bbduk_batch.sh
```

1. Then check whether the job is queued or running and inspect the batch log file for any immediate errors:
```bash
squeue -u $USER
ls -lh bbduk_batch_*.out
```

1. Optionally, wait a while to confirm that the expected cleaned read files are being produced in your practice workspace.
```bash
ls -lh *.bbduk.fastq.gz
```

</div>
</div>

### Inspect the filtered outputs

{% include segment/ref_outputs
  reference_results_path="/reference/workbook/bioinformatics/dataset/reads_short/arabidopsis_PRJNA348194/reads_qc/bbduk/" %}

### Validate the cleanup with follow-up QC

</div>

## Summary
