# Changelog

All notable changes to JSON to Zod Pro are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.0] — 2025-04-28

### Fixed

- **Union explosion on object arrays (`friends`-style).**  
  The engine previously treated each array item in isolation, generating a `z.union([...])` with one branch per structural variation found. For a 10-item array with minor field differences this could produce 7–10 redundant union branches. `resolveArray()` now performs a **CMT deep-merge** (Common Minimum Type) across all items before emitting any schema: fields absent in some items become `.optional()`, fields null in some become `.nullable()`, and the result is always a single `z.object({})` regardless of how many items are in the input.

- **`discriminatedUnion` on structurally-identical branches.**  
  When all discriminated branches had the same key set and differed only in the discriminant value, the engine emitted N redundant `z.discriminatedUnion` branches instead of a single object with `z.enum([...])`. Now: identical-structure branches (regardless of count) collapse to a single object where the discriminant field is `z.enum(["v1", "v2", ...])`. Branches with structurally different keys still emit a proper `z.discriminatedUnion`.

- **`discriminatedUnion` applied to empty-string discriminant values.**  
  Tags like `""` are semantically useless as discriminants. Empty-string literal tags now disqualify the candidate key from being used as a discriminant entirely.

- **Metadata-style `z.union` for near-identical objects differing only in nullability.**  
  `{lastLogin: null, verified: true}` and `{lastLogin: "2024-...", verified: false}` previously produced `z.union([z.object({lastLogin: z.string().nullable()}), z.object({lastLogin: z.string()})])`. They now collapse to a single `z.object({lastLogin: z.string().nullable(), verified: z.boolean()})`.

- **`.optional()` contamination inside `discriminatedUnion` branches.**  
  Fields absent from one branch (e.g. `data` in the `error` branch) were incorrectly marked `.optional()` in the branch where they are required (e.g. the `success` branch). `buildObjectWithLiteral` now derives optionality exclusively from within-branch absence, never from the global cross-branch `optMap`. A characteristic field of a branch is required inside that branch.

- **`z.array(z.unknown())` for fields that mix empty arrays with populated arrays.**  
  When a field like `friends` was `[]` in 2 out of 13 items and `[{...}]` in 11 others, the engine returned `z.array(z.unknown())` because empty arrays suppressed all type information. Three-layer fix: `buildObjectFromItems` now filters empty sub-arrays before flattening; `resolveArray` (allSubArrays path) filters empty items before resolving; the object threshold was lowered from `items.length >= 2` to `>= 1`.

- **`z.array(z.unknown())` for recursive fields at base-case depth.**  
  In a tree structure (`children: []` at the leaf level), the `children` field was inferred as `z.array(z.unknown())` because no populated samples existed at that depth. When `ctx.recursiveFps.size > 0`, empty array literals are now emitted as `z.array(z.lazy(() => XSchema))` instead.

- **`function detectPattern` declaration missing.**  
  A `str_replace` in a previous refactor attached `isBigIntLike` immediately before the body of `detectPattern` without its function keyword, leaving the body as orphan top-level statements. Both functions are now correctly declared.

- **BigInt false positive for strings with leading zeros.**  
  `"0001234567890123456"` (account numbers, EANs, padded codes) was incorrectly inferred as `z.coerce.bigint()`. Added guard: any digit-only string whose first character is `'0'` is never treated as a bigint — it is always a code/ID string.

- **`generateTsInterface` returning empty string for root arrays.**  
  When the root JSON was an array, `generateTsInterface(parsed, ...)` received an array and returned `''`, producing a malformed recursive output block. The function now extracts the first representative item from the root array and builds the interface from it. The `z.ZodType<>` annotation is `z.ZodType<TypeName[]>` for root arrays and `z.ZodType<TypeName>` for root objects.

- **Recursive detection false positive on sibling arrays.**  
  Arrays of objects with identical shapes (e.g. paginated API responses) were incorrectly flagged as recursive and emitted `z.lazy()`. The detection now uses a DFS ancestor-stack algorithm: a shape is recursive only when its fingerprint appears in its own ancestor chain, not merely elsewhere in the tree. Sibling objects — items in the same array — are processed independently and never contaminate each other's ancestor context.

### Added

- **BigInt inference for JSON numbers at/above `Number.MAX_SAFE_INTEGER`.**  
  JSON numbers that equal or exceed `9007199254740991` have already lost precision during `JSON.parse`. These now emit `z.coerce.bigint()` with a comment explaining the precision boundary. String values representing integers above `MAX_SAFE_INTEGER` without leading zeros also trigger `z.coerce.bigint()`. Semantic key names (`snowflake`, `discordId`, `telegramId`, `int64`, `uint64`…) lower the threshold to 10+ digit strings.

