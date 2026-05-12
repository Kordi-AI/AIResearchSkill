# Common Failure Modes — extended

Each entry is a real failure pattern observed when an agent ships multi-PR features. Symptom / cause / prevention.

## Library API assumptions

**Symptom.** "No reaction when I click the button"; no error in the console; the call appears to complete but produces no output.

**Cause.** Assumed an API shape that has changed between versions. Common examples:

- Old version: `lib.method(callback)` — new version: `lib.method()` returns a Promise. The old callback path silently swallows the call.
- Old version: `lib.config = { ... }` — new version: `lib.configure({ ... })`. Direct assignment is ignored or stored on a no-op property.
- Old version: default export is the API — new version: named export. The default-import pulls a config stub.

**Prevention.**

1. Before calling unfamiliar lib methods, grep the installed package's `dist/index.d.ts` (or `dist/index.js`) for the actual signature.
2. When in doubt, add console logs at every step ("calling X" / "X returned: <shape>" / "callback fired") so a hang surfaces immediately.
3. If the lib is brand new in the project, write a five-line probe in a throwaway file first to verify behaviour before integrating into the real call site.

## Cross-worktree Edit failures

**Symptom.** "File has not been read yet" error from the agent's Edit tool when starting work in a freshly-created git worktree.

**Cause.** Many agent file-edit tools track Read/Edit state per session and per filesystem path. Switching to a new worktree means even files at the same logical path (different absolute path) need a fresh Read before Edit will succeed.

**Prevention.** First time you touch each file in a new worktree, Read it first — even if you've read the same file at a different path in the prior worktree.

## Bundle budget for heavy deps

**Symptom.** Bundle size budget script fails after adding a dependency.

**Cause.** Any single library > 200 KB raw will likely fail a 700 KB initial-chunk budget when bundled with siblings.

**Prevention.**

1. Before adding the dep, check its size: `du -sh node_modules/.../dist/`.
2. Plan `React.lazy(() => import(...))` (or equivalent dynamic import) from the start, not after the budget fails.
3. Add a chunk group in `vite.config.js` (or your bundler equivalent) so the dep clusters with similar lazy-only code (`canvas-vendor`, `download-vendor`, etc.).
4. If the lazy chunk genuinely exceeds the global budget because the lib is genuinely huge, add a per-chunk override in the budget script. **Document why** in the override comment so the next reviewer understands.

Initial-download chunks must stay under the global budget. Lazy chunks may exceed it if the dep is unavoidable and the user only loads it when they actually use the feature.

## Locale / Unicode coverage

**Symptom.** Characters render as empty boxes (tofu) — typically CJK, Arabic, Hebrew, or emoji.

**Cause.** The default font in the chosen library has no glyphs for the user's content. Examples: pdfmake's bundled Roboto has no CJK; many Western fonts have no Arabic; emoji often need a separate emoji font.

**Prevention.** Before committing to a text-rendering library, check which scripts and glyph ranges its default font covers. If the v1 ships Latin-only, file a follow-up issue immediately to extend coverage.

## Debug logging hygiene

**Symptom.** Production code leaks `console.log('debugging XYZ')` or commented-out code, often visible in PR review.

**Cause.** Forgot to remove temporary logs after the bug was found. Easy to forget when the fix is in a different file from the logs.

**Prevention.**

- Prefix all temp logs with the feature name (`[scratch/pdf] ...`) so they're greppable.
- Remove logs in the **same commit** as the fix, OR split into a dedicated "remove debug logs" commit. Either way they don't ship to main.
- A pre-commit hook that warns on `console.log` or `print` in modified files catches the rest.

## "Just one more retry"

**Symptom.** Same approach attempted four times in a row, each with a slight tweak, none working.

**Cause.** Didn't actually understand the failure; trying mutations rather than investigating.

**Prevention.** If you've tried twice and the third attempt isn't based on **new evidence**, stop. Add diagnostic logs, read source, ask the human. Velocity from confidence beats velocity from grinding.

A useful rule of thumb: if you can articulate a hypothesis about why the current attempt will succeed where the last didn't, try it. If you can't articulate a hypothesis, you're just hoping — stop hoping and gather evidence.

## Scope creep through "small additions"

**Symptom.** PR ends up touching unrelated files; the reviewer asks "why is this here?"

**Cause.** While debugging or implementing, noticed adjacent issues and "fixed them too" because they felt cheap.

**Prevention.** Every adjacent fix is a separate issue. Even one-liners. The discipline of writing the issue title forces you to articulate the scope, which usually reveals that the "small fix" actually intersects with other things you weren't planning to touch.

When tempted: open the new issue, link it in the current PR's description, move on. The two-minute cost of filing the issue is worth the saved review confusion.

## Stale lazy chunk on HMR

**Symptom.** After fixing a bug in code that's behind a `React.lazy(() => import(...))` boundary, the browser still shows the broken behaviour even after HMR runs.

**Cause.** Module-level cached promises (`let ready: Promise<X> | null = null`) don't get re-evaluated by HMR; they hold onto the previous module instance.

**Prevention.** Hard refresh (`Cmd+Shift+R`) when changing code inside lazy boundaries. If a smoke-test bug seems impossible after a fix, ask the user to hard refresh first before debugging further.

## Identity / git config drift

**Symptom.** Commits attributed to the wrong author (e.g. machine hostname instead of GitHub email).

**Cause.** No local repo `user.name` / `user.email` set, falling back to system defaults.

**Prevention.** Before the first commit on a new repo, verify identity with `git -C <worktree> config user.email`. If wrong:

- **Don't update global git config** without explicit human authorization — that affects every other repo on the machine.
- Set local repo config only: `git config user.name "..."` and `git config user.email "..."`.
- For a single commit fix, `git commit --amend --reset-author --no-edit` works after the local config is set.

## Misleading PR descriptions

**Symptom.** PR claims to implement X, but X doesn't actually work or isn't fully done.

**Cause.** Wrote the PR description before the final verify step; the reality of the diff diverged from the plan.

**Prevention.** Write the PR description from the actual diff, not from the original issue. Each acceptance criterion should be tickable based on what's verifiably true in the branch *now*. If you can't tick it, don't claim it.

## Putting it together

Most of these failures share a root cause: the agent assumed something about state (the library's API, the bundler's behaviour, the cache's freshness, the test's actual coverage) without verifying. The fix in every case is to add a verification step at the boundary where the assumption lives.

Update this file when you encounter a new failure mode worth surfacing for future agents.
