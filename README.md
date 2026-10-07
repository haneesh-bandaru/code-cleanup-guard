# code-cleanup-guard

A Claude skill that catches the leftovers AI-assisted edits leave behind: unused code, orphaned logic, and logic that no longer makes sense. It **asks before removing anything**.

## What it does

After Claude writes or changes code, it audits **only what changed**:

- the diff of your current branch against its base (main, master or develop), or
- the specific flow you name (e.g. "check the checkout flow"), or
- just the code from the current chat, if there is no git repo.

It never sweeps the rest of your codebase.

It reports two kinds of findings:

| Kind | Examples | What happens |
|---|---|---|
| Safe to remove | uncalled functions, unused imports/variables, unreachable code, replaced old versions, stray debug logs | Numbered list; you reply "remove 1 and 2" |
| Suspicious logic | always-true/false conditions, code overwriting each other, duplicated logic that may disagree, names that no longer match behavior | Highlighted with a reason; never auto-fixed |

## Example output

```
## Cleanup check
Scope: branch `feature/discounts` vs `main` (6 files changed)

### Safe to remove (needs your OK)
1. `oldFormatDate()` in utils.js, line 42: replaced by `formatDate()`, no remaining calls found.
2. `import axios` in api.js: no longer used.

### Highlighted: logic that may not make sense
⚠️ `checkout.js` line 88: `if (total > 0 && total < 0)` can never be true.
   Likely cause: the earlier discount change left this condition behind.

Reply "remove 1 and 2" (or "remove all safe ones") and I'll do it.
```

## Install

**Claude Code**

```bash
git clone https://github.com/<your-username>/code-cleanup-guard ~/.claude/skills/code-cleanup-guard
```

Or place the folder in a project's `.claude/skills/` directory and commit it so your team gets it too.

**Claude.ai**

Download `code-cleanup-guard.skill` from the Releases page and upload it under skills settings (requires a plan that allows custom skills).

**Claude API**

Upload the folder through the skills endpoint (`/v1/skills`).

## Safety

This skill is instructions only. It contains no scripts and executes no code. As with any skill, read it before installing.

## Contributing

Issues and pull requests are welcome, especially false positives and missed leftovers. When reporting one, include the code snippet and what you expected.

## License

MIT, see [LICENSE](LICENSE).
