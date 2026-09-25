---
title: "Long-read ONT DNA: basecalling from POD5 and read QC with Dorado"
description: "Use Dorado for platform-native preprocessing and read-level QC of long-read ONT DNA data starting from raw-signal POD5 input."
svg: /genomics.svg
author: Aleksandra Badaczewska
tags: [quality control, long-read, Oxford Nanopore, Dorado]
language: Bash

header:
  overlay_image: 07-wrangling/assets/img/07_data_acquisition_banner.png
index:
order: 1
wg: Bioinformatics
type: interactive tutorial

related:
  - "[Sequencing Reads Quality Control](/bioinformatics/reads-qc/)"
  - "[Long-read QC and trimming](/bioinformatics/reads-qc/long-read/)"
  - "[Oxford Nanopore](/bioinformatics/reads-qc/long-read/ont/)"
  - "[Long-read ONT quality check with NanoPlot](/bioinformatics/reads-qc/long-read/ont/qc_nanoplot)"

references:
  - "[Dorado documentation](https://software-docs.nanoporetech.com/dorado/latest/)"
  - "[Dorado simplex basecalling](https://software-docs.nanoporetech.com/dorado/latest/basecaller/simplex/)"
  - "[Dorado summary command](https://software-docs.nanoporetech.com/dorado/latest/basecaller/summaries/)"
  - "[Dorado read trimming](https://software-docs.nanoporetech.com/dorado/1.1.1/basecaller/read_trimming/)"

tools:
  python:
    module: python_3
    version: 3.11.1
  dorado:
    name: Dorado
    exec: dorado
    module: dorado
    version: 2.0.1
  pod5:
    name: pod5
    exec: pod5

environment:
  name: pod5_0.3.47
  path: /reference/workbook/bioinformatics/env/venv

tutorial:
  root_practice: /90daydata/shared/$USER
  workdir: tutorials/long_reads_qc/dorado_dna
  data_path: /reference/workbook/bioinformatics/dataset/reads_long/ont/dna_r10.4.1_e8.2_400bps_5khz
  pilot_sample: dna_r10.4.1_e8.2_400bps_5khz-FLO_PRO114M-SQK_LSK114_XL-5000.pod5
  sample: dna_r10.4.1_e8.2_400bps_5khz-FLO_PRO114M-SQK_MLK114_96_XL-5000.pod5
  dataset: ont_dna_dorado_pod5

tutorial2:
  root_practice: /90daydata/shared/$USER
  workdir: tutorials/long_reads_qc/dorado_dna
  data_path: /reference/workbook/bioinformatics/dataset/reads_long/ont/monkeypox_SQB000004/00_raw_data
  scale_sample: 10244_MPOX-67.pod5
  scale_read_ids: 10244_MPOX-67_ids
  scale_read_ids_5: 10244_MPOX-67_ids_5
  dataset: mpox_sqb000004

objectives:
  - Use `Dorado` as the ONT DNA entry point when the available input is raw-signal `POD5`.
  - Distinguish platform-native basecalling from later read-level QC decisions on exported reads.
  - Use Dorado read-level summaries to decide whether additional cleanup such as trimming is justified.
  - "[RNA trimming](https://software-docs.nanoporetech.com/dorado/1.1.1/basecaller/read_trimming/) is always done along basecalling and cannot be done afterwards using `dorado trim`."

applications:
  - Starting ONT DNA analysis from raw `POD5` signal and producing basecalled reads (raw or aligned).
  - Benchmarking Dorado CPU, memory, and batch-size settings on a few longest reads before scaling up.
  - Basecalling reads with or without reference alignment in a single Dorado workflow.
  - Assessing basecalled reads with `dorado summary` and deciding whether optional trimming or correction is justified.

terms:
  - long-read
  - term: ONT
    definition: Oxford Nanopore Technologies, a sequencing platform that produces long-read data.
  - term: POD5
    definition: Oxford Nanopore's binary format for storing raw electrical sequencing signal.
  - term: basecalling
    definition: Converting raw nanopore signal into nucleotide sequences using a computational model.
  - term: signal chunk
    definition: A fixed-size segment of raw nanopore signal that Dorado processes through the basecalling model.
  - term: batch size
    definition: The number of signal chunks Dorado processes together in one model call; it affects memory use and runtime, not basecalling quality.
  - term: CPU runner
    definition: A Dorado CPU model worker used for basecalling. On HPC systems, the runner count should be matched to the CPUs allocated to the job.
  - term: platform-native preprocessing
    definition: Processing specific to the sequencing platform, such as ONT basecalling and other raw-signal-aware steps performed before downstream read analysis.
  - term: post-basecalling reads QC/assessment
    definition: Evaluation of already basecalled reads using summary metrics, with optional cleanup steps such as trimming or correction when justified.
  - term: adapter/primer trimming
    definition: Removal of technical adapter or primer sequences from basecalled reads before downstream analysis.

materials:
  - "Example ONT DNA `POD5` datasets are available under `/reference/workbook/bioinformatics/dataset/reads_long/ont`."
  - "Use `/90daydata/shared/$USER/` as the practice workspace."

overview: [objectives, applications, terminology, materials]
---

{% include alert class="warning" content="This tutorial is for **ONT DNA workflows**. Do *not* use it for direct RNA input such as `RNA004`. **cDNA-based RNA** data can follow a similar Dorado workflow, but often require cDNA-aware adjustments, such as poly(A/T) estimation or primer handling." %}

## Overview

This tutorial uses `Dorado` for **long-read ONT DNA** workflows, but the correct entry point depends on the data you already have.
If you have raw-signal `POD5`, `Dorado` begins at platform-native preprocessing with `basecaller`.
If you already have exported reads in `SAM`, `BAM`, `CRAM`, or `FASTQ`, you can use `dorado demux` to do demultiplexing, and `dorado trim` can be used to trim adapters and primer sequences.

{% include overviews %}

## Getting Started

{% include segment/getting_started time="04:00:00" tasks="1" cpus_per_task="4" mem="32G" %}

### Tutorial Steps:

