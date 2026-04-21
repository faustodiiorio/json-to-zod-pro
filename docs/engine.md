# Engine Architecture

This document describes the internal design of the JSON to Zod Pro inference engine.  
The entire engine is contained in the `<script>` block of `index.html`.

---

## Data flow

```
User input (JSON string)
        │
        ▼
    JSON.parse()
        │
        ▼
  Pre-analysis phase
  ┌─────────────────────────────────────────┐
  │ findRecursiveFingerprints(root)          │  DFS ancestor-stack
  │ collectEnums(samples)                   │  gather string values per key
  │ detectOptionals(samples)                │  cross-sample key presence
  └─────────────────────────────────────────┘
        │  ctx = { recursiveFps, enumHints, optMap, ... }
        ▼
  generateSchema()
        │
        ├─ root is Array?
        │     ├─ all objects? → buildObjectFromItems(items)   [per-key union]
        │     └─ mixed?      → resolveArray(null, items)
        │
        └─ root is Object? → buildObject(root)
               │
               └─ for each key:
                     resolveZodType(key, value, opts, depth, ctx)
                           │
                           ├─ null    → inferNullableBase(key) + .nullable()
                           ├─ bool    → z.boolean()
                           ├─ number  → z.number() or z.number().int()
                           ├─ string  → enum check → pattern check → z.string()
                           ├─ array   → resolveArray(key, items)
                           └─ object  → recursive? z.lazy() : buildObject()
        │
        ▼
  widenTypes() applied wherever a Set of Zod type strings is collected
        │
        ▼
  Output assembly (import, schema const, type alias, validate fns)
        │
        ▼
  highlight(code) — token-based syntax highlighting
        │
        ▼
  innerHTML update
```

---

## Key functions

### `findRecursiveFingerprints(root) → Set<string>`

DFS traversal maintaining an `ancestors` set of fingerprints of all objects currently open on the call stack. An object whose fingerprint is already in `ancestors` when first visited is marked as recursive. Array items are processed as independent siblings — they do not contaminate each other's ancestor context.

**Fingerprint**: sorted key names joined by `|` (e.g., `"child|id|metadata"`).

---

### `resolveZodType(key, value, opts, depth, ctx) → string`

Central dispatch. Returns a Zod type string for a single `(key, value)` pair.

Priority order for string values:
1. Enum check (`isLikelyEnum` or `ENUM_HINT_KEYS` single-value path)
2. Pattern detection (UUID, email, URL, datetime, …)
3. `z.string()`

---

### `resolveArray(key, items, opts, depth, ctx) → string`

Handles arrays. Decision tree:
1. **Tuple**: all items are sub-arrays of the same fixed length → `z.tuple([...])`
2. **Primitives only**: deduplicate types, widen, form union or single type
3. **Objects, discriminant found**: `z.discriminatedUnion(...)`
4. **Objects, avg overlap ≥ 0.65**: key factoring → single object with optional variant keys
5. **Objects, distinct shapes**: `z.union([...])`
6. **Mixed** (objects + primitives + nested arrays + null): per-item type collection, deduplicate, widen, union

---

### `buildObjectFromItems(items, opts, depth, ctx) → string`

Used for root arrays of objects. For each key, collects all observed values across all items, resolves each to a Zod type, applies `widenTypes()`, and forms:
- Single type if all values agree
- `z.union([...])` if types differ
- `.nullable()` if any value was null
- `.optional()` if any item lacked the key

---

### `widenTypes(types: string[]) → string[]`

Applies the least-upper-bound before forming a union:

| Input | Output |
|---|---|
| `[z.number().int(), z.number()]` | `[z.number()]` |
| `[z.string().email(), z.string()]` | `[z.string()]` |
| `[z.string().email(), z.string().url()]` | `[z.string()]` |
| `[z.string().email(), z.string().email()]` | `[z.string().email()]` |

---

### `inferNullableBase(key) → string`

When a field is observed as `null`, this function infers the most likely base type from the field name. Used by `buildObject`, `buildObjectFromItems`, and `buildObjectFactored`.

Returns a bare Zod type string (e.g., `"z.string().datetime({ offset: true })"`) without `.nullable()` — the caller appends that.

---

### `findDiscriminant(items) → string | null`

Evaluates candidate keys to find the best discriminant for a `z.discriminatedUnion`. A valid discriminant must:
- Be present in all items as a string
- Have ≥ 2 distinct values
- Have ≤ `DISC_MAX_UNIQUE` (12) distinct values
- Not match `DISC_IDENTIFIER_KEYS` (username, email, name, bio, …)
- Either be in `DISC_PREFERRED` (bypasses cardinality ratio) or have `unique/total ≤ DISC_CARDINALITY_RATIO` (0.75)

Candidates are sorted by position in `DISC_PREFERRED` so `type` beats `status` beats `role`.

---

### `isLikelyEnum(key, valuesArr) → boolean`

Returns true when observed string values for a key are likely a bounded enum:
- 2 ≤ unique values ≤ 8
- unique/total ≤ 0.6
- Key name does not match `ID_LIKE_KEYS`
- No value longer than 32 characters

---

### `detectPattern(key, value) → Pattern | null`

Tests `value` against the ordered `PATTERNS` array. JWT is tested before URL to prevent misclassification (JWTs contain dots but no slashes).

---

### `highlight(code) → string`

Token-masking approach:
1. HTML-escape (`&`, `<`, `>`)
2. Replace each recognized token with `\x00N\x00` (numeric placeholder)
3. Store the rendered `<span>` in `tokens[N]`
4. After all passes, restore with `code.replace(/\x00(\d+)\x00/g, ...)`

This ensures no pass can re-process an already-highlighted token.

---

## Context object (`ctx`)

Passed through all recursive calls. Accumulates state across the traversal:

```js
{
  enumHints:      { [key]: string[] },   // all observed string values per key
  optMap:         { [key]: true },       // keys absent in ≥1 sample
  recursiveFps:   Set<string>,           // fingerprints flagged as recursive
  lazyName:       string,                // schema name for z.lazy(() => Xschema)
  lazyRefs:       Map<fp, name>,         // which fps use z.lazy
  hasLazy:        boolean,               // true if any z.lazy was emitted
  fieldCount:     number,                // running total for status bar
  depthMax:       number,
  arrayCount:     number,
  patternCount:   number,
  enumCount:      number,
  discUnionCount: number,
}
```

---

## Thresholds and constants

| Constant | Value | Purpose |
|---|---|---|
| `RECURSIVE_THRESHOLD` | — | Removed in v7; replaced by DFS |
| `FACTOR_THRESHOLD` | 0.65 | Min Jaccard overlap for key factoring |
| `DISC_MAX_UNIQUE` | 12 | Max distinct values for a discriminant |
| `DISC_CARDINALITY_RATIO` | 0.75 | Max unique/total ratio for non-preferred discriminants |
