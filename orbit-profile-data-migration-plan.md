# Orbit Profile Identity and Data Migration Execution Plan

Status: Active execution planning  
Date: 2026-06-05  
Audience: Orbit engineering, data platform, infrastructure, product engineering

## 2026-06-05 Execution Snapshot

Completed additive CockroachDB work:

- `orbit-infrastructure` pushed directly to `main` through `92950d1`.
- Live CRDB has `orbit_profiles.username`, `orbit_profiles.avatar_url`, and `users.linked_orbit_id`.
- Live CRDB has:
  - `orbit_profile_fun_facts`
  - `orbit_profile_images`
  - `orbit_profile_bio_sections`
  - `orbit_profile_social_handles`
  - `orbit_profile_sources`
  - `orbit_profile_aliases`
- Large existing-table indexes on `orbit_profiles.username` and `users.linked_orbit_id` were intentionally deferred after an index job began scanning `orbit_profiles`; they need a separate maintenance-window plan.
- `airflow` now has `SELECT` on Orbit CRDB tables for CRDB-to-Snowflake export. App roles still have full table grants from the schema migration.

Completed Airflow/Snowflake work:

- `orbit-airflow` PR: https://github.com/orbit-search/orbit-airflow/pull/106.
- PR branch head after live fixes: `2570684`.
- Composer environment: `orbit-airflow-v3`, Airflow `3.1.7+composer`.
- Airflow 3 uses `/api/v2`; `gcloud composer environments run ... dags/variables` is not supported for this environment.
- Live DAG `orbit_crdb_to_snowflake` is paused, parse-green, and has no import errors after the final GCS deploy.
- Restored live safety variables:
  - `ORBIT_CRDB_TO_SNOWFLAKE_KILL_SWITCH=true`
  - `ORBIT_CRDB_TO_SNOWFLAKE_ALLOW_FULL_EXTRACT=false`
  - `ORBIT_CRDB_TO_SNOWFLAKE_ALLOW_MANUAL_BACKFILL=false`
  - `ORBIT_CRDB_TO_SNOWFLAKE_TABLE_ALLOWLIST=people_search,orbit_profile_generation_failures,orbit_profiles,smart_search_events`
  - `ORBIT_PROFILE_IDENTITY_SYNC_ENABLED=true`
- Low-impact target-table bootstrap succeeded:
  - DagRun: `manual__orbit_identity_target_bootstrap__20260606T004710Z`
  - State: `success`
  - Tasks: 37/37 success
- One earlier manual bootstrap failed safely on the kill switch before data work:
  - DagRun: `manual__orbit_identity_target_bootstrap__20260606T004407Z`
- One scheduled run was created while the DAG was briefly unpaused and failed safely before data work.

Snowflake state after target-table bootstrap:

- `ORBIT_PROD.ORBIT.ORBIT_PROFILE_ALIASES`: 4 rows.
- `ORBIT_PROD.ORBIT.ORBIT_PROFILE_BIO_SECTIONS`: 0 rows.
- `ORBIT_PROD.ORBIT.ORBIT_PROFILE_FUN_FACTS`: 0 rows.
- `ORBIT_PROD.ORBIT.ORBIT_PROFILE_IMAGES`: 0 rows.
- `ORBIT_PROD.ORBIT.ORBIT_PROFILE_SOCIAL_HANDLES`: 0 rows.
- `ORBIT_PROD.ORBIT.ORBIT_PROFILE_SOURCES`: 0 rows.

The new profile-owned Snowflake target tables exist. They are empty until the CRDB content backfills run. The resource-intensive full identity refresh should run after the backfill populates the new CRDB tables.

## Current Objective

Move Orbit profile identity from generated `users` rows to canonical `orbit_profiles.orbit_id`.

Immediate execution order:

1. Add only additive CockroachDB schema in `orbit-infrastructure`.
2. Add complete CRDB-to-Snowflake backup coverage in `orbit-airflow`.
3. Run full table backups/bootstrap after the new target tables exist.
4. Backfill normalized `orbit_profile_*` tables.
5. Deploy service code changes after data is migrated.
6. Run generated-user cleanup only after all read/write, ES, HelixDB, Snowflake, and parity gates pass.

No generated-user deletes, destructive column drops, or `orbit_profiles` row deletes are part of the initial schema/backfill movement.

## Target Identity Rules

- `orbit_profiles.orbit_id` is the canonical profile identity.
- `users` represents authenticated accounts only.
- `users.linked_orbit_id` is the only account/profile link.
- `users.firebase_id` is already live and is the Firebase auth identity. Do not add or use `firebase_uid` as a database column.
- Generated synthetic users are exactly:

