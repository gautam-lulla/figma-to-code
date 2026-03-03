# Phase 11: Inline Editor Data Attributes Audit

**CRITICAL:** Verify ALL editable elements have CMS data attributes AND those attributes point to valid CMS data.

## Automated Check

```bash
# Count elements with data attributes
grep -r "data-cms-entry" src/components/ --include="*.tsx" | wc -l
grep -r "data-cms-field" src/components/ --include="*.tsx" | wc -l
```

## Validation Rules

| Element Type | Required Attributes |
|--------------|---------------------|
| Text (h1, h2, p, span) | `data-cms-entry`, `data-cms-field` |
| Button/Link text | `data-cms-entry`, `data-cms-field` |
| Image | `data-cms-entry`, `data-cms-field`, `data-cms-type="image"` |
| Rich text | `data-cms-entry`, `data-cms-field`, `data-cms-type="richtext"` |
| Array container | `data-cms-entry`, `data-cms-field`, `data-cms-type="array"` |
| Array items | `data-cms-field="arrayName[index].property"` |
| Form placeholder | `data-cms-readonly="true"` |

---

## CRITICAL: Attributes Correctness Validation

**This is the most important part of the audit.** Incorrect attributes will cause the inline editor to fail silently.

### Common Mistakes to Check

| Mistake | Example | Why It Fails |
|---------|---------|--------------|
| Wrong entry slug | `data-cms-entry="menu"` when data is in `menu-mains` | Entry doesn't contain the field |
| Virtual path | `data-cms-field="categories[0].items[0].name"` | Path built from multiple entries |
| Missing root field | `data-cms-field="hero.title"` but entry has `heroTitle` | Field doesn't exist at that path |
| Nested data assumption | `data-cms-field="data.title"` | Double nesting breaks path resolution |

### Validation Process

For each component that renders CMS content:

1. **Identify the content source:**
   - Where does this component get its data? (props from parent, direct fetch)
   - What entry slug was used to fetch that data?

2. **Verify entry slug matches:**
   ```tsx
   // If data comes from getPageContent('homepage')
   // Then data-cms-entry MUST be "homepage"

   // If rendering items from multiple entries (like menu categories)
   // Each item MUST have its own entry slug
   <MenuCategory entry={category.slug} />  // ✅ Correct
   <MenuCategory entry="menu" />           // ❌ Wrong - all categories use same entry
   ```

3. **Verify field path exists in that entry:**
   ```tsx
   // If entry data is: { title: "...", heroImage: "..." }
   data-cms-field="title"       // ✅ Exists
   data-cms-field="hero.title"  // ❌ Wrong path - no "hero" object

   // If entry data has items array: { items: [{ name, price }] }
   data-cms-field="items[0].name"   // ✅ Correct
   data-cms-field="menuItems[0].name" // ❌ Wrong - field is "items" not "menuItems"
   ```

4. **Check array rendering:**
   ```tsx
   // When mapping over arrays, the entry must be the one containing the array
   {categories.map((cat, idx) => (
     // If categories come from different entries, pass the entry slug
     <Item entry={cat.slug} field={`items[${idx}].name`} />

     // NOT a computed path through a parent entry
     <Item entry="menu" field={`categories[${idx}].items[0].name`} /> // ❌
   ))}
   ```

### Quick Test

Open the website in edit mode and click on each editable element. In the browser console look for:
- `[CMS Editor] currentValue: undefined` → **Entry/field mismatch**
- `[SCHEMA DEBUG] entry.data keys: ...` → Check if expected field exists

---

## Manual Audit

For EVERY component rendering CMS content:

### Text Elements
- [ ] Has `data-cms-entry` with correct entry slug
- [ ] Has `data-cms-field` with correct dot-notation path
- [ ] Field path matches CMS data structure

### Images
- [ ] Has `data-cms-entry` and `data-cms-field`
- [ ] Has `data-cms-type="image"`

### Arrays
- [ ] Parent has `data-cms-type="array"`
- [ ] Each item uses indexed path: `field[0].property`

### Page-to-Component Props (CRITICAL)
- [ ] Page passes **ALL fields** from content type to component - not just visible ones
- [ ] Check each section component receives every field defined in its content type
- [ ] Missing props = broken page duplication and hidden inline editor fields

## Report Format

```
INLINE EDITOR DATA ATTRIBUTES AUDIT
===================================

Navigation.tsx:
  ✓ Logo image - data-cms-entry="global-settings" data-cms-type="image"
  ✓ Menu links - indexed paths for each item
  ✓ CTA button - correct attributes

Footer.tsx:
  ✓ All elements have correct attributes

... (all components)

SUMMARY:
- Total editable elements: 147
- Elements with correct attributes: 147
- Missing/incorrect: 0

✓ All editable elements have correct CMS data attributes.
```

## If Elements Missing Attributes

1. List each with file, line, and what's missing
2. Add correct attributes
3. Re-run audit
4. **Do NOT proceed until all elements correct**

---

## Troubleshooting Attribute Issues

### Issue: "currentValue: undefined" in console

**Cause:** The entry doesn't contain the field at the specified path.

**Debug steps:**
1. Check the server logs for `[SCHEMA DEBUG] entry.data keys:` to see actual fields
2. Verify the `data-cms-entry` slug matches the entry containing the data
3. Verify the field path exists in that entry's data structure

### Issue: Data from multiple entries displayed together

**Solution:** Each item must have its own entry slug.

```tsx
// When rendering items from different entries
{categories.map((category) => (
  <CategoryCard
    key={category.slug}
    entry={category.slug}      // ← Pass the actual entry slug
    name={category.name}
    items={category.items}
  />
))}
```

**In the component:**
```tsx
function CategoryCard({ entry, name, items }) {
  return (
    <div>
      <h2 data-cms-entry={entry} data-cms-field="name">{name}</h2>
      {items.map((item, idx) => (
        <span
          data-cms-entry={entry}
          data-cms-field={`items[${idx}].name`}
        >
          {item.name}
        </span>
      ))}
    </div>
  );
}
```

### Issue: Content type uses references to other entries

**Solution:** The data fetching layer must include entry slugs, then components must use the correct entry for each piece of data.

```tsx
// In lib/content.ts - include slug when fetching
export async function getAllCategories() {
  const items = await fetchEntries('category');
  return items.map(item => ({
    slug: item.slug,  // ← Include the entry slug
    ...item.data
  }));
}
```

---

## Git Commit

```bash
git commit -m "chore: inline editor data attributes audit passed"
```

## Checkpoint

```bash
osascript -e 'display notification "Phase 11: Editor audit passed" with title "Claude Code" sound name "Ping"'
```

**Next:** Proceed to [PHASE-12-QA.md](PHASE-12-QA.md)
