---
title: Biobank-scale runs
nav_order: 8
---

# Biobank-scale runs
{: .no_toc }

1. TOC
{:toc}

The [worked example]({{ site.baseurl }}/docs/example/) runs on 1,000 samples and 100
markers. This page collects the settings that change when the cohort is a biobank
exome rather than a toy dataset. It is parameter guidance only — no cohort results.

## Step 0 — one sparse GRM for the whole cohort

Build it once with `createSparseGRM.R` and reuse it for every phenotype. At biobank
scale this is the difference between a tractable step 1 and an intractable one, and it
is the main reason the sparse GRM exists.

## Step 1 — what changes

| Setting | Toy example | Biobank exome |
|---|---|---|
| GRM | full, from `--plinkFile` | sparse, via `--sparseGRMFile` + `--useSparseGRMtoFitNULL=TRUE` |
| `--pcgforUhatforSurvAnalysis` | `FALSE` (the default) | **`TRUE`** |
| `--plinkFile` | the example markers | a pruned variance-ratio marker set, not the exome being tested |
| `--qCovarColList` | unused | the categorical covariates, which must also appear in `--covarColList` |

`--pcgforUhatforSurvAnalysis=TRUE` matters most. The default `FALSE` path builds dense
matrices sized by samples x distinct binned event times, which does not scale; `TRUE`
uses preconditioned conjugate gradient and holds only vectors. See
[Step 1]({{ site.baseurl }}/docs/set-based-test-step1/).

Switching it on is a scalability choice, not an accuracy compromise. Run on the bundled
example with everything else held fixed, the two solvers give a variance ratio of
`0.880056371970398` (`FALSE`) against `0.880057046836688` (`TRUE`), and the 14 group-test
P values differ by at most **1.8e-07**. That is the same magnitude as the difference
between two *identical* runs, which the random marker sampling already produces (see
[Example]({{ site.baseurl }}/docs/example/)) — so changing the solver perturbs the
result no more than re-running does. The `TRUE` run also never prints `dim(Rmat)`, the
dense matrix the default path builds and this one skips.

Choose `--eventTimeBinSize` deliberately rather than by reflex: it is required
(see [FAQ]({{ site.baseurl }}/docs/faq/)), it coarsens the event times, and it sets the
width of that matrix. With event times already recorded in whole units, `1` merges no
two distinct times.

`--ratioCVcutoff` (`0.001`) and `--traceCVcutoff` (`0.0025`) are **coefficient-of-variation
thresholds, not fixed iteration counts**: step 1 keeps adding randomly selected markers
until the variance-ratio CV falls below the first, and keeps adding trace-estimator runs
until the trace CV falls below the second. So they set how much work step 1 does, and
the defaults are normally what you want.

They also connect to reproducibility: because `--ratioCVcutoff` governs how many random
markers the variance ratio is built from, lowering it buys a more stable variance ratio
— and hence less run-to-run drift in the P values — at the cost of a slower step 1.
There is still no seed option.

## Step 2 — what changes

| Setting | Value | Why |
|---|---|---|
| `--AlleleOrder` | **`alt-first`** for UK Biobank exome PLINK | the file's convention, not the cohort's; getting it wrong silently flips effect directions |
| `--LOCO` | `FALSE` | `TRUE` applies leave-one-chromosome-out and then also requires `--chrom`, so a scan driven by a group file spanning chromosomes needs `FALSE` |
| `--minMAC` | `0.5` | keeps singletons, which carry information in a set test |
| `--minMAF` | `0` | no lower MAF bound at the marker level; the masks supply the upper bound |

Do **not** pass `--sparseGRMFile` to step 2 unless step 1 was run with
`--useSparseGRMforVarRatio=TRUE`; see
[Create sparse GRM]({{ site.baseurl }}/docs/create-sparse-grm/).

## Parallelising

Step 1 runs once per phenotype and is the expensive stage. Step 2 is embarrassingly
parallel: split by chromosome, or by chunks of the group file, and concatenate. Each
job needs only the step 1 `.rda` and `.varianceRatio.txt`, both read-only, so there is
no coordination between jobs.

Extra threads do not speed up step 2 — the Docker image sets `OMP_NUM_THREADS=1`
deliberately — so scale out with more jobs, not more threads per job.

## Checking a campaign

Two failure modes are silent at scale and worth asserting on, per job:

1. **An empty result.** On a variant-ID mismatch step 2 exits 0, prints
   `Analysis done!`, and leaves the output file 0 bytes long. Test that each output is
   non-empty; do not trust exit codes.
2. **A missing variance ratio.** If step 1 was interrupted after writing the `.rda` but
   before the `.varianceRatio.txt`, step 2 fails per job rather than up front.

Because the variance ratio is estimated from randomly sampled markers with no seed
option, refitting step 1 shifts every P value slightly. Fit once per phenotype and
reuse that fit across all step 2 jobs — which is the recommended workflow anyway.

---

## Platform recipes

Two worked skeletons for the platforms where GATE-GENE is most often run. Both are
written to be adapted: the option values are the ones that matter, the paths and
instance types are yours to set.

