# Phase 5.5: Media Extraction & CMS Upload

> **CRITICAL:** All images MUST be uploaded through the CMS `uploadMedia` mutation — never directly to R2 or any other storage. The CMS handles CDN storage internally.
> CMS media fields should contain CDN URLs returned by the CMS, not local paths like `/images/...`.

This phase extracts images from Figma and uploads them to the CMS, which stores them in CDN storage (R2).

## Why Upload Through the CMS?

1. **Single upload path**: All media goes through one API — no direct bucket access needed
2. **CMS-managed**: Images can be updated via the CMS without code changes
3. **CDN delivery**: Fast, globally distributed delivery
4. **No repo bloat**: Images don't increase repository size
5. **Inline editing**: Users can update images through the visual editor

## Step 1: Identify Required Images

Review Figma designs and list all unique images needed:

```markdown
### Image Inventory
| Figma Node | Purpose | Suggested Filename | Alt Text |
|------------|---------|-------------------|----------|
| hero-left | Homepage hero left | hero-left.jpg | [Brand] atmosphere |
| hero-right | Homepage hero right | hero-right.jpg | [Brand] dining |
| logo | Navigation logo | nav-logo.svg | [Brand] logo |
| wordmark | Hero wordmark | wordmark.svg | [Brand] wordmark |
```

## Step 2: Export Images from Figma

Use the Figma MCP tools to get screenshots of image nodes:

```typescript
// Get screenshot of a Figma node
const screenshot = await mcp__figma__get_screenshot({
  fileKey: config.figmaFileKey,  // e.g., 'bGCUY43jJhcFUafm33ncgW'
  nodeId: '466:5965',             // Node ID from Figma URL or metadata
  clientFrameworks: 'react,nextjs',
  clientLanguages: 'typescript'
});
```

**To find image node IDs:**
1. Use `mcp__figma__get_metadata` to explore the file structure
2. Search for image names in the metadata output
3. Look for `rounded-rectangle` or `image` elements with descriptive names

## Step 3: Create Upload Script

Create a TypeScript script to upload images through the CMS API:

```typescript
// scripts/upload-media.ts
import * as fs from 'fs';
import * as path from 'path';

const API_URL = process.env.CMS_API_URL;
const ORG_ID = process.env.CMS_ORG_ID;
const PROPERTY_ID = process.env.CMS_PROPERTY_ID;

async function uploadImage(token: string, filepath: string, alt: string) {
  const filename = path.basename(filepath);
  const fileBuffer = fs.readFileSync(filepath);
  const base64Data = fileBuffer.toString('base64');

  // Determine MIME type
  let mimeType = 'image/jpeg';
  if (filename.endsWith('.svg')) mimeType = 'image/svg+xml';
  else if (filename.endsWith('.png')) mimeType = 'image/png';

  const response = await fetch(API_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      query: `mutation UploadMedia($input: UploadMediaInput!) {
        uploadMedia(input: $input) {
          id
          filename
          variants { original thumbnail medium large }
        }
      }`,
      variables: {
        input: {
          organizationId: ORG_ID,
          propertyId: PROPERTY_ID,
          filename,
          mimeType,
          base64Data,
          alt,
        },
      },
    }),
  });

  const data = await response.json();
  return data.data?.uploadMedia?.variants?.original;
}
```

## Step 4: Run Upload Script

```bash
npx tsx scripts/upload-media.ts
```

The script will output a mapping file (`.uploaded-images.json`):

```json
{
  "hero-left.jpg": "f1cbcc92-90b3-4f2b-a8e9-4735df35e75b/original.jpg",
  "hero-right.jpg": "800c4c62-23c3-4437-a0e9-63e19344dd13/original.jpg"
}
```

## Step 5: Construct Full CDN URLs

The R2 CDN base URL is in your infrastructure config:
- **URL Pattern:** `https://pub-{bucket-id}.r2.dev/{asset-id}/{variant}.{ext}`
- **Example:** `https://pub-2d4c0900f460406880526481200075fb.r2.dev/f1cbcc92-90b3-4f2b-a8e9-4735df35e75b/original.jpg`

## Step 6: Configure Next.js Images

Ensure `next.config.ts` allows the R2 domain:

```typescript
images: {
  remotePatterns: [
    {
      protocol: "https",
      hostname: "pub-2d4c0900f460406880526481200075fb.r2.dev",
      pathname: "/**",
    },
  ],
},
```

## Important Notes

1. **Always upload through the CMS** — never directly to R2 or any CDN bucket. No R2 credentials are needed or should be used.
2. **Never use local paths** like `/images/foo.jpg` in CMS entries
3. **Always use the CDN URLs returned by the CMS** — the CMS handles storage internally and returns the full CDN URL
4. **Include alt text** when uploading for accessibility
5. **Always include propertyId** to scope images to the correct website

## CMS Entry Image Fields

When creating CMS entries (Phase 6, 8), use the full R2 CDN URLs:

```typescript
const siteSettings = {
  logoUrl: 'https://pub-xxx.r2.dev/asset-id/original.svg',  // ✓ R2 URL
  // logoUrl: '/images/logo.svg',  // ✗ WRONG - local path
};
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 5.5: Media uploaded via CMS" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-6-GLOBAL-CONTENT.md](PHASE-6-GLOBAL-CONTENT.md)
