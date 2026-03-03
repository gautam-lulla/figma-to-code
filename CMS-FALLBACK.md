# CMS Fallback System

When building sites with the Halo CMS, implement a fallback system so the site stays up even if the CMS is unavailable.

## How It Works

```
Page Request → Fetch from CMS → Success? → Render
                     ↓
                   Fail?
                     ↓
              Load from local JSON → Render with cached content
```

## Implementation

### 1. Create Fallback Loader

Create `src/lib/content/fallback.ts`:

```typescript
import * as fs from 'fs';
import * as path from 'path';

const FALLBACK_DIR = path.join(process.cwd(), 'src/content/fallback');

export function hasFallbackContent(): boolean {
  return fs.existsSync(FALLBACK_DIR);
}

export function loadFallbackEntry<T>(contentType: string, slug: string): T | null {
  try {
    const filePath = path.join(FALLBACK_DIR, contentType, `${slug}.json`);
    if (!fs.existsSync(filePath)) return null;
    return JSON.parse(fs.readFileSync(filePath, 'utf-8')) as T;
  } catch {
    return null;
  }
}

export function loadFallbackEntries<T>(contentType: string): T[] {
  try {
    const typeDir = path.join(FALLBACK_DIR, contentType);
    if (!fs.existsSync(typeDir)) return [];

    return fs.readdirSync(typeDir)
      .filter(f => f.endsWith('.json'))
      .map(f => JSON.parse(fs.readFileSync(path.join(typeDir, f), 'utf-8')));
  } catch {
    return [];
  }
}
```

### 2. Modify Content Fetchers

Wrap CMS calls with try/catch and fallback:

```typescript
async function fetchEntry<T>(contentType: string, slug: string): Promise<T | null> {
  try {
    // Try CMS first
    const data = await client.query({ ... });
    return data;
  } catch (error) {
    console.error(`CMS error for ${contentType}/${slug}:`, error);

    // Fallback to local content
    if (hasFallbackContent()) {
      console.warn(`[Fallback] Using local content for ${contentType}/${slug}`);
      return loadFallbackEntry<T>(contentType, slug);
    }

    return null;
  }
}
```

### 3. Create Export Script

Create `scripts/export-cms-content.ts` to snapshot CMS content:

```typescript
import { ApolloClient, gql } from '@apollo/client';
import * as fs from 'fs';
import * as path from 'path';

const CONTENT_TYPES = ['page', 'hero-section', 'navigation', ...];

async function exportContent() {
  const contentDir = 'src/content/fallback';

  for (const contentType of CONTENT_TYPES) {
    const entries = await fetchAllEntries(contentType);

    // Create directory
    fs.mkdirSync(path.join(contentDir, contentType), { recursive: true });

    // Write each entry
    for (const entry of entries) {
      fs.writeFileSync(
        path.join(contentDir, contentType, `${entry.slug}.json`),
        JSON.stringify(entry.data, null, 2)
      );
    }
  }

  // Write manifest
  fs.writeFileSync(
    path.join(contentDir, 'manifest.json'),
    JSON.stringify({ exportedAt: new Date().toISOString() })
  );
}
```

### 4. Run Export

```bash
npx tsx scripts/export-cms-content.ts
```

## Fallback Content Structure

```
src/content/fallback/
├── manifest.json           # Export timestamp
├── page/
│   ├── home.json
│   ├── about.json
│   └── ...
├── hero-section/
│   ├── hero-homepage.json
│   └── ...
├── navigation/
│   └── global-navigation.json
├── footer/
│   └── global-footer.json
├── site-settings/
│   └── global-settings.json
└── [content-type]/
    └── [slug].json
```

## Best Practices

1. **Commit fallback content to git** - Ensures it's available during deploys even if CMS is down

2. **Run export after major content updates** - Keep fallback reasonably current

3. **Log when using fallback** - Helps debugging and monitoring

4. **Consider a banner** - Optionally show "viewing cached content" in dev/preview

## When to Update Fallback

- After major content changes in CMS
- Before important deployments
- Periodically (weekly/monthly) as a safety net

## Monitoring

Export `isUsingFallback()` function to check if site is running on cached content:

```typescript
export function isUsingFallback(): boolean {
  return usingFallback;
}

// Use in layout to add monitoring header
if (isUsingFallback()) {
  headers.set('X-Content-Source', 'fallback');
}
```
