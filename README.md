# oncoJuxta

![Status](https://img.shields.io/badge/status-in%20progress-orange)

**Near-truth somatic variant scoring from paired tumor-normal sequencing.**
**ML-based pairing QC, evidence reconciliation, and calibrated VCF annotation.**

> oncoJuxta puts the tumor and normal samples **beside** each other and scores each variant to get as **near** the truth as the evidence allows.

> ⚠️ **Status: early development.** The features below describe planned design. Research use only; not validated for clinical decision-making.

---

## Overview

Somatic variant calling from tumor-normal pairs is noisy. Different callers disagree. Germline variants leak into somatic calls. Low-coverage sites in the normal look like "absent" when they are really "unknown." And sequencing or alignment artifacts look like real low-frequency mutations.

oncoJuxta is a machine-learning tool that looks at the evidence for each candidate variant in **both** samples of a pair. It reconciles that evidence into a single, calibrated assessment and writes the result back into the VCF as a per-variant score and classification.

## Core components

### 1. Tumor-normal pairing

Before any variant is scored, oncoJuxta checks that the pair is valid:

- **Sample identity check:** genotype concordance at common germline SNPs confirms the tumor and normal come from the same individual.
- **Sex and ancestry consistency checks**
- **Contamination estimates:** cross-sample contamination in either sample, and tumor-in-normal (TiN) contamination, which can suppress true somatic calls.
- **Pair-level QC report:** depth, coverage overlap and the fraction of the target callable in both samples.

### 2. Tumor-normal reconciliation

For every candidate site, oncoJuxta brings together the evidence from both samples into one view:

- Allele counts, VAF, depth, base/mapping quality and strand balance **in both tumor and normal**
- **"Absent" vs "not callable":** whether the normal truly lacks the variant or simply lacks coverage there
- Merging and harmonizing calls from **multiple upstream callers** (e.g. Mutect2, Strelka2, VarScan), including normalization of representation (left-alignment, multi-allelic splitting)
- Assigning each variant a reconciled category:
  - `SOMATIC`: present in tumor, supported absent in normal
  - `GERMLINE`: present in both at germline-consistent VAF
  - `LOH`: germline heterozygous in normal, shifted toward 0 or 1 in tumor
  - `ARTIFACT`: likely technical (strand bias, low mapping quality, panel-of-normals hit, etc.)
  - `UNCERTAIN`: not enough evidence in one or both samples

### 3. Machine-learning scoring

- **Training data:** labeled tumor-normal variants from established truth/benchmark sets (e.g. SEQC2 somatic reference, GIAB germline sets, orthogonally validated clinical calls)
- **Features:** paired-sample evidence from the reconciliation step, sequence context, caller agreement, population allele frequency and panel-of-normals membership
- **Models:** start with interpretable gradient-boosted trees as a baseline. Explore deep models on read pileups later.
- **Calibration:** scores are calibrated probabilities, so a score of 0.9 should be right about 90% of the time
- **Explainability:** per-variant feature attributions (e.g. SHAP) show *why* a call was scored as it was
- **Validation:** held-out samples, cross-platform and cross-assay evaluation, and benchmarking against existing somatic callers and filters

## Output

oncoJuxta annotates the input VCF rather than replacing it:

```
##INFO=<ID=OJX_SCORE,Number=A,Type=Float,Description="oncoJuxta calibrated probability the variant is a true somatic variant">
##INFO=<ID=OJX_CLASS,Number=A,Type=String,Description="oncoJuxta reconciled class: SOMATIC|GERMLINE|LOH|ARTIFACT|UNCERTAIN">
##INFO=<ID=OJX_NCALL,Number=A,Type=String,Description="Normal sample callability at site: COVERED|LOW_COVERAGE|NOT_COVERED">
```

It also produces a pair-level QC summary (identity, contamination, callable territory).

## Planned usage

```bash
oncojuxta pair-check --tumor tumor.bam --normal normal.bam --ref GRCh38.fa
oncojuxta reconcile  --tumor tumor.bam --normal normal.bam --vcf calls1.vcf.gz calls2.vcf.gz -o reconciled.vcf.gz
oncojuxta score      --vcf reconciled.vcf.gz --model oncojuxta-v1 -o scored.vcf.gz
```

*(Interface subject to change.)*

## Roadmap

- [ ] Pair identity and contamination QC
- [ ] Multi-caller VCF harmonization
- [ ] Paired-evidence feature extraction
- [ ] Training set curation and labeling pipeline
- [ ] Baseline model and calibration
- [ ] VCF annotation and QC report
- [ ] Benchmarking against existing somatic filters
- [ ] Tumor-only fallback mode (exploratory)

## Disclaimer

oncoJuxta is a research tool. Its scores are probabilistic estimates, not diagnoses, and it has not been validated for clinical use.

## License

TBD