- **`coerce strings` flag.**  
  New opt-in checkbox. When enabled, numeric strings without leading zeros that are not pattern-matched (UUID, email, URL…) are emitted as `z.coerce.number()` or `z.coerce.number().int()` instead of `z.string()`. Useful for APIs that serialize numbers as strings. Disabled by default — enabling it on free-text fields would be destructive.

- **Datetime inference from field name + value validation (double gate).**  
  Previous versions inferred `z.string().datetime()` from the field name alone, producing false positives on fields like `registered_address` containing `"Springfield IL"`. Now both conditions must hold: the field name must match the date-key heuristic (`*At`, `*Date`, `registered`, `lastLogin`, etc.) AND the value must pass `isDateLikeValue()` — a validator that requires a 4-digit year or `dd/mm` pattern and optionally confirms with `Date.parse`. Name-only matches fall back to `z.string()`.

- **`z.lazy()` for genuinely recursive structures (tree nodes, linked lists).**  
  The recursive fingerprint detection was rewritten (`findRecursiveFingerprints` v2) with two detection modes:
  - **Type A — ancestor-chain recursion** (DFS path): an object whose key-set has ≥ 80% Jaccard overlap with any open ancestor on the current DFS path.
  - **Type B — array-item recursion** (sibling scan): an array field whose items have ≥ 80% key-overlap with the object that contains the array. This catches `children: [{children: [...], name: "..."}]` without requiring a deep traversal.
  Additionally: `MAX_DETECT_DEPTH = 20` caps recursion depth to prevent stack overflow on pathological inputs; a `visited` Set memoizes already-scanned fingerprints to skip re-processing structurally identical subtrees.

- **Explicit TypeScript interface for recursive schemas.**  
  `z.infer<>` cannot resolve circular types. When `z.lazy()` is emitted, the tool now generates a hand-written `interface TypeName { ... }` and annotates the schema as `z.ZodType<TypeName>` (or `z.ZodType<TypeName[]>` for root arrays) — matching the pattern from Zod's official documentation for recursive schemas.  
  Example output for `{name, children: [...same shape...]}`:
  ```ts
  interface TreeNode {
    name: string;
    children: TreeNode[];
  }

  export const treeNodeSchema: z.ZodType<TreeNode> = z.object({
    name: z.string(),
    children: z.array(z.lazy(() => treeNodeSchema)),
  });
  ```

- **`friends array ✦` preset.**  
  New sample demonstrating the CMT deep-merge fix: 3 person records each with a nested `friends` array and a nullable `metadata.lastLogin` field.

### Changed

- **`resolveArray` root array routing.**  
  `generateSchema` previously bypassed `resolveArray` for root arrays, routing directly to `buildObjectFromItems`. This skipped the discriminant check, causing `[{type:"success", data:"ok"}, {type:"error", error_code:500}]` to produce a flat optional-field object instead of a `discriminatedUnion`. All root arrays now route through `resolveArray`, which falls back to `buildObjectFromItems` for homogeneous arrays.

- **`discriminatedUnion` branch collapse threshold.**  
  Previously, identical-structure branches only collapsed to `z.enum()` when there were more than 2 tags (`tags.length > 2`). With exactly 2 tags, a discriminatedUnion of two structurally identical branches was emitted — wasteful and semantically redundant. Now collapses for any number of identical-structure tags ≥ 2.

- **`widenTypes` — single-pass, non-recursive.**  
  Previous implementation was recursive (up to 3 levels of re-invocation) and allocated intermediate arrays at each step. Rewritten as a single-pass `Set` mutation: O(n) with no recursion and no intermediate allocations.

- **`keyOverlap` — iterate smaller set.**  
  Changed from `[...setA].filter(k => setB.has(k))` (creates an intermediate array) to a direct loop over the smaller of the two sets. O(min(|A|, |B|)) instead of O(|A|).

- **`mergeObjectShapes` — single-pass key count.**  
  Previously ran two separate `.filter()` passes on the full key union (each calling `objs.every()`), giving O(k·n) × 2. Replaced with a single `Map`-based count pass: O(k·n) once.

- **`avgOverlap` — cache `Object.keys()` per object.**  
  The pairwise overlap loop called `Object.keys(objs[i])` O(n²) times. Keys are now computed once into a `keyArrays` cache before the loop.

- **`buildObjectFactored` — Set lookup for variant keys.**  
  `variantKeys.includes(rawKey)` inside `.map()` was O(n) per field → O(n²) total. Replaced with `varKeySet = new Set(variantKeys)` + `.has()` → O(1) per lookup.

- **`findDiscriminant` — cache `Object.keys()` per object.**  
  Keys are now pre-computed into `keysets` before the candidate filtering loop, avoiding repeated `Object.keys()` calls inside the filter predicate.

- **`collectEnums` — null-proto object + `WeakSet` guard.**  
  Changed `hints = {}` to `Object.create(null)` to prevent collisions on keys like `__proto__` or `constructor`. Added a `WeakSet` to guard against circular references in deeply nested JSON.

