# Phase 9: Page Assembly with Dynamic Routing

**CRITICAL:** This phase creates a dynamic routing system where pages are rendered based on CMS content.
New pages can be created from the CMS without code changes.

> **Architecture:** All content pages use a catch-all route (`[...slug]/page.tsx`) that:
> 1. Looks up the page entry in CMS by slug
> 2. Loads the appropriate template based on `page.template`
> 3. Renders with CMS data and inline editor attributes

---

## Step 1: Create Page Templates

Templates are React components that define how a page type renders. Create templates based on patterns identified in YOUR Figma.

### Template Directory Structure

```
src/
├── components/
│   └── templates/
│       ├── index.ts              # Template registry
│       ├── StandardTemplate.tsx  # Most content pages
│       ├── GalleryTemplate.tsx   # Image-heavy pages
│       ├── FormTemplate.tsx      # Contact/inquiry pages
│       └── ListingTemplate.tsx   # Event/blog listing pages
```

### Template Registry

```typescript
// src/components/templates/index.ts
import { StandardTemplate } from './StandardTemplate';
import { GalleryTemplate } from './GalleryTemplate';
import { FormTemplate } from './FormTemplate';
import { ListingTemplate } from './ListingTemplate';
import type { PageData } from '@/types';

// Map template names to components
const templates: Record<string, React.ComponentType<{ page: PageData }>> = {
  standard: StandardTemplate,
  gallery: GalleryTemplate,
  form: FormTemplate,
  listing: ListingTemplate,
};

/**
 * Get template component by name.
 * Falls back to StandardTemplate if template not found.
 */
export function getTemplate(templateName: string) {
  return templates[templateName] || templates.standard;
}

export { StandardTemplate, GalleryTemplate, FormTemplate, ListingTemplate };
```

### CRITICAL: Pass ALL Content Type Fields

When passing CMS content to section components, pass **ALL fields** defined in the content type - not just the ones visible in the current design.

```typescript
// ✗ WRONG - only passing fields visible in design
<HeroSection
  entry={heroSlug}
  headline={hero.headline}
  backgroundImage={hero.backgroundImage}
/>

// ✓ CORRECT - pass ALL fields from the content type
<HeroSection
  entry={heroSlug}
  headline={hero.headline}
  subtitle={hero.subtitle}
  bodyText={hero.bodyText}
  backgroundImage={hero.backgroundImage}
  backgroundImageAlt={hero.backgroundImageAlt}
  leftImage={hero.leftImage}
  leftImageAlt={hero.leftImageAlt}
  rightImage={hero.rightImage}
  rightImageAlt={hero.rightImageAlt}
  logoImage={hero.logoImage}
  logoImageAlt={hero.logoImageAlt}
  galleryImages={hero.galleryImages}
  ctaText={hero.ctaText}
  ctaUrl={hero.ctaUrl}
  // ... ALL other fields defined in content type
/>
```

**Why this matters:**
- **Page duplication** clones ALL fields - if you don't pass them, cloned content won't display
- **Inline editor** needs all fields to be editable
- **Design changes** won't require page code updates

### Standard Template Example

```typescript
// src/components/templates/StandardTemplate.tsx
import { HeroSection, ContentSection, InstagramSection, FAQSection } from '@/components/blocks';
import type { PageData } from '@/types';

interface Props {
  page: PageData;
}

export function StandardTemplate({ page }: Props) {
  const { slug, data } = page;

  return (
    <>
      {/* Hero Section - pass ALL fields */}
      {data.hero && (
        <HeroSection
          entry={slug}
          title={data.hero.title}
          subtitle={data.hero.subtitle}
          imageSrc={data.hero.imageSrc}
          // Pass ALL hero props from content type, not just visible ones
        />
      )}

      {/* Content Sections */}
      {data.sections?.map((section, index) => (
        <ContentSection
          key={section.id || index}
          entry={slug}
          fieldPrefix={`sections[${index}]`}
          heading={section.heading}
          paragraph={section.paragraph}
          imageSrc={section.imageSrc}
          variant={section.variant}
          // Pass section props based on YOUR design
        />
      ))}

      {/* Optional Global Sections */}
      {data.showInstagram && <InstagramSection />}
      {data.showFaq && <FAQSection category={data.faqCategory} />}
    </>
  );
}
```

### Create Templates for YOUR Figma Patterns

