---
title: "Creating Local Sequence Databases for BLAST"
description: "Learn how to inspect nucleotide and protein FASTA files, build and verify custom BLAST databases, retrieve sequences, and run test searches."
author: Rick Masonbrink
type: interactive tutorial
order: 2
tags: [BLAST, command line]

objectives:
  - Distinguish between nucleotide and protein BLAST databases.
  - Build and verify custom databases with `makeblastdb` and `blastdbcmd`.
  - Preserve sequence identifiers and retrieve sequences from a database.
  - Select and run an appropriate BLAST search against a custom database.

applications:
  - Searching newly assembled or unpublished sequences.
  - Maintaining organism-specific reference collections.
  - Reusing curated databases in automated sequence-search workflows.

overview: [objectives, applications]

questions:
  - question: "Which `-dbtype` value should be used for amino acid sequences?"
    qid: 1
    answers:
      - nucl
      - prot
    answer: 2
    solution: "Protein sequences require `-dbtype prot`."

  - question: "Which `-dbtype` value should be used for genomic DNA?"
    qid: 2
    answers:
      - nucl
      - prot
    answer: 1
    solution: "DNA and RNA sequence databases require `-dbtype nucl`."

  - question: "Why is `-parse_seqids` useful?"
    qid: 3
    solution: "It stores parsed sequence identifiers in the database, allowing records to be retrieved later by accession or identifier with `blastdbcmd`."

  - question: "What should be supplied to the BLAST `-db` option?"
    qid: 4
    solution: "Supply the database prefix given to `makeblastdb -out`, not the name of an individual index file."

  - question: "How can you inspect the type, number of sequences, and total length of a BLAST database?"
    qid: 5
    solution: "Run `blastdbcmd -db DATABASE_PREFIX -info`."

  - question: "What must you do after adding, removing, or editing sequences in the source FASTA file?"
    qid: 6
    solution: "Run `makeblastdb` again to rebuild the database. BLAST databases do not update automatically."

updated: 2026-08-11
---

## Overview

BLAST searches can use public databases maintained by the National Center for Biotechnology Information (NCBI) or custom databases built from locally available sequences. A custom database is useful when its sequences are unpublished, organism-specific, newly assembled, or curated for a particular project.

In this tutorial, you will download two reference genomes and their predicted proteomes, build nucleotide and protein BLAST databases, inspect and retrieve database records, and run three types of sequence-similarity search.

{% include overviews %}

## Prerequisites

Before beginning, you should be comfortable with:

* navigating directories at the command line;
* viewing text files and running basic Linux commands;
* recognizing FASTA-formatted nucleotide and protein sequences; and
* using software modules on SCINet.

This tutorial downloads two genomes and two predicted proteomes. Make sure your working location has adequate temporary storage. Record the download sizes and software version when documenting an analysis that must be reproduced later.

## Getting Started

{% include setup/shell %}

{% include setup/90daydata %}

{% include setup/mkdir dir="blast_database_tutorial" %}

### Load NCBI BLAST+

Load the NCBI BLAST+ command-line tools and record the version available in your session:

```bash
module load blast+
makeblastdb -version
blastdbcmd -version
```

The reported version may differ from the version originally used to develop this tutorial. For reproducible project work, load a specific module version when one is available and record the version with your results.

### Match BLAST programs to their inputs

The supplied query and database sequence types determine which BLAST program to use. Programs such as `blastx` and `tblastn` perform translation internally; their nucleotide input does not need to be translated in advance.

| Query supplied | Database supplied | Program behavior | Program |
|---|---|---|---|
| Nucleotide | Nucleotide | Compares nucleotide sequences directly | `blastn` |
| Protein | Protein | Compares protein sequences directly | `blastp` |
| Nucleotide | Protein | Translates the query | `blastx` |
| Protein | Nucleotide | Translates database sequences | `tblastn` |
| Nucleotide | Nucleotide | Translates both query and database sequences | `tblastx` |

