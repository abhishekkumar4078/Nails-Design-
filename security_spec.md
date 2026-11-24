# Security Specification — The Nails Edit

## 1. Data Invariants
1. **PII Isolation**: User profile documents (`/users/{userId}`) contain PII (`email`) and are strictly readable and writable only by the authenticated verified owner (`request.auth.uid == userId`). Blanket `list` on `/users` is forbidden.
2. **Design Ownership & Integrity**: A nail design (`/designs/{designId}`) can only be created by an authenticated, email-verified user where `authorId == request.auth.uid`, `favouritesCount == 0`, and `createdAt == request.time == updatedAt`.
3. **Immutable Ownership & Timestamps**: Once created, `authorId` and `createdAt` on `/designs/{designId}` can never be mutated.
4. **Action-Based Updates**: Updates to `/designs/{designId}` are partitioned into:
   - **Owner Edit**: Only the design owner can edit `name`, `category`, `description`, `imageUrl`, `tags`, `authorName`, `authorPhotoUrl`, and `updatedAt` (must equal `request.time`).
   - **Atomic Like Sync**: Any verified user can increment/decrement `favouritesCount` by exactly `1` ONLY in an atomic batch that synchronously creates or deletes `/designs/{designId}/likes/{request.auth.uid}`.
5. **Bounded Arrays & Strings**: Every string field enforces `.size()` limits matching `firebase-blueprint.json`, `tags` is capped at `<= 8` items, and document IDs match `^[a-zA-Z0-9_\-]+$` up to 128 chars.
6. **Private Favourites**: Saved favourites (`/users/{userId}/favourites/{designId}`) are strictly private to `request.auth.uid == userId`.

## 2. The "Dirty Dozen" Adversarial Payloads
1. **Unauthenticated Design Create**: `{ name: "Blush Tips", ... }` with `auth: null` -> `PERMISSION_DENIED`.
2. **Unverified Email Create**: `{ name: "Blush Tips", ... }` with `email_verified: false` -> `PERMISSION_DENIED`.
3. **Identity Spoofing on Create**: User A creates `/designs/d1` with `authorId: "user_b"` -> `PERMISSION_DENIED`.
4. **Shadow / Ghost Field Injection**: User A creates `/designs/d1` with extra `{ isFeatured: true, role: "admin" }` -> `PERMISSION_DENIED`.
5. **Client Timestamp Forgery**: User A creates `/designs/d1` with `createdAt: "2020-01-01T00:00:00Z"` instead of `request.time` -> `PERMISSION_DENIED`.
6. **Inflated Initial Favourites Count**: User A creates `/designs/d1` with `favouritesCount: 999` -> `PERMISSION_DENIED`.
7. **Unauthorized Owner Edit**: User B tries to update `name` on User A's `/designs/d1` -> `PERMISSION_DENIED`.
8. **Non-Atomic Like Manipulation**: User B updates `favouritesCount` on `/designs/d1` by `+5` or without creating `/designs/d1/likes/user_b` -> `PERMISSION_DENIED`.
9. **Invalid Category Enum**: User A creates `/designs/d1` with `category: "Neon Cyber"` -> `PERMISSION_DENIED`.
10. **ID Poisoning / Resource Exhaustion**: User A writes to `/designs/` + 200-char ID or invalid characters -> `PERMISSION_DENIED`.
11. **PII Snooping**: User B attempts `get` or `list` on `/users/user_a` -> `PERMISSION_DENIED`.
12. **Unbounded Tags Array**: User A creates `/designs/d1` with 20 tags or non-string tag element -> `PERMISSION_DENIED`.
