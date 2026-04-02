# 🧬 Vertebrate Genome Assembly using HiFi, Bionano & Hi-C Data
---
> (https://training.galaxyproject.org/training-material/topics/assembly/tutorials/vgp_genome_assembly/tutorial.html)


---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Objectives](#objectives)
4. [Important Terminology](#important-terminology)
5. [Pipeline Overview](#pipeline-overview)
6. [Input Data](#input-data)
7. [Tools Used](#tools-used)
8. [Step 1 — Get Data](#step-1--get-data)
9. [Step 2 — HiFi Read Preprocessing (Cutadapt)](#step-2--hifi-read-preprocessing-cutadapt)
10. [Step 3 — Genome Profile Analysis](#step-3--genome-profile-analysis)
    - [a. k-mer Counting with Meryl](#a-k-mer-counting-with-meryl)
    - [b. Genome Profiling with GenomeScope2](#b-genome-profiling-with-genomescope2)
11. [Step 4 — Genome Assembly with hifiasm](#step-4--genome-assembly-with-hifiasm)
    - [Assembly Modes](#assembly-modes)
    - [Hi-C Phased Assembly](#hi-c-phased-assembly)
    - [Solo (Pseudohaplotype) Assembly](#solo-pseudohaplotype-assembly)
12. [Step 5 — Assembly Evaluation](#step-5--assembly-evaluation)
    - [gfastats (Summary Statistics)](#gfastats-summary-statistics)
    - [BUSCO (Gene Completeness)](#busco-gene-completeness)
    - [Merqury (k-mer QV)](#merqury-k-mer-qv)
13. [Step 6 — Purging False Duplications (purge_dups)](#step-6--purging-false-duplications-purge_dups)
14. [Step 7 — Scaffolding with Bionano Optical Maps](#step-7--scaffolding-with-bionano-optical-maps)
15. [Step 8 — Hi-C Scaffolding with YaHS](#step-8--hi-c-scaffolding-with-yahs)
    - [Pre-processing Hi-C Data](#pre-processing-hi-c-data)
    - [Initial Hi-C Contact Map](#initial-hi-c-contact-map)
    - [YaHS Scaffolding](#yahs-scaffolding)
    - [Final Evaluation with Pretext](#final-evaluation-with-pretext)
16. [Expected Results Summary](#expected-results-summary)

---

## Overview

This project demonstrates a complete genome assembly workflow using the VGP (Vertebrate Genomes Project) pipeline on the Galaxy platform.

Genome assembly is the process of reconstructing a full genome sequence from small DNA fragments (reads). In this workflow, multiple sequencing technologies are combined to improve accuracy and completeness:

- HiFi reads (PacBio): Highly accurate long DNA sequences  
- Hi-C data: Helps organize sequences into chromosomes  
- Bionano optical maps: Provide large-scale structural information  

The pipeline integrates these datasets step by step to transform raw sequencing data into a high-quality, chromosome-level genome assembly.
 
Raw DNA data → processed → assembled → organized into chromosomes → evaluated


---


## Objectives
### 🔹 Main Objective
To perform a de novo genome assembly and generate a high-quality genome sequence using the VGP pipeline.

---

### 🔹 Specific Objectives

- Understand the complete genome assembly workflow  
- Perform preprocessing and quality control of sequencing data  
- Conduct k-mer analysis to estimate genome properties  
- Assemble genome into contigs using Hifiasm  
- Remove duplicate and redundant sequences  
- Use Hi-C and Bionano data for chromosome-level scaffolding  
- Evaluate assembly quality using metrics like N50 and completeness  
  

---

## Important Terminology

| Term | Definition |
|---|---|
| **Pseudohaplotype assembly** | Assembly with long phased blocks separated by regions where haplotype is ambiguous. Represented by a *primary* and *alternate* assembly. Can result in "switch errors". |
| **Primary assembly** | The more complete representation — includes both homozygous and heterozygous regions. Used for downstream analyses. |
| **Alternate assembly** | Contains the alternate alleles (haplotigs) not in the primary. Less complete since homozygous regions are absent. |
| **Phasing** | Partitioning contigs according to the haplotype they derive from. Can use parental data, Hi-C linkage, or long read linkage. |
| **Contig** | A contiguous, gapless sequence in the assembly. |
| **Scaffold** | One or more contigs joined by gap (N) sequences, using information from optical maps or Hi-C. |
| **Unitig** | The smallest unit in an assembly graph. Represents an unambiguous path in the graph. |
| **False duplication** | An assembly error where one genomic region appears twice. Can be a *haplotypic duplication* or an *overlap*. |
| **Purging** | Removing false duplications, collapsed repeats, and low-coverage regions from the assembly. |
| **HiFi reads** | PacBio reads 10–20 kbp long with >Q20 (99%) accuracy. Combine long reads with short-read-level accuracy. |
| **Misassembly** | Structural assembly error where non-adjacent sequences are placed next to each other. |
| **Missed join** | Two adjacent genomic regions not represented contiguously in the assembly. |
| **T2T assembly** | Telomere-to-telomere: completely gapless chromosome-level assembly. |
| **N50** | Length at which contigs of that size or longer cover ≥50% of the total assembly. Higher = more contiguous. |

---

## Pipeline Overview

The VGP-Galaxy pipeline consists of **10 modular workflows** organized into 5 stages:

```
Stage 1:  k-mer Profiling     →  WF1 (HiFi only) or WF2 (+ parental data)
Stage 2:  Contig Assembly     →  WF3 (solo) | WF4 (HiFi + Hi-C) | WF5 (trio)
Stage 3:  Purging [Optional]  →  WF6 (purge_dups)
Stage 4:  Scaffolding         →  WF7 (Bionano) → WF8 (Hi-C / YaHS)
Stage 5:  Decontamination     →  WF9  |  WF0 (mitochondrial assembly)
```

---

## Input Data 

| Input Data | Assembly Quality | 
|---|---|
| HiFi only | Minimum requirement |
| HiFi + Hi-C | Better haplotype resolution, fewer switch errors | 
| HiFi + Bionano | Better contiguity | 
| HiFi + Hi-C + Bionano | Even better contiguity | 
| HiFi + parental data | Properly phased | 
| HiFi + parental + Hi-C | Better haplotype resolution | 
| HiFi + parental + Bionano | Properly phased + improved contiguity | 
| HiFi + parental + Hi-C + Bionano | Best: properly phased + maximum contiguity | 


---

## Tools Used

| Tool | Purpose |
|---|---|
| **Cutadapt** |  Remove adapter-containing HiFi reads |
| **Meryl** |  k-mer counting and database operations |
| **GenomeScope2** | Genome profiling from k-mer spectra |
| **hifiasm** |  De novo assembly of HiFi reads |
| **gfastats** |  Assembly stats & GFA→FASTA conversion |
| **BUSCO** |  Gene completeness assessment |
| **Merqury** |  k-mer based quality value (QV) and completeness |
| **purge_dups** | Remove false duplications from assembly |
| **Bionano Solve** |  Hybrid scaffolding with optical maps |
| **BWA-MEM2** | Align Hi-C reads to assembly |
| **Samtools** |  SAM/BAM manipulation |
| **YaHS** | Hi-C based scaffolding |
| **PretextMap** | Generate Hi-C contact maps |
| **Pretext Snapshot** | Visualize Hi-C contact maps |

---

                ┌──────────────┐
                │ Input Data   │
                │ (HiFi/Hi-C)  │
                └──────┬───────┘
                       ↓
              ┌─────────────────┐
              │ Preprocessing   │
              │ (Cutadapt)      │
              └──────┬──────────┘
                     ↓
           ┌────────────────────┐
           │ k-mer Analysis     │
           │ (Meryl + GenomeScope)
           └──────┬─────────────┘
                  ↓
           ┌────────────────────┐
           │ Assembly           │
           │ (Hifiasm)          │
           └──────┬─────────────┘
                  ↓
         ┌──────────────────────┐
         │ Purging Duplicates   │
         │ (purge_dups)         │
         └──────┬───────────────┘
                ↓
        ┌───────────────────────┐
        │ Scaffolding           │
        │ (Bionano + Hi-C)      │
        └──────┬────────────────┘
               ↓
       ┌────────────────────────┐
       │ Evaluation             │
       │ (Pretext, QC metrics)  │
       └────────┬───────────────┘
                ↓
         ┌──────────────┐
         │ Final Genome │
         └──────────────┘

    
## Step 1 — Get Data

The tutorial uses **synthetic HiFi reads** generated from the *S. cerevisiae* S288C reference, using the HIsim simulator at 2% mutation rate, 30× coverage. This allows precise ground-truth evaluation.

### HiFi Reads (FASTA format)

```
https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_01.fasta
https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_02.fasta
https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_03.fasta
```

 datatype: `fasta`

### Hi-C Reads (fastqsanger.gz format)

```
https://zenodo.org/record/5550653/files/SRR7126301_1.fastq.gz  → rename: Hi-C_dataset_F
https://zenodo.org/record/5550653/files/SRR7126301_2.fastq.gz  → rename: Hi-C_dataset_R
```

 datatype: `fastqsanger.gz`



### Organizing Data

After uploading, combine the three HiFi FASTA files into a **Galaxy collection** (flat list) named `HiFi_collection`. 

---

## Step 2 — HiFi Read Preprocessing (Cutadapt)

Because PacBio SMRT technology can incorporate adapter sequences anywhere within a read (not just at the ends), we discard entire reads that contain adapter sequences rather than trimming them.

**Tool:** Cutadapt  
**Mode:** Single-end, discard any read containing an adapter

| Parameter | Value |
|---|---|
| Input | `HiFi_collection` |
| Adapter 1 | `ATCTCTCTCAACAACAACAACGGAGGAGGAGGAAAAGAGAGAGAT` |
| Adapter 2 | `ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT` |
| Max error rate | `0.1` |
| Min overlap | `35` |
| Reverse complement search | Yes |
| Discard trimmed reads | Yes |

**Output:** `HiFi_collection (trimmed)`

---

## Step 3 — Genome Profile Analysis

### Overview
+ Genome profiling is an important step performed before genome assembly. It helps in understanding the basic characteristics of the genome using sequencing data.

+ Instead of directly assembling the genome, we first analyze patterns in the reads to estimate properties like genome size, repeats, and heterozygosity.

+ K-mer frequency analysis provides key genome characteristics before assembly begins, including estimated genome size, heterozygosity, repeat content, and error rate.

###  Objectives

- Estimate genome size  
- Identify repetitive sequences  
- Measure heterozygosity (genetic variation)  
- Assess sequencing data quality  
- Prepare data for accurate assembly  

---
### Tools Used
#### 🔹 Meryl
- Counts k-mers (substrings of length k) from sequencing reads  
- Generates a k-mer frequency database  

#### 🔹 GenomeScope2
- Uses k-mer counts to model genome characteristics  
- Generates graphs and statistical estimates  
- use to estimate genome size, repeats, and heterozygosity
  
### a. `k-mer Counting with Meryl`

Meryl decomposes reads into k-mers, counts occurrences, and builds a searchable database. K-mer size **k=31** is used — long enough for uniqueness in most genomes, short enough for robustness to errors.

**Three Meryl runs:**

1. **Count** canonical k-mers per FASTA file in `HiFi_collection (trimmed)` → output: `meryldb` (collection)
2. **Union-sum** merge the collection into one database → output: `Merged meryldb`
3. **Generate histogram** from merged database → output: `meryldb histogram`

### b. `Genome Profiling with GenomeScope2`

GenomeScope2 fits a mixture of negative binomial distributions to the k-mer histogram to estimate genome properties.

**Parameters:**

| Parameter | Value |
|---|---|
| Input | `meryldb histogram` |
| Ploidy | `2` |
| k-mer size | `31` |
| Output summary | Yes |

**GenomeScope2 Outputs:**

| Output | Description |
|---|---|
| Linear plot | k-mer frequency vs. coverage |
| Log plot | Log-transformed version of the linear plot |
| Transformed linear plot | Frequency × coverage (highlights higher-order peaks) |
| Transformed log plot | Log-transformed version of transformed plot |
| Model | Detailed model fitting report |
| Summary | Key genome estimates (size, heterozygosity, error rate) |

**Expected Results:**

```
Estimated haploid genome size:  ~11.7 Mb
Heterozygosity:                  0.576%
Model fit:                       >93%
Heterozygous peak:               ~25× coverage
Homozygous peak:                 ~50× coverage



[Uploading GenomeScope version 2.0
input file = /corral4/main/objects/8/6/1/dataset_861792ca-38f6-41e1-bc78-10bb5fa38e9f.dat
output directory = .
p = 2
k = 31
TESTING set to TRUE

property                      min               max
Homozygous (aa)               99.4165%          99.4241%
Heterozygous (ab)             0.575891%         0.583546%
Genome Haploid Length         11,739,513 bp     11,747,352 bp
Genome Repeat Length          723,114 bp        723,597 bp
Genome Unique Length          11,016,399 bp     11,023,756 bp
Model Fit                     92.5159%          96.5191%
Read Error Rate               0.00094319%       0.00094319%
Galaxy139-[GenomeScope on dataset 134 Summary]

```

The bimodal k-mer distribution is characteristic of a diploid genome. The summary values (especially genome size) are used to parameterize later `purge_dups` runs.




<img width="500" height="500" alt="Galaxy135- GenomeScope on dataset 134 Linear plot" src="https://github.com/user-attachments/assets/abd42695-7a74-4c70-b4a0-98042e89bba2" />

---

<img width="500" height="500" alt="Galaxy136- GenomeScope on dataset 134 Log plot" src="https://github.com/user-attachments/assets/e2aeea85-df89-442d-a85a-01c818264cf0" />

---
## Step 4 — Genome Assembly with hifiasm

hifiasm is a fast, open-source *de novo* assembler for PacBio HiFi reads. It performs 3 rounds of haplotype-aware error correction and builds a phased string graph where heterozygous alleles appear as "bubbles."

### Assembly Modes

| Mode | Input | Output | Notes |
|---|---|---|---|
| **Solo** | HiFi only | Primary + alternate contigs | Requires purging |
| **Hi-C phased** | HiFi + Hi-C | Hap1 + Hap2 contigs | Usually no purging needed |
| **Trio** | HiFi + parental reads | Maternal + paternal contigs | Best phasing; requires parental samples |

### Hi-C Phased Assembly

**Tool:** hifiasm v0.19.8  
**Mode:** Standard (with Hi-C partition)

| Parameter | Value |
|---|---|
| Input reads | `HiFi_collection (trimmed)` |
| Hi-C R1 | `Hi-C_dataset_F` |
| Hi-C R2 | `Hi-C_dataset_R` |

**Outputs:**
- `Hap1 contigs graph` (GFA, tag: `#hap1`)
- `Hap2 contigs graph` (GFA, tag: `#hap2`)

**Convert GFA → FASTA** using gfastats (Tool mode: Genome assembly manipulation, Output: FASTA):
- `Hap1 contigs FASTA`
- `Hap2 contigs FASTA`

### Solo (Pseudohaplotype) Assembly

**Tool:** hifiasm v0.19.8  
**Mode:** Standard (no Hi-C data provided)

**Outputs:**
- `Primary contigs graph` (GFA)
- `Alternate contigs graph` (GFA)

> These outputs require the **purging step** to remove haplotypic duplications before scaffolding.


---

## Step 5 — Assembly Evaluation

Three complementary tools assess different dimensions of assembly quality.

### `gfastats (Summary Statistics)`

Generates N50, NG50, contig count, total length, GC content, and more. Run on GFA files to also capture graph-specific stats.

**Key parameters:**
- Expected genome size: `11747160` (from GenomeScope2 summary)
- Thousands separator: No

**Expected results for Hi-C phased assembly:**

| Metric | Hap1 | Hap2 |
|---|---|---|
| Total contigs | ~16 | ~17 |
| Total contig length | ~11.3 Mb | ~12.2 Mb |
| Largest contig | ~1,532,843 bp | ~1,531,728 bp |
| N50 | ~922,430 bp | ~923,452 bp |

> Values may differ slightly between runs and may be reversed between haplotypes.

After running gfastats separately on hap1 and hap2, use **Column join** to merge the outputs side-by-side, then **Search in textfiles** (Don't Match `scaffold`) to filter to contig-only statistics.

---

### `BUSCO (Gene Completeness)`

BUSCO checks for the presence, duplication, or absence of universal single-copy orthologs (genes expected once per haploid genome) to assess biological completeness.

**Tool:** BUSCO v5.5.0  
**Mode:** Genome assemblies (DNA), Metaeuk  
**Lineage:** Saccharomycetes (auto-detected or manually selected)

**BUSCO result categories:**
- **Complete (C):** Gene found at expected copy number
  - **Single-copy (S):** Present once — ideal
  - **Duplicated (D):** Present more than once — may indicate need for purging
- **Fragmented (F):** Partial match found
- **Missing (M):** Gene not found

> High duplication in BUSCO scores often indicates false duplications that need purging.

<img width="500" height="500" alt="Galaxy161- Busco on dataset 154_ Summary image - Specific lineage" src="https://github.com/user-attachments/assets/13f5d021-3824-4875-bda9-cbba3f0e0c73" />

<img width="500" height="500" alt="Galaxy164- Busco on dataset 153_ Summary image - Specific lineage" src="https://github.com/user-attachments/assets/c1f5e9de-ebc9-4da6-8325-caeb6d07959a" />




---

### `Merqury (k-mer QV)`

Merqury performs reference-free quality assessment by comparing k-mers in the reads to those in the assembly.

**Key metrics:**

| Metric | Description |
|---|---|
| **QV (Quality Value)** | Phred-scale measure of base accuracy. QV40 = 1 error per 10,000 bases. Higher is better. |
| **Completeness** | Fraction of read k-mers present in the assembly. Higher = more complete. |
| **Copy Number (CN) spectrum** | Distribution of k-mer copy numbers in the assembly — reveals if regions are properly diploid, collapsed, or duplicated |



<img width="500" height="500" alt="Galaxy182- output_merqury spectra-cn ln" src="https://github.com/user-attachments/assets/ecd30d1e-8295-4ccb-9d44-0d66022691ed" />


---

## Step 6 — Purging False Duplications (purge_dups)

> **Required for:** Solo assembly (primary/alternate)  
> **Usually not required for:** Hi-C phased or trio assemblies

`purge_dups` removes:
- **Haplotypic duplications:** Heterozygous loci where both haplotypes ended up in the same assembly
- **Contig overlaps:** Unresolved overlaps in the assembly graph that duplicate sequence

### Workflow for Solo Assembly

1. Align HiFi reads to the primary assembly
2. Calculate read depth per base
3. Identify cutoffs for haplotig marking (using GenomeScope2 estimates)
4. Purge haplotigs from primary → move to alternate
5. Purge the alternate assembly to remove redundancy

**Key purge_dups tools:**

| Tool | Purpose |
|---|---|
| `purge_dups` — Map reads | Align reads back to assembly |
| `purge_dups` — PB stat | Calculate read depth histogram |
| `purge_dups` — Calcuts | Identify coverage cutoffs |
| `purge_dups` — Split FASTA | Break assembly at gaps for analysis |
| `purge_dups` — Purge Dups | Execute the actual purging |
| `purge_dups` — Get Seqs | Extract purged primary and haplotigs |

**Result:** A purged primary assembly with haplotigs moved to the alternate set, and a purged alternate set.

---
# SCAFFOLDING

## Step 7 — Scaffolding with Bionano Optical Maps

> **Purpose:** Join contigs into larger scaffolds using long-range physical distance information from fluorescently labeled restriction sites on native DNA molecules (up to hundreds of kb in length).

Bionano optical maps provide **sized gaps** (with accurate distance estimates) between contigs, unlike Hi-C scaffolding which uses arbitrary gap sizes.

**Tool:** Bionano Solve (run via Galaxy wrapper)

### Inputs
- Purged primary assembly (FASTA)
- Bionano CMAP file (optical map data)
- Enzyme: DLE1 (or BSPQI, etc., depending on your experiment)

### Outputs
- Hybrid scaffold FASTA (contigs joined by optical map evidence)
- Non-hybridized contigs (contigs with no optical map support)
- Conflict report (locations where optical maps and sequences disagree)

### Evaluating Bionano Scaffolds

Run **gfastats**, **BUSCO**, and **Merqury** on the Bionano-scaffolded assembly and compare to pre-scaffolding statistics. You should see improvement in:
- Scaffold N50 (substantially increased)
- Total scaffold count (reduced)
- No significant degradation in BUSCO completeness or Merqury QV

---


<img width="500" height="500" alt="Galaxy199- pretext_snapshotFullMap" src="https://github.com/user-attachments/assets/b96c78d7-2346-4c29-bfc4-0bca4baef20a" />




---

## Step 8 — Hi-C Scaffolding with YaHS

Hi-C data captures physical proximity of DNA loci in 3D nuclear space. Sequences on the same chromosome interact more frequently, allowing chromosome-level scaffolding.

### Pre-processing Hi-C Data

1. **Align Hi-C reads** to the (Bionano-scaffolded) assembly using **BWA-MEM2**
   - Use `-5SP` flag for Hi-C-specific alignment
   - Input: `Hi-C_dataset_F`, `Hi-C_dataset_R`, assembly FASTA

2. **Filter and sort** alignments with **Samtools**:
   - Filter for mapped reads (flag 2308)
   - Sort by read name (for pair-aware processing)

3. **Mark duplicates** and filter with **Picard MarkDuplicates**

4. **Filter for valid Hi-C pairs** (reads facing inward, on different restriction fragments)

### Initial Hi-C Contact Map

Generate a preliminary contact map to assess Hi-C data quality and assembly state **before** final scaffolding:

**Tools:** PretextMap → Pretext Snapshot  
- Input: filtered and sorted BAM  
- Output: contact map image  

Inspect the map for:
- Strong diagonal signal (intra-chromosomal contacts — good)
- Off-diagonal blocks (inter-chromosomal contacts — may indicate misassembly)
- Clean separation of chromosomes

### YaHS Scaffolding

YaHS (Yet Another Hi-C Scaffolder) uses Hi-C contact information to join contigs and order them into chromosome-level scaffolds.

**Tool:** YaHS v1.2a.2  
**Inputs:**
- Assembly FASTA (after Bionano scaffolding, or purged primary if no Bionano)
- Aligned Hi-C BAM file

**Key Outputs:**
- `YaHS scaffolds FASTA` — final chromosome-level assembly
- `YaHS scaffolds AGP` — gap/contig layout file

### Final Evaluation with Pretext

Generate a final Hi-C contact map on the **YaHS-scaffolded assembly** for visual quality assessment.

**Workflow:**
1. Align Hi-C reads to YaHS scaffolds (BWA-MEM2)
2. Convert to Pretext format (PretextMap)
3. Visualize (Pretext Snapshot)

**What to look for in the final contact map:**
- Clean, block-diagonal pattern — each block = one chromosome
- Strong intra-chromosomal interactions
- Minimal off-diagonal noise
- Telomeric signals at chromosome ends (if applicable)


<img width="500" height="500" alt="Galaxy247- pretext_snapshotscaffold_16" src="https://github.com/user-attachments/assets/a03dec61-5446-4347-ae72-8b2f536576c3" />

---

## Expected Results Summary

The following table summarizes approximate expected metrics at each pipeline stage for  *S. cerevisiae* dataset:

| Assembly Stage | Contigs/Scaffolds | N50 | BUSCO Complete | Merqury QV |
|---|---|---|---|---|
| hifiasm contigs (hap1) | ~16 | ~922 kbp | ~99% | ~40–50 |
| hifiasm contigs (hap2) | ~17 | ~923 kbp | ~99% | ~40–50 |
| After Bionano scaffolding | Fewer, larger | Higher | ~99% | Similar |
| After YaHS (Hi-C) scaffolding | 16 (= chromosomes) | Full chromosome-scale | ~99% | Similar |

> Final scaffold count should match the known chromosome count of *S. cerevisiae* (16 chromosomes).

---

