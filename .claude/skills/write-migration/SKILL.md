---
name: write-migration
description: Write a Supabase PostgreSQL migration file. Use when adding a new table, modifying an existing schema, adding indexes, or changing constraints. Triggers include "schema change", "new table", "add column", "database migration", "modify the schema".
---

# Skill: Write a Database Migration

Supabase migrations are plain SQL files tracked in version control. They are applied in chronological order and cannot be rolled back automatically — write them carefully.

## Before You Start

1. Read the schema section of `docs/tdd/THE_ROYAL_RUSE_TDD.md` for the existing table shapes.
2. Read `docs/prd/PRD-1b-supabase-setup.md` for the full schema and RLS policies.
3. Confirm the change does not break existing RLS policies.
4. If adding a new table: use the `add-rls-policy` skill immediately after.
5. Never modify an existing migration file that has already been applied. Always add a new one.

## File Naming Convention

```
supabase/migrations/YYYYMMDDHHMMSS_descriptive_name_in_snake_case.sql
```

Generate the timestamp: `date +%Y%m%d%H%M%S`

Examples:
- `20260418120000_add_round_predictions_table.sql`
- `20260501093000_add_wanted_brand_to_users.sql`
- `20260510150000_add_index_session_players_session_id.sql`

## New Table Migration Template

```sql
-- supabase/migrations/YYYYMMDDHHMMSS_add_your_table.sql
-- Migration: Add your_table
-- Author: [your name]
-- Date: YYYY-MM-DD
-- Description: [One sentence explaining why this table is needed.]

-- ── Create Table ──────────────────────────────────────────────────────────────
CREATE TABLE your_table (
  -- Primary key: always UUID, always server-generated
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Foreign keys: reference with ON DELETE CASCADE where appropriate
  session_id   UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  player_id    UUID NOT NULL REFERENCES users(id),

  -- Data columns: explicit NOT NULL + DEFAULT where applicable
  status       TEXT NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending', 'active', 'complete')),
  points       INT NOT NULL DEFAULT 0,
  extra_data   JSONB,                    -- Use JSONB for flexible semi-structured data

  -- Timestamps: always TIMESTAMPTZ — never TIMESTAMP
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Constraints
  UNIQUE (session_id, player_id)         -- Example unique constraint
);

-- ── RLS — mandatory immediately after CREATE TABLE ─────────────────────────────
-- Never leave a table without RLS enabled and at least one SELECT policy.
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;

-- SELECT: who can read rows?
CREATE POLICY "your_table_select_participants" ON your_table
  FOR SELECT USING (
    session_id IN (
      SELECT session_id FROM session_players WHERE user_id = auth.uid()
    )
  );

-- INSERT: typically via Edge Function only — no client INSERT policy
-- UPDATE: typically via Edge Function only — no client UPDATE policy
-- DELETE: typically CASCADE from parent — no client DELETE policy

-- ── Indexes ───────────────────────────────────────────────────────────────────
-- Add indexes on foreign keys and frequently queried columns.
CREATE INDEX idx_your_table_session_id ON your_table (session_id);
CREATE INDEX idx_your_table_player_id  ON your_table (player_id);

-- ── Comments ──────────────────────────────────────────────────────────────────
COMMENT ON TABLE your_table IS 'Stores [description]. Written by [which Edge Function].';
COMMENT ON COLUMN your_table.status IS 'Lifecycle status: pending → active → complete.';
```

## Add Column Migration Template

```sql
-- supabase/migrations/YYYYMMDDHHMMSS_add_column_to_your_table.sql
-- Migration: Add [column_name] to [table_name]

-- Add the column with a safe DEFAULT to avoid locking existing rows
ALTER TABLE your_table
  ADD COLUMN new_column TEXT NOT NULL DEFAULT 'default_value';

-- If adding a nullable column (no default needed):
ALTER TABLE your_table
  ADD COLUMN optional_column TIMESTAMPTZ;

-- Update existing rows if a specific value is needed:
UPDATE your_table SET new_column = 'computed_value' WHERE some_condition;
```

## Add Index Migration Template

```sql
-- supabase/migrations/YYYYMMDDHHMMSS_add_index_your_table_column.sql
-- Migration: Add index on your_table.column_name for [query pattern description]

-- Use CONCURRENTLY to avoid locking the table in production
CREATE INDEX CONCURRENTLY IF NOT EXISTS
  idx_your_table_column_name ON your_table (column_name);
```

## PostgreSQL Standards (Project-Specific)

```sql
-- ✅ UUID PKs
id UUID PRIMARY KEY DEFAULT gen_random_uuid()

-- ✅ TIMESTAMPTZ (not TIMESTAMP)
created_at TIMESTAMPTZ NOT NULL DEFAULT now()

-- ✅ TEXT + CHECK (not PostgreSQL ENUM — harder to migrate)
status TEXT NOT NULL CHECK (status IN ('a', 'b', 'c'))

-- ✅ NOT NULL with DEFAULT where possible
is_active BOOLEAN NOT NULL DEFAULT true

-- ✅ ON DELETE CASCADE for child tables
session_id UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE

-- ✅ JSONB for flexible semi-structured data (not JSON)
metadata JSONB

-- ❌ Never use SERIAL/BIGSERIAL — always UUID
-- ❌ Never use TIMESTAMP without timezone
-- ❌ Never create PostgreSQL ENUM types
-- ❌ Never skip RLS after CREATE TABLE
```

## Applying Migrations

```bash
# Apply all pending migrations to linked Supabase project
supabase db push

# Reset local database and re-apply all migrations (local dev only)
supabase db reset

# Generate migration from local schema diff
supabase db diff --schema public -f your_migration_name
```

## Checklist

- [ ] File named with timestamp prefix: `YYYYMMDDHHMMSS_description.sql`
- [ ] Placed in `supabase/migrations/`
- [ ] UUID primary key with `gen_random_uuid()`
- [ ] All timestamps use `TIMESTAMPTZ`
- [ ] No PostgreSQL ENUM types
- [ ] `NOT NULL` with `DEFAULT` on required columns
- [ ] RLS enabled immediately after `CREATE TABLE`
- [ ] At least one `SELECT` policy defined
- [ ] Indexes added for all foreign keys
- [ ] `COMMENT ON TABLE` added
- [ ] Migration tested locally with `supabase db reset` before pushing
- [ ] Existing RLS policies on related tables still valid after schema change
