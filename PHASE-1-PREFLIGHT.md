# Phase 1: Pre-flight Validation & Authentication

Before building anything, validate credentials, explore the Figma file, and **auto-discover all pages**.

## Step 0: Save Configuration to Project

**CRITICAL: Save the user's pasted configuration before proceeding.**

When the user pastes their configuration prompt:

1. Create the project-config directory:
   ```bash
   mkdir -p project-config
   ```

2. Write the configuration to `/project-config/config.md`:
   ```typescript
   // Save the exact configuration the user pasted
   await writeFile('project-config/config.md', userPastedConfiguration);
   ```

3. Confirm to the user:
   ```
   ✓ Configuration saved to /project-config/config.md

   This file will be your project's reference document.
   It's safe to commit to version control (no secrets).
   ```

**Note:** If `/project-config/config.md` already exists (re-running the skill), read from it instead of asking for configuration again.

## Step 1: Read Configuration & Collect Credentials

1. Read `/project-config/config.md` and extract all configuration values

2. Display configuration summary:
```
Configuration Loaded:
─────────────────────
Project: [Project Name]
Figma File: [Figma Main File URL]

CMS:
  Type: [CMS Type]
  GraphQL URL: [CMS GraphQL URL]
  Organization ID: [CMS Organization ID]
  Admin Email: [CMS Admin Email]
  Admin Password: [provided/missing]
```

3. For any missing credentials, prompt the user:
```
┌─────────────────────────────────────────────────────────────┐
│  CREDENTIALS REQUIRED                                       │
├─────────────────────────────────────────────────────────────┤
│  CMS Admin Login:                                           │
│    URL: [CMS Admin Login URL]                               │
│    Email: [CMS Admin Email]                                 │
│                                                             │
│  Please provide the CMS admin password:                     │
│  > _                                                        │
│                                                             │
│  (This will NOT be written to any files)                    │
└─────────────────────────────────────────────────────────────┘
```

## Step 2: Authenticate with CMS

Test CMS authentication using the login mutation (adapt based on CMS schema):

```typescript
const { data, errors } = await client.mutate({
  mutation: LOGIN,
  variables: { email: cmsAdminEmail, password: cmsAdminPassword },
});

if (errors || !data?.login?.accessToken) {
  throw new Error('CMS authentication failed');
}
```

If login fails, STOP and report the error with troubleshooting steps.

## Step 2.5: Property Setup

**CRITICAL: Every website must be associated with a CMS Property.**

### 2.5.1 Check Property Configuration

Read the Property configuration from `/project-config/config.md`:
- `CMS Property ID` — Existing property ID, OR
- `CMS Property ID: CREATE_NEW` — Create a new property

### 2.5.2 Validate or Create Property

**If Property ID is provided:**

```graphql
query GetProperty($id: ID!) {
  property(id: $id) {
    id
    slug
    name
    organization { id }
  }
}
```

Verify the property exists and belongs to the correct organization.

**If CREATE_NEW:**

```graphql
mutation CreateProperty($input: CreatePropertyInput!) {
  createProperty(input: $input) {
    id
    slug
    name
  }
}
```

**Variables:**
```json
{
  "input": {
    "organizationId": "[CMS Organization ID from config]",
    "name": "[CMS Property Name from config]",
    "slug": "[CMS Property Slug from config]",
    "settings": {
      "timezone": "America/New_York",
      "defaultLanguage": "en"
    }
  }
}
```

### 2.5.3 Save Property ID

After validation/creation, update `/project-config/config.md` with the confirmed Property ID:

```markdown
**CMS Property ID:** prop_abc123 (confirmed)
```

Also update `.env.local`:
```bash
CMS_PROPERTY_ID=prop_abc123
```

### 2.5.4 Display Confirmation

```
┌─────────────────────────────────────────────────────────────┐
│  PROPERTY CONFIGURED                                        │
├─────────────────────────────────────────────────────────────┤
│  Property: NoMad Wynwood                                    │
│  Slug: nomad-wynwood                                        │
│  ID: prop_abc123                                            │
│  Organization: Acme Hotels                  │
│                                                             │
│  All content will be scoped to this property.               │
└─────────────────────────────────────────────────────────────┘
```

