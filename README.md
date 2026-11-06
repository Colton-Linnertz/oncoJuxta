<h1>
  <img src="docs/oncoJuxta-logo.svg" alt="oncoJuxta logo" width="48" align="bottom"> oncoJuxta
</h1>

![Status](https://img.shields.io/badge/status-in%20progress-orange)
![Python](https://img.shields.io/badge/python-3.x-3776AB?logo=python&logoColor=white)

**Near-truth somatic variant scoring from paired tumor-normal sequencing for precision oncology: machine learning–based pairing QC, evidence reconciliation, and calibrated VCF annotation.**

> oncoJuxta puts the tumor and normal samples **beside** each other and scores each variant to get as **near** the truth as the evidence allows.

> ⚠️ **Status: early development.** The features below describe planned design. Research use only; not validated for clinical decision-making.

---

## Overview

Somatic variant calling from tumor-normal pairs is noisy. Different callers disagree. Germline variants leak into somatic calls. Low-coverage sites in the normal look like "absent" when they are really "unknown." Blood-derived clonal hematopoiesis mutations look like tumor mutations. And sequencing or FFPE artifacts look like real low-frequency variants.

oncoJuxta is a machine learning tool that looks at the evidence for each candidate variant in **both** samples of a pair. It reconciles that evidence into a single, calibrated assessment and writes the result back into the VCF as a per-variant score and classification.

## Why it matters for precision oncology

Precision oncology depends on knowing which variants are truly somatic. A misclassified variant can mislead clinical decisions and patient care. 

Tumor-only analysis often gets this wrong:

• False positives: germline variants are reported as somatic.
• False negatives: aggressive filtering removes real somatic mutations, including those at known germline sites.
• Artifacts: permissive filtering lets noise through into downstream analyses.

Simple workarounds have well-known blind spots:

- **Population database filtering** removes true somatic variants that coincide with common polymorphisms and misses rare germline variants, especially in underrepresented populations.
- **VAF cutoffs** (≈50% / ≈100% = germline) break down with tumor-in-normal contamination, tumor heterogeneity, mosaicism, copy-number changes and LOH, and mapping problems in homologous regions.

These are exactly the situations where a model that weighs paired evidence together should outperform fixed rules. 

oncoJuxta aims to give downstream interpretation tools a cleaner, better-characterized variant set.

*oncoJuxta does not interpret clinical significance or recommend treatment. It aims to improve the quality of the variant calls that interpretation relies on.*

## Core components

### 1. Tumor-normal pairing

Before any variant is scored, oncoJuxta checks that the pair is valid:

- **Sample identity check:** genotype concordance at common germline SNPs confirms the tumor and normal come from the same individual.
- **Sex and ancestry consistency checks**
- **Contamination estimates:** cross-sample contamination in either sample, and tumor-in-normal (TiN) contamination, which can suppress true somatic calls.
- **Pair-level QC report:** depth, coverage overlap and the fraction of the target callable in both samples.

**Supported pairings (planned):**

| Tumor sample | Matched normal |
|---|---|
| Tissue (fresh-frozen or FFPE) | Blood or adjacent normal tissue |
| Cell-free DNA (liquid biopsy) | Matched white blood cells (buffy coat), as recommended by ESMO and AMP guidance for cfDNA somatic interpretation |

### 2. Tumor-normal reconciliation

For every candidate site, oncoJuxta brings together the evidence from both samples into one view:

- Allele counts, VAF, depth, base/mapping quality and strand balance **in both tumor and normal**
- **"Absent" vs "not callable":** whether the normal truly lacks the variant or simply lacks coverage there
- **Artifact signals:** FFPE damage signatures (e.g. C>T deamination), strand and read-position bias, panel-of-normals hits
- **Local context:** copy number and allelic imbalance, mappability and homologous regions
- Merging and harmonizing calls from **multiple upstream callers** (e.g. Mutect2, Strelka2, VarScan), including normalization of representation (left-alignment, multi-allelic splitting)

Each variant is assigned a reconciled class:

| Class | Meaning |
|---|---|
| `SOMATIC` | Present in tumor, supported absent in normal |
| `GERMLINE` | Present in both at germline-consistent VAF |
| `CHIP` | Present at low VAF in a blood-derived normal, in a clonal hematopoiesis gene (e.g. *DNMT3A*, *TET2*, *PPM1D*, *ASXL1*), and likely blood-derived rather than tumor-derived |
| `LOH` | Germline heterozygous in normal, shifted toward 0 or 1 in tumor |
| `ARTIFACT` | Likely technical (FFPE damage, strand bias, low mapping quality, panel-of-normals hit) |
| `UNCERTAIN` | Not enough evidence in one or both samples |

### 3. Machine learning scoring

- **Training data:** labeled tumor-normal variants from established truth/benchmark sets (e.g. SEQC2 somatic reference, GIAB germline sets, orthogonally validated calls), with dedicated labels for CH and FFPE artifacts
- **Features:** paired-sample evidence from the reconciliation step, sequence context, caller agreement, population allele frequency, panel-of-normals membership, copy-number context and CH gene membership
- **Models:** start with interpretable gradient-boosted trees as a baseline. Explore deep models on read pileups later.
- **Calibration:** scores are calibrated probabilities, so a score of 0.9 should be right about 90% of the time
- **Explainability:** per-variant feature attributions (e.g. SHAP) show *why* a call was scored as it was
- **Validation:** held-out samples; tissue vs cfDNA and fresh-frozen vs FFPE evaluation; cross-platform and cross-assay evaluation; benchmarking against existing somatic callers and filters

### 4. Tumor-only fallback mode

A matched normal is not always available. In that case oncoJuxta can run in a **tumor-only mode**:

- Uses a model trained without normal-sample features. It relies on population databases, panel of normals, VAF and copy-number context, and CH gene priors.
- Scores are **explicitly flagged** (`OJX_MODE=TUMOR_ONLY`) and calibrated separately, so they are never mistaken for paired results.
- The QC report states the expected loss in accuracy compared with paired mode.

## Output

oncoJuxta annotates the input VCF rather than replacing it:

```
##INFO=<ID=OJX_SCORE,Number=A,Type=Float,Description="oncoJuxta calibrated probability the variant is a true somatic variant">
##INFO=<ID=OJX_CLASS,Number=A,Type=String,Description="oncoJuxta reconciled class: SOMATIC|GERMLINE|CHIP|LOH|ARTIFACT|UNCERTAIN">
##INFO=<ID=OJX_NCALL,Number=A,Type=String,Description="Normal sample callability at site: COVERED|LOW_COVERAGE|NOT_COVERED">
##INFO=<ID=OJX_MODE,Number=1,Type=String,Description="oncoJuxta scoring mode: PAIRED|TUMOR_ONLY">
```

It also produces:

- **Somatic VCF:** scored, annotated variants
- **Germline VCF:** germline variants separated out, for cancer-predisposition review and follow-up
- **Pair-level QC report:** identity, contamination, callable territory, scoring mode

## Planned usage

```bash
oncojuxta pair-check --tumor tumor.bam --normal normal.bam --ref GRCh38.fa
oncojuxta reconcile  --tumor tumor.bam --normal normal.bam --vcf calls1.vcf.gz calls2.vcf.gz -o reconciled.vcf.gz
oncojuxta score      --vcf reconciled.vcf.gz --model oncojuxta-v1 -o scored.vcf.gz

# Tumor-only fallback
oncojuxta score      --vcf tumor_calls.vcf.gz --tumor-only --model oncojuxta-to-v1 -o scored.vcf.gz
```

*(Interface subject to change.)*

## Roadmap

- [ ] Pair identity and contamination QC
- [ ] Multi-caller VCF harmonization
- [ ] Paired-evidence feature extraction
- [ ] CHIP and FFPE artifact labeling
- [ ] Training set curation and labeling pipeline
- [ ] Baseline model and calibration
- [ ] VCF annotation, germline output and QC report
- [ ] cfDNA + white-blood-cell pairing support
- [ ] Tumor-only fallback mode
- [ ] Benchmarking against existing somatic filters

## Disclaimer

oncoJuxta is a research tool. Its scores are probabilistic estimates, not diagnoses, and it has not been validated for clinical use.

## License

TBD
