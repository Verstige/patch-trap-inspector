# CSS replace_all corruption pitfall

**Symptom:** After calling `patch` with `replace_all=true` against a CSS file that contains multiple identical blocks, the resulting file is structurally broken. Look for these telltales in the post-patch diff:

- Broken class names at block boundaries, e.g. `.edu-module-quiz { ... }uiz { ... }` (the closing `}` of one occurrence runs into the opening `}` of the next, and the selector gets eaten)
- Orphan `}` lines hanging between otherwise valid rules
- Unrelated selectors that sat between the duplicated blocks **disappear entirely** (they got swallowed by the merge)

**Why it happens:** The patcher, when it sees multiple matches, replaces each match with the same `new_string` *in place*, but it does not deduplicate the merged content. If the merged region overwrites something nearby (or if `new_string` had a boundary character that fused with the surrounding text), the file's structural integrity is broken. The diff tool shows a misleading "successful" patch because the replacement text was applied, but it can't tell you the result is syntactically invalid CSS.

**Reproduction recipe:**
1. Open a CSS file with 2-3 identical rule blocks (e.g. shared `.card` rules, duplicated theme variables)
2. Call `patch` with `old_string` = the exact block, `new_string` = an expanded block, `replace_all=true`
3. Re-read the file — corruption appears between the third and fourth sibling blocks

**Recovery procedure:**
1. **Stop.** Do not attempt another `patch` until you've seen the full current file state.
2. `read_file` the entire file (offset/limit in chunks if it's long).
3. Look for the three corruption signatures above.
4. For each orphaned/duplicate region: `patch` with the duplicated block as `old_string` and an empty string as `new_string` (NO `replace_all`) to delete the duplicate cleanly.
5. For missing siblings that got swallowed: re-`patch` them in from the original source context (git diff is your friend here — `git diff <file>` against HEAD).
6. Type-check / build / lint to confirm restoration.
7. Add a memory entry or skill note so you don't repeat the mistake.

**Prevention (preferred):**
- Before using `replace_all=true` on a CSS file, `grep` for the target rule. If it appears more than once AND those occurrences are siblings that need to stay siblings, do **not** use `replace_all` — instead, patch each one individually with surrounding context to disambiguate.
- If the duplicates are truly identical and should become one block, first manually consolidate them with targeted non-`replace_all` patches, THEN use `replace_all=true` (which will only match the now-singular instance).
- Consider using `sed -i '' '/^\.rule {$/,/^}$/d' file.css` via terminal to dedupe identical CSS blocks before patching — but only when you're sure the blocks have *zero* contextual differences.

**Real session example (Aug 8 2026, the org OS education audit):**
- Triggered by expanding `.edu-module-quiz` to add hover/disabled states — 3 identical base blocks in `src/styles/pages/Education.css`.
- `replace_all=true` produced a 50-line diff with `}uiz {` and orphan braces, AND silently dropped `.edu-track-locked-note`, `.edu-module`, `.edu-module-head`, `.edu-module-head h4`, `.edu-module-quiz-info strong`, and the base `.edu-quiz-pill` rule.
- Recovery took 4 surgical patches (delete duplicate region, restore missing rules in correct order).
- Net cost: ~6 minutes that would have been ~30 seconds with `replace_all=false`.

**Rule of thumb:** If a CSS file has duplicate rule blocks, treat them as "there for a reason" (browser-specific, scope-specific, etc.) and patch each occurrence explicitly. Only use `replace_all=true` on CSS when you've confirmed the duplicates are truly fungible.
