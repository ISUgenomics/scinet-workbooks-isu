---
title: "Create Local Sequence Databases for BLAST"
description: "Inspect FASTA files, build and verify custom nucleotide and protein BLAST databases, and retrieve query sequences for downstream searches."
svg: /genomics.svg
author: Rick Masonbrink
language: Bash

header:
  overlay_image: 07-wrangling/assets/img/07_data_acquisition_banner.png

index:
type: interactive tutorial
order: 2
wg: Bioinformatics
tags: [BLAST, command line]

related:
  - "[Sequence search and similarity](/bioinformatics/sequence-search/similarity/)"
  - "[Bioinformatics resources](/bioinformatics/resources/)"
  - "[Workspace setup for bioinformatics](/bioinformatics/resources/workspace_setup)"

references:
  - "[NCBI BLAST+ documentation](https://www.ncbi.nlm.nih.gov/books/NBK279690/)"
  - "[NCBI Datasets](https://www.ncbi.nlm.nih.gov/datasets/)"
  - "[SCINet Public Databases workbook](/bioinformatics/resources/databases)"

tools:
  blast:
    name: NCBI BLAST+
    module:
      atlas: blastplus
      ceres: blast+
    version:
      atlas: 2.17.0
      ceres: 2.15.0

objectives:
  - Distinguish between nucleotide and protein BLAST databases.
  - Build and verify custom databases with `makeblastdb` and `blastdbcmd`.
  - Preserve sequence identifiers and retrieve query sequences from a database.
  - Organize database files for reuse in local similarity searches.

applications:
  - Building searchable references from newly assembled or unpublished sequences.
  - Maintaining organism-specific sequence collections.
  - Preparing reusable databases and query sequences for local BLAST searches.

questions:
  - question: "You are building a BLAST database from amino-acid sequences. Which `-dbtype` value is correct?"
    title: "Database molecule type"
    qid: "1"
    answers:
      - nucl
      - prot
    answer: 2
    solution: "Protein sequences require `-dbtype prot`."

  - question: "Which `-dbtype` should be used when building a database from genomic DNA?"
    title: "Database molecule type"
    qid: "2"
    answers:
      - nucl
      - prot
    answer: 1
    solution: "DNA sequence databases require `-dbtype nucl`."

  - question: "Why is `-parse_seqids` useful when creating a BLAST database?"
    title: "Sequence identifiers"
    qid: "3"
    answers:
      - It preserves sequence identifiers for later record retrieval.
      - It translates nucleotide sequences identifiers into protein identifiers.
      - It combines all database index files into one file.
    answer: 1
    solution: "`-parse_seqids` stores the FASTA identifiers so records can later be retrieved with `blastdbcmd`."

  - question: "What should be supplied to the BLAST `-db` option?"
    title: "Database name"
    qid: "4"
    answers:
      - The database prefix given to `makeblastdb -out`.
      - The name of one individual index file, such as `.nin` or `.pin`.
      - The original compressed FASTA filename.
    answer: 1
    solution: "Use the database prefix given to `makeblastdb -out`, not an individual index file."

  - question: "Which command displays the type, sequence count, and total length of a BLAST database?"
    title: "Database metadata"
    qid: "5"
    answers:
      - "`blastdbcmd -db DATABASE_PREFIX -info`"
      - "`makeblastdb -db DATABASE_PREFIX -info`"
      - "`blastdbcmd -db DATABASE_PREFIX -entry all -outfmt '%f'`"
    answer: 1
    solution: "Run `blastdbcmd -db DATABASE_PREFIX -info`."

  - question: "What must you do after adding, removing, or editing sequences in the source FASTA?"
    title: "Database update"
    qid: "6"
    answers:
      - Rebuild the BLAST database with `makeblastdb`.
      - Edit one of the existing BLAST index files.
      - Run `blastdbcmd -info` to update the database.
    answer: 1
    solution: "Run `makeblastdb` again because BLAST databases do not update automatically."

updated: 2026-09-18

