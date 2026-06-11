# Copilot Instructions — Howitt Lab RNA-Fusion Study

## Resuming a session

Read `docs/session_state.md` — it contains the current project status, what was done last session, and the exact run command ready to copy-paste.

---

## Project Overview

Characterizing somatic gene fusions across uterine sarcoma samples and integrating them with EPIC v2 methylation profiles at Stanford (Howitt Lab / PI: Dr. Howitt, contact for blockers: Reem).

- **Pipeline:** nf-core/rnafusion v4.1.0 source, pinned to `-r 4.1.2` for reproducibility
- **HPC:** Stanford Sherlock, SLURM scheduler, Singularity containers (Docker/Conda disabled)
- **Cohort:** 192 samples total — 96 DNAnexus (IM 2020) + 96 Medgenome (MT 2025); 98 with methylation data (primary analyses), 110 RNA-only
- **Subtypes:** LGESS, HGESS, AS, UUS, DDX, IMT, LMSm, LMSc, UTROSCT

Primary run guide: `docs/rnafusion_4.1.2_official_usage.md` ← **most up to date**
Full study design: `docs/rnafusion_study_design.md`
PI summary (all tracks): `docs/Summarization_from_Brooke.md`

---

## Running the Pipeline

Start inside a `screen` or `tmux` — the Nextflow head job must stay alive while SLURM jobs run.

```bash
screen -S rnafusion
export NXF_VER=25.04.4   # REQUIRED — pipeline incompatible with Nextflow 26.x
module load nextflow

# Without COSMIC (use until credentials are obtained):
nextflow run /scratch/users/franzake/BROOKE_LAB/src/nf-core-rnafusion_4.1.0/4_1_0/main.nf \
  -profile bhowitt \
  --input /scratch/users/franzake/BROOKE_LAB/data/samplesheet.csv \
  --no_cosmic \
  -resume

# With COSMIC (pass credentials on the command line — do not commit them):
nextflow run /scratch/users/franzake/BROOKE_LAB/src/nf-core-rnafusion_4.1.0/4_1_0/main.nf \
  -profile bhowitt \
  --input /scratch/users/franzake/BROOKE_LAB/data/samplesheet.csv \
  --cosmic_username YOUR_EMAIL \
  --cosmic_passwd YOUR_PASSWORD \
  -resume
```

All resource, tool, genome, and path settings are already in `conf/bhowitt.config` — do not duplicate them on the command line. Pin version with `-r 4.1.2` for reproducibility.

- `-resume` restarts from the last successful step using the work directory cache
- Do **not** pass individual reference paths — only `--genomes_base` is set in bhowitt.config; all sub-paths resolve from `genome = GRCh38` + `genome_gencode_version = 46`
- Do **not** use `-c custom.config` for parameters — use `--param value` or `-params-file params.yaml`

### Monitor

```bash
squeue -u franzake                              # SLURM job queue
tail -f .nextflow.log                          # Nextflow head log
du -sh /scratch/users/franzake/rnafusion_work  # work dir usage
```

---

## Architecture

### Repository layout

```
BROOKE_LAB/
├── .github/copilot-instructions.md     ← this file
├── data/
│   ├── samplesheet.csv                 ← BUILT ✅ (192 rows, strandedness=reverse)
│   └── references/GRCh38/              ← local references (mostly complete — see status below)
├── docs/
│   ├── rnafusion_4.1.2_official_usage.md  ← PRIMARY run guide (most up to date)
│   ├── session_state.md                   ← current project state, resume here
│   ├── rnafusion_study_design.md          ← full A/B/C/D analysis plan
│   └── Summarization_from_Brooke.md       ← multi-track PI summary
└── src/
    └── nf-core-rnafusion_4.1.0/4_1_0/    ← pipeline source (do not edit core files)
        ├── main.nf                         ← pipeline entry point
        └── conf/bhowitt.config             ← ONLY file to edit for this study
```

### Key paths

