# Template literal corruption — diagnostic + recovery with bytes-level operations

**Companion to §17 + §19 of the parent SKILL.** When the patch tool
corrupts a template literal (drops the leading backtick from
`` `Bearer ${token}` `` so you end up with `*** ${token}` or similar),
the standard `patch` re-fix can hit a **reproducible failure mode** where
the same corruption recurs. This addendum covers the diagnostic and
recovery when patch can't fix it.

## When `patch` keeps corrupting the same template literal

The parent SKILL's §19 covers the 1-2-3 escalation. When you hit the
**second corruption** on the same file:

1. Stop patching immediately.
2. Read the file as **bytes** (not text), inspect the corrupted region,
   and write the file using `bytes.replace` rather than `patch`.

## Diagnostic: shell `sed` output can mislead you

`grep` and `sed` show text. When a template literal's leading backtick
is missing, sed shows the line with `*** ${...}` — which **looks like
literal asterisks** unless you know the original had a backtick there.

```bash
# This may show you the line with literal asterisks
sed -n '837p' src/context/TradingContext.tsx
# "                headers: { Authorization: *** ${sbToken}` },"

# But that's the patched output. The actual on-disk bytes might be different
# (the missing character might be a backtick, not a literal asterisk).
```

## The byte-level diagnostic that actually works

```bash
# See the raw bytes around the suspicious line
sed -n '837p' src/context/TradingContext.tsx | od -c | head -5

# Output:
# 0000000                                                   h   e   a   d
# 0000020   e   r   s   :       {       A   u   t   h   o   r   i   z   a
# 0000040   t   i   o   n   :       `   B   e   a   r   e   r       $   {
# 0000060   s   b   T   o   k   e   n   }   `       }   ,\n
```

The `od -c` hex dump shows the actual bytes. If you see `` ` Bearer ``
in the dump, the file is fine — sed was just rendering the backtick
invisibly. If you see `***` literally, that's a different corruption.

**When sed shows `***` and `od -c` shows `` ` ``** (as in my Sep 9
session): the file is correct, sed was misleading you. **Run `tsc
--noEmit` to confirm** — if it's clean, your file is fine. Don't
patch a non-corruption.

**When sed shows `***` and `od -c` shows `* * *`** (literal asterisks):
the patch tool actually wrote literal asterisks. File is corrupt.

## Python `bytes.replace` for when patch keeps failing

When the same template literal keeps getting corrupted by `patch` (the
parent SKILL's §19 escalation), use Python's `bytes.replace` with the
**exact** broken pattern and the **exact** fixed pattern. Python does
byte-for-byte matching — no fuzzy logic, no template-literal eating:

```python
import os
os.chdir('~ maintainer/Projects/<project>')

path = 'src/<file>.ts'
with open(path, 'rb') as f:
    raw = f.read()

# The broken pattern (literal asterisks if file actually has them,
# or just a misreading from sed — confirm with od -c first)
broken = b'Authorization: *** ${sbToken}` }'  # literal asterisks
fixed = b'Authorization: `Bearer ${sbToken}`,'

n = raw.count(broken)
print(f'Found {n} broken occurrences')

if n > 0:
    raw = raw.replace(broken, fixed)
    with open(path, 'wb') as f:
        f.write(raw)
    print('FIXED')
```

**Why this works when `patch` fails:**

1. **No fuzzy matching** — `bytes.replace` does exact byte-by-byte
   matching. No 9 strategies, no alignment heuristics.
2. **No template literal interpretation** — `b'Authorization: \`Bearer ${sbToken}\`'` is
   just bytes. The `\` and `$` aren't shell variables; they're literal
   characters in the byte string.
3. **No shell expansion** — Python bytes literals don't get processed
   by bash, so `***` stays `***`, backticks stay backticks.

## The "shell ate my backticks" debugging pattern

When your Python script does `raw.replace('Authorization: *** ${sbToken}` ` `,` '...')
and Python reports "0 occurrences found," the issue is usually that
your **shell** already expanded or stripped the asterisks/backticks
before Python saw them. Three fixes:

```python
# Fix 1: use raw strings (no backslash escaping needed)
broken = r'Authorization: *** ${sbToken}` }'
fixed = r'Authorization: `Bearer ${sbToken}`,'

