# Prisma — One-to-One Relation (Line by Line)

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../app/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id String @id @default(cuid())
  email String @unique
  password String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  profile Profile? @relation("UserProfile")
}

model Profile {
  id String @id @default(cuid())
  firstName String
  lastName String
  city String
  postalCode String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  userId String @unique
  user User @relation("UserProfile", fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)
}
```
---

## Model User

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
| `id` | `String` | `@id @default(cuid())` | Primary key। Insert করলে Prisma নিজেই unique id বানিয়ে দেবে |
| `email` | `String` | `@unique` | দুইজন user একই email দিতে পারবে না |
| `password` | `String` | — | Plain field, কোনো constraint নেই |
| `createdAt` | `DateTime` | `@default(now())` | Row create হওয়ার সময় DB থেকে automatically set হয় |
| `updatedAt` | `DateTime` | `@updatedAt` | Row update হলে Prisma automatically সময় update করে |
| `profile` | `Profile?` | `@relation("UserProfile")` | একজন User-এর একটা Profile থাকতে পারে (optional — `?` মানে nullable) |

### `profile Profile? @relation("UserProfile")` — বিস্তারিত

- এটা **virtual field** — database-এ কোনো column তৈরি হয় না
- শুধু Prisma-কে বলছে: "User থেকে তার Profile access করতে পারব"
- `?` মানে Profile না থাকলেও চলবে (optional)
- `"UserProfile"` হলো relation-এর নাম — দুই দিকে একই নাম থাকতে হবে

---

## Model Profile

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
| `id` | `String` | `@id @default(cuid())` | Profile-এর নিজস্ব Primary key |
| `firstName` | `String` | — | Plain text field |
| `lastName` | `String` | — | Plain text field |
| `city` | `String` | — | Plain text field |
| `postalCode` | `String` | — | Plain text field |
| `createdAt` | `DateTime` | `@default(now())` | Automatically set হয় |
| `updatedAt` | `DateTime` | `@updatedAt` | Automatically update হয় |
| `userId` | `String` | `@unique` | **Foreign key** — কোন User-এর Profile সেটা store করে। `@unique` থাকায় একজন User-এর শুধু একটাই Profile হতে পারে |
| `user` | `User` | `@relation(...)` | এই Profile কোন User-এর — সেই User object access করতে দেয় |

### `userId String @unique` — কেন গুরুত্বপূর্ণ?

```
User table:          Profile table:
id = "abc123"   ←── userId = "abc123"  ✅ (One-to-One নিশ্চিত)
id = "xyz789"   ←── userId = "xyz789"  ✅

userId = "abc123" আবার দিলে → ✗ Error (UNIQUE constraint)
```

`@unique` না থাকলে একজন User-এর অনেকগুলো Profile হয়ে যেত — তখন One-to-**Many** হতো।

### `@relation(...)` — বিস্তারিত

```prisma
user User @relation(
  "UserProfile",                    -- relation-এর নাম (User model-এ যা দিয়েছি)
  fields:     [userId],             -- Profile table-এ যে column Foreign Key হিসেবে কাজ করে
  references: [id],                 -- User table-এ যে column-এ সে point করে
  onDelete:   Cascade,              -- User delete হলে তার Profile-ও delete হবে
  onUpdate:   Cascade               -- User-এর id বদলালে userId-ও automatically বদলাবে
)
```

---

## onDelete ও onUpdate — Option তুলনা

| Option | মানে |
|---|---|
| `Cascade` | Parent delete/update হলে Child-ও delete/update হবে |
| `Restrict` | Child থাকলে Parent delete করা যাবে না — Error দেবে |
| `SetNull` | Parent delete হলে Child-এর foreign key `null` হয়ে যাবে |
| `NoAction` | DB-এর default behavior follow করবে |

> এই schema-তে `Cascade` দেওয়া আছে — User delete করলে তার Profile-ও automatically delete হবে।

---

## One-to-One কীভাবে কাজ করে — সংক্ষেপে

```
User                        Profile
─────────────────           ─────────────────────────────
id: "abc123"    ←────────── userId: "abc123"  (@unique)
email: "a@b.com"            firstName: "Wasim"
                            lastName: "Uddin"
```

- **Owning side:** `Profile` — কারণ এখানে actual `userId` column আছে
- **Reference side:** `User` — শুধু virtual `profile` field আছে, DB-তে কোনো column নেই
- **Rule:** Foreign key (`userId`) যে model-এ, সেটাই owning side

---

## Prisma দিয়ে Data Access

```typescript
// User তৈরি করার সময় Profile-ও তৈরি
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

// User সহ তার Profile আনা
const userWithProfile = await prisma.user.findUnique({
  where: { id: "abc123" },
  include: { profile: true },
})

// Profile থেকে User আনা
const profileWithUser = await prisma.profile.findUnique({
  where: { userId: "abc123" },
  include: { user: true },
})
```
