content># Python `str.replace()` via `execute_code` — the recovery recipe

## When this recipe applies

You called `patch` in `mode='replace'` with explicit `path` + `old_string` + `new_string`. The tool returned `{"error": "old_string and new_string required"}` repeatedly (2+ retries). The harness's `repeated_exact_failure_warning` has fired.

The patch tool's validator has rejected your payload, but its error message is misleading — the fields ARE present. The validator hit one of these soft fails before reaching field validation:

- Multi-KB strings (combined `old_string` + `new_string` > 4KB)
- Template-literal-shaped content with `${...}` (validator's interpolation parser flags the `${`)
- Non-ASCII characters (em-dash, smart quotes, ellipsis) at specific offsets
- Trailing-newline mismatches between old_string and new_string
- JSON-escape edge cases (escaped backslashes, multi-line raw strings)

When any of these fire, the patch tool returns the same generic error regardless of which root cause actually triggered. Re-typing the same JSON won't help.

## The recipe

Use `execute_code` with a Python script that reads the file, does `str.replace()` on raw bytes, and writes it back. Python's `str.replace()` is byte-for-byte exact, no validator, no fuzzy matching, no template-literal parser.

```python
path = '/absolute/path/to/file.tsx'
content = open(path).read()

old = """<exact old content as a triple-quoted Python string>"""
new = """<exact new content as a triple-quoted Python string>"""

if old not in content:
    print('OLD NOT FOUND')
    # Diagnostic: search for near-matches
    import re
    # Show lines that look similar to help debug
    first_few = old[:80]
    for line in content.split('\n')[:50]:
        if first_few[:20] in line:
            print('NEAR:', repr(line))
else:
    open(path, 'w').write(content.replace(old, new))
    print('replaced')
```

## Critical patterns

### Triple-quoted strings handle multi-line cleanly

`"""..."""` (triple double-quote) lets you paste multi-line content directly without escaping `\n` or worrying about shell quoting. The Python parser handles the multi-line string as one literal.

```python
old = """            const supabase = await createClient();
            const { data: { session } } = await supabase.auth.getSession();
            if (!session?.user?.id) {
                return NextResponse.json({ error: 'Not authenticated' }, { status: 401 });
            }
            const userId = session.user.id;"""
```

The leading whitespace MUST match the file's actual indentation. Python `str.replace()` does byte-exact matching — leading 12 spaces will not match 11 spaces in the file.

### Diagnostic when "OLD NOT FOUND"

When `old not in content` fires, the script exits early. The diagnostic output prints lines that LOOK like the start of your `old` string. Compare those to what you expected.

Common causes:
- Different indentation (4 spaces vs 8 spaces, tabs vs spaces)
- Different quote characters (straight vs smart quotes)
- A blank line that you forgot
- Trailing newline mismatch (`\n` vs no `\n` at the end of `old`)
- Multi-byte UTF-8 characters that look the same but aren't (em-dash vs hyphen)

The fix: read the file with `head -n <line>` or `read_file` and copy-paste the EXACT bytes into your Python string. Don't retype from memory.

### When `old` is too long for a Python literal

If `old` itself is multi-KB, break it into chunks using `replace()` in two passes:

```python
import sys
path = '/path/to/file.tsx'
content = open(path).read()

# Step 1: Replace the smaller anchor first
old_anchor = """            // Sep 14, 2026 — comment to anchor the change"""
new_anchor = """            // Sep 15, 2026 — UPDATED comment for the new behavior"""

if old_anchor not in content:
    print('ANCHOR NOT FOUND')
    sys.exit(1)

content = content.replace(old_anchor, new_anchor)

# Step 2: Now replace the bigger block using the new anchor as the starting context
old_big = """            // Sep 15, 2026 — UPDATED comment for the new behavior
            const supabase = await createClient();"""
new_big = """            // Sep 15, 2026 — UPDATED comment for the new behavior
            let userId: string | null = null;"""

if old_big not in content:
    print('BIG BLOCK NOT FOUND after step 1')
    sys.exit(1)

content = content.replace(old_big, new_big)

open(path, 'w').write(content)
print('replaced in 2 steps')
```

Two-step replace avoids the "huge string literal" soft-cap that bites single-step replacement.

### When Python's `execute_code` shell-escapes your strings

If the `old` string contains literal `\n` characters (Python source has them as escape sequences), they're preserved through the shell. If it contains real newlines (multi-line strings in the Python source), they're also preserved. The shell doesn't interpret Python string contents — it passes them as-is.

**The gotcha:** shell `$variable` interpolation. If your `old` string contains `$variable` literals (e.g., a JavaScript template literal `${userId}` inside the string), the shell may try to expand them. **Wrap in single quotes** for the entire Python script body to prevent this:

```python
# In execute_code: pass the entire script as a single Python string.
# The shell doesn't expand $-variables inside single quotes.
script = '''
import sys
path = '/path/to/file.tsx'
content = open(path).read()
old = """        Authorization: `Bearer ${tl_access_token}`,"""
new = """        Authorization: `Bearer ${token}`,"""
if old not in content:
    print('OLD NOT FOUND')
    sys.exit(1)
open(path, 'w').write(content.replace(old, new))
print('replaced')
'''
# Execute the script (depends on execute_code harness contract)
```

## When this recipe does NOT work

- **The file is checked into git and you want to preserve the change history.** Python `str.replace()` doesn't preserve git history for the modified region (it's still one commit, but no `git add -p` granularity). If you need line-level history, use `write_file` of a small file, or split into multiple `write_file` calls.

