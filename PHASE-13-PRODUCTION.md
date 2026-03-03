# Phase 13: Production Audit

## Step 1: Migrate Figma Images to CDN

**CRITICAL:** Before production deployment, all Figma MCP URLs must be migrated to permanent CDN storage.

Figma MCP URLs (`https://www.figma.com/api/mcp/asset/...`) are temporary and will eventually expire.

### Run Image Migration

1. **Get CMS auth token** via login mutation
2. **Run migration script** from Phase 4:
   ```bash
   CMS_AUTH_TOKEN='your-token' npx tsx scripts/migrate-figma-images.ts
   ```
3. **Verify** all images now use `pub-*.r2.dev` or your CDN domain
4. **Test** the site to ensure images load correctly

See [PHASE-4-ASSETS.md](PHASE-4-ASSETS.md) for the full migration script.

## Step 2: Build Verification

```bash
npm run build
npm run lint
npx tsc --noEmit
```

All must pass without errors.

## Step 3: Verification Checklist

### Images
- [ ] **Image migration complete** — no Figma MCP URLs remain
- [ ] All images use `next/image`
- [ ] All images reference CDN URLs (pub-*.r2.dev or your domain)
- [ ] No local image paths in components

### Content
- [ ] All content from CMS
- [ ] No hardcoded strings (re-verify)
- [ ] CMS errors handled gracefully
- [ ] `transformImageUrls()` applied to all CMS data before rendering

### Data Attributes
- [ ] All editable elements have `data-cms-*` attributes
- [ ] Field paths are correct
- [ ] CMS data structure is flat (no `data.data` nesting)

### Accessibility
- [ ] All images have alt text (from CMS)
- [ ] Semantic HTML used
- [ ] Focus states work
- [ ] Color contrast acceptable

## Step 4: SEO / GEO Audit

### Renderer Toolkit Verification
- [ ] `@halo/renderer-toolkit` is installed and imported
- [ ] `buildMetadata(slug)` used in catch-all `generateMetadata()` — NOT hand-rolled metadata
- [ ] `buildSitemap()` used in `sitemap.ts` — NOT manual slug queries
- [ ] `buildRobots()` used in `robots.ts` — NOT hardcoded rules
- [ ] `serveLlmsTxt()` used in `app/llms.txt/route.ts`
- [ ] `<JsonLd>` component rendered in root `layout.tsx` (or per-page)
- [ ] `buildJsonLd()` called with correct schema type for the content

### SEO Module Verification
- [ ] **No SEO data stored as content entry fields** — SEO lives in `PageSeoConfig` entities only
- [ ] `savePageSeo` / `bulkSavePageSeo` used in Phase 8 to populate SEO (NOT content entry `data`)
- [ ] Each page has a `PageSeoConfig` with `metaTitle` and `metaDescription`
- [ ] `ogImage` set for key pages (homepage, about, contact) via `PageSeoConfig`
- [ ] `canonicalUrl` set when pages share content across properties

### Metadata
- [ ] `generateMetadata()` uses `buildMetadata(slug)` from renderer toolkit
- [ ] Returns `title`, `description`, `openGraph`, `twitter`, `alternates` from `PageSeoConfig`
- [ ] Each page has unique `metaTitle` and `metaDescription` in CMS

### Sitemap & Robots
- [ ] `sitemap.ts` exists and generates valid XML at `/sitemap.xml`
- [ ] Sitemap uses `buildSitemap()` which queries CMS for all published slugs
- [ ] New pages created in CMS appear in sitemap without code changes
- [ ] `robots.ts` exists, uses `buildRobots()`, and references sitemap URL
- [ ] `robots.ts` includes AI crawler directives (GPTBot, ClaudeBot, PerplexityBot, Google-Extended)
- [ ] `NEXT_PUBLIC_SITE_URL` env var set to production domain

### llms.txt
- [ ] `app/llms.txt/route.ts` exists and returns valid llms.txt content
- [ ] Uses `serveLlmsTxt()` from renderer toolkit
- [ ] Content describes the site, its pages, and entity types for AI crawlers
- [ ] Response has `Content-Type: text/plain` header