tutorial:
  root_practice: /90daydata/shared/$USER
  workdir: tutorials/sequence_search/database_prep
  data_path_1: /reference/workbook/bioinformatics/dataset/ref_proteome/grape_phylloxera
  data_path_2: /reference/workbook/bioinformatics/dataset/ref_proteome/grape_wine
  dvitifoliae_genome: DvitifoliaeGenome.fasta
  dvitifoliae_proteome: DvitifoliaeProteome.fasta
  vvinifera_genome: VviniferaGenome.fasta
  vvinifera_proteome: VviniferaProteome.fasta
  dvitifoliae_genome_db: DvitifoliaeGenomeDB
  dvitifoliae_proteome_db: DvitifoliaeProteomeDB
  vvinifera_genome_db: VviniferaGenomeDB
  vvinifera_proteome_db: VviniferaProteomeDB
  protein_query: XP_050524263.1
  nucleotide_query: NW_026099572.1
  nucleotide_range: 72542000-72552000
  threads: 2
  evalue: 1e-5
  max_target_seqs: 10

materials:
  - "*Daktulosphaira vitifoliae* and *Vitis vinifera* genome and predicted-proteome FASTA files from NCBI."
  - "Use `/90daydata/shared/$USER/` as a practice workspace with enough space for compressed inputs, extracted FASTA files, BLAST index files, and search results."

terms:
  - term: FASTA
    definition: A text format that stores biological sequences and their identifiers.
  - term: BLAST database
    definition: A searchable collection of indexed nucleotide or protein sequences.
  - term: database prefix
    definition: The path and base name used to identify all files belonging to a BLAST database.
  - term: sequence identifier
    definition: The unique label after `>` in a FASTA header that BLAST stores for record retrieval.

overview: [objectives, applications, terminology, materials]
---

## Overview

BLAST searches can use public databases maintained by the National Center for Biotechnology Information (NCBI) or custom databases built from locally available sequences. A custom database is useful when sequences are unpublished, organism-specific, newly assembled, or curated for a particular project.

In this tutorial, you will download two reference genomes and their predicted proteomes: *Vitis vinifera* (common grapevine) and its pest *Daktulosphaira vitifoliae* (grape phylloxera). Using these files, you will build nucleotide and protein BLAST databases to inspect and retrieve database records.

{% include overviews %}


## Getting Started

{% include segment/getting_started time="02:00:00" cpus_per_task="2" mem="4G" %}

