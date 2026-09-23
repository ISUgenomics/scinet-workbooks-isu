---
title: "Search Sequence Similarity with BLAST"
description: "Use BLAST+ to compare nucleotide and protein queries against custom databases and interpret the resulting alignments."
svg: /genomics.svg
author: [Rick Masonbrink, Aleksandra Badaczewska]
language: Bash

header:
  overlay_image: 07-wrangling/assets/img/07_data_acquisition_banner.png

index: 
type: interactive tutorial
order: 3
wg: Bioinformatics
tags: [BLAST, sequence similarity, command line]

related:
  - "[Create local sequence databases for BLAST](/bioinformatics/sequence-search/similarity/database_prep2)"
  - "[Sequence search and similarity](/bioinformatics/sequence-search/similarity/)"
  - "[Bioinformatics resources](/bioinformatics/resources/)"

references:
  - "[NCBI BLAST+ documentation](https://www.ncbi.nlm.nih.gov/books/NBK279690/)"

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
  - Match query and database molecule types to the appropriate BLAST program.
  - Run direct and translated searches with `blastp`, `tblastn`, `blastn`, `blastx`, and `tblastx`.
  - Evaluate alignments using E-value, identity, alignment length, query coverage, and annotation.
  - Compare complementary search hits and integrate the results.
  - Record reproducible search settings and scale searches with SLURM.

applications:
  - Finding homologous proteins or conserved genomic regions.
  - Comparing protein and nucleotide sequences across related organisms.
  - Supporting candidate-gene and coding-region identification.
  - Automating repeatable similarity searches against project-specific databases.

questions:
  - question: "Which BLAST program compares a protein query with a protein database?"
    title: "Direct protein search"
    qid: "1"
    answers:
      - blastp
      - tblastn
      - blastx
    answer: 1
    solution: "`blastp` compares a protein query directly with a protein database."

  - question: "You have a protein query and a nucleotide database that must be translated before comparison. Which BLAST program should you use?"
    title: "Protein query against translated DNA"
    qid: "2"
    answers:
      - blastp
      - tblastn
      - blastn
    answer: 2
    solution: "`tblastn` translates the nucleotide database and compares it with the protein query."

  - question: "To find protein matches from a nucleotide query, which BLAST program translates the query first?"
    title: "Translated nucleotide query"
    qid: "3"
    answers:
      - blastn
      - blastx
      - tblastx
    answer: 2
    solution: "`blastx` translates the nucleotide query and compares it with a protein database."

  - question: "Both the query and database are nucleotide sequences, but the comparison should occur at the protein level. Which BLAST program fits this search?"
    title: "Translated nucleotide search"
    qid: "4"
    answers:
      - blastn
      - tblastn
      - tblastx
    answer: 3
    solution: "`tblastx` translates both nucleotide sequences in six reading frames before comparison."

  - question: "A hit has a strong E-value. What else should be considered during interpretation?"
    title: "Alignment interpretation"
    qid: "5"
    answers:
      - Percent identity, alignment length, query coverage, and annotation.
      - Only the number of database hits.
      - Only the E-value and the database name.
    answer: 1
    solution: "Interpret the E-value together with percent identity, alignment length, query coverage, and biological context."

  - question: "When can protein-query and nucleotide-query results be compared biologically?"
    title: "Integrating search results"
    qid: "6"
    answers:
      - When they represent corresponding sequences, such as a protein and the DNA region that encodes it.
      - Always, because E-values and query coverage have the same meaning in every BLAST search.
      - Never, because protein and nucleotide searches cannot provide related evidence.
    answer: 1
    solution: "Protein and nucleotide searches can support the same biological interpretation when the queries have a translation-based relationship. Their raw E-values, percent identity, alignment length, and query coverage cannot be compared directly. Use the alignments, coordinates, and annotations to assess that correspondence."

updated: 2026-09-21

tutorial:
  root_practice: /90daydata/shared/$USER
  workdir: tutorials/sequence_search/blast_search
  database_workdir: tutorials/sequence_search/database_prep
  d_genome_path: /reference/workbook/bioinformatics/dataset/ref_genome/grape_phylloxera
  v_genome_path: /reference/workbook/bioinformatics/dataset/ref_genome/grape_wine
  d_proteome_path: /reference/workbook/bioinformatics/dataset/ref_proteome/grape_phylloxera
  v_proteome_path: /reference/workbook/bioinformatics/dataset/ref_proteome/grape_wine
  d_genome_db: DvitifoliaeGenomeDB/DvitifoliaeGenomeDB
  d_proteome_db: DvitifoliaeProteomeDB/DvitifoliaeProteomeDB
  v_genome_db: VviniferaGenomeDB/VviniferaGenomeDB
  v_proteome_db: VviniferaProteomeDB/VviniferaProteomeDB
  protein_query: XP_050524263.1
  nucleotide_query: NW_026099572.1
  threads: 4
  evalue: 1e-5
  max_target_seqs: 10

