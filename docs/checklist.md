# Bagisto performance checklist

## Web server (nginx or Apache)
- [ ] Served by nginx or Apache + PHP-FPM (not `php artisan serve`)
- [ ] gzip enabled; brotli enabled if the module is available
- [ ] `Cache-Control: public, immutable`, 1y on `/themes/` build assets
- [ ] Long cache on `/cache/` and `/storage/`
- [ ] Missing `/cache/` files fall through to `index.php` (image resizing works)
- [ ] No blanket `Cache-Control` override on PHP responses
- [ ] HTTP/2 (or HTTP/3) enabled
- [ ] Apache only: `rewrite headers expires deflate` modules enabled

## CDN (Cloudflare)
- [ ] Brotli on
- [ ] Polish on — Lossy + WebP (paid plan only; otherwise optimise images at origin)
- [ ] Static assets show `cf-cache-status: HIT`
- [ ] Origin is a direct public IP, or tunnel latency accepted
- [ ] Analytics beacon disabled (if RUM not needed)
- [ ] Cache purged after deploy/config changes

## Bagisto / Laravel
- [ ] `php artisan optimize` run for production
- [ ] Front-end assets built with `npm run build`
- [ ] redis for cache / session / queue
- [ ] `php artisan queue:work` running
- [ ] Elasticsearch configured for large catalogs
- [ ] `php artisan optimize:clear` after any `.env` / config change

## Measuring
- [ ] Lighthouse run in Incognito (extensions disabled)
- [ ] Tested from a location near the origin
- [ ] Judged on several runs, not one
