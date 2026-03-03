# Phase 7: Component Generation

**CRITICAL:** Every component must:
1. Accept ALL content via props — ZERO hardcoded strings
2. Include `data-cms-entry` and `data-cms-field` on ALL editable elements

> **IMPORTANT:** Build components from the Figma Component Library first.
> Then use those components in pages. DO NOT rebuild components that exist.

---

## Step 0: Read Phase 1 Analysis (REQUIRED)

**Before building ANY component, read these files from Phase 1:**

```bash
# These files determine what and how you build
/project-config/components.md          # Component inventory with node IDs
/project-config/component-coverage.md  # Build order and gaps
/project-config/figma-structure.md     # Figma page node IDs
```

### What You'll Find

**`/project-config/components.md`** tells you:
- All components defined in Figma with node IDs
- Variant information (Default, Hover, Expanded, etc.)
- Nested component relationships (Navigation → MenuLink → Button)
- Which pages use each component

**`/project-config/component-coverage.md`** tells you:
- Coverage percentage (defined vs used)
- Missing components (need to create as exceptions)
- **Build order** (Level 1 → 2 → 3 → 4)

---

## Step 1: Build Global Component Library

**Build components in dependency order as specified in `component-coverage.md`.**

### Level 1: Atomic Components (No Dependencies)

These components have no nested children. Build them first.

```
src/components/ui/
├── Button.tsx        # button variants
├── Link.tsx          # link variants
├── Input.tsx         # form input variants
├── Dropdown.tsx      # dropdown variants
├── Radio.tsx         # radio button
├── Checkbox.tsx      # checkbox
└── index.ts          # barrel export
```

**Example: Button Component**

```typescript
// src/components/ui/Button.tsx
import Link from 'next/link';
import { cn } from '@/lib/utils';

interface ButtonProps {
  children: React.ReactNode;
  href?: string;
  onClick?: () => void;
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'sm' | 'md' | 'lg';
  className?: string;
  // CMS data attributes passed through
  'data-cms-entry'?: string;
  'data-cms-field'?: string;
}

export function Button({
  children,
  href,
  onClick,
  variant = 'primary',
  size = 'md',
  className,
  ...cmsProps  // Spread CMS data attributes
}: ButtonProps) {
  const baseStyles = 'inline-flex items-center justify-center font-medium transition-colors';
  const variantStyles = {
    primary: 'bg-primary text-white hover:bg-primary-dark',
    secondary: 'bg-secondary text-primary hover:bg-secondary-dark',
    outline: 'border-2 border-primary text-primary hover:bg-primary hover:text-white',
  };
  const sizeStyles = {
    sm: 'px-4 py-2 text-sm',
    md: 'px-6 py-3 text-base',
    lg: 'px-8 py-4 text-lg',
  };

  const styles = cn(baseStyles, variantStyles[variant], sizeStyles[size], className);

  if (href) {
    return (
      <Link href={href} className={styles} {...cmsProps}>
        {children}
      </Link>
    );
  }

  return (
    <button onClick={onClick} className={styles} {...cmsProps}>
      {children}
    </button>
  );
}
```

### Level 2: Molecule Components (Use Level 1)

These components use Level 1 atomic components inside them.

```
src/components/molecules/
├── MenuLink.tsx      # navigation link (may use Link styling)
├── FaqAccordion.tsx  # accordion item (standalone)
├── EventCard.tsx     # event card (uses Button)
├── AwardCard.tsx     # award display
└── index.ts
```

**Example: EventCard (uses Button)**

```typescript
// src/components/molecules/EventCard.tsx
import Image from 'next/image';
import { Button } from '@/components/ui';

interface EventCardProps {
  entry: string;  // CMS entry slug for this event
  title: string;
  date: string;
  time?: string;
  description?: string;
  imageSrc?: string;
  imageAlt?: string;
  ctaText?: string;
  ctaUrl?: string;
}

export function EventCard({
  entry,
  title,
  date,
  time,
  description,
  imageSrc,
  imageAlt,
  ctaText,
  ctaUrl,
}: EventCardProps) {
  return (
    <article className="event-card">
      {imageSrc && (
        <Image
          src={imageSrc}
          alt={imageAlt || ''}
          width={400}
          height={300}
          data-cms-entry={entry}
          data-cms-field="imageSrc"
          data-cms-type="image"
        />
      )}
      <div className="event-card-content">
        <h3
          data-cms-entry={entry}
          data-cms-field="title"
        >
          {title}
        </h3>
        <time
          data-cms-entry={entry}
          data-cms-field="date"
        >
          {date}
        </time>
        {time && (
          <span
            data-cms-entry={entry}
            data-cms-field="time"
          >
            {time}
          </span>
        )}
        {description && (
          <p
            data-cms-entry={entry}
            data-cms-field="description"
          >
            {description}
          </p>
        )}
        {ctaText && ctaUrl && (
          <Button
            href={ctaUrl}
            variant="outline"
            data-cms-entry={entry}
            data-cms-field="ctaText"
          >
            {ctaText}
          </Button>
        )}
      </div>
    </article>
  );
}
```

