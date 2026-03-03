# Phase 6: Global Content Extraction & CMS Population

Extract ALL global content from Figma and populate the CMS.
No content should be hardcoded in components.

> **IMPORTANT:** Extract ONLY what exists in YOUR Figma designs.
> Do not invent content or assume fields exist.

> **CRITICAL:** All global entries MUST be created with `isGlobal: true`.
> This prevents them from being duplicated when users clone pages in the inline editor.

> **CRITICAL:** All entries MUST include `propertyId` from your configuration.
> This scopes content to the specific website/property.

## Step 0.5: Check for Existing Global Entries (Wizard Reconciliation)

**CRITICAL: The wizard may have already created global entries.** Before creating `site-settings`, `navigation`, or `footer` entries, check if they already exist. If they do, **update** them with Figma-extracted content instead of creating duplicates.

**IMPORTANT: Look up by CONTENT TYPE + PROPERTY, not by entry slug.** The wizard may use different slug conventions (e.g., `navigation` vs `global-navigation`). Querying by content type guarantees you find the existing entry regardless of its slug.

```typescript
/**
 * Find any existing global entry for a content type within this property.
 * Returns the first match — global entries are singletons per content type per property.
 */
async function findGlobalEntryByContentType(
  contentTypeSlug: string,
  propertyId: string
): Promise<{ id: string; slug: string; data: Record<string, unknown> } | null> {
  const result = await graphqlQuery(`
    query FindGlobalEntry($orgId: String!, $filters: ContentEntryFilters) {
      contentEntries(organizationId: $orgId, filters: $filters, pagination: { limit: 1 }) {
        items { id slug data }
      }
    }
  `, {
    orgId: config.cmsOrganizationId,
    filters: {
      contentTypeSlug,
      propertyId,
      isGlobal: true,
    },
  });

  return result.data?.contentEntries?.items?.[0] ?? null;
}

/**
 * Upsert a global entry: find by content type + property, then update or create.
 * This handles wizard-created entries regardless of their slug naming convention.
 */
async function upsertGlobalEntry(
  contentTypeSlug: string,
  entrySlug: string,
  data: Record<string, unknown>
) {
  // Step 1: Find ANY existing global entry for this content type + property
  const existing = await findGlobalEntryByContentType(contentTypeSlug, config.cmsPropertyId);

  if (existing) {
    // Step 2a: Update — merge Figma content into existing entry
    // (wizard slug may differ from F2C slug — that's fine, keep the existing slug)
    console.log(`Found existing "${contentTypeSlug}" entry (slug: "${existing.slug}", ID: ${existing.id}) — updating with Figma content`);

    await updateContentEntry(existing.id, {
      data: { ...existing.data, ...data },  // Merge: Figma values overwrite, wizard values kept if not in Figma
    });

    return existing.id;
  } else {
    // Step 2b: Create from scratch
    console.log(`No existing "${contentTypeSlug}" entry found — creating as "${entrySlug}"`);

    const result = await createOrUpdateEntry(contentTypeSlug, entrySlug, data, {
      isGlobal: true,
      propertyId: config.cmsPropertyId,
    });

    return result.id;
  }
}
```

**Why this matters:** The wizard creates global entries with its own slug convention (e.g., `navigation`, `footer`, `site-settings`). F2C may use different slugs (e.g., `global-navigation`). By querying by content type instead of slug, the upsert always finds the correct entry and updates it in place — no duplicates, no slug coupling between the two systems.

---

## Step 1: Extract Navigation Content

Use Navigation Figma frames to extract ONLY what appears:

```typescript
// PATTERN - extract fields that exist in YOUR navigation
const navigationContent = {
  menuLinks: [
    // { label: '[exact text from Figma]', href: '[destination]' },
  ],
  // Include ONLY if present in your Figma:
  // ctaButtonText: '[button text]',
  // ctaButtonUrl: '[button link]',
};

// Use upsert — finds wizard's existing entry by content type, updates it in place
await upsertGlobalEntry('navigation', 'navigation', navigationContent);
```

## Step 2: Extract Footer Content

Use Footer Figma frames:

```typescript
// PATTERN - extract fields that exist in YOUR footer
const footerContent = {
  // footerLinks: [...],  // only if present
  // copyrightText: '...', // only if present
  // newsletterTitle: '...', // only if newsletter exists
};

// Use upsert — finds wizard's existing entry by content type, updates it in place
await upsertGlobalEntry('footer', 'footer', footerContent);
```

## Step 3: Extract Site Settings

```typescript
// PATTERN - extract what appears across your designs
const siteSettings = {
  logoUrl: '[CDN URL for logo]',
  logoAlt: '[Brand name]',
  // address: '[only if appears]',
  // phone: '[only if appears]',
};

// Use upsert — finds wizard's existing entry by content type, updates it in place
await upsertGlobalEntry('site-settings', 'site-settings', siteSettings);
```

## Step 4: Additional Global Content

Create entries ONLY for content that exists in Figma:
- Social media section (if present)
- Testimonials (if global)
- Partner logos (if present)

> **Remember:** All shared/global content entries should use `isGlobal: true`.

## Step 5: Error Page Content

**Create ONLY if 404 page exists in Figma:**

```typescript
// const errorPageContent = {
//   title: '[Title from Figma]',
//   message: '[Message from Figma]',
//   buttonText: '[Button text from Figma]',
// };
// await createOrUpdateEntry('error-page', '404', errorPageContent);
```

## Step 6: Verify All Global Content

```typescript
const nav = await getNavigation();
const footer = await getFooterContent();
const settings = await getSiteSettings();

console.log('✓ Navigation:', nav.menuLinks.length, 'links');
console.log('✓ Footer created');
console.log('✓ Site Settings:', settings.logoUrl);
```

## Documentation

Create `/docs/cms-entries.md` listing all created entries:

```markdown
# CMS Entries

## Global Entries
| Content Type | Entry Slug | Purpose |
|--------------|------------|---------|
| site-settings | site-settings | Logo, contact info |
| navigation | navigation | Menu links |
| footer | footer | Footer content |

> **Note:** Slugs shown above are defaults. If the wizard already created entries
> with different slugs, the upsert uses the existing slug — no rename needed.
```

## Git Commit

```bash
git commit -m "feat: Global content extracted and populated"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 6: Global content populated" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-7-COMPONENTS.md](PHASE-7-COMPONENTS.md)
