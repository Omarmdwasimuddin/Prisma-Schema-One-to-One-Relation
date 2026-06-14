# Prisma `@default()` ID Types — বিস্তারিত গাইড

> **Topic:** `autoincrement()` vs `uuid()` vs `cuid()` — কখন কোনটা ব্যবহার করবো?

---

## ১. `@default(autoincrement())`

```prisma
model User {
  id Int @id @default(autoincrement())
}
```

### কীভাবে কাজ করে?

Database নিজেই সংখ্যা বাড়িয়ে দেয়। প্রথম row insert হলে `id = 1`, দ্বিতীয় row = `2`, তৃতীয় = `3` — এভাবে চলতে থাকে।

- Type: `Int` (4 bytes)
- Generate করে: **Database** (PostgreSQL SERIAL / IDENTITY)
- Example value: `1`, `2`, `3`, `100`, `9999`

### সুবিধা

- সহজ এবং readable
- Index performance ভালো (sequential integer)
- Debug করতে সুবিধা (`id = 5` মানে ৫ম row)
- Storage কম (মাত্র 4 bytes)

### অসুবিধা

- **Guessable** — `/users/1`, `/users/2` দেখে যে কেউ বুঝতে পারে মোট কতজন user আছে (Security risk)
- **Distributed system-এ collision** — দুটো আলাদা DB server-এ একই `id` তৈরি হতে পারে
- Multiple database merge করা কঠিন

### কখন ব্যবহার করবো?

✅ Internal admin panel (বাইরের কেউ URL দেখবে না)
✅ Small personal project
✅ Audit log, counter, sequence যেখানে order important
❌ Public-facing API বা URL-এ id expose হলে — **avoid করো**

---

## ২. `@default(uuid())`

```prisma
model User {
  id String @id @default(uuid())
}
```

### কীভাবে কাজ করে?

RFC 4122 standard অনুযায়ী **128-bit random ID** generate করে। এটা UUID version 4 (v4) — সম্পূর্ণ random।

