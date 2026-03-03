# Phase 5: CMS Content Type Setup

Create content types based on **Figma component patterns**, not individual pages.

> **CRITICAL INSIGHT:** Content Types should map to reusable Figma components.
> Pages then reference these component entries, enabling reuse across the site.

---

## Pre-Requisite: Read Component Analysis

**Before creating content types, read the outputs from Phase 1:**

1. `/project-config/components.md` — Component inventory with nested relationships
2. `/project-config/component-coverage.md` — Coverage analysis and build order

These files tell you:
- Which components exist in Figma
- Which components are nested inside others
- The dependency order for building

---

## The Three-Tier Content Model

### Why Component-Based?

| Approach | Problem |
|----------|---------|
| One CT per page | Duplicates fields, can't share content between pages |
| One CT for everything | Too generic, hard to validate |
| **Component-based** | Reusable, validates well, maps to Figma design |

### The Tier System

```
┌─────────────────────────────────────────────────────────────────┐
│                     TIER 1: GLOBAL (3 CTs)                      │
│  site-settings  │  navigation  │  footer                        │
│  (1 entry each - site-wide configuration)                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   TIER 2: COMPONENTS (varies)                   │
│  hero-section  │  content-section  │  gallery  │  faq-item      │
│  (N entries each - reusable building blocks from Figma)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     TIER 3: PAGES (1 CT)                        │
│  page                                                           │
│  (N entries - references component entries above)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Analyze Figma Components

Before creating content types, analyze your Figma file's Components page.

**Look for:**
1. **Hero variants** — How many hero styles exist? (split-screen, full-screen, gallery, etc.)
2. **Section patterns** — What reusable section layouts appear? (half-screen, centered text, etc.)
3. **Card types** — What card components exist? (event cards, team members, awards, etc.)
4. **Global components** — Navigation, footer, menu overlay

**Document in `/project-config/content-model.md`:**

```markdown
# Content Model Plan

## Figma Components Found
- hero-option-1 (split-screen, full-screen variants)
- hero-option-2 (standard hero)
- section-half-screen (image + text, alternating)
- gallery (6-image grid)
- faq-accordion (question/answer)
- navigation (desktop/mobile)
- footer

## Proposed Content Types
[Map each component to a content type]
```

---

## Step 1.5: Check for Existing Content Types (Wizard Reconciliation)

**CRITICAL: The wizard may have already created content types.** Before creating any content type, check if it already exists. Use an **upsert pattern** — update if exists, create if not.

### Why This Matters

The website creation wizard now creates only a minimal `page` content type (with `title`, `slug`, `template` fields). F2C owns the creation of all design-specific content types. However, if a wizard-created project exists, the `page` type may already be present.

### Upsert Pattern

```typescript
/**
 * Create or update a content type. Checks by slug first.
 * If the content type exists, updates its fields.
 * If it doesn't exist, creates it from scratch.
 */
async function createOrUpdateContentType(
  slug: string,
  name: string,
  fields: FieldDefinition[]
) {
  // Step 1: Check if content type exists by querying with slug
  const existing = await findContentTypeBySlug(slug);

  if (existing) {
    // Step 2a: Update — merge new fields with existing ones
    // Existing fields not in the new set are kept (no data loss)
    // New fields are added
    // Existing fields with matching slugs get their config updated
    console.log(`Content type "${slug}" exists (ID: ${existing.id}) — updating fields`);

    await updateContentType(existing.id, {
      name,
      fields: mergeFields(existing.fields, fields),
    });

    return existing.id;
  } else {
    // Step 2b: Create from scratch
    console.log(`Content type "${slug}" not found — creating`);

    const result = await createContentType({
      organizationId: config.cmsOrganizationId,
      slug,
      name,
      fields,
    });

    return result.id;
  }
}

/**
 * Find a content type by slug within the organization.
 */
