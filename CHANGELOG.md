# Changelog

All notable changes to JSON to Zod Pro are documented here.  
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.0] — 2025-04-21

### Fixed
- **Recursive detection false positive on sibling arrays.**  
  Arrays of objects with identical shapes (e.g. paginated API responses) were incorrectly flagged as recursive and emitted `z.lazy()`. The detection now uses a DFS ancestor-stack algorithm: a shape is recursive only when its fingerprint appears in its own ancestor chain, not merely elsewhere in the tree. Sibling objects — items in the same array — are processed independently and never contaminate each other's ancestor context.

### Changed
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
