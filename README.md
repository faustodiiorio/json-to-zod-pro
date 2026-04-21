# JSON to Zod Pro

**Intelligent TypeScript schema generator** — paste any JSON, get a production-ready [Zod](https://zod.dev) schema instantly.

→ **[Live demo](https://faustodiiorio.github.io/json-to-zod-pro)**

---

## Why this tool?

Most JSON→Zod converters do a naive type-mapping pass. This tool does significantly more:

| Feature | Basic tools | JSON to Zod Pro |
|---|---|---|
| Object schema | ✓ | ✓ |
| Nested objects | ✓ | ✓ |
| Pattern detection (UUID, email, URL…) | ✗ | ✓ |
| Discriminated unions | ✗ | ✓ |
| Recursive schemas (`z.lazy()`) | ✗ | ✓ |
| Multi-sample optional field detection | ✗ | ✓ |
| Type widening (int+float → number) | ✗ | ✓ |
| Null field semantic inference | ✗ | ✓ |
| Enum auto-detection | ✗ | ✓ |
| Key quoting for non-identifier keys | ✗ | ✓ |
| Tuple detection | ✗ | ✓ |

---

## Usage

### Online
Open `index.html` in any browser — no server, no dependencies, no install.

### Local
```bash
git clone https://github.com/faustodiiorio/json-to-zod-pro
open json-to-zod-pro/index.html
```

### Self-host
Upload `index.html` to any static host (GitHub Pages, Netlify, Vercel, Cloudflare Pages).

---

## Features

### Pattern detection
The engine recognises 11 string formats and maps them to the appropriate Zod validator:

| Pattern | Example value | Zod output |
|---|---|---|
| UUID | `550e8400-e29b-41d4-a716-446655440000` | `z.string().uuid()` |
| Email | `user@example.com` | `z.string().email()` |
| URL | `https://example.com` | `z.string().url()` |
| ISO datetime | `2024-01-15T10:30:00Z` | `z.string().datetime({ offset: true })` |
| ISO date | `2024-01-15` | `z.string().date()` |
| IPv4 | `192.168.1.1` | `z.string().ip({ version: "v4" })` |
| Semver | `1.2.3-beta` | `z.string().regex(...)` |
| Hex color | `#6366f1` | `z.string().regex(...)` |
| CUID | `cjld2cyuq0000...` | `z.string().cuid()` |
| CUID2 | `tz4a98xxat96iws9...` | `z.string().cuid2()` |
| JWT | `eyJ...` | `z.string().jwt()` |

### Enum auto-detection
With multiple JSON samples, string fields with low cardinality (≤8 unique values, ratio ≤0.6) are automatically promoted to `z.enum([...])`. Semantic fields (`status`, `role`, `type`, `kind`…) get `z.enum(["value"])` even from a single sample.

### Discriminated unions
Arrays of objects sharing a string discriminant key (`type`, `kind`, `action`…) are automatically modelled as `z.discriminatedUnion()` — faster validation and better error messages than plain `z.union()`. Per-branch optional fields are detected and correctly marked `.optional()`.

### Multi-sample merging
Click **+ merge sample** to provide additional JSON objects. The engine cross-references all samples to:
- Mark fields missing in some samples as `.optional()`
- Build accurate enums from observed values
- Widen types correctly (`id` seen as integer in one sample and string in another → `z.union([z.number().int(), z.string()])`)

### Null field inference
A null value in a single sample does not produce the useless `z.null()`. Instead, the engine infers the most likely base type from the field name:

```
bio: null       → z.string().nullable()
deletedAt: null → z.string().datetime({ offset: true }).nullable()
avatarUrl: null → z.string().url().nullable()
itemCount: null → z.number().nullable()
isActive: null  → z.boolean().nullable()
```

### Recursive schemas
Genuinely self-referential structures (tree nodes, linked lists) are detected via a DFS ancestor-stack algorithm and emitted with `z.lazy()` + a companion TypeScript interface. Repeated sibling objects (e.g., an array of items with the same shape) are correctly **not** flagged as recursive.

### Type widening
When collecting types across multiple values, the engine applies the least-upper-bound before forming a union:

```
z.number().int() + z.number()      → z.number()
z.string().email() + z.string()    → z.string()
z.string().email() + z.string().url() → z.string()
z.string().email() × N             → z.string().email()   (same refinement kept)
```

---

## Options

| Option | Description |
|---|---|
| **Schema name** | Name used for the exported const and type |
| **Output mode** | Schema only / type only / both / with validate functions |
| **Naming** | Preserve / camelCase / snake_case key transformation |
| **optional** | Mark fields absent in some samples as `.optional()` |
| **nullable** | Infer nullable types from null values |
| **jsdoc** | Emit JSDoc comments with semantic hints |
| **strict** | Add `.strict()` to all objects (rejects unknown keys) |
| **enum detect** | Auto-detect enums from repeated string values |
| **pattern detect** | Map string values to Zod format validators |
| **import** | Prepend `import { z } from "zod"` |

---

## Output modes

### Schema + type (default)
```ts
import { z } from "zod";

export const userSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
  createdAt: z.string().datetime({ offset: true }),
});

export type User = z.infer<typeof userSchema>;
```

### With validation functions
```ts
export function parseUser(data: unknown): User {
  return userSchema.parse(data);
}

export function safeParseUser(data: unknown) {
  return userSchema.safeParse(data);
}
```

---

## Keyboard shortcuts

| Key | Action |
|---|---|
| `Ctrl/Cmd + Enter` | Generate schema |
| `Ctrl/Cmd + Shift + C` | Copy schema |
| `Ctrl/Cmd + Shift + F` | Format JSON |

---

## Diffusion strategy

### Embed in docs
Since it's a single HTML file, you can embed it in any documentation site with an `<iframe>`:
```html
<iframe src="https://faustodiiorio.github.io/json-to-zod-pro" 
        width="100%" height="600" frameborder="0"></iframe>
```

---

## Contributing

Bug reports and PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

For bugs, please include:
- Input JSON
- Current output
- Expected output

---

## License

MIT — see [LICENSE](LICENSE).
