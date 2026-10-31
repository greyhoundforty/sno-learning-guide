VERDICT: PASS

Scope: documentation only. Reviewed `docs/architecture.md` and `labs/schema.md` on `factory/001-lab-architecture` (round 1 fixes, commit `daa5a91`) against `spec.md`, `build.md` and my round 1 review. Both round 1 blockers are resolved. Nothing blocks UI implementation.

## Round 1 blockers: resolved

1. **Step state machine (was blocker 1): resolved.**
   - `docs/architecture.md:34-43` defines six states (`locked`, `pending`, `checking`, `pass`, `fail`, `error`). Each has a meaning, and the table says which are persisted (only `pass`).
   - `checking` disables the button, sets `aria-busy` and ignores stale responses (`:40`). This covers double-click and out-of-order responses.
   - `error` is separate from `fail` and never changes an existing `pass` (`:43`). `docs/architecture.md:47-58` maps each failure cause to an HTTP code, `error_kind` and guide message, and says errors show as a warning banner. The verifier-unreachable case, including the 10 s timeout, is covered (`:51`).
   - `labs/schema.md:57-67` gives a `status: pass|fail|error` response with examples, so the UI can branch on one field.
   - Hint order is stated (`docs/architecture.md:45`). `hint` is `""` on pass (`labs/schema.md:67`). Manual steps have `pass` with **Undo** (`:41`) and no API call (`:27`).
   - Reload behaviour is defined (`docs/architecture.md:62`): re-verify stored `pass` steps in order, demote on `fail`, lock dependents, leave state alone on `error`. The storage key and shape are given.

2. **Diagram-node contract (was blocker 2): resolved.**
   - Optional `diagram_node` field, defaulting to `id` (`labs/schema.md:23`).
   - SVG convention `data-step="<diagram_node>"` (`docs/architecture.md:68`).
   - Shared-node roll-up (`:69`): pass if all pass, fail if any fails, otherwise pending. `checking` and `error` do not change the node.
   - Missing-node fallback (`:70`): only the card updates. A node with no step is a lint warning.
   - The example uses `diagram_node: app` (`labs/schema.md:95`).

## Accessibility notes: adequate

`docs/architecture.md:71-75` covers the points I asked for in round 1:
- State shown by text or icon, not colour alone.
- Real `<button>` with a visible focus ring.
- `aria-live="polite"` region for states, hints and errors.
- SVG node `<title>` with step title and state.
- WCAG AA contrast.
- `viewBox` scaling, with cards and diagram stacking on narrow screens.

This is enough for a UI job to implement without inventing rules. Round 1 items 4 to 8 (responsive, manual steps, hint order, lab wrapper and order, per-lab storage key) are also addressed (`labs/schema.md:5-11,22,29`; `docs/architecture.md:41,45,62,75`).

## Non-blocking nits (decide during UI implementation)

- `docs/architecture.md:38`: a `locked` button is "disabled". Consider `aria-disabled="true"` so the "Complete <title> first" reason stays focusable and readable, or put the reason in text next to the button.
- `docs/architecture.md:69`: node state for `locked` is not named. Treat it as `pending`, or state that.
- `docs/architecture.md:62`: re-verify on load applies to stored `pass` steps, but manual steps have no API call. State that manual steps are restored from storage without a check.
- `docs/architecture.md:30`: "scrolls to it" should respect `prefers-reduced-motion`.
- `docs/architecture.md:51-56`: the guide message comes from the table, while the `error` response also carries `hint` (`labs/schema.md:64`). Say the client message wins, or that `hint` is ignored on `error`.
- `labs/schema.md:24`: `hint` is plain text, but its example contains backticks. Say whether inline code is rendered.
- `docs/architecture.md:42`: state whether the button is enabled again on `fail` and `error`. This is implied by "retrying is safe" (`:32`) but not written.
