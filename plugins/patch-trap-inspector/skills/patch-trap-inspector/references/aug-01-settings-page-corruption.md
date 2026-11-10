# Aug 1 2026 — settings/page.tsx Corruption Reference

## What Was Being Edited
`src/app/dashboard/trading/settings/page.tsx` — 1,120-line Next.js settings page with tabbed layout, Framer Motion animations, and JSX-heavy component structure.

## Changes Being Applied
1. Remove `compactLayout` state + toggle UI (Notifications tab)
2. Remove `copyTradingToken`/`apiKey`/`generatingKey` state + `handleGenerateApiKey` handler
3. Rename "Telegram Broadcasts" → "Analysis Alerts"
4. Remove MT5 + API key card (Trading tab)
5. Add the broker accounts dropdown
6. Clean up dead imports

## How It Corrupted

### Patch 1 — Worked
Removed `handleGenerateApiKey` function cleanly.

### Patch 2 — Worked
Removed `copyTradingToken`/`apiKey`/`generatingKey` state.

### Patch 3 — Worked
Renamed "Telegram Broadcasts" → "Analysis Alerts".

### Patch 4 — Corrupted
Attempted to remove the Compact Layout toggle. The toggle block (a `<label>` with a nested `<div>` toggle) shared the same `<Settings>` icon pattern as other parts of the file. The `patch` tool's fuzzy matching inserted the replacement text in the wrong location (inside the existing toggle instead of replacing it), creating a duplicate block.

### Patch 5 — Corrupted Further
Attempted to remove the MT5 card. By this point the file had structural damage from Patch 4. The patch's `old_string` matched but the replacement created a duplicate `<div className="sv-card">` wrapper around the Partner card instead of removing the MT5 card.

## Resulting Damage
- Extra `</div>` in Notifications tab
- Duplicate Partner card block (the card appeared twice — once with no `sv-card-body`, once with correct body)
- Dangling `}` from removed `handleGenerateApiKey`
- Null-byte corrupted ellipsis `…` (U+2026) in `Loading accounts…` — bytes became `e2 80 00 00` instead of `e2 80 a6`
- `AnimatePresence` wrapper indentation wrong (`<motion.div>` at 12 spaces = same as `<AnimatePresence>` instead of nested inside)
- Outer `</DashboardShell>` closing tag removed from end of file
- Duplicate `handleSavePreferences` function (two copies introduced by cascading patches)
- Orphan `)}` dangling in JSX (partially-removed MT5 card left its closing JSX expression)

## Error Cascade

```
TS17008: JSX element 'DashboardShell' has no corresponding closing tag.  (line 505)
TS17008: JSX element 'div' has no corresponding closing tag.           (line 506)
TS1381: Unexpected token. Did you mean `{'}'}`?                         (line 939)
TS17002: Expected corresponding JSX closing tag for 'div'.              (line 1045)
TS1005: ')' expected.                                                (line 1046)
TS17002: Expected corresponding JSX closing tag for 'motion.div'.   (line 1072)
TS17002: Expected corresponding JSX closing tag for 'AnimatePresence'. (line 1073)
TS2451: Cannot redeclare block-scoped variable 'handleSavePreferences'.(2 occurrences)
```

Note: Turbopack (via `npm run build`) reported "Unterminated regexp literal" at line 931 even though the actual problem was the null-byte corrupted ellipsis 120 lines earlier.

## Additional Corruption Patterns Found in Recovery (same session)

### Pattern 4: AnimatePresence nesting collapse
`<motion.div>` ended up at the same 12-space indent as `<AnimatePresence>` instead of being nested at 16 spaces inside it.

### Pattern 5: Duplicate function definitions
Two copies of `handleSavePreferences` introduced — one from the original committed version and one from a partial patch that inserted the function body without the `const handleSavePreferences = async (e: React.FormEvent) => {` declaration.

### Pattern 6: Orphan `)}` dangling in JSX
When a JSX block is partially removed (e.g. MT5 card removal that deleted the `<div className="sv-card">` wrapper but left its `)}` closing expression).

### Pattern 7: LSP diagnostics are STALE after a patch
After a successful `patch` call, LSP diagnostics in the tool result are from the state BEFORE the patch was applied. They reference old line numbers and old error conditions. Do NOT treat them as authoritative — they are stale. If the patch says `files_modified`, the file changed. To verify real errors, run `npx tsc --noEmit` in a separate terminal call.

### Pattern 8: LSP line numbers shift after each patch
Each successful `patch` call shifts subsequent line numbers. An error reported at "line 436" after patch 3 is at a different physical line in the file than it was before patch 3.

### Pattern 9: The "corrupt middle" pattern — build errors point to the wrong location
When a null-byte corrupted character appears in a string literal, Turbopack reports the parse error at a syntactically innocent line 120+ lines later. TypeScript is more forgiving and reports more accurately.

## Key Lesson

When a `patch` call's diff shows content inserted in the WRONG location (not where you targeted), the tool's fuzzy matcher has found a different match than you intended. **Stop immediately.** Do not continue with more patches. The file is already structurally corrupted.

## How It Was Resolved

```bash
# Stop patching. Restore committed version.
git checkout HEAD -- src/app/dashboard/trading/settings/page.tsx
npm run build  # confirm it passes

# Write ONE Python script that applies ALL changes at once
python3 << 'PYEOF'
path = '/path/to/page.tsx'
with open(path) as f:
    content = f.read()
content = content.replace('old_1', 'new_1')
content = content.replace('old_2', 'new_2')
# ... all N changes
with open(path, 'w') as f:
    f.write(content)
PYEOF

npx tsc --noEmit && npm run build
```

## Real Session Recovery Steps (Aug 1 2026)

1. `git checkout HEAD -- src/app/dashboard/trading/settings/page.tsx` — restored clean version (1,120 lines)
2. Python script — applied all 6 changes in one call using `str.replace()` on exact committed-file patterns
3. Three additional fixes:
   - Duplicate `handleSavePreferences` (lines 331-351) — removed
   - `AnimatePresence` indentation fixed via surgical patch
   - TL dropdown state (`tlDropdownOpen`) + JSX added
4. Final build: clean. Committed as `014d8743` + `34ee5e57`.

## LSP vs Build Discrepancy Rule

| Tool | Parser | Reports accurately? |
|------|--------|-------------------|
| `tsc --noEmit` | TypeScript compiler | Yes — line numbers accurate |
| `npm run build` (Turbopack) | ECMAScript parser | No — reports downstream of actual error |

When `tsc` passes but `npm run build` fails with parse errors, trust `tsc` for error location and suspect a structural corruption (tag imbalance, corrupted character) rather than a real syntax error.