## Step 2.6: WebsiteProject Setup

**CRITICAL: Every F2C build must be linked to a WebsiteProject in the CMS.**

The `WebsiteProject` entity is the DB-driven composition anchor. It connects page templates, component definitions, design tokens, and build tasks to a single project. The wizard creates one during site initialization; F2C must either link to an existing one or create a new one.

### 2.6.1 Check WebsiteProject Configuration

Read `websiteProjectId` from `/project-config/config.md`:
- **If provided:** Validate it exists and belongs to the correct organization
- **If `CREATE_NEW` or not provided:** Create a new WebsiteProject

### 2.6.2 Validate or Create WebsiteProject

**If WebsiteProject ID is provided:**

```graphql
query GetWebsiteProject($id: ID!) {
  websiteProject(id: $id) {
    id
    name
    slug
    status
    organizationId
    propertyId
    figmaFileKey
    githubRepoUrl
    vercelProjectId
    productionUrl
  }
}
```

Verify the project exists and belongs to the correct organization/property.

**If CREATE_NEW or not provided:**

```graphql
mutation CreateWebsiteProject($input: CreateWebsiteProjectInput!) {
  createWebsiteProject(input: $input) {
    id
    name
    slug
    status
  }
}
```

**Variables:**
```json
{
  "input": {
    "organizationId": "[CMS Organization ID from config]",
    "propertyId": "[CMS Property ID from config]",
    "name": "[Project Name from config]",
    "slug": "[Project Slug from config]",
    "figmaFileKey": "[extracted from Figma URL]",
    "siteType": "[business type if known]",
    "status": "BUILDING"
  }
}
```

### 2.6.3 Save WebsiteProject ID

After validation/creation, update `/project-config/config.md`:

```markdown
**CMS WebsiteProject ID:** wp_abc123 (confirmed)
```

Also update `.env.local`:
```bash
WEBSITE_PROJECT_ID=wp_abc123
```

### 2.6.4 Display Confirmation

```
┌─────────────────────────────────────────────────────────────┐
│  WEBSITE PROJECT CONFIGURED                                  │
├─────────────────────────────────────────────────────────────┤
│  Project: NoMad Wynwood Website                              │
│  ID: wp_abc123                                               │
│  Status: BUILDING                                            │
│  Figma File Key: bGCUY43jJhcFUafm33ncgW                     │
│                                                              │
│  F2C will register components, compositions, and             │
│  design tokens against this project.                         │
└─────────────────────────────────────────────────────────────┘
```

### 2.6.5 What This Enables

With a linked `WebsiteProject`, F2C can:
- Register `ComponentDefinition` and `ComponentVariant` entities (Phase 7)
- Create `PageTemplate` and `PageComposition` records (Phase 7/9)
- Sync `DesignTokenSet` from Figma (Phase 3)
- Populate `PageSeoConfig` via the SEO module (Phase 8)
- Report build progress to `BuildTask` system (Phase 14)
- Update project with deployment info after deploy (Phase 14)

## Step 3: Figma Page Discovery (AUTO)

**CRITICAL: Pages are auto-discovered from Figma, NOT manually entered.**

Use the Figma MCP to automatically inventory all pages in the design file:

### 3.1 Extract File Key and Get Metadata

```typescript
// Extract file key from the main Figma URL
// https://www.figma.com/design/bGCUY43jJhcFUafm33ncgW/Project-Name
const fileKey = figmaMainUrl.match(/\/design\/([^\/]+)/)?.[1];

// Use Figma MCP to get file metadata
const metadata = await mcp__figma__get_metadata({
  fileKey,
  nodeId: "0:1" // Root node to get all pages
});
```

### 3.2 Identify Page Frames

Parse the metadata to identify all website page frames. Look for:
- Frames with dimensions matching standard breakpoints (1440px desktop, 375px mobile)
- Frames named with page identifiers (Homepage, About, Menu, Contact, etc.)
- Desktop/Mobile pairs for each page

