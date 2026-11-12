---
title: Input file formats
nav_order: 6
---

# Input file formats
{: .no_toc }

1. TOC
{:toc}

## Phenotype file

Delimited text with a header. One row per sample.

The first rows of the bundled `extdata/input/pheno_1000samples_survival.txt`,
tab-separated:

```
survTime  X                  casecontrol  censorTime        AgeOfEventFinal  IsCensor  IND_ID  IND_index  Fam_index
127       1.54924188625294   1            504.376195815808  127              0         1a1     1          1
4883      -0.445333850734177 0            142.243983806111  142.243983806111 1         1a2     2          1
```

Columns the run does not name through an option are simply ignored, which is why this
file carries more columns than the analysis uses.

Required columns, named through options rather than by position:

| Option | Column | Contents |
|---|---|---|
| `--sampleIDColinphenoFile` | e.g. `IND_ID` | sample ID, matching the genotype files |
| `--phenoCol` | e.g. `casecontrol` | **event indicator**, 1 = event observed, 0 = censored |
| `--eventTimeCol` | e.g. `AgeOfEventFinal` | **event / follow-up time** |
| `--covarColList` | e.g. `X,sex,PC1,...` | covariates |

Missing values must be `NA`. Step 1 reduces the merged phenotype/covariate table with
`complete.cases` (`R/SAIGE_fitGLMM_fast.R:1161`), so a sample with a missing value in
any column the run uses is dropped entirely. A sentinel such as `-9` is **not**
recognised as missing — it will be modelled as a real value.

All participants must share one time origin, and the event time must be measured from
it consistently. Mixing "age at event" for some participants with "years since
enrolment" for others produces a model that converges and means nothing.

## Group file

Two or three lines per set, whitespace-delimited:

```
GENE1 var  1:1:A:C 1:2:A:C 1:3:A:C ...
GENE1 anno lof     lof     lof     ...
GENE3 var  1:51:A:C
GENE3 anno intergenic
```

| Line type | Contents |
|---|---|
| `var` | variant IDs, in the genotype file's exact naming |
| `anno` | one annotation per variant, same order and count as the `var` line |
| `weight` | optional, one weight per variant |

Omit the `weight` line and weights default to `Beta(MAF, 1, 25)`, which up-weights rarer
variants; `--weights.beta=1,25` changes the two shape parameters and
`--is_no_weight_in_groupTest=TRUE` turns weighting off.

{: .note }
> `--help` states that "the variant ids must be in the format `chr:pos_ref/alt`". That is
> not a hard requirement, and the bundled example files do not follow it: both
> `group_new_chrposa1a2.txt` (`1:1:A:C`) and `group_new_snpid.txt` (`rs1`) run correctly
> against their matching genotype files. What matters is that the IDs match the genotype
> file exactly — not that they take any particular shape.

{: .warning }
> The `var` line's IDs must match the genotype file exactly. A mismatch leaves the
> set-based output file 0 bytes long, with a zero exit code and an `Analysis done!`
> message. See [Step 2]({{ site.baseurl }}/docs/set-based-test-step2/).

The annotations used here must match the labels in `--annotation_in_groupTest`. A set
annotated only with labels absent from that list produces no output rows.

## Sparse GRM

Two files from `createSparseGRM.R`: a Matrix Market `.mtx` and its `.sampleIDs.txt`.
The matrix carries no identifiers, so the sample ID file is required, not optional.

## Step 1 PLINK set

A pruned, approximately LD-independent marker set, supplied to step 1 as `--plinkFile`.
It has two jobs: it builds the **full GRM** when no sparse GRM is supplied, and markers
are sampled from it to calibrate the **variance ratio**. It is not the data being
tested. SAIGE's guidance on including some rarer variants in this set applies
unchanged.

## Genotypes for step 2

PLINK, PGEN, BGEN, VCF, BCF or SAV. Sample IDs must match the phenotype file.

For BGEN a `--sampleFile` is required, because the file does not always carry sample
IDs — the example BGEN does not, and the run reports `Sample IDs were not found in the
bgen file.` before falling back to the sample file. It is **one column, no header**
(the log confirms: `sample file only has one column and has no header`).

{: .warning }
> The sample file must list **every sample in the BGEN, in the BGEN's own order** — not
> the samples you are analysing. In the bundled example `samplelist.txt` has 10,000
> lines while the phenotype file has 1,000 samples; the BGEN holds 10,000 and the run
> selects its 1,000 from them by ID. Listing only the analysed samples misaligns
> genotypes to people, and nothing in the output will say so.