async function findContentTypeBySlug(slug: string) {
  const { data } = await client.query({
    query: GET_CONTENT_TYPES,
    variables: {
      filter: {
        organizationId: config.cmsOrganizationId,
        slug,
      },
    },
  });

  return data?.contentTypes?.items?.[0] || null;
}
```

### Special Case: The `page` Content Type

The wizard creates a minimal `page` type with 3 fields: `title`, `slug`, `template`. F2C extends this with additional fields (SEO, hero references, section references, etc.). When the `page` type already exists:

1. **Keep existing fields** — don't remove `title`, `slug`, `template`
2. **Add new fields** — SEO fields, hero slug, sections array, etc.
3. **Don't create duplicate** — use `updateContentType` to add fields

```typescript
// The page content type may already exist from the wizard
const pageTypeId = await createOrUpdateContentType('page', 'Page', [
  // Required (may already exist from wizard)
  { slug: 'title', name: 'Page Title', type: 'text', required: true },
  { slug: 'slug', name: 'URL Slug', type: 'text', required: true },
  { slug: 'template', name: 'Page Template', type: 'text', required: true },

  // F2C additions — hero reference, sections, toggles
  { slug: 'heroSlug', name: 'Hero Section Slug', type: 'text' },
  { slug: 'sections', name: 'Page Sections', type: 'json' },
  { slug: 'showInstagram', name: 'Show Instagram Feed', type: 'boolean' },
  { slug: 'showFaq', name: 'Show FAQ Section', type: 'boolean' },
  { slug: 'faqCategory', name: 'FAQ Category Filter', type: 'text' },

  // Note: SEO fields (metaTitle, metaDescription, ogImage) are NO LONGER
  // stored on the page content type. They live in PageSeoConfig (SEO module).
  // See Phase 8 for SEO module integration.
]);
```

---

## Step 2: Create Tier 1 Content Types (Global)

These have exactly **one entry** each.

### Site Settings

```typescript
await createOrUpdateContentType('site-settings', 'Site Settings', [
  // Basic info
  { slug: 'siteName', name: 'Site Name', type: 'text', required: true },
  { slug: 'siteDescription', name: 'Site Description', type: 'text' },

  // Logos
  { slug: 'logoUrl', name: 'Primary Logo', type: 'media', required: true },
  { slug: 'logoAlt', name: 'Logo Alt Text', type: 'text', required: true },
  { slug: 'logoAltUrl', name: 'Alternative Logo (Footer)', type: 'media' },

  // Contact
  { slug: 'phone', name: 'Phone Number', type: 'text' },
  { slug: 'email', name: 'Email Address', type: 'text' },
  { slug: 'address', name: 'Full Address', type: 'json' }, // {street, city, state, zip}

  // Social & External
  { slug: 'socialLinks', name: 'Social Links', type: 'json' }, // [{platform, url}]
  { slug: 'instagramHandle', name: 'Instagram Handle', type: 'text' },
  { slug: 'reservationUrl', name: 'Reservation URL', type: 'text' },

  // Hours (if applicable)
  { slug: 'hours', name: 'Operating Hours', type: 'json' }, // [{days, open, close}]
]);
```

### Navigation

```typescript
await createOrUpdateContentType('navigation', 'Navigation', [
  {
    slug: 'menuLinks',
    name: 'Menu Links',
    type: 'json',
    required: true,
    // JSON Schema for structured link editing
    validation: {
      schema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            label: { type: 'string', minLength: 1 },
            url: { type: 'string', pattern: '^(https?://|/|#)' },
            isExternal: { type: 'boolean' }
          },
          required: ['label', 'url']
        }
      }
    }
  },

  { slug: 'ctaText', name: 'CTA Button Text', type: 'text' },
  { slug: 'ctaUrl', name: 'CTA Button URL', type: 'text' },

  // Optional mobile-specific
  { slug: 'mobileMenuLinks', name: 'Mobile Menu Links', type: 'json' },
]);
```

### Footer

```typescript
// Reusable links schema for structured editing
const linksSchema = {
  type: 'array',
  items: {
    type: 'object',
    properties: {
      label: { type: 'string', minLength: 1 },
      url: { type: 'string' }
    },
    required: ['label', 'url']
  }
};