### Level 3: Organism Components (Use Level 1 & 2)

These are complex components that compose atomic and molecule components.

```
src/components/organisms/
├── Navigation.tsx       # uses logo, MenuLink, Button
├── NavigationMobile.tsx # uses logo, Hamburger, MenuLink
├── Footer.tsx           # uses Link, Button
├── HeroSection.tsx      # hero variants, uses Button
├── Gallery.tsx          # image gallery
├── FaqSection.tsx       # uses FaqAccordion
├── InstagramFeed.tsx    # instagram embed
└── index.ts
```

**Example: Navigation (uses MenuLink, Button, Logo)**

```typescript
// src/components/organisms/Navigation.tsx
'use client';

import Image from 'next/image';
import Link from 'next/link';
import { useState } from 'react';
import { Button } from '@/components/ui';

interface NavigationProps {
  logoUrl: string;
  logoAlt: string;
  menuLinks: Array<{ label: string; href: string }>;
  ctaText?: string;
  ctaUrl?: string;
}

export function Navigation({
  logoUrl,
  logoAlt,
  menuLinks,
  ctaText,
  ctaUrl,
}: NavigationProps) {
  const [isMenuOpen, setIsMenuOpen] = useState(false);

  return (
    <nav className="navigation">
      <Link href="/">
        <Image
          src={logoUrl}
          alt={logoAlt}
          width={120}
          height={40}
          data-cms-entry="global-site-settings"
          data-cms-field="logoUrl"
          data-cms-type="image"
        />
      </Link>

      <ul className="nav-links">
        {menuLinks.map((link, index) => (
          <li key={link.href}>
            <Link
              href={link.href}
              data-cms-entry="global-navigation"
              data-cms-field={`menuLinks[${index}].label`}
            >
              {link.label}
            </Link>
          </li>
        ))}
      </ul>

      {ctaText && ctaUrl && (
        <Button
          href={ctaUrl}
          variant="primary"
          data-cms-entry="global-navigation"
          data-cms-field="ctaText"
        >
          {ctaText}
        </Button>
      )}
    </nav>
  );
}
```

### Level 4: Page Templates

Templates compose organisms into page layouts. These are used by the dynamic routing system.

```
src/components/templates/
├── StandardTemplate.tsx   # Hero + sections
├── GalleryTemplate.tsx    # Hero + image grid
├── FormTemplate.tsx       # Hero + contact form
├── ListingTemplate.tsx    # Hero + item list + pagination
└── index.ts               # Template registry
```

---

## Step 2: Handle Missing Components (Exceptions)

Components in Wireframes but NOT in the Component Library are built as **exceptions**.

Check `/project-config/component-coverage.md` for the "Missing Components" section.

**Options for missing components:**

1. **Build inline** — Create in the specific page that needs it
2. **Create as reusable** — If used 2+ times, add to component library
3. **Flag for designer** — May indicate a gap in the Figma file

```typescript
// src/components/exceptions/CustomBadge.tsx
// NOTE: This component was not in Figma Component Library
// Consider promoting to /components/ui/ if reused elsewhere

interface CustomBadgeProps {
  entry: string;
  text: string;
  variant?: 'new' | 'featured' | 'sale';
}

export function CustomBadge({ entry, text, variant = 'new' }: CustomBadgeProps) {
  return (
    <span
      className={`badge badge-${variant}`}
      data-cms-entry={entry}
      data-cms-field="badgeText"
    >
      {text}
    </span>
  );
}
```

---

## Step 3: Component Directory Structure

```
src/components/
├── ui/                    # Level 1: Atomic (Button, Input, Link)
│   ├── Button.tsx
│   ├── Input.tsx
│   ├── Link.tsx
│   └── index.ts
│
├── molecules/             # Level 2: Molecules (Cards, Accordions)
│   ├── EventCard.tsx
│   ├── FaqAccordion.tsx
│   ├── MenuLink.tsx
│   └── index.ts
│
├── organisms/             # Level 3: Organisms (Navigation, Hero, Footer)
│   ├── Navigation.tsx
│   ├── Footer.tsx
│   ├── HeroSection.tsx
│   ├── Gallery.tsx
│   └── index.ts
│
├── templates/             # Level 4: Page Templates
│   ├── StandardTemplate.tsx
│   ├── GalleryTemplate.tsx
│   ├── FormTemplate.tsx
│   ├── ListingTemplate.tsx
│   └── index.ts
│
├── exceptions/            # Components not in Figma library
│   └── CustomBadge.tsx
│
└── layout/                # App shell components
    ├── RootLayout.tsx
    └── index.ts
```

