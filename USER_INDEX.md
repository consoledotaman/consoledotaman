# UserIndex - Design

## 1. Field Decisions

**Rule:** Only store in `UserIndex` if:
- Needed for DB-level filtering/sorting, OR
- Clerk can't filter by it server-side, meaning finding users by this field would require fetching everyone from Clerk

Fields you fetch from Clerk per-request (name, email, imageUrl) stay in Clerk — you already have the user object when you need those, caching them is just redundancy and extra sync surface.

| Clerk field | Store? | Reason |
|---|---|---|
| `id` | ✅ PK | Join key between Clerk and our DB |
| `createdAt` | ✅ as `clerkJoinedAt` | "Date joined" filter. Clerk has no server-side filter for this |
| `lastActiveAt` | ✅ as `clerkLastActiveAt` | "Last active" filter. Same — Clerk can't filter by it |
| `lastSignInAt` | ✅ as `clerkLastSignInAt` | Different from `lastActiveAt`. This is the last explicit login. Useful for detecting users who signed up but never returned — a churn signal you can't query otherwise |
| `planType` | ✅ | Not in Clerk at all, lives in our DB via `TherapistSubscription` |
| `firstName` / `lastName` | ❌ | Fetched from Clerk when rendering the user, no need to duplicate |
| `email` | ❌ | Same — fetched per-request from Clerk |
| `imageUrl` | ❌ | Same — fetched per-request |
| `banned` | ❌ | Clerk handles ban enforcement entirely. No business logic in our app depends on it. View it in Clerk dashboard directly |
| `locked` | ❌ | Temporary Clerk-managed state. No app logic depends on it |
| `publicMetadata` | ❌ | Too dynamic, changes per feature. Fetch on demand |

---

## 2. Proposed Schema

```prisma
model UserIndex {
  clerkId           String            @id @map("clerk_id")
  planType          TherapistPlanType @default(FREE) @map("plan_type")
  clerkJoinedAt     DateTime          @map("clerk_joined_at")
  clerkLastActiveAt DateTime?         @map("clerk_last_active_at")
  clerkLastSignInAt DateTime?         @map("clerk_last_sign_in_at")
  lastSyncedAt      DateTime          @default(now()) @map("last_synced_at")
  createdAt         DateTime          @default(now()) @map("created_at")
  updatedAt         DateTime          @updatedAt @map("updated_at")

  @@index([planType])
  @@index([clerkJoinedAt])
  @@index([clerkLastActiveAt])
  @@index([clerkLastSignInAt])
  @@map("user_index")
}
```

### Why `lastSyncedAt`

The reconciliation job uses it to detect stale rows. Without it, there's no way to tell a freshly synced row from one that hasn't been touched in weeks.

### Why `clerkLastSignInAt`

`lastActiveAt` updates on every API call — it reflects app usage. `lastSignInAt` updates only on explicit login. The gap between them is useful: a user with `clerkJoinedAt = 3 months ago` and `clerkLastSignInAt = never` is someone who was invited but never actually used the product.

---

## 3. What This Unlocks

| Capability | Today | With UserIndex |
|---|---|---|
| Filter by plan (FREE/PLUS/PRO) | DB for paid + Clerk fetch-all for FREE | Pure DB |
| Filter by joined date | Clerk fetch-all + post-filter | Pure DB |
| Filter by last active | Clerk fetch-all + post-filter | Pure DB |
| Filter by last sign-in | Not possible | Pure DB |
| Churn detection (inactive 30d) | Not possible at scale | `WHERE clerkLastActiveAt < now() - 30d` |
| "Signed up, never returned" cohort | Not possible | `WHERE clerkLastSignInAt IS NULL` |
| Consistent pagination on any filter | Broken — page sizes vary | Correct — DB handles pagination |

---

## 4. Sync Strategy — No Data Drift Under Any Circumstances

**Current state:** There is no Clerk webhook handler anywhere in the codebase. Only a Stripe webhook exists at `/payments/webhook`. This is the most important gap.

Four independent sync layers. Each is a safety net for the one above it.

---

### Layer 1 — Clerk Webhooks (Primary, Real-Time)

Add `/clerk/webhook` handling three events. These fire for every user change in Clerk regardless of whether the user is logged in or using the app.

**`user.created`** — fires when a new user signs up
```
→ INSERT UserIndex row with clerkJoinedAt, clerkLastSignInAt, planType: FREE
```

**`user.updated`** — fires when anything on the user changes
```
→ UPDATE clerkLastActiveAt, clerkLastSignInAt, lastSyncedAt
```