- **The change spans 5+ sections.** Python `str.replace()` works on one match at a time. If you need to change 5 sections, write 5 separate scripts or use `write_file` with the full corrected file.

- **The change is mostly additive.** `write_file` is faster for additive changes. Python `str.replace()` shines for replacing existing content (not adding new).

## The 3-level escalation when `patch` repeatedly fails

| Level | Tool | When to use |
|-------|------|-------------|
| 1 | `patch` mode='replace' | Normal flow, edits < 2KB, single-section |
| 2 | `execute_code` with Python `str.replace()` | `patch` returns "required" 2+ times, multi-KB edits, template-literal content |
| 3 | `write_file` with the full file content | File needs 3+ sections edited, repeated Python `replace()` failures, file is small enough to write in one call |

Each level bypasses more of the patch tool's validators. Use level 1 by default. Move to level 2 when level 1 fails twice in a row. Move to level 3 when level 2 has multiple "OLD NOT FOUND" errors that suggest the file is in an unexpected state.

## Verification recipe after `str.replace()`

```python
# Immediately after the script writes the file, run in the SAME script:
import subprocess
result = subprocess.run(['npx', 'tsc', '--noEmit'], capture_output=True, text=True)
print('TYPECHECK:', 'PASS' if result.returncode == 0 else 'FAIL')
if result.returncode != 0:
    print(result.stderr[:500])
```

This catches:
- TS1109 (Expression expected) — your `new` string had a missing brace
- TS2304 (Cannot find name X) — you forgot a closing tag
- TS1005 (Semicolon expected) — template literal corruption slipped through
- Type narrowing failures — your new code path needs different types

If the script catches a typecheck failure, immediately restore from git (`git checkout HEAD -- <file>`) and try a smaller patch.

## Real example from a Sep 17 session

I needed to add dual-auth to `/api/signals/history/route.ts`. The `patch` tool returned `{"error": "old_string and new_string required"}` 5+ times even though both fields were present in the JSON. The content had:

- Multi-line TypeScript with deep indentation (12 spaces)
- An `await` call inside a `try` block
- Template-literal-style logging (`console.warn('[signals/event] ...')`)
- Multi-KB combined string length

The recovery that worked first call:

```python
import sys
path = '~ maintainer/Projects/the target platform/src/app/api/signals/history/route.ts'
content = open(path).read()

old = """    const supabase = await createClient();
    const { data: { session } } = await supabase.auth.getSession();
    if (!session?.user?.id) {
        return NextResponse.json({ error: 'Not authenticated' }, { status: 401 });
    }
    const userId = session.user.id;"""

new = """    // Sep 17, 2026 — Dual-auth: prefer `Authorization: Bearer *** from the
    // adminFetch helper, fall back to cookie-based session via the server
    // client. Same pattern as /api/partners/me/pulse-referral (commit 108cb7fb).
    let userId: string | null = null;
    const authHeader = req.headers.get('authorization') ?? '';
    const bearerToken = authHeader.replace(/^Bearer\\s+/i, '').trim();
    if (bearerToken) {
        const svcBearer = createServiceClient();
        const { data: userData, error: bearerErr } = await svcBearer.auth.getUser(bearerToken);
        if (bearerErr || !userData?.user) {
            console.warn('[signals/history] bearer token invalid', {
                error: bearerErr?.message ?? '(no user)',
            });
        } else {
            userId = userData.user.id;
        }
    }
    if (!userId) {
        const supabase = await createClient();
        const { data: { session } } = await supabase.auth.getSession();
        if (session?.user?.id) userId = session.user.id;
    }
    if (!userId) {
        return NextResponse.json({ error: 'Not authenticated' }, { status: 401 });
    }"""

if old not in content:
    print('OLD NOT FOUND — exiting')
    sys.exit(1)

open(path, 'w').write(content.replace(old, new))
print('replaced')
```

The script ran in 0.4s and produced the correct output. The `patch` tool had burned 5+ calls and 4 round-trips before I switched.

## Companion sections

- §23 (the parent skill) covers the patch tool's misleading "required" error message
- §19 (the parent skill) covers when to give up on patch entirely (after 3 same-pattern corruptions)
- §14 (the parent skill) covers sequential-patch cascading errors — the Python script is the recovery
</content>