---

## Step 4: Variant Handling

Figma variants (Default, Hover, Expanded) map to React:

| Figma Variant | React Implementation |
|---------------|---------------------|
| Default/Hover | CSS `:hover` pseudo-class |
| Active | React state + CSS class |
| Expanded/Collapsed | React state toggle |
| Disabled | `disabled` prop + CSS |
| Size (sm/md/lg) | `size` prop |
| Desktop/Mobile | CSS media queries or responsive props |

**Example: Accordion with Expanded variant**

```typescript
'use client';

import { useState } from 'react';

interface FaqAccordionProps {
  entry: string;
  question: string;
  answer: string;
  defaultExpanded?: boolean;
}

export function FaqAccordion({
  entry,
  question,
  answer,
  defaultExpanded = false,
}: FaqAccordionProps) {
  const [isExpanded, setIsExpanded] = useState(defaultExpanded);

  return (
    <div className={`faq-accordion ${isExpanded ? 'expanded' : ''}`}>
      <button
        onClick={() => setIsExpanded(!isExpanded)}
        className="faq-accordion-trigger"
        aria-expanded={isExpanded}
      >
        <span
          data-cms-entry={entry}
          data-cms-field="question"
        >
          {question}
        </span>
        <span className="faq-accordion-icon" />
      </button>
      <div
        className="faq-accordion-content"
        hidden={!isExpanded}
      >
        <div
          data-cms-entry={entry}
          data-cms-field="answer"
          data-cms-type="richtext"
          dangerouslySetInnerHTML={{ __html: answer }}
        />
      </div>
    </div>
  );
}
```

---

## Step 5: Using Figma Node IDs

For each component, get exact specs from Figma using the node ID from `components.md`:

```typescript
// Get design specs for button component
const buttonSpecs = await mcp__figma__get_design_context({
  fileKey: 'bGCUY43jJhcFUafm33ncgW',
  nodeId: '355:7366'  // From /project-config/components.md
});

// Extract:
// - Exact colors (background, text, border)
// - Exact spacing (padding, margin)
// - Typography (font-size, font-weight, line-height)
// - Border radius
// - Shadows
```

**Always reference design tokens from Phase 3:**
```css
/* Use tokens, not hardcoded values */
.button-primary {
  background-color: var(--color-primary);
  padding: var(--spacing-md) var(--spacing-lg);
  font-size: var(--font-size-base);
  border-radius: var(--radius-md);
}
```

---

## Step 6: Verification Checklist

For EVERY component:

### Content Rules
- [ ] No quoted strings render to users
- [ ] Every rendered string comes from props
- [ ] No default values contain user-facing text
- [ ] Children components receive content via props (not hardcoded)

### CMS Data Attributes
- [ ] ALL text elements have `data-cms-entry`
- [ ] ALL text elements have `data-cms-field`
- [ ] Images have `data-cms-type="image"`
- [ ] Rich text has `data-cms-type="richtext"`
- [ ] Array items use indexed paths: `field[${index}].property`

### Component Structure
- [ ] Follows dependency order (Level 1 → 2 → 3)
- [ ] Reuses existing components (no duplicate Button implementations)
- [ ] Handles all Figma variants (Default, Hover, Expanded)
- [ ] Uses design tokens from Phase 3 (not hardcoded values)

### TypeScript
- [ ] Props interface defined
- [ ] Optional props marked with `?`
- [ ] CMS data attribute props typed or spread

---

---

## Step 7: Register Components in DB (Composition System)

**CRITICAL: After building each React component, register it in the DB composition system.** This enables the admin UI to manage page layouts, and allows the wizard's default compositions to be overwritten with Figma-accurate versions.

### 7.1 Register ComponentDefinitions

For each React component F2C generates, create or update a `ComponentDefinition` entity:

```graphql
mutation SaveComponentDefinition($input: SaveComponentDefinitionInput!) {
  saveComponentDefinition(input: $input) {
    id
    slug
    name
    componentPath
    category
    allowedSlots
    contentTypeId
  }
}
```

**Variables per component:**
```json
{
  "input": {
    "websiteProjectId": "[from Phase 1]",
    "slug": "hero-section",
    "name": "Hero Section",
    "componentPath": "components/organisms/HeroSection",
    "category": "hero",
    "allowedSlots": ["hero", "main"],
    "contentTypeId": "[ID of the hero-section content type]",
    "description": "Full-width hero with headline, subtitle, CTA, and background image"
  }
}
```

