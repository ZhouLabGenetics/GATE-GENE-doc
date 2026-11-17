---
title: Step 1 — null model
parent: Set-based test
nav_order: 1
---

# Step 1 — fit the null survival model
{: .no_toc }

1. TOC
{:toc}

## Command

```bash
step1_fitNULLGLMM.R \
    --plinkFile=./input/nfam_100_nindep_0_step1_includeMoreRareVariants_poly \
    --phenoFile=./input/pheno_1000samples_survival.txt \
    --phenoCol=casecontrol \
    --eventTimeCol=AgeOfEventFinal \
    --covarColList=X \
    --sampleIDColinphenoFile=IND_ID \
    --traitType=survival \
    --eventTimeBinSize=1 \
    --outputPrefix=./output_survival/example_survival \
    --nThreads=4 \
    --LOCO=FALSE \
    --minMAFforGRM=0.01 \
    --skipModelFitting=FALSE \
    --tol=0.01 \
    --isCovariateOffset=FALSE \
    --IsOverwriteVarianceRatioFile=TRUE
```

To fit with a precomputed sparse GRM rather than the full GRM, add the options shown in
[Create sparse GRM]({{ site.baseurl }}/docs/create-sparse-grm/) — three to use it for
the null model, and a fourth if step 2 will be given the sparse GRM too.

## The survival options

| Option | Default | Meaning |
|---|---|---|
| `--traitType` | `binary` | **`survival`** selects GATE-GENE. |
| `--phenoCol` | `""` | the **event indicator**: `1` = event observed, `0` = censored. |
| `--eventTimeCol` | `""` | the **event / follow-up time**. Required for survival. |
| `--eventTimeBinSize` | `NULL` | width of the bins the event times are floored into, **in the same units as `--eventTimeCol`**. See below. |
| `--pcgforUhatforSurvAnalysis` | `FALSE` | solver used for the variance-ratio estimation. `FALSE` takes the exact-solve path, which materialises dense matrices sized by samples x distinct (binned) event times; `TRUE` uses preconditioned conjugate gradient and holds only vectors. Set `TRUE` at biobank scale: on the example data, with everything else held fixed, the two paths move the group-test P values by at most 1.8e-07 — no more than re-running perturbs them — so this is a scalability choice, not an accuracy one. |

{: .warning }
> **`--phenoCol` and `--eventTimeCol` are not interchangeable.** One is the 0/1
> indicator, the other the time. Swapping them runs to completion without error and
> produces meaningless results. Nothing in the output will tell you.

{: .warning }
> **`--eventTimeBinSize` is required in version 1.5.2.** Omit it and step 1 fits the null
> model, reports a converged `Tau`, then aborts with
> `object 'minEventTime' not found`. The variable is assigned only inside this option's
> branch (`R/SAIGE_fitGLMM_fast.R:1149`) but read unconditionally at lines 1575 and
> 1592.
>
> It is not a no-op: event times become `(t - min(t)) %/% binSize + 1`, so continuous
> times are floored into integer bins. Choose the width to match the resolution your
> phenotype actually has, rather than copying `1` by reflex.
>
> The bin count is also what sizes the exact-solve path. With `--eventTimeBinSize=1` on
> the example data the log prints `dim(Rmat):  1000 154` — 1,000 samples by 154 distinct
> binned event times. A finer bin means more columns, so this option trades time
> resolution against memory.

### Choosing the bin width

The width is **in whatever unit `--eventTimeCol` is measured in**. The option has no
notion of years or days; it simply divides. So if follow-up time is recorded in years:

| `--eventTimeBinSize` | bin width | use when |
|---|---|---|
| `1` | one year | time is known only to the year, or a coarse baseline hazard is acceptable |
| `0.25` | one quarter | a compromise at large sample size |
| `0.083` (≈ 1/12) | one month | time is known to the month — the usual case for EHR-derived phenotypes |
| `0.0027` (≈ 1/365) | one day | time is known to the day; expect many bins, and see the memory note above |

If follow-up is instead recorded **in days**, the same four widths are `365`, `91`, `30`
and `1`. The arithmetic is identical; only the unit of the column changes.

Two consequences to weigh against each other:

- **Too coarse** discards genuine ordering. Two participants whose events are four
  months apart fall in the same bin and become ties, flattening the baseline hazard.
