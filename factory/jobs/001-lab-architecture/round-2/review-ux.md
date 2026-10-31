VERDICT: PASS

Scope: `docs/architecture.md` and `labs/schema.md` on `factory/001-lab-architecture`, read against `spec.md`, `build.md` (including the Round 1 section), `round-1/review-ux.md`, and `git diff main...factory/001-lab-architecture`. Line numbers refer to the files on the branch. For a design-level doc, all three blocking items and all five should-fix items are resolved. I found no new blocking problems.

## Blocking items from Round 1

### B1. Error vs fail: resolved
- `labs/schema.md:57-67` makes the response a three-way `status` (`pass`, `fail`, `error`). `error` carries an `error_kind`. Only a check that actually ran returns HTTP 200 with `pass` or `fail`.
- `docs/architecture.md:34-45` adds the step-state table. `fail` (the work is not right) and `error` (the check could not run) are separate states. `error` "never changes an existing `pass`", so a stopped VM cannot undo a learner's progress.
- `docs/architecture.md:47-58` has an Errors table covering every case I raised:
  - verifier unreachable, CORS rejection, and the 10 s timeout (line 51);
  - cluster down or TLS failure (52);
  - 401 (53);
  - 403 (54);
  - unknown step and invalid `expected` (55-56).
  Each row gives the HTTP code, the `error_kind`, and the guide message. The messages are actionable.
- Errors render as a warning banner, distinct from `fail` (line 58). `GET /api/health` gives cluster status on load (line 58). This is what I asked for.

### B2. Ordering and prerequisites: resolved
- `labs/schema.md:3,11` and `29` say order is list order. `labs/schema.md:22` adds an optional `requires` that defaults to the previous step, with `[]` meaning none.
- `labs/schema.md:29` says dangling or forward references are rejected.
- `docs/architecture.md:38` defines the `locked` state with the "Complete <title> first" message.
- `docs/architecture.md:30` says that on a pass the next step opens and scrolls into view.
- The example uses `requires: [create-project]` (`labs/schema.md:94`). The route now includes the lab id (`architecture.md:28`), which also resolves S5.

### B3. Reset: resolved
- `docs/architecture.md:60-62` specifies:
  - a `lab:<lab-id>:progress` key with a version field (`{ "v": 1, ... }`);
  - re-verification of stored `pass` steps on load, in order;
  - demotion of a step that now fails, with its dependents becoming `locked`, and an `error` leaving state alone;
  - a **Reset lab** action that clears the key and "never touches the cluster".
- `labs/schema.md:25` adds an optional text-only `cleanup` field, which preserves the no-shell rule. The example uses it at line 81.
- This also closes S1: the key, the lab wrapper (`labs/schema.md:5-11`), and the schema version are all present.

## Should-fix items

| Item | Status | Evidence |
| --- | --- | --- |
| S1 persistence key and format | Resolved | `architecture.md:62`, `schema.md:5-11` |
| S2 hint semantics | Mostly resolved | Hint order is stated at `architecture.md:45` and `schema.md:24`. Per-step multiple hints and a success message were deliberately dropped to stay under the word limit (`build.md:61`). I accept this for a design doc; see N1. |
| S3 transient UI states | Resolved | `checking` state with a disabled button and `aria-busy`, stale responses ignored (`architecture.md:40`), 10 s timeout (`architecture.md:51`) |
| S4 diagram contract | Resolved | `diagram_node` (`schema.md:23`) and the diagram rules table (`architecture.md:64-75`). It covers the `data-step` attribute, shared-node roll-up, and missing-node behaviour. Manual steps are covered by the pass row at line 41. |
| S5 single source of truth and lab-scoped route | Resolved | `architecture.md:28-29` |

The minor items were also picked up:
- The accessibility rules cover not relying on colour alone, a real `<button>`, an `aria-live` region, SVG `<title>`, AA contrast, and `viewBox` (`architecture.md:71-75`).
- The example image is now a real, pullable one (`schema.md:89`).
- Retry safety is stated (`architecture.md:32`).

## New problems introduced

None blocking. The new anchors resolve: `../labs/schema.md#response` and `#resource_exists-kinds` match headings at `schema.md:55` and `41`. The YAML example is a full lab with two non-manual steps and one manual step (`schema.md:71-107`), so criterion 6 holds.

## Non-blocking nits (for a later pass or a follow-up)

- N1. Because the verifier always returns a specific hint on `fail`, the step-level `hint` (`schema.md:24`) will rarely be shown. The doc does not say that checks must return state-specific messages, for example "exists but 0/1 pods ready". Add one line under Verifier rules (`architecture.md:77-84`). A short optional success note remains a possible later addition.
- N2. `/api/health` reports `cluster` as `ok`, `cluster_unreachable`, or `auth` (`architecture.md:58`). It omits `forbidden`, which is in the Errors table at line 54. Align the two lists.
- N3. CORS allows only `http://127.0.0.1:<GUIDE_PORT>` (`architecture.md:83`). A learner who opens `http://localhost:8080` will get a CORS failure. The guide will then show "Verifier isn't running" (line 51), which is misleading. State that the `mise` task prints the 127.0.0.1 URL, or add a hint to that message to use the printed URL.
- N4. Reset and re-verify details left open:
  - whether a demoted step is also removed from the stored `done` list (`architecture.md:62`);
  - whether manual steps are re-checked on load (they cannot be);
  - whether **Reset lab** asks for confirmation;
  - where the `cleanup` text is displayed (in the card, or only after a reset).
  These are implementation decisions for the guide job.
- N5. The docs refer to "lint" (`schema.md:29`, `architecture.md:70`), but no lint tool or task is defined. This is fine as a pointer to a later job. Add it to the backlog.
- N6. `schema.md:19` has no convention of "goal sentence, commands, expected-result line" for `instructions`. The example (`schema.md:85-92`) has no expected-result line.