Analyze your Figma pages and identify common patterns:

| Figma Pattern | Template | Pages Using It |
|---------------|----------|----------------|
| Hero + text blocks | `standard` | About, Getting Here |
| Hero + image grid | `gallery` | Gallery, Portfolio |
| Hero + contact form | `form` | Contact, Private Events |
| Hero + item list | `listing` | Programming, Events, Blog |

**Create a template for each pattern. Don't create templates you won't use.**

---

## Step 1.5: Create Page Compositions in DB

**CRITICAL: For each page in the Figma design, create a `PageComposition` record that maps template slots to component variants.** This is the DB-driven composition model that enables the admin UI to rearrange pages.

### 1.5.1 Determine Page Templates

For each page, identify which `PageTemplate` to use. Match or create:

```graphql
mutation SavePageTemplate($input: SavePageTemplateInput!) {
  savePageTemplate(input: $input) {
    id
    slug
    name
    slots
  }
}
```

**Variables:**
```json
{
  "input": {
    "websiteProjectId": "[from Phase 1]",
    "slug": "standard",
    "name": "Standard Layout",
    "slots": [
      { "name": "header", "allowedCategories": ["header"] },
      { "name": "hero", "allowedCategories": ["hero"] },
      { "name": "main", "allowedCategories": ["section", "card", "widget"] },
      { "name": "footer", "allowedCategories": ["footer"] }
    ]
  }
}
```

### 1.5.2 Create PageComposition for Each Page

For each page in Figma, create a composition record mapping slots to components:

```graphql
mutation SavePageComposition($input: SavePageCompositionInput!) {
  savePageComposition(input: $input) {
    id
    pageEntryId
    templateId
    placements
  }
}
```

**Variables:**
```json
{
  "input": {
    "websiteProjectId": "[from Phase 1]",
    "pageEntryId": "[ID of the page content entry]",
    "templateId": "[ID of the page template]",
    "placements": [
      {
        "slotName": "header",
        "componentDefinitionSlug": "navigation",
        "componentVariantSlug": "navigation-default",
        "sortOrder": 0
      },
      {
        "slotName": "hero",
        "componentDefinitionSlug": "hero-section",
        "componentVariantSlug": "hero-split-screen",
        "sortOrder": 0,
        "contentEntrySlug": "hero-homepage"
      },
      {
        "slotName": "main",
        "componentDefinitionSlug": "content-section",
        "componentVariantSlug": "content-section-default",
        "sortOrder": 0,
        "contentEntrySlug": "section-home-intro"
      },
      {
        "slotName": "main",
        "componentDefinitionSlug": "gallery-section",
        "componentVariantSlug": "gallery-default",
        "sortOrder": 1,
        "contentEntrySlug": "gallery-homepage"
      },
      {
        "slotName": "footer",
        "componentDefinitionSlug": "footer",
        "componentVariantSlug": "footer-default",
        "sortOrder": 0
      }
    ]
  }
}
```

### 1.5.3 Overwrite Wizard Compositions

If the wizard already created a `PageComposition` for a page entry, F2C overwrites it with the Figma-accurate version. The `savePageComposition` mutation detects existing compositions by `pageEntryId` and replaces them.

### 1.5.4 Build the Universal Page Renderer

Instead of hardcoded template components that statically import sections, F2C generates a **universal page renderer** that reads composition data from the API and dynamically renders components:

```typescript
// components/page-renderer/PageRenderer.tsx
'use client';

import { componentRegistry } from './component-registry';

interface PlacedComponent {
  componentDefinitionSlug: string;
  componentVariantSlug: string;
  contentEntrySlug?: string;
  sortOrder: number;
}

interface Composition {
  [slotName: string]: PlacedComponent[];
}

export function PageRenderer({
  composition,
  slotContents,
}: {
  composition: Composition;
  slotContents: Record<string, Record<string, unknown>>;
}) {
  return (
    <>
      {Object.entries(composition).map(([slotName, components]) =>
        components
          .sort((a, b) => a.sortOrder - b.sortOrder)
          .map((placed, index) => {
            const Component = componentRegistry[placed.componentDefinitionSlug];
            const content = placed.contentEntrySlug
              ? slotContents[placed.contentEntrySlug]
              : {};
            return Component ? (
              <Component
                key={`${slotName}-${index}`}
                entry={placed.contentEntrySlug}
                {...content}
              />
            ) : null;
          })
      )}
    </>
  );
}
```