- **Too fine** grows the number of distinct bins, which costs memory on the exact-solve
  path and buys nothing beyond the resolution the data actually has. Binning to the day
  when the record resolves only to the month is precision that is not there.

A worked case: the *All of Us* trajectory analyses this documentation draws on used
`--eventTimeBinSize=0.083` with time in years — monthly bins — because the underlying
phecode event dates resolve to a month rather than a day.

## Other options worth setting deliberately

| Option | Default | Note |
|---|---|---|
| `--covarColList` | `""` | comma-separated covariates. |
| `--qCovarColList` | `""` | which of those are categorical. Note the capital **C** — `--qcovarColList` is not a valid option. |
| `--sampleIDColinphenoFile` | `IID` | the ID column; must match the genotype files' sample IDs. Only needs setting if the column is not already named `IID`. |
| `--LOCO` | `TRUE` | leave-one-chromosome-out. The bundled examples pass `FALSE`; what matters for a set-based exome scan is `--LOCO=FALSE` in **step 2**, since `TRUE` there would also require `--chrom`. |
| `--minMAFforGRM` | `0.01` | MAF floor for markers entering the GRM. |
| `--isCovariateOffset` | `TRUE` | **no effect on a survival run** — step 1 forces it to `FALSE`. See the note below. |
| `--IsOverwriteVarianceRatioFile` | `FALSE` | set `TRUE` when re-running, or the existing file is kept. |
| `--skipVarianceRatioEstimation` | `FALSE` | `TRUE` writes no `.varianceRatio.txt`. |
| `--useSparseGRMtoFitNULL` | `FALSE` | fit the null model with the sparse GRM. |
| `--useSparseGRMforVarRatio` | `FALSE` | also estimate a **sparse** variance ratio. Required if step 2 will be given the sparse GRM — see [Create sparse GRM]({{ site.baseurl }}/docs/create-sparse-grm/). |

{: .note }
> **`--isCovariateOffset` does nothing on a survival run.** Whatever you pass,
> `fitNULLGLMM` overrides it to `FALSE` before it is used, as soon as it sees
> `--traitType=survival` (`R/SAIGE_fitGLMM_fast.R:971-974`):
>
> ```r
> }else{
>     isCovariateOffset = FALSE
>     cat("trait type is survival. Covariates are not included as offset\n")
> }
> ```
>
> Step 1 prints that line, so you can confirm it in your own log. The covariates in
> `--covarColList` are estimated as fixed effects either way, and the `.rda` handed to
> step 2 is the same. The examples here pass `--isCovariateOffset=FALSE` to match what
> the run actually does; omitting it is equivalent.
>
> This is also why the bundled examples can pass `FALSE`
> (`extdata/cmd_survival.sh:62`, `extdata/cmd.sh:177`) and `TRUE`
> (`extdata/cmd_survival.sh:81`) for survival without disagreeing in behaviour.
>
> For binary and quantitative traits the option does apply, and the **option name is the
> correct reading of it**: `TRUE` absorbs the covariates into an offset, refits the null
> model as `pheno ~ 1`, and does not estimate their coefficients — step 1 prints
> `isCovariateOffset=TRUE, so fixed effects coefficients won't be estimated.`
> (`R/SAIGE_fitGLMM_fast.R:1397-1405`). The help text, *"Whether to estimate fixed effect
> coeffciets. [default, 'TRUE']"*, states the opposite and is wrong. The defaults differ
> between interfaces as well: `TRUE` from the command line
> (`extdata/step1_fitNULLGLMM.R:120`) against `FALSE` in the R function
> (`R/SAIGE_fitGLMM_fast.R:960`).

## Output

| File | Contents |
|---|---|
| `<prefix>.rda` | the fitted null model, including the trait type |
| `<prefix>.varianceRatio.txt` | the variance ratio(s) used to calibrate step 2 |

Both are passed to step 2. Because the trait type is stored in the `.rda`, step 2 needs
no survival-specific options.

`--skipVarianceRatioEstimation=TRUE` suppresses the second file, so step 1 then writes
only the `.rda` and step 2 has no `--varianceRatioFile` to be given. Leave it at
`FALSE` unless you know you do not need the variance ratio.

## Checking it converged

Step 1 prints the variance components and fixed effects near the end:

```
Tau:
[1] 1.0000000 0.2290464
Fixed-effect coefficients:
           [,1]
[1,] -0.8976721
```

A `Tau` second element at or very near zero means no detectable random-effect variance,
which is worth investigating before proceeding.
