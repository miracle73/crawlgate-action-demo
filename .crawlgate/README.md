# crawlgate

SEO breakage is a deploy bug that nobody writes tests for. A refactor points canonicals at staging, `og:image` disappears from three templates, a route starts returning 200 where it should 404. Nobody notices for weeks, until traffic drops or someone shares a link and gets a blank grey card.

crawlgate catches this in the pull request. It crawls the preview deployment, extracts the SEO surface of every page, diffs it against a baseline committed to the repo, and fails the check when something breaks.

## Before and after: tipmaster.net

The first real run was against `https://tipmaster.net/de` (2026-10-06, `--depth 1`, no config):

```
$ crawlgate check https://tipmaster.net/de --depth 1
## crawlgate: FAIL for https://tipmaster.net/de

🛑 BLOCK 3 · ⚠️ WARN 23 · ℹ️ INFO 13 · known (baselined) 0 · pages 13

### 🛑 BLOCK (3)
| noindex_production_route | https://tipmaster.net/daily/de/signin | page is noindex `noindex, nofollow` |
| noindex_production_route | https://tipmaster.net/daily/de/signup | page is noindex `noindex, nofollow` |
...
### ⚠️ WARN (23)
| og_image_too_small | https://tipmaster.net/de | og:image is 1200x624, need >=1200x630 `https://tipmaster.net/img/og/daily.jpg` |
| og_image_too_small | https://tipmaster.net/en | og:image is 1200x624, need >=1200x630 ... |
  ... the same on es, fr, id, it, nl, pt, th, tr and the signin/signup pages
| hreflang_not_reciprocal | https://tipmaster.net/daily/de/signin | `de` target does not link back `de https://tipmaster.net/` |
| description_length | https://tipmaster.net/fr | meta description is 161 chars (50-160) |
$ echo $?
1
```

What this run shows:

- **The site now ships an `og:image`.** It's 1200x624, six pixels short of the 1200x630 minimum, on every locale. Facebook and WhatsApp may crop it or fall back to the small card. The finding was reported once per page, on all 10 locale homepages and the auth pages.
- **Sign-in and sign-up pages are `noindex`.** That is intentional, so it goes in config. The gate blocked because it can't tell an intentional noindex from an accidental one.
- **The auth pages' hreflang points at locale homepages, which don't link back.** That is a real reciprocity bug.

After telling crawlgate which routes are meant to be noindex (see [`crawlgate.example.toml`](crawlgate.example.toml)):

```
$ crawlgate check https://tipmaster.net/de --config crawlgate.example.toml --depth 1
## crawlgate: WARN for https://tipmaster.net/de
🛑 BLOCK 0 · ⚠️ WARN 37 · ℹ️ INFO 2
$ echo $?
0
```

The build passes and the warnings show in the PR comment. Run `crawlgate baseline update` to accept them, and from then on only new regressions show up.

A page with no `og:image` at all (the blank-grey-box case, from the `no-og` fixture in `tests/`):

```
| og_image_missing      | http://127.0.0.1:…/no-og | missing og:image |
| twitter_image_missing | http://127.0.0.1:…/no-og | missing twitter:image (and no og:image fallback) |
```

## Usage

```bash
uv sync && uv run playwright install chromium

crawlgate crawl https://example.com --out report.json            # extract + findings, no gate
crawlgate baseline update https://example.com                     # writes crawlgate.baseline.json; commit it
crawlgate check https://pr-42.preview.example.com --baseline crawlgate.baseline.json --fail-on block
crawlgate report report.json --format html
```

`check` writes `report.json` and `report.md`, which is the PR comment. It exits 1 when the gate fails.

## The gate

- **Three severities: BLOCK, WARN, INFO.** They are hard-coded in [`severity.py`](src/crawlgate/severity.py). They are not set in config and not decided by a model.
- **Config can raise a severity but never lower it.** `noindex_in_sitemap = "info"` is ignored and logged as `severity downgrade ignored`. A prompt is a request, a check is a guarantee.
- **The baseline handles intentional change.** Findings already in the baseline are marked `known`, and known WARN/INFO findings don't fail the build. BLOCK findings always fail it, baseline or not.
- **Baselines compare paths, not hosts.** A baseline taken from production can gate a preview on `pr-42.vercel.app`.
- **Reports are byte-stable.** URLs are sorted, the crawl runs BFS level by level in a fixed order, JSON keys are sorted, and there are no timestamps or timings. Two runs against the same site produce identical bytes. Lighthouse metric values go to the log, never into the report.

| BLOCK | WARN |
|---|---|
| noindex on a production route | missing og:image, og:image relative / <1200x630 / not 200 / not image/* |
| canonical on a non-production host | meta description outside 50-160 chars |
| canonical pointing at a noindex page | redirect chain > 1 hop |
| sitemap URL returns 404/5xx | internal link to 404 |
| noindex page listed in sitemap.xml | hreflang not reciprocal / incomplete |
| hreflang target is dead | orphan page (in sitemap, unreachable) |
| status regressed 2xx -> 4xx/5xx vs baseline | SEO tags only present after hydration |
| robots.txt disallows everything | JSON-LD invalid or missing required props |

The full table is in [`severity.py`](src/crawlgate/severity.py).

## What it extracts

For each URL: title, meta description, canonical, meta robots, X-Robots-Tag, the hreflang set, `og:*`, `twitter:*`, JSON-LD blocks (parsed, `@type` checked, required properties), HTTP status, redirect chain and final URL. It also probes the og:image itself: status, content type, real pixel dimensions, and the width/height/alt meta tags. Each page is rendered in Chromium, and any SEO field that exists only after hydration becomes a finding, because WhatsApp, Facebook and many bots never run JavaScript.

Across the whole site it checks robots.txt, sitemap.xml (including sitemap indexes), orphan pages, internal links that hit 404s or redirects, hreflang reciprocity, and indexability conflicts. It can also check Lighthouse Core Web Vitals against a budget (`[crawlgate.lighthouse]`, needs `npm i -g lighthouse`).

The crawler works breadth-first in a deterministic order. It uses a bounded worker pool, token-bucket rate limiting, robots.txt, and exponential-backoff retries on 429/5xx and network errors. Depth, page count and include/exclude regexes are configurable.

## GitHub Action

```yaml
on: pull_request
permissions: { contents: read, pull-requests: write }
jobs:
  seo-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: your-org/crawlgate@v0
        with:
          url: ${{ steps.preview.outputs.url }}
          baseline: crawlgate.baseline.json
          fail-on: block
```

The action posts (and updates) one PR comment with changes grouped by severity, writes the job summary, and fails the check on a block. See [`.github/workflows/crawlgate.yml`](.github/workflows/crawlgate.yml).

## Development

```bash
uv run pytest -q
```

The fixtures serve a real local site: a known-good page, a page missing og:image, a noindex page in the sitemap, a broken hreflang set, and a 3-hop redirect chain.