The database supplied to `blastn`, `tblastn`, or `tblastx` is built with `-dbtype nucl`. The database supplied to `blastp` or `blastx` is built with `-dbtype prot`.

## Tutorial datasets

This tutorial uses versioned genome assemblies from NCBI:

| Organism | Assembly accession | Files used |
|---|---|---|
| *Daktulosphaira vitifoliae* | [GCF_025091365.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_025091365.1/) | Genome and predicted proteome |
| *Vitis vinifera* | [GCF_030704535.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_030704535.1/) | Genome and predicted proteome |

A protein FASTA record contains an identifier line beginning with `>` followed by one or more lines of amino acid sequence:

{:.no-copy}
```text
>protein_001
MSTNPKPQRKTKRNTNRRPQDVKFPGGGQIVGGVLTALA...
```

A nucleotide FASTA record has the same structure but contains nucleotide sequence:

{:.no-copy}
```text
>scaffold_001
ATGCGTACGTAGCTAGCTAGCTAGCTAGCTAGCTAGC...
```

## 1. Download the FASTA files

The `-c` option allows `wget` to continue a partially completed download:

```bash
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/025/091/365/GCF_025091365.1_ASM2509136v1/GCF_025091365.1_ASM2509136v1_genomic.fna.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/025/091/365/GCF_025091365.1_ASM2509136v1/GCF_025091365.1_ASM2509136v1_protein.faa.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/030/704/535/GCF_030704535.1_ASM3070453v1/GCF_030704535.1_ASM3070453v1_genomic.fna.gz
wget -c https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/030/704/535/GCF_030704535.1_ASM3070453v1/GCF_030704535.1_ASM3070453v1_protein.faa.gz
```

Extract and rename the files:

```bash
gunzip GCF_025091365.1_ASM2509136v1_genomic.fna.gz
gunzip GCF_025091365.1_ASM2509136v1_protein.faa.gz
gunzip GCF_030704535.1_ASM3070453v1_genomic.fna.gz
gunzip GCF_030704535.1_ASM3070453v1_protein.faa.gz

mv GCF_025091365.1_ASM2509136v1_genomic.fna DvitifoliaeGenome.fasta
mv GCF_025091365.1_ASM2509136v1_protein.faa DvitifoliaeProteome.fasta
mv GCF_030704535.1_ASM3070453v1_genomic.fna VviniferaGenome.fasta
mv GCF_030704535.1_ASM3070453v1_protein.faa VviniferaProteome.fasta
```

## 2. Inspect the input FASTA files

Before building a database, review the sequence type, record structure, and number of records in each input file.

Inspect the beginning of one protein and one nucleotide file:

```bash
head VviniferaProteome.fasta
head DvitifoliaeGenome.fasta
```

Count the FASTA identifiers in all four files:

```bash
grep -c ">" *fasta
```

Each count represents the number of FASTA records in that file.

## 3. Choose the database type

Every BLAST database is built as either a protein or nucleotide database:

| Input sequences | `makeblastdb` option | Programs that can search the database |
|---|---|---|
| Protein | `-dbtype prot` | `blastp`, `blastx` |
| Nucleotide | `-dbtype nucl` | `blastn`, `tblastn`, `tblastx` |

The database type must match the sequences in the input FASTA file. It cannot be changed after the database is built.

## 4. Create nucleotide databases

Build databases from the two genome FASTA files:

```bash
makeblastdb \
  -in DvitifoliaeGenome.fasta \
  -dbtype nucl \
  -parse_seqids \
  -out DvitifoliaeGenomeDB

makeblastdb \
  -in VviniferaGenome.fasta \
  -dbtype nucl \
  -parse_seqids \
  -out VviniferaGenomeDB
```

The options have the following roles:

* `-in` identifies the input FASTA file.
* `-dbtype nucl` creates a nucleotide database.
* `-parse_seqids` stores FASTA identifiers for later extraction.
* `-out` sets the database prefix.

## 5. Create protein databases

