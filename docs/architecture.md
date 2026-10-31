# Lab architecture

Status: accepted. Every lab job follows this document and [`labs/schema.md`](../labs/schema.md).

## Decision

| Part | Choice |
| --- | --- |
| Guide | One static HTML file with an inline SVG diagram, served by `python -m http.server` through a `mise` task |
| Verifier | Small Python HTTP API: FastAPI plus the official `kubernetes` client, run through `mise` and `uv` |
| IPC | `fetch()` from the guide to the verifier on localhost |

The guide is always served over HTTP by a `mise` task. It is never opened as `file://`.

## Ports

| Variable | Default | Purpose |
| --- | --- | --- |
| `GUIDE_PORT` | 8080 | Guide static server |
| `VERIFIER_PORT` | 8081 | Verifier API |

Port 8000 is taken by the factory dashboard, so it is not used. The `mise` tasks export both variables and inject the verifier port into the guide at serve time.

## Guide to verifier flow

1. The learner clicks **I did it**.
2. A `manual` step is marked done client-side. No API call.
3. Otherwise the guide sends `POST http://127.0.0.1:<VERIFIER_PORT>/api/verify/<lab-id>/<step-id>`.
4. The verifier loads the step from the lab YAML (the single source of truth, also embedded in the guide by the serve task), validates `expected`, runs the check, and returns the [response](../labs/schema.md#response).
5. The guide updates the step card and SVG node, then opens the next step and scrolls to it.

This matches the click-to-complete pattern of the reference guide. Checks are read-only, so retrying is safe.

### Step states

| State | Meaning | Persisted |
| --- | --- | --- |
| `locked` | A `requires` step is not `pass`. Button disabled; "Complete <title> first". | no |
| `pending` | Not yet attempted. | no |
| `checking` | Request in flight. Button disabled with `aria-busy`; stale responses ignored. | no |
| `pass` | Check passed, or manual step marked done (manual steps have **Undo**). | yes |
| `fail` | Check ran; the work is not right. Card stays open and shows the hint. | no |
| `error` | Check could not run. Not a learner failure; never changes an existing `pass`. | no |

Hint order: response `hint`, step `hint`, then "Not done yet."

### Errors

| Cause | HTTP | `error_kind` | Guide message |
| --- | --- | --- | --- |
| `fetch()` fails, CORS rejected, or 10 s timeout | none | `verifier_unreachable` | "Verifier isn't running on 127.0.0.1:<VERIFIER_PORT>. Start its `mise` task." |
| Cluster down, TLS failure | 503 | `cluster_unreachable` | "Can't reach your cluster. Is the VM running?" |
| Cluster returned 401 | 502 | `auth` | "Cluster rejected the credentials. Issue a new token." |
| Cluster returned 403 | 502 | `forbidden` | "Verifier lacks permission. Update the ClusterRole." |
| Unknown lab or step | 404 | `unknown_step` | "Step misconfigured. Tell the lab author." |
| Invalid `expected` | 422 | `invalid_expected` | Same as above. |

Errors show as a warning banner, distinct from `fail`. `GET /api/health` returns `{ "verifier": "ok", "cluster": "ok" | "cluster_unreachable" | "auth" }` with HTTP 200; it lists one namespace, already allowed. The guide calls it on load and shows cluster status.

### Progress and reset

Progress is stored in `localStorage` key `lab:<lab-id>:progress` as `{ "v": 1, "done": ["<step-id>"] }`. On load the guide re-verifies stored `pass` steps in order. A step that now fails is demoted to `fail` and its dependents become `locked`; an `error` leaves state alone. **Reset lab** clears the key and never touches the cluster; a step's `cleanup` text tells the learner how to undo it.

### Diagram and accessibility

| Rule | Detail |
| --- | --- |
| Node attribute | SVG node has `data-step="<diagram_node>"`; `diagram_node` defaults to the step `id`. |
| Shared nodes | Node is `pass` if all its steps pass, `fail` if any fails, else `pending`. `checking` and `error` never change it. |
| Missing node | Only the card updates. A node with no step is a lint warning. |
| Not colour alone | Card and node show a text or icon label ("Passed", "Failed"). |
| Button | **I did it** is a real `<button>` with a visible focus ring. |
| Announcements | An `aria-live="polite"` region announces states, hints, and errors. |
| Node names | Each node has a `<title>` with step title and state. |
| Layout | WCAG AA contrast; SVG uses `viewBox`; cards and diagram stack on narrow screens. |

## Verifier rules

- It runs only allow-listed check types: `project_exists`, `resource_exists`, `deployment_ready`, `manual` (client-side only).
- Parameters are schema-validated. No arbitrary `oc` or shell strings; it never shells out.
- `resource_exists.kind` must be in the [enumerated allow-list](../labs/schema.md#resource_exists-kinds), checked before any cluster call.
- Every check is a read-only `get` or `list`.
- CORS allows only `http://127.0.0.1:${GUIDE_PORT}`.
- **Both servers bind to `127.0.0.1` only**: the verifier, and the guide via `python -m http.server --bind 127.0.0.1 "$GUIDE_PORT"`.

## Cluster access

The verifier needs an explicit credential source. It never falls back to `~/.kube/config`.

| Option | Variables |
| --- | --- |
| Kubeconfig | `KUBECONFIG=<path>` |
| Token | `SA_TOKEN=<token>` and `API_SERVER_URL=<url>` |

Issue tokens with TokenRequest, valid at most 8 hours:

```bash
oc create token lab-verifier -n lab-verifier --duration=8h
```

Never use legacy long-lived `kubernetes.io/service-account-token` Secrets. Credential files go under `auth/` or `kubeconfig*` (gitignored). With no credentials the verifier exits.

A multi-node IBM Cloud cluster is targeted by changing `KUBECONFIG` only.

## RBAC

All rules are read-only: `get` and `list` only. There is no `watch`, `create`, `patch`, `update`, or `delete`, and no access to credential-bearing kinds.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-verifier
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lab-verifier
  namespace: lab-verifier
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lab-verifier-read
rules:
  - apiGroups: ["project.openshift.io"]
    resources: ["projects"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["namespaces", "pods", "services", "configmaps"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab-verifier-read
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: lab-verifier-read
subjects:
  - kind: ServiceAccount
    name: lab-verifier
    namespace: lab-verifier
```

Each allow-listed `resource_exists` kind maps to one rule above. A new kind needs a role rule and a schema allow-list entry in the same change.

## Alternatives rejected

| Alternative | Reason rejected |
| --- | --- |
| Flask | Works, but FastAPI gives request validation and OpenAPI for free, which fits schema-validated parameters |
| Node/Express verifier | Adds a second runtime; `mise` already manages Python |
| Shelling out to `oc` | Opens command injection and needs `oc` installed; the Python client is safer |
| Guide as a framework SPA | Needs a build step; a single static file is easier to share and review |
| Guide reads the cluster directly | The browser cannot hold cluster credentials safely |
| Reading `~/.kube/config` by default | Can target the wrong cluster silently |