**The catch-all route fetches composition data and passes it to the renderer:**

```typescript
// In [..slug]/page.tsx
import { PageRenderer } from '@/components/page-renderer/PageRenderer';

// Fetch page composition from API
const composition = await getPageComposition(pageEntryId);
const slotContents = await resolveSlotContents(composition);

return <PageRenderer composition={composition} slotContents={slotContents} />;
```

**Note:** The template-based system from Step 1 still works as a **fallback**. If no DB composition exists for a page (e.g., it was just created in the CMS), the template system renders it. The page renderer takes precedence when a composition is available.

---

## Step 2: Create the Catch-All Route

This single route handles ALL content pages dynamically.

**CRITICAL: Slug Validation** — The catch-all route MUST validate slugs before querying the CMS. This prevents path traversal attacks (`../../admin`) and injection (`<script>`).

```typescript
// src/app/[...slug]/page.tsx
import { notFound } from 'next/navigation';
import { Metadata } from 'next';
import { buildMetadata } from '@halo/renderer-toolkit';
import { getPageBySlug, getAllPageSlugs } from '@/lib/content';
import { getTemplate } from '@/components/templates';

export const dynamic = 'force-dynamic';

/** Only allow lowercase alphanumeric, hyphens, and forward slashes */
const VALID_SLUG = /^[a-z0-9]+(?:[-/][a-z0-9]+)*$/;

interface Props {
  params: { slug: string[] };
}

/**
 * Dynamic page route.
 * Fetches page content from CMS and renders using the appropriate template.
 */
export default async function DynamicPage({ params }: Props) {
  const slug = params.slug.join('/');

  // Validate slug format before querying CMS
  if (!VALID_SLUG.test(slug)) {
    return notFound();
  }

  // Fetch page from CMS
  const page = await getPageBySlug(slug);

  // 404 if page doesn't exist
  if (!page) {
    return notFound();
  }

  // Get template component based on page.template field
  const Template = getTemplate(page.data.template || 'standard');

  return <Template page={page} />;
}

/**
 * Generate metadata from the SEO module via renderer toolkit.
 * This replaces the old approach of reading SEO from content entry fields.
 */
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const slug = params.slug.join('/');

  // Use the renderer toolkit — reads from PageSeoConfig + SiteSeoConfig
  return buildMetadata(slug, {
    apiUrl: process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL!,
    organizationId: process.env.CMS_ORGANIZATION_ID!,
    propertyId: process.env.CMS_PROPERTY_ID,
  });
}

/**
 * Pre-render known pages at build time.
 * New pages created after build will be server-rendered on demand.
 */
export async function generateStaticParams() {
  const slugs = await getAllPageSlugs();

  return slugs.map((slug) => ({
    slug: slug.split('/'),
  }));
}
```

---

## Step 3: Add Content Fetching Functions

```typescript
// src/lib/content/index.ts (add these functions)

/**
 * Fetch a page by its URL slug.
 * Returns null if page doesn't exist.
 */
export async function getPageBySlug(slug: string): Promise<PageData | null> {
  const client = getServerClient();

  try {
    const typeId = await getContentTypeId('page');

    const { data } = await client.query({
      query: GET_PAGE_BY_SLUG,
      variables: {
        contentTypeId: typeId,
        organizationId: CMS_ORG_ID,
        slug,
      },
    });

    if (!data?.contentEntryBySlug) {
      return null;
    }

    return {
      id: data.contentEntryBySlug.id,
      slug: data.contentEntryBySlug.slug,
      data: transformImageUrls(data.contentEntryBySlug.data),
    };
  } catch (error) {
    console.error(`Failed to fetch page: ${slug}`, error);
    return null;
  }
}

/**
 * Get all page slugs for static generation.
 */
export async function getAllPageSlugs(): Promise<string[]> {
  const client = getServerClient();

  try {
    const typeId = await getContentTypeId('page');

    const { data } = await client.query({
      query: GET_ALL_PAGE_SLUGS,
      variables: {
        filter: {
          contentTypeId: typeId,
          organizationId: CMS_ORG_ID,
          take: 100,
        },
      },
    });

    return data?.contentEntries?.items?.map((item: { slug: string }) => item.slug) || [];
  } catch (error) {
    console.error('Failed to fetch page slugs', error);
    return [];
  }
}
```

