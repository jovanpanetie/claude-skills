# Claude Skills

Custom Claude Code skills for non technical profiles (more to come. open sources. feel free to open PR.)

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

### Option 1: CLI Install (Recommended)

Use [npx skills](https://www.npmjs.com/package/skills) to install skills directly:

```bash
# Install all skills
npx skills add jovanpanetie/claude-skills

# Install specific skills
npx skills add jovanpanetie/claude-skills --skill seo-cdn

# List available skills
npx skills add jovanpanetie/claude-skills --list
```

This automatically installs to your `.claude/skills/` directory.

### Option 2: Claude Code Plugin

Install via Claude Code's built-in plugin system:

```
# Add the marketplace
/plugin marketplace add jovanpanetie/claude-skills

# Install all skills
/plugin install claude-skills
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