await createOrUpdateContentType('footer', 'Footer', [
  // Column content - adapt to YOUR Figma footer design
  { slug: 'column1Title', name: 'Column 1 Title', type: 'text' },
  { slug: 'column1Content', name: 'Column 1 Content', type: 'json' },
  { slug: 'column2Title', name: 'Column 2 Title', type: 'text' },
  { slug: 'column2Content', name: 'Column 2 Content', type: 'json' },
  {
    slug: 'column3Links',
    name: 'Column 3 Links',
    type: 'json',
    validation: { schema: linksSchema }
  },

  // Newsletter (if present)
  { slug: 'newsletterHeading', name: 'Newsletter Heading', type: 'text' },
  { slug: 'newsletterPlaceholder', name: 'Newsletter Placeholder', type: 'text' },

  // Legal
  { slug: 'copyrightText', name: 'Copyright Text', type: 'text' },
  {
    slug: 'legalLinks',
    name: 'Legal Links',
    type: 'json',
    validation: { schema: linksSchema }
  },
]);
```

---

## Step 3: Create Tier 2 Content Types (Components)

These map to Figma components and can have **multiple entries**.

### Hero Section

Create ONLY if your Figma has hero components:

```typescript
await createOrUpdateContentType('hero-section', 'Hero Section', [
  // Variant selection based on Figma hero types
  { slug: 'variant', name: 'Hero Variant', type: 'text', required: true },
  // Options: 'split-screen', 'full-screen', 'gallery', 'text-only'

  // Content fields
  { slug: 'headline', name: 'Headline', type: 'text' },
  { slug: 'subtitle', name: 'Subtitle', type: 'text' },
  { slug: 'bodyText', name: 'Body Text', type: 'richText' },

  // Media
  { slug: 'backgroundImage', name: 'Background Image', type: 'media' },
  { slug: 'backgroundVideo', name: 'Background Video URL', type: 'text' },
  { slug: 'images', name: 'Gallery Images', type: 'json' }, // For gallery variant

  // CTA
  { slug: 'ctaText', name: 'CTA Button Text', type: 'text' },
  { slug: 'ctaUrl', name: 'CTA Button URL', type: 'text' },

  // Layout
  { slug: 'textAlignment', name: 'Text Alignment', type: 'text' }, // left, center, right
  { slug: 'overlayOpacity', name: 'Overlay Opacity', type: 'number' },
]);
```

### Content Section

For half-screen sections, text blocks, etc.:

```typescript
await createOrUpdateContentType('content-section', 'Content Section', [
  // Variant based on Figma section patterns
  { slug: 'variant', name: 'Section Variant', type: 'text', required: true },
  // Options: 'half-screen-left', 'half-screen-right', 'text-centered', 'text-left'

  // Content
  { slug: 'heading', name: 'Heading', type: 'text' },
  { slug: 'subheading', name: 'Subheading', type: 'text' },
  { slug: 'bodyText', name: 'Body Text', type: 'richText' },
  { slug: 'image', name: 'Section Image', type: 'media' },

  // CTA
  { slug: 'ctaText', name: 'CTA Text', type: 'text' },
  { slug: 'ctaUrl', name: 'CTA URL', type: 'text' },

  // Style
  { slug: 'backgroundColor', name: 'Background Color', type: 'text' }, // light, dark, pink
]);
```

### Gallery

```typescript
await createOrUpdateContentType('gallery', 'Gallery', [
  { slug: 'name', name: 'Gallery Name', type: 'text', required: true },
  {
    slug: 'images',
    name: 'Images',
    type: 'json',
    required: true,
    validation: {
      schema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            src: { type: 'string', minLength: 1 },
            alt: { type: 'string', minLength: 1 },
            caption: { type: 'string' }
          },
          required: ['src', 'alt']
        }
      }
    }
  },
]);
```

### FAQ Item

```typescript
await createOrUpdateContentType('faq-item', 'FAQ Item', [
  { slug: 'question', name: 'Question', type: 'text', required: true },
  { slug: 'answer', name: 'Answer', type: 'richText', required: true },
  { slug: 'category', name: 'Category', type: 'text' }, // For filtering
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);
```

### Additional Component Types (Create as needed)

**Event/Programming:**
```typescript
await createOrUpdateContentType('event', 'Event', [
  { slug: 'title', name: 'Title', type: 'text', required: true },
  { slug: 'date', name: 'Date', type: 'text', required: true },
  { slug: 'time', name: 'Time', type: 'text' },
  { slug: 'description', name: 'Description', type: 'richText' },
  { slug: 'image', name: 'Image', type: 'media' },
  { slug: 'category', name: 'Category', type: 'text' },
  { slug: 'ticketUrl', name: 'Ticket URL', type: 'text' },
  { slug: 'isFeatured', name: 'Featured', type: 'boolean' },
]);
```

**Team Member:**
```typescript
await createOrUpdateContentType('team-member', 'Team Member', [
  { slug: 'name', name: 'Name', type: 'text', required: true },
  { slug: 'title', name: 'Job Title', type: 'text' },
  { slug: 'photo', name: 'Photo', type: 'media' },
  { slug: 'bio', name: 'Biography', type: 'richText' },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);
```

**Menu Category (for restaurants):**
```typescript
// Example for restaurant sites - adapt to your industry
await createOrUpdateContentType('menu-category', 'Menu Category', [
  { slug: 'name', name: 'Category Name', type: 'text', required: true },
  { slug: 'availability', name: 'Availability', type: 'text' },
  {
    slug: 'items',
    name: 'Menu Items',
    type: 'json',
    required: true,
    // JSON Schema enables STRUCTURED EDITING in admin UI
    validation: {
      schema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            name: { type: 'string', minLength: 1 },
            description: { type: 'string' },
            price: { type: 'string' }
          },
          required: ['name']
        }
      }
    }
  },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);
