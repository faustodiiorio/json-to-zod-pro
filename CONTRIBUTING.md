# Contributing

Thank you for your interest in improving JSON to Zod Pro.

## Reporting bugs

Please open an issue and include:

1. **Input JSON** — the exact JSON that triggers the problem
2. **Current output** — what the tool currently generates
3. **Expected output** — what it should generate, with a brief explanation
4. **Options used** — which checkboxes/selectors were active

The more precise the reproduction case, the faster it can be fixed.

## Suggesting features

Open an issue with the label `enhancement`. Describe:
- The use case (what kind of JSON / schema pattern)
- The expected Zod output
- Why the current output is insufficient

## Pull requests

1. Fork and clone the repo
2. Make changes to `index.html` (the engine is entirely self-contained)
3. Test against the cases in `docs/test-cases.md`
4. Open a PR with a clear description of what changed and why

### Engine architecture

The entire tool is a single `index.html` file. The JavaScript engine lives in the `<script>` block and is organized into clearly labelled sections:

```
PATTERN DETECTION       — regex-based string format recognition
KEY QUOTING             — identifier validation for object property names
RECURSIVE DETECTION     — DFS ancestor-stack algorithm
ENUM CARDINALITY GUARD  — isLikelyEnum() with ratio + identifier checks
DISCRIMINANT DETECTION  — findDiscriminant() with validity guards
TYPE WIDENING           — widenTypes() least-upper-bound
NULL INFERENCE          — inferNullableBase() field-name heuristics
CORE TYPE RESOLVER      — resolveZodType() dispatch
ARRAY RESOLVER          — tuple / primitives / objects / mixed paths
OBJECT BUILDERS         — buildObject / buildObjectFromItems / factored / literal
AUTO COMMENTS           — JSDoc generation
MAIN GENERATOR          — generateSchema() orchestration
SYNTAX HIGHLIGHTER      — token-based highlight()
RENDER / UI             — DOM manipulation, tab switching, copy, etc.
```

### Adding a new pattern

1. Add a `const RE_YOURPATTERN` near the top of the pattern section
2. Add an entry to the `PATTERNS` array: `{ name, re, zod }`
3. Ensure it doesn't conflict with earlier patterns (order matters — first match wins)
4. Add a test case to `docs/test-cases.md`

### Adding a new null inference rule

Add a branch to `inferNullableBase(key)` before the default `return 'z.string()'`.
Use `k.toLowerCase()` and regex tests against key name fragments.

## Code style

- Plain ES2020, no build step, no dependencies
- Use `const` / `let`, no `var`
- Descriptive variable names over brevity
- Section headers use the `═══` box-drawing style already present
- Comments explain *why*, not *what*
