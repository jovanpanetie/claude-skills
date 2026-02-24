# SEO Header Audit Checklist

Complete checklist of HTTP headers to verify when auditing CDN configuration for SEO.

---

## Cache Headers

### HTML Pages

| Header | Expected Value | Issue if Wrong |
|---|---|---|
| `Cache-Control` | `public, max-age=0, must-revalidate` or `no-cache` | Stale content served to Googlebot |
| `ETag` | Present (unique per content version) | No conditional request support |
| `Last-Modified` | Present (accurate timestamp) | Googlebot can't detect content changes |
| `Vary` | `Accept-Encoding` (minimum) | Wrong cached version served |

### Static Assets (JS, CSS with hashed filenames)

| Header | Expected Value | Issue if Wrong |
|---|---|---|
| `Cache-Control` | `public, max-age=31536000, immutable` | Unnecessary re-downloads |
| `Content-Type` | Correct MIME type | Browser parsing errors |
| `Content-Encoding` | `gzip` or `br` | Larger transfer size |

### Images

| Header | Expected Value | Issue if Wrong |
|---|---|---|
| `Cache-Control` | `public, max-age=2592000` (30 days) | Unnecessary re-downloads |
| `Content-Type` | `image/webp`, `image/avif`, `image/png`, etc. | Display issues |
| `Accept-Ranges` | `bytes` | No progressive loading |

### Fonts

| Header | Expected Value | Issue if Wrong |
|---|---|---|
| `Cache-Control` | `public, max-age=31536000, immutable` | FOUT/FOIT on repeat visits |
| `Access-Control-Allow-Origin` | `*` or specific domain | CORS blocking in cross-origin requests |
| `Content-Type` | `font/woff2`, `font/woff`, etc. | Browser rejection |

### API Responses

| Header | Expected Value | Issue if Wrong |
|---|---|---|
| `Cache-Control` | `private, no-store` | Sensitive data cached on CDN |
| `Vary` | `Authorization, Cookie` | Shared cached responses |

---

## SEO-Critical Headers

These headers MUST pass through the CDN unchanged from the origin server.

### X-Robots-Tag

```
X-Robots-Tag: noindex
X-Robots-Tag: noindex, nofollow
X-Robots-Tag: noarchive
X-Robots-Tag: googlebot: noindex
X-Robots-Tag: max-snippet:-1, max-image-preview:large, max-video-preview:-1
```

**Verification:**
```bash
curl -sI https://yourdomain.com/page | grep -i "x-robots-tag"
```

**If stripped:** Pages may get indexed when they shouldn't (or miss indexing directives).

### Link Header (hreflang, canonical, preload)

```
Link: <https://example.com/de/page>; rel="alternate"; hreflang="de"
Link: <https://example.com/page>; rel="canonical"
Link: </styles/main.css>; rel="preload"; as="style"
```

**Verification:**
```bash
curl -sI https://yourdomain.com/page | grep -i "^link:"
```

**If stripped:** International targeting fails, canonical signals lost.

### Vary

```
Vary: Accept-Encoding
Vary: Accept-Encoding, Accept-Language
Vary: Accept-Encoding, Cookie
```

**Verification:**
```bash
curl -sI https://yourdomain.com/page | grep -i "^vary:"
```