```

**Award/Recognition:**
```typescript
await createOrUpdateContentType('award', 'Award', [
  { slug: 'name', name: 'Award Name', type: 'text', required: true },
  { slug: 'organization', name: 'Organization', type: 'text' },
  { slug: 'logo', name: 'Logo', type: 'media' },
  { slug: 'year', name: 'Year', type: 'text' },
  { slug: 'url', name: 'URL', type: 'text' },
]);
```

---

## Step 4: Create Tier 3 Content Type (Page)

A single flexible page type that references component entries and specifies a rendering template.

> **CRITICAL:** The `template` field enables dynamic routing. The website uses a catch-all route
> that renders pages based on their template. This allows new pages to be created from the CMS
> without code changes.

```typescript
// Use the upsert pattern from Step 1.5 — the wizard may have
// already created a minimal page type that we need to extend.
await createOrUpdateContentType('page', 'Page', [
  // Identity (may already exist from wizard)
  { slug: 'title', name: 'Page Title', type: 'text', required: true },
  { slug: 'slug', name: 'URL Slug', type: 'text', required: true },

  // Template Selection (REQUIRED for dynamic routing)
  { slug: 'template', name: 'Page Template', type: 'select', required: true,
    validation: { options: ['standard', 'gallery', 'form', 'listing'] },
    uiConfig: { helpText: 'Determines the layout and components used to render this page' } },

  // Accessibility
  { slug: 'pageAnnouncement', name: 'Screen Reader Announcement', type: 'text' },

  // Hero reference
  { slug: 'heroSlug', name: 'Hero Section Slug', type: 'text' },

  // Sections (ordered array of references)
  { slug: 'sections', name: 'Page Sections', type: 'json' },
  // Format: [{type: 'content-section', slug: 'homepage-intro'}, ...]

  // Global section toggles
  { slug: 'showInstagram', name: 'Show Instagram Feed', type: 'boolean' },
  { slug: 'showFaq', name: 'Show FAQ Section', type: 'boolean' },
  { slug: 'faqCategory', name: 'FAQ Category Filter', type: 'text' },

  // NOTE: SEO fields (metaTitle, metaDescription, ogImage, canonicalUrl,
  // noIndex) are NOT stored on this content type. They live exclusively
  // in PageSeoConfig entities (SEO module). See Phase 8 for details.
]);
```

### Available Templates

Identify templates based on YOUR Figma page patterns:

| Template | Use For | Typical Sections |
|----------|---------|------------------|
| `standard` | Most content pages | Hero + text sections + optional CTA |
| `gallery` | Portfolio, photos | Hero + image grid + lightbox |
| `form` | Contact, inquiries | Hero + form + info sidebar |
| `listing` | Events, blog, team | Hero + filterable list + pagination |

**Customize templates based on your Figma.** If your design has unique patterns, create additional templates.
```

