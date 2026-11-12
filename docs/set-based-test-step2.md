---
title: Step 2 — gene-based test
parent: Set-based test
nav_order: 2
---

# Step 2 — gene-based test
{: .no_toc }

1. TOC
{:toc}

Step 2 takes **no survival-specific options**: it reads the trait type from the fitted
null model (`R/readInGLMM.R:129`), so the command is identical in form to SAIGE-GENE+.

Its *internals* do branch on trait type, which matters in one place — see
[Conditional analysis](#conditional-analysis) below.

## Command

```bash
step2_SPAtests.R \
    --bgenFile=./input/genotype_100markers.bgen \
    --bgenFileIndex=./input/genotype_100markers.bgen.bgi \
    --sampleFile=./input/samplelist.txt \
    --SAIGEOutputFile=./output_survival/example_survival_groupTest_out.txt \
    --GMMATmodelFile=./output_survival/example_survival.rda \
    --varianceRatioFile=./output_survival/example_survival.varianceRatio.txt \
    --groupFile=./input/group_new_chrposa1a2.txt \
    --annotation_in_groupTest=lof,missense:lof,missense:lof:synonymous \
    --maxMAF_in_groupTest=0.0001,0.001,0.01 \
    --chrom=1 \
    --AlleleOrder=ref-first \
    --minMAF=0 \
    --minMAC=0.5 \
    --LOCO=FALSE
```

## Masks

`--annotation_in_groupTest` and `--maxMAF_in_groupTest` are **crossed**. The call above
defines 3 annotation classes x 3 MAF cutoffs = 9 masks per set, plus a Cauchy
combination across them.

Annotation classes are colon-separated cumulative groupings:

| Spec | Includes |
|---|---|
| `lof` | loss-of-function only |
| `missense:lof` | missense and LoF |
| `missense:lof:synonymous` | all three |

The labels must match the annotations in the group file exactly. A set whose
annotations are absent from this list produces no rows — see the FAQ.

{: .note }
> The separator changes between input and output: you request `missense:lof` with a
> **colon**, and the `Group` column of the result reports it as `missense;lof` with a
> **semicolon**. A script that greps the output for the string it passed in will match
> nothing.

## Genotype input

| Format | Options |
|---|---|
| PLINK | `--bedFile --bimFile --famFile` |
| BGEN | `--bgenFile --bgenFileIndex --sampleFile` |
| PGEN | `--pgenPrefix` |
| VCF / BCF | `--vcfFile --vcfFileIndex --vcfField` |
| SAV | `--savFile --savFileIndex` |

`--chrom` is required for VCF/BCF/SAV input and whenever LOCO is in use; it is otherwise
optional. The string must match the file's chromosome naming exactly — `1` does not
match `chr1`.

{: .warning }
> **`--AlleleOrder` silently flips effect directions if wrong.** It follows the file, not
> the cohort: UK Biobank exome PLINK files need `alt-first`, while UK Biobank imputed
> BGEN needs `ref-first`. Most other sources are `ref-first`. Nothing in the output
> indicates the choice was wrong, and note the option's own default is `alt-first`.

{: .warning }
> **Variant IDs in the group file must match the genotype file exactly.** On a mismatch
> step 2 **exits 0**, prints `Analysis done!`, and creates the set-based output file
> **completely empty — 0 bytes, with no header row**. The companion `.singleAssoc.txt`
> gets a header and nothing else. The only clue is two lines in the log:
>
> ```
> 101  markers in 'RegionFile' are not in 'GenoFile'.
> Read in  3  region(s) from the group file.
> The chunk is empty
> ```
>
> Test that the output file is non-empty — an exit-code check or a header-skipping
> parser will both report success.
>
> This is a live trap in the bundled example data, which ships **two** group files for
> the same genotypes:
>
> | Group file | Variant IDs | Pair it with |
> |---|---|---|
> | `group_new_chrposa1a2.txt` | `1:1:A:C` | the BGEN |
> | `group_new_snpid.txt` | `rs1` | the PLINK bed/bim/fam |
>
> Crossing them matches nothing. Both work with their own genotype file — each gives 14
> rows on the example data at `--maxMAF_in_groupTest=0.001,0.01`.

## Conditional analysis

To test a set conditional on one or more known variants, name them with `--condition`,
comma-separated, in the genotype file's own ID naming:

```bash
    --condition=rs17,rs60
```

Step 2 asks for them **in the order they occur in the genotype file** — it prints
`Please specify the conditioning markers in the order as they are store in the
genotype/dosage file.` — but it does not check. A run given them in the wrong order
completes without any warning, so the ordering is yours to get right.

`--weights_for_condition` supplies per-variant weights if the defaults are not wanted.

Five columns are added, **inserted after `SE_Burden` rather than appended** — a run with
`--condition` has 16 columns, not 11:

```
Region Group max_MAF Pvalue Pvalue_Burden Pvalue_SKAT BETA_Burden SE_Burden
  Pvalue_cond Pvalue_Burden_cond Pvalue_SKAT_cond BETA_Burden_cond SE_Burden_cond
MAC Number_rare Number_ultra_rare
```

{: .warning }
> **Conditional analysis does not work with BGEN input.** The BGEN reader sets the
> marker-info table to `NULL` (`R/Geno.R:327`), and the conditional set-up merges the
> conditioning IDs against it, so the run dies with a misleading message:
>
> ```
> Error in merge.data.table(conditionDat, markerInfo, ...) :
>   The following columns listed in `by.y` are missing from y: [ID]
> ```
>
> Use PLINK, PGEN or VCF input for conditional analysis.

{: .warning }
> **A monomorphic conditioning marker crashes the run**, after printing a message that
> does not stop it:
>
> ```
> ERROR: Conditioning marker is monomorphic
> Error in if (abs(q.sum - m1)/sqrt(var1) < 2) { :
>   missing value where TRUE/FALSE needed
> ```
>
> The `ERROR:` line is printed once per offending marker and does not itself stop the
> run; the abort comes afterwards, from the `NaN` it leaves behind.
>
> The second error is a consequence of the first, not an independent problem. Check the
> MAC of every conditioning marker before using it.

{: .warning }
> Conditional group tests on a time-to-event trait were **wrong before April 2026**.
> `SAIGE_Test_main.R` gated the conditional set-up on `traitType == "binary"`, so a
> survival trait fell through to a path that does not condition correctly. The fix
> ([commit `5cf8fe84`](https://github.com/ZhouLabGenetics/GATE-GENE/commit/5cf8fe84))
> extends that branch to `survival`.
>
> It landed **after** the `1.5.2` version bump and did not change the version number, so
> `1.5.2` names two different code states. The published image
> `wzhou88/saigetest:1.5.2` is the **fixed** one — its `/app/R/SAIGE_Test_main.R:475`
> carries the survival branch and the package was built 2026-04-06 17:31 UTC, after the
> 15:57 commit. A 1.5.2 built from source before that commit is not. If you use
> conditional analysis on a survival trait, verify your build.

## Other options

| Option | Default | Note |
|---|---|---|
| `--AlleleOrder` | `alt-first` | see the warning above — the default is **not** what the example data needs. |
| `--minMAF` | `0` | MAF floor. `0` keeps everything. |
| `--minMAC` | `0.5` | MAC floor. Retains singletons, which carry information in a set test. |
| `--is_fastTest` | `FALSE` | faster approximate path. |
| `--MACCutoff_to_CollapseUltraRare` | `10` | variants at or below this MAC are collapsed rather than tested individually; drives `Number_ultra_rare`. |
| `--r.corr` | `0` | `0` gives SKAT-O; `1` makes it a Burden-only test. |
| `--weights.beta` | `1,25` | shape parameters of the `Beta(MAF, a, b)` variant weights used when the group file has no `weight` line. |
| `--is_no_weight_in_groupTest` | `FALSE` | drop MAF weighting entirely. |
| `--SPAcutoff` | `2` | the saddlepoint approximation is used beyond this many standard deviations from the mean; within it, the plain score-test P value is returned. |
| `--dosage_zerod_cutoff` | `0.2` | dosages at or below this are set to 0, for variants under `--dosage_zerod_MAC_cutoff`. |
| `--dosage_zerod_MAC_cutoff` | `10` | the MAC below which that zeroing applies. |
| `--is_single_in_groupTest` | `FALSE` | only has an effect on a Burden-only run: SKAT and SKAT-O write single-variant results regardless. |
| `--sparseGRMFile`, `--sparseGRMSampleIDFile` | — | supply the sparse GRM to step 2 as well. |

## Saddlepoint approximation

The SPA is what keeps type I error controlled when events are rare, and
`--SPAcutoff=2` is what decides when it is used: a variant whose statistic sits within
2 standard deviations of the mean gets the ordinary score-test P value, and the SPA is
applied beyond that. The single-variant output records the choice per variant in its
`Is.SPA` column.

On the bundled example no variant reaches that threshold — the largest `|Z|` is 1.68 and
`Is.SPA` is `FALSE` throughout — so the worked example does **not** exercise the SPA
path. It is exercised in real data, where the tail is populated.

{: .note }
> Two defaults modify dosages before testing, and the log states both:
> `Any dosages <=  0.2  for genetic variants with MAC <=  10  are set to be 0.` For
> rare-variant work this is worth knowing about rather than discovering later; change it
> with `--dosage_zerod_cutoff` and `--dosage_zerod_MAC_cutoff`.

## Parallelising

Step 2 is embarrassingly parallel. Split by chromosome, or by chunks of the group file,
and concatenate the outputs. Each job needs only the step 1 `.rda` and
`.varianceRatio.txt`, which are read-only.
