# Phase 14: Deployment

**Skip if Deployment Platform = None in config**

## For Vercel

1. Install Vercel CLI:
```bash
npm i -g vercel
```

2. Deploy:
```bash
vercel --prod
```

3. Set environment variables in Vercel dashboard:
   - `NEXT_PUBLIC_CMS_GRAPHQL_URL`
   - `CMS_ORGANIZATION_ID`
   - `CDN_BUCKET_URL`

## For AWS Amplify

1. Create Amplify app in AWS Console

2. Connect to repository

3. Configure build settings:
```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

4. Set environment variables in Amplify Console

## For Netlify

1. Install Netlify CLI:
```bash
npm i -g netlify-cli
```

2. Deploy:
```bash
netlify deploy --prod
```

3. Set environment variables in Netlify dashboard

## Inline Editor Script

Add the inline editor script to `src/app/layout.tsx` before the closing `</body>` tag:

Use the `InlineEditorLoader` component to conditionally load the editor when `?edit=true` is in the URL:

```tsx
// src/components/InlineEditorLoader.tsx
'use client';

import { useEffect } from 'react';
import { useSearchParams } from 'next/navigation';

interface InlineEditorLoaderProps {
  orgSlug: string;
  apiBase?: string;
  adminBase?: string;
}

export function InlineEditorLoader({
  orgSlug,
  apiBase = 'https://backend-production-162b.up.railway.app',
  adminBase = 'https://www.myhalo.agency',
}: InlineEditorLoaderProps) {
  const searchParams = useSearchParams();
  const isEditMode = searchParams.get('edit') === 'true';

  useEffect(() => {
    if (!isEditMode) return;
    if (document.querySelector('script[data-cms-editor]')) return;

    const script = document.createElement('script');
    script.src = `${apiBase}/inline-editor.js`;
    script.dataset.cmsOrg = orgSlug;
    script.dataset.cmsApi = apiBase;
    if (adminBase) script.dataset.cmsAdmin = adminBase;
    script.dataset.cmsEditor = 'true';
    script.defer = true;
    document.head.appendChild(script);
  }, [isEditMode, orgSlug, apiBase, adminBase]);

  return null;
}
```

Then in `layout.tsx`:
```tsx
import { Suspense } from 'react';
import { InlineEditorLoader } from '@/components/InlineEditorLoader';

// In the body:
<Suspense fallback={null}>
  <InlineEditorLoader orgSlug="your-org-slug" />
</Suspense>
```

This enables inline editing for authenticated CMS users when they visit with `?edit=true`.

## CMS Fallback for Resilience

Before going to production, set up the CMS fallback system so the site stays up even if the CMS is unavailable.

See [CMS-FALLBACK.md](CMS-FALLBACK.md) for full implementation details.

**Quick setup:**
1. Create `src/lib/content/fallback.ts` with local JSON loader
2. Wrap content fetchers with try/catch fallback logic
3. Create `scripts/export-cms-content.ts` to snapshot content
4. Run export and commit fallback content:

```bash
npx tsx scripts/export-cms-content.ts
git add src/content/fallback/
git commit -m "Add CMS fallback content"
```

**Result:** Site serves cached content if CMS is down.

## Build Pipeline Integration

**CRITICAL: Report F2C progress to the `BuildTask` system** so the admin dashboard shows real-time build status.

### Report Build Progress

At the start of each F2C phase, create a build task entry:

```typescript
// At the start of each phase:
const buildTask = await startBuildPhase({
  websiteProjectId: config.websiteProjectId,
  phase: 7,  // Current phase number
  phaseName: 'Component Generation',
});

// During the phase, log progress:
await addBuildLog(buildTask.id, 'info', 'Built 12/24 components');
await addBuildLog(buildTask.id, 'info', 'Registered 12 ComponentDefinitions in DB');

// At the end of the phase:
await completeBuildPhase(buildTask.id);

// If the phase fails:
await failBuildPhase(buildTask.id, 'Error: Component XYZ failed to compile');
```

### GraphQL Operations

```graphql
mutation StartBuildPhase($input: StartBuildPhaseInput!) {
  startBuildPhase(input: $input) {
    id
    phase
    phaseName
    status
    startedAt
  }
}

mutation CompleteBuildPhase($taskId: ID!) {
  completeBuildPhase(taskId: $taskId) {
    id
    status
    completedAt
  }
}

mutation FailBuildPhase($taskId: ID!, $error: String!) {
  failBuildPhase(taskId: $taskId, errorMessage: $error) {
    id
    status
    errorMessage
  }
}

mutation AddBuildLog($input: AddBuildLogInput!) {
  addBuildLog(input: $input) {
    id
    level
    message
    timestamp
  }
}
```

### Update WebsiteProject After Deployment

After successful deployment, update the `WebsiteProject` with deployment info:

```graphql
mutation UpdateWebsiteProject($id: ID!, $input: UpdateWebsiteProjectInput!) {
  updateWebsiteProject(id: $id, input: $input) {
    id
    status
    vercelProjectId
    productionUrl
    githubRepoUrl
    figmaFileKey
  }
}
```

```typescript
// After successful deployment:
await updateWebsiteProject(config.websiteProjectId, {
  status: 'LIVE',
  vercelProjectId: vercelProject.id,       // From Vercel CLI output
  productionUrl: deployedUrl,              // e.g., 'https://nomad-wynwood.com'
  githubRepoUrl: gitRemoteUrl,            // e.g., 'https://github.com/org/repo'
  figmaFileKey: config.figmaFileKey,       // From Phase 1
});
```

### Phase-Level Build Reporting

| F2C Phase | Build Task Phase Name |
|-----------|----------------------|
| Phase 1 | Pre-flight & Analysis |
| Phase 2 | Project Initialization |
| Phase 3 | Design Token Extraction |
| Phase 4 | Asset Extraction |
| Phase 5 | CMS Content Types |
| Phase 6 | Global Content |
| Phase 7 | Component Generation |
| Phase 8 | Page Content |
| Phase 9 | Page Assembly |
| Phase 10 | Content Audit |
| Phase 11 | Editor Audit |
| Phase 12 | QA |
| Phase 13 | Production Audit |
| Phase 14 | Deployment |

---

## Post-Deployment

### Lighthouse Audit

```bash
npx lighthouse https://your-deployed-url.com --output html --output-path ./lighthouse-report.html
```

Targets:
- Performance: > 90
- Accessibility: > 90
- Best Practices: > 90
- SEO: > 90

### Verify Production

- [ ] All pages load correctly
- [ ] CMS content displays
- [ ] Images load from CDN
- [ ] No console errors

## Git Commit

```bash
git commit -m "feat: deployed to [Platform]"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 14: Deployment complete!" with title "Claude Code" sound name "Ping"'
```

---

## BUILD COMPLETE

The website is now:
- ✓ Built from Figma designs
- ✓ All content from CMS
- ✓ Zero hardcoded strings
- ✓ Inline editor ready
- ✓ Production deployed