---

## Nested Component Content Modeling

**CRITICAL: When a Figma component contains other components, you must decide how to model the content.**

### The Nesting Problem

Consider this hierarchy from Phase 1's component analysis:

```
navigation-desktop
├── logo (NoMad-Wynwood__Mark-01)
├── menu-link (×5)
│   └── (has label, href)
└── button (CTA)
    └── (has text, href)
```

**Question:** How do we model this in CMS?

### Decision Tree: Composition vs Embedding

| Scenario | Approach | Why |
|----------|----------|-----|
| Child is reused across many parents | **Composition** (reference) | FAQ items used on FAQ page AND About page |
| Child is only used within this parent | **Embedding** (JSON) | Menu links only exist in navigation |
| Child has 3+ fields with validation | **Composition** (reference) | Better admin UX with form fields |
| Child is simple (1-2 fields) | **Embedding** (JSON) | Avoids admin clutter |
| Child needs independent editing | **Composition** (reference) | Different editors manage different content |
| Parent-child always edited together | **Embedding** (JSON) | Navigation links change with navigation |

### Pattern 1: Embedding (Nested JSON)

**Use when:** Children are tightly coupled to parent, simple structure.

```typescript
// navigation content type — menu-links embedded as JSON
await createOrUpdateContentType('navigation', 'Navigation', [
  { slug: 'logoUrl', name: 'Logo', type: 'media', required: true },
  {
    slug: 'menuLinks',
    name: 'Menu Links',
    type: 'json',
    validation: {
      schema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            label: { type: 'string', minLength: 1 },
            href: { type: 'string' },
            isActive: { type: 'boolean' }
          },
          required: ['label', 'href']
        }
      }
    }
  },
  { slug: 'ctaText', name: 'CTA Button Text', type: 'text' },
  { slug: 'ctaUrl', name: 'CTA Button URL', type: 'text' },
]);
```

**React component receives:**
```typescript
interface NavigationProps {
  entry: string;
  logoUrl: string;
  menuLinks: Array<{ label: string; href: string; isActive?: boolean }>;
  ctaText?: string;
  ctaUrl?: string;
}
```

**CMS data attributes for embedded arrays:**
```tsx
{menuLinks.map((link, index) => (
  <a
    key={link.href}
    href={link.href}
    data-cms-entry="global-navigation"
    data-cms-field={`menuLinks[${index}].label`}
  >
    {link.label}
  </a>
))}
```

### Pattern 2: Composition (References)

**Use when:** Children are reused independently, complex structure, or need separate editing.

```typescript
// faq-item content type — exists independently
await createOrUpdateContentType('faq-item', 'FAQ Item', [
  { slug: 'question', name: 'Question', type: 'text', required: true },
  { slug: 'answer', name: 'Answer', type: 'richText', required: true },
  { slug: 'category', name: 'Category', type: 'text' },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);

// page content type — references FAQ items
await createOrUpdateContentType('page', 'Page', [
  // ... other fields
  {
    slug: 'faqItems',
    name: 'FAQ Items',
    type: 'json',
    // Array of slugs referencing faq-item entries
    // Format: ["faq-reservations", "faq-parking", "faq-hours"]
  },
]);
```

