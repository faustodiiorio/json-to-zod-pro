# Test Cases

Reference inputs and expected outputs for regression testing.  
All cases use default options (optional ✓, nullable ✓, jsdoc ✓, pattern detect ✓, enum detect ✓, import ✓).

---

## TC-01 · Standard object with patterns

**Input**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "metadata": {
    "tags": ["typescript", "parsing", "schema"],
    "version": 1.2,
    "active": true
  },
  "items": [
    { "sku": "ITEM-001", "price": 29.99, "quantity": 5 }
  ]
}
```

**Expected output**
```ts
import { z } from "zod";

export const MySchemaSchema = z.object({
  /** unique identifier */
  id: z.string().uuid(),
  metadata: z.object({
    tags: z.array(z.string()),
    /** semantic version string */
    version: z.number(),
    active: z.boolean(),
  }),
  items: z.array(z.object({
    sku: z.string(),
    price: z.number(),
    quantity: z.number().int(),
  })),
});

export type MySchema = z.infer<typeof MySchemaSchema>;
```

**What to verify**
- `id` → `z.string().uuid()` (UUID pattern)
- `version: 1.2` (float) → `z.number()`, not `z.number().int()`
- `quantity: 5` (integer) → `z.number().int()`
- `tags` → `z.array(z.string())`

---

## TC-02 · Nullable fields with name-based inference

**Input**
```json
{
  "username": "dev_user",
  "bio": null,
  "website": "https://example.com",
  "settings": { "theme": "dark" }
}
```

**Expected output**
```ts
export const MySchemaSchema = z.object({
  username: z.string(),
  /** bio — inferred nullable; update base type if known */
  bio: z.string().nullable(),
  website: z.string().url(),
  settings: z.object({
    theme: z.string(),
  }),
});
```

**What to verify**
- `bio: null` → `z.string().nullable()` (NOT `z.null()`)
- `website` → `z.string().url()` (URL pattern)
- Comment on `bio` signals the inferred type

---

## TC-03 · Discriminated union

**Input**
```json
{
  "notifications": [
    { "type": "email", "address": "test@example.com", "subject": "Hello" },
    { "type": "sms", "phoneNumber": "+393331234567" }
  ]
}
```

**Expected output**
```ts
export const MySchemaSchema = z.object({
  notifications: z.array(
    z.discriminatedUnion("type", [
      z.object({
        type: z.literal("email"),
        address: z.string().email(),
        subject: z.string(),
      }),
      z.object({
        type: z.literal("sms"),
        phoneNumber: z.string(),
      }),
    ])
  ),
});
```

**What to verify**
- `type` chosen as discriminant (in DISC_PREFERRED, bypasses cardinality ratio)
- Each branch gets `z.literal("email")` / `z.literal("sms")`
- `address` → `z.string().email()` (email pattern)
- `phoneNumber` and `subject` → present only in their respective branches (NOT `.optional()` on each other's branch unless merged)

---

## TC-04 · Semantic enum fields

**Input**
```json
{
  "status": "published",
  "published_at": "2024-05-20T10:00:00Z",
  "ip_address": "192.168.1.1"
}
```

**Expected output**
```ts
export const MySchemaSchema = z.object({
  /** enum value — add more samples to auto-detect z.enum([...]) */
  status: z.enum(["published"]),
  /** ISO 8601 timestamp */
  published_at: z.string().datetime({ offset: true }),
  ip_address: z.string().ip({ version: "v4" }),
});
```

**What to verify**
- `status` → `z.enum(["published"])` (ENUM_HINT_KEYS single-value path)
- `published_at` → `z.string().datetime({ offset: true })` (datetime pattern)
- `ip_address` → `z.string().ip({ version: "v4" })` (IPv4 pattern)

---

## TC-05 · Root array with heterogeneous types per key

**Input**
```json
[
  { "id": 101, "metadata": "versione_1", "tags": ["prod", "internal"] },
  { "id": "UUID-102", "metadata": null, "tags": null },
  { "id": 103, "tags": [1, 2, 3] }
]
```

**Expected output**
```ts
export const MySchemaSchema = z.array(z.object({
  id: z.union([z.number().int(), z.string()]),
  /** metadata — inferred nullable; update base type if known */
  metadata: z.string().nullable().optional(),
  tags: z.array(z.union([z.string(), z.number().int()])).nullable().optional(),
}));
```

**What to verify**
- `id` seen as int (101, 103) and string ("UUID-102") → `z.union([z.number().int(), z.string()])`
- `metadata` null in one sample, missing in one → `.nullable().optional()`
- `tags` is array of strings in one, array of ints in another, null in one → union array + nullable

---

## TC-06 · Reserved / non-identifier key names

**Input**
```json
{
  "default": true,
  "import": "module",
  "var": 0,
  "2fa_enabled": false,
  "user.profile-image": "https://cdn.link/img.png",
  "parse": "collision_test",
  "safeParse": "collision_test"
}
```

**Expected output**
```ts
export const MySchemaSchema = z.object({
  default: z.boolean(),
  import: z.string(),
  var: z.number().int(),
  "2fa_enabled": z.boolean(),
  "user.profile-image": z.string().url(),
  parse: z.string(),
  safeParse: z.string(),
});
```

**What to verify**
- `default`, `import`, `var`, `parse`, `safeParse` → NO quotes (valid JS property names)
- `2fa_enabled` → quoted (starts with digit)
- `user.profile-image` → quoted (contains dot and hyphen)

---

## TC-07 · Recursive tree (z.lazy)

**Input**
```json
{
  "id": "node_1",
  "metadata": { "depth": 1, "tags": ["root"] },
  "child": {
    "id": "node_2",
    "metadata": { "depth": 2, "tags": ["branch"] },
    "child": { "id": "node_3", "metadata": null, "child": null }
  }
}
```

**What to verify**
- `child` fields → `z.lazy(() => MySchemaSchema)` with `.nullable().optional()`
- A TypeScript `interface` is emitted before the schema
- Root array of identical objects does NOT trigger recursive detection

---

## TC-08 · Root array of identical objects (NOT recursive)

**Input**
```json
[
  { "id": "a", "name": "Alice" },
  { "id": "b", "name": "Bob" }
]
```

**Expected output**
```ts
export const MySchemaSchema = z.array(z.object({
  id: z.string(),
  name: z.string(),
}));
```

**What to verify**
- No `z.lazy()`, no recursive interface
- Simple `z.array(z.object({...}))`

---

## TC-09 · Mixed array (primitives + objects + nested arrays + null)

**Input**
```json
{
  "mixed_array": [
    42,
    "stringa",
    { "type": "point", "coords": [1, 2] },
    null,
    [true, false]
  ]
}
```

**Expected output**
```ts
export const MySchemaSchema = z.object({
  mixed_array: z.array(z.union([
    z.number().int(),
    z.string(),
    z.object({ type: z.string(), coords: z.tuple([z.number().int(), z.number().int()]) }),
    z.null(),
    z.array(z.boolean()),
  ])),
});
```

**What to verify**
- `null` → `z.null()` as a union member, NOT `.nullable()` on other members
- `[true, false]` → `z.array(z.boolean())`
- `[1, 2]` → `z.tuple([z.number().int(), z.number().int()])` (fixed-length same-type)
- `type: "point"` may become `z.enum(["point"])` (ENUM_HINT_KEYS match)

---

## TC-10 · Type widening

**Input** (as array for multi-value per key)
```json
[
  { "count": 5, "label": "admin" },
  { "count": 5.5, "label": "user" }
]
```

**Expected output**
```ts
export const MySchemaSchema = z.array(z.object({
  count: z.number(),
  label: z.enum(["admin", "user"]),
}));
```

**What to verify**
- `count`: int (5) + float (5.5) → `z.number()` (NOT `z.union([z.number().int(), z.number()])`)
- `label`: two values → `z.enum(["admin", "user"])` via isLikelyEnum
