# Changelog

All notable changes to the Figma-to-Code (f2c) skill will be documented in this file.

## [6.0.1] - 2026-02-20

### Fixed
- **Global entry upsert now queries by content type + property, not entry slug** — resolves slug mismatch where wizard creates `navigation`/`footer`/`site-settings` but PHASE-6 assumed `global-navigation`/`global-footer`/`global-settings`. The upsert now finds whatever entry exists for that content type, regardless of its slug.

### Changed (halo backend)
- **Wizard no longer creates empty component content entries** — `createPopulatedComposition()` in `content-bootstrap.service.ts` now sets `contentEntryId: null` for non-global components instead of creating `data: {}` stubs with random UUID-suffixed slugs. This prevents orphaned entries when F2C creates its own entries with different slugs. The wizard defines page structure only; content is populated by F2C or the admin editor.

## [6.0.0] - 2026-02-20

### Added
- **DB-Driven Composition** — Components and pages registered in database via `ComponentDefinition`, `ComponentVariant`, `PageTemplate`, and `PageComposition` entities
- **SEO Module Integration** — SEO data now lives in `PageSeoConfig` entities, not on content entry fields; uses `savePageSeo` / `bulkSavePageSeo` mutations
- **Renderer Toolkit** — `@halo/renderer-toolkit` shared package provides `buildMetadata()`, `buildSitemap()`, `buildRobots()`, `serveLlmsTxt()`, `buildJsonLd()`, and `<JsonLd>` component
- **WebsiteProject awareness** in Phase 1 — validate/create `WebsiteProject` during preflight
- **DesignTokenSet sync** in Phase 3 — F2C syncs Figma-extracted tokens to DB entity
- **BuildTask progress reporting** in Phase 14 — deployment updates `WebsiteProject` status
- **GEO readiness checks** in Phase 13 — llms.txt, JSON-LD, AI crawler directives

### Changed
- **Phase 2**: Renderer toolkit wiring — `@halo/renderer-toolkit` installed, env vars updated
- **Phase 5**: Content type upsert — `createOrUpdateContentType()` pattern for wizard coexistence; SEO fields removed from `page` content type (now in `PageSeoConfig`)
- **Phase 6**: Global content upsert — `upsertGlobalEntry()` handles wizard-created entries
- **Phase 7**: DB composition — component registry with `ComponentDefinition` + `ComponentVariant`
- **Phase 8**: SEO module — `savePageSeo` / `bulkSavePageSeo` replaces content entry SEO fields
- **Phase 9**: DB page composition — universal `PageRenderer`; SEO/GEO functions from renderer toolkit
- **Phase 13**: Production audit — renderer toolkit verification, SEO module checks
- **Phase 14**: Build pipeline — `BuildTask` progress reporting, `WebsiteProject` update after deployment
- **Site Framework Architecture** section updated to reference renderer toolkit and `PageSeoConfig` model

## [5.1.0] - 2026-02-19

### Fixed
- **propertyId now required on all `createContentEntry` calls** — added to GRAPHQL-EXAMPLES.md variables example and added warning that API rejects calls without it
- **propertyId now required on all `uploadMedia` calls** — added to both upload code examples in PHASE-4-ASSETS.md
- **Media must always go through the CMS** — never directly to R2 or any storage bucket. Reframed PHASE-5.5-MEDIA-EXTRACTION.md and LEARNINGS.md to make this explicit; removed misleading "upload to R2" language
- Renamed generated script from `upload-to-r2.ts` → `upload-media.ts` to reflect correct upload path

## [5.0.0] - 2026-01-31

### Added
- **EXECUTION MODEL: PHASE GROUPS & BREAKPOINTS** — comprehensive context management system
- Phase Groups (A/B/C/D) organizing 14 phases into logical units
- `/clear` breakpoints after each phase group for context optimization
- **Continuity file system** for preserving context across breakpoints
- **Dynamic breakpoints** based on project complexity assessment
- Complexity assessment matrix (components, pages, nesting depth, coverage)
- Decision matrix for breakpoint placement
- Resumption protocol for continuing after `/clear`

### Changed
- SKILL.md major restructure with execution model at top
- Enhanced phase documentation with breakpoint awareness

## [4.0.0] - 2026-01-30

### Added
- **PHASE-5.5-MEDIA-EXTRACTION.md** — dedicated media extraction phase
- Enhanced asset pipeline with better CDN integration
- Media manifest generation for tracking uploaded assets
- Improved image optimization workflow