# Fix 2: use bytes literals (immutable, no shell interpretation)
broken = b'Authorization: *** ${sbToken}` }'
fixed = b'Authorization: `Bearer ${sbToken}`,'

# Fix 3: write to a script file via write_file() — never let the shell
# process the patch parameter
# Write your python to /tmp/fix.py, then bash -c "python3 /tmp/fix.py"
```

**The diagnostic for "shell ate my characters":**

```bash
# Write the bytes to a file, then read them back
echo -n 'Authorization: *** ${sbToken}` }' > /tmp/test.txt
od -c /tmp/test.txt
# Compare with the actual file:
sed -n '837p' src/<file>.ts > /tmp/test2.txt
od -c /tmp/test2.txt
# If the byte sequences differ, the shell is eating characters.
```

## When the file is correct but sed makes it look broken

The diagnostic that saves the most time:

```bash
# 'od -c' on the line sed flagged as broken
sed -n '<broken_line>' <file> | od -c
```

If `od -c` shows the correct backtick (`` ` ``) followed by `$`:
**the file is fine**. The `patch` tool did NOT corrupt anything.
`sed` is just rendering backticks invisibly (they're not displayed by
default in some terminals). Run `tsc --noEmit` and `npx next build`
to confirm.

If `od -c` shows the bytes are actually different from what you
expected — yes, there's corruption. Use Python `bytes.replace` to fix.

## The full recovery recipe

```bash
# 1. Confirm corruption with od -c
sed -n '<broken_line>' <file>.ts | od -c | head -3
# Look for missing opening backtick before $ or ${
# (it should be ` $ — single backtick, then space, then $)
# If you see "*** $" or similar, that's the corruption.

# 2. Extract the exact broken bytes
# Use a python script to print the byte sequence
python3 -c "
with open('<file>.ts', 'rb') as f:
    raw = f.read()
lines = raw.split(b'\n')
# Print line N as bytes
print(repr(lines[N-1]))
"

# 3. Fix with bytes.replace
python3 << 'PYEOF'
import os
os.chdir('~ maintainer/Projects/<project>')
path = '<file>.ts'
with open(path, 'rb') as f:
    raw = f.read()
broken = b'<exact broken bytes from step 2>'
fixed = b'<exact fixed bytes>'
if broken not in raw:
    print('Broken bytes not found — check your repr output')
else:
    raw = raw.replace(broken, fixed)
    with open(path, 'wb') as f:
        f.write(raw)
    print('FIXED')
PYEOF

# 4. Verify
npx tsc --noEmit && npx next build

# 5. Commit
git add -A
git commit -F /tmp/commit_msg.txt
```

## What this addendum doesn't cover

- The initial `patch` failure mode that produces the corruption in the
  first place — covered in the parent SKILL's §17 (general pattern)
  and §19 (escalation). When the first corruption happens, the parent
  SKILL's recipe still works. This addendum is for the **second-or-later
  corruption on the same file** where patch keeps producing it.
- The sed-vs-od discrepancy — covered in the parent SKILL's §22 (debug
  with `cat -v`). This addendum extends that with `od -c` and bytes-
  level recovery specifically for template literal corruption.

## Honest caveat

Python `bytes.replace` is the **last** fix in the escalation chain,
not the first. If the file has only one corruption and `patch` is
likely to handle it cleanly next time, retry `patch` first — only
escalate to `bytes.replace` after the parent SKILL's §19 tells you
"this file has a reproducible failure mode." Otherwise you're
debugging with a heavier tool than the problem requires.
