# Patch Trap Inspector: Edits That Silently Corrupt Code

> The find-and-replace edits that silently corrupt a codebase, and how to make them safely. Covers generic-pattern replace-all, mixed-indentation traps, template-literal damage and the byte-level recovery recipe.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg) [![Available on Agensi](https://img.shields.io/badge/Agensi-%2414.99-ff5a5f.svg)](https://agensi.io/skills/patch-trap-inspector-edits-that-silently-corrupt-code)

## What it does

Inspect find-and-replace edits before they corrupt code. The workflow pitfalls of the patch tool -- fuzzy matches, replace_all, template literals -- and byte-level recovery steps.

`replace_all=true` replaces EVERY occurrence. If your `old_string` is something like `<p class="lead">\n      Everything below is current state of the platform on <code>main</code>, July 27, 2026.\n    </p>` and that paragraph appears on 8 different slides each with a UNIQUE preceding sentence, you will clobber all 8 unique leads with the same generic text.

## The failure it prevents

Each reference file is one specific failure: the symptom, the actual cause, and the fix. The ones that come up most often:

- Sequential JSX Patches — Compounding Corruption
- Aug 01 settings page corruption
- CSS replace_all corruption pitfall
- Prisma generate silent failure — `[Context: getDmmf]` reference
- Step 1: Replace the smaller anchor first
- Template literal corruption — diagnostic + recovery with bytes-level operations

## When to use it

- avoid a find-and-replace that corrupts a file
- review a patch before applying it
- recover a file corrupted by a bad edit

## What's inside

```
plugins/
└── patch-trap-inspector/
    ├── .claude-plugin/plugin.json
    └── skills/patch-trap-inspector/
        ├── SKILL.md
        ├── references/            (6 files)
        │   └── aug-01-jsx-corruption.md
        │   └── aug-01-settings-page-corruption.md
        │   └── css-replace-all-corruption.md
        │   └── prisma-generate-silent-failure.md
        │   └── python-bytes-replace-recipe.md
        │   └── template-literal-bytes-recovery.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/patch-trap-inspector
claude plugin install patch-trap-inspector@patch-trap-inspector
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/patch-trap-inspector.git
cp -R patch-trap-inspector/plugins/patch-trap-inspector/skills/patch-trap-inspector ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "avoid a find-and-replace that corrupts a file"
```

## Reference material

Loaded on demand — the skill only reads what the task needs.

| File | Covers |
| --- | --- |
| `aug-01-jsx-corruption.md` | Sequential JSX Patches — Compounding Corruption |
| `aug-01-settings-page-corruption.md` | Aug 01 settings page corruption |
| `css-replace-all-corruption.md` | CSS replace_all corruption pitfall |
| `prisma-generate-silent-failure.md` | Prisma generate silent failure — `[Context: getDmmf]` reference |
| `python-bytes-replace-recipe.md` | Step 1: Replace the smaller anchor first |
| `template-literal-bytes-recovery.md` | Template literal corruption — diagnostic + recovery with bytes-level operations |

## License

MIT. See [LICENSE](LICENSE).

---

**Also available as a paid skill on Agensi** — $14.99. Install it with one command, no clone required:

[agensi.io/skills/patch-trap-inspector-edits-that-silently-corrupt-code](https://agensi.io/skills/patch-trap-inspector-edits-that-silently-corrupt-code)

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
