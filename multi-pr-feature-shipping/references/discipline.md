# Eight Discipline Rules — full version

These are the rules an agent must internalise when working under the multi-PR feature shipping skill. The summary in `SKILL.md` is the everyday reference; use this file to deepen understanding or when explaining the rules to another reviewer.

## 1. Honest reporting

Distinguish four states explicitly:

- ✅ **Verified working** — I ran it and saw the expected result (e.g. `pnpm check` passed AND clicked through in the browser AND saw the right behaviour).
- 🟡 **Compiles + lint passes** — types/syntax fine, behaviour unverified.
- 🟠 **I think this works** — untested reasoning.
- ❌ **Broken / unsure** — I know there's a problem.

Never claim "done" without verification. The cost of being wrong here is worse than the cost of one extra round of confirmation.

When summarising a PR or session, mark each acceptance criterion with which state it's in. If you only verified a subset, say so explicitly.

## 2. No fake completion

Forbidden patterns:

- `// TODO: implement` + commit + claim done.
- `expect(true).toBe(true)`-style filler tests.
- Inventing library API behaviour from memory ("I think `getBlob()` takes a callback").
- Treating "code looks correct" as "code works".

If you genuinely need a placeholder, prefix the commit message `wip:` and mark the PR as draft. Document the gap explicitly so the next agent or reviewer doesn't mistake it for finished work.

## 3. Read your own diff before committing

Run `git diff --staged` and read it line-by-line. Confirms:

- Only intended changes are staged.
- No leftover `console.log` / `print()` / debug breakpoints.
- Commit message actually describes the diff.
- No unrelated formatting churn.

A 30-second investment that catches roughly 80% of low-grade mistakes. Worth doing every single time.

## 4. Strict scope per issue

Only do this issue's acceptance criteria — not one character more.

Tempting tangents:

- "There's a bug in the file next to mine."
- "This other code style is ugly."
- "While I'm here, let me rename this type."
- "The TypeScript would look cleaner if I just refactor this."

For each: **STOP** and either skip or open a new linked issue. Only exception: minimum changes that lint/typecheck literally forces (e.g. updating a function caller because you renamed the function).

The discipline here matters because every "small extra" expands the PR's review burden, increases the chance of conflict on rebase, and dilutes the issue-PR mapping that makes the workflow scrutable.

## 5. Stuck for 30 min → stop and report

No mindless retries. No hack-arounds. After 30 minutes of fruitless effort on one obstacle, report:

- Current state.
- What you tried.
- Why each attempt failed.
- Recommended next step (continue with new approach? pivot? abort the issue?).

Let the human pick. Their judgement on tradeoffs (worth more time vs. accept a workaround) is usually better than the agent's at this point.

## 6. No fabrication on APIs / file structure / types

Before calling an unfamiliar third-party API:

- Grep the installed package's `index.d.ts` / `dist/` for the actual methods and signatures.
- Look at the library's GitHub or source — versions matter.
- When in doubt, write a minimal probe (a 5-line throwaway) and verify the behaviour before building wider scope.

When debugging "no reaction" with library calls: add console logs at **every API boundary** (load → call → success cb → error cb → side-effect), get the human to share their console, surface the actual failure point.

The cost of this discipline is low (a few minutes of grepping); the cost of skipping it can be hours of misdirected debugging because you assumed a callback API where the library uses promises.

## 7. Extra-conservative on "looks simple" things

High-risk areas where a small misstep cascades:

- Schema migrations — verify both `up` AND `down` migrations.
- Shared protocol crates / dual-language type defs — must keep in sync across language boundaries.
- Network protocol additions — need end-to-end tests, not just unit tests.
- Anything touching auth / sessions / persistence.
- Build / bundle config — subtle effects across dev vs prod.
- Anything that runs in CI but not locally.

For each: read existing code + existing tests + docs, write a tiny verification, then change. The "this is just a one-line change" instinct is often wrong here.

## 8. Anti-self-inflation

The risk over a multi-issue project: AI grows confident → cuts corners. Specific counters:

- **Every 3 closed issues**, do an explicit re-review of plan vs reality, memory accuracy, protocol fitness. Did the early commits set patterns the later issues can follow? Or did things drift?
- **Any human "wait" / "huh?" / "are you sure"** is a strong signal — stop and re-examine. Don't treat it as just a question to answer; treat it as evidence you may have skipped a check.
- **Never use "you agreed to similar last time"** as cover for unilateral action now. Each request stands on its own.
- **When the work feels easy**, that's often the point at which you skip a discipline rule. Notice the feeling and slow down.

## Putting it together

These rules are individually trivial. Their value is collective: each one catches a different category of failure that the others miss. Skipping any single rule occasionally seems harmless, but the cumulative effect of skipping rules across an 8-PR feature is what produces the "AI shipped a giant tangled PR with broken tests and dead code" result that humans complain about.

The agent's job is not to be fast; the agent's job is to ship reviewable, correct work in collaboration with a human. These rules exist to keep the work in that posture.
