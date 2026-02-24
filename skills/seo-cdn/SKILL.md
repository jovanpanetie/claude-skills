---
name: seo-cdn
description: Audit and fix CDN misconfigurations that destroy SEO rankings. Use when the user mentions "CDN SEO," "CDN configuration," "Googlebot blocked," "cache headers SEO," "CDN caching issues," "rankings dropping with CDN," "bot detection SEO," "CDN redirect loops," "X-Robots-Tag stripped," or "cloaking." Also use when rankings drop despite good Core Web Vitals, or when indexed content appears stale.
metadata:
  version: 1.0.0
---

# SEO CDN Audit & Fix

You are an expert in CDN configuration for SEO. Your goal is to identify and fix CDN misconfigurations that silently destroy search rankings even when Core Web Vitals look perfect.

## Why This Matters

A CDN misconfiguration can tank rankings while every speed metric shows green. These issues are invisible to standard performance audits because the problem isn't speed -- it's what Google sees (or doesn't see) when crawling the site.

---

## Initial Assessment

**Check for product marketing context first:**
If `.claude/product-marketing-context.md` exists, read it before asking questions.

Before auditing, understand:

1. **CDN Provider** -- Which CDN? (Cloudflare, AWS CloudFront, Fastly, Vercel, Netlify, Akamai, KeyCDN, Bunny, etc.)
2. **Stack** -- What framework/server? (Next.js, Nuxt, Rails, Django, WordPress, static site, etc.)
3. **Symptoms** -- What's happening? (Rankings dropping, pages not indexing, stale content in SERPs, "Crawled - currently not indexed," etc.)
4. **Scope** -- Full CDN audit or specific issue?
5. **Access** -- Can we access server config, CDN dashboard config, or only the codebase?

---

## Audit Framework

### Priority Order

1. **HTML Caching Policy** (is Google seeing fresh content?)
2. **Cache Headers** (are headers correct and consistent?)
3. **SEO-Critical Header Preservation** (are headers reaching Googlebot?)
4. **Bot Access** (can Googlebot crawl without being blocked?)
5. **Content Parity** (does Googlebot see the same content as users?)
6. **Geographic Routing** (are redirects creating loops?)
7. **Canonical & URL Consistency** (is the CDN altering URLs?)

---

## Mistake 1: Caching HTML Pages

CDNs cache everything by default -- including HTML. This means Googlebot sees cached, outdated content instead of your latest pages.

### Symptoms
- New pages take days to appear in search
- Updated content doesn't get re-indexed
- Dynamic content (pricing, availability, user-generated) becomes stale
- Google Search Console shows outdated cached versions

### What to Check

**In the codebase:**
- Server-side cache-control headers for HTML responses
- CDN configuration files (e.g., `_headers`, `netlify.toml`, `vercel.json`, `wrangler.toml`, `next.config.js`)
- Middleware or edge functions that set caching headers
- Framework-level cache configuration

**Patterns to search for:**

```
# Files that configure caching behavior
_headers, netlify.toml, vercel.json, next.config.js, nuxt.config.ts,
nginx.conf, .htaccess, cdn-config.*, wrangler.toml, cloudflare-workers/*,
middleware.ts, middleware.js, server.ts, server.js
```

### The Fix

HTML pages should NOT be cached at the CDN level, or should use very short TTLs with `stale-while-revalidate`:

```
# For HTML pages:
Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=300

# Or for fully dynamic pages:
Cache-Control: no-cache, no-store, must-revalidate

# Or short-lived cache with revalidation:
Cache-Control: public, max-age=0, must-revalidate
ETag: "abc123"
```

**Key principle:** Static assets (JS, CSS, images, fonts) can be cached aggressively. HTML should not.

See [CDN Configuration Patterns](references/cdn-config-patterns.md) for provider-specific configurations.

---

## Mistake 2: Wrong Cache Headers

The server sends cache headers, the CDN respects them. If the headers are wrong, you get cascading problems.

### Symptoms
- Stale content served to users and crawlers
- Images not loading (no-cache on images)
- Dynamic pages cached for too long
- Inconsistent content between CDN edge locations

### What to Check

**Common misconfigurations:**
- `Cache-Control: max-age=31536000` on HTML pages (1 year cache!)
- `Expires` headers set in the past (forces re-fetch every time, overloads origin)
- `no-cache` on static assets (images, CSS, JS) -- kills performance
- `public` cache on authenticated/dynamic pages (leaks private content)
- Missing `Vary` headers when content differs by header (e.g., Accept-Encoding, Accept-Language)
- Conflicting `Cache-Control` and `Expires` headers

**Search patterns:**
```
max-age=31536000    # 1-year cache on HTML?
Cache-Control.*public   # Public cache on dynamic pages?
no-cache.*no-store      # Overly aggressive no-cache?
Expires.*1999|Expires.*2000  # Ancient expires headers?
```

### The Fix

Apply correct cache headers per content type:

| Content Type | Cache-Control | Notes |
|---|---|---|
| HTML pages | `no-cache` or `max-age=0, s-maxage=60, stale-while-revalidate=300` | Always revalidate or short TTL |
| CSS/JS (hashed filenames) | `public, max-age=31536000, immutable` | Safe because filename changes on update |
| CSS/JS (unhashed) | `public, max-age=86400, must-revalidate` | 1 day, then revalidate |
| Images | `public, max-age=2592000` | 30 days |
| Fonts | `public, max-age=31536000, immutable` | Fonts rarely change |
| API responses | `private, no-store` | Never cache on CDN |
| Authenticated pages | `private, no-store` | Never cache on CDN |

---

## Mistake 3: Stripping SEO-Critical Headers

Some CDNs strip HTTP headers that Google needs for proper indexation and international targeting.

### Symptoms
- `X-Robots-Tag` directives ignored by Googlebot
- Hreflang not recognized (Link headers stripped)
- Content negotiation broken (Vary headers stripped)
- MIME type issues (Content-Type altered)

### Headers That Must Be Preserved

| Header | Purpose | SEO Impact if Stripped |
|---|---|---|
| `X-Robots-Tag` | Controls indexation per-page | Pages indexed/deindexed incorrectly |
| `Link` (rel=alternate hreflang) | International targeting | Wrong country/language versions ranked |
| `Vary` | Content negotiation | Cached wrong version served to crawlers |
| `Content-Type` | MIME type | Rendering issues, indexation problems |
| `Content-Language` | Language signal | Incorrect language classification |
| `Canonical` (Link header) | Canonical URL signal | Duplicate content issues |

### What to Check

Search for CDN configuration that might strip or override headers:

```
# Look for header manipulation in:
# - CDN worker/edge functions
# - Reverse proxy configs (nginx, Apache)
# - CDN dashboard rules (check docs for provider)
# - Middleware that modifies response headers
```

### The Fix

Configure CDN to pass through all SEO-critical headers from origin. See [CDN Configuration Patterns](references/cdn-config-patterns.md) for provider-specific header preservation configs.

**Test with:**
```bash
# Compare headers from origin vs CDN
curl -sI https://your-origin-server.com/page | sort
curl -sI https://your-cdn-domain.com/page | sort

# Check specific header
curl -sI https://yourdomain.com/page | grep -i "x-robots-tag\|link\|vary\|content-type"
```

---

## Mistake 4: Bot Detection Blocking Googlebot

CDN bot protection can accidentally block or challenge Googlebot, preventing crawling and indexation.

### Symptoms
- Pages "Crawled - currently not indexed" in Search Console
- Pages indexed but not rendering (JS not executed)
- Mobile indexing issues (mobile Googlebot blocked differently)
- Rich results disappearing
- Sudden drop in crawl rate in Search Console
- 403/429 errors in server logs for Googlebot IPs

### What to Check

**In the codebase / CDN config:**
- Bot protection / WAF rules
- Rate limiting configurations
- CAPTCHA / JavaScript challenge settings
- IP allowlists/blocklists
- User-agent based rules

**Known Googlebot user agents to allowlist:**
```
Googlebot/2.1 (+http://www.google.com/bot.html)
Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)
Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/W.X.Y.Z Mobile Safari/537.36 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)
```

**Known Googlebot IP ranges:**
Googlebot IPs can be verified via DNS reverse lookup. The CDN should NOT block IPs that reverse-resolve to `*.googlebot.com` or `*.google.com`.

### The Fix

1. **Allowlist verified search engine bots** in bot protection rules
2. **Never serve CAPTCHAs or JS challenges** to verified Googlebot
3. **Exempt Googlebot from rate limiting** (or set very generous limits)
4. **Verify bot identity** using reverse DNS, not just user-agent strings
5. **Test regularly** with Google Search Console's URL Inspection tool

**CDN-specific configs:** See [CDN Configuration Patterns](references/cdn-config-patterns.md) for bot allowlisting per provider.

---

## Mistake 5: Serving Different Content to Bots (Cloaking)

The CDN detects Googlebot and serves different content than real users see. This is a cloaking violation that can result in manual penalties.

### Symptoms
- Different images served to bots
- Missing JavaScript-rendered content for crawlers
- Simplified HTML version served to bots
- Interactive elements removed for crawlers
- Google showing content in SERPs that doesn't match what users see

### What to Check

**In the codebase:**
- User-agent detection that alters response content
- Separate rendering paths for bots vs. users
- Pre-rendering services that serve different HTML
- CDN edge functions that modify response body based on user-agent
- A/B testing tools that serve different variants to bots

**Search patterns:**
```
# Look for bot-specific content serving
user-agent.*googlebot|bot.*detection|is_bot|isBot|crawler.*detect
prerender|rendertron|puppeteer.*bot
# Check for different rendering paths
if.*bot|if.*crawler|if.*spider
```

### The Fix

1. **Serve identical content** to Googlebot and users -- always
2. If using pre-rendering / SSR for bots, ensure the output is **exactly** what a user's browser would render
3. **Dynamic rendering** (serving pre-rendered HTML to bots) is acceptable only if the content is identical
4. Remove any bot-specific content modifications from CDN edge functions
5. **Test:** Compare rendered output via `curl -A "Googlebot"` vs `curl -A "Mozilla/5.0"`

---

## Mistake 6: Geographic Redirect Loops

CDN redirects based on visitor location, creating infinite loops or incorrect redirects for Googlebot.

### Symptoms
- Infinite redirect loops for some users/regions
- Googlebot (US-based) always redirected to US version
- Hreflang signals ignored because CDN overrides them
- International pages not indexed
- "Redirect error" in Search Console

### Example Problem
```
User in Germany -> visits .com
CDN redirects to .de based on geo-IP
.de redirects back to .com (server rule)
= Infinite loop
```

### What to Check

**In the codebase / CDN config:**
- Geographic redirect rules in CDN config
- Server-side geo-redirect logic
- Middleware handling locale detection
- Edge function redirects based on `CF-IPCountry`, `X-Country`, etc.

**Search patterns:**
```
# Geographic redirect patterns
geo.*redirect|country.*redirect|locale.*redirect
CF-IPCountry|X-Country|X-Geo|geoip
redirect.*301.*locale|redirect.*302.*locale
Accept-Language.*redirect
```

### The Fix

1. **Use hreflang tags instead of forced redirects** -- let Google serve the right version
2. If geo-redirects are needed, use **302 (temporary)** not 301 (permanent)
3. **Never redirect Googlebot** based on geo-IP (Googlebot crawls from the US)
4. Allow users to **override** geo-detection (language selector that sets a cookie)
5. Use `Vary: Accept-Language` header if content varies by language
6. Implement **proper hreflang** with `x-default` for the fallback version

```html
<link rel="alternate" hreflang="en" href="https://example.com/" />
<link rel="alternate" hreflang="de" href="https://example.com/de/" />
<link rel="alternate" hreflang="fr" href="https://example.com/fr/" />
<link rel="alternate" hreflang="x-default" href="https://example.com/" />
```

---

## Mistake 7: CDN Altering URLs and Breaking Canonicals

The CDN modifies URLs -- adding trailing slashes, changing case, appending query parameters, or rewriting paths -- breaking canonical signals.

### Symptoms
- Duplicate content from URL variations (`/page` vs `/page/` vs `/Page`)
- Canonical tags pointing to wrong URL version
- CDN query parameters (`?v=`, `?_cf_chl_`) indexed by Google
- Multiple versions of same page in Google index
- Link equity split across URL variations

### What to Check

**In the codebase / CDN config:**
- URL normalization rules
- Trailing slash handling
- Case sensitivity settings
- Query parameter handling
- CDN-added tracking parameters

**Search patterns:**
```
# URL manipulation
trailingSlash|trailing_slash|trailing-slash
url.*rewrite|rewrite.*url|path.*rewrite
canonical.*url|rel.*canonical
query.*param|utm_|_cf_|_vercel
redirect.*slash|slash.*redirect
toLowerCase|tolowercase|case.*sensitive
```

### The Fix

1. **Pick one URL format** (with or without trailing slash) and enforce it consistently
2. **Self-referencing canonical tags** on every page pointing to the normalized URL
3. **Strip CDN-added query parameters** from canonical tags
4. **Configure CDN to normalize URLs** consistently (redirect one format to the other)
5. **Set canonical in both HTML and HTTP header:**
   ```
   Link: <https://example.com/page>; rel="canonical"
   ```
6. **Handle case sensitivity:** redirect mixed-case URLs to lowercase

---

## Verification Checklist

After applying fixes, verify each point:

### Quick Verification Commands

```bash
# 1. Check cache headers on HTML page
curl -sI "https://yourdomain.com/" | grep -i "cache-control\|expires\|etag\|last-modified"

# 2. Verify SEO headers are preserved
curl -sI "https://yourdomain.com/" | grep -i "x-robots-tag\|link\|vary\|content-type\|content-language"

# 3. Test bot access (simulate Googlebot)
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" "https://yourdomain.com/"

# 4. Compare bot vs user response
diff <(curl -s -A "Googlebot" "https://yourdomain.com/") <(curl -s -A "Mozilla/5.0" "https://yourdomain.com/")

# 5. Check for redirect loops
curl -sIL "https://yourdomain.com/" 2>&1 | grep -i "location\|HTTP/"

# 6. Verify canonical URL consistency
curl -s "https://yourdomain.com/page" | grep -i "canonical"
curl -s "https://yourdomain.com/page/" | grep -i "canonical"

# 7. Check CDN-specific headers (identify CDN)
curl -sI "https://yourdomain.com/" | grep -i "cf-ray\|x-vercel\|x-amz\|x-fastly\|server\|via"
```

### Search Console Verification
- URL Inspection tool: check rendered page matches live page
- Coverage report: look for "Crawled - currently not indexed" spikes
- Core Web Vitals report: verify no regressions after changes
- Crawl Stats: check for crawl rate drops or error spikes

---

## Output Format

### Audit Report Structure

**Executive Summary**
- CDN provider identified
- Number of issues found by severity
- Estimated SEO impact

**Findings**
For each issue:
- **Issue**: What's wrong (which mistake category)
- **Severity**: Critical / High / Medium / Low
- **Evidence**: Config code or curl output showing the problem
- **Fix**: Specific code change with before/after
- **Files to modify**: Exact file paths and line numbers
- **Verification**: Command to confirm the fix works

**Prioritized Action Plan**
1. Critical: Issues actively blocking indexation or causing penalties
2. High: Issues degrading rankings or causing stale content
3. Medium: Issues that could become problems at scale
4. Low: Best-practice improvements

---

## How to Audit a Project

When auditing a project's CDN SEO configuration:

1. **Identify the CDN and stack** by checking:
   - `package.json` for framework (Next.js, Nuxt, Gatsby, etc.)
   - Config files: `vercel.json`, `netlify.toml`, `wrangler.toml`, `next.config.*`, `nuxt.config.*`
   - Server config: `nginx.conf`, `.htaccess`, `docker-compose.yml`
   - Response headers: `cf-ray` (Cloudflare), `x-vercel-id` (Vercel), `x-amz-cf-id` (CloudFront)

2. **Search for cache configuration** across the codebase:
   - Grep for `cache-control`, `max-age`, `s-maxage`, `stale-while-revalidate`
   - Check middleware files for header manipulation
   - Look for CDN-specific config files

3. **Search for bot/crawler handling**:
   - Grep for `googlebot`, `bot`, `crawler`, `user-agent`, `isBot`
   - Check WAF/security configuration
   - Look for pre-rendering or dynamic rendering setup

4. **Search for redirect logic**:
   - Grep for `redirect`, `301`, `302`, `location`, `geo`, `country`, `locale`
   - Check for geo-IP based routing
   - Look for hreflang implementation

5. **Search for URL handling**:
   - Grep for `canonical`, `trailingSlash`, `rewrite`, `redirect`
   - Check URL normalization rules
   - Look for query parameter handling

6. **Cross-reference with reference files** for CDN-specific patterns:
   - [CDN Configuration Patterns](references/cdn-config-patterns.md)
   - [Header Checklist](references/header-checklist.md)

---

## References

- [CDN Configuration Patterns](references/cdn-config-patterns.md): Provider-specific config examples for Cloudflare, Vercel, Netlify, CloudFront, Fastly, Nginx
- [Header Checklist](references/header-checklist.md): Complete header audit checklist with correct values per content type

---

## Related Skills

- **seo-audit**: For broader SEO auditing beyond CDN issues
- **schema-markup**: For implementing structured data (which CDNs can also break)
- **page-cro**: For optimizing page conversions after fixing SEO foundations
- **analytics-tracking**: For measuring recovery after CDN fixes
