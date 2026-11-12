---
title: Output
nav_order: 7
---

# Output
{: .no_toc }

1. TOC
{:toc}

## Set-based results

One row per set x mask, plus one `Cauchy` row per set combining its masks.

| Column | Meaning |
|---|---|
| `Region` | set name from the group file |
| `Group` | annotation class tested |
| `max_MAF` | MAF cutoff for this mask |
| `Pvalue` | **SKAT-O P value** — the headline result, with the default `--r.corr=0` |
| `Pvalue_Burden` | Burden P value |
| `Pvalue_SKAT` | SKAT P value |
| `BETA_Burden` | burden effect size |
| `SE_Burden` | its standard error |
| `MAC` | minor allele count across the mask |
| `Number_rare` | variants tested individually |
| `Number_ultra_rare` | variants collapsed together rather than tested individually: MAC at or below `--MACCutoff_to_CollapseUltraRare` (default `10`). The log describes the scheme: `Ultra rare variants with MAC <=  10  will be collpased for set-based tests in the 'absence or presence' way.` — a carrier indicator, not a sum of alleles. |

On a `Cauchy` row all three P values are populated — each is a Cauchy combination of
that statistic across the set's masks — while `max_MAF`, `BETA_Burden`, `SE_Burden`,
`MAC`, `Number_rare` and `Number_ultra_rare` are all `NA`, because they describe a
single mask and the Cauchy row describes none.

With `--condition`, five more columns are **inserted after `SE_Burden`** (not appended):
`Pvalue_cond`, `Pvalue_Burden_cond`, `Pvalue_SKAT_cond`, `BETA_Burden_cond`,
`SE_Burden_cond`, giving 16 columns. Read the warnings on
[Step 2]({{ site.baseurl }}/docs/set-based-test-step2/) before using them on a survival
trait.

{: .note }
> A survival run's output carries **no** `MAC_case` / `MAC_control` columns. A
> binary-trait run does. Pipelines that parse by column position rather than by name
> will break when moved between trait types.

### Reading `BETA_Burden`

`BETA_Burden` from a survival fit is **not** on the log-odds scale, so a binary-trait
interpretation does not carry over.

[CONFIRM before release: state the exact scale — log hazard ratio — and whether
exponentiating yields a hazard ratio. This is not documented in the source and is the
single easiest thing for a user to misreport.]

### Which P value to use

`Pvalue` (SKAT-O) is the primary result. `Pvalue_Burden` and `Pvalue_SKAT` are its
components, reported for diagnosis: a set significant under Burden but not SKAT suggests
effects aligned in direction, the reverse suggests mixed directions. The `Cauchy` row
combines masks and is the right unit when reporting one P value per gene.

## Single-variant results

Written to `<SAIGEOutputFile>.singleAssoc.txt`. These cover the variants in the sets
and are **not** what the set-based P values refer to.

On a survival run the columns are:

```
CHR POS MarkerID Allele1 Allele2 AC_Allele2 AF_Allele2 MissingRate BETA SE
Tstat var p.value p.value.NA Is.SPA AF_event AF_censor N_event N_censor
```

`AF_event` / `AF_censor` and `N_event` / `N_censor` take the place of the case/control
columns a binary run would give. There is no `MAC` column here — allele count is
`AC_Allele2`.

The `--is_single_in_groupTest` flag defaults to `FALSE` on the command line
(`extdata/step2_SPAtests.R:124`), but single-variant results are written anyway
whenever SKAT or SKAT-O runs, because those need the per-variant statistics. The log
says so explicitly — `is_single_in_groupTest = TRUE. Single-variant assoc tests results
will be output` — even on a run that passed `FALSE`. The flag only has an effect on a
Burden-only run (`--r.corr=1`).

## Index file

`<SAIGEOutputFile>.index` records progress so an interrupted run can resume. It is not
a result file.

## Missing rows

A set or mask absent from the output is usually not an error:

| Absence | Reason |
|---|---|
| a `max_MAF` level missing | no variant in the set is that rare |
| a set missing entirely | its annotations are not in `--annotation_in_groupTest`, or none of its variants are in the genotype file |
| **every** set missing | variant IDs do not match the genotype file — see [Step 2]({{ site.baseurl }}/docs/set-based-test-step2/) |
