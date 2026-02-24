# Claude Skills

Custom Claude Code skills for technical SEO auditing.

## Skills

### seo-cdn

Audit and fix the 7 CDN misconfigurations that silently destroy SEO rankings -- even when Core Web Vitals are perfect.

**What it checks:**

1. **Caching HTML Pages** -- CDN serving stale content to Googlebot
2. **Wrong Cache Headers** -- incorrect Cache-Control/Expires per content type
3. **Stripping SEO-Critical Headers** -- X-Robots-Tag, Link (hreflang), Vary removed by CDN
4. **Bot Detection Blocking Googlebot** -- WAF/rate-limiting/CAPTCHAs hitting crawlers
5. **Serving Different Content to Bots** -- cloaking violations via CDN edge functions
6. **Geographic Redirect Loops** -- geo-IP redirects conflicting with hreflang
7. **CDN Altering URLs & Breaking Canonicals** -- trailing slashes, case changes, query params

**Supported CDN providers:** Cloudflare, Vercel, Netlify, AWS CloudFront, Fastly, Akamai, Nginx, Apache

**Usage:**

```
/claude-skills:seo-cdn
```

## Installation

```bash
claude install-plugin /path/to/claude-skills
```

## Structure

```
claude-skills/
├── .claude-plugin/
│   └── marketplace.json
└── skills/
    └── seo-cdn/
        ├── SKILL.md
        └── references/
            ├── cdn-config-patterns.md
            └── header-checklist.md
```

## License

MIT
