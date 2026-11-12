---
title: Set-based test
nav_order: 5
has_children: true
---

# Set-based test

The GATE-GENE workflow: fit a null survival model once per phenotype, then test every
variant set against it.

- [Step 1 — fit the null survival model]({{ site.baseurl }}/docs/set-based-test-step1/)
- [Step 2 — gene-based test]({{ site.baseurl }}/docs/set-based-test-step2/)
- [Example]({{ site.baseurl }}/docs/example/)

Step 1 is the expensive stage. It reads genotypes, but only the pruned `--plinkFile`
marker set used for the GRM and the variance ratio — never the variants being tested.
Step 2 is parallel over chromosomes or group-file chunks and reuses step 1's output
unchanged.