**PART 1: [Explore tools in `dorado` suite](#part-1-explore-tools-in-dorado-suite)**
1. [Prepare the practice workspace](#prepare-the-practice-workspace) on any SCINet cluster.
1. [Load Dorado module](#load-dorado-module) and list tools available in the suite. 

**PART 2: [Learn Dorado `basecaller` syntax and features](#part-2-learn-dorado-basecaller-syntax-and-features)**
1. [Get the minimal dataset](#get-the-dataset-demo) and learn about raw-signal `POD5` file format.
1. [Run Dorado basecaller simplex on the DNA `POD5` files](#run-basecaller-on-the-dna-pod5-files).
1. [Export reads in FASTQ, SAM or CRAM](#export-reads-in-fastq-sam-or-cram) file formats and [Learn integrated one-pass features](#integrated-one-pass-features-optional).

**PART 3: [Use Dorado on a real `POD5` dataset](#part-3-use-dorado-on-a-real-pod5-dataset)**
1. [Get the real ONT dataset](#get-the-dataset-monkeypox) for monkeypox virus.
1. [Estimate resources](#estimate-resources-needed) and [run pilot test(s)](#run-pilot-tests-in-the-interactive-session).
1. [Prepare SLURM submission](#prepare-slurm-submission) and choose a batch strategy.
1. [Run basecalling with or without alignment](#run-basecaller-with-or-without-alignment).

**PART 4: [Assess reads with dorado summary QC](#part-4-assess-reads-with-summary-qc)**
1. [Perform read-level QC](#run-dorado-summary) on the raw reads and aligned reads.
1. [Extract QC metrics](#how-to-extract-qc-metrics) from unaligned and aligned summaries.
1. [Decide whether `trim` or `correct` is justified](#decide-whether-trim-or-correct-is-justified).


## PART 1: Explore tools in `dorado` suite

<div class="process-list" markdown='1'>

### Prepare the practice workspace

{% include setup/practice_workspace workdir=page.tutorial.workdir %}

### Load Dorado module
{: .on-this-page__heading }


{% include tool_summary tool_key="dorado" %}

{% include segment/find_tool env_root="bioinformatics" %}

{% include setup/module tool_key="dorado" known="true" %}

{% include setup/tool_verify tool_key="dorado" skip="true" %}

<pre><small>Usage: dorado subcommand [options]
Subcommands: aligner  basecaller  correct  demux  download  duplex  polish  smallvar  summary  trim
</small></pre>

`Dorado` is a multi-tool package:

<div class="table-small" markdown="1">

| subcommand | input | output | role in workflow |
| --- | --- | --- | --- |
| `basecaller` | `POD5` | `BAM/SAM/CRAM/FASTQ` | basecalling from raw `POD5` signal to generate reads; supports one-pass demultiplexing, adapter/primer trimming and minimap2-based alignment |
| `duplex` | `POD5` | `BAM/FASTQ` | generating duplex basecalls from compatible read pairs |
| `summary` | `BAM` | `TSV` | producing read-level summary tables from basecalled output |
| `demux` | `BAM/FASTQ/SAM/CRAM` | demultiplexed `BAM/FASTQ` | demultiplexing reads by barcode |
| `trim` | `BAM/FASTQ/SAM/CRAM` | trimmed `BAM/FASTQ` | trimming adapter or primer sequence from basecalled reads |
| `correct` | `FASTQ/FASTQ.gz` | `FASTA` | correcting reads before downstream analysis |
| `aligner` | reads + ref. `FASTA` | aligned `BAM/SAM/CRAM` | aligning reads to a reference sequence |
| `polish` | aligned `BAM` + draft `FASTA` | polished `FASTA` | polishing a consensus sequence with aligned reads |
| `smallvar` | aligned `BAM` + ref. `FASTA` | `VCF/gVCF` | calling small variants from aligned reads |
| `download` | model name | model files | downloading Dorado models |

</div>

Explore options of a selected tool with:
```bash
dorado basecaller -h
```

</div>


## PART 2: Learn Dorado `basecaller` syntax and features

<div class="process-list" markdown='1'>

### Get the dataset (demo)

{% include setup/get_dataset
  dataset_path=page.tutorial.data_path
  dataset=page.tutorial.dataset
  ref_dataset_controls="dorado-demo-ref-dataset-acc"
  mode="reference"
  list="true" %}

<pre><small>16K dna_r10.4.1_e8.2_400bps_5khz-FLO_PRO114M-SQK_LSK114_XL-5000.pod5
16K dna_r10.4.1_e8.2_400bps_5khz-FLO_PRO114M-SQK_MLK114_96_XL-5000.pod5
16K dna_r10.4.1_e8.2_400bps_5khz-FLO_PRO114M-SQK_RAD114-5000.pod5
</small></pre>

<div class="highlighted highlighted--basic"><div class="highlighted__body" markdown="1">
`POD5` is Oxford Nanopore's binary raw-signal format. It stores the underlying electrical signal and run metadata needed for platform-native tools such as `dorado basecaller`.
</div></div>


### Run basecaller on the DNA `POD5` files

<div class="highlighted highlighted--basic"><div class="highlighted__body" markdown="1">
Dorado `basecaller` uses machine learning models to decode raw ONT sequencing signal from `POD5` input and emit the most likely nucleotide sequence as reads in a selected HTS format, such as `BAM`, `FASTQ`, `SAM`, or `CRAM`. It can also trim adapter and/or primer sequences during basecalling without disrupting demultiplexing, perform follow-up minimap2-based alignment, and estimate poly(A)/poly(T) tail lengths as a beta feature primarily meant for cDNA and dRNA use cases.
</div></div>

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="Run a tool interactively first to learn the syntax and validate paths" controls="dorado-dna-live-first" class="outline" icon=false %}
<div id="dorado-dna-live-first" class="accordion_content" markdown='1' hidden>

Start with **one ONT DNA `POD5` file** in the interactive session so you can learn tool syntax, confirm paths, and browse outputs.
For a first trial, keep the basecalling mode choice simple and let Dorado resolve the current compatible DNA model behind the `hac` (high-accuracy) option.

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| model | machine learning [basecalling model](https://software-docs.nanoporetech.com/dorado/1.1.1/models/models/) | `fast`: fastest, lowest accuracy <br>`hac`: high-accuracy, good default for a first run <br>`sup`: highest accuracy, typically slower and heavier |
| input file | data directory or `POD5` file path | `dna_r10.4.1_FLO_PRO114M-SQK_MLK114.pod5` |
| standard output | basecalled output is written as `BAM` | `> dna_calls.bam` |

</div>

Move into your practice workspace and run the command for selected sequencing-kit variant:

```bash
cd {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}
# set a variable for input directory
DATA_DIR={{ page.tutorial.data_path }}
```

<div class="process-list ul" markdown='1'>

### Run simplex basecalling for non-barcoded DNA kits
{: .no_toc }

Use this syntax for the [simplex basecalling](https://software-docs.nanoporetech.com/dorado/latest/basecaller/simplex/) on standard ligation DNA or rapid DNA.

```bash
# ligation DNA 
INPUT="${DATA_DIR}/{{ page.tutorial.pilot_sample }}"

dorado basecaller hac "${INPUT}" > dna_ligation_reads.bam
```

<details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>[info]  - downloading dna_r10.4.1_e8.2_400bps_hac@v6.0.0 with httplib
[info] > Creating basecall pipeline
[info] > Finished in (ms): 93757
[info] > Simplex reads basecalled: 1
[info] > Basecalled @ Samples/s: 2.173704e+01
[info] > Finished

real    2m29.889s
user    1m30.641s
sys     0m11.010s
</small></pre>
</details>


### Run simplex basecalling for barcoded kits
{: .no_toc }

The [barcoded kits](https://software-docs.nanoporetech.com/dorado/2.1.0/barcoding/barcoding/) such as native barcoding, rapid barcoding, 16S barcoding, microbial amplicon barcoding, 
or multiplexed ligation DNA require barcode-aware handling for the matching kit. 

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| `--kit-name` | enable barcode classification for the matching multiplex kit during basecalling | `--kit-name SQK-MLK114-96-XL` |
| `--no-trim` | preserve barcode, adapter, and primer sequence for later separate demultiplexing | `--no-trim` |

</div>

By default, Dorado trims detected barcode, adapter, and primer sequence during basecalling.

```bash
# multiplexed ligation DNA 
INPUT="${DATA_DIR}/{{ page.tutorial.sample }}"

dorado basecaller hac "${INPUT}" --kit-name SQK-MLK114-96-XL > dna_multiplex_trimmed.bam
```

<details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>[info]  - downloading dna_r10.4.1_e8.2_400bps_hac@v6.0.0 with httplib
[info] > Creating basecall pipeline
[info] > Finished in (ms): 94049
[info] > Simplex reads basecalled: 1
[info] > Basecalled @ Samples/s: 2.166956e+01
[info] > 1 reads demuxed @ classifications/s: 1.063276e-02
[info] > Finished

real    3m15.310s
user    1m31.566s
sys     0m12.407s
</small></pre>
</details>

If you want to preserve barcode sequence for a later steps, add `--no-trim` flag.

```bash
dorado basecaller hac "${INPUT}" --kit-name SQK-MLK114-96-XL --no-trim > dna_multiplex_notrim.bam
```

<details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>[info]  - downloading dna_r10.4.1_e8.2_400bps_hac@v6.0.0 with httplib
[info] Failed to load NVML
[info] > Creating basecall pipeline
[info] > Finished in (ms): 93542
[info] > Simplex reads basecalled: 1
[info] > Basecalled @ Samples/s: 2.178700e+01
[info] > 1 reads demuxed @ classifications/s: 1.069039e-02
[info] > Finished

real    3m18.474s
user    1m31.415s
sys     0m13.974s
</small></pre>
</details>

What can change beyond just the kit name:
- `--barcode-both-ends` - Useful for double-ended barcode kits when you want stricter barcode assignment and fewer false positives.
- `--primer-sequences` - Needed only when the [primer set is custom](https://software-docs.nanoporetech.com/dorado/latest/barcoding/custom_primers), third-party, older, or not the default one Dorado would infer for that kit.
- `--barcode-arrangement` and `--barcode-sequences` - Needed only for custom barcode layouts or custom barcode sets, not for standard ONT kits.

</div>

### Validate the output before scaling up

First confirm that the basecalled `BAM` files were created in your workspace:

```bash
ls -lh *.bam
```

<pre><small> 783 dna_ligation_reads.bam
3.1K dna_multiplex_trimmed.bam
3.1K dna_multiplex_notrim.bam
</small></pre>

The reported file sizes should confirm that the generated `BAM` files are non-empty.

Use observed runtimes as a rough guide when choosing `#SBATCH --time`, CPU count, and memory for the batch job.

</div>


{% include accordion title="Scale up with SLURM to automate the task for all inputs" controls="dorado-dna-slurm-all" class="outline" icon=false %}
<div id="dorado-dna-slurm-all" class="accordion_content" markdown='1' hidden>

{% include segment/cli_to_slurm tool="Dorado" %}

1. In your practice workspace, create a SLURM script file and open it for editing:
```bash
cd {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}
touch dorado_dna_batch.sh
nano dorado_dna_batch.sh
```

1. Copy and paste the following script body. Before saving it, inspect the path settings carefully. Use `pwd` in your current workspace if you want to confirm the absolute working-directory path.
```bash
    #!/bin/bash
    #SBATCH --job-name=dorado_dna
    #SBATCH --partition=<value>        # EDIT partition; Ceres: ceres, scavenger; Atlas: atlas
    #SBATCH --nodes=1
    #SBATCH --ntasks=1
    #SBATCH --cpus-per-task=4
    #SBATCH --account=<account>        # EDIT ACCOUNT, provide your SCINet project account
    #SBATCH --mem=32G
    #SBATCH --time=08:00:00
    #SBATCH --output=dorado_dna_%j.out

    module load dorado
    dorado -v

    INPUTS={{ page.tutorial.data_path }}/{{ page.tutorial.pilot_sample }}
    WORKDIR={{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}

    dorado basecaller hac "${INPUTS}" > "${WORKDIR}/dna_ligation_reads.bam"
```

    - <small>`INPUTS` points to one standard ligation DNA `POD5` file. If multiple `POD5` files require the same processing, their parent directory can be specified as `INPUTS` instead.</small>
    - <small>Dorado auto-detects internal CPU workers/runners from the visible hardware. Use `--cpus-per-task=4`, `--mem=32G`, and `--time=08:00:00` as conservative starter values; increase CPUs up to `48` per node on Atlas, and up to `96` or `256` logical cores per node on current Ceres node types, and increase memory if needed.</small>

1. Submit it with:
```bash
sbatch dorado_dna_batch.sh
```

1. Then check whether the job is queued or running and inspect the batch log file for any immediate errors:
```bash
squeue -u $USER
ls -lh dorado_dna_*.out
```

1. Optionally, check later whether the expected basecalled `BAM` file is being produced in your practice workspace.
```bash
ls -lh *.bam
```

</div>
</div>


### Export reads in FASTQ, SAM or CRAM

By default, Dorado `basecaller` produces unaligned ONT DNA reads in `BAM` format and the output file is written to `stdout`. 
If you need reads in a different output format, use one of the Dorado output arguments and adjust the file extension.

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| default output | basecalled reads in unaligned `BAM` format | `> dna_ligation_reads.bam` |
| `--emit-fastq` | emit basecalled reads in `FASTQ` format | `--emit-fastq > dna_ligation_reads.fastq` |
| `--emit-sam` | emit basecalled reads in `SAM` format | `--emit-sam > dna_ligation_reads.sam` |
| `--emit-cram` | emit basecalled reads in `CRAM` format | `--emit-cram > dna_ligation_reads.cram` |

</div>

Examples:

```bash
dorado basecaller hac "${INPUT}" --emit-fastq > dna_ligation_reads.fastq

dorado basecaller hac "${INPUT}" --emit-sam > dna_ligation_reads.sam

dorado basecaller hac "${INPUT}" --emit-cram > dna_ligation_reads.cram
```

### <span class="text-base">Integrated one-pass features *(optional)*</span>

{% include alert class="warning" noicon="true" content="The examples below are syntax guides only, because  
- this tutorial does not provide a genome reference for POD5 datasets, 
- and poly(A/T) estimation is primarily meant for cDNA and dRNA workflows." 
%}

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="Align reads during basecalling" controls="dorado-dna-integrated-alignment" class="outline" icon=false %}
<div id="dorado-dna-integrated-alignment" class="accordion_content" markdown='1' hidden>

Add a reference with `--reference` if you want Dorado to basecall and align reads in one pass. Alignment is performed with minimap2, and the output format can be selected as `BAM`, `SAM`, or `CRAM` using the [corresponding export options](#export-reads-in-fastq-sam-or-cram).

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| `--reference` | reference sequence used for minimap2-based alignment during basecalling | `--reference genome.fa` |
| `--mm2-opts` | additional minimap2 alignment options | `--mm2-opts "-k 15 -w 10"` |
| `--bed-file` | BED file of regions of interest; Dorado counts overlaps with each alignment and writes the count to the `bh` tag | `--bed-file targets.bed` |
| `--emit-summary` | emit a summary file with details of the primary alignments for each read in the current working directory | `--emit-summary` |

</div>

```bash
dorado basecaller hac "${INPUT}" --reference genome.fa --emit-summary > dna_ligation_aligned.bam
```

</div>

{% include accordion title="Estimate poly(A) or poly(T) during basecalling" controls="dorado-dna-integrated-polya" class="outline" icon=false %}
<div id="dorado-dna-integrated-polya" class="accordion_content" markdown='1' hidden>

Poly(A)/poly(T) tail-length estimation is a beta feature for cDNA and dRNA workflows. The estimate is written to output tags and does not change the basecalled sequence.

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| `--estimate-poly-a` | enable poly(A)/poly(T) tail-length estimation during basecalling | `--estimate-poly-a` |
| `--poly-a-config` | configuration file for non-default poly(A)/poly(T) estimation behavior | `--poly-a-config polya_config.toml` |

</div>

```bash
dorado basecaller hac "${INPUT}" --estimate-poly-a > cdna_reads_with_polya.bam
```

</div>

{% include accordion title="Call modified bases during basecalling" controls="dorado-dna-integrated-modbases" class="outline" icon=false %}
<div id="dorado-dna-integrated-modbases" class="accordion_content" markdown='1' hidden>

Modified-base calling matters when the sequencing signal is used to infer methylation or other base modifications, not just the canonical sequence. In practice, this is mainly relevant for DNA methylation-aware workflows and downstream tools that read modification tags such as `MM` and `ML`.

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| `--modified-bases` | space-separated list of [modification codes](https://software-docs.nanoporetech.com/dorado/latest/basecaller/mods/); Dorado resolves compatible models automatically | `--modified-bases 5mC 6mA` |
| `--modified-bases-models` | comma-separated modified-base [model names or paths](https://software-docs.nanoporetech.com/dorado/latest/models/list/) | `--modified-bases-models 5mC_model` |
| `--modified-bases-threshold` | minimum predicted modification probability to emit in output tags | `--modified-bases-threshold 0.45` |

</div>

```bash
dorado basecaller hac "${INPUT}" --modified-bases 5mC 6mA > dna_modbase_reads.bam
```
 - <small>Place `--modified-bases ...` after the positional arguments such as `hac` and `"${INPUT}"`.</small>
 - <small>Only one modification model per canonical base can be active at once, e.g., `5mC 6mA` is a valid combination because the two codes target different canonical bases.</small>

</div>
</div>

</div>



## PART 3: Use Dorado on a real `POD5` dataset

This part of the tutorial demonstrates the challenges associated with real-life datasets due to their size, experimental design, and availability of a genomic reference.
The goal is to apply Dorado basecalling on raw POD5 at a realistic scale and then perform the quality check (QC) on the generated reads.

Your work can be continued in the same workspace created in [Prepare the practice workspace](#prepare-the-practice-workspace) step.  
Navigate to your workspace in `/90daydata` and create a working directory for a new dataset:
```bash
cd {{ page.tutorial2.root_practice }}/{{ page.tutorial2.workdir }}
mkdir -p {{ page.tutorial2.dataset }}
cd {{ page.tutorial2.dataset }}
```

<div class="process-list" markdown='1'>

### Get the dataset (monkeypox)

{% include setup/get_dataset
  dataset_path=page.tutorial2.data_path
  dataset=page.tutorial2.dataset
  ref_dataset_controls="dorado-real-ref-dataset-acc"
  mode="reference"
  list="true" %}

<pre><small>2.2M 10244_MPOX-65.pod5
299K 10244_MPOX-66.pod5
 40M 10244_MPOX-67.pod5
</small></pre>

### Estimate resources needed

See how the input files for this dataset are larger than the small trial file `SQK_LSK114_XL-5000.pod5` used in Part 2. The largest file, `{{ page.tutorial2.scale_sample }}` (40M), is approximately 2,500 times larger than the `SQK_LSK114_XL` ligation DNA file (16K). Datasets for other species can easily span gigabytes per `POD5` file and hundreds of files in a dataset (see example human dataset [PAW79146](https://42basepairs.com/browse/s3/ont-open-data/giab_2025.01/flowcells/HG001/PAW79146/pod5)).

<div class="highlighted highlighted--highlighted"><div class="highlighted__body" markdown="1">
Because datasets vary widely, no fixed combination of CPU/GPU, memory, and runtime applies to every dataset; estimate requirements by testing most demanding reads in a trial interactive run on a computing node.
</div></div>

<div class="process-list ul" markdown='1'>

### Check the total size of the dataset 
{: .no_toc }

Assessing the size and structure of your dataset before analysis helps request sufficient resources. 

Check the total size of the input files:
```bash
du -sh {{ page.tutorial2.data_path }}
```
   <pre><small>42M</small></pre>


### Count the number and length of reads 
{: .no_toc }

File size provides a rough indication of the resources needed. Computing nodes (CPU/GPU), memory and runtime use also depend on the **number and length of reads** and the basecalling model. 

<div class="highlighted highlighted--basic"><div class="highlighted__body" markdown="1">
ONT provides a Python-based [pod5](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#tools) tool package for inspecting ([view](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-view)) and manipulating ([merge](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-merge), [filter](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-filter), [subset](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-subset)) `POD5` files before basecalling, and additional tools for converting [from](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-convert-fast5) and [into](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-convert-to_fast5) `FAST5` file format, [update](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-update) old `POD5` files to use the latest schema, or to [recover](https://github.com/nanoporetech/pod5-file-format/blob/master/python/pod5/README.md#pod5-recover) data from corrupted or truncated `POD5` files.
</div></div>

{% include setup/venv known_venv="true" activate="true" tool_key="pod5" skip_note="true" %}

{% include setup/tool_verify tool_key="pod5" skip="true" %}

Perform simple statistics on each `POD5` file:

```bash
DATASET={{ page.tutorial2.data_path }}

for file in "${DATASET}"/*.pod5; do
  output="$(basename "${file%.pod5}")"
  reads=$(pod5 view "${file}" --include "read_id,num_samples" | tail -n +2)
  read_count=$(printf '%s\n' "${reads}" | wc -l)
  printf '%s\t%s\n' "${output}" "${read_count}" >> read_counts

  printf '%s\n' "${reads}" | sort -t $'\t' -k2,2n | tail -n 3 | awk -v prefix="${output}" '{ print prefix "\t" $0 }' >> "3_longest.tsv"
done

sample=$(sort -t $'\t' -k3,3n 3_longest.tsv | tail -1 | cut -f1)
awk -F"\t" -v sample="${sample}" '$1 == sample { print $2 }' 3_longest.tsv > "${sample}_ids"
```

<details class="padding-x-2 bg-success-lighter"><summary><i>preview output content</i></summary>

<div class="grid-row grid-gap">
<div class="tablet:grid-col-3" markdown="1">
<pre><small><i>read_counts</i>
10244_MPOX-65 474 
10244_MPOX-66 70 
10244_MPOX-67 8763 
</small></pre>
</div>

<div class="tablet:grid-col-5" markdown="1">
<pre><small><i>3_longest.tsv</i>
10244_MPOX-65   a9e50149-f623-4b06-9ffc-1760b915c21a    11865
10244_MPOX-65   ac568467-4a7e-4707-8ec0-b22c211c8ed6    12743
10244_MPOX-65   14c57abc-47d1-4629-a890-c58e0f164c31    13138
10244_MPOX-66   66101319-c1d0-42b8-ba03-64539f0fe517    7067
10244_MPOX-66   c133648f-abf3-4d79-acae-1387bc4f5eeb    7943
10244_MPOX-66   ed7e46b4-43c0-45fe-a510-ef627b0f535a    11143
10244_MPOX-67   897b8c54-6c91-46f3-862c-6615269d5194    20800
10244_MPOX-67   af6c7662-365b-4b6d-8dae-53e33bbc9baa    24182
10244_MPOX-67   4e91ef2e-8409-4b67-8ebd-7c5473a2f588    28916
</small></pre>
</div>

<div class="tablet:grid-col-4" markdown="1">
<pre><small><i>10244_MPOX-67_ids</i> 
897b8c54-6c91-46f3-862c-6615269d5194
af6c7662-365b-4b6d-8dae-53e33bbc9baa
4e91ef2e-8409-4b67-8ebd-7c5473a2f588
</small></pre>
</div>
</div>
</details>

- `read_counts`: one line per `POD5` file with its number of reads; read count helps estimate total per-file basecalling workload and runtime.
- `3_longest.tsv`: the three longest raw-signal records from each `POD5` file, including `filename`, `read ID`, and `num_samples`; these indicate most-demanding memory requirements.
- `{{ page.tutorial2.scale_read_ids }}`: IDs from a single file selected for trial; pass it to `dorado basecaller` with `-l, --read-ids` in the interactive run to test and optimize the required resources.

### Run pilot test(s) in the interactive session

Using a few largest reads in the trial interactive run can provide solid estimates for choosing resources before scaling up into a SLURM batch job. Always run your `dorado basecaller` pilot test with `-v` flag to include debug messages in the command log. Also, use `memory` and `time` prefixes to report the peak memory usage and runtime. 

{% include setup/export_path path="/reference/workbook/bioinformatics/utils" command="memory" shell_file="~/.bashrc" title="Make the <i>memory</i> helper command available" controls="dorado-memory-helper-path" class="note" %}

```bash
mkdir test_run
cd test_run
echo $DATASET     # confirm dataset path
ls $DATASET       # confirm that input files exist

INPUT=${DATASET}/{{ page.tutorial2.scale_sample }}
time memory dorado basecaller hac "${INPUT}" -v -l ../{{ page.tutorial2.scale_read_ids }} > test_1.bam
```

<details class="padding-x-2 bg-success-lighter"><summary><i>command log (run @ceres)</i></summary>

<pre><small>[info]  - downloading dna_r10.4.1_e8.2_400bps_hac@v6.0.0 with httplib
[info] Failed to load NVML
[debug] - CPU calling: set num_cpu_runners to 96
[info] > Creating basecall pipeline
[debug] BasecallerNode chunk size 12288
[debug] Load reads from file ...10244_MPOX-67.pod5
[info] > Finished in (ms): 177993
[info] > Simplex reads basecalled: 3
[info] > Basecalled @ Samples/s: 4.150051e+02
[debug] > Including Padding @ Samples/s: 3.239e+04 (1.57%)
[info] > Finished
[debug] Deleting temporary model path '/90daydata/ ... /.temp_dorado_model-23cbf9fed990ea0'

# memory
Peak system RAM: 27.74 GB (27088248 MiB)
GPU memory: nvidia-smi unavailable; not measured
# time
real    2m43.190s
user    5m36.183s
sys     0m41.758s
</small></pre>
</details>

#### Runtime and Memory 

[This run](#run-pilot-tests-in-the-interactive-session) completed successfully in ~3 minutes and used ~28GB RAM at peak on Ceres (using CPU only) within an interactive session set with `-n 1 --cpus-per-task=4 --mem=32G`. When tracking the progress of `[debug] Load reads...` you could notice that all reads are preloaded at once; that means just the 3 reads almost reached the available `mem=32G` in the session.

{% capture exercise_1 %}
Repeat the [Count the number and length of reads](#count-the-number-and-length-of-reads) step, this time selecting the five longest reads into `5_longest.tsv` and writing their read IDs to `{{ page.tutorial2.scale_read_ids_5 }}`. Then rerun the Dorado basecaller command for the 5 IDs and check whether the run completes successfully. <br>**TIP:** *Compare the runtime and peak memory with the three-read pilot run.*

<details markdown="1"><summary>SOLUTION</summary>

```bash
time memory dorado basecaller hac "${INPUT}" -v -l ../{{ page.tutorial2.scale_read_ids_5 }} > test_2.bam
```
<pre class="padding-x-2 bg-error-lighter"><small>[info] - downloading dna_r10.4.1_e8.2_400bps_hac@v6.0.0 with httplib
[info] > Creating basecall pipeline
[debug] - CPU calling: set num_cpu_runners to 96
[debug] BasecallerNode chunk size 12288
[debug] Load reads from file
Killed

Peak system RAM: 33.39 GB (32608120 MiB)
GPU memory: nvidia-smi unavailable; not measured

real    1m15.471s
user    1m17.822s
sys     0m32.696s
</small></pre>
Peak RAM exceeded the available in-session 32 GB and the task was killed. 
</details>
{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_1 %}

Dorado may detect the full compute-node CPU *(here: 96)* count rather than the CPUs assigned to the job *(here: 4)*, and use that to auto-configure CPU runners, which can inflate memory use and result in the job being killed. To prevent Dorado from over-allocating CPU runners and memory, set its dedicated `DORADO_CPU_RUNNERS` environment variable to match the exact number of CPUs requested with `--cpus-per-task`:
```bash
# set in the interactive session and/or your SLURM script
export DORADO_CPU_RUNNERS=4
```


{% capture exercise_2 %}
Repeat the exercise.  <br>*Did your run complete successfully this time?*

<details markdown="1"><summary>SOLUTION</summary>

```bash
export DORADO_CPU_RUNNERS=4
time memory dorado basecaller hac "${INPUT}" -v -l ../{{ page.tutorial2.scale_read_ids_5 }} > test_3.bam
```
<pre class="padding-x-2 bg-success-lighter"><small>[info] - downloading dna_r10.4.1_e8.2_400bps_hac@v6.0.0 with httplib
[info] > Creating basecall pipeline
[info] Overriding CPU runners to 4
[debug] - CPU calling: set num_cpu_runners to 4
[debug] BasecallerNode chunk size 12288
[debug] Load reads from file
[info] > Finished in (ms): 194449
[info] > Simplex reads basecalled: 5
[info] > Basecalled @ Samples/s: 5.804041e+02
[debug] > Including Padding @ Samples/s: 3.236e+04 (1.79%)
[info] > Finished
[debug] Deleting temporary model path: '/90daydata/shared/.../test_run/.temp_dorado_model-2ff27f96e63f9477'

Peak system RAM: 32.93 GB (32608120 MiB)
GPU memory: nvidia-smi unavailable; not measured

real    3m26.810s
user    9m7.506s
sys     3m42.271s
</small></pre>
The log from the repeated run confirmed that Dorado limited the CPU runners as specified, allowing the job to complete successfully.
</details>
{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_2 %}


#### Model download/persist

The `hac` flag enabled automatic model selection, and the command log specified it as `dna_r10.4.1_e8.2_400bps_hac@v6.0.0`. However, the downloaded copy got delated at process exit `[debug] Deleting temporary model path`. With this setting the download-delate cycle will repeat with every trial run and batch job. It is useful to run it once to discover the model needed. Then, you can find it on the shared `/reference` path and use directly instead of `{fast,hac,sup}` flag in the `dorado basecaller`.

```bash
ls /reference/workbook/bioinformatics/models/ont_models
ls /reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0 

#MODEL=/reference/workbook/bioinformatics/models/ont_models/<selected_model>
MODEL=/reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0  
# Use as: dorado basecaller ${MODEL} ${INPUT} -v > output.bam
```

If missing on the shared path, download a model manually:
```bash
dorado download -v --model dna_r10.4.1_e8.2_400bps_hac@v6.0.0 --models-directory {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}
ls {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}
```

<details markdown="1"><summary>Download all models</summary>

If you work with diverse ONT chemistry datasets, it might be useful to `--list` available models or download all. 
```bash
dorado download --list

mkdir -p {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}/ont_models
dorado download -v --model all --models-directory {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}/ont_models
ls {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}/ont_models
```
*Use `dorado download -h` for more options.* 
</details>


#### Signal chunks

Dorado {{ page.tools.dorado.version }} splits raw signal into fixed-size chunks (`12,288` samples) with `600` samples of overlap before basecalling. This means each new chunk advances by `12,288 - 600 = 11,688` samples. 
For a read with `S` raw signal samples, chunk size `C`, and overlap `O`:
```bash
N_chunks = ceil((S - C) / (C - O)) + 1
```

<pre><small>N_chunks = (28,916 - 12,288) / (12,288 - 600) + 1 = (16,628 / 11,688) + 1 = 2.42  ≈  3</small></pre>

For the longest read in this dataset (*4e91ef2e-8409-4b67-8ebd-7c5473a2f588* from 10244_MPOX-67), with `28,916` raw signal samples, Dorado therefore requires 3 chunks:  
<pre><small>Chunk 1:     0 ───────────── 12,288
Chunk 2:          11,688 ───────────── 23,976
Chunk 3:                    23,376 ───────────── 35,664
</small></pre>
So, we should expect no more than 3 chunks per read in a batch, and for the largest POD5 [file containing 8,763 reads](#count-the-number-and-length-of-reads), we should assume about **26k** as a conservative upper bound on the total number of signal chunks.  
<pre><small>8763 reads x 3 = 26,289 signal chunks (in the largest POD5 file)</small></pre>


#### Batch size

The `-b`/`--batchsize` parameter controls how many signal chunks Dorado processes together in one model batch. **Batch size affects memory use and runtime**, but not the amount of signal context used for each chunk or the expected basecalling quality. 
- This is a throughput parameter: it controls how many already-defined signal chunks Dorado sends through the neural network at once, not how much signal context the model sees. 
- A smaller batch size does not make basecalling simpler or lower quality; Dorado processes the same chunks with the same model, just in smaller groups across more model calls.
- With the default setting (`-b 0`), Dorado selects the optimal batch size automatically, but *does not report the selected value* in the log, so any manual tuning must be done empirically by testing explicit `-b` values. 

If a pilot run fails due to insufficient memory, **either request more memory or reduce the batch size manually.** A practical approach is to start with -b 128 as a reference *(Dorado’s CPU memory heuristic is calibrated at this batch size)* and test smaller values until peak memory and runtime are satisfactory for the target HPC allocation. 

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="Batch size benchmark using CPU-only" controls="batch-size-benchmark" class="info" icon=true %}
<div id="batch-size-benchmark" class="accordion_content" markdown='1' hidden>

This benchmark used the five longest reads, totaling `112,909` raw signal samples, in a CPU-only SLURM session with 4 CPUs and 32 GB RAM.

| dorado CPU runners | Batch size (`-b`) | Peak RAM | Runtime | Result |
|---|---:|---:|---:|---|
| (96 assumed) | auto | 33.39 GB | —        | <span class="bg-error-lighter"> Killed </span>  |
|<small>`DORADO_CPU_RUNNERS=4`</small> | auto | 32.93 GB | 3m 26.8s | Success |
| 4 | 128  | 32.40 GB | 3m 32.8s | Success |
| 4 | 96   | 32.39 GB | 3m 21.2s | Success |
| 4 | 72   | 25.05 GB | 2m 48.2s | Success |
| 4 | 64   | 26.72 GB | 2m 49.7s | Success |
| 4 | 30   | 13.12 GB | 1m 32.3s | Success |
| 4 | 15   |  8.74 GB | 1m 18.9s | Success |
| 4 | 9    |  5.93 GB | 1m 08.5s | Success |

Peak memory increased substantially with larger batch sizes. 
For this CPU setup, **smaller batches were faster**: processing fewer chunks per model call reduced the per-call memory and computational workload enough to outweigh the cost of making more model calls.
</div>
</div>

{% include alert class="tip" content="Don’t assume a small batch size will make Dorado slower - on CPU, smaller batches can use much less memory and may even complete faster than larger or automatic batch sizes.  
*For example, 1,000 chunks with `-b 10` require about 100 model calls, while `-b 100` requires about 10. But if each b=100 call takes more than `10×` as long as a b=10 call on that CPU, the smaller batch is still faster.*" %}


### Prepare SLURM submission

#### Walltime *(for batch job)*

Because the [interactive trial used five longest reads](#runtime-and-memory), the full-POD5 walltime can be safely estimated by scaling the pilot runtime (`3m27s`) by the number of reads (`8763`):
```bash
walltime ≈ pilot_runtime × (reads_in_POD5 / benchmarked_reads)
```

<pre><small>walltime = 210 seconds × (8763 / 5) = 6134 minutes  ≈ 102h </small></pre>

*This estimate will generally be higher than the actual runtime because most reads are shorter than those in the benchmark, providing a safety margin against termination due to insufficient walltime.*

If this estimated walltime is too long for your project, start a new interactive session requesting more CPUs and memory, and repeat the pilot benchmark. 

{% capture exercise_3 %}
Repeat the run with five longest reads (IDs saved to `{{ page.tutorial2.scale_read_ids_5 }}`) using a larger allocation in the interactive session, for example by doubling both CPUs and memory.
Test with automatic batch size to see whether additional resources improve runtime.

**TIP:** *Match `DORADO_CPU_RUNNERS` to the new `--cpus-per-task` value. Start with the default batch size, then test `-b 32`, followed by 16 and 8 if runtime continues to decrease. Use pre-downloaded model.*

<details markdown="1"><summary>SOLUTION</summary>

Close current interactive session with `exit` and request new resources: 
```bash
srun -A <account> -t 04:00:00 -n 1 --cpus-per-task=8 --mem=64G --pty bash 
```
Load dorado module and set shell variables again:
```bash
module load dorado   # load dorado module 
export DORADO_CPU_RUNNERS=8   # match requested cores 

DATASET={{ page.tutorial2.data_path }}   
INPUT=${DATASET}/{{ page.tutorial2.scale_sample }}
MODEL=/reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0

time memory dorado basecaller ${MODEL} ${INPUT} -v -l ../{{ page.tutorial2.scale_read_ids_5 }} > test_4.bam
```
<pre class="padding-x-2 bg-success-lighter"><small>[info] Overriding CPU runners to 8
[debug] - CPU calling: set num_cpu_runners to 8
[info] > Creating basecall pipeline
[debug] BasecallerNode chunk size 12288
[debug] Load reads from file
[info] > Finished in (ms): 198453
[info] > Simplex reads basecalled: 5
[info] > Basecalled @ Samples/s: 8.747810e+02
[debug] > Including Padding @ Samples/s: 6.096e+04 (1.44%)
[info] > Finished

Peak system RAM: 39.16 GB

real    2m15.886s
user    8m10.628s
sys     1m2.697s
</small></pre>
The larger allocation improved performance substantially, this run completed successfully in about 2 minutes with 39 GB peak RAM, leaving comfortable headroom within the 64 GB limit. Since Dorado does not report the selected batch size, test an explicit `-b 64` next, then decrease to 32 and 16 if runtime continues to improve.

|   | auto | b=64 | b=32 | b=16 | b=8 |
|---|------|------|------|------|-----|
| RAM  | 39.16 GB | 15.97 GB | 9.96 GB  | **3.64 GB**  | 3.71 GB  |
| real | 2m15.88s | 0m54.39s | 0m39.39s | **0m18.13s** | 0m22.21s |

<pre><small>walltime = 20 seconds × (8763 / 5) = 584 minutes  ≈ 10h</small></pre>  
The estimated full-file walltime decreased about 10x (from ~102h to ~10h), with 8 requested CPUs and batch size `-b 16`, while using only ~4 GB RAM.

</details>
{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_3 %}

Once the [CPU/memory allocation](#runtime-and-memory) and [batch size](#batch-size) are optimized and the estimated per-`POD5`-file [walltime is reasonable](#walltime-for-batch-job), decide how to organize the batch processing:

| [Single job](#single-job) | [One job per POD5](#one-job-per-pod5) | [Job array](#job-array) |
|---|---|---|
|process all POD5 files from their shared parent directory; simplest, but requires increasing walltime for the entire dataset | estimate walltime from the number of reads in each file and submit each independently | process one POD5 file per array task; usually the most scalable and convenient option for many files |

#### Single job

The walltime must cover the entire dataset. [Count the number of reads](#count-the-number-and-length-of-reads) in all POD5 inputs. 

<pre><small>8763 + 70 + 474 = 9307 reads
walltime = 20 seconds × (9307 / 5) = 620 minutes  ≈ 11h
</small></pre>

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="SLURM script: dorado_batch.sh" controls="dorado-batch-script" class="note" icon=true %}
<div id="dorado-batch-script" class="accordion_content" markdown='1' hidden>

- use the shared parent folder as input; Dorado accepts a directory as input
- add `--recursive` if POD5 files are nested below it

```bash
#!/bin/bash
#SBATCH --job-name=dorado         # EDIT
#SBATCH --partition=<value>       # EDIT: on Ceres: ceres; on Atlas: atlas
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8         # EDIT: adjust with benchmark; up to 72-96 on Ceres; up to 48 on Atlas 
#SBATCH --account=<account>       # EDIT: specify project account
#SBATCH --mem=16G                 # EDIT: adjust with benchmark; no less than 16-32GB
#SBATCH --time=12:00:00           # EDIT: adjust with estimation for entire dataset; add +10% to estimation
#SBATCH --output=dorado_dna_%j.out

module load dorado
dorado -v

export DORADO_CPU_RUNNERS=${SLURM_CPUS_PER_TASK}

# EDIT values to match your project
BS=16     # benchmarked batchsize
MODEL=/reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0
INPUTS=/path/to/pod5_parent_folder    # e.g., /reference/workbook/bioinformatics/dataset/reads_long/ont/monkeypox_SQB000004/00_raw_data
WORKDIR=/90daydata/shared/$USER/tutorials/long_reads_qc/dorado_dna

# ---------- NO EDITS BELOW ----------
mkdir -p "${WORKDIR}"

dorado basecaller "${MODEL}" "${INPUTS}" --recursive -b "${BS}" > "${WORKDIR}/reads.bam"
```
</div>
</div>

Submit to the SLURM queue with:
```bash
sbatch dorado_batch.sh
```


#### One job per POD5

Use this when individual POD5 files differ substantially in size and you want to assign walltime separately. [Count the number of reads](#count-the-number-and-length-of-reads) in each POD5 input. 

<pre><small>walltime #1 = 20 seconds × (8763 / 5) = 584 minutes  ≈ 10h
walltime #2 = 20 seconds × (70 / 5) = 5 minutes  ≈ 1h
walltime #3 = 20 seconds × (474 / 5) = 32 minutes  ≈ 1h
</small></pre>

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="SLURM script: dorado_one_file.sh" controls="dorado-one-file-script" class="note" icon=true %}
<div id="dorado-one-file-script" class="accordion_content" markdown='1' hidden>

```bash
#!/bin/bash
#SBATCH --job-name=dorado         # EDIT
#SBATCH --partition=<value>       # EDIT: on Ceres: ceres; on Atlas: atlas
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8         # EDIT: adjust with benchmark; up to 72-96 on Ceres; up to 48 on Atlas 
#SBATCH --account=<account>       # EDIT: specify project account
#SBATCH --mem=16G                 # EDIT: adjust with benchmark; no less than 16-32GB
#SBATCH --time=12:00:00           # EDIT: adjust with estimation for a given file; add +10% to estimation
#SBATCH --output=dorado_dna_%j.out

module load dorado
dorado -v

export DORADO_CPU_RUNNERS=${SLURM_CPUS_PER_TASK}

# EDIT values to match your project
BS=16     # benchmarked batchsize
MODEL=/reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0
INPUTS=${INPUTS}     # or /path/to/sample.pod5 e.g., /reference/workbook/bioinformatics/dataset/reads_long/ont/monkeypox_SQB000004/00_raw_data/{{ page.tutorial2.scale_sample }}
WORKDIR=/90daydata/shared/$USER/tutorials/long_reads_qc/dorado_dna

# ---------- NO EDITS BELOW ----------
mkdir -p "${WORKDIR}"

NAME=$(basename "${INPUTS}" .pod5)
dorado basecaller "${MODEL}" "${INPUTS}" -b "${BS}" > "${WORKDIR}/${NAME}.bam"
```
</div>
</div>

Submit to the SLURM queue with:
```bash
# with a specific path assigned to INPUTS
sbatch dorado_one_file.sh   
# with a specific path assigned on the fly
sbatch --export=INPUTS=/path/to/file1.pod5 dorado_one_file.sh
sbatch --export=INPUTS=/path/to/file2.pod5 dorado_one_file.sh
sbatch --export=INPUTS=/path/to/file3.pod5 dorado_one_file.sh
```


#### Job array 

Each array task processes one POD5 independently, allowing files to run in parallel.

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="SLURM script: dorado_array.sh" controls="dorado-array-script" class="note" icon=true %}
<div id="dorado-array-script" class="accordion_content" markdown='1' hidden>

```bash
#!/bin/bash
#SBATCH --job-name=dorado         # EDIT
#SBATCH --partition=<value>       # EDIT: on Ceres: ceres; on Atlas: atlas
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8         # EDIT: adjust with benchmark; up to 72-96 on Ceres; up to 48 on Atlas 
#SBATCH --account=<account>       # EDIT: specify project account
#SBATCH --mem=16G                 # EDIT: adjust with benchmark; no less than 16-32GB
#SBATCH --time=12:00:00           # EDIT: adjust with the highest estimation across files; add +10% to estimation
#SBATCH --array=0-<N-1>           # EDIT: adjust to N-1 input files, e.g., use --array=0-2 for 3 POD5 input files 
#SBATCH --output=dorado_dna_%A_%a.out

module load dorado
dorado -v

export DORADO_CPU_RUNNERS=${SLURM_CPUS_PER_TASK}

# EDIT values to match your project
BS=16     # benchmarked batchsize
MODEL=/reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0
INPUT_DIR=/path/to/pod5_files     # e.g., /reference/workbook/bioinformatics/dataset/reads_long/ont/monkeypox_SQB000004/00_raw_data
WORKDIR=/90daydata/shared/$USER/tutorials/long_reads_qc/dorado_dna

# ---------- NO EDITS BELOW ----------
mkdir -p "${WORKDIR}"

mapfile -t POD5_FILES < <(find "${INPUT_DIR}" -maxdepth 1 -type f -name '*.pod5' | sort)

INPUTS="${POD5_FILES[$SLURM_ARRAY_TASK_ID]}"
NAME=$(basename "${INPUTS}" .pod5)

dorado basecaller "${MODEL}" "${INPUTS}" -b "${BS}" > "${WORKDIR}/${NAME}.bam"
```
</div>
</div>

Submit to the SLURM queue with:
```bash
sbatch dorado_array.sh
```
*The script builds the inputs list and uses `${SLURM_ARRAY_TASK_ID}` to select one file per task.*

</div>

### Run basecaller with or without alignment

Dorado can basecall raw-signal `POD5` files and [align the resulting reads](#dorado-dna-integrated-alignment) to a provided reference genome in one step by adding the `--reference` option. This can simplify downstream workflows by avoiding a separate alignment step and producing mapped reads directly.

| | Basecalling without alignment | Basecalling with alignment | |
|---|---|---|---|
| `INPUTS` | ✓ | ✓ | raw-signal `POD5` file(s) or input directory |
| `MODEL`  | ✓ | ✓ | a compatible Dorado model |
| `REFERENCE` | ✗ |  ✓ | reference genome in `FASTA` format; uncompressed or bgzip-compressed | 
| output | raw reads | aligned reads | written by default as a `BAM` file to specified `WORKDIR` | 


{% capture tip_basecalling_alignment %}
```bash
REFERENCE=/path/to/reference_genome.fna

dorado basecaller "${MODEL}" "${INPUTS}" --reference "${REFERENCE}" > "${WORKDIR}/reads_aligned.bam"
```
{% endcapture %}
{% include alert class="highlighted" title="<small>To enable minimap2 read alignment, use: <code>--reference PATH</code></small>" content=tip_basecalling_alignment %}

[Before preparing SLURM submission](#prepare-slurm-submission), test the complete basecalling + `minimap2` alignment workflow on the same five longest reads subset to check whether alignment adds meaningful runtime or memory overhead.

{% capture exercise_4 %}
Run the pilot on the same five longest reads subset with `--reference` enabled. 

**TIP:** *Start with settings optimized in the [Run pilot test(s) in the interactive session](#run-pilot-tests-in-the-interactive-session) step: set `DORADO_CPU_RUNNERS`, limit batch size to 16, use pre-downloaded model.*

<details markdown="1"><summary>SOLUTION</summary>

Confirm resource allocation in your interactive session, or start a new one: 
```bash
srun -A <account> -t 04:00:00 -n 1 --cpus-per-task=8 --mem=64G --pty bash 
```
Load dorado module and set shell variables, including a new REFERENCE path:
```bash
module load dorado   # load dorado module 
export DORADO_CPU_RUNNERS=8   # match requested cores 

MODEL=/reference/workbook/bioinformatics/models/ont_models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0
DATASET={{ page.tutorial2.data_path }}   
INPUT=${DATASET}/{{ page.tutorial2.scale_sample }}
REFERENCE=/reference/workbook/bioinformatics/dataset/ref_genome/monkeypox_virus_NC_063383/GCF_014621545.1_ASM1462154v1_genomic.fna

time memory dorado basecaller ${MODEL} ${INPUT} -v -b 16 -l ../{{ page.tutorial2.scale_read_ids_5 }} --reference "${REFERENCE}" > test_aligned.bam
```
<pre class="padding-x-2 bg-success-lighter"><small>[info] Overriding CPU runners to 8
[debug] - CPU calling: set num_cpu_runners to 8
[info] > Creating basecall pipeline
→ [debug] Loaded index with 1 target seqs
→ [debug] Computing SQ M5 hashes.
→ [debug] Finished computing SQ M5 hashes.
[debug] BasecallerNode chunk size 12288
[debug] Load reads from file
[info] > Finished in (ms): 24789
[info] > Simplex reads basecalled: 5
[info] > Basecalled @ Samples/s: 4.552786e+03
[debug] > Including Padding @ Samples/s: 3.966e+04 (11.48%)
[info] > Finished

Peak system RAM: 5.53 GB

real    0m39.396s
user    1m17.713s
sys     0m30.400s
</small></pre>
Adding alignment *(see new logs in the stdout)* roughly doubled the runtime (now: ~40 seconds) and increased peak RAM from ~3.6 GB to ~5.5 GB. 

</details>

{% include alert class="tip" noicon="true" content="The same CPU and memory allocation remains sufficient, but the [walltime estimate](#walltime-for-batch-job) for the SLURM batch job **should be roughly doubled** to account for the added alignment step."%}

{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_4 %}

Always benchmark the final basecaller settings on your own dataset, especially when adding minimap2 options with `--mm2-opts` or restricting/reporting regions with `--bed-file`, as these can change runtime and resource use. *See [Integrated one-pass features](#integrated-one-pass-features-optional) for more optional settings.*


</div>

## PART 4: Assess reads with summary QC

This part of the tutorial examines the reads produced by Dorado `basecaller` using read-level QC tools. The focus shifts from platform-native processing of raw-signal `POD5` to inspecting the resulting `BAM` and deciding whether additional cleanup is justified. The goal is to generate a read-level summary, review the output, and use the observed read quality and structure to guide any subsequent QC steps.

<div class="process-list" markdown='1'>

### Run `dorado summary`

Use `Dorado summary` as the first read-level QC step on the produced `BAM`.
This keeps the workflow inside the same tool suite while making the output easier to inspect before you decide whether filtering, trimming, or other downstream QC is required. 
Use it to compare runs or basecalling choices. 

<div class="highlighted highlighted--basic"><div class="highlighted__body" markdown="1">
[Dorado](https://github.com/nanoporetech/dorado) `summary` is a reporting tool that reads metadata stored in a Dorado-generated `BAM` and writes a tab-separated sequencing summary. The table makes read counts, lengths, quality scores, timing, barcode information, and alignment fields (when present) easy to inspect with standard command-line or scripting tools. The command itself does not modify the input `BAM`. 

<div class="table-small" markdown="1">

| Option | Description | Example |
| --- | --- | --- |
| input `BAM` | Basecalled read file produced by Dorado | `input.bam` |
| output table | Text summary written to a tabular file | `> summary.tsv` |

</div>

**SYNTAX:** `dorado summary input.bam > summary.tsv`
</div></div>

The `dorado summary` command can be run on a `BAM` file containing either:
- unaligned basecalled reads or 
- reads aligned to a reference genome. 

{% include alert class="highlighted" content="The input should retain the sequencing metadata and tags produced by Dorado basecalling, so an arbitrary `BAM` from another workflow may not contain the information required for a complete Dorado summary."%}

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="Run Dorado summary on the basecalled BAM" controls="dorado-summary-first" class="outline" icon=false %}
<div id="dorado-summary-first" class="accordion_content" markdown='1' hidden>

Start from the `test_1.bam` file created in the [Run pilot test(s) in the interactive session](#run-pilot-tests-in-the-interactive-session).
This is the natural place to inspect **raw read** output after basecalling.

```bash
cd {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}/test_run
module load dorado

dorado summary test_1.bam > raw_reads_summary.tsv
```

Preview the summary:
```bash
less raw_reads_summary.tsv
```

<pre><small>input_filename  batch_id        parent_read_id  read_id run_id  channel mux     minknow_events  start_time      duration        passes_filtering        template_start  num_events_template     template_duration       sequence_length_template        mean_qscore_template    pore_type       experiment_id   sample_id       end_reason
10244_MPOX-67.pod5      0       638c3740-6ea1-42c4-89fc-82988d2e5ff0    638c3740-6ea1-42c4-89fc-82988d2e5ff0    0b914691-6c20-4665-93ae-050bf6db0c0c    34      1       1958    14228.067383    3.743600        TRUE    14229.686523    1770    2.124000        751     20.393127       not_set virba3  virus_batch_3   signal_positive
10244_MPOX-67.pod5      0       af6c7662-365b-4b6d-8dae-53e33bbc9baa    af6c7662-365b-4b6d-8dae-53e33bbc9baa    0b914691-6c20-4665-93ae-050bf6db0c0c    510     2       2425    150825.796875   4.835600        TRUE    150827.000000   3032    3.638400        1110    17.323486       not_set virba3  virus_batch_3   signal_positive
10244_MPOX-67.pod5      0       4e91ef2e-8409-4b67-8ebd-7c5473a2f588    4e91ef2e-8409-4b67-8ebd-7c5473a2f588    0b914691-6c20-4665-93ae-050bf6db0c0c    274     4       2814    155637.796875   5.782400        TRUE    155638.125000   4548    5.457600        724     12.414871       not_set virba3  virus_batch_3   signal_positive
10244_MPOX-67.pod5      0       054d81f9-b25a-42ea-b10d-1b2ad4f79941    054d81f9-b25a-42ea-b10d-1b2ad4f79941    0b914691-6c20-4665-93ae-050bf6db0c0c    419     3       2238    106576.359375   4.058000        TRUE    106576.710938   3088    3.705600        1240    13.297963       not_set virba3  virus_batch_3   signal_positive
10244_MPOX-67.pod5      0       897b8c54-6c91-46f3-862c-6615269d5194    897b8c54-6c91-46f3-862c-6615269d5194    0b914691-6c20-4665-93ae-050bf6db0c0c    367     3       1876    81018.414062    4.160000        TRUE    81019.171875    2833    3.399600        533     12.533134       not_set virba3  virus_batch_3   signal_positive
</small></pre>

For the unaligned BAM, the most useful fields are: `sequence_length_template`, `mean_qscore_template`, `duration`, and the original sequencing metadata. *That output tells you about the reads themselves, but nothing about whether they match the expected genome.* 

Because the original summary contains many tab-separated columns and can be difficult to inspect directly, proceed to [How to extract QC metrics](#how-to-extract-qc-metrics) for a guide to practical steps.

</div>

{% include accordion title="Run Dorado summary on the aligned BAM" controls="dorado-summary-second" class="outline" icon=false %}
<div id="dorado-summary-second" class="accordion_content" markdown='1' hidden>

Start from the `test_aligned.bam` file created in the [Run basecaller with or without alignment](#run-basecaller-with-or-without-alignment).
This is the natural place to inspect **raw read** output after basecalling.

```bash
cd {{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}/test_run
module load dorado

dorado summary test_aligned.bam > aligned_reads_summary.tsv
```

Preview the summary:
```bash
less aligned_reads_summary.tsv
```

<pre><small>input_filename  batch_id        parent_read_id  read_id run_id  channel mux     minknow_events  start_time      duration        passes_filtering        template_start  num_events_template     template_duration       sequence_length_template        mean_qscore_template    pore_type       experiment_id   sample_id       end_reason      alignment_genome        alignment_direction     alignment_genome_start  alignment_genome_end    alignment_strand_start  alignment_strand_end    alignment_num_insertions        alignment_num_deletions      alignment_num_aligned   alignment_num_correct   alignment_identity      alignment_accuracy      alignment_score alignment_coverage      alignment_bed_hits      alignment_mapping_quality       alignment_num_alignments        alignment_num_secondary_alignments      alignment_num_supplementary_alignments
10244_MPOX-67.pod5      0       af6c7662-365b-4b6d-8dae-53e33bbc9baa    af6c7662-365b-4b6d-8dae-53e33bbc9baa    0b914691-6c20-4665-93ae-050bf6db0c0c    510     2       2425    150825.796875   4.835600        TRUE    150827.000000   3032    3.638400        1110    17.323486       not_set virba3  virus_batch_3   signal_positive NC_063383.1     +       95377   96550   13      1097    3       92      1081    1074    0.993524        0.913265        1974    0.973874        0       60      1       0       0
10244_MPOX-67.pod5      0       897b8c54-6c91-46f3-862c-6615269d5194    897b8c54-6c91-46f3-862c-6615269d5194    0b914691-6c20-4665-93ae-050bf6db0c0c    367     3       1876    81018.414062    4.160000        TRUE    81019.171875    2833    3.399600        533     12.533241       not_set virba3  virus_batch_3   signal_positive NC_063383.1     -       134049  134516  68      533     5       7       460     458     0.995652        0.970339        860     0.863039        0       60      1       0       0
10244_MPOX-67.pod5      0       638c3740-6ea1-42c4-89fc-82988d2e5ff0    638c3740-6ea1-42c4-89fc-82988d2e5ff0    0b914691-6c20-4665-93ae-050bf6db0c0c    34      1       1958    14228.067383    3.743600        TRUE    14229.686523    1770    2.124000        751     20.393127       not_set virba3  virus_batch_3   signal_positive NC_063383.1     +       150312  151072  0       742     0       18      742     737     0.993261        0.969737        1402    0.988016        0       60      1       0       0
10244_MPOX-67.pod5      0       054d81f9-b25a-42ea-b10d-1b2ad4f79941    054d81f9-b25a-42ea-b10d-1b2ad4f79941    0b914691-6c20-4665-93ae-050bf6db0c0c    419     3       2238    106576.359375   4.058000        TRUE    106576.710938   3088    3.705600        1240    13.297963       not_set virba3  virus_batch_3   signal_positive NC_063383.1     +       157591  158789  27      1224    20      21      1177    1151    0.977910        0.944992        1988    0.949194        0       60      1       0       0
10244_MPOX-67.pod5      0       4e91ef2e-8409-4b67-8ebd-7c5473a2f588    4e91ef2e-8409-4b67-8ebd-7c5473a2f588    0b914691-6c20-4665-93ae-050bf6db0c0c    274     4       2814    155637.796875   5.782400        TRUE    155638.125000   4548    5.457600        724     12.414871       not_set virba3  virus_batch_3   signal_positive NC_063383.1     -       36404   37102   15      724     15      4       694     681     0.981268        0.955119        1228    0.958564        0       60      1       0       0
</small></pre>

The aligned BAM adds fields such as:

<div class="table-small" markdown="1">

| alignment field | description|
|---|---|
| `alignment_genome`      | reference sequence matched *(here: NC_063383.1)* |
| `alignment_direction`   | forward + or reverse - |
| `alignment_genome_start`/`end` | mapped coordinates on the reference |
| `alignment_num_insertions`/`deletions`   | indels relative to the reference |
| `alignment_num_aligned` | aligned bases  |
| `alignment_num_correct` | matching bases |
| `alignment_identity`    | fraction of aligned positions that match |
| `alignment_accuracy`    | alignment quality including errors/indels |
| `alignment_coverage`    | fraction of the read covered by the alignment |
| `alignment_mapping_quality` | confidence in the mapping |
| secondary/supplementary alignment counts | whether alternative/split mappings were reported |

</div>

*That output tells you how well the reads match the expected genome.* 

Because the original summary contains many tab-separated columns and can be difficult to inspect directly, proceed to [How to extract QC metrics](#how-to-extract-qc-metrics) for a guide to practical steps.

</div>
</div>

### How to extract QC metrics


For a large dorado summary TSV, it is much easier to extract only the QC columns you need and calculate simple statistics from them.

<div class="process-list ul" markdown='1'>

### QC from basecalled reads
{: .no_toc }

Useful fields include `sequence_length_template`, `mean_qscore_template`, and `duration`. 
Commands below provide quick estimates of read count, read-length range, average Q-score, and sequencing duration.

1. First, identify their column numbers:
```bash
SUMMARY=raw_reads_summary.tsv
head -1 "${SUMMARY}" | tr '\t' '\n' | nl
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>     1  input_filename
    2  batch_id
    3  parent_read_id
    4  read_id
    5  run_id
    6  channel
    7  mux
    8  minknow_events
    9  start_time
→   10  duration
    11  passes_filtering
    12  template_start
    13  num_events_template
    14  template_duration
→   15  sequence_length_template
→   16  mean_qscore_template
    17  pore_type
    18  experiment_id
    19  sample_id
    20  end_reason
    </small></pre>
    </details>

1. Then extract a compact table for inspection:
```bash
awk -F'\t' '
NR==1 {
    for (i=1; i<=NF; i++) {
        if ($i=="read_id") rid=i;
        if ($i=="sequence_length_template") len=i;
        if ($i=="mean_qscore_template") q=i;
        if ($i=="duration") dur=i;
    }
    print "read_id\tlength\tmean_qscore\tduration";
    next;
}
{
    print $rid "\t" $len "\t" $q "\t" $dur;
}
' "${SUMMARY}" > summary_qc_raw.tsv
```

1. View it in well-formatted columns:
```bash
column -t -s $'\t' summary_qc_raw.tsv | less -S
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>read_id                               length  mean_qscore  duration
638c3740-6ea1-42c4-89fc-82988d2e5ff0  751     20.393127    3.743600
af6c7662-365b-4b6d-8dae-53e33bbc9baa  1110    17.323486    4.835600
4e91ef2e-8409-4b67-8ebd-7c5473a2f588  724     12.414871    5.782400
054d81f9-b25a-42ea-b10d-1b2ad4f79941  1240    13.297963    4.058000
897b8c54-6c91-46f3-862c-6615269d5194  533     12.533134    4.160000
    </small></pre>
    </details>

1. Calculate basic read-length statistics:
```bash
awk -F'\t' 'NR>1 {n++; sum+=$2; if (n==1 || $2<min) min=$2; if ($2>max) max=$2} END {print "reads:", n; print "mean length:", sum/n; print "min length:", min; print "max length:", max}' summary_qc_raw.tsv
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>reads: 5
mean length: 871.6
min length: 533
max length: 1240
</small></pre>
    </details>

1. Calculate mean Q-score:
```bash
awk -F'\t' 'NR>1 {sum+=$3; n++} END {print "mean Q-score:", sum/n}' summary_qc_raw.tsv
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>mean Q-score: 15.1925</small></pre>
    </details>

1. Calculate mean read duration:
```bash
awk -F'\t' 'NR>1 {sum+=$4; n++} END {print "mean duration:", sum/n}' summary_qc_raw.tsv
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>mean duration: 4.51592</small></pre>
    </details>


**QC Results:** Our five reads have called sequence lengths from about 533 to 1240 bases, mean Q-scores: 15.2 and mean duration: 4.5. 

{% capture exercise_5 %}
The same basecalling QC fields remain available in Dorado-aligned `BAM`s, so you can calculate the same read-length, Q-score, and duration summaries there as well. 
**TIP:** *Assign `SUMMARY=aligned_reads_summary.tsv` and run the same QC commands.*

<details markdown="1"><summary>SOLUTION</summary>

```bash
SUMMARY=aligned_reads_summary.tsv
QC=summary_qc_aligned.tsv

# comapct table for QC
awk -F'\t' 'NR==1 {for (i=1;i<=NF;i++) {if ($i=="read_id") rid=i; if ($i=="sequence_length_template") len=i; if ($i=="mean_qscore_template") q=i; if ($i=="duration") dur=i;} print "read_id\tlength\tmean_qscore\tduration"; next} {print $rid "\t" $len "\t" $q "\t" $dur}' "${SUMMARY}" > "${QC}"
# read-length statistics
awk -F'\t' 'NR>1 {n++; sum+=$2; if (n==1 || $2<min) min=$2; if ($2>max) max=$2} END {print "reads:", n; print "mean length:", sum/n; print "min length:", min; print "max length:", max}' "${QC}"
# mean Q-score
awk -F'\t' 'NR>1 {sum+=$3; n++} END {print "mean Q-score:", sum/n}' "${QC}"
# mean read duration
awk -F'\t' 'NR>1 {sum+=$4; n++} END {print "mean duration:", sum/n}' "${QC}"
```
<pre class="padding-x-2 bg-success-lighter"><small>reads: 5
mean length: 871.6
min length: 533
max length: 1240
mean Q-score: 15.1925
mean duration: 4.51592
</small></pre>
The unaligned and aligned `BAM`s produced the exact same basecalling QC statistics.
</details>
{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_5 %}

### QC from aligned reads 
{: .no_toc }

Aligned `BAM`s add mapping-specific fields; most useful include `alignment_identity`, `alignment_accuracy`, `alignment_coverage`, `alignment_mapping_quality`, `alignment_num_secondary_alignments` / `alignment_num_supplementary_alignments`, `alignment_num_insertions`, and `alignment_num_deletions`.  
Commands below provide metrics such as alignment identity, accuracy, coverage, mapping quality, reference coordinates, and secondary/supplementary alignments. 

1. Extract the key alignment columns:
    ```bash
    SUMMARY=aligned_reads_summary.tsv
    QC=summary_qc_aligned.tsv

    awk -F'\t' 'NR==1 {for(i=1;i<=NF;i++){if($i=="read_id") rid=i; if($i=="alignment_genome") ref=i; if($i=="alignment_genome_start") start=i; if($i=="alignment_genome_end") end=i; if($i=="alignment_identity") ident=i; if($i=="alignment_accuracy") acc=i; if($i=="alignment_coverage") cov=i; if($i=="alignment_mapping_quality") mapq=i; if($i=="alignment_num_secondary_alignments") sec=i; if($i=="alignment_num_supplementary_alignments") supp=i;} print "read_id\treference\tstart\tend\tidentity\taccuracy\tcoverage\tMAPQ\tsecondary\tsupplementary"; next} {print $rid"\t"$ref"\t"$start"\t"$end"\t"$ident"\t"$acc"\t"$cov"\t"$mapq"\t"$sec"\t"$supp}' "${SUMMARY}" > "${QC}"
    ```

2. View it in well-formatted columns:
```bash
column -t -s $'\t' "${QC}" | less -S
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>read_id                               reference    start   end     identity  accuracy  coverage  MAPQ  secondary  supplementary
af6c7662-365b-4b6d-8dae-53e33bbc9baa  NC_063383.1  95377   96550   0.993524  0.913265  0.973874  60    0          0
897b8c54-6c91-46f3-862c-6615269d5194  NC_063383.1  134049  134516  0.995652  0.970339  0.863039  60    0          0
638c3740-6ea1-42c4-89fc-82988d2e5ff0  NC_063383.1  150312  151072  0.993261  0.969737  0.988016  60    0          0
054d81f9-b25a-42ea-b10d-1b2ad4f79941  NC_063383.1  157591  158789  0.977910  0.944992  0.949194  60    0          0
4e91ef2e-8409-4b67-8ebd-7c5473a2f588  NC_063383.1  36404   37102   0.981268  0.955119  0.958564  60    0          0
    </small></pre>
    </details>

3. Calculate mean identity, accuracy, coverage, and mapping quality:
```bash
awk -F'\t' 'NR>1 {n++; ident+=$5; acc+=$6; cov+=$7; mapq+=$8} END {print "reads:",n; print "mean identity:",ident/n; print "mean accuracy:",acc/n; print "mean coverage:",cov/n; print "mean MAPQ:",mapq/n}' "${QC}"
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>reads: 5
mean identity: 0.988323
mean accuracy: 0.95069
mean coverage: 0.946537
mean MAPQ: 60
    </small></pre>
    </details>

4. Count reads with secondary or supplementary alignments:
```bash
awk -F'\t' 'NR>1 {if($9>0) sec++; if($10>0) supp++} END {print "reads with secondary alignments:",sec+0; print "reads with supplementary alignments:",supp+0}' "${QC}"
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>reads with secondary alignments: 0
reads with supplementary alignments: 0
    </small></pre>
    </details>

5. Count reads with a reported `alignment_genome` as *mapped* and blank entries as *unmapped*.
```bash
awk -F'\t' 'NR==1 {for(i=1;i<=NF;i++) if($i=="alignment_genome") ref=i; next} {if($ref!="") mapped++; else unmapped++} END {print "mapped:",mapped+0; print "unmapped:",unmapped+0}' "${SUMMARY}"
```
    <details class="padding-x-2 bg-success-lighter"><summary><i>command log</i></summary>

    <pre><small>mapped: 5
unmapped: 0
    </small></pre>
    </details>


**QC Results:** So all five reads mapped uniquely and confidently to the reference: `MAPQ = 60`, with no secondary or supplementary alignments. Identity is also high, around **98.8%** with **95%** accuracy and **94.6%** coverage.

</div>


### Decide whether `trim` or `correct` is justified

Do **not** treat every Dorado subcommand as a required step. 
Use the [QC results](#how-to-extract-qc-metrics) to decide whether any additional read-level processing is justified. 
- **Q-score evaluates the basecall**
- **MAPQ evaluates confidence in where the read maps**
- **identity/accuracy evaluate how well it matches there**
- **coverage tells how much of the read was aligned**

{% include alert class="success" content="In the **five longest reads subset**, the reads show reasonable basecalling quality and strong, unique alignment to the expected reference, so there is no evidence from these metrics alone that trimming or correction is necessary.
- A Q-score of 15 corresponds to roughly 97% per-base accuracy, so this is reasonable for ONT reads, although higher is better.  
- All five reads also mapped uniquely to the expected reference with MAPQ 60, which is the maximum value commonly reported by minimap2 and indicates a highly confident primary alignment.  
- The reads show about 98.8% alignment identity, meaning nearly all aligned bases match the reference; 95% alignment accuracy, indicating relatively few mismatches and indels overall; and 94.6% alignment coverage, meaning most of each read participates in the alignment.  
"%}

{% include alert class="tip" content="At this stage, use the QC results to decide whether the reads show a specific problem that trim or correct is designed to address.
- Always use `summary`, because it helps you inspect what the basecaller produced. *See [Run `dorado summary`](#run-dorado-summary)*
- Use [dorado trim](#dorado-trim) only when adapters or primers are confirmed to remain in the basecalled reads.
- Use [dorado correct](#dorado-correct) only when the downstream analysis specifically benefits from error-corrected reads; it is not a routine QC step.
"%}

{% include alert class="warning" title="Do not force all Dorado subcommands into one run" content="`summary`, `trim`, and `correct` are **not** a required sequence. Use them selectively. In the ONT reads QC, the main default step is `summary`; stronger post-basecalling cleanup should be justified by visible evidence and the downstream analysis plan." %}

<div class="process-list ul" markdown='1'>

### dorado trim
{: .no_toc }

<div class="highlighted highlighted--basic"><div class="highlighted__body" markdown="1">
`dorado trim` removes adapter and primer sequences from basecalled reads. It is usually unnecessary after standard [dorado basecaller runs](#run-basecaller-on-the-dna-pod5-files), because trimming is enabled by default, but is useful for externally generated reads or reads basecalled with `--no-trim`.
</div></div>

[Dorado basecaller](#run-basecaller-on-the-dna-pod5-files) normally attempts adapter/primer trimming automatically; using `--no-trim` preserves those sequences so you can inspect them explicitly. 

A practical check is to **basecall without trimming** first, then scan the resulting reads specifically for known adapter/primer sequences.
```bash
dorado basecaller "${MODEL}" "${INPUT}" --no-trim --emit-fastq > reads_untrimmed.fastq
```

Confirm adapter/primer contamination by searching the basecalled read ends for the expected library-specific sequences. 
If they are detected in a meaningful fraction of reads, trimming is justified; **if not, additional trimming is unnecessary**.

#### Option #1

Dorado can run this detection itself on **existing basecalled data** and it supports custom primer sequences when needed.
```bash
dorado trim reads_untrimmed.fastq > trimmed.bam
# or: dorado trim reads_untrimmed.fastq --emit-fastq > trimmed.fastq
```

<details markdown="1"><summary>Optional arguments</summary>

- `-k`, `--sequencing-kit`   
*Use when you know the ONT sequencing kit and want Dorado to select the corresponding built-in adapters and primers.* 
```bash
dorado trim reads.bam -k SQK-LSK114 > reads_trimmed.bam
```
- `--primer-sequences`  
*Use when primers are custom, or when using a supported third-party primer set such as `10X_Genomics`.* 
```bash
dorado trim reads.bam --primer-sequences primers.fasta > reads_trimmed.bam
dorado trim reads.bam --primer-sequences 10X_Genomics > reads_trimmed.bam
```
- `--no-trim-primers`  
*Use when you want to trim adapters but keep primer sequences in the reads.* 
```bash
dorado trim reads.bam -k SQK-LSK114 --no-trim-primers > reads_adapters_trimmed.bam
```
- `--rna`  
*Use for RNA reads so Dorado applies RNA-specific trimming behavior.* 
```bash
dorado trim reads.bam --rna > reads_trimmed.bam
```
</details>

#### Option #2

Search the `FASTQ` read ends for the expected adapter/primer sequences with a tool such as `cutadapt`, `seqkit`, or `grep` for an exact-match spot check.
```bash
module load cutadapt
cutadapt -v           # save version used
cutadapt -g ADAPTER_SEQUENCE -a ADAPTER_SEQUENCE reads_untrimmed.fastq -o /dev/null
```
*Cutadapt will report how many reads contain the sequence and where it was detected.*


### dorado correct
{: .no_toc }

<div class="highlighted highlighted--basic"><div class="highlighted__body" markdown="1">
`dorado correct` performs **single-read error correction** using information from overlapping reads. It is intended mainly for workflows such as **de novo assembly**, where improving read accuracy can be useful before assembly; it is not a routine QC step.
</div></div>

A practical question is therefore whether your downstream analysis actually benefits from corrected reads. For targeted mapping to a known reference, such as the workflow in this tutorial, correction is usually not necessary when the reads already map uniquely and with high identity.

#### When needed only...

Run correction directly on basecalled reads in `FASTQ` format.

```bash
dorado correct reads.fastq > corrected_reads.fasta
```

`dorado correct` accepts `FASTQ` input and outputs corrected reads in `FASTA` format. The correction workflow is computationally intensive and is designed to use substantial CPU, memory, and typically GPU resources.

<details markdown="1"><summary>Optional arguments</summary>

- `-m`, `--model-path`  
*Use a pre-downloaded correction model instead of allowing Dorado to obtain it automatically.*
```bash
dorado download --model herro-v1
dorado correct -m herro-v1 reads.fastq > corrected_reads.fasta
```
- `--to-paf`  
*Run only the read-overlap mapping stage and save the overlaps for later correction.*
```bash
dorado correct reads.fastq --to-paf > overlaps.paf
```
- `--from-paf`  
*Run correction using previously generated read overlaps.*
```bash
dorado correct reads.fastq --from-paf overlaps.paf > corrected_reads.fasta
```
{% include alert class="tip" content="This can be useful on HPC systems because the CPU-heavy overlap stage and GPU-heavy correction stage can be run separately."%}

</details>

</div>


### Continue to read quality profiling

If the remaining questions are about read quality, length distribution, or retained yield, continue to [Long-read ONT quality check with NanoPlot](/bioinformatics/reads-qc/long-read/ont/qc_nanoplot).

If you still need ONT-specific or raw signal-aware preprocessing, explore other tools in the [Dorado suite](#load-dorado-module).

</div>


## FAQ and troubleshooting

Use these quick checks to troubleshoot input format, model download, or CPU run issues.

<div class="usa-accordion" data-allow-multiple>

{% include accordion title="Is .fast5 still supported?" controls="dorado-faq-fast5" expanded=false class="outline" icon=false %}
<div id="dorado-faq-fast5" class="accordion_content" markdown="1">
Support for `.fast5` is deprecated. Current Dorado workflows are centered on `POD5`.

`pod5` is a separate Oxford Nanopore command-line tool/package for working with `POD5` files, including conversion from `.fast5`; see the official [POD5 tools documentation](https://software-docs.nanoporetech.com/pod5/latest/tools/).

If your input is multi-read `.fast5`, convert it to `POD5` with:

```bash
pod5 convert fast5 ./input/*.fast5 --output converted.pod5
```

If you want one `.pod5` per input `.fast5` instead:

```bash
pod5 convert fast5 ./input/*.fast5 --output output_pod5s/ --one-to-one ./input/
```

If your files are single-read `.fast5`, first convert them to multi-read `.fast5` with `ont_fast5_api`, then run `pod5 convert fast5`.
</div>

{% include accordion title="How does Dorado choose the basecalling model?" controls="dorado-faq-model" expanded=false class="outline" icon=false %}
<div id="dorado-faq-model" class="accordion_content" markdown="1">
The choice is made from the `POD5` metadata, not from the filename alone. Dorado reads metadata stored in the `POD5` input, determines the matching chemistry context such as `dna_r10.4.1_e8.2_400bps`, and combines that with the requested speed-vs-accuracy option to choose the full model name, for example `dna_r10.4.1_e8.2_400bps_hac@v6.0.0`. The model category is specified as the second positional argument in `dorado basecaller <model> <pod5>` and can be given as `fast`, `hac` (high-accuracy), or `sup` (super-accurate).
</div>


{% include accordion title="Is an ONT multiplex POD5 dataset available?" controls="dorado-faq-multiplex" expanded=false class="outline" icon=false %}
<div id="dorado-faq-multiplex" class="accordion_content" markdown="1">
For a quick example multiplex `POD5` input, use:

`/reference/workbook/bioinformatics/dataset/reads_long/ont/dna_r10.4.1_e8.2_400bps_5khz/`

{% include segment/ref_dataset_ed dataset="ont_dna_dorado_pod5" %}
</div>


{% include accordion title="How do I download a dataset from SquiDBase?" controls="dorado-faq-squidbase" expanded=false class="outline" icon=false %}
<div id="dorado-faq-squidbase" class="accordion_content" markdown="1">

SquiDBase exposes dataset metadata through a public API. The workbook example dataset is [SQB000012](https://squidbase.org/submissions/SQB000012), and its metadata endpoint is:

`https://api.squidbase.org/api/v1/submissions?scientific_id=SQB000012&b64=true`

This pattern can also be reused for your own dataset. Replace `PROJ="SQB000012"` with the SquiDBase dataset ID you want to download.

That JSON includes the file list, download URLs, and the sample sheet metadata needed to select a subset such as the PAO1 files.

To download all PAO1 `POD5` files from that dataset:

```bash
PROJ="SQB000012"
SPECIES="Pseudomonas aeruginosa PAO1"

mkdir -p pod5_pao1
curl -s "https://api.squidbase.org/api/v1/submissions?scientific_id=${PROJ}&b64=true" \
| jq -r '
  [.samplesheet | to_entries[]
    | select((.value.species_taxon_name // "") == env.SPECIES)
    | .key] as $pa
  | .files[]
  | select(.name as $n | $pa | index($n))
  | [.name, .url] | @tsv
' \
| while IFS=$'\t' read -r name url; do
    curl -L "$url" -o "pod5_pao1/$name"
  done
```

If you are working with a different SquiDBase dataset:

1. Open the dataset page and note its dataset ID, such as `SQB000012`.
2. Set `PROJ` to that dataset ID in the command above.
3. Set `SPECIES` to the species or sample label you want to extract.
4. Run the API query again when you are ready to download, because the returned file URLs can expire.

You do not need to generate those file URLs manually. The API response already includes them, and rerunning the API request gives you a fresh set when needed.

</div>

{% include accordion title="Unknown chemistry or Failed to resolve basecaller models" controls="dorado-faq-unknown-chemistry" expanded=false class="error" icon=false %}
<div id="dorado-faq-unknown-chemistry" class="accordion_content" markdown="1">

If Dorado fails with an error like:

<pre class="bg-error-lighter"><small>[error] No supported chemistry found for flowcell_code: 'FLO-MIN112' sequencing_kit: '__UNKNOWN_KIT__' sample_rate: 4000 
[error] This is typically seen when using prototype kits. Please download an appropriate model for your data and select it by model path 
[error] Exception thrown: Failed to resolve basecaller models: Could not resolve chemistry from data: Unknown chemistry 
</small></pre>

then the problem is usually **not** your command syntax. Dorado could not resolve a usable chemistry from the `POD5` metadata.

- **Legacy condition → use a different tool** <br>
  If the dataset is based on **R10.4**, this behavior is expected. ONT documents **R10.4** as a legacy condition that should be basecalled with **Guppy**. Dorado supports **DNA R10.4.1**, not legacy **R10.4**.

- **Confirmed Dorado support - specify exact model** <br>
  If your data is **not** a legacy condition and you know the exact Dorado model that matches the chemistry, you can bypass auto-selection by providing the model directly instead of using `hac`, `sup`, or `fast`.
  ```bash
  # Download the exact model you need
  dorado download --model dna_r10.4.1_e8.2_400bps_hac@v6.0.0 --models-directory ./models
  # Run dorado basecaller with the model name or model path
  dorado basecaller ./models/dna_r10.4.1_e8.2_400bps_hac@v6.0.0 "${INPUT}" > calls.bam
  ```

</div>


{% include accordion title="CPU basecalling is killed" controls="dorado-faq-cpu-killed" expanded=false class="error" icon=false %}
<div id="dorado-faq-cpu-killed" class="accordion_content" markdown="1">

If Dorado falls back to CPU and then exits with `Killed`:

<pre class="bg-error-lighter"><small>[info] > Creating basecall pipeline
Killed
</small></pre>

the job often has too many visible CPUs and too much memory pressure. Do not rely on the number of tasks only (e.g., `-n 2`), because that is *not* a safe CPU limit for one Dorado process. Use one task with explicit CPU and memory limits instead.

- For interactive testing:
```bash
srun -A <project_name> -t 04:00:00 -n 1 --cpus-per-task=4 --mem=32G --pty bash
```

- For batch scripts:
```bash
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
```

If needed, start with a small test run:

- `-n <N>` limits how many **reads** Dorado will basecall in total
- `-b <N>` limits how many signal **chunks** Dorado sends to the model at once

Here, a **read** is one nanopore read from the `POD5` file, while a **chunk** is one internal signal segment cut from a read for model inference. Lower `-b` values reduce memory use much more directly than lower `-n` values.

```bash
dorado basecaller -x cpu -n 200 -b 4 hac "${INPUT}" > test_calls.bam
```
</div>

{% include accordion title="Failed to build index file for reference file" controls="dorado-faq-missing-index" expanded=false class="error" icon=false %}
<div id="dorado-faq-missing-index" class="accordion_content" markdown="1">

If Dorado `basecaller` fails with an error like:

<pre class="bg-error-lighter"><small>[warning] Empty index path generated for input file:
[error] Failed to create a .fai index for reference 
[E::fai_build3_core] Cannot index files compressed with gzip, please use bgzip
</small></pre>

Dorado's **HTSlib** library cannot create a `.fai` index for plain gzip-compressed `FASTA`; it requires an uncompressed or bgzip-compressed `FASTA`.

Check if your reference is compressed with ordinary `gzip`; if so, decompress it:
```bash
gunzip -c reference.gz > reference_genomic.fna
```
</div>

</div>

## Summary

This tutorial keeps the ONT DNA Dorado workflow split into two practical stages.
First, use `basecaller` as the **platform-native preprocessing** step from raw-signal `POD5`.
Then use Dorado read-level outputs, especially `summary`, to decide whether additional QC or cleanup is justified before downstream analysis.
