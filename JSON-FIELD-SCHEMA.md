# JSON Field Schema Validation Guide

This document defines **generic JSON Schema patterns** for validating JSON fields in the CMS. These patterns should be **adapted to your specific project needs**.

---

## Overview

The CMS supports optional JSON Schema validation (draft-07) for JSON fields. When a schema is configured:
- The admin UI shows a **structured form** instead of a raw JSON editor
- Validation errors are shown on save
- Content editors don't need to understand JSON syntax

### Key Features

- **Backward Compatible**: No schema = no validation (existing fields unaffected)
- **Structured Editing**: Arrays render as add/remove/reorder item cards
- **User-Friendly Errors**: Technical validation errors are translated to readable messages
- **Standard Format**: Uses JSON Schema draft-07, a widely adopted specification

---

## Core Pattern: Array of Objects

The most common use case is an **array of structured items**. This pattern works for:
- Navigation links
- Feature cards
- Team members
- Product listings
- FAQ items
- Gallery images
- Any repeatable content

### Basic Template

```json
{
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
```

### TypeScript Interface Pattern

```typescript
interface Item {
  title: string;        // Required
  description?: string;
  url?: string;
}

type ItemsArray = Item[];
```

---

## Common Schema Patterns

### 1. Links Array

For navigation, footer links, social links, etc.

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "label": { "type": "string", "minLength": 1 },
      "url": { "type": "string" },
      "isExternal": { "type": "boolean" }
    },
    "required": ["label", "url"]
  }
}
```

**Use for:** Navigation menus, footer columns, resource links, social profiles

---

### 2. Cards/Features Array

For feature highlights, benefits, services, etc.

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "title": { "type": "string", "minLength": 1 },
      "description": { "type": "string" },
      "icon": { "type": "string" },
      "imageUrl": { "type": "string" }
    },
    "required": ["title"]
  },
  "maxItems": 6
}
```

**Use for:** Feature grids, benefit cards, service offerings, team members

---

### 3. Images/Gallery Array

For image galleries, portfolios, media collections.

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "src": { "type": "string", "minLength": 1 },
      "alt": { "type": "string", "minLength": 1 },
      "caption": { "type": "string" }
    },
    "required": ["src", "alt"]
  }
}
```

**Use for:** Photo galleries, portfolio grids, product images, testimonial photos

---

### 4. Items with Tags/Categories

For content that needs categorization or filtering.

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "minLength": 1 },
      "description": { "type": "string" },
      "tags": {
        "type": "array",
        "items": { "type": "string" }
      },
      "category": { "type": "string" }
    },
    "required": ["name"]
  }
}
```

**Use for:** Blog posts, products, portfolio items, resources

---

### 5. Items with Enum Constraints

When items should only have specific allowed values.

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "minLength": 1 },
      "status": {
        "type": "string",
        "enum": ["active", "inactive", "pending"]
      },
      "priority": {
        "type": "string",
        "enum": ["low", "medium", "high"]
      }
    },
    "required": ["name", "status"]
  }
}
```

**Use for:** Status indicators, priority levels, type selectors, any fixed-option fields

---

### 6. Nested Structure

For items with sub-items or grouped content.

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "sectionTitle": { "type": "string", "minLength": 1 },
      "items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name": { "type": "string" },
            "value": { "type": "string" }
          },
          "required": ["name"]
        }
      }
    },
    "required": ["sectionTitle"]
  }
}
```

**Use for:** Accordion sections, grouped lists, categorized content, multi-level navigation

---

## Industry-Specific Examples

Adapt these patterns to your project's domain:

### E-commerce: Product Variants
```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "sku": { "type": "string" },
      "size": { "type": "string" },
      "color": { "type": "string" },
      "price": { "type": "number" },
      "inStock": { "type": "boolean" }
    },
    "required": ["sku", "price"]
  }
}
```

### Real Estate: Property Features
```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "feature": { "type": "string" },
      "category": { "type": "string", "enum": ["interior", "exterior", "amenity"] }
    },
    "required": ["feature"]
  }
}
```

### Events: Schedule Items
```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "time": { "type": "string" },
      "title": { "type": "string" },
      "speaker": { "type": "string" },
      "location": { "type": "string" }
    },
    "required": ["time", "title"]
  }
}
```

### Restaurant: Menu Items
```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "minLength": 1 },
      "description": { "type": "string" },
      "price": { "type": "string" }
    },
    "required": ["name"]
  }
}
```

---

## Applying Schemas to Content Types

### Via TypeScript (Recommended)

```typescript
await createOrUpdateContentType('my-content-type', 'My Content Type', [
  {
    slug: 'items',
    name: 'Items',
    type: 'json',
    validation: {
      schema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            title: { type: 'string', minLength: 1 },
            description: { type: 'string' }
          },
          required: ['title']
        }
      }
    }
  }
]);
```

### Via GraphQL

```graphql
mutation CreateFieldDefinition($input: CreateFieldDefinitionInput!) {
  createFieldDefinition(input: $input) {
    id
    slug
    validation
  }
}
```

Variables:
```json
{
  "input": {
    "contentTypeId": "ct_xxx",
    "slug": "items",
    "name": "Items",
    "type": "JSON",
    "validation": {
      "schema": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "title": { "type": "string", "minLength": 1 }
          },
          "required": ["title"]
        }
      }
    }
  }
}
```

---

## Schema Constraints Reference

| Constraint | Applies To | Example |
|------------|-----------|---------|
| `type` | All | `"type": "string"` |
| `required` | Objects | `"required": ["name", "url"]` |
| `minLength` | Strings | `"minLength": 1` |
| `maxLength` | Strings | `"maxLength": 100` |
| `pattern` | Strings | `"pattern": "^[a-z]+$"` |
| `enum` | Strings | `"enum": ["a", "b", "c"]` |
| `minimum` | Numbers | `"minimum": 0` |
| `maximum` | Numbers | `"maximum": 100` |
| `minItems` | Arrays | `"minItems": 1` |
| `maxItems` | Arrays | `"maxItems": 10` |

---

## Best Practices

### 1. Start Simple
Begin with basic type validation, add constraints as needed:
```json
{ "type": "array", "items": { "type": "object" } }
```

### 2. Only Require What's Essential
Don't over-constrain - only mark truly required fields:
```json
{ "required": ["title"] }  // Not ["title", "description", "image", ...]
```

### 3. Use Descriptive Property Names
Property names become form labels in the admin UI:
```json
// Good: "companyName", "phoneNumber", "streetAddress"
// Bad: "cn", "ph", "addr"
```

### 4. Consider the Editor Experience
The admin shows form fields based on your schema. Design for content editors, not developers.

---

## Security Limits

The validator enforces limits to prevent DoS attacks:

| Limit | Value | Purpose |
|-------|-------|---------|
| Schema size | 50,000 chars | Prevents memory exhaustion |
| Schema depth | 10 levels | Prevents stack overflow |

---

## Version History

**v1.0.0** (January 2026)
- Initial JSON Schema validation support
- Generic patterns for common use cases
- Structured form editing in admin UI