**If stripped or wrong:** CDN serves wrong cached version (e.g., gzipped content to a client that doesn't support it, or English content to a French user).

### Content-Type

```
Content-Type: text/html; charset=utf-8
Content-Type: application/json; charset=utf-8
Content-Type: text/css
Content-Type: application/javascript
```

**Verification:**
```bash
curl -sI https://yourdomain.com/page | grep -i "^content-type:"
```

**If wrong:** Rendering issues, MIME type sniffing vulnerabilities, incorrect indexation.

### Content-Language

```
Content-Language: en
Content-Language: fr
Content-Language: de-DE
```

**Verification:**
```bash
curl -sI https://yourdomain.com/page | grep -i "^content-language:"
```

**If stripped:** Google may misclassify the page's language.

---

## Security Headers (SEO-Adjacent)

These don't directly affect SEO but can cause rendering issues that impact indexation.

| Header | Purpose | SEO Impact |
|---|---|---|
| `Content-Security-Policy` | Controls resource loading | Can block JS rendering for Googlebot |
| `X-Frame-Options` | Prevents framing | Can block Google Cache preview |
| `Strict-Transport-Security` | Forces HTTPS | Good for SEO (HTTPS signal) |
| `Referrer-Policy` | Controls referer header | Affects analytics, not ranking |

### CSP Warning

Overly restrictive CSP can block Googlebot's rendering. Ensure CSP allows:
- Inline scripts (if used by your framework)
- Google Tag Manager domains (if using GTM)
- CDN domains for assets
- Font domains

```bash
# Check if CSP might block rendering
curl -sI https://yourdomain.com/ | grep -i "content-security-policy"
```

---

## Response Code Headers

### Redirect Headers

| Scenario | Expected | Issue if Wrong |
|---|---|---|
| HTTP to HTTPS | `301` + `Location: https://...` | Link equity not passed |
| www to non-www (or vice versa) | `301` + correct `Location` | Duplicate content |
| Geo-redirect | `302` (temporary) | `301` makes redirect permanent for Googlebot |
| Old URL to new URL | `301` + new `Location` | Lost link equity |
| Trailing slash normalization | `301` + normalized URL | Duplicate content |

### Status Codes to Verify

```bash
# Check for redirect chains (should be max 1 redirect)
curl -sIL https://yourdomain.com/page 2>&1 | grep "HTTP/"

# Expected: one 200, or one 301/302 followed by one 200
# Bad: multiple 301/302 (chain) or no 200 (loop)
```

---

## CDN-Identification Headers

Use these to identify which CDN is in use:

| Header | CDN |
|---|---|
| `cf-ray` | Cloudflare |
| `cf-cache-status` | Cloudflare (HIT/MISS/DYNAMIC) |
| `x-vercel-id` | Vercel |
| `x-vercel-cache` | Vercel (HIT/MISS/STALE) |
| `x-amz-cf-id` | AWS CloudFront |
| `x-amz-cf-pop` | AWS CloudFront (edge location) |
| `x-cache` | AWS CloudFront or Varnish |
| `x-fastly-request-id` | Fastly |
| `x-served-by` | Fastly |
| `x-nf-request-id` | Netlify |
| `server: Netlify` | Netlify |
| `x-cdn` | Generic CDN indicator |
| `via` | Often contains CDN info |
| `server: AkamaiGHost` | Akamai |
| `x-bunny-*` | BunnyCDN |

```bash
# Identify CDN
curl -sI https://yourdomain.com/ | grep -i "cf-ray\|x-vercel\|x-amz\|x-fastly\|x-nf\|x-served-by\|x-cache\|server\|via\|x-cdn\|x-bunny"
```

---

## Complete Audit Command

Run this single command to capture all relevant headers at once:

```bash
# Full header dump for SEO audit
echo "=== Response Headers ===" && \
curl -sI "https://yourdomain.com/" && \
echo "" && \
echo "=== Googlebot Response ===" && \
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" "https://yourdomain.com/" && \
echo "" && \
echo "=== Redirect Chain ===" && \
curl -sIL "https://yourdomain.com/" 2>&1 | grep -i "HTTP/\|location:" && \
echo "" && \
echo "=== Trailing Slash Variant ===" && \
curl -sI "https://yourdomain.com/page/" 2>&1 | grep -i "HTTP/\|location:\|cache-control"
```

---

## Quick Reference: What Breaks What

| CDN Misconfiguration | Symptom in Google Search Console | Fix Category |
|---|---|---|
| HTML cached too long | Stale content in SERPs | Mistake 1 |
| max-age=31536000 on HTML | New pages not appearing | Mistake 2 |
| X-Robots-Tag stripped | Wrong pages indexed | Mistake 3 |
| Link hreflang header stripped | Wrong country version ranking | Mistake 3 |
| Bot protection blocks Googlebot | "Crawled - not indexed" | Mistake 4 |
| Rate limiting hits Googlebot | Crawl rate drops | Mistake 4 |
| Different content for bots | Manual action penalty | Mistake 5 |
| Geo-redirect with 301 | Only one version indexed | Mistake 6 |
| Redirect loops | "Redirect error" status | Mistake 6 |
| Trailing slash inconsistency | Duplicate pages indexed | Mistake 7 |
| CDN query params in index | Duplicate content | Mistake 7 |