**Tutorial Steps:**
1. [Prepare the practice workspace](#prepare-the-practice-workspace).
1. [Get the dataset](#get-the-dataset) from NCBI.
1. [Load NCBI BLAST+ tools](#load-ncbi-blast-tools).
1. [Choose the database type](#choose-the-database-type) for each sequence collection.
1. [Create databases with `makeblastdb`](#create-databases-with-makeblastdb).
1. [Query a database with `blastdbcmd`](#query-a-database-with-blastdbcmd).
1. [Rebuild a database after changing its FASTA file](#rebuild-a-database-after-changing-its-fasta-file).
1. Continue with the next tutorial: [Search sequence similarity with BLAST](/bioinformatics/sequence-search/similarity/blast#tutorial-steps).

## Tutorial Steps

<div class="process-list" markdown="1">

### Prepare the practice workspace

{% include setup/practice_workspace workdir=page.tutorial.workdir %}


### Get the dataset

This tutorial uses versioned genome assemblies from NCBI:

| Organism | Assembly accession | Files used | File format | Size |
|---|---|---|---|---|
| *Daktulosphaira vitifoliae* | [GCF_025091365.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_025091365.1/) | Genome and predicted proteome | `FASTA` | 89M |
| *Vitis vinifera* | [GCF_030704535.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_030704535.1/) | Genome and predicted proteome | `FASTA` | 152M |

A **nucleotide FASTA** record contains an identifier line beginning with `>` followed by one or more lines of nucleotide sequence:

{:.no-copy}
```text
>scaffold_001
ATGCGTACGTAGCTAGCTAGCTAGCTAGCTAGCTAGC...
```

A **protein FASTA** record  has the same structure but contains amino acid sequence:

{:.no-copy}
```text
>protein_001
MSTNPKPQRKTKRNTNRRPQDVKFPGGGQIVGGVLTALA...
```


#### Download the FASTA files

Both datasets total less than `250 MB` and should download within a few seconds.  
The `-c` option allows `wget` to continue a partially completed download:
```bash
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/025/091/365/GCF_025091365.1_ASM2509136v1/GCF_025091365.1_ASM2509136v1_genomic.fna.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/025/091/365/GCF_025091365.1_ASM2509136v1/GCF_025091365.1_ASM2509136v1_protein.faa.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/030/704/535/GCF_030704535.1_ASM3070453v1/GCF_030704535.1_ASM3070453v1_genomic.fna.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/030/704/535/GCF_030704535.1_ASM3070453v1/GCF_030704535.1_ASM3070453v1_protein.faa.gz
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>total 239M
 84M GCF_025091365.1_ASM2509136v1_genomic.fna.gz
5.0M GCF_025091365.1_ASM2509136v1_protein.faa.gz
142M GCF_030704535.1_ASM3070453v1_genomic.fna.gz
9.6M GCF_030704535.1_ASM3070453v1_protein.faa.gz
</small></pre>
</details>

Verify the downloaded archives:
```bash
gzip -t GCF*.fna.gz && echo "All gzip files are OK"
```
<pre><small>All gzip files are OK</small></pre>

{% include alert class="tip" content="If the file is valid, it usually prints nothing and exits successfully. If the gzip file is corrupted or incomplete, it reports an error." %}

Extract and rename the files:
```bash
gunzip GCF_*.gz

mv GCF_025091365.1_ASM2509136v1_genomic.fna DvitifoliaeGenome.fasta
mv GCF_025091365.1_ASM2509136v1_protein.faa DvitifoliaeProteome.fasta
mv GCF_030704535.1_ASM3070453v1_genomic.fna VviniferaGenome.fasta
mv GCF_030704535.1_ASM3070453v1_protein.faa VviniferaProteome.fasta

ls
```
<pre><small>DvitifoliaeGenome.fasta  DvitifoliaeProteome.fasta  VviniferaGenome.fasta  VviniferaProteome.fasta</small></pre>

Define [shell variables](/computing-skills/command-line/configuration/variables#shell-variables) for filenames to avoid typo issues:
```bash
D_GENOME="{{page.tutorial.dvitifoliae_genome }}"
D_PROTEOME="{{ page.tutorial.dvitifoliae_proteome }}"
V_GENOME="{{ page.tutorial.vvinifera_genome }}"
V_PROTEOME="{{ page.tutorial.vvinifera_proteome }}"
```


#### Inspect the input FASTA files

Before building a database, review the sequence type, record structure, and number of records in each input file.

Inspect the beginning of one protein and one nucleotide file:

```bash
head "${V_PROTEOME}"
head "${D_GENOME}"
```

<details class="padding-x-2 bg-info-lighter"><summary><i>V_PROTEOME</i></summary>

<pre><small>>NP_001267815.1 nhx1 antiporter [Vitis vinifera]
MGFELGSVVMKLGMVSTSDHSSVVSMNLFVALLCACIVVDHWLEEYRWMNESITALALGLCTGIIILLTTRGKSSHILVF
SEDLFFIYLLPPIIFNAGFQVKKKQFFRTFMTIMLFGAIGTXISFGIISLGAIHFFKKMKIGSXDIEDXLALGXIFSATD
SVCTLQVLNQDETPLLYSLVFGEGVVNDATSVVLFNAIQSFDLSHIDSSIALQFIGNFLYLFITSTMLGVFAGLLSAYII
KKLYFGRHSTDREVAIMILMAYLSYMLAELFYLSAILTVFFCGIVMSHYTWHNVTESSRVTTKHAFATLSFVAEIFIFLY
VGMDALDIEKWRFVSDSPGKSIGVSSILLGLVLVERAAFVFPLSFLSNLTKKSSSEKIHIEQQVTIWWAGLMRGAVSMAL
AYNQFTRAGHTQLRGNAIMITSTISVVLFSTVVFGLMTKPLVRLLLPSPKPFSSMISSEPSSPKYLVVPLIGNGEETETD
QASQNVPRPTSLRMLLSTPSHTVHHYWRKFDDSFMRPVFGGRGFTPFIPGSPTEPNLGQWR
>NP_001267817.1 hexose transporter HT2 [Vitis vinifera]
MAVGGFAADDNSRAFSGKVTASVVITCIVAASGGLIFGYDIGISGGVTTMQPFLKKFFPVVLRKAADAKTNIYCVYDSHV
</small></pre>
</details>
<details class="padding-x-2 bg-info-lighter"><summary><i>D_GENOME</i></summary>

<pre><small>>NW_026099572.1 Daktulosphaira vitifoliae isolate Bord-2020 unplaced genomic scaffold, ASM2509136v1 Scaffold_1, whole genome shotgun sequence
GTGAGAATCGAATGTGATTCTCAAAGCGAGAATGGCAAGTTAACCTAAATGAGAGGGGCACTATACAAAACTACGCCGTG
TTGATACATGATTGTATGATACTCAAGGCAGTCAACTTGTTAAGGCGAGAAAAATGAGTACAAtaagtacataaataata
cagaCCACTTTCAtctgtcataatttttttttatttgggaCCTTTTCATTTTATACACGATGGGTTCAAATCTTGAAATA
TTCgagtaaaaagtagaactttttaaaattaagtattgaaaatgttccgtaaATATTGTGTCTTTtgaacatatgttaaa
tttaagtttatgttacatactataagttaaaaaaaaaattgtcccaaaGACTTATCTTCTTTGTGCAAATGATCATCgaa
gttgaacaaaaaaatgagttaattttgcttatataataatccaaatgaaaaatttgaaaacagtgtgaaatattgattta
gaaaagtttattttagtttacgtTGAAAATTACTTGTAGacaaatacatttattcataattaaccaatcatttttttttt
aaagtactgCACAAGctctttaatttgtataaacgtttatcataaaagtagttataatttaataaattttttataatttc
ttagtaaaaattacattattaaattattaaatatcagctcatactatatttaatcgctatagtaaatatttattgataca
</small></pre>
</details>


Count the FASTA identifiers in all four files:
```bash
grep -c '^>' *fasta
```

<pre><small>DvitifoliaeGenome.fasta:8544
DvitifoliaeProteome.fasta:29510
VviniferaGenome.fasta:21
VviniferaProteome.fasta:40633
</small></pre>

Each count represents the number of FASTA records in that file.


### Load NCBI BLAST+ tools

{% include tool_summary tool_key="blast" %}

{% include setup/module tool_key="blast" known="true" %}
{% include setup/tool_verify tool_key="blast" exec="makeblastdb" version_cmd="makeblastdb -version" help_cmd="makeblastdb -help" %}


### Choose the database type

Every BLAST database is built as either a protein or nucleotide database:
- The database supplied to `blastn`, `tblastn`, or `tblastx` is built with `-dbtype nucl`. 
- The database supplied to `blastp` or `blastx` is built with `-dbtype prot`.

{% include alert class="highlighted" noicon="true" content="The database type must match the sequences in the input FASTA file. It cannot be changed after the database is built.

| Input sequences | `makeblastdb` option | Programs that can search the database |
|---|---|---|
| Nucleotide | `-dbtype nucl` | `blastn`, `tblastn`, `tblastx` |
| Protein | `-dbtype prot` | `blastp`, `blastx` |" %}


### Create databases with `makeblastdb`

{% capture tool_makeblastdb %}
**makeblastdb** converts a FASTA file into a searchable BLAST database. 

{:.no-copy}
```bash
makeblastdb -in <input_file> -out <database_name> -dbtype <molecule_type> -parse_seqids
```
<details class="padding-x-2 margin-top-0" markdown="1"><summary><i>explanation of options</i></summary>

* `-in` identifies the input FASTA file.
* `-dbtype nucl` creates a nucleotide database.
* `-parse_seqids` stores FASTA identifiers for later extraction.
* `-out` sets the database prefix; it can include nested path, where the last element becomes database name.

*Use `makeblastdb -h` or `makeblastdb -help` for more options.*
</details>

**NOTE:** Putting each BLAST database in its own directory keeps the multiple index files created by `makeblastdb` together and makes cleanup, transfer, and reuse easier. To keep all database files in a parent directory with the same name, use pattern: `-out <db_name>/<db_name>` .
{% endcapture %}
{% include alert class="basic" content=tool_makeblastdb %}

<div class="process-list ul" markdown="1">

### Nucleotide databases

Build a database from the genome `FASTA` files:

```bash
makeblastdb \
  -in DvitifoliaeGenome.fasta -out DvitifoliaeGenomeDB/DvitifoliaeGenomeDB \
  -dbtype nucl -parse_seqids
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>Building a new DB, current time: 09/18/2026 16:20:22
New DB name:   /90daydata/.../database_prep/DvitifoliaeGenomeDB/DvitifoliaeGenomeDB
New DB title:  DvitifoliaeGenome.fasta
Sequence type: Nucleotide
Keep MBits: T
Maximum file size: 3000000000B
Adding sequences from FASTA; added 8544 sequences in 2.58362 seconds.

real    0m2.997s
</small></pre>
</details>

{% capture exercise_1 %}
Build a database for the second genome: `VviniferaGenome`.

<details markdown="1"><summary>SOLUTION</summary>

```bash
#V_GENOME="VviniferaGenome.fasta"   # input FASTA, defined earlier
V_GENOME_DB="{{ page.tutorial.vvinifera_genome_db }}"     # database name, stored in a shell variable

makeblastdb \
  -in "${V_GENOME}" -out "${V_GENOME_DB}"/"${V_GENOME_DB}" \
  -dbtype nucl -parse_seqids
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>Building a new DB, current time: 09/18/2026 16:21:35
New DB name:   /90daydata/.../database_prep/VviniferaGenomeDB/VviniferaGenomeDB
New DB title:  VviniferaGenome.fasta
Sequence type: Nucleotide
Keep MBits: T
Maximum file size: 3000000000B
Adding sequences from FASTA; added 21 sequences in 2.69122 seconds.

real    0m3.595s
</small></pre>
</details>

</details>
{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_1 %}


### Protein databases

Build databases from the two predicted proteomes:

```bash
makeblastdb \
  -in DvitifoliaeProteome.fasta -out DvitifoliaeProteomeDB/DvitifoliaeProteomeDB \
  -dbtype prot -parse_seqids
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>Building a new DB, current time: 09/18/2026 16:28:22
New DB name:   /90daydata/.../database_prep/DvitifoliaeProteomeDB/DvitifoliaeProteomeDB
New DB title:  DvitifoliaeProteome.fasta
Sequence type: Protein
Keep MBits: T
Maximum file size: 3000000000B
Adding sequences from FASTA; added 29510 sequences in 0.816099 seconds.

real    0m1.177s
</small></pre>
</details> 

{% capture exercise_2 %}
Build a database for the second genome: `VviniferaProteome` .

<details markdown="1"><summary>SOLUTION</summary>

```bash
#V_PROTEOME="VviniferaProteome.fasta"   # input FASTA, defined earlier
V_PROTEOME_DB="{{ page.tutorial.vvinifera_proteome_db }}"     # database name, stored in a shell variable

makeblastdb \
  -in "${V_PROTEOME}" -out "${V_PROTEOME_DB}"/"${V_PROTEOME_DB}" \
  -dbtype prot -parse_seqids
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>Building a new DB, current time: 09/18/2026 16:28:55
New DB name:   /90daydata/.../database_prep/VviniferaProteomeDB/VviniferaProteomeDB
New DB title:  VviniferaProteome.fasta
Sequence type: Protein
Keep MBits: T
Maximum file size: 3000000000B
Adding sequences from FASTA; added 40633 sequences in 0.958617 seconds.

real    0m2.214s
</small></pre>
</details>

</details>
{% endcapture %}
{% include alert class="question" title="Exercise" content=exercise_2 %}


### Checkpoint: database files

List the files produced by `makeblastdb`:

<div class="grid-row grid-gap-2"><div class="tablet:grid-col-6" markdown="1">
```bash
# nucleotide db
ls {{ page.tutorial.dvitifoliae_genome_db }}/
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>DvitifoliaeGenomeDB.ndb  DvitifoliaeGenomeDB.nin  
DvitifoliaeGenomeDB.nog  DvitifoliaeGenomeDB.not  
DvitifoliaeGenomeDB.ntf  DvitifoliaeGenomeDB.nhr  
DvitifoliaeGenomeDB.njs  DvitifoliaeGenomeDB.nos  
DvitifoliaeGenomeDB.nsq  DvitifoliaeGenomeDB.nto
</small></pre>
</details>

</div><div class="tablet:grid-col-6" markdown="1">
```bash
# protein db
ls {{ page.tutorial.dvitifoliae_proteome_db }}/
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>DvitifoliaeProteomeDB.pdb  DvitifoliaeProteomeDB.pin  
DvitifoliaeProteomeDB.pog  DvitifoliaeProteomeDB.pot  
DvitifoliaeProteomeDB.ptf  DvitifoliaeProteomeDB.phr  
DvitifoliaeProteomeDB.pjs  DvitifoliaeProteomeDB.pos  
DvitifoliaeProteomeDB.psq  DvitifoliaeProteomeDB.pto
</small></pre>
</details>

</div></div>

*You should see several index files for each of database prefixes.*

{% include alert class="tip" noicon="true" content="If you created a database directly in the current directory without specifying a parent directory, list its index files with:
```bash
cd DvitifoliaeGenomeDB/
find . -maxdepth 1 -type f -name '*DB.*' -print | sort
```
"%}


{% capture db_use %}
The names like `{{ page.tutorial.dvitifoliae_genome_db }}`, `{{ page.tutorial.dvitifoliae_proteome_db }}` are database prefixes, not individual index files. 

{:.no-copy}
```text
# Incorrect
-db "{{ page.tutorial.dvitifoliae_genome_db }}.phr"
```
The value of `-db` param in **blast** tools must always end with the database prefix. Use the prefix alone when the database files are in the current directory, or include the parent directory followed by the prefix:

{:.no-copy}
```text
# Correct
-db "{{ page.tutorial.dvitifoliae_genome_db }}"
-db "{{ page.tutorial.dvitifoliae_genome_db }}/{{ page.tutorial.dvitifoliae_genome_db }}"
```
{% endcapture%}
{% include alert class="highlighted" content=db_use%}

</div>

### Query a database with `blastdbcmd`

{% capture tool_blastdbcmd %}
**blastdbcmd** lets you inspect a BLAST database and retrieve records from it, such as:
  - Database metadata (`-info`)
  - Sequence identifiers
  - Sequence titles and lengths
  - Full sequences or FASTA records


{:.no-copy}
```bash
blastdbcmd -db <database_name> -entry <seq_identifier> -outfmt <output_fields>
```
<details class="padding-x-2 margin-top-0" markdown="1"><summary><i>explanation of options</i></summary>

* `-db` specifies the database path, ending with the database prefix.
* `-entry` selects a sequence identifier; use `all` to retrieve every record.
* `-outfmt` controls the output fields, such as `%a` for the identifier, `%l` for length, `%t` for title, `%s` for sequence, or `%f` for a FASTA record.

*Use `blastdbcmd -h` or `blastdbcmd -help` for more options.*
</details>
{% endcapture %}
{% include alert class="basic" content=tool_blastdbcmd %}

You can use database names directly with `-db` param or create shell variables to avoid repeating long absolute paths:
```bash
D_GENOME_DB="{{ page.tutorial.dvitifoliae_genome_db }}/{{ page.tutorial.dvitifoliae_genome_db }}"   # relative path to a workdir
V_GENOME_DB="{{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}/{{ page.tutorial.vvinifera_genome_db }}/{{ page.tutorial.vvinifera_genome_db }}"
# use databases from your completed exercises or default to the /reference path  
D_PROTEOME_DB="{{ page.tutorial.data_path_1 }}/{{ page.tutorial.dvitifoliae_proteome_db }}/{{ page.tutorial.dvitifoliae_proteome_db }}"
V_PROTEOME_DB="{{ page.tutorial.data_path_2 }}/{{ page.tutorial.vvinifera_proteome_db }}/{{ page.tutorial.vvinifera_proteome_db }}"

# always confirm that defined paths are correct
ls -l "$(dirname "$D_GENOME_DB")"
ls -l "$(dirname "$V_GENOME_DB")"
ls -l "$(dirname "$D_PROTEOME_DB")"
ls -l "$(dirname "$V_PROTEOME_DB")" 
```

<div class="process-list ul" markdown="1">

### Check database metadata

Use `blastdbcmd -info` to confirm the database name, sequence count, total length, and `BLASTDB` version.

```
blastdbcmd -db "${D_PROTEOME_DB}" -info
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>Database: DvitifoliaeProteome.fasta
        29,510 sequences; 17,754,223 total residues
Date: Sep 21, 2026  11:22 AM    Longest sequence: 16,273 residues
BLASTDB Version: 5
Volumes:
        /90daydata/.../database_prep/DvitifoliaeProteomeDB/DvitifoliaeProteomeDB
</small></pre>
</details>

### List database identifiers

In a FASTA header, the sequence identifier is normally the text after `>` and before the first whitespace. List the parsed identifiers in the *D. vitifoliae* protein database:

```bash
blastdbcmd -db "${D_PROTEOME_DB}" -entry all -outfmt '%a' | head -5
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>XP_050519702.1
XP_050519703.1
XP_050519704.1
XP_050519705.1
XP_050519706.1
</small></pre>
</details>

Display identifiers, lengths, and titles:

```bash
blastdbcmd -db "${D_PROTEOME_DB}" -entry all -outfmt '%a\t%l\t%t' | head -5
```
<details class="padding-x-2 margin-top-0 bg-success-lighter"><summary><i>command log</i></summary>

<pre><small>XP_050519702.1\t856\tgamma-aminobutyric acid type B receptor subunit 1-like isoform X5 [Daktulosphaira vitifoliae]
XP_050519703.1\t630\tBardet-Biedl syndrome 2 protein homolog isoform X2 [Daktulosphaira vitifoliae]
XP_050519704.1\t231\tuncharacterized protein LOC126910583 [Daktulosphaira vitifoliae]
XP_050519705.1\t317\tuncharacterized protein LOC126910586 [Daktulosphaira vitifoliae]
XP_050519706.1\t747\tcullin-4B [Daktulosphaira vitifoliae]
</small></pre>
</details>

Useful `blastdbcmd` output fields include:

| Format code | Meaning | Example |
|---|---|---|
| `%a` | Accession or parsed identifier | `XP_050519702.1` |
| `%t` | Sequence title | `gamma-aminobutyric acid type B receptor subunit 1-like isoform X5 [Daktulosphaira vitifoliae]` |
| `%l` | Sequence length | `856` |
| `%s` | Sequence only | Sequence letters (not shown in the log) |
| `%f` | FASTA-formatted record | `>XP_050519702.1` followed by its sequence (not shown in the log) |

### Retrieve query sequences

Confirm that the selected identifiers are present in the corresponding databases:
```bash
PROTEIN_QUERY="{{ page.tutorial.protein_query }}"
blastdbcmd -db "${D_PROTEOME_DB}" -entry "${PROTEIN_QUERY}" -outfmt '%a'
```
<pre><small>XP_050524263.1</small></pre>

```bash
NUCLEOTIDE_QUERY="{{ page.tutorial.nucleotide_query }}"
blastdbcmd -db "${D_GENOME_DB}" -entry "${NUCLEOTIDE_QUERY}" -outfmt '%a'
```
<pre><small>NW_026099572.1</small></pre>

Retrieve one protein in FASTA format:
```bash
blastdbcmd \
  -db "${D_PROTEOME_DB}" \
  -entry "${PROTEIN_QUERY}" \
  -outfmt '%f' \
  -out "${PROTEIN_QUERY}.fasta"
```

To make nucleotide query shorter, retrieve only 10,000 bases of contig `{{ page.tutorial.nucleotide_query }}`:

```bash
blastdbcmd \
  -db "${D_GENOME_DB}" \
  -entry "${NUCLEOTIDE_QUERY}" \
  -range 72542000-72552000 \
  -outfmt '%f' \
  -out "${NUCLEOTIDE_QUERY}_subset.fasta"
```

{% include alert class="highlighted" content="For larger `blastdbcmd` queries, submit a SLURM job with an explicit CPU and memory request rather than relying on an interactive session." %}



</div>

#### Continue to blast search

The databases and query FASTA files are ready. Continue with [Search Sequence Similarity with BLAST](/bioinformatics/sequence-search/similarity/blast#tutorial-steps) tutorial to run different blast tools and interpret results.


### Rebuild a database after changing its FASTA file

[OPTIONAL]  
BLAST databases do not update automatically when the source FASTA file changes. After adding, removing, or editing sequences, run `makeblastdb` on the new FASTA file. Using a new prefix will preserve the previous database.

```bash
UPDATED_PROTEOME="DvitifoliaeProteome_updated.fasta"
UPDATED_DB="DvitifoliaeProteome_updatedDB"

makeblastdb \
  -in "${UPDATED_PROTEOME}" \
  -dbtype prot \
  -parse_seqids \
  -out "${UPDATED_DB}"

blastdbcmd \
  -db "${UPDATED_DB}" \
  -info
```

</div>


## Summary

This tutorial prepared local BLAST databases from NCBI FASTA files. It covered selecting the nucleotide or protein database type, building and verifying indexed databases with `makeblastdb` and `blastdbcmd`, preserving sequence identifiers, and retrieving query sequences for follow-up BLAST searches.

<div class="usa-accordion " >
{% include accordion title="Quick Quiz: Check your understanding" controls="quiz-database-prep" expanded=false class="question" icon=true %}
<div id="quiz-database-prep" class="accordion_content" markdown='1'>
Check your understanding of BLAST database types, prefixes, identifiers, metadata, and rebuilding.

{% include question qid="1,2,3,4,5,6" %}
</div>
</div>