terms:
  - term: query sequence
    definition: The sequence supplied to BLAST for comparison against a database.
  - term: subject sequence
    definition: A sequence in the database being compared with the query.
  - term: nucleotide database
    definition: A BLAST database containing DNA or RNA sequences.
  - term: protein database
    definition: A BLAST database containing amino acid sequences.
  - term: alignment
    definition: A comparison of two sequences that identifies matching or similar regions.
  - term: translated search
    definition: A search that translates nucleotide sequences into amino acid sequences before comparison.
  - term: E-value
    definition: The expected number of equally strong matches found by chance.
  - term: query coverage
    definition: The percentage of the query sequence included in an alignment.
  - term: HSP
    definition: A high-scoring local alignment between a query and database sequence.
  - term: ortholog
    definition: A corresponding gene in different species inherited from a common ancestor.

overview: [objectives, applications, terminology]
---

## Overview

BLAST (Basic Local Alignment Search Tool) identifies local similarities between a query sequence and sequences in a nucleotide or protein database. The search program must match the molecule type of the query and database. To assess whether an alignment hit is biologically meaningful, consider its E-value, percent identity, alignment length, query coverage, and whether its annotation fits the expected function.

In this tutorial, you will use one *Daktulosphaira vitifoliae* protein query and nucleotide query to run direct (`blastp`, `tblastn`, `blastn`) and translated (`blastx`, `tblastx`) searches against *Vitis vinifera* protein and nucleotide databases.


{% include overviews %}

## Getting Started

{% include segment/getting_started time="02:00:00" cpus_per_task="4" mem="8G" %}

