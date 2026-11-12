---
title: Installation
nav_order: 3
---

# Installation
{: .no_toc }

1. TOC
{:toc}

## Docker

The quickest route, and the one these pages are verified against.

```bash
docker pull wzhou88/saigetest:1.5.2
docker run --rm wzhou88/saigetest:1.5.2 step1_fitNULLGLMM.R --help
```

For a run you will publish, pull by digest instead, so the exact code is pinned — the
`1.5.2` tag does not identify it uniquely
(see [About]({{ site.baseurl }}/docs/about/)):

```bash
docker pull wzhou88/saigetest@sha256:db70848a657f0850ecc193255d46fc7cd15936616e2021a84e4d8496c5549948
```

Four entry points are installed on `PATH` as executables, so they are invoked directly
rather than through `Rscript`:

```
createSparseGRM.R   step1_fitNULLGLMM.R   step2_SPAtests.R   step3_LDmat.R
```

`step3_LDmat.R` writes LD matrices for downstream meta-analysis and is not part of the
association workflow on these pages; see the
[SAIGE documentation](https://saigegit.github.io/SAIGE-doc/) for it.

{: .warning }
> Plain `Rscript` is **not** on `PATH` in the image. Only the four scripts above are. A
> wrapper script that calls `Rscript` directly will fail with
> `Rscript: command not found`. It does exist, at
> `/app/.pixi/envs/default/bin/Rscript`, if you need it for something else.

The four scripts are executable because of their shebang:

```
#!/usr/bin/env -S pixi run --manifest-path /app/pixi.toml Rscript
```

That path is hardcoded to `/app`, where the image keeps the repository. In a source
checkout `/app` does not exist, so `./step1_fitNULLGLMM.R` fails there — which is why
the source instructions call `pixi run Rscript extdata/step1_fitNULLGLMM.R` instead of
executing the script directly.

To analyse your own data, mount it:

```bash
docker run --rm -v /path/to/data:/data -w /data \
  wzhou88/saigetest:1.5.2 step1_fitNULLGLMM.R --phenoFile=/data/pheno.txt ...
```

The image is built for `linux/amd64` only. On Apple Silicon it still runs, under
emulation — Docker selects the platform itself and prints

```
WARNING: The requested image's platform (linux/amd64) does not match the detected
host platform (linux/arm64/v8) and no specific platform was requested
```

Passing `--platform linux/amd64` silences that warning; it does not change the result,
and it does not avoid the emulation, which is several times slower than native Linux.

## Singularity / Apptainer

For HPC systems without Docker:

```bash
singularity build gate-gene_1.5.2.sif docker://wzhou88/saigetest:1.5.2
singularity exec gate-gene_1.5.2.sif step1_fitNULLGLMM.R --help
```

Bind your working directory with `-B /path/to/data:/data` as usual.

## Source install with pixi

```bash
git clone https://github.com/ZhouLabGenetics/GATE-GENE.git
cd GATE-GENE
pixi install
pixi run all
pixi run R CMD INSTALL .
pixi run Rscript extdata/step1_fitNULLGLMM.R --help
```

{: .warning }
> `pixi run all` is **not optional**. It runs the `install-plink2` task, which downloads
> plink-ng v2.0.0-a.6.16 and builds `libplink2_includes.a`. `src/Makevars` links against
> that library, so `R CMD INSTALL` fails without it.

`pixi.toml` pins the dependency set and declares `activate.sh` as an activation script,
which is why the commands are prefixed with `pixi run` rather than sourcing it by hand.

## Rebuilding the image

From the repository root:

```bash
docker build -t gate-gene:1.5.2 -f docker/Dockerfile .
```

The build installs the environment with pixi, compiles plink-ng, runs `R CMD INSTALL`,
and moves the four scripts to `/usr/local/bin`, verifying each responds to `--help`.
`OMP_NUM_THREADS=1` is set in the image deliberately: additional threads do not speed up
step 2.

## Verifying the installation

Run the bundled example, which exercises all three stages end to end:

```bash
docker run --rm -v "$PWD":/gate -w /gate \
  wzhou88/saigetest:1.5.2 bash tutorial/run_example.sh
```

It should finish with `TUTORIAL PASSED`. See [Example]({{ site.baseurl }}/docs/example/).
