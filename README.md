# GATE-GENE documentation

Source for the GATE-GENE documentation site. The software itself lives in
[ZhouLabGenetics/GATE-GENE](https://github.com/ZhouLabGenetics/GATE-GENE).

Built with [Jekyll](https://jekyllrb.com) and the
[Just the Docs](https://just-the-docs.com) theme, matching
[SAIGE-doc](https://github.com/saigegit/SAIGE-doc).

## Publishing

GitHub Pages builds this site directly from the branch: **Settings > Pages >
Build and deployment > Deploy from a branch**. No Actions workflow is needed —
`jekyll-remote-theme` is supported natively by GitHub Pages.

Before the first publish, set `url` and `baseurl` in `_config.yml` to match the
organisation that owns this repository.

## Local preview

The `Gemfile` here mirrors what GitHub Pages builds: the `github-pages` gem,
which pins **jekyll 3.6.2**. That chain needs Ruby **3.1–3.3** — it does not
run on Ruby 3.4+ (several stdlib gems it requires were removed), and macOS
system Ruby 2.6 is too old.

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>. `_site/` is generated output and is
gitignored.

On a newer Ruby, use the jekyll 4 route under *Status* below instead.

## Status: builds clean

The site **has been built** and the output checked (2026-11-12, Jekyll 4.3,
`just-the-docs` 0.12.0, Ruby 4.0.9):

- all 14 pages generate, at pretty permalinks (`/docs/<page>/`);
- **277 internal links, 0 broken**;
- the sidebar navigation renders in the intended order, with Step 1, Step 2 and
  Example nested under *Set-based test*;
- no unrendered Liquid (`{{` or `{%`) survives anywhere in `_site/`;
- callouts render as callouts — 13 `warning`, 7 `note`.

One caveat on *how* it was built. macOS system Ruby (2.6.10) cannot run Jekyll
at all, and the `github-pages` gem in the `Gemfile` pins jekyll 3.6.2, which
does not run on a current Ruby either. The verification build therefore used
jekyll 4 with the `just-the-docs` **gem** in place of `remote_theme`, via a
separate Gemfile outside this repository. So the content, front matter,
navigation, Liquid, callouts and links are all verified; what is **not** is the
`remote_theme` fetch itself and the exact theme version GitHub Pages resolves.
Walk the sidebar once after the first publish.

To reproduce that build:

```bash
# in a scratch directory, not this repo
cat > Gemfile <<'EOF'
source "https://rubygems.org"
gem "jekyll", "~> 4.3"
gem "just-the-docs", "~> 0.10"
gem "jekyll-seo-tag"
gem "webrick"
EOF
cat > _config.preview.yml <<'EOF'
theme: just-the-docs
remote_theme: null
plugins:
  - jekyll-seo-tag
EOF
bundle install
bundle exec jekyll build -s /path/to/GATE-GENE-doc -d /path/to/GATE-GENE-doc/_site \
  --config /path/to/GATE-GENE-doc/_config.yml,./_config.preview.yml
```

## Editing

One page per file in `docs/`. Each has Just the Docs front matter:

```yaml
---
title: Page title
nav_order: 3
---
```

Sub-pages add `parent: <parent title>`, matching the parent page's `title`
exactly. Changing `nav_order` reorders the sidebar.

Do **not** rely on the theme supplying a layout: Jekyll does not read a theme's
own `_config.yml`, so `layout: default` is set once for every page by the
`defaults:` block in this site's `_config.yml`. A page added without that block
in place renders as bare HTML with no navigation.

`{: .warning }` and `{: .note }` callouts only render as callouts because they
are declared under `callouts:` in `_config.yml`; a new callout class needs an
entry there too.

## Keeping it true

Every command and quoted output on this site was run against
`wzhou88/saigetest:1.5.2`, and the quoted text is copied from the resulting
logs — see the "These pages" section of `docs/about.md` for the full list of
what was executed. `tutorial/run_example.sh` in the code repository is the
executable version of the pipeline pages; when a command changes, re-run it
rather than editing the prose.

The pixi source install, the Singularity build and the Docker image rebuild are
documented from the `pixi.toml` and `docker/Dockerfile` recipes, not from a run;
treat those three as unverified.
