# QIIME2 Pre-processing & Analysis (V4–V5, 16S/18S)

Author: ILF  
Last updated: `r Sys.Date()`


> **Note:** This repository is currently private and will be made public upon acceptance of the associated manuscript. A Zenodo DOI will be issued at that time for reproducibility and citation.

##Repository Overview

- QIIME2-based preprocessing of V4–V5 16S/18S reads
- Metadata and manifest templates
- Shell and R Markdown scripts for alpha/beta diversity, core microbiome, and differential abundance (in separate files)
- Analytical outputs (figures, tables) are excluded here but included in the manuscript submission

---

## Overview
This repository contains shell + QIIME2 commands (also embedded in an R Markdown) to:
1) Trim primers/adapters with **cutadapt**  
2) (Optional) Separate reads into **prokaryote (16S)** vs **eukaryote (18S)** using **bbsplit**  
3) Create a QIIME2 **manifest**  
4) **Import** reads, **inspect quality**, and **denoise** via **DADA2**  
5) **Export** features/statistics, **classify** ASVs, make **taxonomy barplots**, and **build a tree**

> Tested with QIIME2 2023.9. Adjust flags/paths if using another version.

---

## Requirements

- Linux/macOS shell with `bash`
- QIIME2 (e.g., 2023.9) with plugins: `dada2`, `feature-classifier`, `fragment-insertion`, `taxa`, `tools`, `demux`
- `cutadapt` (>= 4.x recommended)
- **Optional:** `bbmap` (for `bbsplit.sh`), `biom-format` (for `biom convert`)
- RStudio (optional) if using the `.Rmd` version
- A trained Naive Bayes classifier for your target region (e.g., `515926gg2-classifier.qza`)

Activate your conda env (example):
```bash
conda activate qiime2-2023.9
```

---

## Directory Layout

```
.
├── 00-raw/                # raw FASTQ files: SAMPLE_R1.fq.gz, SAMPLE_R2.fq.gz
├── 01-trimmed/            # output of cutadapt
├── 02-PROKs/00-fastq/     # optional: PROK reads after bbsplit
├── 02-EUKs/00-fastq/      # optional: EUK reads after bbsplit
├── 03-DADA2d/             # DADA2 outputs
├── 05-GG2-classified/     # taxonomy classification outputs
├── 06-GG2.barplots/       # taxa barplots
└── logs/                  # logs for each step
```

> Create the base folders as needed; most scripts here create them automatically.

---

## Quickstart

### 1) Trim primers/adapters with cutadapt
Save as `run_cutadapt.sh` (or run in an Rmd **bash** chunk):

```bash
mkdir -p logs/01-trimmed 01-trimmed

for item in 00-raw/*_R1.fq.gz; do
  filestem=$(basename "$item" _R1.fq.gz)
  R1=00-raw/${filestem}_R1.fq.gz
  R2=00-raw/${filestem}_R2.fq.gz

  cutadapt --no-indels --pair-filter=any --error-rate=0.2 \
    -g ^GTGYCAGCMGCCGCGGTAA -G ^CCGYCAATTYMTTTRAGTTT \
    -o 01-trimmed/${filestem}.R1.trimmed.fastq \
    -p 01-trimmed/${filestem}.R2.trimmed.fastq \
    "$R1" "$R2" 2>&1 | tee -a logs/01-trimmed/${filestem}.cutadapt.log
done
```

> **Tip (Rmd users):** Use a bash chunk header:  
> \`\`\`{bash} … \`\`\`  
> If you must use an R chunk, wrap commands with `system()`.

### 2) (Optional) Split PROK vs EUK reads with bbsplit
Save as `run_bbsplit.sh` and edit `path=` to your bbsplit DB.

```bash
mkdir -p 02-PROKs/00-fastq 02-EUKs/00-fastq logs/02-bbsplit

