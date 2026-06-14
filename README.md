## Prisma-Schema-One-to-One-Relation

```schema
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
