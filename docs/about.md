---
title: About
nav_order: 11
---

# About
{: .no_toc }

1. TOC
{:toc}

GATE-GENE is developed in the [Zhou Lab](https://github.com/ZhouLabGenetics).

| | |
|---|---|
| Source | [ZhouLabGenetics/GATE-GENE](https://github.com/ZhouLabGenetics/GATE-GENE) |
| Docker image | `wzhou88/saigetest:1.5.2` |
| Image digest | `sha256:db70848a657f0850ecc193255d46fc7cd15936616e2021a84e4d8496c5549948` |
| Package version | 1.5.2 (`DESCRIPTION`, dated 2026-03-30) |
| Issues | [GitHub issues](https://github.com/ZhouLabGenetics/GATE-GENE/issues) |

{: .warning }
> `1.5.2` is not a unique code state. A fix to conditional group tests on time-to-event
> traits landed after the version bump without changing the version number. The
> published image is the fixed build; a 1.5.2 compiled from an earlier commit is not.
> See [Step 2]({{ site.baseurl }}/docs/set-based-test-step2/).

## Reporting a problem

Please include:

- the version and how you installed it (image tag, or commit for a source build);
- the complete command;
- the full log, not only the final error;
- for a result problem, the first few lines of the group file and of the genotype
  file's variant IDs — most result problems are ID mismatches.

## Related software

| | |
|---|---|
| [SAIGE / SAIGE-GENE+](https://saigegit.github.io/SAIGE-doc/) | binary and quantitative traits |
| GATE | single-variant time-to-event analysis — the same package, `--traitType=survival` without a group file |

## These pages

Every **pipeline command** and every **quoted output** on these pages was run against
`wzhou88/saigetest:1.5.2`, on the data bundled with the repository, and the quoted text
is copied from the resulting logs. What was executed:

- the full pipeline — `createSparseGRM.R`, step 1, step 2 — via
  `tutorial/run_example.sh`, which is the executable version of these pages; if it and
  a page disagree, the script is correct;
- the sparse-GRM route, both the failing pairing and the working one with
  `--useSparseGRMforVarRatio=TRUE`, plus step 2 with `--sparseGRMFile` and
  `--is_fastTest=TRUE`;
- the PLINK route with `group_new_snpid.txt`;
- `--pcgforUhatforSurvAnalysis=TRUE`, the biobank-scale recommendation, against a
  control run identical except for that one option;
- conditional analysis with `--condition`, including the BGEN and monomorphic-marker
  failures;
- the variant-ID mismatch, to establish that the output file is written 0 bytes.

Not executed, and therefore not verified:

- the pixi source install, the Singularity build and the Docker image rebuild —
  documented from the `pixi.toml` and `docker/Dockerfile` recipes, not from a run;
- the settings on [Biobank-scale runs]({{ site.baseurl }}/docs/biobank-scale/) that are
  specific to real cohort data, `--AlleleOrder=alt-first` for UK Biobank exome PLINK
  among them. That page is guidance drawn from the options' own help text and from
  established practice; only the comparisons it quotes were run here, and those were run
  on the bundled example rather than on a biobank.

---

## Results browser

[gategene.org](https://gategene.org) hosts the PheWAS and multi-state trajectory
analyses in the UK Biobank and the *All of Us* Research Program.

---

## Repository layout

```
R/            R sources
src/          C++ sources (Rcpp / RcppArmadillo), compiled on install
extdata/      entry-point scripts, example data (input/) and example commands
                step1_fitNULLGLMM.R   step2_SPAtests.R
                createSparseGRM.R     step3_LDmat.R
                run_RareEffect.R      calculate_RareEffect_PRS.R
                cmd.sh                worked example commands
                cmd_survival.sh       survival command reference
tutorial/     runnable end-to-end example and its expected output
man/          R documentation
docker/       Dockerfile and build recipe
thirdParty/   vendored dependencies
pixi.toml     pinned environment for the source install
activate.sh   puts the pixi environment on PATH
```
