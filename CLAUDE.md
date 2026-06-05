# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal brand site for Philipp Jahoda (CTO & Co-Founder at Ahoi Kapptn!). Vanilla HTML/CSS/JS, **no build step, no dependencies, no framework**. Three standalone pages: `index.html` (the site), `imprint.html` and `privacy.html` (legal). Deployed to GitHub Pages at philippjahoda.com.

## Commands

```bash
# Develop — open index.html directly, or serve locally (needed for absolute /assets paths)
python3 -m http.server 8000   # then open http://localhost:8000

# Deploy — there is no build/CI step. Pushing to `main` publishes via GitHub Pages.
git push origin main

# Optimize a webp asset (cwebp from the `webp` Homebrew formula; ffmpeg here lacks libwebp)
cwebp -q 82 -resize <W> <H> input.png -o assets/<name>.webp
```

There is no test suite, linter, or package manager.

## Architecture

**Each HTML page is fully self-contained.** Styles live in an inline `<style>` and scripts in inline `<script>` blocks within each file — there is **no shared stylesheet or JS file**. The same base styles and fonts are duplicated across the three pages; a change to shared-looking styling must be made in each page that needs it.

**Design system** lives in the `:root` custom-property block at the top of each file's `<style>` (colors `--cyan`/`--indigo`/`--muted`/etc., `--radius`, `--maxw`, `--shadow`). It's a dark, glassmorphism theme — prefer these tokens over hard-coded values. Layout is responsive via `@media` breakpoints (~980/860/700/480px) and honors `prefers-reduced-motion`.

**`index.html` layout:** the `<body>` is divided by `<!-- ===== SECTION ===== -->` comment banners (NAV, HERO, LEADERSHIP/WHAT I DO, AI, SPEAKING, OPEN SOURCE, TRACK RECORD, GALLERY, CONTACT, FOOTER). Two `<script>` blocks at the bottom: (1) a smooth-scroll handler that intercepts `a[href^="#"]` anchors and offsets for the sticky nav; (2) the gallery carousel.

**Gallery** is the only non-trivial JS. Its data is inlined as a `<script type="application/json" id="gallery-data">` array (objects: `src`, `title`, `subtitle`) and parsed locally — *not* fetched, to avoid a request chain on load. To edit the gallery, edit that JSON block. The carousel duplicates tiles for a seamless marquee and defers the animation start until the section is visible AND all images (including off-screen duplicates) have loaded — this works around a Safari bug where the `-50%` transform is baked against the track's width at animation start. Don't "simplify" that deferral away.

**JS style:** ES5-flavored vanilla — `var`, function expressions, IIFEs, feature-detection (`IntersectionObserver`), no transpilation. Match it.

## Conventions

- **Inline `style="…"` is the accepted idiom** for one-off, single-element tweaks (e.g. section padding, a single paragraph's spacing). Reusable styling goes in the `<style>` block as a class.
- Beware similarly-named classes with different rules: `.section-sub` (global) vs `.contact-card .sub` (container-scoped, different size/spacing). Check which one an element uses before restyling.
- When matching "the old look" of something, check `git log -p`/blame — recent refactors have swapped classes, which silently changes spacing/sizing.

## Performance constraints (this site is tuned for PageSpeed; keep it that way)

- Fonts are **self-hosted** in `assets/fonts/` (no third-party Google requests) with `font-display: swap` and `*-Fallback` faces using `size-adjust` to prevent layout shift. Both woff2 files are `<link rel="preload" as="font" crossorigin>`-ed in `<head>`.
- The hero portrait (`assets/portrait-hero.webp`) is the LCP element: preloaded with `fetchpriority="high"`. Keep image intrinsic dimensions close to their largest rendered size (the `<img width/height>` should match the file's actual pixels).
- **Caching cannot be fixed in-repo.** GitHub Pages serves everything with a hardcoded `cache-control: max-age=600` and ignores `_headers`/`.htaccess`. PageSpeed's "efficient cache lifetimes" finding is only addressable via infrastructure (Cloudflare proxy with a cache rule, or migrating to Netlify/Cloudflare Pages/Vercel) — don't propose repo-level cache-header fixes.

## Deploy notes

`CNAME` pins the custom domain (philippjahoda.com) — don't delete it. Asset paths are **absolute** (`/assets/...`), so previewing requires serving from the repo root, not opening files via `file://` for the asset-dependent parts.

`_config.yml` exists only to set Jekyll's `exclude:` — it keeps `CLAUDE.md`, `README.md`, `pagespeed_report/` and `vercel.json` in git but **out of the published site** (they returned HTTP 200 before). Don't add a `.nojekyll` file: that disables Jekyll and would start serving those excluded files (and any dotfiles) raw.

## Security headers (infra-bound, like caching)

GitHub Pages can't set response headers in-repo, so the site currently ships **no** HSTS/CSP/X-Frame-Options/X-Content-Type-Options/Referrer-Policy/Permissions-Policy. Impact is low (fully static, no cookies/forms/auth), but the fix is the same infra move as the caching one. Two ready-to-use configs are committed for a future migration and are **inert on GitHub Pages today**:

- `vercel.json` — security headers + `/assets/*` cache rule for **Vercel** (the planned target). Vercel reads it at deploy; here it's `exclude:`-d so it isn't served.
- `_headers` — same policy for **Netlify / Cloudflare Pages**. Ignored by both GitHub Pages and Vercel; auto-unpublished by its leading underscore.

The CSP needs `'unsafe-inline'` for `script-src`/`style-src` because all CSS/JS (and many `style="…"` attrs) are inline — tightening it would mean externalizing scripts or adding hashes. If you instead front Pages with a **Cloudflare proxy**, set the same headers there via a Transform Rule and it also fixes the cache-TTL finding. Separately: GitHub domain verification is **not** enabled (latent takeover guard) — add it at github.com/settings/pages.
