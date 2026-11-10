# Sequential JSX Patches — Compounding Corruption (Aug 1 2026)

## The Failure Mode

Removing 3+ distinct JSX blocks from the same component file with sequential patches. Each edit modifies surrounding indentation and closing-tag structure. After 4-6 patches the file is structurally broken: duplicate `</div>`, dangling `}`, corrupted string literals.

**Symptom:** TypeScript (`tsc --noEmit`) passes clean. `npm run build` (Turbopack) fails with "Unterminated regexp literal" pointing at a line that looks syntactically correct. The real corruption is earlier in the file.

**Root cause:** Turbopack's ECMAScript parser is stricter than TypeScript's. A null-byte corrupted character or a structural tag imbalance causes a parse failure that TypeScript tolerates.

## Real Session: settings/page.tsx (Aug 1 2026)

Six sequential patches to remove MT5 token + compact layout toggle from a 1,100-line settings page produced cascading corruption across 9 distinct failure patterns.

## 9 Corruption Patterns

### P1: Duplicate closing `</div>` in sibling tab
When a wrapper `<div>` is removed from one section, the closing tag for a sibling section gets consumed instead. Both sections now have mismatched open/close counts.

### P2: Dangling `}` from removed handler function
Removing a function body but leaving its closing brace. The orphaned `}` sits in the middle of other code as a stray statement terminator.

### P3: Null-byte corrupted Unicode
`…` (U+2026 ellipsis) encoded as `e2 80 00 00` instead of `e2 80 a6`. `cat -v` shows `Loading accounts[M-^@][M-^@]`. Turbopack misparses; TypeScript tolerates.

**Detection:** `sed -n 'L,P' file.tsx | xxd | grep "00 00"`

### P4: `AnimatePresence` nesting collapse
`<motion.div>` ends up at the same indent as `<AnimatePresence>` instead of nested inside it. The parent and child become siblings. `</AnimatePresence>` then has no matching open.

### P5: Duplicate function definitions
A partial patch inserts a function body without its declaration line. The original function declaration remains. Result: two definitions of the same name → `TS2451: Cannot redeclare block-scoped variable`.

### P6: Orphan `)}` dangling in JSX
A JSX expression closer (`)}`) remains after the expression's opening tag was removed. The orphaned closer confuses the parser.

### P7: LSP diagnostics are STALE after a patch
After a successful `patch` call, LSP diagnostics in the tool result reflect the state BEFORE the patch applied. They reference old line numbers. Do NOT trust them for error location — they are stale until you run `tsc --noEmit` fresh.

### P8: LSP line numbers shift after each patch
Each `patch` call shifts physical line numbers. An error at "line 436" after patch 3 is at a different physical line after patch 4. Always re-run `tsc --noEmit` after each fix.

### P9: "Corrupt middle" — build error reported downstream of actual corruption
Null-byte in a string literal causes Turbopack to report parse failure 50-150 lines AFTER the actual corruption point. TypeScript reports more accurately. If the reported line looks fine, scan backward.

## Prevention Rule

When a file needs 3+ distinct JSX structural changes:

1. Read the full file ONCE
2. Plan all removals
3. Use `write_file` for a clean one-call rewrite

Sequential `patch` is safe ONLY when each edit is in a non-overlapping region separated by hundreds of lines AND you've re-read the file between calls.

## Recovery Recipe

```bash
# 1. STOP — do not send more patches
# 2. Restore committed version
git checkout HEAD -- src/app/dashboard/trading/settings/page.tsx
npm run build  # confirm it passes

# 3. Write ONE Python script that applies ALL changes at once
python3 << 'PYEOF'
path = '/path/to/page.tsx'
with open(path) as f:
    content = f.read()
content = content.replace('old_1', 'new_1')  # exact byte match
content = content.replace('old_2', 'new_2')
with open(path, 'w') as f:
    f.write(content)
PYEOF

# 4. Verify
npx tsc --noEmit  # → 0 errors
npm run build      # → clean

# 5. Review diff before committing
git diff HEAD -- file.tsx | head -80
```

## Why Python str.replace() Works When patch Fails

Python's `str.replace()` does exact byte-for-byte match on raw file content. `grep` shows text that IS in the file. If `patch` says "not found," the file has subtle differences (extra spaces, different quote types, encoding). Python's replace with exact repr output eliminates the ambiguity.

## Real Session Recovery (Aug 1 2026)

Six sequential patches → cascading corruption across 9 patterns. Recovery:
1. `git checkout HEAD -- file.tsx` — restored (1,120 lines)
2. Python script — all 6 changes in one call
3. `grep -n "const handleSavePreferences"` — found duplicate at lines 331-351 — removed
4. `AnimatePresence` indentation — surgical patch
5. TL dropdown state + JSX added
6. Final: `tsc` → 0 errors, `build` → clean
7. Committed as `014d8743` + `34ee5e57`
