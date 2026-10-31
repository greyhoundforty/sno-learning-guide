VERDICT: CHANGES

Scope: documentation only (`docs/architecture.md`, `labs/schema.md` on `factory/001-lab-architecture`, commit 767ec6f). No UI is implemented, so the checks below are about whether the schema and architecture give the guide UI what it needs. Line numbers refer to the files on that branch. Two items are marked blocking. The rest are non-blocking.

## Blocking (schema/doc gaps that would stall UI work)

1. **Step state model is incomplete (pending/checking/pass/fail).**
   - `docs/architecture.md:26-36` defines only two outcomes: `pass: true` turns the card "complete", and `pass: false` keeps the card "open" and shows `hint`. There is no named `pending` or `checking` state. The state for the time between the click and the response is missing.
   - Nothing says the "I did it" button is disabled or marked busy during the request. Without that, double-clicks and out-of-order responses can happen.
   - `fail` is not defined as a persisted or visible state. `docs/architecture.md:36` persists only "completed step ids", so the UI cannot restore a failed or pending state after a reload.
   - Verifier unreachable, network error, HTTP 4xx/5xx and a step the verifier rejects for an invalid `expected` map (`labs/schema.md:16`) are not covered. `labs/schema.md:29-33` defines only `{pass, hint}`. The guide needs a defined fourth `error` state, or a rule that these cases map to `fail` with a client-generated message. Otherwise a stopped verifier looks like a learner failure.
   - Fix: add a short "Step states" table to `docs/architecture.md` (for example `pending`, `checking`, `pass`, `fail`, plus the error handling above), and name which states are persisted.

2. **Diagram node mapping is implicit and one-to-one only.**
   - `labs/schema.md:9` says `id` is "used … as the SVG node key", and `docs/architecture.md:35` says "the matching SVG node changes state". That is the only link between a step and the diagram.
   - The docs do not say that every step must have a node in the SVG. They do not say what an element looks like (for example `id="node-<step-id>"` or `data-step="<step-id>"`). They do not say what the guide does when a step has no node, such as `manual` steps or steps that do not map to a component.
   - Several steps lighting one node, or one step lighting several, cannot be expressed.
   - Fix: either keep one node per step and state the convention and the fallback rule in `docs/architecture.md` (about 2 lines), or add an optional `diagram_node` field to the fields table at `labs/schema.md:7-14`, defaulting to `id`. The first option is enough if it is written down.

## Non-blocking (should be recorded before UI implementation)

3. **No accessibility requirements.** Neither document mentions accessibility. The UI job will invent the rules unless they are written down. Add a short list to `docs/architecture.md`, near line 35:
   - Each state needs a text or icon label, not just a colour change. This applies to both the step card and the SVG node (for example "Passed" and "Failed" text plus a distinct shape or icon).
   - "I did it" is a real `<button>`, reachable by Tab and activated by Enter and Space, with a visible focus state.
   - State changes and the `hint` are announced, for example with `aria-live="polite"`.
   - SVG nodes carry accessible names (`<title>` or `aria-label`) that include the step title and its state.
   - Contrast meets WCAG AA.

4. **No responsive guidance.** Nothing covers narrow viewports. Add one line saying the SVG scales with `viewBox`, that the step cards and the diagram stack on small screens, and that node labels stay legible. The diagram is inline in a single file (`docs/architecture.md:9`), so this is cheap to state now.

5. **Manual steps have no state definition.** `docs/architecture.md:27` says the guide marks the step "done client-side" without a `checking` phase. The docs do not say whether the step can be un-done or whether the node uses the same `pass` look. Say so, because `expected: {}` for `manual` (`labs/schema.md:13,59`) carries no other information.

6. **`hint` is ambiguous in the UI.** `labs/schema.md:14` defines the step `hint` as a fallback "when the check gives no specific message", while the response `hint` (`labs/schema.md:32`, `docs/architecture.md:32`) is the verifier's message. The docs do not state the display order (response `hint`, then step `hint`, then a generic message) or that `hint` is `""` on pass. The `hint` is also a plain string; if the UI should render inline code (the example at `labs/schema.md:53` contains backticks), say so.

7. **Lab-level and step-order metadata is missing.** The schema is a bare list of steps (`labs/schema.md:3`). There is no lab title, no explicit ordering rule (list order is assumed), and no reference to which SVG file or diagram a lab uses. The UI can work with list order, but the docs should state it.

8. **Persistence key and shape are unspecified.** `docs/architecture.md:36` says `localStorage` keeps completed ids but not the key name or the lab scoping. Two labs that use the same step ids (such as `create-project`) would collide. Suggest a per-lab key.

## What already works

- Step `id` is a stable slug used in the API path and as the node key (`labs/schema.md:9`).
- The check-and-update flow is clear (`docs/architecture.md:26-35`). The click-to-complete behaviour, the guide never being opened as `file://` (`docs/architecture.md:13`) and the CORS origin (`docs/architecture.md:45`) are all specified, so no UI-side surprises there.
- `instructions` is markdown with fenced code blocks (`labs/schema.md:11`), which gives the UI a clear rendering contract.
