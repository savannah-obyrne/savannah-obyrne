# Relational Booking Schema

**Status:** Adapted Prisma excerpt. Unrelated fields and models were omitted; relationship fields, optionality, foreign keys, indexes, and the shown uniqueness constraint are unchanged.

```prisma
model Customer {
  id       String    @id @default(cuid())
  players  Player[]
  bookings Booking[]
  payments Payment[]
}

model Player {
  id         String @id @default(cuid())
  customerId String
  customer   Customer @relation(
    fields: [customerId], references: [id], onDelete: Restrict
  )
  waivers    Waiver[]
  dailyPasses DailyPlayerPass[]

  @@index([customerId])
}

model Booking {
  id         String @id @default(cuid())
  customerId String
  customer   Customer @relation(
    fields: [customerId], references: [id], onDelete: Restrict
  )
  participants BookingParticipant[]
  payments     Payment[]
  dailyPasses  DailyPlayerPass[]

  @@index([customerId])
}

model BookingParticipant {
  id         String @id @default(cuid())
  bookingId  String
  playerId   String?
  guardianId String?
  booking    Booking @relation(
    fields: [bookingId], references: [id], onDelete: Cascade
  )
  player     Player? @relation(
    fields: [playerId], references: [id], onDelete: SetNull
  )
  guardian   Guardian? @relation(
    fields: [guardianId], references: [id], onDelete: SetNull
  )

  @@unique([bookingId, playerId])
  @@index([guardianId])
}

model Waiver {
  id         String @id @default(cuid())
  playerId   String
  bookingId  String?
  guardianId String?
  player     Player @relation(
    fields: [playerId], references: [id], onDelete: Restrict
  )
  booking    Booking? @relation(
    fields: [bookingId], references: [id], onDelete: SetNull
  )
  guardian   Guardian? @relation(
    fields: [guardianId], references: [id], onDelete: SetNull
  )
}

model DailyPlayerPass {
  id        String   @id @default(cuid())
  playerId  String
  bookingId String?
  playDate  DateTime @db.Date
  player    Player   @relation(fields: [playerId], references: [id])
  booking   Booking? @relation(fields: [bookingId], references: [id])

  @@unique([playerId, playDate])
}
```

**Demonstrates:** one-to-many relationships, optional foreign keys, join-style participant records, delete behaviour, indexes, and a migration-backed uniqueness rule for one daily record per player and play date.
