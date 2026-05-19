# Cloudflare setup for Bagisto

Cloudflare puts a CDN + edge cache in front of the origin. Configured well it
removes most network latency. Configured poorly (or via a slow tunnel) it can
*add* latency.

## Recommended settings

Cloudflare dashboard:

| Setting | Where | Value |
|---|---|---|
| Brotli | Speed → Optimization → Content | On |
| Polish | Speed → Optimization → Images | Lossy + WebP |
| Auto Minify | (legacy) Speed → Optimization | Optional |
| Early Hints | Speed → Optimization | On |
| HTTP/3 (QUIC) | Network | On |
| Browser Cache TTL | Caching → Configuration | Respect Existing Headers |

`Polish` is the biggest easy win — it re-compresses and serves WebP/AVIF for
every image at the edge, including large banner/hero images.

## Static assets are cached automatically

CSS, JS, fonts and images with a long `Cache-Control` (set by the nginx config)
are cached at the Cloudflare edge — confirm with the `cf-cache-status: HIT`
response header. These never touch the origin after the first request.

## HTML is dynamic

Bagisto sends the HTML document with `Cache-Control: no-cache, private` (cart,
session, CSRF). Cloudflare shows `cf-cache-status: DYNAMIC` — every page load
goes to the origin.

If origin latency is high, this is the main cost. Options:

- **Optional: cache the HTML at the edge for guests.** Add a Cloudflare Cache
  Rule for the homepage / catalog pages: *Eligible for cache*, a short Edge TTL
  (1–5 min), and **bypass when a cart/login cookie is present**. This makes TTFB
  near-zero globally. Test cart/checkout carefully afterwards.

## The Cloudflare Tunnel caveat

If the origin is exposed through **`cloudflared` (Cloudflare Tunnel)** instead of
a public IP, every request travels:

```
visitor → Cloudflare edge → cloudflared tunnel → origin → back
```

The tunnel hop adds TTFB (often 0.3–0.9s). Measured example:

```
Direct origin (nginx)        TTFB ~0.16s
Same origin via the tunnel   TTFB ~0.5–0.9s
```

For a demo box a tunnel is fine. For production performance, prefer a **direct
origin** (public IP behind Cloudflare's proxy) — the edge pulls from the origin
directly, no tunnel hop. Cloudflare Argo Smart Routing also helps if the origin
is far from visitors.

## Remove the analytics beacon (optional)

Cloudflare **Browser Insights / Web Analytics** injects
`static.cloudflareinsights.com/beacon.min.js`. Lighthouse then flags it under
"Use efficient cache lifetimes". Disable it under Analytics if you do not need
Cloudflare's RUM data.

## After any change

Purge the Cloudflare cache (Caching → Configuration → **Purge Everything**),
otherwise the edge keeps serving stale responses — including stale error
responses cached before a fix.
