# CDN Configuration Patterns for SEO

Provider-specific configurations to fix the 7 CDN mistakes that destroy SEO.

---

## Cloudflare

### Cache Rules (HTML bypass)

```toml
# In Cloudflare Dashboard > Caching > Cache Rules
# Or via API / Terraform:

# Rule: Bypass cache for HTML pages
# Match: Content-Type contains "text/html"
# Action: Bypass Cache

# Rule: Cache static assets aggressively
# Match: File extension in {js, css, png, jpg, webp, woff2, svg}
# Action: Cache with Edge TTL = 1 month
```

### Page Rules (legacy)

```
# Bypass cache for HTML
URL: example.com/*
Cache Level: Bypass (for HTML)

# Cache everything for static assets
URL: example.com/assets/*
Cache Level: Cache Everything
Edge Cache TTL: 1 month
```

### Cloudflare Workers (header preservation)

```javascript
// Ensure SEO-critical headers pass through
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const response = await fetch(request)
  const newResponse = new Response(response.body, response)

  // Preserve SEO-critical headers from origin
  const seoHeaders = [
    'x-robots-tag',
    'link',
    'vary',
    'content-type',
    'content-language',
    'last-modified',
    'etag'
  ]

  for (const header of seoHeaders) {
    const value = response.headers.get(header)
    if (value) {
      newResponse.headers.set(header, value)
    }
  }

  // Set correct cache headers for HTML
  const contentType = response.headers.get('content-type') || ''
  if (contentType.includes('text/html')) {
    newResponse.headers.set('Cache-Control', 'public, max-age=0, s-maxage=60, stale-while-revalidate=300')
  }

  return newResponse
}
```

### Bot Management (allowlist Googlebot)

```
# Cloudflare Dashboard > Security > Bots
# - Set "Verified Bots" policy to "Allow"
# - Do NOT enable "Bot Fight Mode" for verified bots
# - Super Bot Fight Mode: set "Verified Bots" to "Allow"

# If using WAF custom rules:
# Skip rule when: cf.client.bot = true
```

---

## Vercel

### vercel.json (headers and caching)

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=0, must-revalidate"
        }
      ]
    },
    {
      "source": "/static/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    },
    {
      "source": "/_next/static/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ],
  "trailingSlash": false
}
```

### Next.js specific (next.config.js)

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Enforce consistent URL format
  trailingSlash: false,

  // Custom headers
  async headers() {
    return [
      {
        // HTML pages: no aggressive caching
        source: '/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=0, must-revalidate',
          },
        ],
      },
      {
        // Preserve X-Robots-Tag from API routes
        source: '/api/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'private, no-store',
          },
        ],
      },
    ]
  },

  // Redirects for URL consistency
  async redirects() {
    return [
      // Example: redirect uppercase to lowercase
      // Handle in middleware for dynamic patterns
    ]
  },
}

module.exports = nextConfig
```

### Next.js Middleware (bot detection fix)

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const response = NextResponse.next()
  const userAgent = request.headers.get('user-agent') || ''

  // NEVER serve different content to bots
  // NEVER block search engine bots
  // NEVER redirect bots based on geo-IP

  const isSearchBot = /googlebot|bingbot|yandex|baiduspider|duckduckbot/i.test(userAgent)

  // If geo-redirect logic exists, skip it for search bots
  if (isSearchBot) {
    // Let the bot see the content at the URL it requested
    return response
  }

  // For real users, you can suggest (not force) locale
  // Use 302 (not 301) for geo-based suggestions
  // Always provide a way to override via cookie

  return response
}
```

---

## Netlify

### netlify.toml

```toml
# Cache headers
[[headers]]
  for = "/*"
  [headers.values]
    Cache-Control = "public, max-age=0, must-revalidate"

[[headers]]
  for = "/static/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/_next/static/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/*.js"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "/*.css"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

# Preserve SEO headers (Netlify passes most headers by default)
# But explicitly set X-Robots-Tag if needed:
[[headers]]
  for = "/private-page"
  [headers.values]
    X-Robots-Tag = "noindex, nofollow"

