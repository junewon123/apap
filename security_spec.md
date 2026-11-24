# Community Board Security Specification

## 1. Data Invariants
1. **Identity Integrity**: Every post and comment must set `authorId == request.auth.uid` on creation; `authorId` and `createdAt` are immutable forever.
2. **Verified Users Only for Writes**: Unauthenticated users or unverified emails (`email_verified != true`) cannot create, update, or delete any document.
3. **Atomic Like Synchronization**: Adding or removing a like doc in `/communityPosts/{postId}/likes/{userId}` MUST atomically increment or decrement `likesCount` on `/communityPosts/{postId}` in the same Firestore batch (`existsAfter` / `getAfter`).
4. **Atomic Comment Synchronization**: Creating a comment in `/communityPosts/{postId}/comments/{commentId}` MUST verify the parent post exists and atomically increment `commentsCount` by 1 (`getAfter`).
5. **Admin Delete Privilege**: Only the post/comment author (`existing().authorId == request.auth.uid`) OR a verified community admin (`yjw201008@gmail.com` with `email_verified == true` or `/admins/{uid}` doc) may delete a post or comment.
6. **Payload & ID Bounds**: All document IDs match `^[a-zA-Z0-9_-]+$` (`<= 128` chars), and all strings enforce explicit `size()` limits including `photoDataUrl <= 220000`.
7. **Zero PII Exposure**: `communityPosts` and `comments` store only public display nicknames and never store emails, phone numbers, or addresses.

## 2. Dirty Dozen Attack Payloads (All Must Return PERMISSION_DENIED)
1. **Unauthenticated Post Creation**: `auth = null`, `create /communityPosts/post1` with valid payload -> Denied.
2. **Unverified Email Spoof Admin**: `auth = { uid: 'attacker', token: { email: 'yjw201008@gmail.com', email_verified: false } }`, `delete /communityPosts/post1` -> Denied.
3. **Identity Spoofing on Post Create**: `auth.uid = 'userA'`, payload sets `authorId: 'userB'` -> Denied.
4. **Shadow Field Injection on Post Create**: `create /communityPosts/post1` includes extra ghost field `isVerified: true` -> Denied (`hasOnly`).
5. **Initial Like Inflation**: `create /communityPosts/post1` sets `likesCount: 999` -> Denied (`likesCount == 0`).
6. **Non-Author Post Edit**: `userB` attempts `update /communityPosts/post1` changing `title` or `content` -> Denied.
7. **Non-Author Non-Admin Post Delete**: `userB` (not admin) attempts `delete /communityPosts/post1` owned by `userA` -> Denied.
8. **Standalone Like Counter Tampering**: `userB` updates `communityPosts/post1` `likesCount: +10` without creating `/likes/userB` in batch -> Denied (`existsAfter` + `+1` check).
9. **Double Like Bypass**: `userB` creates `/likes/userB` when it already exists -> Denied.
10. **Like Impersonation**: `userB` tries to create `/likes/userA` -> Denied (`userId == request.auth.uid`).
11. **Oversized Photo DoS**: `userA` submits `photoDataUrl` exceeding 220,000 chars -> Denied (`size() <= 220000`).
12. **ID Poisoning**: `userA` creates `/communityPosts/bad$id!@#` -> Denied (`isValidId`).

## 3. Popular Searches (`/popularSearches/{termId}`) Invariants & Payloads
- **Invariants**:
  1. `term` is immutable (`1 <= size <= 40`), `count` must start at `1` on create and strictly increment by `+1` on update, `lastSearchedAt == request.time`, and `lastSearcherId == request.auth.uid`.
  2. Only verified signed-in users (`isVerifiedUser()`) may create/increment `/popularSearches/{termId}`; public `list` queries are strictly bounded (`request.query.limit <= 30`).
- **Blocked Attack Payloads**:
  - `count: 9999` on create or `count: existing().count + 50` on update -> Denied.
  - Renaming `term` on update -> Denied (`incoming().term == existing().term`).
  - Unauthenticated create/update or `lastSearcherId != request.auth.uid` -> Denied.
  - Unbounded list query without `limit <= 30` -> Denied.

## 4. User Accounts (`/userAccounts/{userId}`) PII Isolation & Moderation Invariants
- **Invariants**:
  1. `email` is PII: `get` is restricted strictly to document owner (`request.auth.uid == userId`) or verified admin (`isAdmin()`). `list` is restricted exclusively to verified primary admin (`yjw201008@gmail.com` with `email_verified == true`) with `limit <= 100`.
  2. Users may only create/update their own document (`userId == request.auth.uid`), with `email == request.auth.token.email`, `uid == request.auth.uid`, immutable `firstLoginAt`, monotonic `+1` `loginCount` increments, and may NEVER write `isBanned`, `banReason`, `bannedAt`, or `kickedAt`.
  3. Only verified primary admin (`yjw201008@gmail.com`) may execute kick (`kickedAt == request.time`) or ban/unban (`isBanned`, `banReason`, `bannedAt`) updates on non-self user accounts.
  4. Banned users (`isBanned == true`) are blocked from creating/updating/deleting community posts, comments, likes, popular searches, or updating their own account doc.
- **Blocked Attack Payloads**:
  - Non-admin user calls `list /userAccounts` or `get /userAccounts/otherUser` -> Denied (prevents email/PII leak).
  - Unverified admin email spoof calls `list /userAccounts` or updates `isBanned`/`kickedAt` -> Denied (`email_verified == true` required).
  - User tries to write another user's email or spoof `uid` -> Denied (`incoming().email == request.auth.token.email`).
  - User tries to inflate `loginCount`, alter `firstLoginAt`, or clear their own `isBanned`/`kickedAt` -> Denied (`affectedKeys().hasOnly(...)`).
  - Banned user attempts to write a community post, comment, like, or self-login update -> Denied (`isNotBanned()`).

## 5. Guest Visits (`/guestVisits/{visitorId}`) Zero-PII & Moderation Invariants
- **Invariants**:
  1. Zero PII: Documents store ONLY `visitorId` (`^guest_[a-zA-Z0-9_-]+$`, `10..80` chars), constant `label == '비로그인'`, `visitCount`, and server timestamps.
  2. `list` queries are restricted strictly to verified primary admin (`yjw201008@gmail.com`) with `limit <= 100`.
  3. Unauthenticated visitors can only initialize (`visitCount == 1`) or monotonically increment (`visitCount + 1`) their own `guest_*` record when not banned, and may NEVER modify `label`, `visitorId`, `firstVisitAt`, `isBanned`, `banReason`, `bannedAt`, or `kickedAt`.
  4. Only verified primary admin may execute kick (`kickedAt == request.time`) or ban/unban (`isBanned`, `banReason`, `bannedAt`) updates on guest visit records.
- **Blocked Attack Payloads**:
  - Unauthenticated caller calls `list /guestVisits` -> Denied (`isPrimaryAdmin()` required).
  - Unauthenticated caller injects arbitrary `label` or `visitCount: 9999` or `isBanned: false` -> Denied.
  - Non-admin caller tries to kick/ban/unban a guest visit -> Denied (`isPrimaryAdmin()` required).