Build databases from the two predicted proteomes:

```bash
makeblastdb \
  -in DvitifoliaeProteome.fasta \
  -dbtype prot \
  -parse_seqids \
  -out DvitifoliaeProteomeDB

makeblastdb \
  -in VviniferaProteome.fasta \
  -dbtype prot \
  -parse_seqids \
  -out VviniferaProteomeDB
```

### Checkpoint: database files

List the files produced by `makeblastdb`:

```bash
find . -maxdepth 1 -type f -name '*DB.*' -print | sort
```

You should see several index files for each of these four prefixes:

{:.no-copy}
```text
DvitifoliaeGenomeDB
DvitifoliaeProteomeDB
VviniferaGenomeDB
VviniferaProteomeDB
```

The displayed names above are prefixes rather than individual files. Always give BLAST the prefix:

{:.no-copy}
```text
# Correct
-db DvitifoliaeProteomeDB

# Incorrect
-db DvitifoliaeProteomeDB.phr
```

## 6. List database identifiers

In a FASTA header, the sequence identifier is normally the text after `>` and before the first whitespace. List the parsed identifiers in the *D. vitifoliae* protein database:

```bash
blastdbcmd \
  -db DvitifoliaeProteomeDB \
  -entry all \
  -outfmt '%a' \
  | head
```

Display identifiers, lengths, and titles:

```bash
blastdbcmd \
  -db DvitifoliaeProteomeDB \
  -entry all \
  -outfmt '%a\t%l\t%t' \
  | head
```

Useful `blastdbcmd` output fields include:

| Format code | Meaning |
|---|---|
| `%a` | Accession or parsed identifier |
| `%t` | Sequence title |
| `%l` | Sequence length |
| `%s` | Sequence only |
| `%f` | FASTA-formatted record |

## 7. Retrieve query sequences

Retrieve one protein in FASTA format:

```bash
blastdbcmd \
  -db DvitifoliaeProteomeDB \
  -entry XP_050524263.1 \
  -outfmt '%f' \
  -out XP_050524263.1.fasta
```

For a shorter nucleotide example, retrieve the first 10,000 bases of contig `NW_026099572.1`:

```bash
blastdbcmd \
  -db DvitifoliaeGenomeDB \
  -entry NW_026099572.1 \
  -range 1-10000 \
  -outfmt '%f' \
  -out NW_026099572.1_1-10000.fasta
```

Verify that both query files contain a FASTA identifier and sequence data:

```bash
head XP_050524263.1.fasta
head NW_026099572.1_1-10000.fasta
```

For larger searches, submit a SLURM job with an explicit CPU and memory request rather than relying on an interactive session.

## 8. Search a genome with `tblastn`

Use the extracted *D. vitifoliae* protein as a query against the translated *V. vinifera* genome database:

```bash
tblastn \
  -query XP_050524263.1.fasta \
  -db VviniferaGenomeDB \
  -out XP_050524263.1_VviniferaGenomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "$THREADS"
```

Inspect the output:

```bash
head XP_050524263.1_VviniferaGenomeDB_hits.tsv
wc -l XP_050524263.1_VviniferaGenomeDB_hits.tsv
```

An empty file means that no alignment passed the selected reporting criteria. A nonempty file contains one tab-delimited alignment per row.

## 9. Search a proteome with `blastp`

Search for similar proteins in the *V. vinifera* predicted proteome:

```bash
blastp \
  -query XP_050524263.1.fasta \
  -db VviniferaProteomeDB \
  -out XP_050524263.1_VviniferaProteomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "$THREADS"
```

Inspect the results:

```bash
head XP_050524263.1_VviniferaProteomeDB_hits.tsv
wc -l XP_050524263.1_VviniferaProteomeDB_hits.tsv
```

`-max_target_seqs 10` limits the number of reported target sequences per query. Evaluate alignments by interpreting the E-value together with percent identity, alignment length, query coverage, and biological context.

## 10. Search nucleotide sequences with `blastn`