**Naming conventions to detect:**
- `1440 - Homepage`, `375 - Homepage` (dimension prefix)
- `Homepage Desktop`, `Homepage Mobile` (device suffix)
- `Homepage`, `Homepage (Mobile)` (parenthetical)

### 3.3 Auto-Detect Global Components

Identify navigation and footer frames:
- Look for frames named: `Navigation`, `Nav`, `Header`, `Menu Overlay`, `Expanded Menu`
- Look for frames named: `Footer`, `Site Footer`
- Match desktop (1440px) and mobile (375px) variants

### 3.4 Generate Page Inventory

Create `/project-config/pages.md` with discovered pages:

```markdown
# Auto-Discovered Pages

> Generated from Figma file: [Main File URL]
> Discovery date: [Current Date]

## Website Pages

| Page Name | Slug | Desktop Frame | Mobile Frame |
|-----------|------|---------------|--------------|
| Homepage | / | [node-id=195:1342](url) | [node-id=418:3286](url) |
| Menu | /menu | [node-id=297:1732](url) | [node-id=428:4784](url) |
| About | /about | [node-id=112:6096](url) | [node-id=418:3764](url) |
...

## Global Components

| Component | Desktop Frame | Mobile Frame |
|-----------|---------------|--------------|
| Navigation | [node-id=X:Y](url) | [node-id=X:Y](url) |
| Footer | [node-id=X:Y](url) | [node-id=X:Y](url) |
| Expanded Menu | [node-id=X:Y](url) | [node-id=X:Y](url) |

## Summary

- **Total Pages:** X
- **Desktop Frames:** X (1440px)
- **Mobile Frames:** X (375px)
```

### 3.5 User Confirmation

Display discovered pages and ask for confirmation:

```
┌─────────────────────────────────────────────────────────────┐
│  FIGMA PAGE DISCOVERY COMPLETE                              │
├─────────────────────────────────────────────────────────────┤
│  Found 9 pages:                                             │
│    ✓ Homepage (Desktop + Mobile)                            │
│    ✓ Menu (Desktop + Mobile)                                │
│    ✓ Programming (Desktop + Mobile)                         │
│    ✓ Private Events (Desktop + Mobile)                      │
│    ✓ About (Desktop + Mobile)                               │
│    ✓ Contact (Desktop + Mobile)                             │
│    ✓ FAQ (Desktop + Mobile)                                 │
│    ✓ 404 (Desktop + Mobile)                                 │
│                                                             │
│  Global Components:                                         │
│    ✓ Navigation (Desktop + Mobile)                          │
│    ✓ Footer (Desktop + Mobile)                              │
│    ✓ Expanded Menu (Desktop + Mobile)                       │
│                                                             │
│  Saved to: /project-config/pages.md                         │
│                                                             │
│  Review and confirm these pages are correct.                │
│  You can edit pages.md to exclude pages or adjust slugs.    │
└───────────────────────────���─────────────────────────────────┘
```

**If pages are missing or incorrectly identified:**
- User can manually edit `/project-config/pages.md`
- Or provide hints in config.md under `## Page Hints` section

### 3.6 Export Wireframe Screenshots (Visual Reference)

**CRITICAL: Export every page wireframe as a PNG image for pixel-accurate development reference.**

These exported images serve as the ground truth throughout all subsequent phases. Every component, layout, and page built in code must be compared against these screenshots to ensure a 1:1 match with the Figma design.

#### 3.6.1 Create Export Directory

```bash
mkdir -p docs/figma-wireframes/desktop
mkdir -p docs/figma-wireframes/mobile
mkdir -p docs/figma-wireframes/global-components
```

#### 3.6.2 Export All Page Frames

For every page discovered in Step 3, export both desktop and mobile wireframes:

```typescript
// Export each page wireframe as a full-frame screenshot
for (const page of discoveredPages) {
  // Desktop wireframe
  if (page.desktopNodeId) {
    const desktopScreenshot = await mcp__figma__get_screenshot({
      fileKey,
      nodeId: page.desktopNodeId,
      clientFrameworks: 'react,nextjs',
      clientLanguages: 'typescript'
    });
    // Save to docs/figma-wireframes/desktop/{page-slug}.png
  }

  // Mobile wireframe
  if (page.mobileNodeId) {
    const mobileScreenshot = await mcp__figma__get_screenshot({
      fileKey,
      nodeId: page.mobileNodeId,
      clientFrameworks: 'react,nextjs',
      clientLanguages: 'typescript'
    });
    // Save to docs/figma-wireframes/mobile/{page-slug}.png
  }
}
```