```sql
orbit_profiles.is_linked = false
AND orbit_profiles.sendit_id = users.id
```

Generated-user cleanup deletes only generated `users` rows and generated-user-owned legacy child rows after all replacement rows are proven. It must never delete `orbit_profiles` or new `orbit_profile_*` rows.

## Additive CockroachDB Schema

Add to `orbit_profiles`:

```sql
ALTER TABLE orbit_profiles ADD COLUMN IF NOT EXISTS username text;
ALTER TABLE orbit_profiles ADD COLUMN IF NOT EXISTS avatar_url text;
```

Add to `users`:

```sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS linked_orbit_id text;
```

Create normalized profile-owned tables:

- `orbit_profile_fun_facts`
- `orbit_profile_images`
- `orbit_profile_bio_sections`
- `orbit_profile_social_handles`
- `orbit_profile_sources`

Do not create `orbit_profile_generation_enrichers`.

Do not add `source_user_id`, `source_response_id`, `source_section_id`, or generated-user pointers to any new profile-owned table.

## Target Table Shapes

`orbit_profile_fun_facts`:

```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
orbit_id text NOT NULL,
text text NOT NULL,
labels text[],
display_order int,
created_at timestamptz NOT NULL DEFAULT now(),
last_modified_at timestamptz NOT NULL DEFAULT now()
```

`orbit_profile_images`:

```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
orbit_id text NOT NULL,
image_url text NOT NULL,
image_role text NOT NULL,
metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
display_order int,
created_at timestamptz NOT NULL DEFAULT now(),
last_modified_at timestamptz NOT NULL DEFAULT now()
```

Allowed `image_role` values:

- `primary`: face match or high-priority/default/manual image allowed into the main slideshow.
- `social_media`: social-media-derived image that may not have a face match.

`orbit_profile_bio_sections` mirrors `social_profile_bio_sections` with `user_id` replaced by `orbit_id`. Basics responses from legacy `social_profile_responses` map to `bio_type = 'BASICS'` and store fields in `ext_data`.

`orbit_profile_social_handles` mirrors `social_media_handles` with `user_id` replaced by `orbit_id`.

`orbit_profile_sources` mirrors `orbit_sources` with `user_id` replaced by `orbit_id` and includes source-owned `images jsonb`.

## Backfill Sources

- `orbit_profiles.username`: normalized `social_profiles.username`, not profile JSON.
- `orbit_profiles.avatar_url`: profile-owned picture/image source only.
- `users.linked_orbit_id`: resolved real account/profile links.
- `orbit_profile_fun_facts`: normalized fun-fact `social_profile_responses`.
- `orbit_profile_images`: normalized photo/image `social_profile_responses`.
- `orbit_profile_bio_sections`: `social_profile_bio_sections` plus supported basics responses.
- `orbit_profile_social_handles`: `social_media_handles`.
- `orbit_profile_sources`: `orbit_sources`.

Do not backfill profile-owned content from:

- `orbit_profiles.full_profile`
- `orbit_profiles.social_profiles`
- `orbit_profiles.sources_all`
- `orbit_profiles.sources_filtered`
- `orbit_profiles.images_used`
- `orbit_profiles.images_filtered`

## Live Elasticsearch Audit

Checked on 2026-06-05 using production ES credentials from local Orbit env.

- Cluster is reachable and green.
- Cluster shape: 12 nodes, 241 active primary shards, 482 active shards, 0 unassigned shards.
- Current aliases:
  - `users_current -> orbit_users_v1i`, write index true.
  - `users_current_read -> orbit_users_v1i`.
  - `users_current_write -> orbit_users_v1i`, write index true.
- Current relevant indices:
  - `orbit_users_v1i`: green/open, 418,321,012 docs, 549.4 GB, 96 primaries, 1 replica.
  - `orbit_users`: green/open, 3,000 docs, 3.4 MB, 1 primary, 1 replica.
- `orbit_profiles_current_read` and `orbit_profiles_current_write` do not exist yet.
- In sampled `users_current` documents, ES `_id` and source `id` are generated/user-style IDs, while source `orbit_id` is a different canonical profile ID.
- `sendit_id` is not mapped as a top-level field in `orbit_users_v1i`; the key serving issue is `_id` and source `id`, not a mapped `sendit_id` field.
- Sampled counts:
  - docs with source `orbit_id`: 225,101,012.
  - docs with source `id`: 225,100,831.
  - approximate unique `orbit_id`: 227,497,915.
  - approximate unique `id`: 223,132,937.

ES migration implications:

- Build a new `orbit_profiles` index family instead of mutating `users_current` in place.
- New aliases should be `orbit_profiles_current_read` and `orbit_profiles_current_write`.
- New ES document `_id` and source `id` should both be canonical `orbit_id`.
- Keep `users_current` untouched as rollback/read fallback until `orbit_profiles_current_read` parity and hydration checks pass.

