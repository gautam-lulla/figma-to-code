# Nested Components Reference

This document explains how to handle Figma components that contain other components, and how to model their content in the CMS.

---

## What Are Nested Components?

In Figma, components often contain other components. For example:

```
navigation-desktop
├── logo (NoMad-Wynwood__Mark-01)
├── menu-link (×5)
│   └── (label, href, isActive)
└── button (CTA)
    └── (text, url, variant)

hero-option-1
├── background-image
├── headline
├── subtitle
└── button (CTA)
    └── (text, url)

event-card
├── image
├── title
├── date
├── description
└── button (Learn More)
    └── (text, url)
```

The question is: how do we model this in the CMS?

---

## The Decision: Composition vs Embedding

### Composition (Separate CMS Entries)

The child component has its **own content entry** that the parent references.

**Use when:**
- Child is reused across multiple parents
- Child has 3+ fields worth managing
- Child needs independent editing lifecycle
- Different editors manage different content

**Example:** FAQ items used on FAQ page AND About page

```typescript
// faq-item content type (separate entries)
await createOrUpdateContentType('faq-item', 'FAQ Item', [
  { slug: 'question', name: 'Question', type: 'text', required: true },
  { slug: 'answer', name: 'Answer', type: 'richText', required: true },
  { slug: 'category', name: 'Category', type: 'text' },
]);

// page references FAQ items
await createOrUpdateContentType('page', 'Page', [
  { slug: 'faqItems', name: 'FAQ Items', type: 'json' },
  // Value: ["faq-reservations", "faq-parking"]
]);
```

**React:**
```tsx
// Each FAQ item has its own CMS entry
{faqs.map((faq) => (
  <FaqAccordion
    key={faq.slug}
    entry={faq.slug}  // "faq-reservations"
    question={faq.question}
    answer={faq.answer}
  />
))}
```

### Embedding (JSON Field in Parent)

The child's content is **embedded as JSON** within the parent's entry.

**Use when:**
- Child is only used within this parent
- Child is simple (1-2 fields)
- Parent and child always edited together
- Separate entries would create admin clutter

**Example:** Navigation menu links

```typescript
// navigation content type (embedded links)
await createOrUpdateContentType('navigation', 'Navigation', [
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
          },
          required: ['label', 'href']
        }
      }
    }
  },
]);
```

**React:**
```tsx
// Links embedded in parent entry
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

---

## Decision Tree

```
Is the child used in multiple parents?
├── YES → Composition (separate entries)
└── NO → Continue...

Does the child have 3+ fields?
├── YES → Composition (better admin UX)
└── NO → Continue...

Are parent and child always edited together?
├── YES → Embedding (JSON field)
└── NO → Composition (independent editing)

Would separate entries create admin clutter?
├── YES → Embedding
└── NO → Composition
```

---

## Common Patterns

### Navigation (Embedding)

Menu links are navigation-specific and always edited together.

```typescript
// Content type
{ slug: 'menuLinks', type: 'json' }
// Entry data: [{ label: "Menu", href: "/menu" }, { label: "About", href: "/about" }]
```

```tsx
// Component
{menuLinks.map((link, index) => (
  <Link
    href={link.href}
    data-cms-entry="global-navigation"
    data-cms-field={`menuLinks[${index}].label`}
  >
    {link.label}
  </Link>
))}
```

### Footer (Embedding)

Footer links are footer-specific.

```typescript
// Content type
{ slug: 'column1Links', type: 'json' }
{ slug: 'socialLinks', type: 'json' }
```

### Hero with CTA (Embedding)

The button content is specific to this hero.

```typescript
// Content type
{ slug: 'headline', type: 'text' }
{ slug: 'ctaText', type: 'text' }
{ slug: 'ctaUrl', type: 'text' }
```

```tsx
// Component
<Button
  href={ctaUrl}
  data-cms-entry={entry}
  data-cms-field="ctaText"
>
  {ctaText}
</Button>
```

### FAQ Section (Composition)

FAQ items are reused across multiple pages.

```typescript
// faq-item content type (separate)
{ slug: 'question', type: 'text' }
{ slug: 'answer', type: 'richText' }
{ slug: 'category', type: 'text' }