#### 3.6.3 Export Global Component Frames

Also export navigation, footer, and other global components:

```typescript
for (const component of globalComponents) {
  // Desktop variant
  if (component.desktopNodeId) {
    const screenshot = await mcp__figma__get_screenshot({
      fileKey,
      nodeId: component.desktopNodeId,
      clientFrameworks: 'react,nextjs',
      clientLanguages: 'typescript'
    });
    // Save to docs/figma-wireframes/global-components/{name}-desktop.png
  }

  // Mobile variant
  if (component.mobileNodeId) {
    const screenshot = await mcp__figma__get_screenshot({
      fileKey,
      nodeId: component.mobileNodeId,
      clientFrameworks: 'react,nextjs',
      clientLanguages: 'typescript'
    });
    // Save to docs/figma-wireframes/global-components/{name}-mobile.png
  }
}
```

#### 3.6.4 Generate Wireframe Index

Create `/docs/figma-wireframes/INDEX.md`:

```markdown
# Figma Wireframe Exports

> Exported from Figma file: [Main File URL]
> Export date: [Current Date]
>
> **These images are the design source of truth.**
> Every page and component built in code must visually match these wireframes.

## How to Use These References

1. **During component building (Phase 7):** Open the relevant wireframe to verify
   layout, spacing, and visual hierarchy match the design
2. **During page assembly (Phase 9):** Compare the built page side-by-side with
   the wireframe to catch layout drift
3. **During QA (Phase 12):** Playwright screenshots are compared against these
   wireframes for pixel-level verification

## Desktop Wireframes

| Page | Wireframe | Figma Node |
|------|-----------|------------|
| [Page Name] | [desktop/{slug}.png](desktop/{slug}.png) | node-id=X:Y |
...

## Mobile Wireframes

| Page | Wireframe | Figma Node |
|------|-----------|------------|
| [Page Name] | [mobile/{slug}.png](mobile/{slug}.png) | node-id=X:Y |
...

## Global Components

| Component | Desktop | Mobile |
|-----------|---------|--------|
| [Name] | [global-components/{name}-desktop.png](...) | [global-components/{name}-mobile.png](...) |
...
```

#### 3.6.5 Gitignore Consideration

