# GraphQL Examples

> **IMPORTANT:** Adapt these to match YOUR CMS schema.
> Read `/project-config/schema.graphql` first.

## Authentication

```graphql
mutation Login($email: String!, $password: String!) {
  login(email: $email, password: $password) {
    accessToken
    refreshToken
    user {
      id
      email
    }
  }
}
```

## Content Types

### Create Content Type

```graphql
mutation CreateContentType($input: CreateContentTypeInput!) {
  createContentType(input: $input) {
    id
    slug
    name
    fields {
      slug
      name
      type
    }
  }
}
```

Variables:
```json
{
  "input": {
    "slug": "page-content",
    "name": "Page Content",
    "organizationId": "org_xxx",
    "fields": [
      { "slug": "data", "name": "Page Data", "type": "json", "required": true },
      { "slug": "metaTitle", "name": "Meta Title", "type": "string" }
    ]
  }
}
```

### Create Field with JSON Schema (Structured Editing)

Add a JSON schema to enable structured form editing in the admin UI instead of raw JSON:

```graphql
mutation CreateFieldDefinition($input: CreateFieldDefinitionInput!) {
  createFieldDefinition(input: $input) {
    id
    slug
    name
    type
    validation
  }
}
```

Variables example (adapt schema to your needs):
```json
{
  "input": {
    "contentTypeId": "ct_xxx",
    "slug": "items",
    "name": "Items",
    "type": "JSON",
    "required": true,
    "validation": {
      "schema": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "title": { "type": "string", "minLength": 1 },
            "description": { "type": "string" },
            "url": { "type": "string" }
          },
          "required": ["title"]
        }
      }
    }
  }
}
```

**Result:** Admin UI shows a structured form with:
- Collapsible item cards
- Individual input fields for each property
- Add/remove/reorder buttons
- Toggle to switch between structured and raw JSON modes

**See [JSON-FIELD-SCHEMA.md](./JSON-FIELD-SCHEMA.md) for more schema patterns.**

## Content Entries

### Create Entry

```graphql
mutation CreateContentEntry($input: CreateContentEntryInput!) {
  createContentEntry(input: $input) {
    id
    slug
    data
    isGlobal
  }
}
```

Variables:
```json
{
  "input": {
    "contentTypeId": "ct_xxx",
    "organizationId": "org_xxx",
    "propertyId": "prop_xxx",
    "slug": "my-entry",
    "data": { "title": "My Entry" },
    "isGlobal": false
  }
}
```

> **IMPORTANT:** `propertyId` is **required** — all entries must be scoped to a property. Entries without it will be rejected by the API.
> Set `isGlobal: true` for global entries (navigation, footer, settings).
> Global entries are NOT duplicated when users clone pages in the inline editor.

### Get Entry by Slug

```graphql
query GetContentBySlug($slug: String!, $contentTypeSlug: String!) {
  contentEntryBySlug(slug: $slug, contentTypeSlug: $contentTypeSlug) {
    id
    slug
    data
    contentType {
      id
      slug
    }
  }
}
```

### Update Entry

```graphql
mutation UpdateContentEntry($input: UpdateContentEntryInput!) {
  updateContentEntry(input: $input) {
    id
    slug
    data
    updatedAt
  }
}
```

### List Entries by Type

```graphql
query ListEntries($contentTypeSlug: String!) {
  contentEntries(contentTypeSlug: $contentTypeSlug) {
    id
    slug
    data
  }
}
```

## Apollo Client Setup

```typescript
// src/lib/apollo-client.ts
import { ApolloClient, InMemoryCache, HttpLink, from } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';

const CMS_URL = process.env.NEXT_PUBLIC_CMS_GRAPHQL_URL;

export function getServerClient(token?: string) {
  const httpLink = new HttpLink({ uri: CMS_URL, fetch });

  const authLink = setContext((_, { headers }) => ({
    headers: {
      ...headers,
      ...(token && { authorization: `Bearer ${token}` }),
    },
  }));

  return new ApolloClient({
    link: from([authLink, httpLink]),
    cache: new InMemoryCache(),
    defaultOptions: {
      query: { fetchPolicy: 'no-cache', errorPolicy: 'all' },
    },
  });
}
```

## Content Layer Functions

```typescript
// src/lib/content/index.ts
import { getServerClient } from '../apollo-client';
import { GET_CONTENT_BY_SLUG } from '../queries';

export async function getPageContent(pageSlug: string) {
  const client = getServerClient();
  const { data } = await client.query({
    query: GET_CONTENT_BY_SLUG,
    variables: { slug: pageSlug, contentTypeSlug: 'page-content' },
  });
  return data.contentEntryBySlug;
}

export async function getSiteSettings() {
  const client = getServerClient();
  const { data } = await client.query({
    query: GET_CONTENT_BY_SLUG,
    variables: { slug: 'global-settings', contentTypeSlug: 'site-settings' },
  });
  return data.contentEntryBySlug?.data || {};
}

export async function getNavigation() {
  const client = getServerClient();
  const { data } = await client.query({
    query: GET_CONTENT_BY_SLUG,
    variables: { slug: 'global-navigation', contentTypeSlug: 'site-navigation' },
  });
  return data.contentEntryBySlug?.data || {};
}

export async function getFooterContent() {
  const client = getServerClient();
  const { data } = await client.query({
    query: GET_CONTENT_BY_SLUG,
    variables: { slug: 'global-footer', contentTypeSlug: 'site-footer' },
  });
  return data.contentEntryBySlug?.data || {};
}
```