{: .warning }
> These two recipes are **not verified by execution** in the way the rest of this
> documentation is. Every other command on this site was run against
> `wzhou88/saigetest:1.5.2` and its output copied from the log. These two are adapted
> from production pipelines but cannot be re-run here, because neither platform is
> reachable outside its own environment. Treat them as starting points and check the
> first job's log before launching a campaign.

### UK Biobank RAP (DNAnexus, WDL)

RAP runs workflows as WDL compiled with `dxCompiler`. The null model is one job per
phenotype; step 2 is one job per chromosome or group-file chunk.

```wdl
version 1.0

workflow gate_gene_step1 {
  input {
    File bedFile
    File bimFile
    File famFile
    File phenoFile
    File sparseGRMFile
    File sparseGRMSampleIDFile
    String outputPrefix
    String eventIndicatorCol      # 0/1 indicator -> --phenoCol
    String eventTimeCol           # follow-up time -> --eventTimeCol
    String eventTimeBinSize       # bin width, in the units of eventTimeCol
    String sampleIDCol
    String covarColList
    String qCovarColList
  }

  call fit_null {
    input:
      bedFile = bedFile, bimFile = bimFile, famFile = famFile,
      phenoFile = phenoFile,
      sparseGRMFile = sparseGRMFile,
      sparseGRMSampleIDFile = sparseGRMSampleIDFile,
      outputPrefix = outputPrefix,
      eventIndicatorCol = eventIndicatorCol,
      eventTimeCol = eventTimeCol,
      eventTimeBinSize = eventTimeBinSize,
      sampleIDCol = sampleIDCol,
      covarColList = covarColList,
      qCovarColList = qCovarColList
  }

  output {
    File modelFile = fit_null.modelFile
    File varianceRatioFile = fit_null.varianceRatioFile
  }
}

task fit_null {
  input {
    File bedFile
    File bimFile
    File famFile
    File phenoFile
    File sparseGRMFile
    File sparseGRMSampleIDFile
    String outputPrefix
    String eventIndicatorCol
    String eventTimeCol
    String eventTimeBinSize
    String sampleIDCol
    String covarColList
    String qCovarColList
  }

  command <<<
    set -euo pipefail
    step1_fitNULLGLMM.R \
      --bedFile=~{bedFile} \
      --bimFile=~{bimFile} \
      --famFile=~{famFile} \
      --phenoFile=~{phenoFile} \
      --sparseGRMFile=~{sparseGRMFile} \
      --sparseGRMSampleIDFile=~{sparseGRMSampleIDFile} \
      --useSparseGRMtoFitNULL=TRUE \
      --useSparseGRMforVarRatio=TRUE \
      --traitType=survival \
      --phenoCol=~{eventIndicatorCol} \
      --eventTimeCol=~{eventTimeCol} \
      --eventTimeBinSize=~{eventTimeBinSize} \
      --pcgforUhatforSurvAnalysis=TRUE \
      --sampleIDColinphenoFile=~{sampleIDCol} \
      --covarColList=~{covarColList} \
      --qCovarColList=~{qCovarColList} \
      --outputPrefix=~{outputPrefix} \
      --LOCO=FALSE \
      --nThreads=1 \
      --IsOverwriteVarianceRatioFile=FALSE
  >>>

  output {
    File modelFile = outputPrefix + ".rda"
    File varianceRatioFile = outputPrefix + ".varianceRatio.txt"
  }

  runtime {
    docker: "wzhou88/saigetest:1.5.2"
  }
}
```

Step 2 is the same shape. The task body is:

```bash
step2_SPAtests.R \
  --bgenFile=~{bgenFile} \
  --bgenFileIndex=~{bgenFileIndex} \
  --sampleFile=~{sampleFile} \
  --GMMATmodelFile=~{modelFile} \
  --varianceRatioFile=~{varianceRatioFile} \
  --sparseGRMFile=~{sparseGRMFile} \
  --sparseGRMSampleIDFile=~{sparseGRMSampleIDFile} \
  --groupFile=~{groupFile} \
  --chrom=~{chrom} \
  --annotation_in_groupTest=~{annotations} \
  --maxMAF_in_groupTest=~{maxMAFs} \
  --SAIGEOutputFile=~{outputPrefix} \
  --LOCO=FALSE \
  --AlleleOrder=ref-first \
  --is_output_moreDetails=TRUE
```

Three RAP-specific points:

- **Watch line continuations.** A backslash followed by a trailing space is not a line
  continuation — it escapes the space, and every option after it is silently dropped.
  This is easy to introduce when editing a WDL `command` block and produces a run that
  looks successful.
- **`--AlleleOrder=ref-first`** matches the UKB BGEN convention. Getting it wrong
  flips the sign of every `BETA_Burden` without any error.
- **Pin the image by digest** rather than the `1.5.2` tag for anything you publish —
  see [Installation]({{ site.baseurl }}/docs/installation/).

### All of Us Researcher Workbench (dsub)

