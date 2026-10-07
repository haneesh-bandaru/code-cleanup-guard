---
name: code-cleanup-guard
description: Audits ONLY the code changed in the current branch (or the specific flow the user names) for leftovers after AI-assisted edits - unused code, orphaned logic and illogical logic - and asks before removing anything. Use this skill EVERY time code is created, modified, refactored, or fixed in a conversation, and whenever the user says things like "clean up", "remove unused", "dead code", "leftover", "this logic makes no sense", "review my changes", or "check this flow". Trigger even if the user does not mention cleanup, because the point is catching stale code that earlier edits left behind. Never audits the rest of the codebase.
---

# Code Cleanup Guard

When code is changed repeatedly, especially with AI help, the old version often survives next to the new one: a function nobody calls, a variable that is set but never read, a branch that can never run. The user cannot easily see these, and they quietly cause bugs and confusion later. This skill makes Claude check for them after every code change, and never delete anything without the user's say-so.

## Scope: only what changed

The audit covers **only the changes made on the current branch, or the flow the user asked about**. It never sweeps the whole codebase. Old messes that were there before this work are not this skill's business, and reporting them buries the real findings. Decide the scope in this order:

1. **The user named a flow or feature** ("check the checkout flow", "review the login changes"): trace that flow's code path and audit only the files and functions on it.
2. **A git repo is available** (terminal, Claude Code, or uploaded repo): audit the diff of the current branch against its base.
   ```
   git branch --show-current
   BASE=$(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null || git merge-base HEAD develop)
   git diff $BASE...HEAD --stat      # which files changed
   git diff $BASE...HEAD             # the changes themselves
   git diff                          # plus uncommitted work
   ```
   If the base branch is unclear, ask the user once which branch to compare against.
3. **Chat only, no git**: audit only the code written or modified in this conversation, plus the immediate code those edits touch.

State the scope in one line at the top of the report (e.g. "Scope: branch `feature/discounts` vs `main`, 6 files changed") so the user can correct you if it is wrong.

### Looking outside the diff, without reporting on it

You may need to read other files to judge a change, for example to search whether an old function still has callers elsewhere. That is fine. The rule is about what you **report**, not what you read.

- **Report** a finding only if the changed code caused it or contains it: new code that is unused, an edit that left an old version behind, or a change that removed the last caller of something (making an untouched function newly unused).
- **Do not report** problems that already existed in code the branch never touched. If you spot something serious there, add at most one line under "Outside scope, not checked" so the user knows, and leave it.

## When to run

After you write or edit code in this conversation, and before the final answer, run the audit on the scoped changes. Leftovers often sit slightly away from the edit (the old caller of a renamed function, the old import, the old state variable), so look at the whole changed file's relationship to the change, but report only what the change is responsible for.

## The audit

Look for two categories. Treat them differently.

### A. Unused code (safe to propose removing, once confirmed)

- Functions, methods, classes, components never called or referenced
- Variables, constants, state, props, parameters assigned but never read
- Imports and dependencies no longer used
- Code after `return` / `throw` / `break`, or inside conditions that are always false
- Old versions of something that was replaced (e.g. `fetchDataOld`, commented-out blocks, duplicate handlers doing the same job)
- CSS classes, routes, config keys, or event listeners with no matching user
- Leftover debug code: stray `console.log`, `print`, TODO stubs that were superseded

Before calling something unused, **search for its name across the code you can see** (including exports, string references, dynamic calls, tests). If it is exported or might be used by files you cannot see, say so and mark it "possibly used elsewhere" rather than "unused".

### B. Illogical or suspicious logic (flag only, never auto-fix)

These need the user's judgment, so highlight them rather than rewriting:

- Conditions that are always true/false or contradict each other
- Two pieces of code that fight each other (one sets a value, another immediately overwrites it)
- Duplicate logic implemented in two places that may now disagree
- State updated but the UI or output never reflects it
- Error handling that swallows errors silently, or catches something that cannot be thrown
- Loops, retries, or fallbacks that cannot terminate or can never trigger
- Names that no longer match behavior after an edit (e.g. `getUser` that now deletes)
- A change that fixed one spot while an identical pattern elsewhere still has the old behavior

For each, explain in plain words **why it looks wrong**, not just that it does.

## How to report

Do the task the user asked for first. Then add the audit at the end, in this shape:

```
## Cleanup check
Scope: branch `feature/discounts` vs `main` (6 files changed)

### Safe to remove (needs your OK)
1. `oldFormatDate()` in utils.js, line 42: replaced by `formatDate()`, no remaining calls found.
2. `import axios` in api.js: no longer used.

### Highlighted: logic that may not make sense
⚠️ `checkout.js` line 88: `if (total > 0 && total < 0)` can never be true.
   Likely cause: the earlier discount change left this condition behind.
   Suggestion: decide which rule you actually want; I have not changed it.

### Possibly used elsewhere (I can't see the other files)
- `exportReport()`: exported but not used in this file.

### Outside scope, not checked
- Older unused helpers exist in `legacy/` but were not touched by this branch.

Reply "remove 1 and 2" (or "remove all safe ones") and I'll do it.
```

Rules for the report:
- Number every item so the user can answer in a few words.
- Give file and line or function name so it can be found fast.
- If the audit finds nothing, say "Cleanup check: no leftover or suspicious code found in <scope>." in one line. Do not invent issues to look thorough.
- Keep each item to one or two lines. The user wants a scan-able list, not an essay.

## Never delete without asking

Do not remove anything from category A or B in the same turn you discover it, unless the user explicitly said to auto-clean. Removing code you believed unused is a common way to break working software, and the user's trust in this check depends on it never surprising them. Once they confirm, make the removal, then re-run the audit on the result, since deleting one thing often exposes another unused piece (a helper that was only called by what you just removed).

## Prevent leftovers while editing

When you replace behavior, remove the old version in the same edit if the user asked you to *replace* it. If you are unsure whether they want the old one kept, keep it, and list it in "Safe to remove" so the decision is theirs. Never leave both silently.