for item in 01-trimmed/*.R1.trimmed.fastq; do
  filestem=$(basename "$item" .R1.trimmed.fastq)
  R1in=01-trimmed/${filestem}.R1.trimmed.fastq
  R2in=01-trimmed/${filestem}.R2.trimmed.fastq

  /path/to/bbmap/bbsplit.sh \
    usequality=f qtrim=f minratio=0.30 minid=0.30 pairedonly=f threads=20 -Xmx100g \
    path=/path/to/EUK-PROK-bbsplit-db/ \
    in="$R1in" in2="$R2in" basename=${filestem}.trimmed.%_.fastq \
    2>&1 | tee -a logs/02-bbsplit/$filestem.bbsplit.log
done

mv *EUK*fastq 02-EUKs/00-fastq
mv *PROK*fastq 02-PROKs/00-fastq
```

### 3) Create a QIIME2 manifest (paired-end)
**Assumes** files are in `00-fastq/` and follow `*1.fastq` / `*2.fastq` pattern.  
Save as `make_manifest.sh`:

```bash
set -euo pipefail
find ./00-fastq -type f -size 0 -print0 | xargs -0 -r rm --

cutME="trimmed.SILVA_132_PROK.cdhit95pc_1.fastq"

for item in 00-fastq/*1.fastq; do
  basename "$item" "$cutME" | sed 's/_/-/g'
done > names

for item in 00-fastq/*1.fastq; do printf "%s/%s\n" "$PWD" "$item"; done > reads
for item in 00-fastq/*2.fastq; do printf "%s/%s\n" "$PWD" "$item"; done >> reads

for item in 00-fastq/*1.fastq; do printf "forward\n"; done > direction
for item in 00-fastq/*2.fastq; do printf "reverse\n"; done >> direction

printf "sample-id,absolute-filepath,direction\n" > manifest.csv
paste -d, names reads direction >> manifest.csv
rm names reads direction
```

### 4) Import sequences to QIIME2
```bash
qiime tools import \
  --type PairedEndSequences \
  --input-path paired-end-sequences \
  --output-path paired-end-sequences.qza
```

### 5) Quality summary
```bash
qiime demux summarize \
  --i-data 16s.qza \
  --output-dir 02-quality-plots-R1-R2 \
  --verbose
```

### 6) DADA2 (paired-end)
```bash
mkdir -p logs/03-DADA2

trunclenf=210   # example; set from your quality plots
trunclenr=194   # example

qiime dada2 denoise-paired \
  --i-demultiplexed-seqs 16s.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f $trunclenf \
  --p-trunc-len-r $trunclenr \
  --output-dir 03-DADA2d \
  --p-n-threads 10 \
  --verbose 2>&1 | tee -a logs/03-DADA2/DADA2.stderrout
```

**Single-end (if needed):**
```bash
qiime dada2 denoise-single \
  --i-demultiplexed-seqs ./demux_seqs.qza \
  --p-trunc-len 190 \
  --o-table ./dada2_table.qza \
  --o-representative-sequences ./dada2_rep_set.qza \
  --o-denoising-stats ./dada2_stats.qza
```

### 7) Export DADA2 outputs
```bash
qiime tools export --input-path 03-DADA2d/representative_sequences.qza --output-path 03-DADA2d/
qiime tools export --input-path 03-DADA2d/denoising_stats.qza --output-path 03-DADA2d/
qiime tools export --input-path 03-DADA2d/table.qza --output-path 03-DADA2d/

biom convert -i 03-DADA2d/feature-table.biom \
             -o 03-DADA2d/feature-table.biom.tsv \
             --to-tsv
```

### 8) Classify ASVs (Naive Bayes, V4–V5)
```bash
qiime feature-classifier classify-sklearn \
  --i-classifier 515926gg2-classifier.qza \
  --i-reads 03-DADA2d/representative_sequences.qza \
  --output-dir 05-GG2-classified
```

### 9) Taxonomy barplots
```bash
qiime taxa barplot \
  --i-table 03-DADA2d/table.qza \
  --i-taxonomy 05-GG2-classified/classification.qza \
  --m-metadata-file alldata.tsv \
  --output-dir 06-GG2.barplots
```

### 10) Phylogenetic tree (SEPP)
```bash
qiime fragment-insertion sepp \
  --i-representative-sequences 03-DADA2d/gg2_rep_seqs_final.qza \
  --i-reference-database sepp-refs-gg-13-8.qza \
  --o-tree gg2.asvs-tree.qza \
  --o-placements gg2.insertion-placements.qza \
  --p-threads 5 \
  --o-filtered-data 03-DADA2d/rep_seqs_final.qza
```

---

## R Markdown Usage

- Prefer bash chunks:  
  ```
  ```{bash}
  # your shell commands here
  ```
  ```
- If you must run shell from an R chunk, use `system()` in R:
  ```r
  system("cutadapt ...")
  ```

**Common issue:** “unexpected token” errors occur when pasting bash into an `r` chunk. Use a `{bash}` chunk or `system()`.

---

## Troubleshooting

- **`cutadapt: command not found`**: Ensure PATH is set in the knitting environment. Use full path (`/usr/bin/cutadapt`) or activate conda.
- **Windows line endings**: Convert scripts with `dos2unix script.sh` or `sed -i 's/\r$//' script.sh`.
- **Rmd knitting uses different PATH**: Knit from a terminal with `Rscript -e "rmarkdown::render('file.Rmd')"` after activating the conda env.
- **Manifest errors**: Check that `sample-id` entries are unique and that absolute paths are correct.

---

## Citations

- Bolyen et al. 2019. *QIIME 2: Reproducible, interactive, scalable, and extensible microbiome data science.*  
- QIIME2 Tutorials (Atacama soils; PD mice).  
- Fuhrman et al. 2019. *Fuhrman Lab 515F-926R 16S and 18S rRNA Gene Sequencing Protocol.* DOI: 10.17504/protocols.io.vb7e2rn  
- Callahan et al. DADA2 tutorial: https://benjjneb.github.io/dada2/tutorial.html

---

## FAIR & Reproducibility Notes

- Include `manifest.csv`, `metadata.tsv` (MIxS-compatible), `README.md`, and `data_manifest.tsv` with your upload (Zenodo/Qiita).  
- Record software versions, parameters (e.g., DADA2 truncation lengths), and reference databases in `logs/` and/or the Rmd header.
- Consider archiving classifiers (`*.qza`) alongside your workflow.

---


##Citation

To cite this workflow and analysis:

Forteza, I.L., et al. (2025). Gut Microbiome of Velvet Worms Across Deadwood Microhabitats in Australian Forest Refugia. Zenodo. [DOI to be assigned upon publication]

##FAIR & Reproducibility Notes

Include manifest.csv, metadata.tsv, and README.md with your Zenodo deposit

Scripts track parameters, software versions, and intermediate outputs

Pre-trained classifiers and .qza artifacts can be archived upon release

---
## License
This workflow and associated code are made available under the MIT License
.
