# Phase 8: Page Content Extraction & CMS Population

> **Note:** Page entries should NOT have `isGlobal: true` (default is `false`).
> Pages are the entries that users will clone in the inline editor.
> Global entries (navigation, footer, settings) are created in Phase 6 with `isGlobal: true`.

> **CRITICAL:** All entries MUST include `propertyId` from your configuration.
> This scopes content to the specific website/property.

**FOR EACH PAGE (in order):**

## Step A: Identify Page Template

Before extracting content, determine which template this page uses based on its Figma layout:

| Figma Pattern | Template Value |
|---------------|----------------|
| Hero + text sections | `standard` |
| Hero + image grid | `gallery` |
| Hero + contact/inquiry form | `form` |
| Hero + list of items (events, blog) | `listing` |

**The template field is REQUIRED for dynamic routing to work.**

## Step B: Extract ALL Text from Figma

- Use Figma MCP to get every text element
- Copy EXACTLY as shown — no rewording

## Step C: Map Images to CDN URLs

Use `/docs/image-manifest.json` to get CDN URLs for each image.

## Step D: Create CMS Entry

> **CRITICAL: Use FLAT data structure!**
> All sections must be at the TOP LEVEL of the data object.
> Do NOT nest content inside a `data` property - this breaks inline editing.

✅ **Correct (flat):** `entry.data.hero.title`
❌ **Wrong (nested):** `entry.data.data.hero.title`

> **IMPORTANT:** The example shows the PATTERN.
> Page names, sections, and fields depend on YOUR Figma.
> Do NOT assume specific sections exist.

```typescript
// Use actual page slug from YOUR Figma
// REQUIRED: propertyId scopes this entry to the current website
const pageEntry = await createOrUpdateEntry('page', '[your-page-slug]', {
  propertyId: config.cmsPropertyId,  // REQUIRED: from /project-config/config.md
  // REQUIRED: Page identity
  title: '[Page Title]',
  slug: '[url-slug]',  // e.g., 'about', 'contact', 'getting-here'

  // REQUIRED: Template for dynamic routing
  template: '[template-name]',  // 'standard', 'gallery', 'form', or 'listing'

  // NOTE: SEO fields (metaTitle, metaDescription, ogImage) are NOT stored
  // here. They are stored in PageSeoConfig via the SEO module.
  // See "Step E: Populate SEO via SEO Module" below.

  // Hero section (if present)
  hero: {
    title: '[Exact heading from Figma]',
    subtitle: '[Subheading if present]',
    imageSrc: '[CDN URL]',
  },

  // Content sections at top level
  sections: [
    {
      heading: '[Section heading]',
      paragraph: '[Section content]',
      imageSrc: '[CDN URL if present]',
      variant: 'left' | 'right' | 'centered',  // Based on Figma layout
    },
    // ... more sections
  ],

  // Optional global section toggles
  showInstagram: true,  // If Instagram feed appears on this page
  showFaq: false,

  // Add sections ONLY for what exists in Figma
});
```

## Step E: Populate SEO via SEO Module

**CRITICAL: SEO data lives exclusively in the `PageSeoConfig` entity (SEO module), NOT as fields on the page content type.** This enables the admin SEO editor, SEO scoring, and the renderer toolkit to access SEO data through a unified API.

For each page entry created, call `savePageSeo` to populate SEO metadata:

```graphql
mutation SavePageSeo($input: SavePageSeoInput!) {
  savePageSeo(input: $input) {
    id
    title
    description
    ogTitle
    ogDescription
    ogImage
    seoScore
  }
}
```

```typescript
// After creating each page entry, populate its SEO config
await savePageSeo({
  pageEntryId: pageEntry.id,
  organizationId: config.cmsOrganizationId,

  // SEO title and description from Figma content
  title: '[Brand Name] | [Page Title]',
  description: '[Description extracted from Figma page content]',

  // Open Graph — use hero image as OG image
  ogTitle: '[Page Title]',
  ogDescription: '[Description]',
  ogImage: '[CDN URL of hero or key page image]',
  ogImageAlt: '[Alt text for OG image]',
  ogType: 'website',  // or 'article', 'restaurant', 'hotel'

  // Twitter Card
  twitterCard: 'summary_large_image',

  // Indexing
  noIndex: false,
  noFollow: false,
});
```

### Bulk SEO Population

For efficiency, use `bulkSavePageSeo` to populate all pages at once:

```typescript
// After creating all page entries, bulk-populate SEO
const seoItems = pages.map(page => ({
  pageEntryId: page.id,
  title: `${config.brandName} | ${page.data.title}`,
  description: extractDescription(page),
  ogImage: page.data.hero?.imageSrc || null,
  ogType: 'website',
  twitterCard: 'summary_large_image',
}));

await bulkSavePageSeo({
  organizationId: config.cmsOrganizationId,
  items: seoItems,
});
```

### Handling Existing PageSeoConfig

If the wizard already created `PageSeoConfig` records (with auto-generated title/description), `savePageSeo` **updates** them rather than creating duplicates. The mutation uses `pageEntryId` as the unique key.

### What NOT to Do

```typescript
// ❌ WRONG — Do NOT store SEO as content entry fields
await createOrUpdateEntry('page', 'about', {
  metaTitle: 'About Us',           // ← NO! Lives in PageSeoConfig
  metaDescription: '...',          // ← NO! Lives in PageSeoConfig
  ogImage: 'https://cdn/img.jpg',  // ← NO! Lives in PageSeoConfig
});

// ✅ CORRECT — Store SEO in the SEO module
await savePageSeo({
  pageEntryId: aboutPageEntry.id,
  title: 'About Us',
  description: '...',
  ogImage: 'https://cdn/img.jpg',
});
```

## Repeat for ALL Pages

For each page in your Figma file:
1. Extract text from Figma MCP
2. Map images to CDN URLs
3. Create CMS entry
4. Verify entry was created

## Create Collection Entries (If Applicable)

Create entries for collections ONLY if they exist in Figma:

```typescript
// FAQ items - include propertyId for all entries
// await createOrUpdateEntry('faq-item', 'faq-1', {
//   propertyId: config.cmsPropertyId,
//   question: '...', answer: '...'
// });

// Team members - include propertyId for all entries
// await createOrUpdateEntry('team-member', 'john-doe', {
//   propertyId: config.cmsPropertyId,
//   name: '...', title: '...'
// });
```

## Documentation

Update `/docs/cms-entries.md` with all page entries:

```markdown
## Page Entries
| Content Type | Entry Slug | Page |
|--------------|------------|------|
| page-content | home | Homepage |
| page-content | about | About page |
```

## Git Commit

```bash
git commit -m "feat: All page content populated in CMS"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 8: Page content populated" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-9-PAGES.md](PHASE-9-PAGES.md)