# Redirects -- use 302 for geo, always provide hreflang instead
# AVOID forced geo-redirects:
# [[redirects]]
#   from = "/*"
#   to = "/de/:splat"
#   status = 302
#   conditions = {Country = ["DE"]}
# USE hreflang tags in HTML instead
```

### _headers file (alternative)

```
# Default: no aggressive caching for HTML
/*
  Cache-Control: public, max-age=0, must-revalidate

# Static assets: cache aggressively
/static/*
  Cache-Control: public, max-age=31536000, immutable

# Fonts
/fonts/*
  Cache-Control: public, max-age=31536000, immutable
  Access-Control-Allow-Origin: *
```

---

## AWS CloudFront

### Cache Policy

```json
{
  "Comment": "SEO-friendly cache policy",
  "DefaultTTL": 0,
  "MaxTTL": 86400,
  "MinTTL": 0,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": {
      "HeaderBehavior": "whitelist",
      "Headers": {
        "Items": ["X-Robots-Tag", "Link", "Vary", "Content-Language"]
      }
    },
    "CookiesConfig": {
      "CookieBehavior": "none"
    },
    "QueryStringsConfig": {
      "QueryStringBehavior": "none"
    }
  }
}
```

### Origin Response Headers Policy

```json
{
  "Comment": "Preserve SEO headers from origin",
  "ResponseHeadersPolicyConfig": {
    "CustomHeadersConfig": {
      "Items": []
    },
    "SecurityHeadersConfig": {},
    "ServerTimingHeadersConfig": {
      "Enabled": false
    }
  }
}
```

### CloudFront Functions (bot handling)

```javascript
// viewer-request function
function handler(event) {
  var request = event.request;
  var headers = request.headers;
  var userAgent = headers['user-agent'] ? headers['user-agent'].value : '';

  // Never block or redirect verified search engine bots
  var isSearchBot = /googlebot|bingbot|yandex|baiduspider/i.test(userAgent);

  if (isSearchBot) {
    // Pass through without geo-redirect or challenge
    return request;
  }

  // Non-bot logic here (geo-redirect with 302, etc.)
  return request;
}
```

### Cache Behaviors (per path pattern)

```
# HTML pages (default behavior)
Path Pattern: Default (*)
Cache Policy: SEO-friendly (TTL=0, honors origin headers)
Origin Request Policy: Forward SEO headers

# Static assets
Path Pattern: /static/*
Cache Policy: CachingOptimized (TTL=86400)

# API routes
Path Pattern: /api/*
Cache Policy: CachingDisabled
```

---

## Fastly

### VCL Configuration

```vcl
sub vcl_recv {
  # Identify search engine bots
  if (req.http.User-Agent ~ "(?i)(googlebot|bingbot|yandex|baiduspider)") {
    set req.http.X-Is-Search-Bot = "true";
    # Never challenge or redirect bots
    # Skip geo-redirect logic
  }
}

sub vcl_fetch {
  # HTML pages: short cache with revalidation
  if (beresp.http.Content-Type ~ "text/html") {
    set beresp.ttl = 60s;
    set beresp.stale_while_revalidate = 300s;
    set beresp.http.Cache-Control = "public, max-age=0, s-maxage=60, stale-while-revalidate=300";
  }

  # Preserve SEO-critical headers
  # Fastly preserves most headers by default, but verify:
  # - X-Robots-Tag
  # - Link (hreflang)
  # - Vary
  # - Content-Language
}

sub vcl_deliver {
  # Ensure headers reach the client
  if (resp.http.X-Robots-Tag) {
    set resp.http.X-Robots-Tag = resp.http.X-Robots-Tag;
  }
}
```

---

## Nginx (as reverse proxy / CDN origin)

### nginx.conf

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # HTML pages: no aggressive caching
    location / {
        proxy_pass http://upstream;

        # Pass SEO-critical headers from upstream
        proxy_pass_header X-Robots-Tag;
        proxy_pass_header Link;
        proxy_pass_header Vary;
        proxy_pass_header Content-Language;
        proxy_pass_header Content-Type;

        # Set cache headers for HTML
        add_header Cache-Control "public, max-age=0, must-revalidate" always;

        # Do NOT strip these headers
        proxy_hide_header X-Powered-By;
        # Do NOT add: proxy_hide_header X-Robots-Tag;
    }

    # Static assets: cache aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|webp|woff|woff2|ttf|eot)$ {
        proxy_pass http://upstream;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    # Bot detection: never block Googlebot
    # If using rate limiting:
    map $http_user_agent $is_search_bot {
        default 0;
        "~*googlebot" 1;
        "~*bingbot" 1;
        "~*yandex" 1;
    }

    # Apply rate limiting only to non-bots
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;

    location / {
        if ($is_search_bot = 0) {
            limit_req zone=general burst=20 nodelay;
        }
    }

    # URL normalization: consistent trailing slash
    # Choose ONE of these:

    # Option A: Remove trailing slash
    rewrite ^/(.*)/$ /$1 permanent;

    # Option B: Add trailing slash
    # rewrite ^(.*[^/])$ $1/ permanent;

    # Redirect uppercase to lowercase
    # (use a map or Lua for this)
}
```

---

## Apache (.htaccess)

```apache
# Cache headers for HTML
<FilesMatch "\.(html|htm|php)$">
    Header set Cache-Control "public, max-age=0, must-revalidate"
</FilesMatch>

# Cache headers for static assets
<FilesMatch "\.(js|css|png|jpg|jpeg|gif|ico|svg|webp|woff|woff2|ttf|eot)$">
    Header set Cache-Control "public, max-age=31536000, immutable"
</FilesMatch>

# Preserve SEO headers (don't unset these)
# Header unset X-Robots-Tag  <-- NEVER DO THIS
# Header unset Link          <-- NEVER DO THIS

# URL normalization: remove trailing slash (except root)
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_URI} (.+)/$
RewriteRule ^ %1 [R=301,L]

# Force lowercase URLs
RewriteMap lowercase int:tolower
RewriteCond %{REQUEST_URI} [A-Z]
RewriteRule (.*) ${lowercase:$1} [R=301,L]

# Do NOT geo-redirect with mod_geoip -- use hreflang instead
```

---

## Common Mistakes in Framework Configs

### Next.js ISR / SSG Cache Issues

```javascript
// WRONG: Caching dynamic pages for too long
export const revalidate = 86400 // 1 day cache for frequently-changing content

// RIGHT: Short revalidation for content that changes often
export const revalidate = 60 // 1 minute

// RIGHT: On-demand revalidation for content updates
// Use revalidatePath() or revalidateTag() in API routes
```

### Nuxt.js (nuxt.config.ts)

```typescript
export default defineNuxtConfig({
  routeRules: {
    // HTML pages: short cache
    '/**': {
      headers: {
        'Cache-Control': 'public, max-age=0, must-revalidate'
      }
    },
    // Static assets: long cache
    '/_nuxt/**': {
      headers: {
        'Cache-Control': 'public, max-age=31536000, immutable'
      }
    },
    // API: no cache
    '/api/**': {
      headers: {
        'Cache-Control': 'private, no-store'
      }
    }
  }
})
```

### Gatsby (gatsby-config.js + hosting)

```javascript
// Gatsby generates static HTML, so caching strategy matters.
// In your hosting config (not Gatsby itself):

// HTML files: always revalidate (content changes on rebuild)
// page-data.json: always revalidate
// static/*: cache forever (hashed filenames)
// *.js, *.css in /webpack: cache forever (hashed)
```

### WordPress + CDN

```php
// In your theme's functions.php or a must-use plugin:

// Set proper cache headers for HTML
add_action('send_headers', function() {
    if (!is_admin()) {
        header('Cache-Control: public, max-age=0, s-maxage=300, stale-while-revalidate=600');
    }
});

// Ensure X-Robots-Tag is not stripped
// This is typically a CDN config issue, not WordPress
// Check your CDN settings to preserve this header
```
