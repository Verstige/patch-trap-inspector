---
name: patch-trap-inspector
description: "Inspect find-and-replace edits before they corrupt code. The workflow pitfalls of the patch tool -- fuzzy matches, replace_all, template literals -- and byte-level recovery steps."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# `patch` Tool — Workflow Pitfalls

## 1. Never `replace_all=true` with a generic pattern

`replace_all=true` replaces EVERY occurrence. If your `old_string` is something like `<p class="lead">\n      Everything below is current state of the platform on <code>main</code>, July 27, 2026.\n    </p>` and that paragraph appears on 8 different slides each with a UNIQUE preceding sentence, you will clobber all 8 unique leads with the same generic text.

**Rule:** if the pattern you want to edit is even slightly generic (a CSS class, a shared tag, a common phrase that the agent template uses in many distinct places), you must:

1. Check `grep -c` first to count matches. If it's >1, assume they are distinct and do NOT use `replace_all`.
2. Edit each match individually with surrounding context that uniquely identifies it (preceding title, eyebrow text, sibling element).
3. If the change is genuinely identical across all matches and the context is so stable you can prove uniqueness per slot, only then use `replace_all`.

## 2. The mixed-indentation corruption trap

The `patch` tool uses fuzzy matching (9 strategies) and tries to align whitespace between `old_string` and `new_string`. If `old_string` is indented at 6 spaces and `new_string` is indented at 8 spaces (or vice versa) the tool will preserve the original leading whitespace in some strategies and re-indent in others — producing diffs where every line is "shifted" by 2 spaces and the closing tags end up at the wrong depth. The file still parses but the rendered HTML is wrong and any sibling element that depends on the closing-tag position breaks.

**Rule:**

- Match the leading whitespace of `new_string` EXACTLY to the leading whitespace of `old_string`. Do not invent new indentation in the replacement.
- If the surrounding region uses a different indent (e.g. a `<section>` at 4 spaces, its `<div>`s at 6, content at 8), read those exact spaces and copy them — do not eyeball.
- After any `patch` call that targets a nested region, run `grep -c` on a distinctive sibling element to confirm the surrounding structure is still intact before the next edit.

## 3. Recovery after a bulk-replace clobbers unique sections

If you already made the mistake:

1. Stop. Do not run more patches before you have inventoried the damage.
2. `grep -n` for the clobbered pattern (the wrong generic text you just inserted). Count hits.
3. For each hit, read the lines just above and below to identify which slide/section it belongs to.
4. Patch each one with `old_string` = the now-clobbered generic text + enough surrounding context to be unique (preceding title, h1, h2, sibling element).
5. Verify with a final `grep` for any signature phrase from any restored paragraph to confirm all unique variants are back.

This is slower than getting it right the first time, but it is faster than rebuilding the whole file.

## 4. When editing HTML decks with repeating boilerplate

Vertical-scroll HTML decks use the same handful of patterns in many slides:

- `p.lead` for slide-introduction text
- `.eyebrow` for the slide's label/category
- `.panel.glow-*` cards
- `.pill.live / .pill.wip / .pill.todo` status pills
- `.rank-row.header` for table headers
- `.checklist li` and `.todo-list li` for item lists

Any of these as the SOLE old_string in `replace_all=true` is dangerous. Always include the slide-unique anchor (h1 title text, h2 section heading, sibling panel name) on both `old_string` and `new_string`.

## 5. Before any `replace_all=true` call — the pre-flight check

```bash
grep -c "<your pattern>" <file>
```

If the count is 1, safe. If it's >1:

- Look at each occurrence with `grep -n` and read the line context.
- Decide whether the change is truly identical across all of them.
- If yes → `replace_all=true` is fine, but copy the exact whitespace from one of the grep hits into your `new_string`.
- If no → edit individually with surrounding context per slot.

## 6. The clean pattern that prevents all of this

```python
# Pseudo — what you should be doing in your head before every patch call:
matches = grep(file, pattern)
assert len(matches) == 1, "Pattern is not unique — abort replace_all, edit each match"
patch(file, pattern, new_text, replace_all=False)
```

The `replace_all` flag is for the rare case where you genuinely want to change the same exact phrase everywhere (e.g. a brand-rename sweep on a string that is ONLY used in one meaning). It is NOT a shortcut for "this pattern shows up a few times."

## 7. `git add -A` will sweep up untracked files into your commit (added Jul 27 2026)

**Trigger:** You're done editing source files. You want to commit. You type `git add -A` and commit. Now you've committed 4 files of random untracked junk — old HTML decks, scratch notes, a one-off `the org-vision.html` from a previous session that was never gitignored.

**The rule:** before `git commit`, ALWAYS run `git status --short` and read every line. Anything that starts with `??` (untracked) is NOT supposed to be in your commit unless you intentionally added it. The 4 untracked files in your working tree are usually:
- Old HTML decks from prior sessions
- Test scripts
- Log files / debug outputs
- One-off analysis files

**The pattern that prevents this:**

```bash
git add -A
git status --short          # ← read this BEFORE committing
# If anything starts with '??' that you don't want, restore it:
git restore --staged <untracked-file>
git status --short          # ← read again, confirm clean
git commit -m "..."
```

**Real example (Jul 27 2026):** Twice in one session, I committed stray `the org-os-platform-deck.html` (1,252 lines, 61KB) and `the org-vision.html` (2,743 lines, 159KB) along with the actual feature work. Both times I had to `git reset HEAD~1 --soft` and re-stage. Cost: 2x wasted commits.

**Better pattern (long-term):** if you have stray untracked files you never want to commit, add them to `.gitignore` once. For the the target platform project specifically, the right `.gitignore` line is something like:

```
# Standalone HTML decks (not part of the build)
the org-*-deck.html
the org-vision.html
```

Add that to `.gitignore` once and you stop having this problem.

**Even better:** use `git add <specific files>` instead of `git add -A`. `git add -A` is for "I'm committing literally everything in the working tree" — that's almost never what you mean. Default to `git add <file> <file> ...` or `git add src/ supabase/migrations/` (path-scoped).

## 8. Patching files another agent has modified — the stale-read guard (added Jul 28 2026)

**Trigger:** Two agents are working on the same project in parallel. Agent A writes to `src/app/layout.tsx`. Agent B then tries to `patch` the same file.

**Symptom:** The `patch` call fails with:

> "_warning: ... was modified by sibling subagent 'XYZ' but this agent never read it. Read the file before writing to avoid overwriting the sibling's changes."

This is a feature, not a bug — it prevents Agent B from clobbering Agent A's just-written content.

**The fix is one extra read:**

1. `read_file(path, offset=N, limit=10)` — read the section you need to patch
2. `patch(path, old_string, new_string)` — now the read cache is fresh

**Why this happens:** `patch` keeps a per-session "last known content" hash for each file. If the file's on-disk content hash has changed since your last read, the patch refuses. The read re-syncs the hash.

**Same trap with `write_file`:** the curator-mode `write_file` (for skill support files) refuses to write if the target file's content hash hasn't been loaded yet — you'll see "current X content has not been loaded in this review turn." Same fix: `skill_view` the file first.

**Why it bites harder than you'd expect:** when you're working through a list of edits fast, you assume the file you patched 30 seconds ago is still the version you patched. But a sibling agent may have edited it in parallel. Without the stale-read guard, you'd silently overwrite their work. With it, you spend one extra `read_file` per affected file — worth it.

## 9. `patch` requires `path` on every call — the silent loop trap (added Jul 29 2026)

**Trigger:** You're using `patch` in `mode: 'replace'` for a series of edits. The JSON tool call has params `mode`, `old_string`, `new_string`, and `path`. You forgot `path`.

**Symptom:** Every single call returns `{"error": "path required"}`. The harness fires a `repeated_exact_failure_warning` after 2 identical failures, but the agent routinely burns 5+ calls before noticing — each one hits the same wall and the tool returns the same error string.

**Real cost (Jul 29 2026):** while wiring the Genesis Stripe cancellation hook into `flow_engine_connections`, the agent tried to `patch` 5 things in a row without `path`. The patcher returned 5 identical "path required" errors. The loop warning fired only AFTER the 5th identical failure. ~10 turns of wasted context.

**The root cause:** `patch` is overloaded — same tool for `replace` mode (find-and-replace) and `patch` mode (V4A multi-file patches). The `path` parameter is required for `replace` mode but often confused with the inline `patch` content for `patch` mode. When you're in `mode: 'replace'`, the `path` tells the tool WHERE to apply the edit. Without it, the tool has nothing to act on.

**The pattern that prevents this:**

```python
# ALWAYS include path as the FIRST invariant in your mental model for replace mode:
patch(
    mode='replace',          # ← picker
    path='/abs/path/to/file', # ← required (for replace mode)
    old_string='...',
    new_string='...',
)
```

**Three sanity checks before calling `patch`:**

1. Is `mode='replace'`? → `path` is **required**.
2. Is `mode='patch'`? → `path` is **forbidden** (the patch content IS the multi-file spec).
3. Did I just copy a `patch` template from a previous call? → explicitly retype `path` before sending.

**Recovery:** if you're already in the loop, stop, count the failures, then re-send EACH call with `path` added. Don't try to "force" the tool to retry — the loop warning is informational, not a recovery signal.

**Why this bites:** the JSON tool surface doesn't have a hard "required param" check at the client level. The agent code is happy to send a call without `path` and the server rejects it. There's no client-side feedback until the round-trip completes. So the loop is invisible until you specifically look at the error string.

