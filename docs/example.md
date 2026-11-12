---
title: Example
parent: Set-based test
nav_order: 3
---

# Worked example
{: .no_toc }

1. TOC
{:toc}

A complete run on the data bundled with the repository. No data application needed.

## Run it

```bash
git clone https://github.com/ZhouLabGenetics/GATE-GENE.git
cd GATE-GENE
docker run --rm -v "$PWD":/gate -w /gate \
  wzhou88/saigetest:1.5.2 bash tutorial/run_example.sh
```

The script runs `createSparseGRM.R`, step 1 and step 2, then validates the result. On
success it ends:

```
  columns (11): Region	Group	max_MAF	Pvalue	Pvalue_Burden	Pvalue_SKAT	BETA_Burden	SE_Burden	MAC	Number_rare	Number_ultra_rare
  rows (set x mask): 14
  P values: 14 non-NA, range 0.172 to 0.96

TUTORIAL PASSED
```

{: .note }
> Step 1 in this example fits with the **full** GRM estimated from `--plinkFile`, not
> with the sparse GRM the script just built. That is deliberate: it exercises
> `createSparseGRM.R` and the no-sparse-GRM path in one run. To use the sparse GRM
> instead, add the options from
> [Create sparse GRM]({{ site.baseurl }}/docs/create-sparse-grm/) to step 1.

## The data

`extdata/input/pheno_1000samples_survival.txt`, 1,000 samples. The file carries nine
columns; the run uses these four and ignores the rest:

| Column | Role |
|---|---|
| `IND_ID` | sample ID |
| `casecontrol` | event indicator, 1 = event, 0 = censored |
| `AgeOfEventFinal` | event / follow-up time |
| `X` | a covariate |

## The result

`Pvalue_SKAT` is omitted below to keep the table narrow; the file has all 11 columns.

```
Region  Group                    max_MAF  Pvalue    Pvalue_Burden  BETA_Burden  SE_Burden  MAC  Number_rare  Number_ultra_rare
GENE1   lof                      0.001    0.960060  0.960060       0.002336     0.046638     5  0            4
GENE1   lof                      0.01     0.466026  0.315501       0.011009     0.010968    98  3            12
GENE1   missense;lof             0.001    0.950713  0.950713       0.002163     0.034986    10  0            7
GENE1   missense;lof             0.01     0.742115  0.551574       0.006119     0.010278   119  4            17
GENE1   Cauchy                   NA       0.917178  0.909003       NA           NA          NA  NA           NA
GENE2   lof                      0.001    0.388944  0.388944       0.036736     0.042640     5  0            3
...
```

Fourteen rows: two sets x six populated masks, plus one `Cauchy` row each.

Two absences are instructive rather than errors:

- **No `max_MAF=0.0001` rows.** No variant in the example data is that rare.
- **No `GENE3`.** Its only annotation is `intergenic`, which is not among those
  requested in `--annotation_in_groupTest`.

## Files produced

All paths are relative to `extdata/output_tutorial/`.

| File | Contents |
|---|---|
| `sparseGRM_relatednessCutoff_0.125_1000_randomMarkersUsed.sparseGRM.mtx` | the sparse GRM (built, but not used by step 1 here) |
| `…sparseGRM.mtx.sampleIDs.txt` | its sample order |
| `example_survival_groupTest_out.txt` | set-based results, one row per set x mask |
| `example_survival_groupTest_out.txt.singleAssoc.txt` | single-variant results |
| `example_survival_groupTest_out.txt.index` | progress/resume index |
| `example_survival.rda` | the step 1 null model |
| `example_survival.varianceRatio.txt` | the variance ratio |

A reference copy of the expected result is committed at
`tutorial/expected/example_survival_groupTest_out.txt`.

{: .note }
> **The pipeline is not deterministic.** `createSparseGRM.R` samples
> `--numRandomMarkerforSparseKin` markers at random, and step 1 samples
> `--numRandomMarkerforVarianceRatio` markers to estimate the variance ratio; neither
> script exposes a seed option. Two runs on the same machine and image produced P values
> differing by up to 1.8e-07. Set-based P values are therefore reproducible to about
> six significant figures, not bitwise. The script's check is structural for this
> reason.