**Fetching composed content:**
```typescript
// In content layer, resolve references
async function getPageWithFaqs(slug: string) {
  const page = await getPageBySlug(slug);

  // Resolve FAQ references
  if (page.data.faqItems?.length) {
    const faqs = await Promise.all(
      page.data.faqItems.map(faqSlug => getFaqItem(faqSlug))
    );
    page.data.resolvedFaqs = faqs;
  }

  return page;
}
```

**CMS data attributes for composed content:**
```tsx
// Each FAQ item has its own entry slug
{faqs.map((faq) => (
  <div key={faq.slug}>
    <h3
      data-cms-entry={faq.slug}  // e.g., "faq-reservations"
      data-cms-field="question"
    >
      {faq.question}
    </h3>
    <div
      data-cms-entry={faq.slug}
      data-cms-field="answer"
      data-cms-type="richtext"
    >
      {faq.answer}
    </div>
  </div>
))}
```

### Pattern 3: Hybrid (Parent Owns Layout, References Children)

**Use when:** Parent defines how children are arranged, but children are reusable.

```typescript
// hero-section owns layout decisions, references button entries for CTA
await createOrUpdateContentType('hero-section', 'Hero Section', [
  { slug: 'variant', name: 'Variant', type: 'text', required: true },
  { slug: 'headline', name: 'Headline', type: 'text' },
  { slug: 'subtitle', name: 'Subtitle', type: 'text' },
  { slug: 'backgroundImage', name: 'Background', type: 'media' },

  // CTA is embedded (tight coupling) — button text/url specific to this hero
  { slug: 'ctaText', name: 'CTA Text', type: 'text' },
  { slug: 'ctaUrl', name: 'CTA URL', type: 'text' },
  { slug: 'ctaStyle', name: 'CTA Style', type: 'text' }, // 'primary', 'secondary'

  // Gallery images embedded as JSON (only used in this hero)
  { slug: 'galleryImages', name: 'Gallery Images', type: 'json' },
]);
```

### Nested Component Examples from Figma

Based on typical `/project-config/components.md` output:

| Parent Component | Nested Children | Modeling Decision |
|------------------|-----------------|-------------------|
| `navigation-desktop` | logo, menu-link, button | **Embed** — links are nav-specific |
| `footer` | link groups, social icons | **Embed** — footer-specific content |
| `hero-option-1` | button | **Embed** — CTA is hero-specific |
| `event-card` | button | **Embed** — button text is event-specific |
| `faq-section` | faq-accordion | **Compose** — FAQs reused across pages |
| `team-section` | team-member | **Compose** — members managed independently |
| `gallery-section` | gallery images | **Embed** — images are gallery-specific |

### Impact on Phase 7 Components

When building React components, respect the content modeling:

**For Embedded Children:**
```tsx
// Button content is props from parent
interface HeroSectionProps {
  ctaText?: string;
  ctaUrl?: string;
}

function HeroSection({ ctaText, ctaUrl, entry }: HeroSectionProps) {
  return (
    <section>
      {/* ... */}
      {ctaText && (
        <Button
          href={ctaUrl}
          data-cms-entry={entry}
          data-cms-field="ctaText"
        >
          {ctaText}
        </Button>
      )}
    </section>
  );
}
```

**For Composed Children:**
```tsx
// Each child has its own entry
interface FaqSectionProps {
  sectionTitle: string;
  faqs: Array<{ slug: string; question: string; answer: string }>;
}

function FaqSection({ sectionTitle, faqs, entry }: FaqSectionProps) {
  return (
    <section>
      <h2 data-cms-entry={entry} data-cms-field="sectionTitle">
        {sectionTitle}
      </h2>
      {faqs.map((faq) => (
        <FaqAccordion
          key={faq.slug}
          entry={faq.slug}  // Child has its OWN entry
          question={faq.question}
          answer={faq.answer}
        />
      ))}
    </section>
  );
}
```

---

## Example: Content Type Creation Script

