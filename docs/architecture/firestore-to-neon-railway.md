# Firestore → Neon + Railway

Camp Notes data lives in Neon Postgres. The iOS app talks to a Node API on Railway. Firebase remains for Auth and Storage. Auth-triggered Cloud Functions (for example account deletion) stay on Firebase. Remaining data jobs (not rating rollups) run on the Railway API.

## Stack

| Piece | Choice |
| --- | --- |
| Database | Neon Postgres — one project, branches for dev and prod; PostGIS enabled |
| API | `Camp-Notes/api` on Railway |
| HTTP | Fastify |
| SQL | Drizzle |
| Phone store | SwiftData |
| Auth to API | Firebase ID token; server verifies and sets database user context |
| Authorization | Server checks and Postgres row rules |
| Admin role | `users.role` in Postgres, keyed by Firebase uid |

Schema: [postgres-schema.md](./postgres-schema.md)

Firebase projects today: `camp-notes-dev` (dev), `campmate-cctplus` (prod).

## Offline and sync

The phone holds the working copy in SwiftData. Synced tables: `users`, `user_auth_providers`, `campgrounds`, `sites`, `site_amenities`, `site_tags`, `visits`, `visit_weather`, `visit_photos`, `visit_shares`, `site_ratings`, `feedback`, `feedback_screenshots`.

When online, the app pushes queued edits, then pulls changes since a cursor over HTTP. Conflict rule: the phone sends `updated_at`; the server accepts the write only if that timestamp is newer than what is stored. New rows use phone-minted UUIDs. Deletes are soft (`deleted_at`) so they propagate on pull.

Rating and visit aggregates are computed (views / query), not synced as stored columns or tables.

Photos: files in Firebase Storage; metadata as `visit_photos` rows (and `feedback_screenshots` for feedback). The phone queues local files, uploads when online, then syncs the storage URL with the row.

Anonymous Firebase users sync the same way as signed-in users.

Version gating (`app_platforms`) is config, not sync. The app fetches required/latest versions and checks the installed build against them.

## Cutover

Hard cutover: build API and schema, rehearse Firestore → Postgres migrate on the Neon dev branch, ship a TestFlight build pointed at the new API and run it for a while to catch issues, run prod migrate and flip, keep a short Firestore read-only window, then remove hot Firestore data paths from the app.
