# 🗄️ Prisma — One-to-One Relation

> **লক্ষ্য:** এই ডকুমেন্টে শিখবো কিভাবে Prisma Schema-তে **One-to-One** সম্পর্ক তৈরি হয়।  
> ব্যবহৃত উদাহরণ: `User ↔ Profile` (একজন User-এর একটিই Profile থাকতে পারে)

---

## 📌 সম্পর্কের ধরন — এক নজরে

| Relation Type | উদাহরণ | মানে |
|---|---|---|
| **One-to-One** | User ↔ Profile | একজন User-এর একটিই Profile থাকে |
| **One-to-Many** | User → Posts | একজন User অনেকগুলো Post লিখতে পারে |
| **Many-to-Many** | Post ↔ Tags | একটি Post-এ অনেক Tag, একটি Tag অনেক Post-এ থাকতে পারে |

> এই ডকুমেন্টে আমরা **One-to-One** নিয়ে কাজ করবো।

---

## Schema

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../app/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

// ─────────────────────────────────────────
// Model: User
// ─────────────────────────────────────────
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  profile   Profile? @relation("UserProfile")  // One-to-One ← এটাই আমাদের focus
}

// ─────────────────────────────────────────
// Model: Profile
// ─────────────────────────────────────────
model Profile {
  id         String   @id @default(cuid())
  firstName  String
  lastName   String
  city       String
  postalCode String
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  userId String @unique                         // Foreign Key — @unique মানে One-to-One
  user   User   @relation("UserProfile", fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)
}
```

---

## 🔍 `User` Model — লাইন বাই লাইন ব্যাখ্যা

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  password  String
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  profile   Profile?  @relation("UserProfile")
}
```

| Field | Type | Attribute | মানে |
|---|---|---|---|
| `id` | `String` | `@id @default(cuid())` | Primary Key। Insert করলে Prisma নিজেই unique id বানিয়ে দেবে |
| `email` | `String` | `@unique` | দুইজন User একই email দিতে পারবে না |
| `password` | `String` | — | Plain field, কোনো constraint নেই |
| `createdAt` | `DateTime` | `@default(now())` | Row create হওয়ার সময় DB থেকে automatically set হয় |
| `updatedAt` | `DateTime` | `@updatedAt` | Row update হলে Prisma automatically সময় update করে |
| `profile` | `Profile?` | `@relation("UserProfile")` | একজন User-এর একটা Profile থাকতে পারে (optional — `?` মানে nullable) |

### 📌 `profile Profile? @relation("UserProfile")` — বিস্তারিত

| অংশ | মানে |
|---|---|
| `profile` | Field-এর নাম — Prisma Client-এ এই নামে access করবো |
| `Profile?` | Type হলো Profile, `?` মানে optional (Profile নাও থাকতে পারে) |
| `@relation("UserProfile")` | Relation-এর নাম — Profile model-এও একই নাম থাকতে হবে |

> 💡 এটা **virtual field** — Database-এ কোনো column তৈরি হয় না।  
> শুধু Prisma-কে বলছে: "User থেকে তার Profile access করতে পারবো।"

---

## 🔍 `Profile` Model — লাইন বাই লাইন ব্যাখ্যা

```prisma
model Profile {
  id         String    @id @default(cuid())
  firstName  String
  lastName   String
  city       String
  postalCode String
  createdAt  DateTime  @default(now())
  updatedAt  DateTime  @updatedAt
  userId     String    @unique
  user       User      @relation("UserProfile", fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)
}
```

| Field | Type | Attribute | মানে |
|---|---|---|---|
| `id` | `String` | `@id @default(cuid())` | Profile-এর নিজস্ব Primary Key |
| `firstName` | `String` | — | Plain text field |
| `lastName` | `String` | — | Plain text field |
| `city` | `String` | — | Plain text field |
| `postalCode` | `String` | — | Plain text field |
| `createdAt` | `DateTime` | `@default(now())` | Automatically set হয় |
| `updatedAt` | `DateTime` | `@updatedAt` | Automatically update হয় |
| `userId` | `String` | `@unique` | **Foreign Key** — কোন User-এর Profile সেটা store করে। `@unique` থাকায় একজন User-এর শুধু একটিই Profile হতে পারে |
| `user` | `User` | `@relation(...)` | এই Profile কোন User-এর — সেই User object access করতে দেয় |

---

## 🔑 `userId String @unique` — কেন এত গুরুত্বপূর্ণ?