// page references by slug
{ slug: 'faqItems', type: 'json' }  // ["faq-parking", "faq-hours"]
```

```tsx
// Component fetches and renders
{faqs.map((faq) => (
  <FaqAccordion
    entry={faq.slug}  // Each has its own entry
    question={faq.question}
    answer={faq.answer}
  />
))}
```

### Team Section (Composition)

Team members are managed independently and may appear on multiple pages.

```typescript
// team-member content type (separate)
{ slug: 'name', type: 'text' }
{ slug: 'title', type: 'text' }
{ slug: 'photo', type: 'media' }
{ slug: 'bio', type: 'richText' }
```

### Event Cards (Composition)

Events are managed independently, with their own edit lifecycle.

```typescript
// event content type (separate)
{ slug: 'title', type: 'text' }
{ slug: 'date', type: 'text' }
{ slug: 'description', type: 'richText' }
{ slug: 'image', type: 'media' }
{ slug: 'ctaText', type: 'text' }  // Button embedded within event
{ slug: 'ctaUrl', type: 'text' }
```

### Gallery Images (Embedding)

Images are gallery-specific.

```typescript
// gallery content type
{
  slug: 'images',
  type: 'json',
  validation: {
    schema: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          src: { type: 'string' },
          alt: { type: 'string' },
          caption: { type: 'string' }
        }
      }
    }
  }
}
```

---

## CMS Data Attributes

### For Embedded Children

Use indexed paths within the parent entry:

```tsx
// Parent entry: "global-navigation"
// Field path: "menuLinks[0].label"
<span
  data-cms-entry="global-navigation"
  data-cms-field={`menuLinks[${index}].label`}
>
  {link.label}
</span>
```

### For Composed Children

Each child has its own entry:

```tsx
// Child entry: "faq-parking" (its own slug)
// Field path: "question" (direct field, not nested)
<h3
  data-cms-entry={faq.slug}
  data-cms-field="question"
>
  {faq.question}
</h3>
```

---

## Impact on React Components

### Embedded: Props from Parent

```tsx
interface HeroSectionProps {
  entry: string;
  headline: string;
  ctaText?: string;  // Button content embedded
  ctaUrl?: string;
}

function HeroSection({ entry, headline, ctaText, ctaUrl }: HeroSectionProps) {
  return (
    <section>
      <h1 data-cms-entry={entry} data-cms-field="headline">
        {headline}
      </h1>
      {ctaText && (
        <Button
          href={ctaUrl}
          data-cms-entry={entry}
          data-cms-field="ctaText"  // Part of parent entry
        >
          {ctaText}
        </Button>
      )}
    </section>
  );
}
```

### Composed: Props with Own Entry

```tsx
interface FaqSectionProps {
  entry: string;          // Parent page entry
  sectionTitle: string;
  faqs: FaqItem[];        // Already fetched child entries
}

interface FaqItem {
  slug: string;           // Child's own entry slug
  question: string;
  answer: string;
}

function FaqSection({ entry, sectionTitle, faqs }: FaqSectionProps) {
  return (
    <section>
      <h2 data-cms-entry={entry} data-cms-field="faqSectionTitle">
        {sectionTitle}
      </h2>
      {faqs.map((faq) => (
        <FaqAccordion
          key={faq.slug}
          entry={faq.slug}  // Child has ITS OWN entry
          question={faq.question}
          answer={faq.answer}
        />
      ))}
    </section>
  );
}
```

---

## Summary Table

| Component | Children | Modeling | Reason |
|-----------|----------|----------|--------|
| Navigation | menu-link, logo, button | **Embed** | Nav-specific, simple |
| Footer | link groups, social icons | **Embed** | Footer-specific |
| Hero | button (CTA) | **Embed** | Hero-specific CTA |
| Event Card | button | **Embed** | Event-specific CTA |
| FAQ Section | faq-accordion | **Compose** | FAQs reused across pages |
| Team Section | team-member | **Compose** | Members managed independently |
| Gallery | images | **Embed** | Gallery-specific images |
| Awards Section | award | **Compose** | Awards reused on multiple pages |

---

## Fetching Composed Content

When using composition, you need to resolve references in your content layer:

```typescript
// src/lib/content/index.ts

async function getPageWithResolvedContent(slug: string) {
  const page = await getPageBySlug(slug);

  // Resolve FAQ references
  if (page.data.faqItems?.length) {
    const faqSlugs = page.data.faqItems;
    const faqs = await Promise.all(
      faqSlugs.map(faqSlug => getFaqItem(faqSlug))
    );
    page.data.resolvedFaqs = faqs.filter(Boolean);
  }

  // Resolve team member references
  if (page.data.teamMembers?.length) {
    const teamSlugs = page.data.teamMembers;
    const team = await Promise.all(
      teamSlugs.map(slug => getTeamMember(slug))
    );
    page.data.resolvedTeam = team.filter(Boolean);
  }

  return page;
}
```

---

## Related Documents

- [PHASE-5-CONTENT-TYPES.md](PHASE-5-CONTENT-TYPES.md) — Content type creation with nested patterns
- [PHASE-7-COMPONENTS.md](PHASE-7-COMPONENTS.md) — Component build order and structure
- [CMS-FIELD-DEFINITIONS.md](CMS-FIELD-DEFINITIONS.md) — Field types and JSON schemas
