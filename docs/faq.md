---
title: FAQ
nav_order: 9
---

# Frequently asked questions
{: .no_toc }

1. TOC
{:toc}

## Step 1 fails with `object 'minEventTime' not found`

Add `--eventTimeBinSize`.

In version 1.5.2 the variable is assigned only inside that option's branch
(`R/SAIGE_fitGLMM_fast.R:1149`) but read unconditionally when the trait is survival
(lines 1575 and 1592). Omitting the option therefore fits the null model successfully
and then aborts while storing metadata.

Nothing incorrect is computed — the run stops rather than producing a wrong answer —
and runs that set the option are unaffected.

## My gene-based test produced an empty file but exited 0

The variant IDs in your group file do not match the genotype file. Step 2 reports this
only as a line in the log:

```
101  markers in 'RegionFile' are not in 'GenoFile'.
Read in  3  region(s) from the group file.
The chunk is empty
```

and still prints `Analysis done!`. Compare an ID from the group file's `var` line with
the genotype file's IDs: `1:1:A:C` and `rs1` are the same variant under different
naming, and will not match.

The set-based output file is created **0 bytes long, without a header**, so test that
it is non-empty rather than trusting the exit code. (`[[ -s file ]]` is the check the
tutorial uses.)

## Which column is the phenotype?

`--phenoCol` is the **event indicator** (1 = event, 0 = censored) and `--eventTimeCol`
is the **time**. They are not interchangeable: swapping them runs to completion and
produces meaningless results.

## `Rscript: command not found` inside the Docker image

Only the four step scripts are on `PATH` in the image — `createSparseGRM.R`,
`step1_fitNULLGLMM.R`, `step2_SPAtests.R`, `step3_LDmat.R`. Call those directly rather
than through `Rscript`.

## `R CMD INSTALL` fails on a source install

Run `pixi run all` first. It builds `libplink2_includes.a`, which `src/Makevars` links
against.

## How do I choose `--eventTimeBinSize`?

It floors event times into bins: `(t - min(t)) %/% binSize + 1`. Match it to the
resolution your phenotype genuinely has. With integer-valued times — whole days, say —
`binSize=1` only relabels them `1, 2, 3, …` and merges no two distinct times, so the
model is unchanged. With continuous times, such as ages in years with decimals, `1`
really does coarsen them to whole years, which may or may not be what you want.

The bin count also sizes the exact-solve path in step 1 — see
[Step 1]({{ site.baseurl }}/docs/set-based-test-step1/).

## My P values differ slightly from a previous run

Expected. `createSparseGRM.R` and step 1 both sample markers at random — for the
relatedness estimate and for the variance ratio respectively — and neither exposes a
seed option, so the variance ratio differs a little between runs and carries into every
P value. On the bundled example two runs differed by up to 1.8e-07. Treat set-based
P values as reproducible to roughly six significant figures.

To remove the variance-ratio contribution specifically, reuse one step 1 fit across all
your step 2 jobs rather than refitting, which is the recommended workflow anyway.

## Do I need a sparse GRM?

Not strictly — step 1 can estimate a full GRM from `--plinkFile`. The sparse GRM is
much cheaper at biobank scale and is computed once per cohort for all phenotypes. Use
`--useSparseGRMtoFitNULL=TRUE` to activate it; without that flag the files are ignored.

If you also intend to pass the sparse GRM to **step 2**, add
`--useSparseGRMforVarRatio=TRUE` to step 1 as well, or step 2 aborts with
`the variance ratio for sparse GRM was not estimated in Step 1`. See
[Create sparse GRM]({{ site.baseurl }}/docs/create-sparse-grm/).

## Can I run single-variant time-to-event tests?

That is GATE rather than GATE-GENE. Step 2 also writes single-variant results for the
variants inside your sets, to `<output>.singleAssoc.txt`, but those are a by-product of
the set test rather than a genome-wide scan.

## Effect direction looks reversed

Check `--AlleleOrder`. It follows the file format, not the cohort: UK Biobank exome
PLINK needs `alt-first`, UK Biobank imputed BGEN needs `ref-first`, and most other
sources are `ref-first`. Nothing in the output flags a wrong choice.

## How should I parallelise?

Split step 2 by chromosome or by chunks of the group file and concatenate. Step 1 runs
once per phenotype, so there is nothing to split there. Extra threads do not speed up
step 2; the Docker image sets `OMP_NUM_THREADS=1` for that reason.

That environment variable does **not** disable `--nThreads` in step 1, which uses its
own threading — a step 1 run given `--nThreads=2` inside the image logs
`2  threads are set to be used`.
