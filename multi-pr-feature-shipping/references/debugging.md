# Debugging Methodology — boundary log pattern

When something doesn't work as expected, the agent's instinct is often to tweak and retry. That instinct is usually wrong. The faster path is the boundary log pattern below.

## Five-step protocol

### 1. Reproduce on a clean reload

Confirm it's not a stale HMR / cache / build artifact issue. Hard-refresh the browser, restart the dev server, clear the bundler cache. Many "bugs" disappear here and you save the next four steps.

If the bug persists after a hard refresh, it's a real bug. Continue.

### 2. Add diagnostic logs at every API boundary

Use a feature-prefixed tag so the logs are greppable and removable. Boundary points to log:

- "calling X" — right before the unfamiliar API call.
- "X returned: <shape>" — right after, log the actual return value.
- "promise resolved: <value>" — for async results.
- "callback fired with <args>" — for callback APIs.
- "error caught: <err>" — in every catch block.

Example:

```ts
console.log('[scratch/pdf] calling createPdf');
const pdfDoc = pdfMake.createPdf(docDef);
console.log('[scratch/pdf] createPdf returned, pdfDocumentPromise?', !!pdfDoc.pdfDocumentPromise);
pdfDoc.pdfDocumentPromise?.then(
  (val) => console.log('[scratch/pdf] resolved:', val),
  (err) => console.error('[scratch/pdf] rejected:', err),
);
console.log('[scratch/pdf] calling getBlob');
pdfDoc.getBlob((blob) => {
  console.log('[scratch/pdf] getBlob cb fired, size:', blob?.size);
});
```

The pattern: log the **shape** of the value (typeof, keys, sizes) at every step. Don't just log "it ran" — log enough to see whether the value matches your assumption.

### 3. Get the human to share their console

Ask the human to:

- Open the browser DevTools console (or the relevant log surface for non-web apps).
- Trigger the bug.
- Copy the full output, including any rows that don't have your prefix (other errors might be the real issue).

The actual failure point usually surfaces in 1–2 iterations of this pattern. The shape mismatch is visible in the logs:

- "I expected `pdfMake` to have `createPdf`, but `Object.keys(pdfMake)` shows only config fields" → the import shape is wrong.
- "I expected the promise to resolve quickly, but `getBlob cb fired` never appears" → the callback isn't being called; check the API signature.
- "I expected `vfs` to have font keys, but it's empty" → the assignment isn't reaching the right object.

### 4. Once root cause is clear, fix and remove the logs in the same commit

Don't ship debug logs to main. Two acceptable patterns:

- **Same commit as the fix** — clean, atomic. The fix and the cleanup are one unit.
- **Dedicated "remove debug logs" commit** — slightly less atomic but easier to keep the fix commit pure.

Either way: don't open a PR with `console.log` calls in production code paths.

### 5. Update the protocol if the failure mode is new

If the failure surfaced a new gotcha (like "this library's getBlob() became async in v0.3"), add it to the team's `failure-modes.md` reference. Future agents (including future-you) avoid the same trap.

## What not to do

These are the anti-patterns that this protocol exists to replace:

### Don't guess at fixes without evidence

If you find yourself thinking "let me try this approach instead", ask: what evidence do I have that this approach will work where the last didn't?

If the answer is "I just have a feeling", you're guessing. Stop and gather evidence first.

### Don't iterate purely on lint errors

Lint errors tell you what the type system thinks. They don't tell you the runtime story. A clean lint pass after three rounds of fixes is not the same as the bug being gone — verify the actual behaviour separately.

### Don't mark a task done because "it should work now"

"Should work" is an unverified claim. Always confirm the actual behaviour matches the expectation. The cost of one extra verification round is much lower than the cost of telling the human "it works" when it doesn't.

### Don't drown the human in logs

Add the boundary logs you need, ask the human to share the output, then **remove the logs**. Don't leave six rounds of debugging instrumentation in the codebase. Clean as you go.

## Calibrating diagnostic depth

How much logging is "enough"?

- **First diagnostic round**: log every boundary in the call path. Better to over-log and narrow down than to under-log and need another round.
- **Second round (after first round shows where it fails)**: log inside the failing call, e.g. log the inputs and outputs of the function that didn't behave as expected.
- **Third round**: if you're still guessing, you're missing context — read the library source.

Three rounds is usually enough. If you're at round four, the issue is probably not where you're looking. Step back and reconsider the assumption you started with.

## When to skip this protocol

Some bugs are obvious from the symptom alone:

- Off-by-one indexing — just check the loop bounds.
- Typo in a variable name — TypeScript usually catches this; if not, grep.
- Wrong function called — review the call site.

For obvious bugs, fix them directly without the boundary log dance. Reserve the protocol for the cases where you'd otherwise be tempted to "just try one more thing".

## In practice

The boundary log pattern is the antidote to confidence-driven debugging. The agent's confidence in its mental model of the library / framework / system is exactly what gets it stuck — by adding logs at the boundaries between trusted code and external code, you collapse confidence into evidence, and from evidence the fix is usually obvious.

It's not flashy. It works.
