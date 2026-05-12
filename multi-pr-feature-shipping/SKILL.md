---
name: multi-pr-feature-shipping
description: Discipline and workflow for shipping a multi-issue feature as a series of small sub-PRs into a long-lived integration branch. Use when starting a feature that spans 3+ issues, when establishing collaboration norms with a human reviewer, or when you need to keep main stable while iterating. Covers branch strategy, the per-issue six-step cycle, always-ask gates, eight discipline rules, common failure modes, and a debugging methodology.
---

# Multi-PR Feature Shipping

A working contract between a human reviewer and an AI coding agent for shipping a feature that spans multiple issues. Treat every rule as load-bearing; the value is in the discipline, not the words.

## When to use this skill

- A feature is large enough to be split into 3+ GitHub issues.
- Main branch must stay stable and reviewable while the feature is in progress.
- You want each issue to ship as its own small, reviewable PR rather than one giant merge.
- You need to establish working norms with a human reviewer (commit cadence, push gates, scope discipline).

## Working agreement

- Plan before code. Surface architectural decisions as **option tables** with explicit tradeoffs and a recommendation; let the human pick.
- Cap each planning round with a small numbered list of "things I need you to confirm" (3–5 items max).
- Don't write code until the human says go. Brief acknowledgements like "ok / sure / 可以" count as green lights; broader silence does not.
- Tone: substantive but scannable — tables, headers, terse bullets.

## Branch strategy

- One **long-lived integration branch** off `main`, named after the feature (e.g. `feat/<feature>`).
- Every sub-PR targets the integration branch, **never** `main` directly.
- Each sub-issue gets its own sub-branch AND its own git worktree:

  ```bash
  git worktree add -b feat/<feature>-NN-<slug> .claude/worktrees/<feature>-NN-<slug> origin/feat/<feature>
  ```
- Naming: include the issue number/slug (`-01-shell`, `-02-editor`, etc.) so PR order is obvious.
- Final big merge `feat/<feature> → main` happens once after all sub-PRs close.
- Rebase the integration branch from main every 2–3 closed sub-PRs to avoid drift.

## The six-step cycle per issue

```
[1] Setup → [2] Implement → [3] Self-check → [4] Commit → [5] Push → [6] Open PR
```

| Step | Action | Confirm with human? |
|---|---|---|
| 1 | Create sub-branch + worktree; copy issue acceptance criteria into a todo list | Auto |
| 2 | Code in small steps; tick each todo as it lands | Auto |
| 3 | Run `lint` / `typecheck` / `tests`; read own staged diff line-by-line; UI changes → start dev server, click through | Auto |
| 4 | **Commit** | **Always ask** — show files staged + commit message draft |
| 5 | **Push** | **First push of branch: ask. Subsequent normal pushes: auto. Force push: always ask** |
| 6 | **Open PR** | **Always ask** — show PR title + body draft |

## Always-ask gates (never auto)

- Any commit
- First push of a branch / any force push / rebase that rewrites history
- Installing a new dependency
- Schema migrations / config file changes / lockfile-shape changes
- Cross-issue "while I'm here" fixes — open a new linked issue instead
- Any departure from a design previously locked in
- Any destructive git op (`reset --hard`, branch delete, file delete, `--no-verify`, `--amend`)
- Modifying git config (and prefer local repo scope, never global)

## Auto-allowed (don't ask)

- Reading files, running grep / find / lint / typecheck / tests
- Writing code in the active worktree
- Starting/stopping dev server for verification
- Updating local memory / tracking files
- Maintaining a todo list

## Eight discipline rules

The summary below is enough for everyday work. For the full version with examples and counter-examples, see [references/discipline.md](references/discipline.md).

1. **Honest reporting.** Distinguish *verified working* / *compiles + lint passes* / *I think this works* / *broken*. Never claim done without verification.
2. **No fake completion.** No `// TODO: implement` + claim done. No filler tests. No invented library APIs from memory.
3. **Read your own diff before committing.** `git diff --staged` line-by-line catches 80% of low-grade mistakes.
4. **Strict scope per issue.** Tempting tangents → STOP and either skip or open a new linked issue.
5. **Stuck for 30 min → stop and report.** No mindless retries. State current state, what was tried, why it failed, recommended next step.
6. **No fabrication on APIs / types / file structure.** Grep the installed package's `dist/` or `index.d.ts` for the actual signature. When unsure, write a minimal probe before building wider scope.
7. **Extra-conservative on "looks simple" things.** Schema migrations, shared protocol changes, network protocol additions, anything touching auth / sessions / persistence.
8. **Anti-self-inflation.** Every 3 closed issues, re-review the plan vs reality. Any human "wait" or "are you sure" is a strong signal — stop and re-examine.

