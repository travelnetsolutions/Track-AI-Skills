# Track image serving rules

Every rule here comes from the published guide unless marked
**General practice**. General-practice rules are standard web image
engineering, not claims about how Track behaves. Present them that way.

Source: **Guide**, [The Image Serving Process](https://developer.trackhs.com/docs/the-image-serving-process)

## What the Guide says

- Track APIs return images as either a public URL to the image file or a
  signed URL that expires after 60 minutes.
- Use those URLs only to download the image into your own infrastructure, or
  to load it through the inline image proxy.
- Direct links to the S3 files are unsupported. Originals are often full size
  and may not have caching headers. Sites that hotlink images may be blocked
  without notice.
- The image proxy uses CDN edge caching and adds long expiration headers.
- The first load of many images at once can be slow; later requests are
  faster once the CDN has cached them.
- Lazy-load images outside the viewport.
- Use no more than 3 to 4 image sizes, for better caching and consistency.
- If the proxy does not meet your needs, build your own image distribution.

## Proxy URL format (Guide)

```
https://img.trackhs.com/<transformation>/<s3url>
```

`<s3url>` is the full S3 URL as returned, including `https://`, appended as
is. The Guide's example uses an object on `track-pm.s3.amazonaws.com`.

| Transformation | Result |
| --- | --- |
| `1187x791` | exact width and height |
| `500x` | width 500, height scaled to keep the aspect ratio |
| `x791` | height 791, width scaled to keep the aspect ratio |
| `500x791` | scaled and cropped to a vertical frame |
| `500` | scaled and cropped to a square |
| `500x300` | scaled small, with slight cropping to fit |

Use `<W>x` or `x<H>` to fix one dimension and scale the other.

## Checklist

### I1. No direct links to S3 (Guide)
Browsers must not load the raw S3 URL: not in `<img src>`, `srcset`,
`background-image`, Open Graph tags, emails, or a framework image component
that passes the URL through untouched.
**Flag:** any S3 image URL from Track reaching a browser without going
through `img.trackhs.com` or the developer's own image host.

### I2. Signed URLs expire after 60 minutes (Guide)
Some image URLs are signed and expire after 60 minutes. The Guide does not
list which endpoints return which kind, so detect it per URL: AWS signed URLs
carry query params such as `X-Amz-Signature`, `X-Amz-Expires`,
`X-Amz-Credential` (Signature V4) or `Signature` and `Expires` (V2).
A signed URL is only good for a short-lived use: downloading it now, or
loading it through the proxy now.
**Flag:** signed URLs written to a database, a long-lived cache, static HTML,
a search index, a CDN config, or an email, then shown later. Also flag code
that strips the query string from a signed URL to "clean" it.
**Fix:** either store the image itself (download into your own storage) or
store a reference and fetch a fresh URL from the API when needed. Do not
guess which one the API expects; follow what the endpoint returns.

### I3. Use the proxy or your own image host (Guide)
Every displayed image should go through `https://img.trackhs.com/...` or the
developer's own image infrastructure.
**Flag:** full-size originals on search results, cards, maps, thumbnails, or
anywhere a smaller image would do.

### I4. Documented proxy URL format (Guide)
Build proxy URLs exactly as documented: transformation first, then the full
S3 URL appended as is.
**Flag:** proxy URLs that put the transformation after the S3 URL, drop the
`https://` from the S3 part, use query params for sizing, or percent-encode
the S3 URL. The Guide does not say whether these variants work; tell the
developer to match the documented format rather than claiming they break.

### I5. Three to four sizes (Guide)
The Guide recommends no more than 3 to 4 sizes for better caching and
consistency.
**Flag:** more than 3 or 4 distinct transformations across the site, sizes
computed from the viewport or container at runtime (`500x`, `512x`, `517x`...),
or long `srcset` lists with many widths.
**Fix:** define a small named set (for example thumb, card, hero) in one
place and use it everywhere.

### I6. Lazy-load off-screen images (Guide)
**Flag:** galleries, carousels and result grids that load every image up
front. Use `loading="lazy"` or the framework's lazy option for images below
the fold. Keep the first visible image (the hero) eager.

### I7. Expect a slow first load (Guide)
The first time many images are requested at once, the proxy can be slow until
the CDN has them cached.
**Flag:** timeouts or error handling tuned so tightly that a cold first load
is treated as a failure. If launching a new site or many new listings, warn
that the first page views will be slower.

### I8. Self-hosting is a valid option (Guide)
If the proxy does not fit (custom formats, own CDN, offline use), the Guide
points to building your own image distribution. Options it lists:
- Commercial: Cloudinary
- AWS: Serverless Image Handler (Lambda)
- Open source: imgproxy, willnorris/imageproxy

When self-hosting, download the image while the Track URL is valid (I2), store
the file, and serve it from your own host with your own caching headers.

### I9. Dimensions in markup (General practice)
Set `width` and `height` (or an aspect-ratio box) matching the requested
transformation so the page does not shift as images load.

### I10. Refresh self-hosted copies (General practice)
If you store copies of Track images, re-sync them when listing content
changes so removed or replaced photos do not linger. Pair this with the
`track-api-caching` skill's content sync.

## Out of scope: do not claim

The Guide does not state these. Do not assert them. If the developer asks,
say the docs do not cover it and suggest they contact Track:

- Which endpoints return public URLs and which return signed URLs
- Proxy limits, rate limits, supported output formats (WebP, AVIF), quality
  or other options beyond the dimension examples above
- Exact cache header values or CDN cache duration
- Whether non-documented proxy URL variants work (encoded S3 URLs, query
  params)
- Whether a signed URL that has already expired can still be loaded through
  the proxy
- How or when a site gets blocked for hotlinking
