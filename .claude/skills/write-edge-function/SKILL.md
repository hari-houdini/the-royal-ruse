---
name: write-edge-function
description: Write a new Supabase Edge Function in Deno/TypeScript. Use when server-side authority is needed for game logic — scoring, role assignment, power-up validation, or any state mutation that clients must not be trusted to perform. Triggers include "server-side logic", "new Edge Function", "scoring logic", "validate on server", "server must calculate".
---

# Skill: Write a Supabase Edge Function

Edge Functions run on Supabase's Deno edge runtime. They use the `SUPABASE_SERVICE_ROLE_KEY` (admin key) to bypass RLS and write authoritative data. The client never receives this key.

**Core contract:** The client sends an intent. The Edge Function validates, calculates, writes to the database, and returns resolved state. The client trusts only what the server returns.

## Before You Start

1. Name the function in `kebab-case`. It becomes the URL path: `https://project.supabase.co/functions/v1/your-function-name`.
2. Identify every database table this function reads and writes.
3. Define the input shape (request body) and output shape (response body).
4. Add an idempotency check if this function mutates state — calling it twice should be safe.
5. Read the relevant PRD for the domain context (e.g., `PRD-1d-vote-scoring.md` for scoring functions).

## Template

```typescript
// supabase/functions/your-function-name/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

// ── Input / Output Types ───────────────────────────────────────────────────────
interface RequestBody {
  required_field: string
  optional_field?: string
}

interface ResponseBody {
  status: "success" | "error"
  data?: YourDataShape
  error?: string
}

interface YourDataShape {
  result_field: string
  another_field: number
}

// ── Handler ───────────────────────────────────────────────────────────────────
serve(async (req: Request): Promise<Response> => {
  // 1. Parse and validate input
  let body: RequestBody
  try {
    body = await req.json()
  } catch {
    return errorResponse("Invalid JSON in request body", 400)
  }

  const { required_field, optional_field } = body

  if (!required_field) {
    return errorResponse("required_field is required", 400)
  }

  // 2. Create Supabase admin client (service role — bypasses RLS)
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  )

  // 3. Idempotency check — mandatory for any state-mutating function
  const { data: existing, error: fetchError } = await supabase
    .from("your_table")
    .select("completed_at")
    .eq("id", required_field)
    .single()

  if (fetchError) {
    return errorResponse("Record not found: " + fetchError.message, 404)
  }

  if (existing?.completed_at) {
    return errorResponse("Already processed — idempotency guard", 409)
  }

  // 4. Core logic
  const result = computeYourResult(existing, optional_field)

  // 5. Write to database
  const { error: writeError } = await supabase
    .from("your_table")
    .update({
      result_field: result.result_field,
      completed_at: new Date().toISOString()
    })
    .eq("id", required_field)

  if (writeError) {
    return errorResponse("Database write failed: " + writeError.message, 500)
  }

  // 6. Return resolved state
  return successResponse({ result_field: result.result_field, another_field: 0 })
})

// ── Domain Logic ──────────────────────────────────────────────────────────────
// Pure functions — no Supabase calls. Mirrors the GDScript domain service.

function computeYourResult(
  record: Record<string, unknown>,
  modifier?: string
): YourDataShape {
  // Pure computation — no side effects
  return {
    result_field: String(record.some_field ?? ""),
    another_field: 0
  }
}

// ── Response Helpers ──────────────────────────────────────────────────────────

function successResponse(data: YourDataShape): Response {
  return new Response(
    JSON.stringify({ status: "success", data }),
    { status: 200, headers: { "Content-Type": "application/json" } }
  )
}

function errorResponse(message: string, status: number): Response {
  return new Response(
    JSON.stringify({ status: "error", error: message }),
    { status, headers: { "Content-Type": "application/json" } }
  )
}
```

## Deploying

```bash
# Deploy a single function
supabase functions deploy your-function-name

# Deploy all functions
supabase functions deploy

# Test locally before deploying
supabase functions serve your-function-name --env-file .env.local
```

## Calling from GDScript

```gdscript
# In an Adapter — the Use Case calls the port; the adapter calls the Edge Function.
func invoke_your_function(required_field: String) -> Result:
    var url: String = "%s/functions/v1/your-function-name" % Config.SUPABASE_URL
    var headers: PackedStringArray = [
        "Content-Type: application/json",
        "Authorization: Bearer %s" % Config.SUPABASE_ANON_KEY
    ]
    var body: Dictionary = { "required_field": required_field }

    var http := HTTPRequest.new()
    # ... full HTTP request implementation
    return Result.ok(response_data)
```

## Checklist

- [ ] File at `supabase/functions/your-function-name/index.ts`
- [ ] Input types defined as TypeScript interfaces
- [ ] All required fields validated — 400 returned if missing
- [ ] Idempotency check present before any database writes
- [ ] Uses `SUPABASE_SERVICE_ROLE_KEY` — never the anon key
- [ ] All Supabase errors caught and returned as 500 with message
- [ ] Pure computation functions extracted — no Supabase calls inside them
- [ ] `successResponse()` and `errorResponse()` helpers used consistently
- [ ] Deployed to Supabase and smoke-tested with `curl` before merging
- [ ] GDScript adapter method implemented to call this function
- [ ] GDScript mock updated to simulate the function's response

## Smoke Testing with curl

```bash
curl -X POST https://YOUR_PROJECT.supabase.co/functions/v1/your-function-name \
  -H "Authorization: Bearer YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"required_field": "test_id_123"}'
```

Expected: `{ "status": "success", "data": { ... } }` or `{ "status": "error", "error": "..." }`
