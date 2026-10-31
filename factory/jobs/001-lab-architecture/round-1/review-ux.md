VERDICT: CHANGES

Scope: `docs/architecture.md` and `labs/schema.md` at `factory/001-lab-architecture` (767ec6f), judged against `spec.md`, `build.md`, and the OVERVIEW.md goal (click "I did it" -> server verifies against the cluster -> diagram updates). Line numbers refer to the files on the branch.

The happy path is well specified: `POST /api/verify/<step-id>`, `{pass, hint}`, step card and SVG node update, `localStorage` persistence, and manual steps resolved client-side. The gaps are in the unhappy paths and in step flow. They are cheap to fix now and expensive once every lab job copies the schema.

## Blocking (must fix)

### B1. "Cluster unreachable" cannot be told apart from "check failed"
- `docs/architecture.md:29-35` and `labs/schema.md:29-33` define one response shape, `{ "pass": bool, "hint": string }`. It has two outcomes, pass and fail.
- No other outcome is defined for these cases:
  - Verifier not running, or CORS rejected. The guide's `fetch()` throws.
  - Verifier up but the cluster is unreachable. This covers a stopped SNO VM, a bad `API_SERVER_URL`, TLS failure, an expired `SA_TOKEN`, and a 401.
  - RBAC 403, for example when a later lab needs a kind not in the ClusterRole (`architecture.md:109`).
  - Unknown step id or an `expected` map that fails validation (`architecture.md:29`, `schema.md:16`).
- Without a distinction, the learner sees "not done yet" for an infrastructure problem. They will redo correct work and lose trust in the guide. The hint for "Project demo was not found." (`schema.md:32`) is a correct failure message. The same message from a VM that is powered off would be actively misleading.
- Fix: extend the contract with a third state and tell the guide how to render each state.
  - Option A: `{ "status": "pass" | "fail" | "error", "hint": "...", "error_kind": "cluster_unreachable" | "auth" | "forbidden" | "bad_step" }`.
  - Option B: keep `pass`, add `error` with a non-2xx status (503 for cluster unreachable, 404 for unknown step).
  - Either way, specify the guide behaviour:
    - Fail: the card stays open and shows the hint. It is a normal state, in a neutral or warning style.
    - Error: a visibly different banner, "Can't reach your cluster. Is the VM running? Check KUBECONFIG." The step is not marked failed and is not counted as an attempt.
    - Fetch throws: "Verifier isn't running on localhost:<VERIFIER_PORT>. Start it with <mise task>."
  - Add a short sentence in `architecture.md` Cluster access (around line 57) about a cheap health endpoint such as `GET /api/health`. The guide can call it on load and show cluster status before the learner clicks anything.

### B2. No step ordering or prerequisites
- `labs/schema.md:3` says "a YAML list of steps". Order is implied by list position only. There is no `requires` field, no statement that steps render in list order, and no statement on whether later steps are locked.
- The example `deploy-hello` (`schema.md:38-53`) silently depends on a `demo` project that an earlier step creates. If a learner skips ahead and clicks "I did it" on `deploy-hello`, the failure comes from a missing namespace, with no pointer to the real cause.
- The next action after a pass is also unspecified (`architecture.md:35`). It does not say whether the next card expands, scrolls into view, or unlocks.
- Fix: in `schema.md` define the following.
  - Order is list order.
  - Add an optional `requires: [step-id, ...]` field. Default is the previous step.
  - State the guide behaviour: locked steps show "Complete <title> first", and on pass the next step opens and scrolls into view.
  - State that `requires` ids must exist within the lab, so the verifier or a lint can reject dangling references.

### B3. Reset is not addressed anywhere
- `architecture.md:36` persists "completed step ids" in `localStorage`. Nothing covers resetting a step, a lab, or the cluster state behind it.
- This matters for a learning lab because learners repeat exercises and the cluster is stateful.
- Two failure modes follow:
  - A learner deletes the `demo` project to retry. The guide still shows the step complete and the diagram still shows it done. The state is stale and nothing re-verifies it.
  - A learner clears or switches the browser. Progress is lost while the cluster is unchanged.
- Fix, minimum for this job:
  - Specify a per-lab "Reset progress" action in the guide that clears the lab's `localStorage` key.
  - Specify that the guide re-verifies completed steps on load, or offers a "Re-check all" button, so the diagram reflects the cluster rather than the browser's memory.
  - Say explicitly that resetting the guide does not touch the cluster. Add an optional per-step `cleanup` or `reset` instruction field in markdown (a text instruction, not an executed command, to preserve the no-shell rule at `architecture.md:43`), or record it as a follow-up backlog item.

