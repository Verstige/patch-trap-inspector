# Prisma generate silent failure — `[Context: getDmmf]` reference

Session transcript (Aug 25 2026, HW888 build) — kept as reference for future schema-migration debugging.

## What happened

In one session I added 4 schema fields + 2 relations to `prisma/schema.prisma`. After each addition I ran `npx prisma generate`. The output showed `[Context: getDmmf]` six times.

Each time, I assumed the Prisma client had been updated and continued writing API endpoints that referenced the new fields. The LSP still showed errors like:

```
Property 'shippingOrder' does not exist on type 'PrismaClient'.
Property 'customerSale' does not exist on type 'PrismaUpdateInput'.
```

I dismissed these as "stale LSP, will refresh after the next patch." I kept building. I deployed 6 separate commits. Each commit had the same broken API endpoints.

## The diagnostic I missed

`prisma generate` validation failures look like this:

```
$ DATABASE_URL="postgresql://x" npx prisma generate 2>&1 | head -15
Loaded Prisma config from prisma.config.ts.

Prisma schema loaded from prisma/schema.prisma.
Error: Prisma schema validation - (get-dmmf wasm)
Error code: P1012
error: Error validating field `customerSale` in model `ShowAssignment`:
  The relation field `customerSale` on model `ShowAssignment` is missing
  an opposite relation field on the model `CustomerSale`.
  -->  prisma/schema.prisma:164
[Context: getDmmf]

Prisma CLI Version : 7.9.1
```

Three things are wrong with how I was reading this output:

1. The actual error is ABOVE `[Context: getDmmf]`, not in it
2. `[Context: getDmmf]` is just a marker — the message after it is empty by design
3. `prisma generate` returns exit code 0 even when validation fails (it's a warning, not an error)

I was reading only the last 3 lines, seeing `[Context: getDmmf]`, assuming the schema was valid.

## The fix that would have caught it

```bash
# Pre-flight check before every schema change
cd ~ maintainer/Projects/<project>
DATABASE_URL="postgresql://x" npx prisma generate 2>&1 | tee /tmp/prisma-gen.log

# Three checks
grep -c "Error" /tmp/prisma-gen.log                     # Should be 0
grep -c "Generated Prisma Client" /tmp/prisma-gen.log    # Should be 1
ls -la app/generated/prisma/client.ts                  # Should be NEWER than schema.prisma

# Verify the new field is in the generated client
grep "yourNewField" app/generated/prisma/client.ts
# If 0: prisma generate failed. Fix the schema, re-run.
# If >0: client was updated. LSP errors are stale or real code bugs.
```

## The five recurring schema errors that triggered the loop

1. **Missing back-relation** — Added `foo Foo[] @relation("Bar")` to User, forgot the matching `bar Bar? @relation("Bar")` on the other model.

2. **Self-relation without back-relation** — Added `parentSale Sale?` on Sale, forgot `childSales Sale[] @relation("SaleRefundParent")` — even unused self-relations require the back-relation.

3. **Relation field on the wrong model** — Copied a relation block, ended up with `customerSale CustomerSale?` on Sale when it belonged on ShowAssignment.

4. **Non-null column added to a populated table** — Adding a required field to a table with existing rows requires a default. Without `@default(now())`, the schema validate fails.

5. **Stray `}` from a broken patch** — A previous patch left an extra closing brace inside a model block, breaking schema syntax. The error message points to the line right after the brace, not the line of the brace itself.

## The pre-flight self-check

Before any schema edit:

```
Does my new relation have a matching back-relation on the OTHER model?
  → YES: proceed
  → NO:  prisma generate will fail with [Context: getDmmf]

Does my schema block have stray `}` or misindented @relation?
  → NO:  proceed
  → YES: fix syntax, then proceed
```

## Why this matters more than it looks

Six separate deploys shipped broken because I never noticed the schema validation was failing. Each deploy was a wasted round-trip. The cost of catching this pattern is ~5 seconds per schema change (run `prisma generate`, check the output). The cost of missing it is 6 broken deploys at ~2 minutes each to diagnose.

The lesson: when a tool returns an opaque marker (not an error message, not exit code 1), read the OUTPUT ABOVE the marker. The marker is the location indicator — the actual error is whatever comes before it.