### Changed
- PHASE-4-ASSETS.md updated with new extraction patterns
- Better separation of asset discovery vs upload

## [3.0.0] - 2026-01-29

### Added
- **NESTED-COMPONENTS.md** — comprehensive guide for handling component hierarchies
- Nested component detection and flattening strategies
- Parent-child relationship mapping
- Component composition patterns for CMS integration

### Changed
- PHASE-7-COMPONENTS.md enhanced with nesting guidance
- Improved component library organization

## [2.0.0] - 2026-01-27

### Added
- **SITE FRAMEWORK ARCHITECTURE** section in SKILL.md — comprehensive hybrid routing model
- SEO/GEO requirements: `generateMetadata()`, `sitemap.ts`, `robots.ts`, structured data (JSON-LD), canonical URLs, GEO readiness for AI crawlers
- Security requirements: slug validation regex, CMS output sanitization, content-type allowlist, published-only filter
- Performance requirements: dynamic imports for templates, ISR with on-demand revalidation, edge caching headers
- Layer responsibilities definition: f2c (first filter) → Halo backend (data contract) → Per-site (customization)
- `sitemap.ts` and `robots.ts` generation steps in PHASE-9-PAGES.md
- Slug validation (`/^[a-z0-9]+(?:[-/][a-z0-9]+)*$/`) in catch-all route (PHASE-9)
- SEO/GEO audit checklist in PHASE-13-PRODUCTION.md (metadata, sitemap, structured data, GEO readiness)
- Security audit checklist in PHASE-13-PRODUCTION.md (input validation, content safety, credentials, CSP)
- Performance audit checklist in PHASE-13-PRODUCTION.md (Core Web Vitals, bundle/caching, build verification)
- Architecture learning in LEARNINGS.md — hybrid routing rationale and trade-offs

### Changed
- PHASE-9 verification checklist expanded with routing and SEO/GEO sections
- PHASE-13 restructured from 5 steps to 8 steps with dedicated SEO, security, and performance audit sections
- `NEXT_PUBLIC_SITE_URL` added as required environment variable

## [1.9.0] - 2026-01-26

### Added
- **CONTENT STRUCTURE GUIDELINES** section in SKILL.md critical rules
- Naming conventions to avoid domain collisions (nav-, header-, footer- prefixes for UI concepts)
- Granularity rules: when to create separate entries vs embed as fields
- Entry slug patterns for uniqueness ({property}-{concept}, hero-{page})
- Enhanced PHASE-5-CONTENT-TYPES.md with detailed naming/granularity guidance
- "The Granularity Test" heuristic for decision-making

### Fixed
- Guidance to prevent over-granular content (e.g., separate entries for each address line)
- Naming collision prevention (e.g., "menu" for navigation vs restaurant menu)

## [1.8.0] - 2026-01-25

### Added
- Dynamic routing architecture (catch-all route for all content pages)
- Template system for pages
- Component library discovery in Phase 1

## [1.7.0] - 2026-01-21

### Added
- Initial version tracking and changelog
- 14-phase comprehensive workflow
- CMS content population from Figma
- Design token extraction
- Component generation with CMS data attributes
- Inline editor support
- Multi-page site generation

### Features
- Zero hardcoded content rule enforcement
- Automatic asset upload to CDN
- GraphQL integration with Halo CMS
- TypeScript component generation
- Responsive design support

---

## Version History

| Version | Date | Summary |
|---------|------|---------|
| 6.0.1 | 2026-02-20 | Fix: global entry upsert by content type (not slug); wizard stops creating orphaned component stubs |
| 6.0.0 | 2026-02-20 | DB-driven composition, SEO module, renderer toolkit, WebsiteProject awareness |
| 5.1.0 | 2026-02-19 | propertyId required on entries & media; media always via CMS, never direct R2 |
| 5.0.0 | 2026-01-31 | Execution model: phase groups, /clear breakpoints, continuity files |
| 4.0.0 | 2026-01-30 | Media extraction: dedicated phase 5.5, enhanced asset pipeline |
| 3.0.0 | 2026-01-29 | Nested components: hierarchy handling, composition patterns |
| 2.0.0 | 2026-01-27 | Site framework architecture: SEO/GEO, security, performance standards |
| 1.9.0 | 2026-01-26 | Content structure guidelines: naming conventions, granularity rules |
| 1.8.0 | 2026-01-25 | Dynamic routing, template system, component discovery |
| 1.7.0 | 2026-01-21 | Initial tracked version with full 14-phase workflow |