**Category mapping from Atomic Design levels:**

| Level | Category Values |
|-------|----------------|
| Organisms (heroes) | `hero` |
| Organisms (navigation) | `header` |
| Organisms (footers) | `footer` |
| Organisms (sections) | `section` |
| Molecules | `card`, `form-element`, `widget` |

### 7.2 Register ComponentVariants

Each component gets at least one default variant. Variants store Figma-extracted default styles.

```graphql
mutation SaveComponentVariant($input: SaveComponentVariantInput!) {
  saveComponentVariant(input: $input) {
    id
    slug
    name
    componentDefinitionId
    defaultStyles
    isDefault
  }
}
```

**Variables:**
```json
{
  "input": {
    "componentDefinitionId": "[from step 7.1]",
    "slug": "hero-section-default",
    "name": "Default",
    "isDefault": true,
    "defaultStyles": {
      "backgroundColor": "var(--color-primary)",
      "padding": "var(--spacing-xl)",
      "textAlign": "center",
      "minHeight": "80vh"
    }
  }
}
```

**If Figma has multiple variants** (e.g., split-screen, full-screen, gallery), create a variant for each:

```typescript
const heroVariants = [
  { slug: 'hero-split-screen', name: 'Split Screen', isDefault: true, defaultStyles: { ... } },
  { slug: 'hero-full-screen', name: 'Full Screen', isDefault: false, defaultStyles: { ... } },
  { slug: 'hero-gallery', name: 'Gallery', isDefault: false, defaultStyles: { ... } },
];

for (const variant of heroVariants) {
  await saveComponentVariant({
    componentDefinitionId: heroDefId,
    ...variant,
  });
}
```

### 7.3 Overwrite Wizard Components

If the wizard already created `ComponentDefinition` records with the same slug, F2C overwrites them. The `saveComponentDefinition` mutation uses upsert behavior (find by slug within the project, update if exists).

```typescript
// This will overwrite any wizard-created hero-section component
// with the Figma-accurate version
await saveComponentDefinition({
  websiteProjectId: config.websiteProjectId,
  slug: 'hero-section',  // Same slug = overwrites wizard version
  name: 'Hero Section',
  componentPath: 'components/organisms/HeroSection',
  category: 'hero',
  allowedSlots: ['hero', 'main'],
  contentTypeId: heroSectionTypeId,
});
```

### 7.4 Build Component Registry

After registering all components in the DB, generate the client-side component registry that maps DB component IDs to React components:

```typescript
// components/page-renderer/component-registry.ts
// Auto-generated by F2C — maps DB component slugs to React components
import { HeroSection } from '../organisms/HeroSection';
import { ContentSection } from '../organisms/ContentSection';
import { GallerySection } from '../organisms/GallerySection';
import { FaqSection } from '../organisms/FaqSection';
import { Navigation } from '../organisms/Navigation';
import { Footer } from '../organisms/Footer';
// ...

export const componentRegistry: Record<string, React.ComponentType<any>> = {
  'hero-section': HeroSection,
  'content-section': ContentSection,
  'gallery-section': GallerySection,
  'faq-section': FaqSection,
  'navigation': Navigation,
  'footer': Footer,
  // ... all registered components
};
```

**Note:** The registry uses component **slugs** as keys (not UUIDs), which makes it human-readable and stable across environments. The page renderer resolves slugs to actual components at render time.

### 7.5 Verification

After registering all components, verify:

```
┌─────────────────────────────────────────────────────────────┐
│  COMPONENT DB REGISTRATION COMPLETE                          │
├─────────────────────────────────────────────────────────────┤
│  ComponentDefinitions: 12 registered                         │
│  ComponentVariants: 18 registered                            │
│  Overwritten wizard components: 4                            │
│  Component registry generated at:                            │
│    components/page-renderer/component-registry.ts            │
└─────────────────────────────────────────────────────────────┘
```

---

## Git Commit

```bash
git commit -m "feat: Global component library built with DB composition

- Level 1: Atomic components (Button, Input, Link, etc.)
- Level 2: Molecule components (EventCard, FaqAccordion, etc.)
- Level 3: Organism components (Navigation, Footer, Hero, etc.)
- Level 4: Page templates (Standard, Gallery, Form, Listing)
- All components registered as ComponentDefinition + ComponentVariant in CMS
- Component registry generated for page renderer
- All components use design tokens
- All components have CMS data attributes
- Exceptions documented for components not in Figma library"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 7: Global component library built" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-8-PAGE-CONTENT.md](PHASE-8-PAGE-CONTENT.md)