```
User table:           Profile table:
──────────────        ──────────────────────────────────
id = "abc123"  ←───── userId = "abc123"  ✅ (One-to-One)
id = "xyz789"  ←───── userId = "xyz789"  ✅ (One-to-One)

userId = "abc123" আবার দিতে চাইলে → ✗ Error (UNIQUE constraint violated)
```

> ⚠️ **`@unique` না থাকলে কী হতো?**  
> একজন User-এর অনেকগুলো Profile হয়ে যেত — তখন সেটা One-to-**Many** হতো।  
> `@unique` দেওয়াই নিশ্চিত করে এটা **One-to-One**।

---

## 🔗 `@relation(...)` — বিস্তারিত বিশ্লেষণ

```prisma
user User @relation(
  "UserProfile",       -- relation-এর নাম (User model-এ যা দিয়েছি, একই হতে হবে)
  fields:     [userId],    -- Profile table-এ যে column Foreign Key হিসেবে কাজ করে
  references: [id],        -- User table-এর কোন column-কে point করছে
  onDelete:   Cascade,     -- User delete হলে তার Profile-ও delete হবে
  onUpdate:   Cascade      -- User-এর id বদলালে userId-ও automatically বদলাবে
)
```

| Argument | মানে |
|---|---|
| `"UserProfile"` | Relation-এর নাম — দুই model-এ এই নাম এক হতে হবে |
| `fields: [userId]` | এই model-এ (Profile) কোন field টা FK? → `userId` |
| `references: [id]` | User model-এ কোন field-এর সাথে match করবে? → `id` |
| `onDelete: Cascade` | User delete হলে Profile-ও delete হবে |
| `onUpdate: Cascade` | User-এর `id` পরিবর্তন হলে Profile-এর `userId`-ও update হবে |

---

## 🗺️ Relation Diagram

```
┌──────────────────────┐          ┌──────────────────────────┐
│         User         │          │         Profile          │
├──────────────────────┤          ├──────────────────────────┤
│ id (PK)              │◄────┐    │ id (PK)                  │
│ email                │     │    │ firstName                │
│ password             │     │    │ lastName                 │
│ createdAt            │     │    │ city                     │
│ updatedAt            │     │    │ postalCode               │
│                      │     │    │ createdAt                │
│ profile → Profile?   │     │    │ updatedAt                │
└──────────────────────┘     └────│ userId (FK + @unique)    │
                                  │ user → User              │
         1                        └──────────────────────────┘
      (একজন)
                                           1
                                       (একটিই)
```

**পড়ার নিয়ম:** একজন `User` → সর্বোচ্চ একটি `Profile` রাখতে পারে।  
একটি `Profile` → সর্বদা ঠিক একজন `User`-এর সাথে যুক্ত।

---

## 🏠 Owning Side vs Reference Side

| বিষয় | Owning Side (Profile) | Reference Side (User) |
|---|---|---|
| **কোনটা?** | `Profile` model | `User` model |
| **চেনার উপায়** | যেখানে actual FK column আছে (`userId`) | যেখানে শুধু virtual field আছে (`profile`) |
| **DB-তে column** | ✅ `userId` column তৈরি হয় | ❌ কোনো column তৈরি হয় না |
| **`@relation` details** | `fields`, `references`, `onDelete`, `onUpdate` সব এখানে | শুধু relation নাম থাকে |

> 💡 **Rule:** Foreign Key (`userId`) যে model-এ থাকে, সেটাই **Owning Side**।

---

## ⚖️ `onDelete` ও `onUpdate` — সব Option

| Option | মানে |
|---|---|
| `Cascade` | Parent delete/update হলে Child-ও delete/update হবে |
| `Restrict` | Child থাকলে Parent delete করা যাবে না — Error দেবে |
| `SetNull` | Parent delete হলে Child-এর foreign key `null` হয়ে যাবে |
| `SetDefault` | Parent delete হলে Child-এর FK default value পাবে |
| `NoAction` | DB-এর default behavior follow করবে |

> এই schema-তে `Cascade` দেওয়া আছে — User delete করলে তার Profile-ও **automatically** delete হবে।

---

## 🛠️ Migration-এর পর Database-এ কী হয়?