```typescript
// scripts/create-content-types.ts
import { createOrUpdateContentType } from '@/lib/cms';

async function createAllContentTypes() {
  // TIER 1: Global
  await createOrUpdateContentType('site-settings', 'Site Settings', [...]);
  await createOrUpdateContentType('navigation', 'Navigation', [...]);
  await createOrUpdateContentType('footer', 'Footer', [...]);

  // TIER 2: Components (based on YOUR Figma)
  await createOrUpdateContentType('hero-section', 'Hero Section', [...]);
  await createOrUpdateContentType('content-section', 'Content Section', [...]);
  await createOrUpdateContentType('gallery', 'Gallery', [...]);
  await createOrUpdateContentType('faq-item', 'FAQ Item', [...]);
  // Add more based on your Figma components

  // TIER 3: Page
  await createOrUpdateContentType('page', 'Page', [...]);

  console.log('All content types created!');
}
```

---

## Content Type Naming Conventions

### Content Type Slugs

| Tier | Pattern | Examples |
|------|---------|----------|
| Global | `site-{name}` or just `{name}` | `site-settings`, `navigation`, `footer` |
| Component | `{component-name}` | `hero-section`, `content-section`, `gallery` |
| Page | `page` (single type) | One content type, many entries |

### Entry Slug Conventions

**CRITICAL:** Entry slugs must be **unique** and **descriptive** to avoid admin confusion.

| Content Type | Entry Slug Pattern | Examples |
|--------------|-------------------|----------|
| hero-section | `hero-{page}` | `hero-homepage`, `hero-about`, `hero-contact` |
| content-section | `section-{page}-{purpose}` | `section-about-story`, `section-home-intro` |
| gallery | `gallery-{page}` or `gallery-{name}` | `gallery-homepage`, `gallery-events` |
| faq-item | `faq-{topic}` | `faq-reservations`, `faq-parking` |
| team-member | `team-{name}` | `team-john-doe`, `team-jane-smith` |
| page | `{url-slug}` | `about`, `contact`, `events` |

**For multi-property sites, prefix with property:**
```
wynwood-hours, run-house-hours
wynwood-contact, run-house-contact
hero-wynwood-homepage, hero-run-homepage
```

### Avoiding Domain Collisions

**Problem:** "menu" means navigation in web dev but food items in restaurant context.

**Solution:** Use prefixes for UI concepts:

| Concept | Prefix | Slug Examples |
|---------|--------|---------------|
| Navigation | `nav-` | `nav-links`, `nav-menu`, `nav-item` |
| Header | `header-` | `header-cta`, `header-logo` |
| Footer | `footer-` | `footer-links`, `footer-column` |
| Site-wide | `site-` | `site-settings`, `site-logo` |

**Domain terms use NO prefix:**
- Restaurant: `menu-category`, `menu-item`, `dish`
- Hotel: `room-type`, `amenity`, `rate`
- Retail: `product`, `collection`

---

## Granularity Guidelines

**CRITICAL:** Not everything needs to be a separate content entry. Over-granularity creates admin clutter.

### When to Create Separate Entries

Create a separate content type and entries when:

| Criteria | Example |
|----------|---------|
| **Reused across pages** | Team members shown on About + Team pages |
| **Managed as a list** | FAQ items, events, menu items |
| **Complex structure (3+ fields)** | Team member: name, title, photo, bio |
| **Independent edit lifecycle** | Events updated separately from page |

### When to Keep as Embedded Fields

Keep as a single JSON field when:

| Criteria | Example |
|----------|---------|
| **Always edited together** | Address: street, city, state, zip |
| **Simple structure (1-2 fields)** | CTA: text, url |
| **Only used in one place** | Page-specific config |
| **Would create admin clutter** | Each address line as separate entry |

### Field Structure Examples

**GOOD: Embedded (fields change together)**
```typescript
// site-settings content type
{ slug: 'address', name: 'Address', type: 'json' }
// Entry data: { street: "123 Main", city: "Miami", state: "FL", zip: "33139" }

{ slug: 'hours', name: 'Business Hours', type: 'json' }
// Entry data: [{ days: "Mon-Fri", open: "9am", close: "5pm" }]

{ slug: 'socialLinks', name: 'Social Links', type: 'json' }
// Entry data: [{ platform: "instagram", url: "..." }]
```

