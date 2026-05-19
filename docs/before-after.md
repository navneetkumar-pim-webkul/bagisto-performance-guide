# Before / After — Results

Measured impact of the Bagisto storefront performance + accessibility work
(nginx + CDN tuning in this repo, plus the application-side changes shipped in
the Bagisto codebase).

## Lighthouse scores

| Category | Before | After | Gain |
|---|---|---|---|
| Performance | 78–84 | 94–97 | **+13 to +19** |
| Accessibility | 96 | **100** | +4 |
| Best Practices | 96 | **100** | +4 |
| SEO | 100 | 100 | — |

> "Before" has two baselines: GTmetrix on the original storefront (78) and a
> local Lighthouse run before server tuning (84). "After" is desktop Lighthouse
> on the tuned site; it varies 94–97 run to run (normal measurement variance).

## Core Web Vitals & timing

| Metric | Before | After | Gain |
|---|---|---|---|
| First Contentful Paint | 1.4–1.5s | 0.8–0.9s | ~45% faster |
| Largest Contentful Paint | 2.2s | 1.1–1.3s | ~45% faster |
| Total Blocking Time | 158ms | 0–10ms | ~100% removed |
| Speed Index | 1.9s | 1.2s | ~37% faster |
| Cumulative Layout Shift | 0 | 0 | already perfect |
| TTFB (origin) | 690ms | ~165ms | ~76% faster |

## Infrastructure / payload changes

Not directly scored, but real improvements:

| Area | Before | After |
|---|---|---|
| HTML compression | none | gzip + brotli |
| Static asset cache | `Cache TTL: None` | 1 year, `immutable` |
| Catalog API responses | fresh DB hit every visit | version-keyed cache, edge-cacheable |
| Empty-cart API call | fired on every page view | skipped for guests |
| `Cart::collectTotals()` | ran on every page view | skipped when cart is empty |
| Carousel JSON payload | full (description + gallery) | slim resource (card fields only) |
| LCP hero image | rendered after Vue mount | server-rendered in initial HTML |
| Vue mount | waited for `window.load` | mounts on `DOMContentLoaded` |

## Headline gains

- **Performance: ~78 → ~97** (+~19)
- **LCP cut roughly in half** — 2.2s → 1.1s
- **TBT effectively eliminated** — 158ms → ~0ms
- **3 of 4 categories now 100** — Accessibility, Best Practices, SEO
- **Origin TTFB ~4× faster** — 690ms → ~165ms

## Caveat — GTmetrix grade

GTmetrix can still show a lower grade (e.g. C / ~67%). That is the **measurement
environment**, not the site:

- GTmetrix applies CPU + network throttling and tests from a fixed location.
- If the origin is exposed via a `cloudflared` tunnel, the tunnel adds ~0.3–0.9s
  TTFB (see [`cloudflare.md`](cloudflare.md)).

Desktop Lighthouse on the same site = 94–97. The gap is throttling + tunnel
latency, not a code defect. See [`measuring.md`](measuring.md).
