---
name: add-rls-policy
description: Add Row-Level Security policies to a Supabase table. Use when creating a new table, securing an existing table that is missing policies, or tightening access for a new use case. Triggers include "add RLS", "secure this table", "restrict access to", "row-level security for X table".
---

# Skill: Add Row-Level Security Policies

RLS policies are the database-level security enforcement for The Royal Ruse. They ensure players cannot read other players' roles, alliances, or predictions — even if a bug in the adapter code sends a malformed query.

## Before You Start

1. Identify the table and who should be allowed to read/write each row.
2. Read `docs/prd/PRD-1b-supabase-setup.md` for the full RLS policy matrix.
3. Default posture: **deny all, allow specifically**. If unsure about a case, deny it.
4. Edge Functions use `SUPABASE_SERVICE_ROLE_KEY` which bypasses RLS — they need no policies.

## RLS Policy Templates

### Enable RLS (required first)
```sql
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;
```

### SELECT — Participants of the same session
```sql
CREATE POLICY "your_table_select_session_participants" ON your_table
  FOR SELECT USING (
    session_id IN (
      SELECT session_id FROM session_players WHERE user_id = auth.uid()
    )
  );
```

### SELECT — Own row only
```sql
CREATE POLICY "your_table_select_own_row" ON your_table
  FOR SELECT USING (player_id = auth.uid());
```

### SELECT — Post-round completion only (for hidden data like alliances and predictions)
```sql
CREATE POLICY "your_table_select_post_round" ON your_table
  FOR SELECT USING (
    round_id IN (
      SELECT id FROM rounds WHERE completed_at IS NOT NULL
    )
  );
```

### INSERT — Own row only (player submits their own data)
```sql
CREATE POLICY "your_table_insert_own" ON your_table
  FOR INSERT WITH CHECK (player_id = auth.uid());
```

### No client INSERT/UPDATE/DELETE (Edge Function only)
```sql
-- Do not add INSERT, UPDATE, or DELETE policies.
-- The absence of a policy = access denied for that operation.
-- Edge Functions use service role key and bypass RLS entirely.
```

## Full Policy Matrix Reference

| Table              | SELECT                          | INSERT               | UPDATE | DELETE |
|--------------------|---------------------------------|----------------------|--------|--------|
| users              | own row only                    | Edge Function        | Edge Fn| Edge Fn|
| sessions           | session participants             | Edge Function        | Edge Fn| Edge Fn|
| session_players    | session participants             | Edge Function        | Edge Fn| Edge Fn|
| rounds             | session participants             | Edge Function        | Edge Fn| Edge Fn|
| round_roles        | own role + all post-round       | Edge Function        | Edge Fn| Edge Fn|
| round_predictions  | own + all post-round            | own INSERT only      | Denied | Denied |
| alliances          | post-round only                 | Edge Function        | Edge Fn| Edge Fn|
| powerup_purchases  | own purchases only              | Edge Function        | Edge Fn| Edge Fn|
| scores             | all session participants        | Edge Function (RPC)  | Edge Fn| Edge Fn|

## Checklist

- [ ] `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` added
- [ ] At minimum one `SELECT` policy defined
- [ ] Policy name follows convention: `"tablename_action_subject"`
- [ ] Hidden data tables (alliances, predictions) use post-round guard
- [ ] No client INSERT/UPDATE/DELETE on score-related tables
- [ ] Policies tested in Supabase SQL editor before deploying
- [ ] Existing policies not accidentally overwritten (use `CREATE POLICY` — it fails if name exists)
