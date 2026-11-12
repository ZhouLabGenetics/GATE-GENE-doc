---
title: Home
layout: default
nav_order: 1
---

# GATE-GENE
{: .no_toc }

Gene-based rare-variant association analysis for **time-to-event** phenotypes in large,
related cohorts.
{: .fs-5 .fw-300 }

[Get started]({{ site.baseurl }}/docs/installation/){: .btn .btn-primary .mr-2 }
[Browse results](https://gategene.org){: .btn .mr-2 }
[GATE-GENE on GitHub](https://github.com/ZhouLabGenetics/GATE-GENE){: .btn }

---

## What it is for

You have a biobank-scale cohort, an outcome measured as a *time to event* with
censoring, and you want to know which genes carry a rare-variant burden associated with
**whether and when** that event occurs. Both halves matter: a time-to-event outcome is
the pair (follow-up time, event indicator), and the model uses both, which is what
separates it from a binary test that sees only whether. GATE-GENE answers that while handling the three things that break the naive
approach: cryptic relatedness, very low event rates, and variants too rare to test
individually.

GATE-GENE ships inside the same R package as SAIGE and SAIGE-GENE+. It is what you get
when the trait is a time to event: `--traitType=survival`.

## Browse published results

Results produced with GATE-GENE are browsable at
[gategene.org](https://gategene.org), which hosts the PheWAS and multi-state trajectory
analyses in the UK Biobank and the *All of Us* Research Program. These pages document
how to run the method; the browser is where to look up what it found.

## Which tool

| Outcome | Unit | Use |
|---|---|---|
| Binary or quantitative | gene / variant set | [SAIGE-GENE+](https://saigegit.github.io/SAIGE-doc/) |
| Binary or quantitative | single variant | [SAIGE](https://saigegit.github.io/SAIGE-doc/) |
| **Time to event** | single variant | [GATE](https://github.com/weizhou0/GATE) — the same package, `--traitType=survival` without a group file |
| **Time to event** | **gene / variant set** | **GATE-GENE** |

## Two steps

**[Step 1]({{ site.baseurl }}/docs/set-based-test-step1/)** fits a null frailty survival model once per
phenotype, with a GRM for relatedness — sparse at biobank scale, or estimated in full
from a pruned marker set. Its output is reused for every gene.

**[Step 2]({{ site.baseurl }}/docs/set-based-test-step2/)** tests variant sets, producing Burden, SKAT and
SKAT-O statistics per annotation class and MAF cutoff plus a Cauchy combination. It is
parallel over chromosomes or group-file chunks, and is identical in form to
SAIGE-GENE+ — it reads the trait type from the fitted model.

## Three things that catch people out

1. **`--phenoCol` is the event indicator; `--eventTimeCol` is the time.** Swapping them
   runs to completion and means nothing.
2. **`--eventTimeBinSize` is required in 1.5.2.** Without it step 1 aborts after fitting
   the null model. See [FAQ]({{ site.baseurl }}/docs/faq/).
3. **Variant IDs in the group file must match the genotype file exactly.** On a mismatch
   step 2 exits 0 and writes a 0-byte result file. Test that the output is non-empty,
   not the exit code.

## Start here

- [Overview]({{ site.baseurl }}/docs/overview/) — the model and what it assumes
- [Installation]({{ site.baseurl }}/docs/installation/) — Docker, Singularity, source
- [Set-based test]({{ site.baseurl }}/docs/set-based-test/) — the main workflow
- [Input file formats]({{ site.baseurl }}/docs/input-files/)
- [Output]({{ site.baseurl }}/docs/output/)
- [Biobank-scale runs]({{ site.baseurl }}/docs/biobank-scale/) — what changes beyond the toy example
- [FAQ]({{ site.baseurl }}/docs/faq/)