| Resource | Path |
|---|---|
| Samplesheet | `/scratch/users/franzake/BROOKE_LAB/data/samplesheet.csv` |
| References root | `/scratch/users/franzake/BROOKE_LAB/data/references/GRCh38/` |
| Work directory | `/scratch/users/franzake/rnafusion_work` (fast scratch, **not persistent**) |
| Results output | `/oak/stanford/groups/bhowitt/rnafusion_results/` (persistent) |
| Metadata (Excel) | `/scratch/users/franzake/BROOKE_LAB/src/AS_methylome_clinical_metadata_2025.xlsx` |
| IM FASTQs | `/oak/stanford/groups/bhowitt/DNAnexus/Fastqs/200803_A00646_0057_BHTHTNDMXX/` |
| MT FASTQs | `/oak/stanford/groups/bhowitt/Medgenome2025_RNAseq/Raw files/` |
| IM BAMs (pre-existing) | `/oak/stanford/groups/bhowitt/DNAnexus/Bams/` |
| Existing STAR-Fusion results | `/oak/stanford/groups/bhowitt/DNAnexus/Star_Fusion/` |
| Singularity image cache | `/scratch/users/franzake/nf-core-rnafusion_4.1.0/4_1_0/work/singularity/` |

### Reference status (verified 2026-05-27)

All references are present and complete.

| Reference | Status |
|---|---|
| Genome FASTA, FAI, GTF, refFlat, rRNA intervals | done |
| STAR index (including `SA` file) | done |
| STAR-Fusion CTAT library | done |
| Salmon index, gffread transcriptome | done |
| FusionCatcher, Arriba (v2.5.0) | done |
| Fusion-report DBs (FusionGDB2 + Mitelman) | done |
| Fusion-report COSMIC DB | empty — register at cancer.sanger.ac.uk or use `--no_cosmic` |
| HGNC nomenclature (`hgnc/hgnc_complete_set.txt`) | done |

> Do **not** overwrite `fusion_report_db/` from S3 — it already contains COSMIC; the S3 version does not.

---

## Key Conventions

### The `bhowitt` profile

`conf/bhowitt.config` is the only file to edit for this study. It sets:
- `genomes_base`, `outdir`, `workDir`, `genome = GRCh38`, `genome_gencode_version = 46`
- `tools`, `tools_cutoff = 2` (require ≥2 callers to agree)
- `read_length = 100`, `trim_tail_fusioncatcher = 0` (confirmed 100 bp for both batches)
- SLURM executor `--account=bhowitt`; `process_high`/`process_long` → `owners` partition; `process_high_memory` (STAR-Fusion, FusionCatcher, 256 GB) → `bigmem`
- Singularity enabled with existing cache; Docker/Conda/Podman/Apptainer all disabled

Never commit COSMIC credentials — they are commented out in `bhowitt.config` and must be passed on the command line.

### Process naming in bhowitt.config

When writing `withName:` overrides, use the exact nf-core module process names — not the tool names:

| Tool | Correct process name |
|---|---|
| Arriba | `ARRIBA_ARRIBA` |
| STAR-Fusion | `STARFUSION_DETECT` |
| FusionCatcher | `FUSIONCATCHER_FUSIONCATCHER` |
| Fusion-report | `FUSIONREPORT_DETECT` |
| STAR alignment | `STAR_ALIGN` |

### Samplesheet

**Path:** `data/samplesheet.csv` — 192 rows, 192 unique samples, `strandedness = reverse` for all.

| Batch | n | Sample name format | FASTQ root |
|---|---|---|---|
| Intermountain 2020 (DNAnexus) | 96 | `A{N}_S{M}` | `/oak/.../200803_A00646_0057_BHTHTNDMXX/` |
| MT plate 2025 (Medgenome) | 96 | `GYN_{code}` | `/oak/.../Medgenome2025_RNAseq/Raw files/` |

- IM sample names use the full `A{N}_S{M}` filename prefix; GYN-to-A-number mapping is in metadata col R
- MT plate files use zero-padded 3-digit GYN codes; letter suffix (A/B/C/D) preserved for multi-lesion patients
- Rows with the same `sample` name are **merged as multi-run input** — this is how dual-batch samples (GYN_298, GYN_123) should be handled
- `Positive_Control-1` (Medgenome) is excluded

### Fusion confidence tiers

