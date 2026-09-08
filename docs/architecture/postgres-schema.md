# Postgres schema (draft)

First-cut relational schema for the Firestore → Neon migration.

Source of truth for current shapes: [`campnotes-ios/docs/datamodel.md`](https://github.com/Camp-Notes/campnotes-ios/blob/main/docs/datamodel.md) plus Swift models under `AppPackage/Sources/Models`.

Sync columns on every synced table (locked decisions):

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` (unless noted) | Phone-minted; unique |
| `created_at` | `timestamptz` | |
| `updated_at` | `timestamptz` | Phone proposes; server accepts only if newer |
| `deleted_at` | `timestamptz` null | Soft-delete tombstone |

Row rules: owner (and admin via `users.role`) for private rows; authenticated read for shared catalog as needed. Exact policies land in #15.

---

## `users`

Keyed by Firebase Auth uid (text, not uuid).

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `text` PK | Firebase uid |
| `email` | `text` null | Anonymous may be null |
| `display_name` | `text` null | |
| `role` | `text` not null default `'user'` | `'admin'` for admins |
| `linked_providers` | `jsonb` not null default `'[]'` | `email`, `apple`, … |
| `created_at` / `updated_at` / `deleted_at` | | |

---

## `campgrounds`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `name` | `text` not null | |
| `latitude` | `double precision` not null | |
| `longitude` | `double precision` not null | |
| `geo_hash2` … `geo_hash9` | `text` null | Keep for map queries; may later use PostGIS |
| `average_rating` | `double precision` not null default 0 | Derived |
| `total_ratings` | `integer` not null default 0 | |
| `rating_sum` | `integer` not null default 0 | |
| `total_visits` | `integer` not null default 0 | From visit jobs |
| `created_by` | `text` not null | FK → `users.id` |
| sync columns | | |

---

## `sites`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `campground_id` | `uuid` not null | FK → `campgrounds.id` |
| `campground_name` | `text` not null | Denormalized (matches app today) |
| `site_number` | `text` not null | |
| `amenities` | `text[]` not null default `'{}'` | firering, picnicTable, … |
| `tags` | `text[]` not null default `'{}'` | |
| `latitude` / `longitude` | `double precision` null | Optional GPS |
| `average_rating` / `total_ratings` / `rating_sum` | | Derived from `site_ratings` |
| `total_visits` | `integer` not null default 0 | |
| `created_by` | `text` not null | FK → `users.id` |
| sync columns | | |

---

## `site_ratings`

Was `sites/{siteId}/ratings` subcollection.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `site_id` | `uuid` not null | FK → `sites.id` |
| `user_id` | `text` not null | FK → `users.id` |
| `visit_id` | `uuid` null | FK → `visits.id` |
| `rating` | `smallint` not null | 1–5 |
| sync columns | | |

---

## `visits`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `site_id` | `uuid` null | FK → `sites.id` |
| `site_name` | `text` null | Denormalized |
| `campground_name` | `text` null | Denormalized |
| `created_by` | `text` not null | Owner |
| `start_date` / `end_date` | `timestamptz` not null | |
| `visit_rating` | `smallint` not null | Trip rating 1–5 |
| `site_rating` | `smallint` not null | Site rating 1–5 |
| `notes` | `text` null | |
| `photos` | `text[]` not null default `'{}'` | Firebase Storage URLs |
| `weather` | `jsonb` null | Array of weather objects |
| `visibility` | `text` not null default `'private'` | private / public / shared |
| `shared_with` | `text[]` not null default `'{}'` | Kept; sharing still deferred in product |
| sync columns | | |

---

## `personal_ratings`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `text` PK | Keep `{userId}_{targetType}_{targetId}` or switch to uuid + unique constraint |
| `user_id` | `text` not null | FK → `users.id` |
| `target_id` | `text` not null | Site or campground id |
| `target_type` | `text` not null | `site` \| `campground` |
| `rating_sum` / `total_ratings` / `average_rating` / `total_visits` | | |
| `last_visit_date` | `timestamptz` null | |
| sync columns | | |

---

## `feedback`

| Column | Type | Notes |
| --- | --- | --- |
| `id` | `uuid` PK | |
| `user_id` | `text` not null | |
| `email` | `text` null | |
| `message` | `text` not null | |
| `type` | `text` not null | bug / featureRequest / general |
| `device` / `os_version` / `app_version` | `text` | |
| `status` | `text` not null default `'new'` | |
| `screenshots` | `text[]` | Storage URLs |
| `github_link` | `text` null | |
| sync columns | | |

---

## `achievements`

Catalog of achievement definitions (read for authenticated users; admin write).

Exact columns TBD from current Firestore `achievements` docs when migrate script is written — include sync columns.

---

## App config (version gate + release notes)

Today under Firestore `app/ios` (+ release_notes). Options for Postgres:

1. `app_platforms` + `release_notes` + `release_note_entries` tables, or
2. Keep reading version gate from Firestore `app/ios` for the first cut (public read already) and migrate later.

**Recommendation for first cut:** migrate `app` into Postgres so the API is the single data plane; iOS pulls version/release notes via sync or a small public config route.

---

## Open schema decisions (call in #15)

1. PostGIS `geography(Point)` vs lat/lng + geohash columns  
2. `personal_ratings.id` stay composite text vs uuid  
3. Whether `shared_with` / `visibility` stay in v1 schema (yes for parity)  
4. Whether `app` / release notes move in the first migrate  
