# Pipeline Architecture Notes

Internal notes for anyone (human or AI) touching these workflows. Not linked from the
main README on purpose.

## Why the pipeline is split into 3 jobs, and why blog-posts builds each product in isolation

A single `hugo build` + `hugo deploy` over the whole site (all ~27 products, ~53k content files,
22 languages) is what conholdate-blog still does in one job, and that pattern is what this repo
used to do too — but at this site's scale it grew to 5-6 hours per run, against GitHub Actions'
**hard 6-hour job timeout**. That's why the pipelines (`ceph-staging-pipeline.yml`,
`ceph-production-pipeline.yml`, `aws-production-pipeline.yml`) are split into 3 sequential jobs:

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
3. **`*-deploy-tags.yml`** — same content-stripping approach, syncs only `tag/*` and `search/*`.
   Taxonomies are otherwise the single most expensive part of a full build at this scale. Despite
   the step name `Build Tags Only`, it runs the same Hugo command as the home page build, so both
   stages build the full site, taxonomies included, and differ only in what they upload.

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
and fixed in August 2026 (see step "Force-Refresh Homepage & Pagination" in the
`*-deploy-home-pages.yml` workflows): the fix is **not** to drop `--size-only`
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

**Status (2026-09-09)**: applied to all four `*-deploy-blog-posts.yml` workflows.

## When a scheduled run skips, and why a run's own "success" isn't trusted

Scheduled pipeline runs (`aws-production-pipeline.yml`, `ceph-production-pipeline.yml`) start
with a `check-changes` job, so nothing deploys when there's nothing new. Manual runs skip that job
and always deploy. `ceph-staging-pipeline.yml` has the same job but its schedule is commented out,
so staging only deploys when started by hand.

`check-changes` finds the newest run of the same pipeline that attempted a deploy (job1 not
skipped). If that run didn't finish job3 successfully, this run deploys, even with no new commits.
If it did, this run deploys only when `Aspose/aspose-blog` has new commits or a new push since that
run was created.

Two details there are deliberate. The baseline is the run's `created_at`, because re-running a
failed job moves `run_started_at` forward while job1's deploy stays at the original time. And the
check also compares the repo's `pushed_at`, because a commit made before the baseline but pushed
after it has an old date, so the commit list alone misses it.

It can't simply use the last run marked "success", because two kinds of run report success without
a full deploy:

- A run that skipped for no changes.
- Before November 2026, every manual run. Job2 and job3 had no `if`, and GitHub skips a job when
  anything up its `needs` chain was skipped, which `check-changes` always is on a manual run. So
  manual runs deployed blog posts only and still reported success. The next scheduled run then
  treated that as its baseline and could skip, leaving home pages and tags stale.

**If you add a job to a pipeline**, give it an explicit `if` on the previous job's result, like
job2 and job3 have. If it becomes the new last required stage, update the `job3` check in
`check-changes` too.

**Running stages on their own**: the standalone `AWS Prod-2` / `AWS Prod-3` style workflows aren't
seen by `check-changes`, so the next scheduled run may repeat that work. To retry a failed stage,
prefer **Re-run failed jobs** on the pipeline run: the check reads a run's latest attempt, so a
successful retry counts as a complete deploy. Anything pushed since the run was created still
triggers the next scheduled run. A re-run uses the workflow files from the run's original commit,
so it doesn't pick up a workflow fix pushed since; start a new run for that.

## Memory logging in the home page and tag builds

`Build Skeleton Site` and `Build Tags Only` log total memory and Hugo's own usage every 60 seconds
(`RAM Check` lines). The monitor runs inside the build step on purpose: a background process in its
own step stops being logged once that step ends, so it can't show what happened during a build.

These two builds are the most memory-hungry part of the pipeline. The three active pipelines run
them with 20 GB of swap and `GOMEMLIMIT: 30GiB`, which makes Go collect garbage harder as it nears
the runner's 36 GB of RAM plus swap. The `staging-aws-*` workflows, which no pipeline uses, still
have 10 GB of swap and no limit.

When a stage fails, `failure-report.yml` downloads the failed jobs' logs and writes a memory
summary to the run's summary page: peak swap, peak Hugo memory, lowest available RAM, the last
readings and the last error lines. It also attaches the logs to the run for 14 days. It runs as its
own job on a fresh runner, so it still works when the failed job's runner died. To report on an
earlier failed run, start `Failure Report` by hand with that run's ID. All three pipelines call it
automatically.

`--gc` doesn't reduce memory use: it only removes unused cache files after the build.