### Structured Data (JSON-LD)
- [ ] `<JsonLd>` component from renderer toolkit is in root layout or per-page
- [ ] `buildJsonLd()` called with correct schema type (e.g., `LocalBusiness`, `Restaurant`, `Hotel`, `Article`, `FAQPage`)
- [ ] JSON-LD rendered as `<script type="application/ld+json">` in page source
- [ ] Structured data validated with Google Rich Results Test
- [ ] Schema.org types match content type semantics
- [ ] `BreadcrumbList` schema present for multi-level navigation

### GEO Readiness (AI Crawler Optimization)
- [ ] Semantic HTML throughout — headings (`h1`–`h6`), `nav`, `main`, `article`, `section`
- [ ] Heading hierarchy is correct (one `h1` per page, logical nesting)
- [ ] All images have descriptive `alt` text (from CMS, not empty strings)
- [ ] Content blocks are self-contained and extractable (not split across unrelated DOM nodes)
- [ ] Landmark elements used (`<header>`, `<footer>`, `<main>`, `<nav>`)
- [ ] Structured data present for entity-rich pages
- [ ] No client-side-only rendering for primary content (SSR/SSG confirmed)
- [ ] AI crawlers can access content without JavaScript (verify with `curl`)

## Step 5: Security Audit

### Input Validation
- [ ] Catch-all route validates slug format with regex `/^[a-z0-9]+(?:[-/][a-z0-9]+)*$/`
- [ ] Invalid slugs return 404 without querying CMS
- [ ] Path traversal attempts (`../../`, `%2e%2e`) blocked

### Content Safety
- [ ] CMS HTML output sanitized before rendering (DOMPurify or equivalent)
- [ ] No `dangerouslySetInnerHTML` without sanitization
- [ ] Content-type allowlist — only known types render pages
- [ ] Only published entries returned to the public site (no drafts leak)

### Credentials & Headers
- [ ] No credentials in client-side code or bundles
- [ ] `.env.local` in `.gitignore`
- [ ] No API keys, tokens, or passwords in source code
- [ ] `X-Frame-Options` or `frame-ancestors` CSP set (prevents clickjacking)
- [ ] `Strict-Transport-Security` header configured in deployment platform

### CSP (Content Security Policy)
- [ ] Base CSP configured in `next.config.js` or middleware
- [ ] Dedicated routes with third-party scripts have route-specific CSP relaxation
- [ ] `script-src` does not include `unsafe-eval` in production

## Step 6: Performance Audit

### Core Web Vitals
- [ ] LCP (Largest Contentful Paint) < 2.5s — hero images preloaded, CMS queries fast
- [ ] CLS (Cumulative Layout Shift) < 0.1 — image dimensions set, fonts preloaded
- [ ] INP (Interaction to Next Paint) < 200ms — minimal client-side JS for content pages

### Bundle & Caching
- [ ] Templates loaded with `next/dynamic` to prevent all-in-one bundle
- [ ] `next/image` `remotePatterns` configured for CDN domains
- [ ] Fonts preloaded via `next/font` or `<link rel="preload">`
- [ ] ISR revalidation webhook (`api/revalidate`) configured (or `force-dynamic` for development)
- [ ] Edge caching headers set appropriately per route type

### Build
- [ ] `npm run build` succeeds without errors
- [ ] `npm run lint` passes
- [ ] `npx tsc --noEmit` passes
- [ ] No render-blocking resources in critical path

## Step 7: Error Handling

Verify graceful degradation:
- [ ] CMS unavailable → error boundary works
- [ ] Missing content → fallbacks render
- [ ] Image load failure → handled
- [ ] Invalid slug → 404 (not 500)

## Step 8: Environment Variables

Update README with required variables:

```markdown
## Environment Variables

Required in production:
- `NEXT_PUBLIC_CMS_GRAPHQL_URL` - CMS GraphQL endpoint
- `CMS_ORGANIZATION_ID` - Organization ID
- `CMS_PROPERTY_ID` - Property ID (scopes content to this website)
- `WEBSITE_PROJECT_ID` - WebsiteProject ID (for build task reporting)
- `NEXT_PUBLIC_SITE_URL` - Production site URL (for sitemap, canonical URLs, JSON-LD)
```

## Git Commit

```bash
git commit -m "chore: production audit complete"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 13: Production audit complete" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-14-DEPLOY.md](PHASE-14-DEPLOY.md) (if deployment enabled)
