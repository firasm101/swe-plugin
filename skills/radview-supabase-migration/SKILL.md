---
name: radview-supabase-migration
description: Write and apply Supabase schema migrations the radview way. Use whenever the task involves a schema change (CREATE TABLE, ALTER TABLE, CREATE FUNCTION, CREATE VIEW, INSERT into a registry table). Overrides upstream's generic migration guidance with the procedures specific to this repo.
---

# How to write and apply a Supabase migration

This is the radview-specific procedure. The upstream `code-implementation` skill assumes generic SQL flow; this skill encodes the gates and traps we've actually hit.

## When to use this skill

- Adding/altering columns
- Creating/replacing views (read carefully — VIEW changes have a special trap)
- Creating/replacing functions or RPCs
- Inserting into registry tables (`services`, `data_sources`, `contracts`)
- Any DDL change

## Step 0 — Verify schema before writing SQL (NON-NEGOTIABLE)

This step caught MKT-415 (four-sided drift). Skipping it ships drift.

Before writing one line of SQL, confirm the columns and types you're going to reference actually exist in production. Run a query like:

```bash
curl -sS -X POST "https://deiyfumozjyhfyxdzqlc.supabase.co/rest/v1/rpc/exec_sql" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT column_name, data_type FROM information_schema.columns WHERE table_schema='\''public'\'' AND table_name='\''<TABLE>'\'' ORDER BY ordinal_position;"}'
```

The service-role key from CONVENTION.md §6 (or the chat) is required because `exec_sql` is restricted.

What you're checking:
- Does every column you reference exist?
- Is its type what you assumed?
- For drift-prone tables (CONVENTION.md §3.2), did you remember the canonical name? `content_changes.changed_at`, NOT `created_at`. `pages.last_seen_at`, NOT `last_seen`.

If the schema doesn't match your assumption, **don't write SQL that pretends it does**. Either fix the assumption or expand the migration to add the missing piece.

## DROP+CREATE rule for views (CONVENTION.md §7.2)

Postgres' `CREATE OR REPLACE VIEW` does NOT allow column-type changes. If your migration redefines a view with a different column shape (added column, changed type, renamed column), you must:

```sql
DROP VIEW IF EXISTS public.my_view CASCADE;
CREATE VIEW public.my_view AS SELECT ...;
```

`CASCADE` drops dependent views; recreate them in the same migration if needed. The MKT-295d case study shipped a `CREATE OR REPLACE VIEW` with an added column and broke production until DROP+CREATE was used.

## Idempotency

Every migration must be safe to apply twice. If `run-migration.yml` retries (or another session re-applies because the workflow timed out without recording success), the migration must succeed the second time.

Patterns:
- `CREATE TABLE IF NOT EXISTS`
- `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`
- `CREATE INDEX IF NOT EXISTS` (or `CREATE INDEX CONCURRENTLY` outside a transaction — see special case below)
- `CREATE OR REPLACE FUNCTION`
- For seeds: `INSERT ... ON CONFLICT (key) DO NOTHING` or `DO UPDATE` carefully (never include `owner` in the SET clause for `services` — operator-set, migration must not stomp).

## File naming + location

```
supabase/migrations/<YYYYMMDD>_<mkt-ticket-or-topic>_<short-description>.sql
```

Examples from the repo:
- `20260508_pr196_drift_fix_tenant_onboarding_seeds.sql`
- `20260508_mkt415_pages_content_raw_columns_and_invariant_fixes.sql`

## Apply via run-migration.yml (CONVENTION.md §7.5)

Migrations are NEVER applied directly to production from a developer machine. Always:

1. Commit the migration file in your PR.
2. Open the PR and wait for CI green + merge to master.
3. After merge, dispatch the `run-migration.yml` workflow with input `migration_file=supabase/migrations/<your-file>.sql`.
4. Watch the workflow run; verify the `apply` step says `success`.
5. Re-query `information_schema` to confirm the schema change landed.

For PR comments, write the verification artifact: paste the `information_schema` result that proves the change is live.

## Registry rows

If your migration adds:
- A new table written by code → add a row to `data_sources` in the same migration
- A new handler/cron/runbook → add a row to `services` in the same migration

CI guards (`data-lineage-guard.yml`, `services-catalog-guard.yml`) will fail PRs that add code referencing a table/handler without the registry row.

## After applying

Update `CONVENTION.md`:
- Schema-shaped values you introduced go in §3 (drift-prone columns, RPC names, role enum members, etc.)
- The convention-drift scanner (`scripts/scan_convention_drift.py`) needs updating in the same PR if you introduced a new canonical-value class

## Trap log (real failures, do not repeat)

- **MKT-415:** assumed `pages.content_raw` existed; it didn't. Step 0 schema verify would have caught this. Fixed by ADD COLUMN IF NOT EXISTS in the same migration as the invariant rewrite.
- **MKT-167:** wrote invariant referencing `pages.last_seen` — column is `last_seen_at`. Step 0 would have caught.
- **MKT-295d:** used `CREATE OR REPLACE VIEW` with a column-type change — Postgres rejected it at apply time. Fixed by DROP+CREATE.
- **PR #191:** added handler without a `services` row. CI guard caught it; fixed in PR #196 drift-fix migration.
- **`owner` field in seeds:** included in `ON CONFLICT DO UPDATE SET` clause — stomped operator-set ownership. Always exclude `owner` from UPDATE clauses on `services`.

## Quick reference checklist

- [ ] Step 0: queried `information_schema` to verify columns/types
- [ ] Drift-prone column names match CONVENTION.md §3.2
- [ ] Migration is idempotent (re-apply safe)
- [ ] Filename is `<YYYYMMDD>_<topic>_<description>.sql`
- [ ] If view column shape changed: DROP+CREATE, not CREATE OR REPLACE
- [ ] Registry row(s) added if new table/handler/cron/runbook introduced
- [ ] `owner` field NOT in any `ON CONFLICT DO UPDATE SET` on `services`
- [ ] CONVENTION.md §3 updated if a new canonical value was introduced
- [ ] `scan_convention_drift.py` updated if a new canonical-value class was introduced
- [ ] Plan: dispatch `run-migration.yml` after merge