## Should fix

### S1. Progress persistence key and format are underspecified
- `architecture.md:36` says "completed step ids" in `localStorage` but gives no key or schema.
- Two labs that reuse a step id such as `create-project` would collide.
- Specify the key as `lab:<lab-id>:progress`. This requires a lab-level `id`, which `schema.md` does not define because a lab is only "a YAML list of steps" (`schema.md:3`).
- Say whether the file wraps the list in an object (`id`, `title`, `steps`) so the guide can show a lab title and diagram.
- Add a schema version so progress can survive step renames or removals.

### S2. Hint semantics are ambiguous and too coarse
- `schema.md:14` says `hint` is "text shown on failure when the check gives no specific message". The response `hint` (`architecture.md:31-33`, `schema.md:32`) is the thing the learner sees. The precedence between verifier-generated and author-supplied hints is stated only in that one field description.
- Define the contract in `architecture.md` Verifier rules:
  - Each check must return a specific, state-based message. Examples: "Deployment hello exists but 0/1 pods are ready", "Project demo was not found", "Namespace exists but replicas is 2, expected 1".
  - The author's `hint` is appended or used as a fallback.
- A single static `hint` per step cannot cover the failure modes of `deployment_ready`: missing, not ready, or wrong replica count. Consider a list of hints, or state that the verifier's specific message carries this role.
- On pass, `hint` is `""` (`architecture.md:32`). Consider an optional `success` message or a "what you should see" note so a pass is reinforced, not just a colour change.

### S3. Transient UI states during verification are not specified
- Nothing covers a "checking..." state, disabling the button to prevent double submits, or a request timeout.
- Cluster calls can be slow, and a hung request with no feedback looks like a dead button.
- Add to the flow in `architecture.md:26-35`: the button shows a spinner and is disabled during the request, with a client timeout (for example 10s) that renders as an error state per B1.

### S4. The diagram contract is thin
- `schema.md:9` says `id` is "used as ... the SVG node key". That is the only link between steps and the diagram.
- Unspecified: what happens when a step has no SVG node (steps that are purely conceptual), when a node has no step, which visual states exist (pending, active, done, and error or stale), and whether `manual` steps also colour their node.
- Say that `manual` steps do update the SVG node on click. State that a missing node is silently skipped or is a lint error.

### S5. How the guide gets the lab content and how the verifier finds the step
- `architecture.md:29` says the verifier "looks up the step", but not from where. Which YAML file, how it knows the lab, and whether the guide embeds the steps or fetches them are all unstated.
- The verifier is called with only `<step-id>`, so ids collide across labs unless the route includes the lab, such as `/api/verify/<lab-id>/<step-id>`.
- For the learner, a mismatch between the guide's copy of a step and the verifier's copy would show up as unexplained failures. State that there is a single source of truth, the lab YAML, and who loads it.

## Minor / nice to have

- `schema.md:11` instructions are plain markdown. Recommend a convention of one short goal sentence, then commands, then an expected-result line, to keep steps unambiguous. The example (`schema.md:40-47`) already follows this except for the expected-result line.
- Example image `quay.io/example/hello` (`schema.md:44`) is a placeholder. A learner copying it gets an `ImagePullBackOff` and a confusing failure. Use a real, pullable image or flag it as an example.
- Idempotency is a strength worth stating: every check is a state-based read-only query (`architecture.md:44`), so re-clicking "I did it" is safe. Say so; it supports the retry flow.
- The verifier "exits with an error" when no credentials are set (`architecture.md:57`). That is fine for a developer, but the learner then hits the fetch-throws case from B1. Cross-reference it from there.
- Accessibility of state: the diagram and cards signal state, but the doc does not say that state must not rely on colour alone. Add a one-line requirement (icon or text label alongside colour).

## What works well
- Click-to-complete flow with immediate feedback and a hint on failure (`architecture.md:26-35`).
- Manual steps never fail on a network call (`architecture.md:27`, `schema.md:25`).
- Explicit credential sources and no silent fallback to `~/.kube/config` (`architecture.md:50`). This prevents a confusing wrong-cluster result.
- `localhost`-only binding and a fixed CORS origin make the setup predictable (`architecture.md:45-46`).
- Schema validation rejects malformed steps early (`schema.md:16`), so authoring errors are caught before a learner sees them.

## Summary
The documents meet the acceptance criteria in `spec.md`, but they do not yet support the learner experience beyond the happy path. Required for PASS: B1 (distinct error state for cluster or verifier problems), B2 (ordering and prerequisites), and B3 (reset and re-verification on load). S1-S5 are recommended in the same pass because they change the same two files.
