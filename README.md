# RNAseq Fusion Analysis — Howitt Lab

Characterizing somatic gene fusions across uterine sarcoma samples and integrating them with EPIC v2 methylation profiles.

- **Pipeline:** nf-core/rnafusion v4.1.0 source, pinned to `-r 4.1.2`
- **HPC:** Stanford Sherlock (SLURM + Singularity)
- **Cohort:** 192 samples — 96 DNAnexus (IM 2020) + 96 Medgenome (MT 2025)
- **Subtypes:** LGESS, HGESS, AS, UUS, DDX, IMT, LMSm, LMSc, UTROSCT

## Quick start

```bash
screen -S rnafusion
export NXF_VER=25.04.4
module load nextflow

nextflow run src/nf-core-rnafusion_4.1.3/4_1_3/main.nf \
  -profile bhowitt \
  --input data/samplesheet.csv \
  --no_cosmic \
  -resume
```

## Docs

- `docs/rnafusion_4.1.2_official_usage.md` — primary run guide
- `docs/rnafusion_study_design.md` — full A/B/C/D analysis plan
- `docs/Summarization_from_Brooke.md` — PI summary

## Config

All resource, tool, and path settings are in `src/nf-core-rnafusion_4.1.3/4_1_3/conf/bhowitt.config`.
