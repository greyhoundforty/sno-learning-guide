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

Port 8000 is taken by the factory dashboard, so it is not used. The `mise` tasks export both variables and the guide reads the verifier port from a config value injected at serve time.

## Guide to verifier flow

1. The learner finishes a step and clicks **I did it**.
2. If the step has `check_type: manual`, the guide marks it done client-side. It does not call the API.
3. Otherwise the guide sends `POST http://localhost:<VERIFIER_PORT>/api/verify/<step-id>`.
4. The verifier looks up the step, validates its `expected` map against the schema, runs the matching check, and returns:

```json
{ "pass": true, "hint": "" }
```

5. On `pass: true` the step card turns complete and the matching SVG node changes state. On `pass: false` the card stays open and shows `hint`.
6. Progress (completed step ids) is persisted in the browser's `localStorage`, so a reload keeps state.

This matches the click-to-complete pattern of the reference guide.

## Verifier rules

- It runs only allow-listed check types: `project_exists`, `resource_exists`, `deployment_ready`, `manual` (client-side only).
- Parameters come from the step YAML and are validated against the schema. The API never accepts arbitrary `oc` or shell strings, and never shells out.
- Every check is a read-only `get` or `list` call.
- CORS: the verifier sends `Access-Control-Allow-Origin` only for `http://localhost:${GUIDE_PORT}`.
- It binds to `127.0.0.1` only.

## Cluster access

The verifier needs an explicit credential source. It never falls back to `~/.kube/config`.

| Option | Variables |
| --- | --- |
| Kubeconfig | `KUBECONFIG=<path>` |
| Token | `SA_TOKEN=<token>` and `API_SERVER_URL=<url>` |

Store credential files under `auth/` or as `kubeconfig*`; both are gitignored. If neither option is set, the verifier exits with an error.

A future multi-node IBM Cloud cluster is targeted by pointing the verifier at a different `KUBECONFIG`. No code or guide change is needed.

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

A new resource kind for `resource_exists` requires adding it to this role in the same change.

## Alternatives rejected

| Alternative | Reason rejected |
| --- | --- |
| Flask | Works, but FastAPI gives request validation and OpenAPI for free, which fits schema-validated parameters |
| Node/Express verifier | Adds a second runtime; `mise` already manages Python |
| Shelling out to `oc` | Opens command injection and needs `oc` installed; the Python client is safer |
| Guide as a framework SPA | Needs a build step; a single static file is easier to share and review |
| Guide reads the cluster directly | The browser cannot hold cluster credentials safely |
| Reading `~/.kube/config` by default | Can target the wrong cluster silently |
