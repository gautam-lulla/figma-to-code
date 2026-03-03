# Figma to Code - Learnings

This file contains accumulated insights, patterns, and lessons learned.
It is automatically updated by the `/learn` skill and loaded when this skill is invoked.

---

## Patterns

### Website Content Layer - Image Object Handling (Jan 2026)

When building Next.js websites that fetch content from Halo CMS, the content layer (`src/lib/content/index.ts`) MUST handle image objects from the inline editor.

**The Problem:**
- Inline editor saves images as objects: `{id, url, src, alt, filename}`
- Frontends typically expect plain URL strings: `"https://cdn.example.com/image.jpg"`
- Without proper handling, components receive objects instead of strings and images don't display

**The Solution:**
Add an `isImageObject()` helper and update `transformImageUrls()` to extract URLs:

```typescript
/**
 * Check if an object is an image object from the CMS inline editor.
 * Image objects have url/src and typically id, alt, filename.
 */
function isImageObject(obj: unknown): obj is { url?: string; src?: string } {
  if (typeof obj !== 'object' || obj === null) return false;
  const o = obj as Record<string, unknown>;
  // Must have url or src, and should have typical image fields
  return (typeof o.url === 'string' || typeof o.src === 'string') &&
    (o.id !== undefined || o.filename !== undefined || o.alt !== undefined);
}

/**
 * Transform image data for frontend consumption.
 * - Extracts URL strings from image objects saved by the inline editor
 * - Transforms /images/ paths to CDN URLs (if applicable)
 * - Recursively processes objects and arrays
 */
function transformImageUrls<T>(data: T): T {
  if (data === null || data === undefined) return data;

  if (typeof data === 'string') {
    // Transform /images/... paths to CDN URLs if needed
    if (data.startsWith('/images/')) {
      return data.replace('/images/', `${CDN_BASE_URL}/`) as T;
    }
    return data;
  }

  if (Array.isArray(data)) {
    return data.map(item => transformImageUrls(item)) as T;
  }

  if (typeof data === 'object') {
    // Check if this is an image object - extract URL directly
    if (isImageObject(data)) {
      const imageUrl = (data as { url?: string; src?: string }).url ||
                       (data as { url?: string; src?: string }).src || '';
      return imageUrl as T;
    }

    const result: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(data as Record<string, unknown>)) {
      result[key] = transformImageUrls(value);
    }
    return result as T;
  }

  return data;
}
```

**Usage in getPageContent():**
```typescript
export async function getPageContent<T>(slug: string): Promise<T> {
  const { data } = await client.query(...);
  return transformImageUrls(data?.contentEntryBySlug?.data) as T;
}
```

**Key Points:**
- Always apply `transformImageUrls()` to CMS data before returning to components
- The function is recursive - it handles nested objects and arrays
- Image objects are detected by presence of `url`/`src` AND typical fields like `id`, `filename`, or `alt`
- This pattern is essential for inline editor compatibility

---

## Gotchas

<!-- Common mistakes and edge cases to avoid -->

### Inline Editor Image Field - String URL Spread Bug (Fixed Jan 2026)
When the inline editor stores image URLs as strings (common for nested fields like `whyBeckons.cards[0].imageUrl`), selecting a new image from the media picker could corrupt the data. The bug: `JSON.parse()` returns the raw string, and spreading it (`{...stringUrl}`) creates an object with numeric keys like `{"0":"h","1":"t","2":"t",...}`. The save appears to succeed but the data is corrupted. **Fix applied in `public/inline-editor.js`** - always check `typeof currentValue === 'object'` before spreading.

### Website Content Layer Must Handle Image Objects (Jan 2026)
The inline editor saves images as objects `{id, url, src, alt, filename}`, but frontends typically expect plain URL strings. Without a `transformImageUrls()` function that extracts URLs from image objects, images won't display after inline editing. See the **Patterns** section above for the complete solution.

### Nested Data Structure Breaks Inline Editing (Jan 2026)
Some early CMS implementations stored content with an extra `data` wrapper: `{ data: { data: { hero: {...} } } }`. This breaks inline editing because the editor looks for paths like `data.hero.title`, not `data.data.hero.title`. The backend `inline-editor.service.ts` now has `normalizeFieldPath()` to auto-detect and handle both flat and nested structures, but new sites should always use flat structures: `{ data: { hero: {...} } }`.

---

## Preferences

<!-- Client/team preferences and conventions -->

---

## Architecture

### Hybrid Routing Model — Catch-All + Dedicated Escape Hatches (Jan 2026)