The Workbench runs batch jobs through `dsub` against the workspace bucket. The same two
steps, as a shell task:

```bash
#!/usr/bin/env bash
# GATE-GENE step 1 under dsub. Inputs arrive as ${INPUT_*}, outputs are
# written under ${OUT_DIR} and copied back to the bucket by dsub.
set -euo pipefail

step1_fitNULLGLMM.R \
  --sparseGRMFile="${SPGRM_FILE}" \
  --sparseGRMSampleIDFile="${SPGRM_SAMPLE}" \
  --useSparseGRMtoFitNULL=TRUE \
  --useSparseGRMforVarRatio=TRUE \
  --plinkFile="${PLINK_BFILE}" \
  --phenoFile="${PHENO_FILE}" \
  --traitType=survival \
  --phenoCol="${EVENT_INDICATOR_COL}" \
  --eventTimeCol="${EVENT_TIME_COL}" \
  --eventTimeBinSize="${EVENT_BIN:-0.083}" \
  --pcgforUhatforSurvAnalysis=TRUE \
  --sampleIDColinphenoFile="${SAMPLE_ID_COL}" \
  --covarColList="${COVARS}" \
  --qCovarColList="${QCOVARS}" \
  --outputPrefix="${OUT_DIR}/${PREFIX}" \
  --LOCO=FALSE \
  --nThreads=1 \
  --traceCVcutoff=0.0025 \
  --ratioCVcutoff=0.001 \
  --IsOverwriteVarianceRatioFile=FALSE
```

and step 2:

```bash
#!/usr/bin/env bash
set -euo pipefail

step2_SPAtests.R \
  --bgenFile="${BGEN_FILE}" \
  --bgenFileIndex="${BGEN_INDEX}" \
  --sampleFile="${SAMPLE_FILE}" \
  --GMMATmodelFile="${RDA_FILE}" \
  --varianceRatioFile="${VAR_FILE}" \
  --sparseGRMFile="${SPGRM_FILE}" \
  --sparseGRMSampleIDFile="${SPGRM_SAMPLE}" \
  --groupFile="${GROUP_FILE}" \
  --chrom="${CHROM}" \
  --annotation_in_groupTest='pLoF,missenseLC:pLoF,missenseLC:pLoF:synonymous' \
  --maxMAF_in_groupTest=0.0001,0.001,0.01 \
  --SAIGEOutputFile="${OUT_FILE}" \
  --LOCO=FALSE \
  --AlleleOrder=ref-first \
  --minMAF=0 \
  --minMAC=0.5 \
  --is_output_moreDetails=TRUE
```

Submit with the Workbench's `aou_dsub` helper — not `dsub` directly, since the helper
supplies the project, region, service account and network settings the environment
requires. One task per phenotype for step 1, one per chromosome for step 2, driven by a
TSV task file:

```bash
source ~/aou_dsub.bash

aou_dsub \
  --image "${ARTIFACT_REGISTRY_DOCKER_REPO}/wzhou88/saigetest:1.5.2" \
  --boot-disk-size 30 --preemptible \
  --machine-type n1-standard-2 \
  --name "gate_gene_step1" \
  --logging "${WORKSPACE_BUCKET}/logs/gate_gene_step1" \
  --tasks gate_gene_step1_tasks.tsv \
  --script gate_gene_step1.sh
```

Step 2 is submitted the same way with `--machine-type n1-standard-1`.

These are modest machines on purpose, and the sizing is worth stating because it is
easy to over-provision a campaign by an order of magnitude:

| | step 1 | step 2 |
|---|---|---|
| `--machine-type` | `n1-standard-2` | `n1-standard-1` |
| `--boot-disk-size` | 30 | 30 |
| `--disk-size` | not needed | not needed |

Both steps run with `--nThreads=1`, so extra cores buy nothing — the parallelism is
across *tasks*, not within one. Memory is the binding constraint rather than CPU, and
with `--pcgforUhatforSurvAnalysis=TRUE` the solver holds vectors instead of a dense
matrix, which is what keeps a 2-core standard node sufficient for step 1. Step 2 streams
the genotype file and needs less again.

`--preemptible` is worth taking at this size: the jobs are short and cheap to retry,
which is also why the script should skip tasks whose output already exists. `--tasks`
also accepts a trailing line range, which is the cheapest way to resubmit a failed
subset without rebuilding the TSV.

{: .note }
> The image comes from the workspace's Artifact Registry mirror
> (`${ARTIFACT_REGISTRY_DOCKER_REPO}`), not from Docker Hub — Workbench batch jobs
> cannot pull from Docker Hub directly. Copy the image into the registry once, then
> reference it by that path.

Two Workbench-specific points:

- **Checkpoint and skip completed tasks.** Jobs are preemptible and a long campaign
  will lose some. Have the script test for a non-empty output and exit early if it
  exists, so a resubmission is cheap.
- **Results leaving the Workbench are governed by the All of Us Data and Statistics
  Dissemination Policy.** Check the current participant-count and allele-count
  thresholds before exporting anything, and apply them to group-test output as well as
  to single-variant output.
