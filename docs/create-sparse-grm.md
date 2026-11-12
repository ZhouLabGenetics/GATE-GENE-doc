---
title: Create sparse GRM
nav_order: 4
---

# Create sparse GRM
{: .no_toc }

1. TOC
{:toc}

The sparse GRM encodes relatedness. It is computed **once per cohort** and reused by
every phenotype, so this is not a per-analysis cost.

It is **optional**: step 1 can instead estimate a full GRM from its `--plinkFile` marker
set, which is what the [worked example]({{ site.baseurl }}/docs/example/) does. The
sparse GRM is what makes step 1 affordable at biobank scale.

## Command

```bash
createSparseGRM.R \
    --plinkFile=./input/nfam_100_nindep_0_step1_includeMoreRareVariants_poly \
    --nThreads=4 \
    --outputPrefix=./output_survival/sparseGRM \
    --numRandomMarkerforSparseKin=1000 \
    --relatednessCutoff=0.125
```

## Options

| Option | Default | Meaning |
|---|---|---|
| `--plinkFile` | `""` | PLINK prefix for the markers used to estimate relatedness. A pruned, LD-independent set. |
| `--outputPrefix` | `~/` | prefix for the output files. |
| `--numRandomMarkerforSparseKin` | `2000` | markers sampled to estimate kinship. |
| `--relatednessCutoff` | `0.125` | pairs below this coefficient are set to zero, which is what makes the matrix sparse. `0.05` is a common looser choice. |
| `--nThreads` | `16` | threads. |

## Output

The filename encodes the parameters, so changing them does not silently overwrite a
previous matrix:

```
sparseGRM_relatednessCutoff_0.125_1000_randomMarkersUsed.sparseGRM.mtx
sparseGRM_relatednessCutoff_0.125_1000_randomMarkersUsed.sparseGRM.mtx.sampleIDs.txt
```

Both are passed to step 1. The `.mtx` is Matrix Market format; the `.sampleIDs.txt`
gives the row and column order and is **not** optional — the matrix carries no IDs of
its own.

The coefficients are on the scale where the diagonal is about 1, so `--relatednessCutoff`
is read on that scale rather than as a kinship coefficient. On the example data the
diagonal runs 0.976–1.005 and the smallest retained off-diagonal is 0.12501, right at
the cutoff; `0.125` on this scale corresponds to third-degree relatives and closer. The
example matrix stores 3,700 of the 1,000 x 1,000 entries.

## Using it in step 1

```bash
    --sparseGRMFile=./output_survival/sparseGRM_relatednessCutoff_0.125_1000_randomMarkersUsed.sparseGRM.mtx \
    --sparseGRMSampleIDFile=./output_survival/sparseGRM_relatednessCutoff_0.125_1000_randomMarkersUsed.sparseGRM.mtx.sampleIDs.txt \
    --useSparseGRMtoFitNULL=TRUE
```

Without `--useSparseGRMtoFitNULL=TRUE` the files are ignored and the full GRM is
estimated from `--plinkFile` instead.

## If step 2 will also get the sparse GRM

Add `--useSparseGRMforVarRatio=TRUE` to step 1 as well:

```bash
    --useSparseGRMtoFitNULL=TRUE \
    --useSparseGRMforVarRatio=TRUE
```

{: .warning }
> These two options are not interchangeable, and the first does not imply the second.
> `--useSparseGRMtoFitNULL=TRUE` only fits the null model with the sparse GRM.
> `--useSparseGRMforVarRatio=TRUE` estimates the variance ratio with the full GRM as
> numerator and the sparse GRM as denominator, which adds a `sparse` row to
> `.varianceRatio.txt`.
>
> Step 2 **requires** that row if it is given `--sparseGRMFile`. Without it the run
> aborts:
>
> ```
> Error in Get_Variance_Ratio(...) :
>   sparse GRM is specified but the variance ratio for sparse GRM was not estimated
>   in Step 1. Pleae remove --sparseGRMFile and --sparseGRMSampleIDFile
> ```
>
> With it, `.varianceRatio.txt` holds three rows instead of two:
>
> ```
> 0.99894062514096	sparse	1
> 0.880056371970398	null	1
> 0.877200203301315	null_noXadj	1
> ```
>
> Step 2 reads the ratio from the first column, keyed by the label in the second
> (`R/readInGLMM.R:397-409`). A three-column file is also how step 2 recognises the
> current format: a file with a different column count is treated as having come from a
> version before 1.0.6.