**`user.deleted`** — fires when a user is removed from Clerk
```
→ DELETE the UserIndex row
```

Uses Svix signature verification — same pattern as the existing Stripe webhook in `paymentController.ts`.

---

### Layer 2 — `authenticate` Middleware (Safety Net, Per-Request)

Every authenticated request goes through `auth.ts`. After verifying the token, upsert `UserIndex`:

```ts
await prisma.userIndex.upsert({
  where: { clerkId: user.id },
  create: {
    clerkId: user.id,
    clerkJoinedAt: new Date(user.createdAt),
    clerkLastActiveAt: new Date(user.lastActiveAt ?? Date.now()),
    clerkLastSignInAt: user.lastSignInAt ? new Date(user.lastSignInAt) : null,
    planType: 'FREE',
    lastSyncedAt: new Date(),
  },
  update: {
    clerkLastActiveAt: new Date(user.lastActiveAt ?? Date.now()),
    clerkLastSignInAt: user.lastSignInAt ? new Date(user.lastSignInAt) : null,
    lastSyncedAt: new Date(),
  },
  // NOTE: update block never touches planType — plan is owned by Layer 3
});
```

**Write throttle:** Upsert is skipped if `lastSyncedAt` is less than 5 minutes old — avoids a DB write on every single API request for active users:

```ts
const existing = await prisma.userIndex.findUnique({ where: { clerkId: user.id } });
const stale = !existing || (Date.now() - existing.lastSyncedAt.getTime() > 5 * 60 * 1000);
if (stale) { /* run upsert */ }
```

This self-heals any row missed by webhooks. If `user.created` wasn't delivered while the server was down, this creates the row on the user's next login.

---

### Layer 3 — `paymentController.ts` (Plan Sync)

After every `TherapistSubscription` upsert in the Stripe webhook handler, mirror `planType` to `UserIndex`. This is the **only** layer that writes `planType` — no other layer touches it.

```ts
await prisma.userIndex.upsert({
  where: { clerkId: therapistId },
  create: {
    clerkId: therapistId,
    clerkJoinedAt: new Date(), // placeholder — corrected by Layer 1 or 2
    planType: newPlanType,
    lastSyncedAt: new Date(),
  },
  update: {
    planType: newPlanType,
  },
});
```

---

### Layer 4 — Nightly Reconciliation Job (Drift Catcher)

A cron job that runs once a night. Catches anything that slipped through all three layers — missed webhooks, edge cases, bugs.

**Algorithm:**
1. Page through all Clerk users in batches of 100
2. For each user, if `UserIndex` row is missing or `lastSyncedAt` is stale → upsert
3. After all Clerk users are processed, delete any `UserIndex` rows whose `clerkId` no longer exists in Clerk (ghost rows from missed `user.deleted` events)

```
for each page of Clerk users:
  for each user:
    if no UserIndex row OR lastSyncedAt > 24h old:
      upsert UserIndex row

ghost cleanup:
  allClerkIds = all IDs collected above
  UserIndex.deleteMany({ clerkId NOT IN allClerkIds })
```

---

## 5. Drift Scenarios — Coverage Map

| Scenario | Handled by |
|---|---|
| New user signs up | Layer 1 (`user.created`) |
| New user signs up while server is down | Layer 2 (on first login) |
| User's lastActiveAt changes | Layer 1 (`user.updated`) + Layer 2 (on next request) |
| User's lastSignInAt changes | Layer 1 (`user.updated`) |
| Plan upgrades or downgrades | Layer 3 (paymentController) |
| User deleted from Clerk | Layer 1 (`user.deleted`) |
| `user.deleted` webhook missed | Layer 4 (ghost row cleanup) |
| Webhook delivery fails repeatedly | Layer 2 (next login) + Layer 4 (nightly) |
| DB down during authenticate upsert | Layer 1 retries via Svix + Layer 4 |
| Row manually corrupted in DB | Layer 4 overwrites on next nightly run |

No single point of failure causes permanent drift. Every scenario has at least two independent recovery paths.

---

## 6. Migration Order

1. Add `UserIndex` model → run `prisma migrate dev`
2. Add Clerk webhook endpoint (`/clerk/webhook`) for `user.created`, `user.updated`, `user.deleted`
3. Update `authenticate` middleware with throttled upsert
4. Update `paymentController.ts` to mirror `planType` after every subscription upsert
5. Run one-time backfill script (pages through all Clerk users, seeds the table)
6. Update `adminController.getUsers` to query `UserIndex` — remove `fetchAllClerkUsers` and `applyPostFilters`
7. Add nightly reconciliation cron job