After comparing catch-all routing (`[...slug]/page.tsx`) vs dedicated routes across SEO/GEO, performance, and security, the recommended architecture is a **hybrid model**:

- **Catch-all as default** (95% of pages) — CMS-driven, editorial velocity, consistent validation
- **Dedicated routes as escape hatches** — for pages needing third-party widgets, unique CSP, auth-gated content, or special caching

**Why not pure catch-all?**
- Performance: every request resolves slug → content type → entry → layout (2-3 sequential CMS queries)
- Security: one bug in the catch-all exposes all pages; dedicated routes provide isolation
- Caching: all pages share one handler, making per-route caching strategies harder

**Why not pure dedicated routes?**
- Editorial velocity: new pages require developer + deploy cycle
- SEO maintenance: sitemaps, metadata, canonical URLs need manual management per route
- Consistency: security validation (slug checks, output sanitization) must be duplicated per route

**The hybrid model** gives you catch-all's editorial velocity with dedicated routes' isolation where it matters. Every site generated by f2c starts with catch-all + `sitemap.ts` + `robots.ts` + slug validation. Dedicated routes are added per-site only when needed.

See `SKILL.md` > SITE FRAMEWORK ARCHITECTURE for the full specification.

---

## Performance

<!-- Optimization insights -->

---

## Styling

<!-- Design and CSS related learnings -->

---

## CMS

### Media Storage - Always Upload Through the CMS (Jan 2026)

**The Problem:**
Using placeholder local paths like `/images/hero.jpg` in CMS entries requires:
- Committing images to the git repo (bloats repository)
- Images aren't CMS-managed (can't update via admin)
- Images aren't on CDN (slower delivery)
- Inline editor image updates don't persist to the correct storage

**The Solution:**
All images MUST be uploaded through the CMS `uploadMedia` mutation. Never upload directly to R2 or any storage bucket — no direct bucket credentials are needed. The CMS handles storage internally and returns the CDN URL.

**Workflow:**
1. **Export images from Figma** using `mcp__figma__get_screenshot` or manual export
2. **Upload through the CMS** via `uploadMedia` mutation:
   ```typescript
   const response = await fetch(API_URL, {
     method: 'POST',
     headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
     body: JSON.stringify({
       query: `mutation UploadMedia($input: UploadMediaInput!) {
         uploadMedia(input: $input) { id filename variants { original } }
       }`,
       variables: {
         input: {
           organizationId: ORG_ID,
           propertyId: PROPERTY_ID,
           filename: 'hero.jpg',
           mimeType: 'image/jpeg',
           base64Data: base64EncodedImage,
           alt: 'Hero image description'
         }
       }
     })
   });
   ```
3. **Get CDN URL from CMS response**: `uploadMedia` returns the full CDN URL — use it directly
4. **Store CDN URL in CMS entries** instead of `/images/...` paths

**Configure Next.js to allow the CDN domain:**
```typescript
// next.config.ts
images: {
  remotePatterns: [
    { protocol: "https", hostname: "pub-xxx.r2.dev", pathname: "/**" },
  ],
},
```

**Key Points:**
- Always use `uploadMedia` mutation — no direct R2/bucket access, no bucket credentials
- Always include `propertyId` to scope media to the correct website
- The CMS returns `variants.original` with the full CDN URL — use it as-is
- Name upload scripts clearly: `scripts/upload-media.ts` (not `upload-to-r2.ts`)

### Property ID Can Be Updated (Jan 2026)

**Update:** The API now supports updating `propertyId` on existing content entries.

**Key Insight:**
- `CreateContentEntryInput` accepts `propertyId` at creation time
- `UpdateContentEntryInput` NOW accepts `propertyId` (added Jan 2026)
- Entries can be migrated between properties or from org-level to property-level

**Best Practice:**
- Always set `propertyId` at creation time when possible
- Phase 1 creates/validates the Property BEFORE any content entry creation
- All entry creation in Phases 6 and 8 includes `propertyId` from configuration
- If entries were created without `propertyId`, they can be migrated using `updateContentEntry`

**Migration Example:**
```graphql
mutation UpdateContentEntry($id: ID!, $input: UpdateContentEntryInput!) {
  updateContentEntry(id: $id, input: $input) { id slug }
}
# Variables: { "id": "entry-id", "input": { "propertyId": "property-id" } }
```

---

---

## Tooling

<!-- Build, dev, and tooling insights -->

---

## Debug

<!-- Debugging tips and solutions -->

---