---

## Step 4: Create the Homepage

The homepage is the only page that needs a dedicated route (because `/` can't be handled by `[...slug]`).

```typescript
// src/app/page.tsx
import { getPageBySlug } from '@/lib/content';
import { getTemplate } from '@/components/templates';
import { notFound } from 'next/navigation';

export const dynamic = 'force-dynamic';

export default async function HomePage() {
  // Fetch homepage content (slug: 'homepage' or 'home')
  const page = await getPageBySlug('homepage');

  if (!page) {
    return notFound();
  }

  // Homepage can use a template OR have custom rendering
  // Option A: Use template system
  const Template = getTemplate(page.data.template || 'standard');
  return <Template page={page} />;

  // Option B: Custom homepage rendering (if homepage is unique)
  // return <HomepageLayout page={page} />;
}
```

---

## Step 5: Generate Sitemap (Renderer Toolkit)

**CRITICAL:** Every site MUST have a dynamic sitemap. Use the `@halo/renderer-toolkit` which reads from the CMS `rendererSitemapData` query.

```typescript
// src/app/sitemap.ts
import { buildSitemap } from '@halo/renderer-toolkit';

export default function sitemap() {
  return buildSitemap({
    apiUrl: process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL!,
    organizationId: process.env.CMS_ORGANIZATION_ID!,
    propertyId: process.env.CMS_PROPERTY_ID,
    siteUrl: process.env.NEXT_PUBLIC_SITE_URL!,
  });
}
```

---

## Step 6: Generate Robots.txt (Renderer Toolkit)

```typescript
// src/app/robots.ts
import { buildRobots } from '@halo/renderer-toolkit';

export default function robots() {
  return buildRobots({
    apiUrl: process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL!,
    organizationId: process.env.CMS_ORGANIZATION_ID!,
    propertyId: process.env.CMS_PROPERTY_ID,
    siteUrl: process.env.NEXT_PUBLIC_SITE_URL!,
  });
}
```

---

## Step 6.5: Create llms.txt Route (Renderer Toolkit)

**GEO readiness:** Serve an `llms.txt` file for AI crawlers.

```typescript
// src/app/.well-known/llms.txt/route.ts
import { serveLlmsTxt } from '@halo/renderer-toolkit';

export async function GET() {
  return serveLlmsTxt({
    apiUrl: process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL!,
    organizationId: process.env.CMS_ORGANIZATION_ID!,
    propertyId: process.env.CMS_PROPERTY_ID,
  });
}
```

---

## Step 6.6: Add JSON-LD Structured Data (Renderer Toolkit)

Add the `<JsonLd>` component to the root layout for site-wide structured data:

```typescript
// In src/app/layout.tsx
import { JsonLd, buildJsonLd } from '@halo/renderer-toolkit';

// In the body, after <Footer />:
const jsonLdData = await buildJsonLd('/', {
  apiUrl: process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL!,
  organizationId: process.env.CMS_ORGANIZATION_ID!,
  propertyId: process.env.CMS_PROPERTY_ID,
  siteUrl: process.env.NEXT_PUBLIC_SITE_URL!,
});

// Render in layout:
<JsonLd data={jsonLdData} />
```

For per-page structured data, add `<JsonLd>` in the catch-all route:

```typescript
// In [..slug]/page.tsx, before the Template render:
const pageJsonLd = await buildJsonLd(slug, {
  apiUrl: process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL!,
  organizationId: process.env.CMS_ORGANIZATION_ID!,
  propertyId: process.env.CMS_PROPERTY_ID,
  siteUrl: process.env.NEXT_PUBLIC_SITE_URL!,
});

return (
  <>
    <JsonLd data={pageJsonLd} />
    <Template page={page} />
  </>
);
```

---

## Step 7: Handle Application Pages (Optional)

Some pages may have complex interactive logic that doesn't fit the template system.
These are **application pages**, not content pages.

**Examples:**
- Menu page with category tabs and filtering
- Checkout flow with state management
- Dashboard with authentication

**For application pages, create dedicated routes:**

```
src/app/
├── page.tsx              # Homepage (content page, uses templates)
├── menu/page.tsx         # Menu (application page, custom logic)
├── checkout/             # Checkout (application pages)
│   ├── page.tsx
│   └── confirm/page.tsx
└── [...slug]/page.tsx    # All other content pages (dynamic)
```

> **Rule of thumb:** If a page is primarily displaying CMS content, use the catch-all route.
> If a page has complex client-side state or interactions, create a dedicated route.

---

## Step 6: Update Root Layout

Ensure the root layout fetches global content:

```typescript
// src/app/layout.tsx
import { getSiteSettings, getNavigation, getFooterContent } from '@/lib/content';
import { Navigation, Footer } from '@/components/layout';
import { InlineEditorLoader } from '@/components/InlineEditorLoader';

export const dynamic = 'force-dynamic';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const [siteSettings, navigation, footer] = await Promise.all([
    getSiteSettings(),
    getNavigation(),
    getFooterContent(),
  ]);

  return (
    <html lang="en">
      <body>
        <Navigation
          menuLinks={navigation.menuLinks}
          logoUrl={siteSettings.logoUrl}
          logoAlt={siteSettings.logoAlt}
          ctaText={navigation.ctaText}
          ctaUrl={navigation.ctaUrl}
        />

        <main>{children}</main>

        <Footer
          footerLinks={footer.footerLinks}
          copyrightText={footer.copyrightText}
          // ... other footer props
        />

        {/* Inline editor script loader */}
        <InlineEditorLoader />
      </body>
    </html>
  );
}
```

---

## Step 7: TypeScript Types

```typescript
// src/types/content.ts

export interface PageData {
  id: string;
  slug: string;
  data: {
    // Identity
    title: string;
    template: 'standard' | 'gallery' | 'form' | 'listing';

    // SEO
    metaTitle?: string;
    metaDescription?: string;
    ogImage?: string;

    // Content (varies by template)
    hero?: {
      title: string;
      subtitle?: string;
      imageSrc?: string;
      // ... based on YOUR hero design
    };
    sections?: Array<{
      id?: string;
      heading?: string;
      paragraph?: string;
      imageSrc?: string;
      variant?: string;
      // ... based on YOUR section design
    }>;

    // Optional global sections
    showInstagram?: boolean;
    showFaq?: boolean;
    faqCategory?: string;
  };
}
```

---

## Verification Checklist

### Routing
- [ ] Template registry exports all templates
- [ ] Catch-all route fetches page by slug
- [ ] Catch-all route validates slug format before CMS query
- [ ] Catch-all route returns 404 for missing pages
- [ ] Catch-all route returns 404 for invalid slugs (e.g., `../../admin`, `<script>`)
- [ ] Templates include all `data-cms-*` attributes
- [ ] Homepage renders correctly at `/`
- [ ] Existing pages render via catch-all (e.g., `/about`)
- [ ] New page created in CMS is immediately accessible
- [ ] Inline editing works on dynamic pages

### SEO / GEO (Renderer Toolkit)
- [ ] `generateMetadata()` uses `buildMetadata()` from `@halo/renderer-toolkit`
- [ ] `sitemap.ts` uses `buildSitemap()` from toolkit — reads from CMS
- [ ] `robots.ts` uses `buildRobots()` from toolkit — includes AI crawler directives
- [ ] `app/.well-known/llms.txt/route.ts` uses `serveLlmsTxt()` from toolkit
- [ ] `<JsonLd>` component added to root layout for site-wide structured data
- [ ] Per-page `<JsonLd>` added in catch-all route
- [ ] Canonical URLs managed in `PageSeoConfig` (SEO module, not content entry)
- [ ] No SEO data stored as content entry fields (all in `PageSeoConfig`)
- [ ] `NEXT_PUBLIC_SITE_URL` environment variable set

---

## Testing Dynamic Routing

1. **Test existing page:** Navigate to `/about` — should render via catch-all
2. **Test 404:** Navigate to `/nonexistent` — should show 404 page
3. **Test new page creation:**
   - Create a new `page` entry in CMS with slug `test-page`
   - Navigate to `/test-page` — should render immediately
4. **Test inline editing:** Add `?edit=true` — should enable editing

---

## Git Commit

```bash
git commit -m "feat: Dynamic page routing with template system

- Added catch-all route for CMS-driven pages
- Created template components (standard, gallery, form, listing)
- Homepage uses template system
- Pages render based on CMS template field
- New pages work immediately without code changes"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 9: Dynamic routing complete" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-10-AUDIT.md](PHASE-10-AUDIT.md)