**GOOD: Separate entries (managed independently)**
```typescript
// team-member content type (separate entries for each person)
// Entry: team-john-doe { name: "John Doe", title: "Chef", photo: "...", bio: "..." }
// Entry: team-jane-smith { name: "Jane Smith", title: "Manager", ... }

// faq-item content type (separate entries for each question)
// Entry: faq-reservations { question: "How do I reserve?", answer: "..." }
// Entry: faq-parking { question: "Is there parking?", answer: "..." }
```

**BAD: Over-granular (creates admin clutter)**
```typescript
// ❌ DON'T: Separate entries for each address component
// Entry: address-street { value: "123 Main St" }
// Entry: address-city { value: "Miami" }
// Entry: address-state { value: "FL" }

// ❌ DON'T: Separate entries for each hour row
// Entry: hours-monday { open: "9am", close: "5pm" }
// Entry: hours-tuesday { open: "9am", close: "5pm" }
```

### The Granularity Test

Ask: **"Would a content editor want to manage this independently?"**

- "Add a new team member" → YES → Separate entry
- "Change the street address" → NO → Embedded field
- "Reorder FAQ questions" → YES → Separate entries
- "Update phone and email" → NO → Embedded in contact field

---

## Documentation

Update `.claude-checkpoint.json`:

```json
{
  "phase": 5,
  "contentTypes": {
    "tier1": {
      "site-settings": "ct_abc123",
      "navigation": "ct_def456",
      "footer": "ct_ghi789"
    },
    "tier2": {
      "hero-section": "ct_jkl012",
      "content-section": "ct_mno345",
      "gallery": "ct_pqr678",
      "faq-item": "ct_stu901"
    },
    "tier3": {
      "page": "ct_vwx234"
    }
  },
  "contentModel": "component-based"
}
```

---

## Key Rules Reminder

### 1. Create Types BEFORE Entries
Always create content types first. Creating entries before types results in a JSON blob in the admin UI instead of form fields.

### 2. Use FLAT Field Slugs
```typescript
// ✅ CORRECT
{ slug: 'heroTitle', name: 'Hero Title', type: 'text' }

// ❌ WRONG
{ slug: 'hero.title', name: 'Hero Title', type: 'text' }
```

### 3. Only Create What Exists in Figma
Don't blindly copy these examples. Create content types ONLY for components that appear in YOUR Figma design.

### 4. Component-Based Thinking
Ask: "Is this a reusable pattern in Figma?" If yes, it's probably a Tier 2 content type.

### 5. Use JSON Schema for Structured Editing
When a JSON field has a defined schema, the admin UI shows a **structured form** instead of a raw JSON editor:

```typescript
// WITHOUT schema: Admin shows raw JSON textarea
{ slug: 'items', name: 'Items', type: 'json' }

// WITH schema: Admin shows structured form with individual fields
{
  slug: 'items',
  name: 'Items',
  type: 'json',
  validation: {
    schema: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          name: { type: 'string', minLength: 1 },
          price: { type: 'string' }
        },
        required: ['name']
      }
    }
  }
}
```

**Benefits of JSON Schema:**
- Content editors see form fields, not JSON syntax
- Add/remove/reorder array items with buttons
- Validation happens on save (required fields, patterns, enums)
- Toggle between structured and raw JSON modes

**See [JSON-FIELD-SCHEMA.md](./JSON-FIELD-SCHEMA.md) for common schema patterns.**

---

## Git Commit

```bash
git commit -m "feat: Component-based content types created

- Tier 1: Global types (site-settings, navigation, footer)
- Tier 2: Component types (hero-section, content-section, gallery, faq-item, etc.)
- Tier 3: Page type with section references
- Content model enables reuse across pages"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 5: Content types created" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-6-GLOBAL-CONTENT.md](PHASE-6-GLOBAL-CONTENT.md)