Search the extracted 10-kb *D. vitifoliae* region against the *V. vinifera* genome database:

```bash
blastn \
  -query NW_026099572.1_1-10000.fasta \
  -db VviniferaGenomeDB \
  -out NW_026099572.1_1-10000_VviniferaGenomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "$THREADS"
```

Inspect the result count and first alignments:

```bash
wc -l NW_026099572.1_1-10000_VviniferaGenomeDB_hits.tsv
head NW_026099572.1_1-10000_VviniferaGenomeDB_hits.tsv
```

The output columns used in all three searches are:

| Column | Meaning |
|---|---|
| `qseqid` | Query identifier |
| `sseqid` | Subject or database identifier |
| `pident` | Percentage of identical positions |
| `length` | Alignment length |
| `qstart`, `qend` | Query alignment coordinates |
| `sstart`, `send` | Subject alignment coordinates |
| `evalue` | Number of similarly strong matches expected by chance |
| `bitscore` | Normalized alignment score |
| `qcovs` | Query coverage per subject |
| `stitle` | Subject title |

## 11. Rebuild a database after changing its FASTA file

BLAST databases do not update automatically when the source FASTA file changes. After adding, removing, or editing sequences, run `makeblastdb` again. Using a new prefix will preserve the previous database.

```bash
makeblastdb \
  -in DvitifoliaeProteome_updated.fasta \
  -dbtype prot \
  -parse_seqids \
  -out DvitifoliaeProteome_updatedDB

blastdbcmd \
  -db DvitifoliaeProteome_updatedDB \
  -info
```

## Resource and reproducibility considerations

Before creating or sharing a large database, consider:

* the compressed inputs, uncompressed FASTA files, BLAST index files, and result files all consume storage;
* `/90daydata` is temporary storage and is subject to its retention policy;
* heavily reused project databases may belong in a shared project location;
* Larger searches should use a SLURM job with documented CPU, memory, and time requests;
* the assembly accessions, download date, BLAST+ version, database-building commands, and search parameters should be recorded for publication.

## Summary

In this tutorial, you:

* inspected nucleotide and protein FASTA files;
* selected the correct database type;
* created four custom databases with `makeblastdb`;
* verified database metadata and sequence identifiers with `blastdbcmd`;
* retrieved protein and nucleotide query sequences; and
* ran `tblastn`, `blastp`, and `blastn` searches with reproducible tabular output.

The essential database-building pattern is:

```bash
makeblastdb \
  -in INPUT.fasta \
  -dbtype prot_or_nucl \
  -parse_seqids \
  -out DATABASE_PREFIX
```

Verify the result with:

```bash
blastdbcmd -db DATABASE_PREFIX -info
```

## Knowledge check

{% include quiz qid="1,2,3,4,5,6" %}

## Exercises

### Exercise 1: Build another protein database

Choose a protein FASTA file, build a database, and use `blastdbcmd -info` to verify its type, sequence count, and total sequence length.

### Exercise 2: Retrieve a sequence

List the identifiers in your new database, select one identifier, and retrieve the corresponding FASTA record.

### Exercise 3: Compare related searches

Choose a *D. vitifoliae* protein and search for related sequences in both the *V. vinifera* proteome and genome. Compare the best-supported alignments using E-value, percent identity, alignment length, and query coverage. Does the strongest protein match correspond to a convincing region in the genome search?

## Additional resources

* [NCBI BLAST Command Line Applications User Manual](https://www.ncbi.nlm.nih.gov/books/NBK279690/)
* [SCINet Public Databases workbook](/bioinformatics/resources/databases)
* [SCINet command-line workbook](/computing-skills/command-line/)
* [Getting Started with SCINet Workbooks](/about/use)
* [SCINet storage guide](https://scinet.usda.gov/guides/data/storage)
* [SCINet SLURM guide](https://scinet.usda.gov/guides/use/slurm)
* Run `makeblastdb -help` and `blastdbcmd -help` for locally installed command documentation.
