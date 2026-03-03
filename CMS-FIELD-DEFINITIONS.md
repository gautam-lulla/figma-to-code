# CMS Field Definitions Guide

This document defines the rules and best practices for creating CMS content types and field definitions. Following these rules ensures the CMS admin UI is user-friendly (individual form fields instead of JSON blobs) and data is properly structured.

---

## Core Principles

### 1. Content Types Before Entries

**Always create content types with proper field definitions BEFORE creating content entries.**

```
Order of operations:
1. Create content type (e.g., "page-content")
2. Create field definitions for that type
3. Create content entries

If you create entries before field definitions, the CMS admin will show a single JSON blob editor instead of individual form fields.
```

### 2. Flat Field Structure

**Field slugs must be flat (single level) — no dot notation or nesting.**

```typescript
// ✅ CORRECT: Flat field slugs
{ slug: 'heroTitle', name: 'Hero Title', type: 'text' }
{ slug: 'heroVideoUrl', name: 'Hero Video', type: 'media' }
{ slug: 'aboutBody', name: 'About Body', type: 'richText' }

// ❌ WRONG: Nested/dot notation slugs (won't work)
{ slug: 'hero.title', name: 'Hero Title', type: 'text' }
{ slug: 'hero.videoUrl', name: 'Hero Video', type: 'media' }
```

### 3. Naming Convention

**Use camelCase for field slugs with section prefix:**

```
Pattern: {sectionName}{FieldName}

Examples:
- heroTitle, heroSubtitle, heroVideoUrl, heroPosterUrl
- aboutHeading, aboutBody, aboutCtaText, aboutCtaUrl
- contactEmail, contactPhone, contactAddress
- footerCopyright, footerLinks
```

---

## Field Types Reference

| Type | CMS Admin UI | Use For |
|------|--------------|---------|
| `text` | Single-line text input | Titles, labels, short text |
| `richText` | Rich text editor (Tiptap) | Paragraphs, formatted content |
| `number` | Number input | Counts, sort orders, prices |
| `boolean` | Toggle switch | Flags, visibility toggles |
| `media` | Media picker (file upload) | Images, videos, documents |
| `json` | JSON editor | Complex arrays, nested objects |
| `reference` | Content entry selector | Links to other entries |

### When to Use Each Type

**`text`** - Short strings (< 255 chars)
```typescript
{ slug: 'heroTitle', name: 'Hero Title', type: 'text', required: true }
{ slug: 'buttonLabel', name: 'Button Label', type: 'text' }
```

**`richText`** - Formatted paragraphs, HTML content
```typescript
{ slug: 'aboutBody', name: 'About Body', type: 'richText' }
{ slug: 'faqAnswer', name: 'Answer', type: 'richText' }
```

**`media`** - Images, videos, files
```typescript
{ slug: 'heroVideoUrl', name: 'Hero Video', type: 'media' }
{ slug: 'teamMemberImage', name: 'Photo', type: 'media' }
```

**`json`** - Arrays or complex nested objects
```typescript
// Use for arrays of items
{ slug: 'navigationLinks', name: 'Navigation Links', type: 'json' }
{ slug: 'socialMediaLinks', name: 'Social Links', type: 'json' }

// Use for complex nested structures
{ slug: 'heroSection', name: 'Hero Section', type: 'json' }
```

**`reference`** - Links to other content entries
```typescript
// Link to a single entry
{ slug: 'authorId', name: 'Author', type: 'reference' }

// For arrays of references, consider separate content types
```

### JSON Schema Validation

JSON fields support optional schema validation using JSON Schema (draft-07). This allows you to enforce structure on complex data like menu items, links, and hours of operation.

**See [JSON-FIELD-SCHEMA.md](./JSON-FIELD-SCHEMA.md) for:**
- Menu category items schema
- Common schema patterns (links, sections, hours)
- TypeScript interface patterns
- How to apply schemas to content types
- Error message formatting

**Example: Adding schema validation to a menu items field:**
```typescript
{
  slug: 'menuItems',
  name: 'Menu Items',
  type: 'json',
  validation: {
    schema: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          name: { type: 'string', minLength: 1 },
          price: { type: 'string', pattern: '^\\$?\\d+(\\.\\d{2})?$' }
        },
        required: ['name']
      }
    }
  }
}
```

