# Measuring performance correctly

Different tools give different numbers for the *same site*. That is expected —
they test under different conditions. Know what you are reading.

## Run Lighthouse in Incognito

Browser extensions (ad blockers, profilers, AI assistants, recorders) inject
their own scripts into every page. Lighthouse then blames the site:

- "Reduce unused JavaScript" — often hundreds of KiB that are 100% extension code
- "Minimize main-thread work" — extension execution time
- console errors → lowers Best Practices

**Always run Lighthouse in an Incognito window** (extensions disabled), or use
the Lighthouse CLI. Otherwise the score is measuring the browser, not the site.

## How the Performance score is built

Lighthouse Performance is a weighted blend of 5 metrics:

| Metric | Weight |
|---|---|
| Total Blocking Time (TBT) | 30% |
| Largest Contentful Paint (LCP) | 25% |
| Cumulative Layout Shift (CLS) | 25% |
| First Contentful Paint (FCP) | 10% |
| Speed Index (SI) | 10% |

The diagnostics ("reduce unused CSS", "DOM size", …) do **not** score directly —
they only matter through the metric they influence.

Each metric scores on a curve: a metric scores `1.00` only near the *ideal*, not
just "good". Example (desktop): LCP `1.00` needs ≈ 0.8s; LCP 1.2s ≈ 0.92. A site
can be genuinely fast and still sit at 95–98.

## Scores vary between runs

The same site, same tool, will swing several points run to run — network jitter,
CPU contention, CDN cache state. A permanently frozen 100 is not realistic.
Judge by a few runs and by the raw metric values, not a single number.

## Lighthouse vs GTmetrix

GTmetrix runs Lighthouse internally but adds its own **CPU + network throttling**
and tests from a **fixed server location**. A site that scores 97 on a local
desktop Lighthouse run can show a much lower GTmetrix grade — that gap is the
throttling and the distance from GTmetrix's test node to the origin, not a
code defect.

To compare fairly: test from a location near the origin, with the same
throttling, repeatedly.

## Useful commands

```bash
# Lighthouse CLI, desktop preset, no extensions
npx lighthouse https://example.com/ --preset=desktop \
  --chrome-flags="--headless" --view

# TTFB / latency breakdown
curl -s -o /dev/null -w "dns=%{time_namelookup} connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n" https://example.com/

# Check compression + cache headers
curl -sI -H 'Accept-Encoding: gzip,br' https://example.com/ | grep -i -E 'content-encoding|cache-control|cf-cache-status'
```
