# Firestore → Neon + Railway migration

Locked architecture decisions for moving Camp Notes off Firestore onto Postgres (Neon) and a Node API (Railway), while keeping Firebase Auth and Firebase Storage.

Decided 2026-09-08 via product grilling. Do not reopen these without a new decision.

## Why

- Firestore read cost and rule complexity
- Prefer Postgres row rules plus server checks
- Want a real offline-first iOS experience (including campgrounds and sites)

## Stay on Firebase

- **Auth** (including anonymous users)
- **Storage** (photos)
- **Auth-triggered Cloud Functions** (for example delete-user hooks)

## Move off Firebase

- All app data today in Firestore (campgrounds, sites, visits, ratings, users/profile, feedback, achievements, and related)
- **Data jobs** today in Cloud Functions (for example rating aggregates) → Railway Node server

## Target stack

| Piece | Choice |
| --- | --- |
| Database | Neon Postgres — **one project**, branches for **dev** and **prod** |
| API repo | New `Camp-Notes/api` |
| API host | Railway |
| HTTP | Fastify |
| SQL toolkit | Drizzle |
| Phone store | SwiftData |
| Auth to API | Firebase ID token; server verifies and sets DB user context |
| Authorization | **Both** server checks **and** Postgres row rules |
| Admin role | Postgres `users` table keyed by Firebase uid |

## Offline and sync

- **Everything** works fully offline, including campgrounds and sites
- Phone is source of truth while offline; server is the sync backend
- **Last-write-wins:** phone proposes `updated_at`; server accepts only if newer than stored
- Sync over HTTP: **push** queued edits, then **pull** changes since a cursor
- New records: **phone creates UUIDs**; server accepts them (unique constraint as backstop)
- **Soft deletes** with tombstones so deletes sync across devices
- **Photos:** queue on phone → upload to Firebase Storage when online → sync download URL with the record
- **Anonymous** Firebase users sync like signed-in users

## Cutover

**Hard cutover** (not a long dual-write strangler):

1. Build API + schema to parity
2. Rehearse Firestore → Postgres migrate against the Neon **dev** branch
3. TestFlight build pointed at the API; soak
4. Prod migrate + flip
5. Short Firestore read-only rollback window, then remove hot Firestore paths

## Out of scope (for this migration)

- Realtime websocket/SSE live updates (push/pull sync instead)
- Moving photos off Firebase Storage
- Visit sharing (still deferred product-wise)
- Moving Auth-triggered functions off Firebase in the first cut

## Related

- Existing offline ask: [#3](https://github.com/Camp-Notes/ranger-station/issues/3) (this migration is how we actually get there)
- Firebase projects today: `camp-notes-dev` (dev), `campmate-cctplus` (prod)
- Implementation tickets: see issues under the parent epic in this repo / [Camp Notes project](https://github.com/orgs/Camp-Notes/projects/2)
