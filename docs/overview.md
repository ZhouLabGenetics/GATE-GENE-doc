---
title: Overview
nav_order: 2
---

# Overview
{: .no_toc }

1. TOC
{:toc}

## The problem

A rare-variant set test on a binary outcome discards *when* an event happened and cannot
represent censoring: a participant event-free after 2 years and one event-free after 20
are treated identically. Where age at onset or time to progression is the quantity of
interest, that is a loss of both information and correctness.

Fitting a Cox model variant by variant in a biobank does not work either. Participants
are related, so observations are not independent; events are rare, so the normal
approximation to the score test is badly calibrated in the tail; and the rarest variants
carry too little information to be tested one at a time.

## The approach

GATE-GENE addresses these in the order they bite.

**Relatedness** — a frailty (random-effect) term with a genetic relationship matrix.
Related participants stay in the analysis rather than being pruned out. A *sparse* GRM
is what makes this affordable at biobank scale, but it is optional: step 1 can estimate
a full GRM from the `--plinkFile` marker set instead.

**Rare events and heavy censoring** — a saddlepoint approximation replaces the normal
approximation for the score statistic. This is what keeps type I error controlled when
only a small fraction of the cohort has the event.

**Variants too rare to test singly** — variants are aggregated into sets, with Burden,
SKAT and SKAT-O statistics computed for each combination of annotation class and MAF
cutoff, then combined across masks with a Cauchy combination.

**Cost** — what is reused, and at which level. The dominant cost is building the
**genetic relationship matrix**, not step 1 or step 2 themselves. That is why the GRM is
computed once per *cohort* by `createSparseGRM.R` (step 0) and reused by every phenotype
thereafter — see [Create sparse GRM]({{ site.baseurl }}/docs/create-sparse-grm/). Within
a phenotype, the null model is then fitted once by step 1 and reused for every gene, so
neither the GRM nor the null fit is repeated per gene. Step 1 does read genotypes, but a
different set from step 2: a pruned common-variant PLINK set, used to build the GRM when
no sparse one is supplied and to sample markers for the variance ratio. The variants
being *tested* are read only in step 2.

## What the design does not give you

Stated plainly, because these are easy to assume:

- **Step 1 fits no genetic effects.** It estimates variance components and covariate
  effects under the null. It reads the `--plinkFile` marker set for the GRM and the
  variance ratio, but no variant effect is estimated there; the variants under test
  enter only in step 2.
- **A set-based P value is not a statement about any single variant.** The
  single-variant output accompanying a run is reported separately and is not what the
  Burden/SKAT/SKAT-O P values refer to.
- **Effect sizes are per variant set under the burden model.** `BETA_Burden` summarises
  the weighted aggregate of the variants in that mask — by default
  `Beta(MAF, 1, 25)`, which up-weights rarer variants, or the group file's own `weight`
  line. It is not an average of per-variant effects.
- **Nothing here corrects for index-event or selection bias.** If the risk set is
  conditioned on a prior event, that conditioning is yours to reason about.

## Relationship to SAIGE

GATE-GENE is distributed in the same R package as SAIGE and SAIGE-GENE+ and shares
their file formats, option names and output layout. If you have run SAIGE-GENE+, the
differences are confined to step 1:

| | SAIGE-GENE+ | GATE-GENE |
|---|---|---|
| `--traitType` | `binary` / `quantitative` | `survival` |
| `--phenoCol` | the phenotype | the **event indicator** (1/0) |
| `--eventTimeCol` | not used | the **event time**, required |
| `--eventTimeBinSize` | not used | required in 1.5.2 |
| Step 2 options | — | **identical** (internals branch on trait type; matters for conditional analysis) |
| Output columns | `MAC_case` / `MAC_control` on a **binary** trait | no case/control columns |
