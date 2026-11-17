---
title: Citation
nav_order: 10
---

# Citation

If you use GATE-GENE, please cite:

**GATE-GENE**
: Wang, Y., Luo, W., Hu, L., Chen, T., Lu, W., Dowd, C., Wen, C., Bi, W., Koyama, S.,
  Surakka, I., Neale, B. M., Daly, M., Natarajan, P., Martin, A. & Zhou, W. Exome-wide
  rare-variant associations vary across disease trajectories. *medRxiv*
  2026.11.14.26368035 (2026). <https://doi.org/10.64898/2026.11.14.26368035>

This is a preprint and has not been peer reviewed. Replace it with the journal
reference once the paper is published.

**GATE**, the underlying frailty survival model
: Dey, R., Zhou, W., Kiiskinen, T., Havulinna, A., Elliott, A., Karjalainen, J.,
  Kurki, M., Qin, A., FinnGen, Lee, S., Palotie, A., Neale, B., Daly, M. & Lin, X.
  Efficient and accurate frailty model approach for genome-wide survival association
  analysis in large-scale biobanks. *Nature Communications* **13**, 5437 (2022).
  <https://doi.org/10.1038/s41467-022-32885-x>

## Version

State the version and how you ran it, so the analysis can be reproduced:

> Gene-based rare-variant time-to-event association testing was performed with
> GATE-GENE (SAIGE package v1.5.2), run from the Docker image
> `wzhou88/saigetest:1.5.2` (digest `sha256:db70848a657f0850ecc193255d46fc7cd15936616e2021a84e4d8496c5549948`).

Quote the **digest**, not only the `1.5.2` tag. The tag does not identify the code
uniquely — see [About]({{ site.baseurl }}/docs/about/) — and a tag can be repointed,
so a digest is what makes the run reproducible. For a source build, give the commit
instead.