**Tutorial Steps:**
1. [Prepare the practice workspace](#prepare-the-practice-workspace).
1. [Get the dataset](#get-the-dataset).
1. [Load NCBI BLAST+ tools](#load-ncbi-blast-tools).
1. [Match BLAST programs to their inputs](#match-blast-programs-to-their-inputs).
1. [Run the example BLAST searches](#run-the-example-blast-searches).
- [Search protein in proteome with `blastp`](#search-protein-in-proteome-with-blastp).
- [Search protein in genome with `tblastn`](#search-protein-in-genome-with-tblastn).
- [Search nucleotide in genome with `blastn`](#search-nucleotide-in-genome-with-blastn).
- [Search translated nucleotide in proteome with `blastx`](#search-translated-nucleotide-in-proteome-with-blastx).
- [Search translated nucleotide in translated genome with `tblastx`](#search-translated-nucleotide-in-translated-genome-with-tblastx).
- [Integrate the search results](#integrate-results).
1. [Scale up with SLURM](#scale-up-with-slurm-to-automate-the-task).


## Tutorial Steps

<div class="process-list" markdown="1">


### Prepare the practice workspace

{% include setup/practice_workspace workdir=page.tutorial.workdir %}


### Get the dataset

This tutorial uses two genome assemblies from NCBI:

| Organism | Assembly accession | Files used | File format | Size |
|---|---|---|---|---|
| *Daktulosphaira vitifoliae* | [GCF_025091365.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_025091365.1/) | Genome and predicted proteome | `FASTA` | 89M |
| *Vitis vinifera* | [GCF_030704535.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_030704535.1/) | Genome and predicted proteome | `FASTA` | 152M |

The following steps are natural continuation of the [Create Local Sequence Databases for BLAST](./database_prep2.md) tutorial. If you completed it, use the databases and query FASTA files already created in your workspace. Define shell variables for easier, error-prone path management:
```bash
DATASET={{ page.tutorial.root_practice }}/tutorials/sequence_search/database_prep

D_GENOME_DB="${DATASET}"/{{ page.tutorial.d_genome_db }}
D_PROTEOME_DB="${DATASET}"/{{ page.tutorial.d_proteome_db }}
V_GENOME_DB="${DATASET}"/{{ page.tutorial.v_genome_db }}
V_PROTEOME_DB="${DATASET}"/{{ page.tutorial.v_proteome_db }}

PROTEIN_QUERY="${DATASET}"/{{ page.tutorial.protein_query }}.fasta
NUCLEOTIDE_QUERY="${DATASET}"/{{ page.tutorial.nucleotide_query }}_subset.fasta
```

 

**If you have not completed the database-preparation tutorial**, use the the same files shared at `/reference` path:
```text
D_GENOME_DB={{ page.tutorial.d_genome_path }}/{{ page.tutorial.d_genome_db }}
D_PROTEOME_DB={{ page.tutorial.d_proteome_path }}/{{ page.tutorial.d_proteome_db }}
V_GENOME_DB={{ page.tutorial.v_genome_path }}/{{ page.tutorial.v_genome_db }}
V_PROTEOME_DB={{ page.tutorial.v_proteome_path }}/{{ page.tutorial.v_proteome_db }}

PROTEIN_QUERY={{ page.tutorial.d_proteome_path }}/{{ page.tutorial.protein_query }}.fasta
NUCLEOTIDE_QUERY={{ page.tutorial.d_genome_path }}/{{ page.tutorial.nucleotide_query }}_subset.fasta
```

- `PROTEIN_QUERY` is a FASTA file containing one *D. vitifoliae* protein sequence. 
- `NUCLEOTIDE_QUERY` is a FASTA file containing a 10-kb *D. vitifoliae* genomic region.

Verify that both query files contain a FASTA identifier and sequence data:
```bash
head "${PROTEIN_QUERY}"
head "${NUCLEOTIDE_QUERY}"
```
<div class="grid-row grid-gap-2"><div class="tablet:grid-col-6" markdown="1">
**protein query**
<pre><small>>XP_050524263.1 uncharacterized protein LOC126895952 [Daktulosphaira vitifoliae]
MQIFVKTLTGKTITLEVESSDSIENVKSKIQDKEGIPPDQQRLIFAGKQLEDGRTLSDYNIQKESTLHLVLRLRGGAKKR
KKKNYSTPKKIKHKKRKVKLAVLKFYKVDENGKVIRLKKECTSENCGPGVFMADMSNRHYCGKCSATVPK
</small></pre>

</div><div class="tablet:grid-col-6" markdown="1">
**nucleotide query**
<pre><small>>NW_026099572.1:72542000-72552000 Daktulosphaira vitifoliae isolate Bord-2020 unplaced genomic scaffold, ASM2509136v1 Scaffold_1, whole genome shotgun sequence
CTCAAATACTCAGCATTCCAAATCTGAGTGTTGAAACTTTTTTTAGATGTTGAATATGTCAATATGAGTTCATTTTTGTC
AATAAATGCCTTTAAAAATATAAGTATGAATAACTATGCTATGTACACATATTTTTCACTTACGATGTTGAATATTTCTT
TCTTTTTGATAACATTGGTAAGGATTTCCATCGAGGATTTATAAAAAAATTCACCATTTATTCCTTGAAGTTAATGCGAT
ACTCTTCGGGCTGTTCAAGATTCTTGTCATTTGCTCTGGTAATTGTTAAATTTTGTAAGTAGTCATAATATTTTATGAAA
ATCTGATCTCCTCTCTCTAACGGTGCCATAATTATTTCGCCAATAAATTCGACTTGCGAACGTCATGTTGATTTCAAGTT
TTTGTAATTTGATTTCGACGCGACTGAGACTTCTAGACTAGTTTGACTTCCACAAGAGCTTATATGTTCTTCGTTCCCTG
GTTAGACTAAAAGTCCCAATGCATTTTTAGTTTACTAGATCTTTTATACTAATTTTTTATCGATTTTTTGTCCAACAACA
AATAACGTTGCTAAAACAAATTTTCATTATTTCAAAAATAAATATGTTATTTAACGATCACCATAATAGATTATTTTCAG
CCAATATTTTAATTATTTTTTTCCAGTTATACAAAATGAAAAAATTTGATTTAGTAACGTTATTTGTCGTTGGCCAAAAA
</small></pre>

</div></div>


### Load NCBI BLAST+ tools

{% include tool_summary tool_key="blast" %}

{% include setup/module tool_key="blast" known="true" %}
{% include setup/tool_verify tool_key="blast" exec="blastn" version_cmd="blastn -version" help_cmd="blastn -help" %}


### Match BLAST programs to their inputs

The supplied query and database sequence types determine which BLAST program to use. 

| Query supplied | Database supplied | Program behavior | Program |
|---|---|---|---|
| Nucleotide | Nucleotide | Compares nucleotide sequences directly | `blastn` |
| Protein | Protein | Compares protein sequences directly | `blastp` |
| Nucleotide | Protein | Translates the query | `blastx` |
| Protein | Nucleotide | Translates database sequences | `tblastn` |
| Nucleotide | Nucleotide | Translates both query and database sequences | `tblastx` |

- Programs such as `blastx` and `tblastn` perform translation internally; their nucleotide input does not need to be translated in advance.


### Run the example BLAST searches

All BLAST programs compare a query sequence with a selected database and report local sequence alignments. The program changes with the query and database molecule types, but the main input, search, and output options are shared.

<div class="usa-accordion" data-allow-multiple>
{% include accordion title="INPUT" controls="blast-input" class="outline" icon=false %}
<div id="blast-input" class="accordion_content" markdown="1" hidden>

- a query FASTA file (`-query`) 
- a BLAST database prefix (`-db`)

*Their molecule types must match the [selected BLAST program](#match-blast-programs-to-their-inputs).*

</div>

{% include accordion title="SYNTAX" controls="blast-syntax" class="outline" icon=false %}
<div id="blast-syntax" class="accordion_content" markdown="1" hidden>

{:.no-copy}
```bash
<blast_program> \
  -query <query_file> \ 
  -db <database_prefix> \
  -out <output_file> \
  -outfmt <output_fields> \
  [options]
```

</div>

{% include accordion title="OPTIONS" controls="blast-options" class="outline" icon=false %}
<div id="blast-options" class="accordion_content" markdown="1" hidden>

* `-query` supplies the query FASTA file.
* `-db` selects the database by its prefix, not an individual index file.
* `-out` names the result file.
* `-outfmt '6 ...'` writes tabular output with the selected fields.
* `-evalue` sets the reporting threshold for matches.
* `-max_target_seqs` limits the number of reported targets per query.
* `-num_threads` sets the number of CPUs used by the search.

</div>

{% include accordion title="OUTPUT" controls="blast-output" class="outline" icon=false %}
<div id="blast-output" class="accordion_content" markdown="1" hidden>

Output is typically tab-separated alignment results. 
- A nonempty file contains one tab-delimited alignment per row. 
- An empty file means that no alignment passed the selected reporting criteria. 

Typical columns include:

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

{% include alert class="highlighted" content="Evaluate each result using E-value together with percent identity, alignment length, query coverage, and biological context." %}

</div>

{% include accordion title="PARALLELIZATION" controls="blast-parallelization" class="outline" icon=false %}
<div id="blast-parallelization" class="accordion_content" markdown="1" hidden>

`-num_threads` controls how many CPU threads BLAST uses for one search. When working on HPC (interactive session or SLURM job), match `-num_threads` to the requested `--cpus-per-task`, and store this value in the `THREADS` shell variable so it can be reused across BLAST tools.

```bash
THREADS=4     # match with your requested --cpus-per-task 
```

{% include alert class="tip" content="Common practice is:
  - 1–4 CPUs for small searches or a few queries.
  - 4–16 CPUs for typical local BLAST searches.
  - 16–32+ CPUs only for large multi-query workloads, after benchmarking. 

Performance depends on query count, database size, and search type.  
Always stay within the CPUs allocated to the job." %}

</div>
</div>

Below, BLAST tools examples move from direct comparisons to increasingly complex translated searches:
  1. [blastp](#search-protein-in-proteome-with-blastp): <small>`protein query → proteome database`</small>, identifies protein-level homologs in the proteome
  2. [tblastn](#search-protein-in-genome-with-tblastn): <small>`protein query → genome database`</small>, tests whether the protein also matches a genomic region
  3. [blastn](#search-nucleotide-in-genome-with-blastn): <small>`nucleotide query → genome database`</small>, tests direct nucleotide similarity
  4. [blastx](#search-translated-nucleotide-in-proteome-with-blastx): <small>`translated nucleotide query → proteome database`</small>, useful when annotations are incomplete
  5. [tblastx](#search-translated-nucleotide-in-translated-genome-with-tblastx): <small>`translated nucleotide query → translated genome database`</small>

<div class="process-list ul" markdown="1">

### Search protein in proteome with `blastp`

`protein query` → `proteome database`

Search for the *D. vitifoliae* protein `{{ page.tutorial.protein_query }}` against the *V. vinifera* predicted proteome:
```bash
blastp \
  -query "${PROTEIN_QUERY}" \
  -db "${V_PROTEOME_DB}" \
  -out XP_050524263.1_VviniferaProteomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "${THREADS}"
```
<pre><small>real    0m0.382s
user    0m0.301s
sys     0m0.005s
</small></pre>

Inspect the results:
```bash
wc -l XP_050524263.1_VviniferaProteomeDB_hits.tsv
head -5 XP_050524263.1_VviniferaProteomeDB_hits.tsv
```
<pre><small>24 XP_050524263.1_VviniferaProteomeDB_hits.tsv

XP_050524263.1  ref|XP_002279878.1|     82.993  147     1       147     1       147     1.68e-72        214     98      ubiquitin-ribosomal protein eS31 fusion protein [Vitis vinifera]
XP_050524263.1  ref|XP_003634320.1|     82.993  147     1       147     1       147     1.79e-72        214     98      ubiquitin-ribosomal protein eS31 fusion protein [Vitis vinifera]
XP_050524263.1  ref|XP_002270170.1|     82.313  147     1       147     1       147     5.85e-72        213     98      ubiquitin-ribosomal protein eS31 fusion protein [Vitis vinifera]
XP_050524263.1  ref|XP_010654111.1|     94.737  76      1       76      1       76      1.16e-47        150     51      ubiquitin-ribosomal protein eL40 fusion protein isoform X2 [Vitis vinifera]
XP_050524263.1  ref|XP_002273568.1|     94.737  76      1       76      1       76      1.33e-47        150     51      ubiquitin-ribosomal protein eL40z fusion protein [Vitis vinifera]
</small></pre>

**Interpretation:** *The search returned `24` protein alignments. The top hits cover `98%` of the query with `82.993%` identity and E-values near `1e-72`, supporting a strong homologous protein match. Some shorter hits have high identity but only `51%` query coverage, so they should be treated as partial matches rather than equivalent full-length homologs.*


### Search protein in genome with `tblastn`

`protein query` → `genome database`

Use the extracted *D. vitifoliae* protein `{{ page.tutorial.protein_query }}` as a query against the *V. vinifera* genome database. `tblastn` translates the nucleotide database in six reading frames during the search:

```bash
tblastn \
  -query "${PROTEIN_QUERY}" \
  -db "${V_GENOME_DB}" \
  -out XP_050524263.1_VviniferaGenomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "${THREADS}"
```
<pre><small>real    0m1.771s
user    0m5.663s
sys     0m0.225s
</small></pre>

Inspect the output:
```bash
wc -l XP_050524263.1_VviniferaGenomeDB_hits.tsv
head -5 XP_050524263.1_VviniferaGenomeDB_hits.tsv
```

<pre><small>50 XP_050524263.1_VviniferaGenomeDB_hits.tsv

XP_050524263.1  ref|NC_081811.1|        82.993  147     1       147     4935781 4936221 3.86e-54        186     98      Vitis vinifera cultivar Pinot Noir 40024 chromosome 7, ASM3070453v1
XP_050524263.1  ref|NC_081821.1|        82.993  147     1       147     15992374        15992814        5.21e-54        186     98      Vitis vinifera cultivar Pinot Noir 40024 chromosome 17, ASM3070453v1
XP_050524263.1  ref|NC_081812.1|        82.313  147     1       147     4604097 4604537 1.51e-53        184     98      Vitis vinifera cultivar Pinot Noir 40024 chromosome 8, ASM3070453v1
XP_050524263.1  ref|NC_081812.1|        88.889  36      1       36      18259341        18259448        2.66e-13        69.3    98      Vitis vinifera cultivar Pinot Noir 40024 chromosome 8, ASM3070453v1
XP_050524263.1  ref|NC_081812.1|        96.552  29      35      63      18259689        18259775        3.84e-12        66.2    98      Vitis vinifera cultivar Pinot Noir 40024 chromosome 8, ASM3070453v1
</small></pre>

**Interpretation:** *The search returned `50` reported alignments. The strongest hit covers `98%` of the `147`-aa query with `82.993%` identity and an E-value of `3.86e-54`, supporting a strong protein-to-genome match. Repeated subject accessions and shorter alignments indicate additional matches or partial regions, so the count does not necessarily represent `50` distinct genes or loci.*


### Search nucleotide in genome with `blastn`

`nucleotide query` → `genome database`

Search the extracted 10-kb *D. vitifoliae* nucleotide region `{{ page.tutorial.nucleotide_query }}` against the *V. vinifera* genome database:
```bash
blastn \
  -query "${NUCLEOTIDE_QUERY}" \
  -db "${V_GENOME_DB}" \
  -out NW_026099572.1_subset_VviniferaGenomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "${THREADS}"
```
<pre><small>real    0m0.292s
user    0m0.232s
sys     0m0.033s
</small></pre>

Inspect the result count and first alignments:
```bash
wc -l NW_026099572.1_subset_VviniferaGenomeDB_hits.tsv
head NW_026099572.1_subset_VviniferaGenomeDB_hits.tsv
```
<pre><small>3 NW_026099572.1_subset_VviniferaGenomeDB_hits.tsv

NW_026099572.1:72542000-72552000        ref|NC_081817.1|        72.742  1196    7599    8773    3103726 3104909 4.18e-100       372     12      Vitis vinifera cultivar Pinot Noir 40024 chromosome 13, ASM3070453v1
NW_026099572.1:72542000-72552000        ref|NC_081818.1|        82.524  103     7599    7701    29753727        29753625        1.31e-15        91.6    1       Vitis vinifera cultivar Pinot Noir 40024 chromosome 14, ASM3070453v1
NW_026099572.1:72542000-72552000        ref|NC_081807.1|        79.208  101     7599    7699    7508492 7508592 1.71e-09        71.3    1       Vitis vinifera cultivar Pinot Noir 40024 chromosome 3, ASM3070453v1
</small></pre>

**Interpretation:** *The search returned `3` alignments. The strongest covers `1,196` bases (`12%` of the `10-kb` query) at `72.742%` identity with an E-value of `4.18e-100`, indicating a significant local match rather than a match to the entire extracted region. The two shorter hits cover only `1%` of the query and provide limited support on their own.*


### Search translated nucleotide in proteome with `blastx`

`nucleotide query` → `proteome database`

Use the extracted *D. vitifoliae* nucleotide region `{{ page.tutorial.nucleotide_query }}` to search the *V. vinifera* predicted proteome. `blastx` translates the nucleotide query in six reading frames before comparing it with protein sequences:

```bash
blastx \
  -query "${NUCLEOTIDE_QUERY}" \
  -db "${V_PROTEOME_DB}" \
  -out NW_026099572.1_subset_VviniferaProteomeDB_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "${THREADS}"
```
<pre><small>real    0m1.504s
user    0m3.541s
sys     0m0.035s
</small></pre>

Inspect the result count and first alignments:

```bash
wc -l NW_026099572.1_subset_VviniferaProteomeDB_hits.tsv
head -5 NW_026099572.1_subset_VviniferaProteomeDB_hits.tsv
```
<pre><small>20 NW_026099572.1_subset_VviniferaProteomeDB_hits.tsv

NW_026099572.1:72542000-72552000        ref|XP_002283532.2|     77.621  496     7520    9001    122     617     0.0     734     19      heat shock cognate 70 kDa protein 2 [Vitis vinifera]
NW_026099572.1:72542000-72552000        ref|XP_002283532.2|     64.539  141     7054    7473    8       128     0.0     183     19      heat shock cognate 70 kDa protein 2 [Vitis vinifera]
NW_026099572.1:72542000-72552000        ref|XP_002284063.1|     77.016  496     7520    9001    122     617     0.0     732     19      heat shock cognate 70 kDa protein 2 [Vitis vinifera]
NW_026099572.1:72542000-72552000        ref|XP_002284063.1|     63.830  141     7054    7473    8       128     0.0     182     19      heat shock cognate 70 kDa protein 2 [Vitis vinifera]
NW_026099572.1:72542000-72552000        ref|XP_002263599.1|     76.815  496     7520    9001    122     617     0.0     730     19      heat shock cognate 70 kDa protein 2 [Vitis vinifera]
</small></pre>

**Interpretation:** *The search returned `20` translated alignments. The strongest matches identify a heat shock cognate 70 kDa protein, covering `19%` of the 10-kb nucleotide query at about `77%` amino-acid identity with an E-value reported as `0.0` (below the display precision). The repeated protein accessions and shorter alignments represent multiple high-scoring segments or related proteins, not necessarily `20` separate genes.*


### Search translated nucleotide in translated genome with `tblastx`

`nucleotide query` → `genome database`

Use the same extracted *D. vitifoliae* nucleotide region against the *V. vinifera* genome. `tblastx` translates both nucleotide sequences in six reading frames before searching for translated similarities:

```bash
tblastx \
  -query "${NUCLEOTIDE_QUERY}" \
  -db "${V_GENOME_DB}" \
  -out NW_026099572.1_subset_VviniferaGenomeDB_tblastx_hits.tsv \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue 1e-5 \
  -max_target_seqs 10 \
  -num_threads "${THREADS}"
```
<pre><small>real    1m6.971s
user    3m56.830s
sys     0m0.409s
</small></pre>

Inspect the result count and first alignments:

```bash
wc -l NW_026099572.1_subset_VviniferaGenomeDB_tblastx_hits.tsv
head -5 NW_026099572.1_subset_VviniferaGenomeDB_tblastx_hits.tsv
```
<pre><small>277 NW_026099572.1_subset_VviniferaGenomeDB_tblastx_hits.tsv

NW_026099572.1:72542000-72552000        ref|NC_081817.1|        76.960  421     7739    9001    3103872 3105134 0.0     698     18      Vitis vinifera cultivar Pinot Noir 40024 chromosome 13, ASM3070453v1
NW_026099572.1:72542000-72552000        ref|NC_081817.1|        70.309  421     7739    9001    2809942 2811204 0.0     630     18      Vitis vinifera cultivar Pinot Noir 40024 chromosome 13, ASM3070453v1
NW_026099572.1:72542000-72552000        ref|NC_081817.1|        90.909  66      7054    7251    3102625 3102822 0.0     148     18      Vitis vinifera cultivar Pinot Noir 40024 chromosome 13, ASM3070453v1
NW_026099572.1:72542000-72552000        ref|NC_081817.1|        83.562  73      7520    7738    3103647 3103865 0.0     145     18      Vitis vinifera cultivar Pinot Noir 40024 chromosome 13, ASM3070453v1
NW_026099572.1:72542000-72552000        ref|NC_081817.1|        77.027  74      7520    7741    2809717 2809938 0.0     136     18      Vitis vinifera cultivar Pinot Noir 40024 chromosome 13, ASM3070453v1
</small></pre>

**Interpretation:** <i>The search returned `277` translated alignments. The strongest alignments cover about `18%` of the nucleotide query at `76.960%` identity with E-values reported as `0.0`, confirming a strong conserved coding-region signal on the *V. vinifera* genome. Because both sequences are translated in six frames, the many reported alignments include multiple frames, HSPs, and nearby genomic matches rather than `277` independent genes.</i>

</div>

### Integrate results

Integrate results by comparing alignments across BLAST searches using E-value, percent identity, alignment length, and query coverage.  

{% include alert class="highlighted" content="Scores from different BLAST tools should not be compared directly because the searches use different input types and search spaces." %}

{% capture exercise_1 %}

**HINT:** Compare results separately for the protein and nucleotide queries; they are separate, unrelated queries with different targets. Consider:
- *Within the protein-query searches, do the proteome and genome results support a strong match?*
- *Within the nucleotide-query searches, do the results identify a consistent conserved region?*
- *Do the strongest hits cover most of their respective queries?*
- *Does the genome search show one strong location or several?*
- *What follow-up checks could clarify the biological relevance of these matches?*

<details class="margin-top-3" markdown="1"><summary>SOLUTION</summary>

Our BLAST searches used two different *D. vitifoliae* query records, so we should interpret them as two related evidence tracks rather than five direct comparisons.

**Protein query: `{{ page.tutorial.protein_query }}`**

| Search | Database | Top result | Coverage | Identity | E-value |
|---|---|---|---:|---:|---:|
| `blastp` | *V. vinifera* proteome | `XP_002279878.1` (eS31 fusion protein) | `98%` | `82.993%` | `1.68e-72` |
| `tblastn` | *V. vinifera* genome | `NC_081811.1` (chromosome 7) | `98%` | `82.993%` | `3.86e-54` |

Both searches cover nearly the full `147`-aa query with strong significance. Together they support a conserved protein match in the *V. vinifera* proteome and genome. The `tblastn` output also contains similarly strong hits on chromosomes 8 and 17, so this query likely matches multiple genomic locations.

**Nucleotide query: `{{ page.tutorial.nucleotide_query }}`** (10-kb region)

| Search | Database | Top result | Coverage | Identity | E-value |
|---|---|---|---:|---:|---:|
| `blastn` | *V. vinifera* genome | `NC_081817.1` (chromosome 13) | `12%` | `72.742%` | `4.18e-100` |
| `blastx` | *V. vinifera* proteome | `XP_002283532.2` (HSP70 protein) | `19%` | `77.621%` | `0.0`* |
| `tblastx` | *V. vinifera* genome | `NC_081817.1` (chromosome 13) | `18%` | `76.960%` | `0.0`* |

{% include alert class="highlighted" content="The `0.0` values mean the E-value is below the displayed precision, not that it is mathematically zero." %}

The three searches identify the same type of signal: a conserved part (`12–19%` query coverage) of the 10-kb nucleotide region, not a match across the complete query. `blastn` and `tblastx` both place their strongest matches on *V. vinifera* chromosome 13, while `blastx` identifies a related HSP70 protein. This agreement supports a conserved coding region in the extracted sequence. A useful follow-up would be to use the alignment coordinates and gene annotations to identify the matching *V. vinifera* gene, then use synteny, reciprocal-best-hit (RBH), or phylogenetic analysis to assess whether the evidence supports orthology.

</details>
{% endcapture %}
{% include alert class="question" title="Exercise: Compare related searches" content=exercise_1 %}


### Scale up with SLURM to automate the task 

{% include segment/cli_to_slurm tool="BLAST+" %}

Create a `blast_search.sh` file:
```bash
nano blast_search.sh
``` 
and copy-paste the script body provided below.  
Edit the account name, query, database, and output variables with the search you want to run.

```bash
#!/bin/bash
#SBATCH --job-name=blast_search        # EDIT: choose a descriptive job name
#SBATCH --partition=<value>            # EDIT: Ceres: ceres; Atlas: atlas
#SBATCH --nodes=1                      # BLAST runs on one node
#SBATCH --ntasks=1                     # run one BLAST process
#SBATCH --cpus-per-task=4              # EDIT: must match -num_threads below
#SBATCH --account=<account>            # EDIT: specify your SCINet project account
#SBATCH --mem=8G                       # EDIT: increase for larger databases
#SBATCH --time=00:30:00                # EDIT: estimate the complete search and add a safety buffer
#SBATCH --output=blast_search_%j.out


module load {{ page.tools.blast.module.atlas }}/{{ page.tools.blast.version.atlas }}  # Atlas
# module load {{ page.tools.blast.module.ceres }}/{{ page.tools.blast.version.ceres }}   # Ceres

WORKDIR="{{ page.tutorial.root_practice }}/{{ page.tutorial.workdir }}"
DATABASE_DIR="{{ page.tutorial.root_practice }}/{{ page.tutorial.database_workdir }}"
DATABASE="${DATABASE_DIR}/{{ page.tutorial.v_proteome_db }}"
QUERY="${DATABASE_DIR}/{{ page.tutorial.protein_query }}.fasta"
OUTPUT="${WORKDIR}/{{ page.tutorial.protein_query }}_{{ page.tutorial.v_proteome_db | split: '/' | first }}_hits.tsv"
EVALUE="{{ page.tutorial.evalue }}"
MAX_TARGET_SEQS="{{ page.tutorial.max_target_seqs }}"

blastp \
  -query "${QUERY}" \
  -db "${DATABASE}" \
  -out "${OUTPUT}" \
  -outfmt '6 qseqid sseqid pident length qstart qend sstart send evalue bitscore qcovs stitle' \
  -evalue "${EVALUE}" \
  -max_target_seqs "${MAX_TARGET_SEQS}" \
  -num_threads "${SLURM_CPUS_PER_TASK}"
```

The example uses the same query, database, and search settings as the interactive examples above. Replace them when running a different BLAST search.

</div>


## Summary

This tutorial used BLAST+ to compare *D. vitifoliae* protein and nucleotide queries with *V. vinifera* proteome and genome databases. It demonstrated direct and translated searches with `blastp`, `tblastn`, `blastn`, `blastx`, and `tblastx`, interpreted the resulting alignments, and showed how to scale searches with SLURM.

<div class="usa-accordion " >
{% include accordion title="Quick Quiz: Check your understanding" controls="quiz-blast-search" expanded=false class="question" icon=true %}
<div id="quiz-blast-search" class="accordion_content" markdown='1'>
Check your understanding of BLAST program selection, translated searches, alignment interpretation, and result integration.

{% include question qid="1,2,3,4,5,6" %}
</div>
</div>
