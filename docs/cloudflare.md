# Cloudflare setup for Bagisto

Cloudflare puts a CDN + edge cache in front of the origin. Configured well it
removes most network latency. Configured poorly (or via a slow tunnel) it can
*add* latency.

## Recommended settings

Cloudflare dashboard (menu paths change occasionally — search the dashboard if a
path differs on your account):

| Setting | Plan | Value |
|---|---|---|
| Brotli | All (on by default) | On |
| Early Hints | All | On |
| HTTP/3 (QUIC) | All | On |
| Browser Cache TTL | All | Respect Existing Headers |
| Polish (image optimisation) | **Pro and above** | Lossy + WebP |

`Polish` re-compresses and serves WebP/AVIF for every image at the edge,
including large banner/hero images — the biggest easy image win. It requires a
**paid Cloudflare plan (Pro or higher)**; it is not available on the Free plan.
On the Free plan, optimise images at the origin instead (correctly sized,
WebP/AVIF source files).

> Cloudflare removed the standalone **Auto Minify** feature in 2024. Minify your
> CSS/JS in the build step instead — Vite (`npm run build`) already does this.

## Static assets are cached automatically

CSS, JS, fonts and images with a long `Cache-Control` (set by the nginx or
Apache config) are cached at the Cloudflare edge — confirm with the
`cf-cache-status: HIT` response header. These never touch the origin after the
first request.

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

The tunnel hop adds TTFB — how much depends on the route between the origin and
Cloudflare. One measured example (same origin, same machine):

```
Direct origin (nginx)        TTFB ~0.16s
Same origin via the tunnel   TTFB ~0.5–0.9s
```

Measure your own setup with `curl` before assuming — see
[`measuring.md`](measuring.md).

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