```sql
-- User table (কোনো FK নেই এখানে)
CREATE TABLE "User" (
  "id"        TEXT PRIMARY KEY,
  "email"     TEXT UNIQUE NOT NULL,
  "password"  TEXT NOT NULL,
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW(),
  "updatedAt" TIMESTAMP NOT NULL
);

-- Profile table (userId হলো Foreign Key + UNIQUE)
CREATE TABLE "Profile" (
  "id"         TEXT PRIMARY KEY,
  "firstName"  TEXT NOT NULL,
  "lastName"   TEXT NOT NULL,
  "city"       TEXT NOT NULL,
  "postalCode" TEXT NOT NULL,
  "createdAt"  TIMESTAMP NOT NULL DEFAULT NOW(),
  "updatedAt"  TIMESTAMP NOT NULL,
  "userId"     TEXT UNIQUE NOT NULL,             -- UNIQUE মানে One-to-One নিশ্চিত

  FOREIGN KEY ("userId") REFERENCES "User"("id")
    ON DELETE CASCADE
    ON UPDATE CASCADE
);
```

> 📌 **লক্ষ্য করো:** `profile Profile?` (User model-এ) — এই virtual field Database-এ **কোনো column তৈরি করে না**।  
> Database-এ শুধু `userId` column তৈরি হয়, বাকিটা Prisma Client-এর কাজ।

---

## 🧪 Prisma Client দিয়ে Data Query

### ১. User তৈরি করার সময় Profile-ও তৈরি

```typescript
const user = await prisma.user.create({
  data: {
    email: "wasim@example.com",
    password: "hashed_password",
    profile: {
      create: {
        firstName: "Wasim",
        lastName: "Uddin",
        city: "Dhaka",
        postalCode: "1200",
      },
    },
  },
})
```

---

### ২. User সহ তার Profile আনা (`include`)

```typescript
const userWithProfile = await prisma.user.findUnique({
  where: { id: "abc123" },
  include: { profile: true },  // Profile-ও সাথে আনবে
})

// Result:
// {
//   id: "abc123",
//   email: "wasim@example.com",
//   profile: {
//     id: "...",
//     firstName: "Wasim",
//     city: "Dhaka",
//     ...
//   }
// }
```

---

### ৩. Profile থেকে User আনা

```typescript
const profileWithUser = await prisma.profile.findUnique({
  where: { userId: "abc123" },
  include: { user: true },  // সেই Profile-এর User আনবে
})
```

---

### ৪. আলাদাভাবে Profile তৈরি করা (existing User-এ)

```typescript
const profile = await prisma.profile.create({
  data: {
    firstName: "Wasim",
    lastName: "Uddin",
    city: "Dhaka",
    postalCode: "1200",
    userId: "existing_user_id",  // FK সরাসরি দাও
  },
})
```

---

### ৫. Profile আপডেট করা

```typescript
const updated = await prisma.profile.update({
  where: { userId: "abc123" },
  data: { city: "Chittagong" },
})
```

---

## ⚖️ One-to-One vs One-to-Many — পার্থক্য

| বিষয় | One-to-One (User ↔ Profile) | One-to-Many (User → Posts) |
|---|---|---|
| User Model-এ | `profile Profile?` | `posts Post[]` |
| Child Model-এ FK | `userId String @unique` | `authorId String` (unique নেই) |
| `@unique` লাগে? | ✅ হ্যাঁ (একটাই হবে) | ❌ না (অনেক হতে পারে) |
| Relation নাম | `@relation("UserProfile")` | নাম optional |
| Array ব্যবহার | ❌ না | ✅ হ্যাঁ (`Post[]`) |
| `?` ব্যবহার | ✅ হ্যাঁ (`Profile?` — optional) | ❌ না |

> 🔑 **মূল পার্থক্য:** One-to-One-এ FK-তে `@unique` থাকে — এটাই সবচেয়ে বড় চেনার উপায়।

---

## 📝 মনে রাখার নিয়ম (Summary)

```
✅ One-to-One Rule:
   "Owning" দিকে (Profile) → Foreign Key রাখো (userId) + @unique দাও
   "Reference" দিকে (User)  → Optional field রাখো (profile Profile?)

✅ @unique ছাড়া One-to-One হবে না
   @unique না থাকলে One-to-Many হয়ে যাবে

✅ Virtual fields (profile, user) Database-এ column তৈরি করে না
   শুধু Prisma Client query-তে কাজে লাগে

✅ onDelete: Cascade → Parent (User) মুছলে Child (Profile)-ও মুছবে

✅ Owning Side চেনার উপায়:
   যে model-এ Foreign Key column আছে → সেটাই Owning Side
```

---

## 🔗 এই Schema-তে Relation Summary

```
User (1) ──────────── (1) Profile    ← One-to-One  ✅ (এই ডকুমেন্ট)
User (1) ──────────── (N) Post       ← One-to-Many
```

> **পরবর্তী:** One-to-Many Relation (যেমন User → Posts) শিখতে হবে।

---