## 9b. Mixing up `mode='replace'` vs `mode='patch'` parameter shapes — the silent loop trap (added Jul 31 2026)

**Trigger:** The `patch` tool has two distinct modes, each with its own parameter contract:

- **`mode='replace'`** — single-file find-and-replace. Args: `path`, `old_string`, `new_string`, optionally `replace_all`.
- **`mode='patch'`** — V4A multi-file patch. Args: `patch` (the multi-file spec content).

When you copy a previous patch call as a template, you can accidentally swap the parameter shapes. Sending `mode='replace'` with a `patch` V4A-style body, OR sending `mode='patch'` with `old_string`/`new_string` — both fail in different ways.

**Symptom (mode='replace' → wrong body):** The tool returns `{"error": "path required"}` repeatedly because the `old_string` is being interpreted as the V4A `patch` argument and the JSON shape is unparseable. The harness thinks every call is missing `path` even though the body is rejecting on a different parser. You burn 5+ turns.

**Symptom (mode='patch' as find-and-replace):** The `mode='patch'` invocation expects a V4A-formatted body. If you pass `old_string`/`new_string` instead, you get an error like `invalid format` or `path required` again.

**Real cost (Jul 31 2026):** while extracting the BottomDock from the trading page, the agent tried to `patch` `src/app/dashboard/trading/page.tsx` 6 times in a row. Each call used `mode='replace'` but the body was a V4A `*** Begin Patch` block with `*** Update File`, `@@ context hint @@`, `*** End Patch` markers. The harness returned `path required` for every call. The agent kept retrying the same V4A-block-with-mode-replace pattern instead of switching to `old_string`/`new_string`. ~6 turns of wasted context.

**The mental model that prevents this:**

```python
# For simple find-and-replace on a single file:
patch(
    mode='replace',
    path='/abs/path/to/file',   # ← required, this is the ONLY mode that takes path
    old_string='...',
    new_string='...',
)

# For multi-file V4A patches (toggle switches in different files at once):
patch(
    mode='patch',
    patch='*** Begin Patch\n*** Update File: path1\n@@ context hint @@\n...\n*** End Patch',
)
```

**The shape contract:**

| Mode | Required params | Body shape |
|------|-----------------|------------|
| `replace` | `path`, `old_string`, `new_string` | Plain text find-and-replace |
| `patch` | `patch` (the content) | V4A multi-file body with `*** Begin Patch` markers |

**The diagnostic for "path required" but you're passing path:** count the `{` and `}` in the `old_string`. If it contains `*** Begin Patch`, `*** Update File`, `@ context hint @` — you wrote a V4A body into a `replace` call. Switch to `mode='patch'` and send the V4A body as the `patch` parameter ONLY.

**Recovery:** once you realize you've been in the loop, **change the call shape entirely**, not just one parameter. Stop sending V4A-body into `mode='replace'`. Either: (a) reformat the body into plain `old_string`/`new_string` and keep `mode='replace'`, or (b) keep the V4A body and switch to `mode='patch'` with the body inside the `patch` parameter.

**The recovery rule** when you have sent 3+ identical failing patch calls in a row: the issue is not parameter tweaking — it's call-shape. Stop and re-read the tool's parameter contract before retrying.

## 9c. The repeating-loop pattern — mode/patch confusion across a long session (added Jul 31 2026)

**Trigger:** Long editing session with multiple `patch` calls per file, mixing `mode='replace'` and `mode='patch'`. You confuse the two parameter shapes repeatedly — even after reading the tool contract — and burn 4-6 tool calls in identical loops before catching yourself.

**Why this keeps happening even after section 9b:** the JSON parameter shape for the two modes is fundamentally different. When you copy a previous `patch` call as a template, you may copy the wrong template, AND the harness returns the same opaque error string for both shapes of confusion. The error string gives no signal that you have mixed up the modes — it just says "path required" because the harness cannot parse a V4A body as find-and-replace parameters.

**Real cost (Jul 31 2026):** while extracting `BottomDock` and `AppDrawerPopover` from the trading page, the agent hit this loop **4 separate times** in a single session — each time sending a V4A body with `mode='replace'`, getting "path required", retrying the same body 2-3 times, then finally noticing and switching to `mode='patch'`.

**Even-deeper failure mode (added Jul 31 2026 — same session):** the agent reads the warning text above ("you are sending a V4A body with `mode='replace'`") and **still sends the same V4A body again** 1-2 more times before finally switching. The reading didn't break the loop. The reason: once a JSON tool call template is in working memory, re-typing it (with a one-character edit hoping it'll work) is lower-friction than re-deriving it from the parameter contract. The agent keeps "fixing" one parameter at a time instead of re-deriving the call shape from scratch.

**The hard rule that breaks this:** **when the `repeated_exact_failure_warning` fires, stop and re-derive the call from scratch, do not edit-in-place.** Do not pass a slightly modified version of the failing call. Build the next call from three answers, not from the previous failing call:

1. What `mode` am I in? (`replace` for single-file find-replace, `patch` for V4A multi-file)
2. What does THIS mode require? (`replace` needs `path`+`old_string`+`new_string`; `patch` needs only the `patch` field)
3. What shape is the body in? (Plain code → `replace`. Has `*** Begin Patch` markers → `patch`.)

**Concrete rescue script — when you've burned 3+ identical failing patch calls in one session:**

```bash
# 1. Stop sending the failing call. Do not retry "one more time."
# 2. Read the last 2-3 patch invocations in your conversation history.
#    Look at the SHAPE of the body, not the parameter names.
# 3. If the body has '*** Begin Patch' or '*** Update File':
#      you are writing V4A → switch to mode='patch' and put the body in 'patch'
#    If the body has no markers and no leading prose:
#      you are writing find-and-replace → use mode='replace' with old_string/new_string
# 4. Send ONE call with the corrected shape. Do not re-test the old shape.
```

The reason this rescue is not just "stop and read the docs": in a long editing session, the docs (sections 9/9b/9c above) are visible but the agent pattern-matches on the last failing call rather than re-reading them. The rescue forces the agent to look at the *body shape markers* (`*** Begin Patch`), not the parameter contract names.

**The pattern that breaks the loop — pre-flight self-check before EVERY `patch` call:**

1. Stop. Read the JSON you are about to send. Look at the body field.
2. Does the body contain `*** Begin Patch`, `*** Update File`, or `*** End Patch` markers?
   - YES: switch to `mode='patch'`, put the body in the `patch` field, drop `old_string`/`new_string` and `path`.
   - NO: keep `mode='replace'`, ensure `path` is set, fill in `old_string` and `new_string`.
3. Look at the `path` field. Is it set?
   - YES: proceed (you are in `mode='replace'`).
   - NO: you are in `mode='patch'`, you do not need `path`.
4. Send the call.

**The mental shortcut — pre-built templates you copy from:**

For find-and-replace, copy-paste this skeleton and fill in 4 fields:
```
mode: replace
path: /ABS/PATH/TO/FILE
old_string: (the exact text to find)
new_string: (the replacement)
```

For V4A multi-file patches, copy-paste this skeleton and fill in 1 field:
```
mode: patch
patch: *** Begin Patch
*** Update File: /ABS/PATH/1
@@ context hint @@
context
-old
+new
*** End Patch
*** Begin Patch
*** Update File: /ABS/PATH/2
...
*** End Patch
```

**Hard guardrail — count the special markers in the body you just typed:**

- V4A `patch` body has `*** Begin Patch` and `*** End Patch` markers — see them → probably V4A
- Find-and-replace body is plain code, no V4A markers — see none → probably replace

## 21a. CSS comment with leading `{` — the silent CSS build breaker (added Aug 30 2026)

**Trigger:** You're editing a CSS file and use `patch` to add a section comment block. You accidentally prepend `{` to the start of the comment, making it look like a CSS comment-with-curlies — e.g.:

```css
{/* ─── Trade History (the org-styled rows) ─── */
/* Aug 30 2026 — Trade history is a fixed-height ... */
.history-list {
    ...
}
```

**The bug:** CSS sees the leading `{` and tries to parse a rule with **no selector**. The CSS parser tolerates this up to a point, but Turbopack/Next.js (and modern PostCSS pipelines) fail with:

```
Parsing CSS source code failed
Invalid empty selector
```

**Why it's silent:** the build error points at the comment line itself, which looks completely innocent. There is no broken rule nearby — the bug is the comment itself. Developers waste cycles hunting for "what changed in the CSS that's broken" when nothing is broken except the comment marker.

**Why this happens with `patch`:** when copying a block of CSS that was previously wrapped in `{ ... }` (a media query body, a nested selector, or a `@layer` block), and the agent wants to "promote" the inner content to top-level, they may leave a stray `{` in the `old_string` and forget to remove it from the `new_string`. Or the reverse: they copy a comment template from another section that had `{ ... }` around it.

**Real cost (Aug 30 2026, the target platform dashboard):** A single `patch` to add a CSS comment block above `.history-list` introduced `{/* ─── Trade History ... */}`. The build error was `Invalid empty selector at line 952:1`. Build failed. **Railway silently rolled back to the last working deploy** while the git history was up to date. The user reported "I'm still seeing the same thing" — the live site was the OLD code because the deploy never succeeded. The diagnosis cost an extra round-trip because the local `node_modules` build was hidden in `.next/` from a previous run and the agent didn't re-build.

**The fix:**

```css
/* ─── Trade History (the org-styled rows) ─── */
.history-list {
    ...
}
```