- **`getDepth` — iterative max instead of `Math.max(...spread)`.**  
  `Math.max(d, ...array.map(getDepth))` spread the entire child array onto the call stack — causing stack overflows on JSON with hundreds of sibling keys. Replaced with an explicit loop and a local `max` variable.

- **`z.enum([value])` for single-value bounded fields.**  
  Fields whose names are semantically enum-like (`status`, `role`, `type`, `kind`, `env`, `plan`, `tier`, `phase`…) now emit `z.enum(["observed_value"])` even from a single sample. This gives TypeScript the correct literal type and signals intent to the developer, who can extend the enum as the API evolves. Previously these fields produced `z.string()` with a JSDoc comment.

- **Token-based syntax highlighter.**  
  Replaced the sequential regex-replace highlighter with a token-masking approach. Each matched token is replaced with a numeric placeholder (`\x00N\x00`) during processing and restored at the end. This prevents any pass from accidentally re-processing already-highlighted text, eliminating the `z-export">` artifact.

---

## [0.0.6] — 2025-04-21

### Added
- **`inferNullableBase(key)`** — null fields no longer emit `z.null()` (which only accepts the literal `null`). The engine infers the most likely base type from the field name via pattern-matched heuristics:
  - `*At`, `*Date`, `created*`, `deleted*` → `z.string().datetime({ offset: true })`
  - `*Url`, `*Avatar`, `*Website` → `z.string().url()`
  - `*Email` → `z.string().email()`
  - `*Count`, `*Price`, `*Amount` → `z.number()`
  - `is*`, `has*`, `*Enabled`, `*Active` → `z.boolean()`
  - `bio`, `description`, `name`, `content` → `z.string()`
  - Default → `z.string()`  
  All paths append `.nullable()` (and `.optional()` when the field is absent in some samples).
- **`ENUM_HINT_KEYS`** — semantic hint for single-value bounded fields (moved to `z.enum()` promotion in v7).
- **DISC_PREFERRED cardinality bypass** — discriminant keys in the preferred list (`type`, `kind`, `action`…) bypass the cardinality ratio guard, allowing `discriminatedUnion` to work correctly when there are exactly as many unique values as there are array items.

---

## [0.0.5] — 2025-04-21

### Added
- **`widenTypes()`** — before forming a union from collected types, the engine applies the least-upper-bound: `z.number().int() + z.number()` → `z.number()`; `z.string().email() + z.string()` → `z.string()`; two different refined strings → `z.string()`; identical refinements → kept as-is.
- Applied in `buildObjectFromItems`, `resolveArray` primitive path, and `resolveArray` mixed path.

### Fixed
- Key quoting now only applies to keys that fail the JS identifier regex (`/^[a-zA-Z_$][a-zA-Z0-9_$]*/`). Reserved words (`default`, `import`, `var`, `parse`, `safeParse`) are valid as object property names and no longer quoted unnecessarily.
- Empty objects respect `strict` mode: `z.object({}).strict()` when strict is enabled.

---

## [0.0.4] — 2025-04-21

### Added
- **`buildObjectFromItems()`** — root arrays of objects now perform per-key type union across all items. Each key collects all observed values, resolves each to a Zod type, widens, and forms a union if types differ.
- **Null as union member** — in heterogeneous arrays, `null` values produce `z.null()` as an independent union member rather than adding `.nullable()` to every other member.
- UUID regex relaxed to accept all variant bytes (not only `[89ab]`), covering non-RFC UUIDs from legacy systems.
- Email regex extended to accept internal TLDs (`.internal`, `.local`).

---

## [0.0.3] — 2025-04-21

### Added
- **`inferNullableBase` predecessor** — `buildObjectFactored` gained null-semantic awareness.
- **DISC_PREFERRED** ordered list for discriminant key preference.
- **`buildObjectWithLiteral`** per-branch optional detection (keys absent in some group members → `.optional()`).

### Fixed
- `findDiscriminant` validity guard: identifier-like keys (`username`, `email`, `bio`…) can never be discriminants; cardinality ratio and max-unique caps prevent free-form string fields from being chosen.

---

## [0.0.2] — 2025-04-21

### Added
- DFS-based recursive schema detection with `z.lazy()` and companion TypeScript interface (superseded by path-based DFS in v7).
- Discriminated union detection and generation.
- Key factoring for similar-shape arrays (below the discriminant threshold).
- `widenTypes` precursor in `buildObjectFactored`.

---

## [0.0.1] — 2025-04-21

### Added
- Initial release.
- Pattern detection: UUID, email, URL, datetime, date, IPv4, semver, hex color, CUID, CUID2, JWT.
- Enum auto-detection with cardinality guard.
- Multi-sample merging for optional field detection.
- Output modes: schema, type, both, with validation functions.
- Naming convention: preserve, camelCase, snake_case.
- JSDoc comment generation.
- Strict mode.
- Syntax highlighting.
- Preset samples.
