# Pipeline Architecture Notes

Internal notes for anyone (human or AI) touching these workflows. Not linked from the
main README on purpose.

## Why the pipeline is split into 3 jobs, and why blog-posts builds each product in isolation

A single `hugo build` + `hugo deploy` over the whole site (all ~27 products, ~53k content files,
22 languages) is what conholdate-blog still does in one job, and that pattern is what this repo
used to do too — but at this site's scale it grew to 5-6 hours per run, against GitHub Actions'
**hard 6-hour job timeout**. That's why the pipeline (`ceph-staging-pipeline.yml` /
`aws-production-pipeline.yml`) is split into 3 sequential jobs:

1. **`*-deploy-blog-posts.yml`** — a 27-way matrix, one job per product. Each matrix job
   physically moves every *other* product's folder out of `content/Aspose.Blog/` before running
   Hugo ("ISOLATION STEP"), so Hugo only ever sees one product's content. This is what makes each
   matrix job fast — but it's also why Hugo's own `site.RegularPages` is incomplete during these
   builds (see the prev/next note below).
2. **`*-deploy-home-pages.yml`** — one job, full site (all products present, not isolated), but
   post *bodies* are stripped to front-matter-only before the build ("Fast Content Preparation")
   to avoid OOM. This produces the homepage, pagination, sitemaps, RSS/JSON feeds, CSS/JS — global
   assets that need every product's front matter but not their prose. It deliberately excludes all
   27 product folders (and tag/search) from its sync so it never overwrites what job 1 deployed.
3. **`*-deploy-tags.yml`** — same content-stripping approach, builds and syncs only `tag/*` and
   `category/*` (Hugo taxonomies), which are otherwise the single most expensive part of a full
   build at this scale.

**If you're tempted to "simplify" this back into one job**: don't, without first confirming the
full-site build time is comfortably under 6 hours. It wasn't, which is the entire reason this
structure exists.

## The `--size-only` / fresh-checkout gotcha

Every job does a **fresh `actions/checkout`** and a **fresh `hugo build`**, so every generated
file gets a brand-new local mtime (effectively "now") on every single run — regardless of whether
its content actually changed.

`aws s3 sync`'s *default* comparison is "skip a same-sized file **unless** the local copy is
newer than the S3 copy." Because local mtimes are always fresh, that default effectively means
"always newer, always re-upload everything" for every deploy step, every time — negating the
entire point of the 3-job split above and risking blowing the 6-hour ceiling in the sync step
instead of the build step. This is why every bulk sync in this pipeline uses `--size-only`: it
ignores mtimes and only re-uploads when the byte size actually differs.

**The trade-off**: `--size-only` has a false-negative — if a rebuilt file happens to land on the
exact same byte size as what's already live (common for uniform, auto-generated pages like
paginated post listings), the upload is silently skipped and the stale version stays live
indefinitely. This is what caused the "pagination shows a missing/duplicated post" bug diagnosed
and fixed in August 2026 (currently applied to `staging-ceph-deploy-home-pages.yml` only — see
step "Force-Refresh Homepage & Pagination"; not yet replicated to the AWS/production variants,
pending confirmation it works cleanly on staging first): the fix is **not** to drop `--size-only`
— that reintroduces full-site-reupload-every-run — but to add a small, separate, unconditional
(no `--size-only`) sync scoped *only* to the bounded set of high-churn generated files
(`index.html`, `index.xml`, `index.json`, `p/*`, per language) that change on every run anyway.
The same size-collision risk technically exists for individual post pages and tag pages too, but
is lower-probability there (real prose has much more length entropy than a uniform 10-post
listing) and hasn't been observed — full-site "force refresh" would cost far more than the risk
currently justifies, so it hasn't been applied there.

## Site-wide prev/next post navigation

`aspose-blog/layouts/partials/post_nav_links.html` overrides the PaperMod theme's version. The
theme's version derives prev/next from `site.RegularPages`, which only works when every product's
content is present in the same Hugo build — but the blog-posts isolation step (above) means
`site.RegularPages` only ever contains the *one* product being built at single-post render time.
Left as-is, that makes prev/next silently (and unintentionally) scoped to the same product family.

The fix: `aspose-blog/tools/generate_global_post_order.py` runs *before* the isolation step (while
all products are still present) and writes `data/global_posts/<lang>.json` — a per-language,
newest-first index of every published post's URL/title. Hugo loads `data/*.json` into
`.Site.Data` regardless of which content folders are present, so the isolated build can still
resolve true site-wide neighbors. The override partial looks the current post up in that list
by URL and walks to the adjacent entries. The generator is deliberately best-effort (catches and
logs errors per-file and overall, always exits 0) since this data is cosmetic — a bug in it must
never fail an otherwise-good matrix build.

**Status (2026-08-20)**: applied to `staging-ceph-deploy-blog-posts.yml` only, pending
confirmation before replicating to the AWS/production variants (`production-aws-*`,
`production-ceph-*`, `staging-aws-*`).