Drop the leading `{` before the `/*`.

**The diagnostic when "the deploy looks like it didn't happen":**

1. Verify the latest commit is on `origin/main` (`git log origin/main..main` and `git ls-remote origin main`).
2. If the commit is pushed, run `npx next build` LOCALLY. Railway runs the same `next build` and silently rolls back on failure. The local build will reproduce the failure immediately.
3. Common silent build breakers (in order of how often I've hit them):
   - **CSS comment with leading `{`** (this section)
   - Corrupted Unicode (null-byte ellipsis `…`, smart quotes) — see §13
   - Duplicate CSS rule block (left from an earlier patch iteration)
   - Unclosed selector or property
4. The `npx tsc --noEmit` check does NOT catch CSS issues. You MUST run `npx next build` (or `npm run build`) before trusting a deploy.

**The pre-flight self-check before any CSS patch:**

```
Does my new_string start with `{/*`?
  → YES: I am wrapping a comment in curlies. CSS doesn't allow this.
         Drop the leading `{` from new_string.
  → NO: proceed.

Does my new_string introduce a duplicate of an existing rule?
  → Run `grep -c '<rule-selector>' <file>` before and after the patch.
     If the count goes from 1 to 2, I duplicated a block. Remove one.
```

**The build verification recipe (mandatory before pushing a CSS change):**

```bash
# 1. Run the build locally — it takes ~30s for a medium Next.js project
npx next build 2>&1 | tail -20

# 2. Check the exit code — non-zero means build failed
echo "exit=$?"

# 3. If failed, look for the FIRST error, not just the last line
npx next build 2>&1 | grep -E "Failed to compile|Error:|Invalid" | head -5

# 4. Common CSS-specific errors and what they mean:
#    "Invalid empty selector"     → leading `{` before comment, OR orphaned `{` from a half-removed media query
#    "Expected selector"           → unclosed `{` from a previous patch
#    "Unterminated string"          → stray `'` or `"` in a content value
#    "Unknown word"                 → typo in a property name
```

**Why the build verification matters more than tsc for CSS work:** TypeScript-only changes are safe to verify with `tsc --noEmit`. CSS changes REQUIRE a full `next build` because:
- CSS Modules and PostCSS transformations only run during the build
- Turbopack's CSS parser is stricter than dev-mode CSS loaders
- A CSS build failure rolls back the entire deploy, even if the rest of the app is fine

The cost of skipping local build verification: silent rollback in production, user sees stale UI, you spend a debugging round-trip figuring out "why is the live site not updating" when actually it never updated.

**The Railway-specific pitfall:** if `railway.toml` has `buildCommand = "npm run build"`, the build is the only validation. A failed build means the LAST WORKING build keeps serving. The git log is up to date but the deployed code is from N commits ago. This is silent — there's no error message, no Slack notification, no Railway UI alert (unless you check the dashboard's "Deploys" tab manually).

**Hard rule:** before any CSS patch, run `npx next build` LOCALLY first. If it succeeds locally, push. If it fails, the live deploy will also fail and the user will see stale UI for as long as the broken commit is on `main`.

**Companion to §14 (sequential patches collapse):** §14 is about JSX cascading errors. This section is about CSS silent-failure modes. Both can be prevented by running the full build before pushing, but the build-error patterns are different (TSX cascading vs CSS parse errors).

**Companion to §17 (template literal corruption):** when the patch tool repeatedly corrupts template literals on the same file and `write_file` is too heavy, use Python `bytes.replace` with the exact broken + fixed byte sequences. See `references/template-literal-bytes-recovery.md` for the full recipe (od -c diagnostic, shell-quoting gotchas, when to use this over `write_file`).

---

## 21. Long `git commit -m` messages — the shell arg-list limit (added Aug 16 2026)

**Trigger:** You're writing a long, descriptive commit message — multi-paragraph, with bullets, rationale, list of files touched. You pass it as the `-m` argument to `git commit`.

**Symptom:**

```
/bin/bash: line 47: /usr/local/bin/git: Argument list too long
```

The shell refuses to exec `git commit` because the combined `-m` argument length exceeds the kernel's `ARG_MAX` (typically 128KB on macOS, 2MB on Linux — but bash's argv handling makes the practical limit much lower for commands receiving arguments via `-m`).

Or the call may appear to succeed but `git commit` warns:

```
warning: commit message body has a wide character (> 4 KB)
```

**Real cost (Aug 16 2026):** I tried to commit the public changelog page with a 1.8KB multi-paragraph message via `git commit -m "..."`. Bash returned "Argument list too long" on the first attempt. The fallback (writing the message to `/tmp/commit_msg.txt` and using `git commit -F <path>`) succeeded. ~2 round-trips of churn.

**The fix — `git commit -F <file>` for any message > ~1KB:**

```bash
# 1. Write the message to a file
cat > /tmp/commit_msg.txt << 'EOF'
feat(scope): short subject line under 72 chars

Multi-paragraph body explaining what changed and why.
Each commit message can be ~100 lines if needed.

- Bullet points
- List of files
- Anything else you want to capture
EOF

# 2. Commit using -F (file) instead of -m (string argument)
git commit -F /tmp/commit_msg.txt
```

**Why `-F` works when `-m` doesn't:**

- `-m "string"` — the entire string is one argument to `git commit`. The shell must fit it (plus all other git args) into the kernel's argv limit. Multi-line strings with newlines can also confuse shell escaping (heredocs vs. quoted strings vs. inline `\n`).
- `-F <file>` — git reads the message from the file directly. No shell argument expansion, no argv limit, no escaping issues. The file can be arbitrarily large.

**When to use `-F` vs `-m`:**

| Message length | Method | Why |
|---------------|--------|-----|
| < 500 chars, single line | `git commit -m "..."` | Fine |
| < 2 KB, simple multi-line | `git commit -m "$(cat <<'EOF' ...\nEOF)"` or `-F <file>` | Either works, `-F` is safer |
| > 2 KB, multiple paragraphs | `git commit -F /tmp/commit_msg.txt` | Required — `-m` will hit arg-list limit |

**The bash heredoc + `-F` pattern (use every time):**

```bash
# Save the message first
cat > /tmp/commit_msg.txt << 'EOF'
subject line under 72 chars

body
EOF

# Run the commit with file-based message
git commit -F /tmp/commit_msg.txt

# Verify the message was applied
git log -1 --pretty=format:"%h %s%n%b" | head -5
```

**Companion to §7 (`git add -A` risk):** both are about preventing wasted commits. §7 prevents accidentally committing junk files; §21 prevents the commit command itself failing due to message length. Both cost ~30 seconds to fall back from when caught, but are cheap to avoid with the right pattern.

## 21b. Apostrophes / parens inside `git commit -m "..."` — the bash quoting trap (added Sep 23 2026)

**Trigger:** You're crafting a `git commit -m` message inside an `execute_code` shell call (`bash -c "..."`). The message contains a real apostrophe (e.g. `"couldn't load"`, `"user's status"`) or unbalanced parens (e.g. `"Reason (HTTP 401)"`). You write it as `git commit -m "..."`.

**Symptom:**

```
bash: -c: line 0: syntax error near unexpected token `('
bash: -c: line 0: `cd ~ maintainer/Projects/... && git commit -m 'feat: ... Couldn'\\''t load (HTTP 401). ...' ...'
```

Bash interprets the inner apostrophe as the end of the single-quoted string, then chokes on the next token (`(` or `)`). The `execute_code` shell call returns `exit_code=2` with no commit made.

**Why the obvious fix doesn't work:** switching the outer `-m "..."` to `-m '...'` just moves the same problem one layer in — your apostrophe inside the message still terminates the outer single-quoted string. `\`-escaping inside double quotes doesn't help. `\n` literals look like newlines in git but break shell parsing.

**Real cost (Sep 23 2026, the target platform algo-os):** I tried `git commit -m '... Couldn'\''t load your subscription status ...'` with a 1.1KB message that contained two apostrophes ("Couldn't load", "user's") and a parenthetical `(HTTP 401)`. Bash returned the syntax error. The fallback — write the message to `/tmp/commit_msg.txt` with `write_file`, then `git commit -F /tmp/commit_msg.txt` — succeeded on the first try.

**The fix — same as §21: never pass multi-line / shell-hostile text via `-m`. Use `-F <file>`:**

```python
# In your execute_code call:

# 1. Write the message to a temp file using write_file
#    (write_file handles all the escaping for you; you write the literal message)

# 2. Run the commit via terminal / subprocess, NOT via execute_code's bash -c
import subprocess
subprocess.run(['git', 'commit', '-F', '/tmp/commit_msg.txt'], cwd='~ maintainer/Projects/...')
# OR in execute_code:
r = subprocess.run(['bash', '-c', 'cd ~ maintainer/Projects/... && git commit -F /tmp/commit_msg.txt'], ...)
```

**Why `-F <file>` also fixes the apostrophe trap:** the message lives in a file, not in the bash command line. Bash never sees the apostrophes — it just sees `git commit -F /tmp/commit_msg.txt`. All the quoting/escaping happens at the file-write step, which `write_file` handles for you.

**The pre-flight check before any `git commit -m` call with arbitrary text:**

```
Does my message contain any of: ' " ( ) \ $ ` ?
  → YES: use `git commit -F <file>` instead. Don't even try -m.
  → NO: -m is safe.

Is my message > ~500 chars?
  → YES: use -F regardless. See §21 for the arg-list limit.
  → NO: -m is safe.
```

**Why this lives next to §21 instead of §23 (the patch-tool validator trap):** §23 is about the patch tool's validator rejecting payload fields. This section is about the shell parsing the `bash -c "..."` argument that wraps any `terminal`/`execute_code` call. The bug is in different layers but the workaround is identical — promote the problematic text to a file.

**Companion to §21:** §21 covers long messages hitting `ARG_MAX`. This section covers short messages with shell-hostile characters. Both fail at the `bash -c` layer. Both are solved by `-F <file>`. Default to `-F <file>` whenever the message has any chance of being long or containing punctuation.

## 22. `npm run build` fails but `tsc --noEmit` passes — the "corrupt middle" pattern

This is a specific failure mode when editing JSX/TSX files with sequential patches:

- `npx tsc --noEmit` → **0 errors**
- `npm run build` → **fails** with parse errors ("Unterminated regexp literal", "Unexpected token", "Expected closing tag")

**Root cause:** a null-byte corrupted character or structural tag imbalance that TypeScript tolerates but Turbopack's ECMAScript parser rejects.

**Key diagnostic:** Turbopack reports the error at a syntactically innocent line AFTER the actual corruption. The reported line looks fine. Scan backward 50-150 lines for any string literal with special characters (ellipsis `…`, smart quotes, em-dashes).

**Why it happens:** TypeScript's parser is more forgiving; Turbopack is stricter. TypeScript handles malformed strings gracefully; Turbopack throws.

**Real example (Aug 1 2026):** Turbopack reported "Unterminated regexp literal" at line 931. Line 931 was fine. The corruption was a null-byte ellipsis at line ~812 (`Loading accounts…` → `Loading accounts[M-^@][M-^@]`).

**The rule:** when `tsc` passes but `npm run build` fails on a TSX file, do NOT look at the build error's line number — run `tsc` and fix whatever it reports, then rebuild. If `tsc` passes and the build still fails, suspect a byte-level corruption (run `cat -v | grep "M-\^@"`).

1. Stop sending the failing call. Do not retry "one more time."
2. Read the actual JSON of the last 2-3 calls you sent (look at the conversation history).
3. Identify which field is wrong: is it the `mode`? The `path`? The body shape?
4. Send ONE call with the correct shape. Do not re-test the old shape.

**Why this is different from sections 9 and 9b:** those describe the trap in isolation (single bad call). This section describes the **pattern of repeating the same mistake across a long session** despite having read the warnings. The defense is not "remember the rule" — it is "have a self-check that takes 3 seconds and breaks the loop before the harness fires the warning."

**Adjacent skill: `nextjs-dynamic-tabs/the navigation state pattern` documents the URL-vs-state mismatch that often requires multi-file patches — when extracting a nav into a persistent layout component, expect to use `patch` mode for 3-5 files at once. That is exactly the kind of session that triggers this repeating-loop pattern.**

## 10. Removing an `import` that sits above a same-named local component (added Jul 29 2026)

**Trigger:** A component file has both a named import AND a locally-defined function/class with the SAME name. You remove the import (because it's unused) but the old_string pattern accidentally captures the locally-defined component that immediately follows it — deleting the component body and leaving only the import removed.

**Real cost (Jul 29 2026):** `PartnerPortal.tsx` had both `import CommissionCalculator from './CommissionCalculator'` (line 4) and a locally-defined `function CommissionCalculator({ ... })` (line 774). The patch target was the import. The old_string included the import AND a few lines of the local MiniCell function that happened to appear immediately after in the same patch batch — accidentally deleting MiniCell. Result: `npm run build` failed with `Cannot find name 'MiniCell'` at 3 call sites. Had to restore MiniCell manually.

**The rule:** before removing any import, run `grep -n "import.*<Name>\|function <Name>"` to confirm whether a local definition with that name exists. If the same name appears both as an import AND as a locally-defined function/class in the file, removing the import will break the build — even if the function itself remains, TypeScript will now complain about a missing local reference at the call sites.

**Better: when removing a duplicate import, replace it with empty string — never include adjacent function/class definitions in the same patch.**

## 11. Patches inside patches — the indentation drift trap (added Jul 30 2026)

**Trigger:** You're editing the same file with multiple sequential `patch` calls. The first patch adds new code at one indentation level. The second patch's `old_string` matches the surrounding text — but you copied the `old_string` from your mental model of the *new* code state, and your `new_string` accidentally uses the indentation of the original (pre-patch-1) section. The `patch` tool's fuzzy alignment re-indents everything in between to "fix" the mismatch — leaving a section with mixed 4-space and 8-space indentation that no longer parses.

**Symptom:** the patch's diff looks clean in the response. But the file now has a section like:

```ts
function example() {
    if (cond) {
        new code at 8 spaces
  old code that should still be at 4 spaces   // ← drift
    }
}
```

The file is syntactically invalid. The next `npm run build` fails with cascading errors (TS1005, TS1128) pointing deep inside the broken section. **Worse: the broken code may not be in your patch's old_string or new_string** — it's in the *surrounding* lines that got re-indented to "match" the new content.

**Real cost (Jul 30 2026):** while editing `src/context/TradingContext.tsx`, two sequential `patch` calls produced a section where the surrounding code was re-indented to match an `old_string` whose indentation didn't match the actual file. Took three more patches to fix (each one re-aligned a different part of the now-broken section). The trade-off: the patches worked individually but accumulated drift across the multi-edit sequence.

**The rule — read the file between sequential patches on the same file:**

After any `patch` call that targets a region inside a function/block, read the file's surrounding section BEFORE making the next patch in the same area. Verify:

- Surrounding closing braces are at the right depth
- Sibling code blocks are indented at the right level
- The structure you think exists actually exists in the file

If you can't do an extra read between patches, prefer `write_file` (full rewrite) over multiple `patch` calls when the changes are non-trivial. `write_file` forces you to see the whole file at once and eliminates indentation drift.

**Pre-flight check between sequential patches:**

```bash
# After each patch, run grep on a unique nearby signature:
grep -c '<unique-sibling-marker>' <file>
# If the count changes unexpectedly, the surrounding structure was disturbed
# — read the region before making another patch.
```

This is the "verify the surrounding region is still intact" rule from §2, but applied between patches in the same editing session, not just after one patch.

## 12. Sequential JSX patches without re-reading — compounding corruption (added Aug 1 2026)

**Trigger:** Removing 3+ distinct JSX blocks from the same component with sequential patches. Each edit modifies surrounding indentation and closing-tag structure. After 4-6 patches: duplicate `</div>`, dangling `}`, corrupted string literals. TypeScript (`tsc --noEmit`) passes clean; `npm run build` (Turbopack) fails with "Unterminated regexp literal" — but the reported line looks fine. The real corruption is earlier.

**Why TypeScript passes but Turbopack fails:** TypeScript's parser is more forgiving; Turbopack's ECMAScript parser is stricter and surfaces the error at a misleading location.

**Real cost (Aug 1 2026):** Six sequential patches on `settings/page.tsx` (1,100 lines) to remove MT5 token + compact layout toggle produced: extra `</div>` in Notifications tab, dangling `}` from removed handler, null-byte corrupted ellipsis (`Loading accounts…` → `Loading accounts[M-^@][M-^@]` in `cat -v` output), and an extra `</div>` in the Trading tab. Turbopack failed at line 931 — the reported line was fine; the corruption was ~120 lines earlier.

**Recovery recipe:**

```bash
# 1. Run build to see the error and reported line
npm run build 2>&1 | tail -15

# 2. Read around the reported line
sed -n '925,935p' file.tsx

# 3. Check for non-UTF8 bytes — null bytes show as "00 00" in xxd output
sed -n '810,815p' file.tsx | xxd | tail -20

# 4. Count div balance in the broken region
awk 'NR>=800 && NR<=950 {o+=gsub(/<div/,"&"); c+=gsub(/<\/div/,"&")} END{print "open:",o,"close:",c}' file.tsx
# If closings > openings, there's an extra closing tag

# 5. Fix: sed/awk for diagnosis, then minimum surgical patch
```

**Prevention:** When a file needs 3+ distinct JSX structural changes, read the full file once and use `write_file` for a clean one-call rewrite. Sequential `patch` is safe only when: (a) edits are in non-overlapping regions separated by hundreds of lines, AND (b) you've re-read the file between calls, AND (c) no repeating identical boilerplate nearby.

## 13. Corrupted Unicode characters in source files (added Aug 1 2026)

**Trigger:** `patch` old_string doesn't match even though `grep` shows the text. TypeScript passes, Turbopack fails with "Unterminated regexp literal" on a line that looks correct. Root cause: a multi-byte UTF-8 character (ellipsis `…`, smart quotes `""`, em-dash `—`) was corrupted during a prior edit, encoding as a null byte.

**Real example (Aug 1 2026):** The ellipsis U+2026 in `Loading accounts…` was corrupted to null bytes — `cat -v` showed `Loading accounts[M-^@][M-^@]`. Turbopack misparsed the JSX; TypeScript tolerated it.

**Detection:**

```bash
# See non-printable/corrupted characters
sed -n 'L,P' file.tsx | cat -v
# Byte-level precision
sed -n 'L,P' file.tsx | xxd | tail -20
# Null bytes: "00 00"; valid UTF-8 ellipsis: "e2 80 a6"
```

**Fix:** replace the corrupted line with the correct plain UTF-8 version.

**Prevention:** Avoid pasting Unicode special characters (ellipsis `…`, smart quotes `""`, em-dash `—`) from AI tool output directly into source files — encoding cycles can corrupt them. Use ASCII equivalents (`...`, `"`, `--`) when in doubt.

## 14. The "sequential patches collapse" — git-restore recovery when the working tree is unrecoverable (added Aug 1 2026)

**Trigger:** You've made 6+ sequential patches to the same large JSX file (800+ lines). Build is failing with cascading errors (TS17008, TS1381, TS1005, TS17002) at multiple locations. TypeScript (`npx tsc --noEmit`) passes; Turbopack fails. Errors have moved across 30+ line ranges between attempts. You've fixed one error and three more appeared. The working tree has 6 patches applied and the structure is too corrupted to patch back to health one fix at a time.

**Real cost (Aug 1 2026):** Six sequential patches on `settings/page.tsx` (1,120 lines) to remove MT5 token + compact layout toggle + rename "Telegram Broadcasts" produced: extra `</div>` in Notifications tab, dangling `}` from removed `handleGenerateApiKey` handler, null-byte corrupted ellipsis in `Loading accounts…`, duplicate Partner card block, wrong `AnimatePresence` wrapper indentation (`<motion.div>` at same indent as `<AnimatePresence>` instead of nested inside), and outer `</DashboardShell>` closing tag accidentally removed. Each fix created a new error. Errors cascaded. The working tree could not be repaired patch-by-patch.

**The recovery recipe:**

```bash
# 1. STOP — do not send more patches
# 2. Save the working tree state
cp file.tsx /tmp/broken_file.tsx

# 3. Restore the last known-good committed version
git checkout HEAD -- src/app/dashboard/trading/settings/page.tsx
npm run build  # confirm it passes

# 4. Write ONE Python script that applies ALL changes at once
# Read /tmp/broken_file.tsx to identify exact patterns to change
# Python str.replace() on raw file bytes — exact repr match, no ambiguity

# 5. Run the Python script — one call, all changes applied
python3 apply_all_changes.py

# 6. Verify
npx tsc --noEmit  # should pass
npm run build      # should pass
git diff HEAD -- file.tsx | head -50  # review before committing
```

**Why Python str.replace() works when patch fails:** Python's `str.replace()` does an exact byte-for-byte match on the raw file content. `grep` shows text that IS in the file. If `patch` says "not found," the file has subtle differences (extra spaces, different quote types, encoding). Python's `replace()` with the exact repr output eliminates the guesswork.

**The single-script principle:** when a file needs N distinct structural changes (N≥3), write one Python script that reads the file, makes all N changes in sequence, and writes it back. This avoids cascading errors because you're never editing a file that is already in a partially-corrupted state.

**git-stash diagnostic when unsure:**
```bash
git stash   # save working tree
npm run build  # if this fails → committed version itself is broken
git stash pop  # restore working tree
# If committed builds → the problem is in your working tree changes
```

**Hard rule (Aug 1 2026):** when you have made 4+ sequential patches to the same file and the build is still failing, the correct response is NOT "one more patch" — it is "restore from git, write a Python script, apply once." The cost of restoring and reapplying is lower than the cost of one more cascading-error patch.

## 15. LSP diagnostics are stale after a successful patch

After a `patch` call reports `files_modified: ["path"]`, the LSP diagnostics embedded in the tool result are from the state BEFORE the patch was applied. They reference old line numbers and old error conditions. They are stale and will mislead you.

**The trap:** you fix something, the patch succeeds, the LSP in the result says "Cannot find name 'X' at line Y" — you then make another patch to fix that error, which creates a new problem.

**The rule:** after ANY successful `patch` call on a file with existing errors, run `npx tsc --noEmit` in a SEPARATE terminal call to get the true current state. Do not trust the LSP output embedded in the patch result.

## 16. Line numbers shift after each patch — always re-verify

Each successful `patch` call shifts subsequent line numbers. An error at "line 436" after patch 3 is at a different physical line after patch 4. The only reliable error location is from a fresh `tsc --noEmit` run — not from any previously-reported error location.

## 17. Template literal corruption — backticks and `${...}` interpolation get eaten by fuzzy match (added Aug 10 2026)

**Trigger:** You're patching a TypeScript file that contains a template literal with `${variable}` interpolation. Your `old_string` includes the template literal as the matching pattern, and your `new_string` also includes a template literal. The `patch` tool's fuzzy match sometimes **silently drops the leading backtick or corrupts the interpolation**, producing output like:

```ts
// Before (correct)
const tlHeaders = {
    Authorization: `Bearer ${tl_access_token}`,
    accNum: String(accNum ?? 1),
}

// After a corrupted patch (what the patch tool actually wrote)
const tlHeaders = {
    Authorization: *** ${tl_access_token}`,
    accNum: String(accNum ?? 1),
}
```

**The corruption:** the leading backtick (`` ` ``) gets dropped from the substitution. The variable interpolation `${...}` survives, but the result is a syntax error (TS1109, "Expression expected") that Turbopack surfaces as a build failure. The diagnostic line reads `Authorization: *** ${tl_access_token}` — innocent-looking except for the missing backtick.

**Root cause:** the `patch` tool uses fuzzy matching across 9 strategies. When `old_string` and `new_string` differ in a single character (like the leading backtick), and the surrounding text is otherwise identical, one of the fuzzy-match heuristics can substitute a partial match. This is more likely when:

- The template literal contains regex-special characters (`$`, `{`, `}`)
- The line is long (>80 chars) and appears multiple times in similar contexts
- The match is buried inside a function body with deep indentation

**Real cost (Aug 10 2026, the target platform M1 terminal session):** Three separate `patch` calls in one session each corrupted a template literal the same way. Each time the agent had to read the file, find the broken section, and patch it back to correct syntax. ~3 round-trips of churn per corruption, all of them preventable.

**The fix — copy backticks explicitly when constructing `new_string`:**

```python
# Mental model before EVERY patch on a TS file with template literals:

# 1. Read the exact line you're targeting, including all whitespace
# 2. In new_string, copy the leading backtick BYTE for BYTE
#    - Don't type ` ` from memory. Read it from the file.
#    - Don't escape it as \` — the patch tool reads raw, not escaped.
# 3. After patching, immediately re-read the file to verify
# 4. If new_string has `${...}`, also copy the `${` literal — the
#    `$` IS NOT the same character as the shell-escaped `\$`
```

**The verification recipe (run after EVERY patch with template literals):**

```bash
# Immediately after a successful patch on a TS file:
grep -n '\`.*\${' <file>          # confirm template literals are still intact
# Look for: lines starting with bare backticks followed by content
# Bad pattern: "Authorization: *** ${..." (missing leading backtick)
# Good pattern: "Authorization: `Bearer ${...}"

# tsc is also fast and will catch this:
npx tsc --noEmit 2>&1 | grep <file>
```

**When this happens in a large block of changes** — if the broken template literal is in a region you just rewrote with multiple consecutive `patch` calls, the corruption could be in any of them. The cheapest recovery is:

1. `git diff HEAD -- <file>` — see exactly what the patch calls wrote
2. For each `new_string` that contained a template literal, check the diff
3. Restore the missing backtick via a single targeted `patch`

**The pre-flight self-check before ANY patch with template literals:**

```
Does my new_string contain a backtick (`)?
  → YES: count the backticks in old_string and new_string.
         They MUST match. If off by one, the patch tool WILL
         corrupt something.
  → NO: verify the line I'm replacing didn't HAVE a template
        literal in the old_string that I accidentally dropped.

Does my old_string end with `\n` and my new_string does too?
  → YES (good).
  → NO (one of them missing): fuzzy match will likely pick the
    wrong strategy and you get surprise indentation drift.
```

**Companion to §2 (mixed-indentation corruption):** both are caused by the same underlying issue — the patch tool's fuzzy alignment heuristics picking the wrong match strategy when the strings are *almost* identical but differ in a single character. The fix in both cases is the same: match the exact whitespace and special characters byte-for-byte.

**Real diagnostic recipe when you suspect template literal corruption:**

```bash
# 1. Find all template literal sites in the file
grep -n '`' <file> | head -20

# 2. For each match, manually read the surrounding 3 lines
sed -n '173,175p' <file>

# 3. Look for the bad pattern: "Authorization: *** ${" or similar
#    (missing leading backtick — substitution got truncated)
grep -n '\*\*\* \${' <file>
# This should return zero results in a healthy file.

# 4. Verify by counting backticks per logical region:
awk '/Authorization:/,/accNum:/' <file> | grep -o '`' | wc -l
# Should be an even number for properly-opened template literals.
# Odd = unbalanced / missing backticks.
```

## 19. After 3+ repetitions of the same template-literal corruption in one session, STOP patching and rewrite the file (added Aug 11 2026)

**Trigger:** You are editing a TS file with template literals. The patch tool corrupts the leading backtick of `${...}` interpolation. You fix the corruption with another patch. The next patch on the same file also corrupts the template literal. **After the third repetition, STOP.** The patch tool has a reproducible failure mode for this file shape — keep trying and it will keep failing.

**Real failure (Aug 10–11, 2026, M1 trading terminal session):** The agent hit template-literal corruption on `Authorization: \`Bearer ${tl_access_token}\`` three times in one session across the same file (`src/app/api/the broker/history/route.ts`). Each time the agent:

1. Patched the file with the leading backtick in `new_string`
2. The patch tool dropped it on write
3. tsc reported TS1109 (Expression expected)
4. Agent re-read the corrupted section, patched again with the backtick, same corruption, repeat

Three cycles. Each time the diagnosis was correct ("the patch dropped the backtick") and the fix was correct ("put it back"), but the patch tool reproduced the same failure mode because the surrounding text was the right length for the same fuzzy-match heuristic to fire.

**The right response after the second corruption** (not the third — the second):

1. **Stop patching the file.** The patch tool has a reproducible failure mode for this file shape. Don't keep poking it.
2. **Switch to `write_file` with the full corrected content.** Read the file, fix the corruption, write the entire file in one call. `write_file` doesn't use fuzzy matching, so it can't drop the backtick.
3. **Verify with `tsc --noEmit` and grep for the corruption signature** (`*** ${`) before moving on.

**The right response after the third corruption** (the one the agent should have done):

1. **Roll back to the last known-good committed version** of the file with `git checkout HEAD -- <file>`.
2. **Apply ALL intended changes in one Python script** that does raw `str.replace()` on the file bytes, with the exact correct template literal written once at the start of the script.
3. **Run `tsc --noEmit` to verify.**
4. **Then commit one clean version of the file.**

**The rule — the 1-2-3 escalation:**

- 1 corruption → patch the corruption. Normal flow.
- 2 same corruptions → switch to `write_file` of the whole file. Don't patch the corruption a second time.
- 3 same corruptions → git checkout HEAD, Python str.replace() the entire change set in one script. Don't even use `write_file` at this point — the file is in a corrupted state and you're not sure exactly what got corrupted where.

**The diagnostic — count the corruption signature in your session history:**

```bash
grep -c '\*\*\* \${\|<file>.*[Tt]emplate' <your-session-output>
# >3 = the patch tool has a reproducible failure mode for this file shape
# Treat the next corruption as a hard stop on patch usage
```

**Why this matters more than it looks:** each corruption-and-fix cycle costs ~30 seconds + 1 tool call. Three cycles is 90 seconds and 6 tool calls for ZERO net progress (the file goes broken → fixed → broken → fixed → broken → fixed). The agent spent those 90 seconds thinking it was making progress. Meanwhile the file was repeatedly corrupted, "fixed", re-corrupted. A single `write_file` at the 1-corruption mark would have prevented all three cycles.

**Companion to §14 ("sequential patches collapse"):** §14 is about cascading errors from JSX edits. §19 is about a specific patch-tool failure mode (template literal corruption) that recurs in the same session. The recovery recipe is the same — git checkout HEAD, write the whole file, run one Python script if needed — but the trigger and the diagnostic are different.

**The pre-flight self-check before the 2nd patch on the same file:**

```
Has this file already corrupted a template literal in this session?
  → YES: switch to write_file of the whole file. Don't patch again.
  → NO:  proceed with patch, but run grep '\*\*\* \${' on the file
         immediately after to catch the corruption if it happens.
```

## 19b. "tsc clean" claimed without reading the actual output — the verification-result blind spot (added Aug 11 2026)

**Trigger:** You're about to commit a multi-file change. You run `npx tsc --noEmit`. The tool returns — but you don't carefully inspect the output. You claim "tsc clean" or "zero errors" based on the *appearance* of a quiet tool call, then push.

**Real failure (Aug 11 2026, M1 + C/E/F batch):** During the trading terminal audit session, I ran `npx tsc --noEmit` after a multi-file patch series, claimed the output was zero errors, and pushed commit `5d5c3af3`. The user later tested the build and found TypeScript errors at runtime (`Duplicate identifier 'listBrokers'` in `AccountConnectionModal.tsx`). The tsc run had been returning errors — but I'd read "tool returned successfully" as "no errors" without inspecting the actual error lines in the output.

The same session showed three different ways this blind spot manifests:

1. **Trusting the tool's success state over its content** — `tsc --noEmit` returned with exit code 0 in some invocations and non-zero in others; I treated both as "looks fine, moving on."
2. **Reading truncated output** — the tool result was large; I scanned the first/last lines and assumed the middle was clean.
3. **Trusting LSP diagnostics inside a `patch` result over a fresh tsc run** — the LSP errors shown in the patch tool output are stale (from before the patch); the freshly-run tsc was the only authoritative source, and I didn't actually look at it.

**The rule that prevents this:**

> **Do not say "verified clean" until you have either (a) read every line of the tsc output, or (b) captured the full output to a file and grep'd it for `error` / `Error` / `error TS` patterns.**

Both of these are cheap — they add ~2 seconds to your workflow and they prevent shipping broken code. The cost of NOT doing either is pushing a commit you have to roll back or hot-patch.

**The pre-push self-checklist (use before every `git push`):**

```bash
# 1. Run tsc fresh, capture full output to a file
cd ~ maintainer/Projects/<project> && node_modules/.bin/tsc --noEmit 2>&1 | tee /tmp/tsc-output.log | tail -5

# 2. Grep the saved file for errors
grep -E "error TS|TS\d{4}|Duplicate identifier" /tmp/tsc-output.log | head -20
# Empty output = clean. Any hits = DO NOT PUSH.

# 3. ALSO confirm exit code explicitly:
node_modules/.bin/tsc --noEmit 2>&1; echo "exit=$?"
# exit=0 means zero errors. Anything else = DO NOT PUSH.
```

**The "exit code 0 but errors in stderr" trap:** `tsc --noEmit` can occasionally return exit code 0 even with errors in some configurations (incremental caches, prior failed builds). The grep step is the safety net. Always grep the output, never just read the tail.

**The "truncated output" trap:** when tsc reports errors, the output can be hundreds of lines. Reading only the tail misses errors in the middle. Use `tee` to a file and `grep` across the whole file.

**The "LSP diagnostics in patch output" trap:** LSP diagnostics embedded in `patch` tool results are from the pre-patch state (line numbers and conditions no longer match the file). They are stale. Do not trust them. Re-run tsc after every patch that might have affected types.

**Real diagnostic recipe — when you suspect you may have shipped broken code:**

```bash
# 1. Check the latest commit's diff for the suspicious file
git show HEAD --stat
git show HEAD -- <suspected-file.tsx> | head -40

# 2. Run tsc fresh, save output
cd ~ maintainer/Projects/<project>
node_modules/.bin/tsc --noEmit 2>&1 | tee /tmp/tsc-check.log
echo "exit=$?"

# 3. Grep for any error mentions
grep -E "error|Error" /tmp/tsc-check.log | head -20

# 4. If errors exist, fix them as a separate commit
# 5. Re-run tsc, confirm clean BEFORE the next push
```

**Why this isn't redundant with the existing §15 (LSP diagnostics are stale):** §15 says LSP is stale; this section says the *fresh tsc* itself is what you must inspect, not just trust. The two together form a complete verification behavior: (1) don't trust the stale LSP from patch results, (2) actually read the fresh tsc output you do run.

**Companion to "don't over-claim completion" in `user-interaction-standards`:** the over-claim pattern (claiming something works without verifying) is the umbrella. This section is the specific verification behavior that prevents it — what to actually look at before saying "verified clean."

## 20. The `patch` parameter shape — when "path required" loops persist despite knowing §9c (added Aug 11 2026)

**Trigger:** You know §9c. You know `mode='replace'` needs `path` and `mode='patch'` needs the V4A body in the `patch` field. You get the "path required" error in a session anyway and you re-send the same broken call 2-3 more times before switching.

**Real failure (Aug 11 2026):** During the C/E/F batch session, I tried to `patch` `src/app/api/realtime/poll/route.ts` with `mode='replace'` but the body was a multi-line V4A block inside the `old_string` parameter. The tool returned `path required`. I retried the same V4A-in-old_string shape 5 more times. The tool kept returning the same error.

**The diagnostic that finally unblocked me:** I checked the JSON of my own calls and realized I was passing the V4A content in `old_string` but never supplying `path` as a top-level parameter. The tool was correctly complaining that `mode='replace'` requires `path`, and my payload didn't have it.

**The deeper lesson:** §9c says "build the next call from three answers." When you are mid-edit and tired, the easier path is to retry the same call with one trivial change (add a stray character, fix a typo). That rarely fixes shape errors. The fix is to **re-read the actual tool call as a JSON object** and confirm the structure matches the contract.

**The recovery recipe — when "path required" persists after 2 retries:**

```bash
# 1. Stop sending the failing call.
# 2. cat the JSON you sent (read your own conversation history).
#    Look for the shape of the parameter, not just the values.
# 3. Confirm: does my call have `path` as a top-level key?
#    - YES: you are in mode='replace'. The error must mean
#           something else — read the body. Maybe a malformed
#           string is making the parser fail before reading path.
#    - NO: you forgot path. Add it as a top-level parameter.
# 4. Confirm: does my call's body have `*** Begin Patch` markers?
#    - YES: you are writing V4A in mode='replace'. Switch to
#           mode='patch' and put the body in `patch` field.
#    - NO: find-and-replace is correct. Verify path is set.
```

**Why this matters more than §9c:** §9c is about recognizing the loop. This section is about **breaking the loop** when you are in the middle of it. The difference is operational: §9c tells you what to check BEFORE you send; this tells you what to do WHEN the loop has already started.

**Pre-flight self-check (every patch call):**

```
1. Does my call have `path` as a top-level parameter?
2. Does my call's `old_string` or `new_string` contain `*** Begin Patch`?
3. Am I in mode='replace' or mode='patch'?
   - replace: requires path + old_string + new_string
   - patch: requires ONLY the `patch` field (V4A body)
```

If any answer is unclear, stop and re-derive from the contract.

## 23. The "old_string and new_string required" trap when fields ARE present (added Sep 17 2026)

**Trigger:** You call `patch` in `mode='replace'` with BOTH `old_string` and `new_string` clearly supplied as top-level fields in the JSON. Both are non-empty, multi-line strings. The tool returns:

```json
{"error": "old_string and new_string required"}
```

You check the JSON of your own call. The fields ARE there. They have content. The harness fires `repeated_exact_failure_warning` after 2 identical failures. You retry the same shape 5+ times. Each retry returns the same opaque error.

**Real failure (Sep 17 2026, the target platform history-route auth):** While patching `/api/signals/history/route.ts` to add dual-auth (`Authorization: Bearer *** + cookie fallback), the agent supplied a `patch` call with mode='replace', explicit `path`, and an `old_string` + `new_string` both with real content (one was 8 lines, the other was 22 lines of dual-auth logic). Every call returned `{"error": "old_string and new_string required"}` even though both fields were clearly present. The agent retried the exact shape **5 times** with no behavior change. The harness `repeated_exact_failure_warning` fired after the 2nd identical failure but the agent kept retrying the same payload because the JSON it sent had the fields it claimed to be sending.

**Root cause analysis — the tool's error message is misleading:**

The "old_string and new_string required" error is the patch tool's generic failure when its validator rejects a payload. Most often the actual cause is NOT that the fields are missing — it's that the validator's other checks failed first. Likely causes (none of which the error string reveals):

1. **Strings too long for the validator's soft cap.** Multi-KB `old_string` or `new_string` can hit a soft limit; the validator returns "required" instead of "too long."
2. **JSON encoding quirk.** A specific combination of escapes, smart quotes, em-dashes, or non-ASCII characters in either field can break the parser before schema validation. The patch tool can't tell "missing" from "unparseable" and returns the same generic error.
3. **Whitespace alignment at boundary.** When `old_string` and `new_string` end with inconsistent trailing newlines and the strings differ in length significantly, the validator's pre-check can reject before fuzzy-match logic runs.
4. **Template-literal-shaped content.** When the content has `` `...${var}...` `` patterns (template literal interpolation), the validator sometimes flags the dollar-sign-brace as an unresolvable placeholder and returns the generic "required" error rather than a more specific one.

**The diagnostic that actually unblocks (the Sep 17 fix):** the agent switched to `execute_code` with a Python script that does `file_contents.replace(old, new)` on raw bytes. That worked first call. **If `patch` returns "required" but you're CERTAIN the fields are present**, the issue is one of the four root causes above — none of them can be fixed by re-typing the same JSON.

**The recovery recipe — when `old_string and new_string required` persists past 2 retries:**

1. **Stop calling `patch`.** Each retry is wasted context. The tool isn't telling you the real problem.
2. **Switch to `execute_code` with Python byte-level replace:**

```python
import sys
path = '~ maintainer/Projects/<project>/<file>.ts'
content = open(path).read()

old = """<exact bytes from file>"""
new = """<new bytes>"""

if old not in content:
    print("OLD NOT FOUND — diff the actual file content")
    sys.exit(1)

open(path, 'w').write(content.replace(old, new))
print("replaced")
```

3. **Verify immediately:**

```bash
cd ~ maintainer/Projects/<project>
npx tsc --noEmit 2>&1 | head -5
```

4. **If the file needs MANY changes** (≥3 distinct sections), use `write_file` with the entire corrected file content instead — bypasses the patch tool entirely.

**The companion pattern — same trap on `write_file`:** When you use `write_file` to ship a large section, the tool can ALSO mis-parse content that contains template-literal-shaped code or unusual Unicode — but the error message is different and clearer. If `write_file` ALSO fails on the same content, your content has a real escaping problem; fix the content, don't keep retrying either tool.

**The pre-flight check before calling `patch` on a TS file with multi-section changes:**

```
Does my new_string end with a closing brace `}` followed by a newline `\n`?
  → YES: that's typically the trigger. Add a sentinel comment after the `}`
    in BOTH old_string and new_string to give the parser a stable anchor:
    /* __PATCH_ANCHOR__ */\n\n
  → NO: proceed, but verify the leading whitespace of new_string is
         byte-for-byte identical to old_string.

Am I passing multi-KB strings?
  → YES: cap new_string at ~2KB. Break the change into 2-3 smaller patches.
  → NO: proceed.

Does new_string contain `$\{` (template literal interpolation)?
  → YES: keep the backtick + $ literal in a single unbroken block.
    Don't wrap it across lines mid-interpolation.
  → NO: proceed.
```

**The hard rule — bug-triage the FAILURE, not the error message:**

This section was added after losing 5+ tool calls and 4 round-trips to a misleading error message. The pattern is the same as §19b (don't trust tool success without reading content): **the error message is the tool telling you SOMETHING went wrong, not a literal description of what's wrong.** When the same error message recurs from the same tool call shape 2+ times in a row, switch tools rather than retry.

**Cross-reference:** companion to `hermes-agent-skill-authoring` which describes the read-before-write guard pattern. This section is the runtime version of that pattern — when the patch tool rejects even valid-looking payloads, switch to a tool that doesn't run validators at all (Python `str.replace` on raw bytes via `execute_code`).

**For the full Python `str.replace()` recipe** (including the diagnostic for `OLD NOT FOUND`, the two-step replace pattern for multi-KB edits, shell-quoting gotchas with `${...}` template-literal-shaped content, the immediate `tsc --noEmit` verification, and a real Sep 17 worked-example), see `references/python-bytes-replace-recipe.md`.

## 23a. The `<TOKEN>` literal gets partially consumed by patch in TS/TSX files — the JSX-tag-start detection eats `<IDENTIFIER` substrings (added Sep 24 2026)

**Trigger:** You're patching a TypeScript route file or component that contains a literal `<TOKEN_NAME>` substring as a comment, JSDoc, or inside a string. Example: a JSDoc comment that includes `PUSH_API_TOKEN` wrapped in angle brackets as a placeholder for a credential name.

**Symptom:** The patch tool strips `<PUSH_API_TOKEN` (or whatever identifier) from the written content. The diff shows the text going in correctly. The file on disk is missing the leading angle-bracket + identifier — only the rest of the comment remains. The next `tsc --noEmit` fails with cryptic JSX or template-literal syntax errors far from the actual corruption point.

**Real failure (Sep 24 2026, the target platform chat route + webhook notify):** Two separate `patch` calls in one session both had the same bug. The agent was adding JSDoc to `/api/chat/route.ts` and `/api/signals/notify/route.ts` and the `new_string` contained the substring `<PUSH_API_TOKEN` as part of a doc comment. The patch tool's writer interpreted `<PUSH_API_TOKEN` as the opening of a JSX-style tag and silently consumed the opening angle bracket + identifier, leaving only the rest of the comment in the file. The downstream `tsc` then failed because the surviving text had a dangling closing `>` with no matching opener.

This is **distinct from §17 (template-literal corruption)**: §17 is the patch tool dropping backticks around `${...}`. This new failure mode is the patch tool consuming an **angle-bracket opener** in JSDoc/string content. Same root cause (heuristic content normalization), different failure surface.

**The diagnostic — when tsc reports a parse error in a file you just patched with a JSDoc:**

```bash
# Look for any < followed by an uppercase identifier in the corrupted region:
grep -nE '<[A-Z_]{3,}' <file>
# A JSDoc placeholder like <PUSH_API_TOKEN> shows up as "<PUSH_API_TOKEN"
# in the source, but if the patch tool ate the opener, the file now has
# only "PUSH_API_TOKEN>" — which doesn't match this regex.
# If you see <IDENTIFIER in the diff you sent but it's NOT in the file,
# the patch tool consumed the opener.
```

**The pre-flight check before patching TS/TSX files with comment text that contains angle brackets:**

```
Does my new_string contain `<` followed by an uppercase identifier?
  → YES: the patch tool WILL likely consume `<IDENTIFIER` as a JSX opener.
         Use `***` placeholders instead. For example, write
         "the INTERNAL_API_TOKEN env var" instead of "<INTERNAL_API_TOKEN>".
         Or wrap the literal in backticks/code so it's not interpreted:
         `` `PUSH_API_TOKEN` `` (no angle brackets).
  → NO: proceed.

Does my new_string contain a JSDoc comment with placeholders?
  → YES: prefer `***` or `_NAME_` style placeholders in JSDoc:
         /** Auth via *** (the shared webhook secret). */
         rather than /** Auth via <PUSH_API_TOKEN>. */
  → NO: proceed.
```

**Why the patch tool does this:** the writer's content normalization step treats `<UPPERCASE_IDENTIFIER` as a JSX tag (because that pattern IS a JSX tag in TSX files), and the strip-and-rewrite heuristic removes what looks like an unclosed tag opener. This is correct behavior for `.tsx` files with real JSX, but wrong for `.ts` files with comment/string content.

**The recovery recipe when you've already shipped the bug:**

1. Read the corrupted region of the file: `sed -n 'A,Bp' <file>`.
2. Replace the corrupted comment with a placeholder version using `***`: `sed -i '' 's/the surviving text/***/g' <file>` or use `patch`.
3. Verify with `tsc --noEmit` that the file parses.
4. Update your mental model: ANY angle-bracketed identifier in a JSDoc, comment, or string literal inside `new_string` will be partially consumed by the patch tool. Use `***` placeholders or wrap in backticks.

**Companion to §17 (template-literal corruption) and §23 (multi-KB `old_string` rejection):** all three are the same underlying class — the patch tool's content normalization step makes assumptions about the file type that don't hold for comment/string content. The unifying rule: **never put JSX-shaped content (`<Identifier`) inside `new_string` unless the file actually contains JSX and you intend the literal to be a JSX tag.**

## 24a. Single-line `//` comments inside `.tsx` JSX — the unrecognized-syntax trap (added Sep 27 2026)

**Trigger:** You're adding a comment above or between JSX elements in a `.tsx` file. You instinctively reach for `//` because you're commenting out a block of code or annotating a section. The patch tool accepts the change (no error message), the diff looks clean, but **`tsc --noEmit` fails with `TS1381: Unexpected token. Did you mean {'}'} or &rbrace;?` or `TS17008: JSX expression must have one parent element`** because in JSX, comments MUST use the `{/* ... */}` syntax.

**Real failure (Sep 27 2026, multiple commits in one session):** The agent added `// Comment` to a `new_string` inside `TradingDashboardView.tsx`, `AccountConnectionModal.tsx`, and `src/app/dashboard/settings/page.tsx` three separate times in one session. Each time the patch tool accepted the edit without complaint. Each time `tsc --noEmit` failed with `Unexpected token. Did you mean {'}'} or &rbrace;?` near the comment line. Each fix required re-reading the file, finding the comment, replacing `//` with `{/* */}`, and re-running tsc. ~3 round-trips of churn per instance for ZERO productive progress.

**The specific failure shape:**

```tsx
// GOOD (TypeScript or top-level .tsx)
const x = 1; // works fine

// BAD — single-line // comment inside JSX (the agent's instinct)
<div>
    // ← this is the bug
    <button>...</button>
</div>
```

The patch tool writes the bytes verbatim. The TypeScript parser hits `//` inside JSX and reads it as a malformed token — the JSX grammar only allows `{ ... }` for inline expressions, and `//` has no meaning there.

**The correct shape:**

```tsx
// GOOD — JSX comment syntax (the agent's "second guess" after seeing the error)
{/* This is a JSX comment. */}
<div>
    {/* Multi-line JSX comments also work across lines */}
    <button>...</button>
</div>
```

**Why the patch tool doesn't catch this:** the patch tool is a textual find-and-replace. It doesn't run the TypeScript parser on your output. So an `old_string` containing JSX content + `//` + new JSX content will be written verbatim with no validation. The TS error surfaces only when you (or CI) run `tsc --noEmit` after the patch.

**The fix — the rule that prevents this:**

> **Before any `patch` call that targets a `.tsx` file, check the surrounding context. If the surrounding context contains JSX (i.e., you're inside a `return ( ... )` statement, between `<div>...</div>` tags, or adjacent to JSX elements), every comment in your `new_string` MUST use `{/* ... */}`. Plain `//` is only valid in plain TypeScript regions (above the return statement, inside `useEffect`, inside helper functions, etc.).**

**The pre-flight self-check before any `.tsx` patch:**

```
Does my new_string contain `//`?
  → Look at the position of the comment. Is it inside a JSX expression?
    - YES: replace with `{/* comment */}`. The patch tool won't catch this.
    - NO:  plain `//` is fine.

Quick visual heuristic: am I patching a line inside `<div>...</div>`,
between `<button>...</button>` tags, or anywhere within a JSX tree?
  → YES: comments must be `{/* */}`.
  → NO:  plain `//` is fine.
```

**Why the agent gets this wrong repeatedly:** in pure `.ts` files, `//` is the universal comment syntax. The agent's muscle memory says "comment → `//`". When transitioning to `.tsx`, the rule changes INSIDE JSX expressions but stays the same outside them. The agent writes `//` because it doesn't mentally distinguish "inside JSX" vs "outside JSX" contexts.

**The mental model to break this loop:**

- In `.tsx`, **JSX is its own grammar**. The JSX grammar has no concept of `//` comments.
- The fix is to mentally tag every comment you're about to write with "am I inside JSX?"
- If yes → `{/* comment */}`.
- If no → `//` is fine.

**The recovery recipe when you've already shipped a `//` comment into JSX:**

```bash
# 1. tsc reports TS1381 / TS17008 near your patch
npx tsc --noEmit 2>&1 | head -10

# 2. Open the file at the reported line
sed -n 'A,Bp' <file.tsx>

# 3. Find the JSX comment — it looks like:
#      // <comment text>
#    inside JSX context (between JSX tags or inside return (...))

# 4. Replace with:
#      {/* <comment text> */}

# 5. Re-run tsc
npx tsc --noEmit 2>&1 | head -5
# → should now be clean
```

**The companion pitfalls (all in the same family — "agent writes something the patch tool accepts but the parser doesn't"):**

- **§17 (template-literal corruption)** — backticks eaten from `${...}`
- **§19 (3+ template-literal corruptions)** — escalation: switch to `write_file`
- **§23a (`<TOKEN>` collapse)** — patch tool eats `<IDENTIFIER` in JSDoc
- **§24 (JWT-specific `<TOKEN>` collapse)** — narrower variant of §23a
- **§24a (this section — JSX `//` comments)** — patch tool writes `//` into JSX, parser chokes

All five share the same root cause: **the patch tool is a textual find-and-replace, not a TypeScript-aware editor.** Anything that requires semantic understanding of the file (JSX grammar, template literal interpolation, JSDoc placeholders, comment syntax) is the agent's responsibility to get right BEFORE the patch call.

**The hard rule that prevents all five:**

> **Before any `patch` call on `.tsx` or `.ts`, scan your `new_string` for these five classes of content and verify each is correct for the file type:**
>
> 1. `// foo` inside JSX → must be `{/* foo */}`
> 2. `` `Bearer ${token}` `` template literal → backticks intact, no `*** ${...}` eating
> 3. `<IDENTIFIER` in JSDoc → use `***` or backticked names instead
> 4. `{/* comment */}` outside JSX (in `.ts` file or in pure TS regions) → just use `// comment`
> 5. Multi-KB `new_string` → cap at ~2KB, break into multiple smaller patches

The patch tool will accept any of these errors silently. tsc, Turbopack, or production runtime will catch them — at high cost.

**Real cost (this session alone):** the agent shipped `// foo` comments inside JSX at least 3 times in one session, requiring ~3 round-trips per instance to diagnose, read, fix, re-verify. Each fix was the same pattern — replace `//` with `{/* */}`. The recurring failure pattern is the strongest signal this pitfall deserves a permanent place in the skill.

**Companion to §21a (CSS comment with leading `{`):** §21a is the CSS-specific version of this same family — "comments inside a different grammar than the file type expects." §24a is the JSX-specific version. Both share the underlying agent failure pattern: writing comments in a familiar syntax without checking whether the surrounding context supports that syntax.

**Companion to §11 (patches inside patches — indentation drift):** both fail because the agent's mental model of the file's grammar drifts after the patch is written. §11 is about indentation drift; §24a is about grammar drift. The defense in both cases is the same: read the actual file content (or run `tsc`) between sequential edits.



A specific, narrow variant of §23a that bit twice in one session.

**Trigger:** You're wiring JWT/bearer-token auth in an API route. The actual code uses a template literal like `` `Bearer ${conn.access_token}` ``. The `new_string` you want to ship also includes a **JSDoc comment block above the function** that mentions the auth header literally, like:

```ts
// Auth header fallback pattern: Authorization: Bearer *** is shared-secret for engine webhook auth.
```

You write that as part of the `new_string` body. The patch tool consumes `<PUSH_API_TOKEN` (or whichever literal) because it sees `<UPPERCASE` and treats it as a JSX opener.

**The narrower problem:** the JSDoc comment gets shipped in the file but the corrupted version no longer parses (because the opener got eaten but the closer `>` survived). The actual code (with backticks and `${...}`) is fine — only the comment is broken.

**The narrow fix:** never write `<UPPERCASE_IDENTIFIER>` in JSDoc inside a TS file you're patching. Use one of:

- `***` placeholder: `*** is the shared webhook secret`
- Backticked name: `` `PUSH_API_TOKEN` is the shared webhook secret ``
- Plain prose: `the push API token is the shared webhook secret`

The first form (`***`) is the convention in this codebase — it's what the existing comments already use. Stick with it.

**Why this is §24 instead of a §23a example:** §23a is the general patch-tool rule about angle brackets in JSDoc. §24 is the specific JWT/bearer pattern that bit twice in one session — documenting it separately makes it greppable for future sessions that are wiring auth.

**Real failure (Sep 24 2026, two separate patches in the same session):**
- `/api/signals/[id]/event/route.ts` — added a JSDoc comment with `<PUSH_API_TOKEN` literal.
- `/api/admin/signals/notify-direct/route.ts` — added a doc comment with `<INTERNAL_API_TOKEN` literal.

Both files shipped with the opener consumed. Both were caught by `tsc --noEmit` downstream. Both cost ~1 round-trip to repair.

**The MEMORY.md pattern that documents this for the next session:** the system prompt's MEMORY already carries a line about this exact trap:

> § **Supabase SSR auth format: use @supabase/ssr on BOTH sides.** Sep 14 2026 — Custom storage shim ... (etc) ... (user session memory)

This same trap class — patch tool eating literal text that LOOKS like JSX — should also be documented in MEMORY.md as a reminder for future sessions, since the patch tool's normalization heuristic is non-configurable from the agent side. Use `***` or backticked names in JSDoc to be safe.

