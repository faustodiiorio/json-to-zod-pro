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
| BigInt detection | ✗ | ✓ |
| Coerce numeric strings | ✗ | ✓ |
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

Datetime inference also activates for non-ISO date strings when the **field name** is date-like (`registeredAt`, `lastLogin`, `createdDate`, etc.) **and** the value successfully parses as a date — a double-gate that prevents false positives on free-text fields.

### BigInt inference
JSON numbers at or above `Number.MAX_SAFE_INTEGER` (9007199254740991) have already lost precision during `JSON.parse`. These fields emit `z.coerce.bigint()`. Digit-only strings above the same threshold (without leading zeros) are also promoted to `z.coerce.bigint()`. Semantic key names such as `snowflake`, `discordId`, `telegramId`, `int64`, and `uint64` lower the threshold to 10+ digits.

Strings with leading zeros (`"0001234567890"`) are always treated as codes or identifiers and remain `z.string()`.

### Enum auto-detection
With multiple JSON samples, string fields with low cardinality (≤8 unique values, ratio ≤0.6) are automatically promoted to `z.enum([...])`. Semantic fields (`status`, `role`, `type`, `kind`…) get `z.enum(["value"])` even from a single sample.

### Discriminated unions
Arrays of objects sharing a string discriminant key (`type`, `kind`, `action`…) are automatically modelled as `z.discriminatedUnion()`:

```ts
// Input
[
  { "type": "success", "data": "ok" },
  { "type": "error", "error_code": 500 }
]

// Output
z.discriminatedUnion("type", [
  z.object({ type: z.literal("success"), data: z.string() }),
  z.object({ type: z.literal("error"), error_code: z.number().int() }),
])
```

Key rules:
- Branches with **structurally different keys** → `z.discriminatedUnion()`. Each branch's fields are required unless absent from multiple samples of *that branch specifically*.
- Branches with **identical key sets** (only the discriminant value differs) → single object with `z.enum([...])`. Emitting N identical branches would be redundant.
- Empty-string discriminant values disqualify the key from being a discriminant.

### Common Minimum Type (CMT) for object arrays
When a field appears in some items but not others, or is `null` in some, the engine merges all observations before emitting the schema:

```ts
// Input: 13 person records where "friends" is [] in 2 and [{...}] in 11
friends: z.array(z.object({
  id: z.number().int(),
  name: z.string(),
  alias: z.enum(["Ace", "Alpha", "Shadow"]),
}))

// NOT:
friends: z.union([z.array(z.unknown()), z.array(z.object({...})), ...])
```

Empty arrays carry no type information and never suppress the type inferred from populated siblings.

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
Genuinely self-referential structures (tree nodes, linked lists, nested menus) are detected and emitted with `z.lazy()` plus a companion TypeScript interface — required because `z.infer<>` cannot resolve circular types.

Two detection modes:
- **Ancestor-chain** — an object whose key-set matches an open ancestor in the current DFS path (exact or ≥ 80% Jaccard overlap).
- **Array-item** — an array field whose items have ≥ 80% key-overlap with the object that contains the array. Detects `children: [{children: [], name: "..."}]` without needing to reach a leaf.

```ts
// Input
{ "name": "Root", "children": [{ "name": "Child", "children": [] }] }

// Output
interface MySchema {
  name: string;
  children: MySchema[];
}

export const mySchemaSchema: z.ZodType<MySchema> = z.object({
  name: z.string(),
  children: z.array(z.lazy(() => mySchemaSchema)),
});
```

Sibling objects — items in the same array — are never flagged as recursive. Detection is capped at depth 20 and memoizes visited shapes to prevent stack overflows on pathological inputs.

### Type widening
When collecting types across multiple values, the engine applies the least-upper-bound before forming a union:

```
z.number().int() + z.number()         → z.number()
z.string().email() + z.string()       → z.string()
z.string().email() + z.string().url() → z.string()
z.string().email() × N                → z.string().email()   (same refinement kept)
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
| **coerce strings** | Promote numeric strings to `z.coerce.number()` (opt-in; off by default) |
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

### Recursive schema
```ts
interface Category {
  name: string;
  children: Category[];
}

export const categorySchema: z.ZodType<Category> = z.object({
  name: z.string(),
  children: z.array(z.lazy(() => categorySchema)),
});
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