| Tier | Criteria |
|---|---|
| High | COSMIC/Mitelman hit + ≥2 callers + absent in normals + recurrent across ≥2 patients |
| Medium | ≥2 callers + FusionInspector validated + absent in normals |
| Low | Single caller only, or also in normals |
| Artifact | Read-through (adjacent genes), on Arriba blacklist |

### Multi-sample patients

Five patients have multiple samples: p447 (×3), p481 (×2), p367 (×2), p311 (×2), p590 (×2). Used for intra-patient fusion evolution analysis (Analysis C6) and expression clustering consistency checks.

### Intermountain BAM reuse

Pre-existing STAR BAMs at `/oak/stanford/groups/bhowitt/DNAnexus/Bams/` can be used as BAM input to skip re-alignment. BAI files are gzip-compressed — decompress before use: `gunzip /oak/stanford/groups/bhowitt/DNAnexus/Bams/*.bam.bai.gz`.

### Key analysis tiers

- **A (primary):** Pan-diagnosis landscape, fusion×methylation, batch QC — metadata complete for all 98
- **B (secondary):** FISH validation, age, recurrence, stage — partial metadata
- **C (targeted):** Subtype-specific (LGESS, HGESS, IMT, UTROSCT, DDX reclassification)
- **D (exploratory):** Germline, IHC, hormonal therapy

### Expected output structure

```
/oak/stanford/groups/bhowitt/rnafusion_results/
├── arriba/<sample>/        *.fusions.tsv, *.pdf
├── starfusion/<sample>/    *.fusion_predictions.tsv
├── fusioncatcher/<sample>/ final-list_candidate-fusion-genes.txt
├── fusionreport/<sample>/  *_fusionreport_index.html, *.tsv
├── fusioninspector/<sample>/ *.FusionInspector.fusions.tsv
├── vcf/<sample>/           *_fusion_data.vcf
├── multiqc/                multiqc_report.html
└── pipeline_info/          execution report, timeline, DAG
```

### Downstream post-pipeline analysis workflow

After the pipeline completes, the post-processing steps are:

```
1. Collect FusionInspector.fusions.tsv across all samples
2. Filter: ≥2 callers (tools_cutoff=2 already applied by pipeline)
3. Remove artifacts: Arriba blacklist + GTEx/TCGA read-throughs
4. Rank by FII score (COSMIC/Mitelman 50% weight each)
5. Flag recurrent fusions: ≥2 independent patients → strongest disease signal
6. Analysis A1: compare fusion landscape by DX subtype
7. Analysis A2: merge with EPIC methylation data → fusion × methylation co-occurrence
8. Analysis A3: stratify by tumor classification (primary vs recurrent/metastatic)
9. Analyses B1–B5, C1–C6: secondary, targeted, and multi-sample analyses
```

Prerequisite: clean annotation file from Brooke (GYN codes + histotype + prior FISH/molecular findings) — required for B1 (fusion validation vs FISH) and C1 (DDX reclassification).

### Expected driver fusions by subtype

| Subtype | n | Expected fusions |
|---|---|---|
| LGESS | 26 | JAZF1-SUZ12 (~50%), JAZF1-PHF1, PHF1-MEAF6/EPC1, YWHAE-NUTM2 |
| HGESS | 5 | YWHAE-NUTM2, BCOR, BRD8-PHF1 |
| IMT | 5 | ALK fusions (~50% expected) |
| UTROSCT | 6 | ESR1-NCOA3, GREB1-NCOA2, NCOA2 rearrangements |
| DDX | 13 | Ambiguous — JAZF1 → LGESS, BCOR → HGESS, ESR1-NCOA3 → UTROSCT |

---

## Pending Decisions (ask Reem)

| # | Item |
|---|---|
| 1 | GYN_298 and GYN_123: merge both batches or use one? (GYN_123 has 6× size discrepancy: IM=10.7 GB, MT=1.6 GB) |
| 2 | GYN_447B: FASTQ on disk but `col U = no` in metadata — include or exclude? |
| 3 | Clean annotation file from Brooke (GYN codes + histotype + prior FISH/molecular findings) — needed for analyses B1, C1 |