---

## Handling Arrays and Collections

### Option A: JSON Field (Simpler)

Use JSON fields for small, static arrays:

```typescript
// Field definition
{ slug: 'whyBeckonsCards', name: 'Why Beckons Cards', type: 'json' }

// Entry data
{
  whyBeckonsCards: [
    { iconName: 'mountain', title: 'Adventure', description: '...' },
    { iconName: 'spa', title: 'Wellness', description: '...' },
  ]
}
```

**Pros:** Simple, self-contained
**Cons:** No type-ahead, no individual entry editing

### Option B: Reference Fields (Better UX)

For better admin UX, create a separate content type and use references:

```typescript
// 1. Create the item content type
await createContentType('feature-card', 'Feature Card', [
  { slug: 'iconName', name: 'Icon', type: 'text', required: true },
  { slug: 'title', name: 'Title', type: 'text', required: true },
  { slug: 'description', name: 'Description', type: 'richText' },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);

// 2. Create individual entries
await createEntry('feature-card', 'adventure', { iconName: 'mountain', title: 'Adventure', sortOrder: 1 });
await createEntry('feature-card', 'wellness', { iconName: 'spa', title: 'Wellness', sortOrder: 2 });

// 3. Reference from page (option: JSON array of slugs)
await createEntry('page-content', 'home', {
  // ... other fields
  featureCardSlugs: ['adventure', 'wellness', 'cuisine'],
});

// 4. Frontend fetches entries by slug
const cards = await getContentEntriesBySlug('feature-card', featureCardSlugs);
```

**Pros:** Individual entry editing, type-ahead, reusable across pages
**Cons:** More complex setup, requires joins on frontend

---

## Content Type Patterns

### Page Content Type

For pages with multiple sections, use flat field names:

```typescript
await createContentType('page-content', 'Page Content', [
  // Meta fields
  { slug: 'metaTitle', name: 'Meta Title', type: 'text' },
  { slug: 'metaDescription', name: 'Meta Description', type: 'text' },

  // Hero section
  { slug: 'heroTitle', name: 'Hero Title', type: 'text' },
  { slug: 'heroSubtitle', name: 'Hero Subtitle', type: 'text' },
  { slug: 'heroVideoUrl', name: 'Hero Video', type: 'media' },
  { slug: 'heroPosterUrl', name: 'Hero Poster', type: 'media' },

  // About section
  { slug: 'aboutHeading', name: 'About Heading', type: 'text' },
  { slug: 'aboutBody', name: 'About Body', type: 'richText' },
  { slug: 'aboutCtaText', name: 'About CTA Text', type: 'text' },
  { slug: 'aboutCtaUrl', name: 'About CTA URL', type: 'text' },

  // Arrays stay as JSON
  { slug: 'featureCards', name: 'Feature Cards', type: 'json' },
]);
```

### Collection Content Types

For repeatable items (FAQ, team members, menu items):

```typescript
// FAQ Item
await createContentType('faq-item', 'FAQ Item', [
  { slug: 'question', name: 'Question', type: 'text', required: true },
  { slug: 'answer', name: 'Answer', type: 'richText', required: true },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);

// Team Member
await createContentType('team-member', 'Team Member', [
  { slug: 'name', name: 'Name', type: 'text', required: true },
  { slug: 'title', name: 'Job Title', type: 'text' },
  { slug: 'bio', name: 'Biography', type: 'richText' },
  { slug: 'photoUrl', name: 'Photo', type: 'media' },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);

// Menu Item
await createContentType('menu-item', 'Menu Item', [
  { slug: 'name', name: 'Name', type: 'text', required: true },
  { slug: 'price', name: 'Price', type: 'text' },
  { slug: 'description', name: 'Description', type: 'text' },
  { slug: 'sectionSlug', name: 'Section', type: 'text' },
  { slug: 'sortOrder', name: 'Sort Order', type: 'number' },
]);
```

### Global Content Types

For site-wide content (settings, navigation, footer):