These wireframe PNGs should be committed to the repo (they're reference docs, not build artifacts). However, add a size warning:

```bash
# Check total size of exported wireframes
du -sh docs/figma-wireframes/
# If > 50MB, consider optimizing with:
# find docs/figma-wireframes -name "*.png" -exec pngquant --quality=65-80 --ext .png --force {} \;
```

#### 3.6.6 User Confirmation

```
┌─────────────────────────────────────────────────────────────────┐
│  WIREFRAME EXPORT COMPLETE                                       │
├─────────────────────────────────────────────────────────────────┤
│  Exported [N] wireframe screenshots:                             │
│    • [X] desktop wireframes                                      │
│    • [Y] mobile wireframes                                       │
│    • [Z] global component frames                                 │
│                                                                  │
│  Total size: [size]                                              │
│                                                                  │
│  Saved to: /docs/figma-wireframes/                               │
│  Index: /docs/figma-wireframes/INDEX.md                          │
│                                                                  │
│  These images are your visual reference for the entire build.    │
│  Compare every component and page against these wireframes.      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 4: Figma Structure Detection (4-Page Pattern)

**CRITICAL: Identify the standard 4-page Figma structure before component discovery.**

Well-structured Figma files follow this pattern:

| Page | Purpose | Typical Names |
|------|---------|---------------|
| Wireframes | Page designs (desktop + mobile) | `Wireframes`, `💻 Wireframes`, `Pages`, `Screens` |
| Components | Reusable UI components | `Components`, `🍱 Components`, `UI Kit`, `Component Library` |
| Design System | Tokens (spacing, typography, colors) | `Design System`, `🎨 Design System`, `Tokens`, `Styles` |
| Assets | Logos, icons, brand assets | `Assets`, `📦 Assets`, `Assets & Favicon`, `Brand` |

### 4.1 Detect Page Structure

```typescript
// Get all pages from Figma file
const fileMetadata = await mcp__figma__get_metadata({ fileKey, nodeId: "0:1" });

// Identify each page type
const pagePatterns = {
  wireframes: /^(💻\s*)?(wireframes?|pages?|screens?)$/i,
  components: /^(🍱\s*)?(components?|ui\s*kit|component\s*library)$/i,
  designSystem: /^(🎨\s*)?(design\s*system|tokens?|styles?)$/i,
  assets: /^(📦\s*)?(assets?|brand|assets?\s*&?\s*favicon)$/i,
};

const figmaStructure = {
  wireframes: findPage(pagePatterns.wireframes),
  components: findPage(pagePatterns.components),
  designSystem: findPage(pagePatterns.designSystem),
  assets: findPage(pagePatterns.assets),
};
```

### 4.2 Validate Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  FIGMA STRUCTURE DETECTED                                       │
├─────────────────────────────────────────────────────────────────┤
│  ✓ Wireframes: "💻 Wireframes" (node-id=0:1)                    │
│  ✓ Components: "🍱 Components" (node-id=1393:5395)              │
│  ✓ Design System: "🎨 Design System" (node-id=1393:5396)        │
│  ✓ Assets: "📦 Assets & Favicon" (node-id=8:10846)              │
│                                                                 │
│  ✓ Standard 4-page structure detected!                          │
└─────────────────────────────────────────────────────────────────┘
```

**If pages are missing:**
- Missing Components page: Warn, components will be inline
- Missing Design System: Warn, tokens extracted from components
- Missing Assets: Warn, assets exported from wireframes

Save structure to `/project-config/figma-structure.md`.

---

## Step 5: Component Library Discovery

**CRITICAL: Pre-fetch component metadata to avoid duplicate work during build.**

Figma design files typically have a dedicated Components page containing reusable UI components. Extract this inventory upfront so Phase 7 can reference existing components instead of rebuilding them.

### 5.1 Locate Components Page

Use the structure detected in Step 4:

```typescript
const componentsPageNodeId = figmaStructure.components?.nodeId;

if (!componentsPageNodeId) {
  console.warn('No Components page found - skipping component discovery');
}
```

### 5.2 Extract Component Metadata

Use Figma MCP to get all components on the page:

```typescript
// Get metadata for the Components page
const componentMetadata = await mcp__figma__get_metadata({
  fileKey,
  nodeId: componentsPageNodeId
});
```

### 5.3 Parse Component Structure

Identify all component frames and their variants:
- Top-level frames are typically component categories (Buttons, Cards, Forms, etc.)
- Child frames are individual components
- Look for variant indicators: `Default`, `Hover`, `Active`, `Disabled`, `Small`, `Large`

**Detect nested components:**
- Components that contain other components (e.g., Navigation contains MenuLink contains Button)
- Track parent-child relationships for content modeling

### 5.4 Generate Component Inventory

Create `/project-config/components.md`:

```markdown
# Component Library Inventory

> Generated from Figma file: [Main File URL]
> Components page node ID: [Node ID]
> Discovery date: [Current Date]

## Component Categories

### Heroes
| Component | Node ID | Variants | Nested Components | Used On Pages |
|-----------|---------|----------|-------------------|---------------|
| hero-option-1 | 355:6661 | split-screen, full-screen | button | Homepage, About |
| hero-option-2 | 355:6662 | desktop, mobile | button, link | Menu |

### Navigation
| Component | Node ID | Variants | Nested Components | Used On Pages |
|-----------|---------|----------|-------------------|---------------|
| navigation-desktop | 355:7871 | Default | logo, menu-link, button | Global |
| menu-link | 355:7880 | Default, Hover, Active | — | Navigation |

### Buttons & Links
| Component | Node ID | Variants | Nested Components | Used On Pages |
|-----------|---------|----------|-------------------|---------------|
| button | 355:7366 | Default, Hover × fill/stroke | — | Multiple |
| link | 355:7367 | Default, Hover | — | Multiple |

### Cards
| Component | Node ID | Variants | Nested Components | Used On Pages |
|-----------|---------|----------|-------------------|---------------|
| event-card | 234:567 | Default, Hover | button | Programming |
| faq-accordion | 234:568 | Default, Expanded | — | FAQ, About |

### Form Inputs
| Component | Node ID | Variants | Nested Components | Used On Pages |
|-----------|---------|----------|-------------------|---------------|
| Input | 345:678 | Default, Filled, Error | — | Contact |
| Dropdown | 345:679 | Default, Expanded | — | Contact |

## Nested Component Map

Shows which components are used inside other components:

```
navigation-desktop
├── logo (NoMad-Wynwood__Mark-01)
├── menu-link (×5)
└── button (CTA)

hero-option-1
└── button (CTA)

event-card
└── button (Learn More)
```

## Summary

- **Total Components:** X
- **Categories:** X
- **With Variants:** X
- **Nested Relationships:** X

## Usage Notes

Reference this file during Phase 7 (Component Building) to:
1. Check if a component already exists before building
2. Get the correct node ID for `get_design_context`
3. Understand component variants to implement
4. Build components in dependency order (atomic → molecules → organisms)
```

### 5.5 User Confirmation

```
┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT LIBRARY DISCOVERY COMPLETE                           │
├─────────────────────────────────────────────────────────────────┤
│  Found 24 components in 6 categories:                           │
│    • Heroes (3 components)                                      │
│    • Navigation (4 components)                                  │
│    • Buttons & Links (5 components)                             │
│    • Cards (6 components)                                       │
│    • Form Inputs (4 components)                                 │
│    • Sections (2 components)                                    │
│                                                                 │
│  Nested relationships detected:                                 │
│    • navigation-desktop → menu-link, button, logo               │
│    • hero-option-1 → button                                     │
│    • event-card → button                                        │
│                                                                 │
│  Saved to: /project-config/components.md                        │
└─────────────────────────────────────────────────────────────────┘
```

**If no Components page exists:**
- Log a warning but continue
- Components will be discovered ad-hoc during page building

---

## Step 6: Component Coverage Analysis

**CRITICAL: Compare components USED in Wireframes vs components DEFINED in Components page.**

This analysis ensures the component library covers all instances used in the actual page designs.

### 6.1 Extract Instances from Wireframes

Scan all wireframe pages for component instances:

```typescript
// Get all component instances used in Wireframes
const wireframesMetadata = await mcp__figma__get_metadata({
  fileKey,
  nodeId: figmaStructure.wireframes.nodeId,
  depth: 'full' // Need deep scan to find all instances
});

// Extract unique component names and count usage
const instanceUsage = extractComponentInstances(wireframesMetadata);
// Returns: { 'button': 15, 'faq-accordion': 54, 'event-card': 9, ... }
```

### 6.2 Compare with Component Definitions

Cross-reference instances with the component inventory:

```typescript
const definedComponents = componentsInventory.map(c => c.name);
const usedComponents = Object.keys(instanceUsage);

const coverage = {
  defined: definedComponents,
  used: usedComponents,
  missing: usedComponents.filter(c => !definedComponents.includes(c)),
  unused: definedComponents.filter(c => !usedComponents.includes(c)),
  coveragePercent: Math.round((matched / usedComponents.length) * 100)
};
```

### 6.3 Generate Coverage Report

Create `/project-config/component-coverage.md`:

```markdown
# Component Coverage Analysis

> Wireframes page: [node-id]
> Components page: [node-id]
> Analysis date: [Current Date]

## Coverage Summary

| Metric | Value |
|--------|-------|
| Components Used in Wireframes | 35 |
| Components Defined in Library | 33 |
| Coverage | 94% |
| Missing (need to create) | 2 |
| Unused (defined but not used) | 0 |

## Components by Usage Frequency

| Component | Times Used | Pages | Status |
|-----------|------------|-------|--------|
| faq-accordion | 54 | FAQ, About, Contact | ✅ Defined |
| button | 15 | Multiple | ✅ Defined |
| footer | 15 | All pages | ✅ Defined |
| navigation-desktop | 8 | All pages | ✅ Defined |
| custom-badge | 3 | Homepage | ⚠️ MISSING |

## Gap Analysis

### Missing Components (create as exceptions)

| Component | Used On | Recommendation |
|-----------|---------|----------------|
| custom-badge | Homepage | Create inline, consider promoting to library |

### Mobile Variant Gaps

| Component | Desktop | Mobile | Pages Affected |
|-----------|---------|--------|----------------|
| Input | ✅ | ❌ | Contact |

## Component Build Order

Based on dependencies, build in this order:

### Level 1: Atomic (no dependencies)
- button, link, Input, Dropdown, Radio, Checkbox

### Level 2: Molecules (use Level 1)
- menu-link, faq-accordion, event-card

### Level 3: Organisms (use Level 1 & 2)
- navigation-desktop, navigation-mobile, footer, hero-option-1

### Level 4: Templates
- Page templates compose Level 3 organisms
```

### 6.4 User Confirmation

```
┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT COVERAGE ANALYSIS COMPLETE                           │
├─────────────────────────────────────────────────────────────────┤
│  Coverage: 94% (33/35 components defined)                       │
│                                                                 │
│  ✅ Covered: 33 components                                      │
│  ⚠️  Missing: 2 components (will be created as exceptions)      │
│                                                                 │
│  Build order determined:                                        │
│    Level 1: Atomic (6 components)                               │
│    Level 2: Molecules (3 components)                            │
│    Level 3: Organisms (5 components)                            │
│                                                                 │
│  Saved to: /project-config/component-coverage.md                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 7: Dynamic Breakpoint Assessment

**CRITICAL: After completing all Phase 1 analysis, assess project complexity to determine if additional breakpoints are needed.**

Based on the outputs from Steps 4-6, evaluate whether the default breakpoints are sufficient or if additional breakpoints should be inserted for this project.

### 7.1 Gather Complexity Metrics

From the Phase 1 outputs, extract:

```typescript
const complexityMetrics = {
  totalComponents: componentsInventory.length,           // from components.md
  totalPages: pagesInventory.length,                     // from pages.md
  nestedDepth: maxNestingLevel(componentsInventory),     // from nested component map
  coveragePercent: coverage.coveragePercent,             // from component-coverage.md
  missingComponents: coverage.missing.length,             // from component-coverage.md
  contentTypesNeeded: estimateContentTypes(componentsInventory), // estimate
};
```

### 7.2 Apply Complexity Thresholds

| Factor | Low | Medium | High |
|--------|-----|--------|------|
| Total Components | <15 | 15-30 | >30 |
| Pages | <5 | 5-10 | >10 |
| Nested Component Depth | 1 level | 2 levels | 3+ levels |
| Component Coverage | >90% | 70-90% | <70% |
| Content Types Needed | <5 | 5-10 | >10 |

### 7.3 Determine Dynamic Breakpoints

```typescript
const highFactors = countHighFactors(complexityMetrics);

let dynamicBreakpoints = [];

if (complexityMetrics.totalComponents > 30) {
  dynamicBreakpoints.push('after-phase-7');  // Component Generation
}

if (complexityMetrics.totalPages > 10) {
  dynamicBreakpoints.push('after-phase-8');  // Page Content Extraction
}

if (estimatedContentEntries > 50) {
  dynamicBreakpoints.push('after-phase-6');  // Split Group A
}

if (complexityMetrics.coveragePercent < 70) {
  dynamicBreakpoints.push('after-phase-7');  // Many exceptions to handle
}

if (complexityMetrics.nestedDepth >= 3) {
  dynamicBreakpoints.push('after-phase-7');  // Complex nested content modeling
}
```

### 7.4 Generate Dynamic Breakpoint Assessment

Create `/project-config/dynamic-breakpoints.md`:

```markdown
# Dynamic Breakpoint Assessment

> Based on Phase 1 analysis
> Assessment date: [Current Date]

## Complexity Metrics

| Factor | Value | Rating |
|--------|-------|--------|
| Total Components | 35 | 🔴 HIGH |
| Pages | 9 | 🟡 MEDIUM |
| Nested Component Depth | 2 levels | 🟡 MEDIUM |
| Component Coverage | 94% | 🟢 LOW |
| Content Types Needed | 8 | 🟡 MEDIUM |

**Overall Complexity Score:** MODERATE (2 HIGH factors)

## Dynamic Breakpoints

Based on complexity analysis, the following additional breakpoints are recommended:

| Breakpoint | Location | Reason |
|------------|----------|--------|
| ✅ Dynamic #1 | After Phase 7 | 35 components exceeds threshold (30) |

## Updated Phase Group Structure

```
PHASE GROUP A: Phases 1-6
  └── Static Breakpoint 1 (after Phase 6)

PHASE GROUP B: Phases 7-9
  ├── ⚡ DYNAMIC Breakpoint (after Phase 7) ← NEW
  └── Static Breakpoint 2 (after Phase 9)

PHASE GROUP C: Phases 10-12
  └── Static Breakpoint 3 (after Phase 12)

PHASE GROUP D: Phases 13-14
  └── Build Complete
```

## Continuity Files Expected

- `/project-config/continuity-group-a.md` (static)
- `/project-config/continuity-dynamic-phase-7.md` (dynamic - NEW)
- `/project-config/continuity-group-b.md` (static)
- `/project-config/continuity-group-c.md` (static)
```

### 7.5 User Notification

```
┌─────────────────────────────────────────────────────────────────┐
│  DYNAMIC BREAKPOINT ASSESSMENT                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Project Complexity: MODERATE                                   │
│                                                                 │
│  Factors:                                                       │
│    • 35 components (🔴 HIGH - threshold: 30)                   │
│    • 9 pages (🟡 MEDIUM)                                        │
│    • 94% coverage (🟢 LOW)                                      │
│    • 2-level nesting (🟡 MEDIUM)                                │
│                                                                 │
│  ⚡ 1 Dynamic Breakpoint Added:                                  │
│    • After Phase 7 (Component Generation)                       │
│                                                                 │
│  This will improve output quality for the component library.    │
│                                                                 │
│  Saved to: /project-config/dynamic-breakpoints.md               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 8: Figma Reference Frames

1. Identify reference frames (Style Guide, Components, Design System)
2. Document in `/docs/figma-reference-frames.md`
3. These are NOT pages to build, but resources for token extraction

## Step 9: Font Check

1. Extract all font families from Figma
2. Create `/docs/font-manifest.json`:
```json
{
  "fonts": [
    {"family": "Inter", "source": "google", "status": "available"},
    {"family": "Custom Font", "source": "custom", "status": "missing", "suggestedFallback": "Source Sans Pro"}
  ],
  "missingFonts": ["Custom Font"],
  "fontHandling": "STRICT",
  "canProceed": false
}
```

3. If STRICT mode and fonts missing, STOP and prompt user for resolution

## Checkpoint

```bash
osascript -e 'display notification "Phase 1: Pre-flight complete" with title "Claude Code" sound name "Ping"'
```

**Outputs:**
- `/project-config/config.md` - Project configuration (including websiteProjectId)
- `/project-config/figma-structure.md` - 4-page Figma structure detection
- `/project-config/pages.md` - Auto-discovered page inventory
- `/project-config/components.md` - Component library inventory with nested relationships
- `/project-config/component-coverage.md` - Coverage analysis and build order
- `/project-config/dynamic-breakpoints.md` - Dynamic breakpoint assessment and schedule
- `/docs/figma-wireframes/` - Exported wireframe PNGs (desktop, mobile, global components)
- `/docs/figma-wireframes/INDEX.md` - Wireframe index with links and node IDs
- `/docs/figma-reference-frames.md` - Reference frame documentation
- `/docs/font-manifest.json` - Font availability check
- `.env.local` updated with `WEBSITE_PROJECT_ID`

**Next:** Proceed to [PHASE-2-INIT.md](PHASE-2-INIT.md)
