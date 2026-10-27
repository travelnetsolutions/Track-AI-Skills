---
name: track-api-images
description: Review and fix how a website or app displays, stores and serves property images that come from the Track Hospitality API (developer.trackhs.com), following Track's "The Image Serving Process" guide. Use when a developer integrating with Track has listing or unit photos that are slow, broken, missing, expiring, or blocked; wants to use the Track image proxy (img.trackhs.com) or pick image sizes; is hotlinking S3 image URLs; stores image URLs from the Track API in a database; or asks "how should I serve Track images", "review my Track image handling", "why do my Track photos stop loading after an hour", "resize Track images", or "should I host Track images myself". Works with any language or frontend framework.
---

# Track API image serving review

Audit how a codebase gets images from the Track API to end users and bring it
in line with Track's published guidance. The goal: never send browsers to the
raw S3 file. Either load images through Track's image proxy with a small set
of fixed sizes, or download them into your own image infrastructure.

Source of truth is the public docs. Read [references/track-image-rules.md](references/track-image-rules.md)
before reviewing. It lists every rule this skill checks, each tied to the
published guide. **Do not tell the developer about Track behavior that is not
in that file or on developer.trackhs.com.** In particular, do not say which
Track endpoints return public URLs and which return signed ones: the guide
does not say. Detect it from the URL itself (see rule I2). If you are unsure
how the API behaves, say so and point them to the docs or to Track support.

## Workflow

### 1. Trace the image path

Find where Track image URLs enter the codebase and follow each one to where
it is shown. Typical signals:

- Hosts: `*.s3.amazonaws.com` (the guide's example is `track-pm.s3.amazonaws.com`), `s3.<region>.amazonaws.com`, `img.trackhs.com`
- Signed URL query params: `X-Amz-Signature`, `X-Amz-Expires`, `X-Amz-Credential`, `Signature`, `Expires`
- Code that reads image fields from Track API responses (unit images, unit type images, node images, `coverImage`, `_links.images`)
- Rendering: `<img>`, `srcset`, `<picture>`, CSS `background-image`, framework image components (`next/image`, `NuxtImg`, `Image` in React Native, and so on)
- Storage: columns, cache keys or JSON fields that hold image URLs

Build an inventory: for each image flow, note where the URL comes from, where
it is stored (if at all) and for how long, how it is turned into what the
browser loads, which sizes are requested, and whether it is lazy-loaded.

If you find no Track images, say so and stop.

### 2. Check each flow against the rules

Go through the checklist in [references/track-image-rules.md](references/track-image-rules.md).
For each flow, answer:

1. Does the browser ever load the raw S3 URL directly?
2. Are signed URLs stored and reused after they could have expired?
3. Is the image served through the proxy, or through the developer's own image infrastructure?
4. How many distinct sizes are requested?
5. Are off-screen images lazy-loaded?
6. If self-hosting, is the download and refresh path sound?

### 3. Report

Lead with a short summary (how images reach users today, the biggest risk).
Then one finding per issue, most impactful first:

```
[High] Listing gallery hotlinks raw S3 images
  Where: templates/unit/gallery.html:18
  Rule: I1 (no direct links to S3)
  Why it matters: direct S3 links are unsupported, originals are often full
    size without caching headers, and hotlinking sites may be blocked
    without notice.
  Fix: wrap the URL as https://img.trackhs.com/1200x800/<s3url> using one of
    the site's standard sizes, or serve it from your own image host.
```

Severity guide:

- **High**: raw S3 URLs reaching browsers, signed URLs stored and shown after
  they may have expired, full-size originals on listing or search pages.
- **Medium**: many distinct sizes, proxy URLs that differ from the documented
  format, no lazy loading on large galleries or result grids.
- **Low**: missing `width`/`height` attributes, no `srcset` where it would help,
  self-hosting setups without a refresh path.

Also list what the code already does well, so the developer knows what to keep.

### 4. Offer the fix

Ask before editing. When asked, implement fixes in the project's own language
and frontend stack. Prefer one small helper that builds proxy URLs from a
fixed set of named sizes, used everywhere, over ad-hoc string building. Do not
introduce new frameworks or image services unless the developer asks; if they
want to self-host, point them to the options in the rules file.

Never put API keys in code, logs, or your report. If you see credentials
committed to the repo, flag that as a separate High finding.
