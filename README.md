# Bug Report: Cloudflare Integration Does Not Cache Hashed Assets

**Affected package:** `@astrojs/cloudflare`  
**Severity:** Medium — Silent performance regression on every Cloudflare deployment  
**Branch:** `fix-cf-asset-cache`

---

## Summary

When deploying an Astro site to Cloudflare Pages or Workers, the hashed static assets under `/_astro/*` (JavaScript bundles, CSS, images, and other build outputs) are served **without a `Cache-Control` header** that would permit long-term browser and edge caching. Every visitor re-downloads the full asset bundle on each page load or after each deploy, even when the content is byte-for-byte identical.

---

## Background

Astro's build pipeline generates content-addressed asset filenames — e.g., `/_astro/main.Ck3f8xpP.js`. The hash in the filename is derived from the file's content, so the URL only changes when the content changes. This property makes these assets safe for the most aggressive browser cache policy: `Cache-Control: public, max-age=31536000, immutable`. Once a browser downloads `main.Ck3f8xpP.js`, it never needs to re-validate it because any future change to the JS will produce a *new* URL.

Cloudflare applies HTTP response headers to hosted assets via a `_headers` file placed in the output directory. Without a rule in that file covering `/_astro/*`, Cloudflare falls back to default caching behavior, which is conservative and does not grant browsers permission to cache assets indefinitely.

---

## Observed Behavior

1. A developer builds an Astro site with `@astrojs/cloudflare` and deploys to Cloudflare Pages/Workers.
2. The `_headers` file in the output directory contains **no rule** for the assets path (`/_astro/*` or `/<base>/_astro/*` when a base path is configured).
3. Cloudflare serves `/_astro/*.js`, `/_astro/*.css`, etc. without a `Cache-Control` response header (or with an unhelpful default).
4. Browsers re-fetch all assets on every page load instead of serving them from cache.
5. Cloudflare's own edge caches may not store these assets across deploys, increasing origin load.

### With a custom base path

The bug is compounded when `base` is set in `astro.config.*` (e.g., `base: '/blog'`). In that case the assets directory is `/<base>/_astro/*`. The integration had no awareness of the base path when constructing the `_headers` rule, so even a manually written rule for `/_astro/*` would fail to match the actual asset URLs.

---

## Root Cause

The `@astrojs/cloudflare` integration's `astro:build:done` hook assembled the `_redirects` file and other Cloudflare-specific output artifacts, but contained **no logic to write cache headers for hashed assets**. The integration was responsible for all other `_headers` manipulation (e.g., in related PRs for KV namespaces and session injection) but this particular concern was never addressed.

The omission had a secondary consequence: if a user had already written their own `_headers` file with a `Cache-Control` rule that matched the assets path (e.g., a global `/*` rule with `Cache-Control: no-cache`), Cloudflare's documented behavior is to **merge headers from all matching rules with a comma**. This means a future attempt to add the immutable rule would produce contradictory output like:

```
Cache-Control: no-cache, public, max-age=31536000, immutable
```

This is semantically invalid — `no-cache` and `immutable` are mutually exclusive directives — and browsers handle it inconsistently.

---

## Impact

| Scenario | Impact |
|---|---|
| New Cloudflare deployment, no custom `_headers` | All hashed assets re-fetched on every page load; no browser or CDN caching benefit |
| Re-deployment with unchanged assets | Assets re-downloaded even though the URL (and content) has not changed |
| Site with `base` path configured | Cache headers for base-prefixed assets paths (`/<base>/_astro/*`) never injected |
| User has a `Cache-Control` rule in `_headers` matching `/*` | Merging a new rule would produce contradictory comma-joined cache directives |
| `build.assetsPrefix` set (assets on a CDN/other origin) | No specific impact from this missing header, but the absence of any guard meant future injection logic could incorrectly target a path not served by Cloudflare at all |

---

## Reproduction Steps

1. Create a new Astro project targeting Cloudflare:
   ```sh
   npm create astro@latest -- --template minimal
   npm install @astrojs/cloudflare
   ```
2. Configure `astro.config.mjs`:
   ```js
   import cloudflare from '@astrojs/cloudflare';
   export default defineConfig({
     adapter: cloudflare(),
     output: 'static',
   });
   ```
3. Run `astro build`.
4. Inspect `dist/_headers` — the file either does not exist or contains no rule for `/_astro/*`.
5. Deploy to Cloudflare Pages and open DevTools → Network. Observe that `/_astro/*.js` responses carry no `Cache-Control: immutable` header; repeat page loads re-fetch all assets.

### Variant: base path

Add `base: '/blog'` to `astro.config.mjs`, repeat the above, and verify that `/blog/_astro/*` also has no cache rule.

### Variant: conflicting user `_headers`

Add a `public/_headers` file:
```
/*
  Cache-Control: no-cache
```

Build the project. If the integration naively appended a second `Cache-Control` line for `/_astro/*`, Cloudflare would merge both matching rules and produce `Cache-Control: no-cache, public, max-age=31536000, immutable` on the asset responses.

---

## Expected Behavior

After build, `dist/_headers` (or `dist/client/_headers` for SSR output) should contain:

```
/_astro/*
  Cache-Control: public, max-age=31536000, immutable
```

Or, when `base: '/blog'`:

```
/blog/_astro/*
  Cache-Control: public, max-age=31536000, immutable
```

The rule should appear **before** any user-defined rules so that it is readable first in merge order. If the user's existing `_headers` already sets `Cache-Control` on any rule whose URL pattern would match the assets path, the injection must be **skipped entirely** to avoid Cloudflare's comma-merge producing contradictory directives.

When `build.assetsPrefix` is configured, the assets are not served by Cloudflare at all, so no injection should occur.

---

## Notes

- Cloudflare's `_headers` format is documented at [developers.cloudflare.com/pages/configuration/headers](https://developers.cloudflare.com/pages/configuration/headers/). Key behavior: when multiple rules match a single request, all matching rules' headers are merged (comma-joined for duplicates).
- The pattern-matching logic needed to detect pre-existing `Cache-Control` coverage must implement Cloudflare's splat (`*`) and named-placeholder (`:name`) syntax — a plain substring check is insufficient.
- This bug affects both `output: 'static'` and `output: 'server'` / `output: 'hybrid'` builds (which write to `dist/client/`).