- Type: `String` (36 characters)
- Generate করে: **Prisma (application-side)** অথবা PostgreSQL (`gen_random_uuid()`)
- Format: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`
- Example value: `550e8400-e29b-41d4-a716-446655440000`

### সুবিধা

- **Globally unique** — দুটো আলাদা database বা server-এ একই UUID তৈরি হওয়ার সম্ভাবনা practically শূন্য
- **Not guessable** — random হওয়ায় security ভালো
- **Industry standard** — সব জায়গায় চেনা format
- Distributed system-এ safe

### অসুবিধা

- **36 characters** — storage বেশি লাগে
- **Not sortable by creation time** — random হওয়ায় insert order বোঝা যায় না
- Index fragmentation হতে পারে (random হওয়ায় B-tree index-এ gaps তৈরি হয়)
- Human-readable না — debug করতে একটু কষ্ট

### কখন ব্যবহার করবো?

✅ Public API (URL-এ id থাকে)
✅ Multiple database / Microservices
✅ Security-sensitive data (user id, payment id)
✅ Third-party system-এর সাথে integration
❌ Creation order দরকার হলে — `cuid()` ভালো

---

## ৩. `@default(cuid())`

```prisma
model User {
  id String @id @default(cuid())
}
```

### কীভাবে কাজ করে?

**C**ollision-resistant **U**nique **ID**। Prisma application-side এটা generate করে। UUID-র মতো random কিন্তু আরও smart।

- Type: `String` (25 characters)
- Generate করে: **Prisma (application-side only)**
- Structure: `c` + **timestamp** + **fingerprint** + **random**
- Example value: `clh2m3x4k0000abc123xyz456`

### Structure বিশ্লেষণ

```
c  l  h  2  m  3  x  4  k  0  0  0  0  a  b  c  1  2  3  x  y  z  4  5  6
│  │──────────────────────────┤  ├──────────────────────────────────────────│
│        timestamp part        │           random part
│
└── সবসময় 'c' দিয়ে শুরু (CUID identifier)
```

### সুবিধা

- **Sortable** — timestamp prefix থাকায় আগে তৈরি হলে alphabetically আগে আসে
- **UUID-র চেয়ে ছোট** — 25 chars vs 36 chars
- **Not guessable** — random part আছে
- **Distributed safe** — collision practically অসম্ভব
- URL-safe (hyphen নেই, শুধু lowercase alphanumeric)
- Prisma-র সাথে best compatibility

### অসুবিধা

- **Application-side only** — DB নিজে generate করতে পারে না (Prisma ছাড়া raw SQL দিয়ে insert করলে manually দিতে হবে)
- UUID-র মতো industry-wide standard না

### কখন ব্যবহার করবো?

✅ **Prisma project-এ সবচেয়ে ভালো choice**
✅ Creation order important (logs, feed, timeline)
✅ URL-এ id expose হয়
✅ Distributed / multi-server
✅ Next.js + Prisma combination

---

## তুলনা সারণি

| বিষয় | `autoincrement()` | `uuid()` | `cuid()` |
|---|---|---|---|
| **Type** | `Int` | `String` | `String` |
| **Size** | 4 bytes | 36 chars | 25 chars |
| **Example** | `1`, `2`, `3` | `550e8400-e29b-...` | `clh2m3x4k0000...` |
| **Generate করে** | Database | DB বা App | App (Prisma) only |
| **Guessable?** | ✗ হ্যাঁ | ✓ না | ✓ না |
| **Sortable?** | ✓ হ্যাঁ | ✗ না | ✓ হ্যাঁ |
| **Distributed safe?** | ✗ না | ✓ হ্যাঁ | ✓ হ্যাঁ |
| **URL-safe?** | ✓ হ্যাঁ | ✓ হ্যাঁ | ✓ হ্যাঁ |
| **Prisma best fit?** | মোটামুটি | ভালো | **সবচেয়ে ভালো** |

---

## কখন কোনটা — সিদ্ধান্ত গাইড

| Situation | সুপারিশ |
|---|---|
| Internal admin panel (URL-এ id নেই) | `autoincrement()` |
| Public API / REST endpoint | `uuid()` বা `cuid()` |
| Security-sensitive (user, payment) | `uuid()` বা `cuid()` |
| Multiple DB / Microservices | `uuid()` বা `cuid()` |
| Creation order দরকার (feed, log) | `cuid()` |
| Next.js + Prisma project | **`cuid()`** ✅ |

| Third-party standard UUID দরকার | `uuid()` |

---

## এর বাইরে আরও কিছু Option

Prisma-তে এই তিনটির বাইরেও কিছু option আছে:

### `@default(dbgenerated(...))`

```prisma
id String @id @default(dbgenerated("gen_random_uuid()"))
```

- Database-specific function call করতে দেয়
- PostgreSQL-এ UUID generate করার native way
- ⚠️ DB-dependent — অন্য DB-তে migrate করলে সমস্যা হবে

### `nanoid()` (Third-party)

```prisma
// Prisma-তে built-in নেই
// Application code-এ manually করতে হয়
import { nanoid } from 'nanoid'
const id = nanoid() // "V1StGXR8_Z5jdHi6B-myT"
```

- ছোট, URL-safe random ID
- Prisma `@default()` হিসেবে সরাসরি ব্যবহার করা যায় না
- ⚠️ Third-party package (`nanoid`) install করতে হয়

---

## Next.js + Prisma Project-এর জন্য সুপারিশ

```prisma
// ✅ সব model-এ এটা ব্যবহার করো
model User {
  id        String   @id @default(cuid())
  // ...
}

model Profile {
  id        String   @id @default(cuid())
  userId    String   @unique   // ← Int থেকে String-এ change করতে হবে
  // ...
}

model Product {
  id        String   @id @default(cuid())
  // ...
}

model Branch {
  id        String   @id @default(cuid())
  // ...
}
```

### কেন `cuid()` বেছে নিলাম?

1. **Prisma native** — কোনো extra package দরকার নেই
2. **Sortable** — log, feed, timeline সহজে sort করা যাবে
3. **URL-safe** — `/products/clh2m3x4k0000...` দেখতে ভালো, hyphen নেই
4. **UUID-র চেয়ে ছোট** — 25 chars, DB storage কম
5. **Secure** — guessable না, random part আছে
6. **Future-proof** — Distributed system-এ safe

---

> **⚠️ গুরুত্বপূর্ণ নোট:**
> যদি আগে `autoincrement()` দিয়ে migration করা থাকে, তাহলে `cuid()` বা `uuid()`-তে switch করলে নতুন migration তৈরি করতে হবে এবং existing data-এ কোনো সমস্যা হবে না — নতুন row থেকে নতুন format শুরু হবে। তবে foreign key যেসব জায়গায় আছে (যেমন `userId Int`) সেগুলোও `String`-এ change করতে হবে।
