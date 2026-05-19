# Bagisto Performance Guide

Server, nginx and CDN tuning for a fast Bagisto storefront.

This repo covers the **deployment side** of performance — web server, compression,
caching and CDN. Application-level optimisations (Vue mount timing, API caching,
slim resources, accessibility) belong in the Bagisto codebase itself.

---

## TL;DR

A default Bagisto install served by `php artisan serve` will never score well in
Lighthouse/GTmetrix — the dev server sends **no compression** and **no cache
headers**. Most of the "performance problem" is the web server, not Bagisto.

Fix it in this order:

1. Serve through **nginx** with gzip/brotli + cache headers → see [`nginx/bagisto.conf`](nginx/bagisto.conf)
2. Put a **CDN** in front (Cloudflare) → see [`docs/cloudflare.md`](docs/cloudflare.md)
3. Run the standard Laravel/Bagisto **production optimisations**
4. Re-test correctly → see [`docs/measuring.md`](docs/measuring.md)

Quick checklist: [`docs/checklist.md`](docs/checklist.md)

---

## 1. Web server (nginx)

The dev server (`php artisan serve`) is for development only. In production,
serve Bagisto with nginx + PHP-FPM.

Use [`nginx/bagisto.conf`](nginx/bagisto.conf). It provides:

- **gzip** compression (and brotli, if the module is available)
- **`Cache-Control: public, immutable`, 1 year** for hashed Vite build assets
  (`/themes/...`)
- **long cache** for generated images (`/cache/...`) and uploads (`/storage/...`)
- **on-demand image resizing left intact** — see the warning below

### Critical: do not break Bagisto's image cache

Bagisto resizes images on demand. A request for `/cache/medium/...` that has no
file yet must reach `index.php` so Laravel can generate it.

```nginx
# WRONG - returns 404 for every not-yet-generated image
location ^~ /cache/ { try_files $uri =404; }

# CORRECT - generated files served directly, misses fall through to Laravel
location ^~ /cache/ { try_files $uri /index.php?$query_string; }
```

### Critical: do not force `Cache-Control` in the PHP block

Bagisto sets its own headers per response — HTML is `no-cache`, resized images
are long-lived `public`. A blanket `add_header Cache-Control "no-cache"` in the
`location ~ \.php$` block makes **every resized image uncacheable**. Leave the
app's headers alone.

---

## 2. CDN (Cloudflare)

A CDN serves static assets from an edge near the visitor and removes most of the
round-trip latency. See [`docs/cloudflare.md`](docs/cloudflare.md) for:

- Enabling Brotli, Polish (image optimisation), Auto Minify
- Caching static assets at the edge
- The **Cloudflare Tunnel latency caveat** — a tunnel adds TTFB; a direct origin
  is faster
- Optional HTML edge-caching for guests

---

## 3. Bagisto / Laravel production optimisations

```bash
# Cache config, routes, events, views
php artisan optimize

# Build front-end assets for production (run in each theme package)
npm run build

# Use a fast cache/session/queue driver (redis recommended)
# .env:  CACHE_STORE=redis  SESSION_DRIVER=redis  QUEUE_CONNECTION=redis

# Process queues with a worker (emails, indexing, search terms)
php artisan queue:work

# Elasticsearch for catalog search on large catalogs
```

After any config or `.env` change: `php artisan optimize:clear` then
`php artisan optimize`.

> Note: a stale `bootstrap/cache/config.php` / `events.php` makes the app ignore
> `.env` and newly registered listeners. If something "isn't taking effect",
> run `php artisan optimize:clear` first.

---

## 4. Measuring correctly

Tools disagree because they test under different conditions. See
[`docs/measuring.md`](docs/measuring.md). Key points:

- **Run Lighthouse in an Incognito window** — browser extensions inject scripts
  that corrupt the Performance and Best Practices scores.
- Lighthouse scores are **weighted**: TBT 30%, LCP 25%, CLS 25%, FCP 10%, SI 10%.
- Scores vary run-to-run (network jitter, CPU). A frozen 100 does not exist.
- GTmetrix throttles CPU/network and tests from a fixed location — its grade
  will look lower than a local desktop Lighthouse run of the same site.

---

## What this is not

This repo is **deployment guidance**. It does not patch Bagisto core. The
application-side performance work (rendering, API caching, payload size,
accessibility) is done in the Bagisto codebase via normal pull requests.
