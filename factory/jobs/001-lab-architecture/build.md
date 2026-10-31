# Build: 001-lab-architecture

Branch: `factory/001-lab-architecture`. Commit `767ec6f`.

## What changed

- Added `docs/architecture.md`: stack decision, ports, guide-to-verifier flow, verifier rules, cluster access, RBAC manifests, rejected alternatives.
- Added `labs/schema.md`: step fields, check types, response shape, YAML example.

Open-question decisions applied:

1. `VERIFIER_PORT` default 8081, `GUIDE_PORT` default 8080 (8000 is the factory dashboard).
2. CORS allows only `http://localhost:${GUIDE_PORT}`; guide is always served via a `mise` task, never `file://`.
3. Verifier needs explicit `KUBECONFIG` or `SA_TOKEN` + `API_SERVER_URL`; never defaults to `~/.kube/config`; credential files are gitignored (existing `.gitignore` covers `kubeconfig*`, `auth/`).
4. `manual` steps resolve client-side; no API call.

Also documented: allow-listed check types with schema-validated parameters (no arbitrary `oc`/shell strings), `localStorage` progress, multi-node IBM Cloud via a different `KUBECONFIG`, FastAPI + `kubernetes` client through `mise`/`uv`, and rejected alternatives.

## Verification of acceptance criteria

1. Both docs committed cleanly; pre-commit scanner reported no secrets.
2. Stack and rejected alternatives are in the "Decision" and "Alternatives rejected" sections.
3. "Guide to verifier flow" describes `POST /api/verify/<step-id>` to localhost and the `{pass, hint}` response updating the step card and SVG node.
4. `labs/schema.md` committed.
5. Fields table lists `id`, `title`, `instructions`, `check_type`, `expected` (plus optional `hint`), each with description and example; `check_type` enum is listed.
6. The example `deploy-hello` step uses `deployment_ready`.
7. The RBAC section defines Namespace/ServiceAccount `lab-verifier`, a ClusterRole with only `get`/`list`, and a ClusterRoleBinding. Grep confirms `watch`, `create`, and `delete` appear only in prose stating they are excluded. All YAML blocks parse with PyYAML.
8. Word count script (excluding fenced code blocks and table rows): `docs/architecture.md` 353, `labs/schema.md` 76; both under 600.

## Known gaps

- The mechanism for injecting the verifier port into the guide at serve time is described only at a high level; the `mise` tasks are out of scope for this job.
- The `resource_exists` kind allow-list is defined by the ClusterRole (projects, namespaces, pods, services, configmaps, deployments, routes); later labs must extend the role in the same change.
- `mise` task names were not specified (out of scope).

## Round 1

Commit `daa5a91` (docs only: `docs/architecture.md`, `labs/schema.md`).

Fixes per review item:

- **Code B1 / UX #1:** Added a distinct `error` outcome (`status: pass|fail|error`, `error_kind`) with an Errors table (verifier unreachable, cluster unreachable, 401, 403, unknown step, invalid `expected`) giving HTTP code and guide message for each. Added `GET /api/health`. Added a step-state table (`locked`, `pending`, `checking`, `pass`, `fail`, `error`; persisted: `pass`) including the manual states (Undo) and hint order.
- **Code B2 / UX #7:** Added a lab wrapper (`id`, `title`, `steps`), list-order rule, and optional `requires` (default previous step; must reference earlier steps). Route is now `POST /api/verify/<lab-id>/<step-id>`.
- **Code B3 / S1 / UX #8:** Added a "Progress and reset" section: `localStorage` key `lab:<lab-id>:progress` with a version field, **Reset lab**, re-verify on load with demotion and dependent locking, and an optional `cleanup` field (text only).
- **Code S2 / S3:** Hint precedence stated; `checking` state disables the button, 10 s timeout maps to `verifier_unreachable`.
- **Code S4 / UX #2:** Optional `diagram_node` (default `id`), `data-step` attribute convention, shared-node roll-up, and fallback when a node is missing.
- **Code S5:** Lab YAML is the single source of truth, loaded by the verifier and embedded by the serve task.
- **UX #3, #4, #5:** Accessibility and layout rules added (not colour alone, real button, `aria-live`, SVG `<title>`, AA contrast, `viewBox`).
- **Security F1:** Guide server now bound with `python -m http.server --bind 127.0.0.1`; verifier too. CORS and links use `127.0.0.1`.
- **Security F2:** SA tokens must use TokenRequest (`oc create token ... --duration=8h`); legacy Secret tokens prohibited.
- **Security F3:** Enumerated `resource_exists` kinds (`Project, Namespace, Pod, Service, ConfigMap, Deployment, Route`) in `labs/schema.md`, rejected before any cluster call.
- **Code Must fix (fence) / Code minor:** Example in `labs/schema.md` now uses a four-backtick outer fence and is a full lab with two non-manual steps and a manual one. Image changed to `quay.io/openshifttest/hello-openshift:openshift` (manifest confirmed pullable). Port injection wording tightened.

Re-verification:

- Extracted the YAML example by fence with markdown-it (not by line range): the inner triple-backtick block stays inside `instructions`, and PyYAML parses the lab (3 steps). The `architecture.md` YAML still parses as 4 documents.
- RBAC unchanged: only `get`/`list`; `watch`/`create`/`patch`/`update`/`delete` appear only in the exclusion sentence (and `oc create token` is a command, not an RBAC verb). The health check lists one namespace, already allowed.
- Word counts excluding code blocks and table rows: `architecture.md` 475, `schema.md` 152; both ≤ 600.
- Commit scanner reported no secrets.

Not done (deliberate): per-step multiple hints and an optional success message (S2) were left out to stay under the word limit; the `cleanup` and verifier hint text cover the retry case.
