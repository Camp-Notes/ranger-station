# Postgres schema

Locked relational schema for the Firestore → Neon migration.

Based on current Firestore shapes in [`campnotes-ios/docs/datamodel.md`](https://github.com/Camp-Notes/campnotes-ios/blob/main/docs/datamodel.md) and Swift models under `AppPackage/Sources/Models`, plus migration decisions locked 2026-09-08.

Parent architecture: [firestore-to-neon-railway.md](./firestore-to-neon-railway.md)

## Sync columns (every synced table)

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK (unless noted) | Phone-minted; unique |
| `created_at` | `timestamptz` not null | |
| `updated_at` | `timestamptz` not null | Phone proposes; server accepts only if newer than stored |
| `deleted_at` | `timestamptz` null | Soft-delete tombstone |

Enable PostGIS. Campground and site locations use a PostGIS point (not separate lat/lng + geohash columns).

## Out of scope for this migration

- **Achievements** — leave on Firestore / out of Postgres for now
- **Release notes** — not in use; do not migrate

## In scope

### `users`

Keyed by Firebase Auth uid (`text`, not uuid).

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `text` PK | Firebase uid |
| `email` | `text` null | Anonymous may be null |
| `display_name` | `text` null | |
| `role` | `text` not null default `'user'` | `'admin'` for admins |
| `linked_providers` | `jsonb` not null default `'[]'` | e.g. email, apple |
| sync columns | | |

### `campgrounds`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `name` | `text` not null | |
| `location` | PostGIS point not null | |
| `average_rating` | `double precision` not null default 0 | Derived |
| `total_ratings` | `integer` not null default 0 | |
| `rating_sum` | `integer` not null default 0 | |
| `total_visits` | `integer` not null default 0 | |
| `created_by` | `text` not null | FK → `users.id` |
| sync columns | | |

### `sites`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `campground_id` | `uuid` not null | FK → `campgrounds.id` |
| `campground_name` | `text` not null | Denormalized (matches app today) |
| `site_number` | `text` not null | |
| `amenities` | `text[]` not null default `'{}'` | firering, picnicTable, shower, restroom, waterHookup, electricalHookup, potableWater, wifi |
| `tags` | `text[]` not null default `'{}'` | |
| `location` | PostGIS point null | Optional GPS |
| `average_rating` | `double precision` not null default 0 | Derived from `site_ratings` |
| `total_ratings` | `integer` not null default 0 | |
| `rating_sum` | `integer` not null default 0 | |
| `total_visits` | `integer` not null default 0 | |
| `created_by` | `text` not null | FK → `users.id` |
| sync columns | | |

### `site_ratings`

Was Firestore `sites/{siteId}/ratings`.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `site_id` | `uuid` not null | FK → `sites.id` |
| `user_id` | `text` not null | FK → `users.id` |
| `visit_id` | `uuid` null | FK → `visits.id` |
| `rating` | `smallint` not null | 1–5 |
| sync columns | | |

### `visits`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `site_id` | `uuid` null | FK → `sites.id` |
| `site_name` | `text` null | Denormalized |
| `campground_name` | `text` null | Denormalized |
| `created_by` | `text` not null | Owner; FK → `users.id` |
| `start_date` | `timestamptz` not null | |
| `end_date` | `timestamptz` not null | |
| `visit_rating` | `smallint` not null | Trip rating 1–5 |
| `site_rating` | `smallint` not null | Site rating 1–5 |
| `notes` | `text` null | |
| `photos` | `text[]` not null default `'{}'` | Firebase Storage URLs |
| `weather` | `jsonb` null | Array of weather objects |
| `visibility` | `text` not null default `'private'` | private / public / shared (parity; sharing still deferred in product) |
| `shared_with` | `text[]` not null default `'{}'` | Parity with current model |
| sync columns | | |

### `personal_ratings`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | Phone-minted |
| `user_id` | `text` not null | FK → `users.id` |
| `target_id` | `uuid` not null | Site or campground id |
| `target_type` | `text` not null | `site` or `campground` |
| `rating_sum` | `integer` not null default 0 | |
| `total_ratings` | `integer` not null default 0 | |
| `average_rating` | `double precision` not null default 0 | |
| `total_visits` | `integer` not null default 0 | |
| `last_visit_date` | `timestamptz` null | |
| sync columns | | |

**Unique constraint:** (`user_id`, `target_type`, `target_id`) where `deleted_at` is null (or equivalent partial unique index).

### `feedback`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `user_id` | `text` not null | FK → `users.id` |
| `email` | `text` null | |
| `message` | `text` not null | |
| `type` | `text` not null | bug / featureRequest / general |
| `device` | `text` not null | |
| `os_version` | `text` not null | |
| `app_version` | `text` not null | |
| `status` | `text` not null default `'new'` | new / inProgress / closed / logged |
| `screenshots` | `text[]` not null default `'{}'` | Storage URLs |
| `github_link` | `text` null | |
| sync columns | | |

### `app_platforms` (version gating only)

Replaces Firestore `app/{platform}` version fields. Release notes are **not** migrated.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `text` PK | Platform key, e.g. `ios` |
| `latest_version` | `text` not null | |
| `required_version` | `text` not null | Minimum allowed app version |
| sync columns | | |

## Authorization (row rules)

Locked: enforce in **server code and database row rules** together. Server verifies the Firebase ID token and sets DB user context; row rules are the gate.

- `users`: owner of own row; admin all
- `visits`, `feedback`, `personal_ratings`, `site_ratings`: owning user; admin all
- `campgrounds`, `sites`: authenticated read; create as authenticated; update/delete owner or admin
- `app_platforms`: authenticated read; admin write

## Implementation

Tracked in [#15](https://github.com/Camp-Notes/ranger-station/issues/15).