## Commit & PR conventions

- Commit message: `<type>(<scope>): #<issue> <imperative summary>`
- Many small commits > fewer big ones. One thing per commit.
- Commit body: 1–3 sentences explaining the *why*. The *what* is in the diff.
- PR title: matches the issue title verbatim.
- PR body:
  - First line: `Closes #<issue>` (so GitHub auto-closes on merge to default branch).
  - Acceptance-criteria checklist copied from issue, with the boxes ticked.
  - "Out of scope (follow-up)" section linking any spawned follow-up issues.
  - "Known limitations" section if anything ships imperfect.

## Common failure modes

The big ones are summarised here; for the full set with cause / prevention pairs see [references/failure-modes.md](references/failure-modes.md).

- **Library API assumptions.** Symptom: silent hang. Cure: grep installed `dist/` for the actual signature before calling unfamiliar methods.
- **Bundle budget for heavy deps.** Symptom: chunk budget script fails. Cure: lazy-import any dep > 200 KB raw + dedicated chunk group.
- **Locale / Unicode coverage.** Symptom: tofu boxes. Cure: verify default font's script coverage before committing to a text-rendering library.
- **Debug log leakage.** Cure: prefix all temp logs with the feature name; remove in the same commit as the fix.
- **"Just one more retry."** Cure: after the second failed attempt with no new evidence, stop and add diagnostic logs.
- **Scope creep through "small additions."** Cure: every adjacent fix is a separate issue; the discipline of writing the issue title forces you to articulate scope.

## Debugging methodology

When something doesn't work as expected — see [references/debugging.md](references/debugging.md) for the full guide. Quick version:

1. Reproduce on a clean reload — confirm it's not a stale cache / HMR issue.
2. Add diagnostic logs at every API boundary, prefixed with the feature name (`[feature/format] ...`).
3. Get the human to share their console / logs — the actual failure point usually surfaces in 1–2 iterations.
4. Once root cause is clear, fix and remove the logs in the same commit.
5. Update this skill (or your team's protocol) if the failure mode is new.

## Follow-up issues

When user feedback during smoke test reveals legitimate work that's out of the current PR's scope:

- File a follow-up issue **immediately**.
- Title prefix: `<type>(<scope>): [follow-up to #N] <summary>`.
- Link from the original PR description AND the parent issue's comments.
- Continue the current PR; don't let the follow-up block ship.

## Memory / state for cross-session continuity

If the agent has a persistent memory store, maintain a project memory file with:

- Status section: which issues are merged / open / not started.
- Locked design decisions: things the human committed to that should NOT be re-litigated.
- GitHub artifacts table: branch names, PR numbers, issue numbers.
- Critical path: dependency order across issues.
- Known risks / open questions.

Update before any meaningful pause. A future agent reading this file should be able to pick up cold and know where you left off.

## Quick reference card

| Before doing this | Confirm with human? |
|---|---|
| Read / grep / lint / typecheck / dev server start | No |
| Write or edit code in the active worktree | No |
| Update memory or todos | No |
| **Commit** | **YES** (every time) |
| First push of a branch | **YES** |
| Subsequent push (non-force, same branch) | No |
| Force push / rebase rewriting history | **YES** |
| **Open PR** | **YES** (with title + body draft) |
| Install a new dep | **YES** |
| Edit a config file (vite, tsconfig, lockfile) | **YES** |
| Cross-issue "while I'm here" fix | **YES** (default to filing a new issue instead) |
| Any destructive git op | **YES** |
| Modify git config | **YES** (local repo scope only, never global) |

## Hard stop word

If the human types `STOP` (any case), immediately halt all in-flight operations. No further actions until told to resume. This is a non-negotiable circuit breaker.

## Additional resources

- [references/discipline.md](references/discipline.md) — full text of the eight discipline rules with examples.
- [references/failure-modes.md](references/failure-modes.md) — detailed failure modes with cause / prevention pairs.
- [references/debugging.md](references/debugging.md) — debugging methodology with the boundary-log pattern.