```typescript
// Site Settings
await createContentType('site-settings', 'Site Settings', [
  { slug: 'siteName', name: 'Site Name', type: 'text' },
  { slug: 'siteDescription', name: 'Site Description', type: 'text' },
  { slug: 'logoUrl', name: 'Logo', type: 'media' },
  { slug: 'logoAlt', name: 'Logo Alt Text', type: 'text' },
  { slug: 'primaryColor', name: 'Primary Color', type: 'text' },
]);

// Site Navigation
await createContentType('site-navigation', 'Site Navigation', [
  { slug: 'menuLinks', name: 'Menu Links', type: 'json' },
  { slug: 'ctaText', name: 'CTA Button Text', type: 'text' },
  { slug: 'ctaUrl', name: 'CTA Button URL', type: 'text' },
]);

// Site Footer
await createContentType('site-footer', 'Site Footer', [
  { slug: 'copyrightText', name: 'Copyright Text', type: 'text' },
  { slug: 'footerLinks', name: 'Footer Links', type: 'json' },
  { slug: 'socialLinks', name: 'Social Links', type: 'json' },
]);
```

---

## Entry Data Structure

### Flat Data for Flat Fields

When you have individual field definitions, entry data should match:

```typescript
// Field definitions use flat slugs
// Entry data uses the SAME flat structure

await createEntry('page-content', 'home', {
  // Meta
  metaTitle: 'Welcome to Our Site',
  metaDescription: 'Discover amazing experiences...',

  // Hero section (flat, not nested)
  heroTitle: 'Welcome',
  heroSubtitle: 'Discover the extraordinary',
  heroVideoUrl: 'https://cdn.example.com/hero.mp4',
  heroPosterUrl: 'https://cdn.example.com/hero-poster.jpg',

  // About section
  aboutHeading: 'About Us',
  aboutBody: '<p>We are a team of...</p>',
  aboutCtaText: 'Learn More',
  aboutCtaUrl: '/about',

  // Arrays as JSON
  featureCards: [
    { iconName: 'star', title: 'Quality', description: '...' },
    { iconName: 'clock', title: 'Fast', description: '...' },
  ],
});
```

---

## Frontend Transformation

### Why Transform?

Components often expect nested data structures for organization:

```typescript
// Component expects:
interface HeroProps {
  hero: {
    title: string;
    subtitle: string;
    videoUrl: string;
  };
}
```

But CMS stores flat:

```typescript
// CMS returns:
{
  heroTitle: 'Welcome',
  heroSubtitle: 'Discover',
  heroVideoUrl: 'https://...',
}
```

### Transformation Layer Pattern

Create a content layer that transforms flat CMS data to nested component props:

```typescript
// src/lib/content/index.ts

interface FlatPageData {
  heroTitle: string;
  heroSubtitle: string;
  heroVideoUrl: string;
  aboutHeading: string;
  aboutBody: string;
  // ... all flat fields
}

interface NestedPageData {
  hero: {
    title: string;
    subtitle: string;
    videoUrl: string;
  };
  about: {
    heading: string;
    body: string;
  };
}

function transformPageData(flat: FlatPageData): NestedPageData {
  return {
    hero: {
      title: flat.heroTitle,
      subtitle: flat.heroSubtitle,
      videoUrl: flat.heroVideoUrl,
    },
    about: {
      heading: flat.aboutHeading,
      body: flat.aboutBody,
    },
  };
}

export async function getPageContent(slug: string): Promise<NestedPageData> {
  const flatData = await fetchFromCMS(slug);
  return transformPageData(flatData);
}
```

---

## Migration Checklist

When converting existing JSON blobs to proper field definitions:

1. **Analyze Current Structure**
   - List all fields in the JSON blob
   - Identify nested objects vs arrays
   - Note required fields

2. **Create Field Definitions**
   - Flatten nested objects to `sectionFieldName` format
   - Keep arrays as JSON fields (or create reference types)
   - Set appropriate field types

3. **Transform Entry Data**
   - Convert nested to flat structure
   - Preserve all values
   - Test in CMS admin

4. **Update Frontend**
   - Add transformation layer if components expect nested data
   - Or update components to use flat field names
   - Test rendering

5. **Verify**
   - CMS admin shows individual form fields
   - Editing and saving works
   - Frontend renders correctly

---

## Version History

**v1.0.0** (January 2026)
- Initial field definitions guide
- Flat field structure rules
- Collection patterns (JSON vs references)
- Frontend transformation patterns