## Live HelixDB Audit

Checked on 2026-06-05 using the prod API Helix endpoint and Kubernetes-managed API key.

- Production HelixDB is reachable.
- `CountPersons` returned 85,052 `Person` nodes.
- One sampled profile resolved through both `GetPersonByOrbitId` and `GetPersonBySenditId` to the same Helix node, with both `orbit_id` and `sendit_id` properties present.
- Current query bundle still exposes both canonical and Sendit-ID routes.

Canonical/current-target routes include:

- `GetPersonByOrbitId`
- `GetPersonsByOrbitIds`
- `GetConnectedPeopleByOrbitId`
- `GetProfileContributionsByOrbitId`
- `DeletePersonNodeByOrbitId`

Migration-only legacy routes include:

- `GetPersonBySenditId`
- `GetConnectedPeopleBySenditId`
- `GetConnectedSenditIdsBySenditId`
- `UpdatePersonSenditIdByOrbitId`
- `DeletePersonNodeBySenditId`
- `DeleteConnectedToBetweenSenditIds`

`GetPersonsPaginated` timed out during a small live read, likely because it orders all `Person` nodes by `orbit_id`. Do not rely on that route for broad online parity scans without a safer pagination/export strategy.

HelixDB migration implications:

- Existing-node backfill must verify all active `Person` nodes with `sendit_id` have canonical `orbit_id`.
- Before generated-user cleanup, sampled `GetPersonByOrbitId` and migration-only `GetPersonBySenditId` calls must resolve the same node or a reconciled mapping.
- Sendit-ID Helix routes remain migration-only until parity and rollback windows close.

## Snowflake Backup Requirements

Required source tables:

- `users`
- `orbit_profiles`
- `social_profiles`
- `social_profile_responses`
- `social_profile_bio_sections`
- `social_media_handles`
- `orbit_sources`

Required target tables after schema prep:

- `orbit_profile_fun_facts`
- `orbit_profile_images`
- `orbit_profile_bio_sections`
- `orbit_profile_social_handles`
- `orbit_profile_sources`
- `orbit_profile_aliases`, if treated as live identity metadata for backups.

Before generated-user cleanup:

- CRDB-to-Snowflake configs must include all required source and target tables.
- `ORBIT_CRDB_TO_SNOWFLAKE_TABLE_ALLOWLIST`, when used for a target-table identity bootstrap, must include the complete target group. When the same bootstrap includes legacy source tables too, it must include the complete source group.
- Run the resource-intensive full-table backup/bootstrap after target tables exist and after the CRDB content backfills have populated them.
- Prove freshness with row counts and max `created_at` / `last_modified_at` checks.

## Parallel Workstreams

`orbit-infrastructure`:

- Add additive SQL migration.
- Add table grants.
- Push directly after validation.
- Run only additive DDL if the exact prod migration command is proven.

`orbit-airflow`:

- Add missing Snowflake configs.
- Add full-table backup/bootstrap job or operator path.
- Validate configs and Python.
- Run backups after schema exists.

`2020-api`:

- Cut profile-serving reads/writes to `orbitId`.
- Use `users.firebase_id`, not `firebase_uid`.
- Use `users.linked_orbit_id` for real account/profile links.
- Stop profile-owned content reads through generated `userId`.

`deep-search`:

- Stop creating/requiring generated users for new profiles.
- Write normalized profile-owned rows keyed by `orbit_id`.
- Index ES by `_id = orbit_id` after new ES aliases exist.

`slipi-management-portal`:

- Read username from `orbit_profiles.username`.
- Derive linked state from `users.linked_orbit_id`.
- Stop profile admin joins through `orbit_profiles.sendit_id`.

`orbit-search-web`:

- Keep public routes compatible.
- Treat route values as profile-facing identifiers.
- Prefer username and backend `orbitId`.
- Accept normalized fun-fact/image payloads.

## Hard Stop Conditions

- Missing target CRDB tables.
- Missing app/backfill grants.
- Missing Snowflake source/target coverage.
- Stale Snowflake mirrors.
- `ORBIT_CRDB_TO_SNOWFLAKE_TABLE_ALLOWLIST` skips any required table.
- `orbit_profiles_current_read` / `orbit_profiles_current_write` missing before ES writer cutover.
- 2020-api or Deep Search still requires generated `users.id` for profile serving.
- HelixDB/search-directory sampled reads fail by canonical `orbit_id`.
- Cleanup path is anything other than reviewed identity cleanup DAG/script with explicit dry-run and confirmation token.
