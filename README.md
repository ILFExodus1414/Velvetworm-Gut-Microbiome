## QIIME2 Pre-processing & Analysis (V4–V5, 16S rRNA)

Author: Imelda L. Forteza,Ph.D.  


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


## To cite this workflow and analysis:

Forteza, I.L., et al. (2025). Gut Microbiome of Velvet Worms Across Deadwood Microhabitats in Australian Forest Refugia. Zenodo. [DOI to be assigned upon publication]

## FAIR & Reproducibility Notes

Include manifest.csv, metadata.tsv, and README.md with your Zenodo deposit

Scripts track parameters, software versions, and intermediate outputs

Pre-trained classifiers and .qza artifacts can be archived upon release

---
## License
This workflow and associated code are made available under the MIT License
.
