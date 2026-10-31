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
