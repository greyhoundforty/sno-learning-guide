# Spec: 001-lab-architecture

## Goal

Decide the guide + verifier stack, define the canonical step schema, and specify
the read-only RBAC the verifier needs. Produce two reference documents that every
subsequent lab job can follow without re-making these decisions.

## Scope

- **In:** architecture decision (guide renderer, verifier runtime, IPC between
  them), step schema (fields and types), verifier ServiceAccount / ClusterRole /
  ClusterRoleBinding definition.
- **Out of scope:** implementation of the guide HTML, implementation of the
  verifier service, lab content, `mise` task definitions (all later backlog items).

## Acceptance criteria

1. `docs/architecture.md` exists and commits cleanly to the repo.
2. `docs/architecture.md` records the chosen stack: static single-file HTML guide
   served locally (e.g. `python -m http.server` via a `mise` task) **and** a small
   Python HTTP API (FastAPI or Flask) as the verifier; alternative stacks
   considered and reasons for rejection are noted.
3. The document explains how the guide calls the verifier: the "I did it" button
   sends `POST /api/verify/<step-id>` to `localhost:<port>`; the response
   (`{pass: bool, hint: string}`) updates the step card and the SVG diagram — matching
   the click-to-complete pattern at the reference guide URL.
4. `labs/schema.md` exists and commits cleanly to the repo.
5. `labs/schema.md` defines the step schema with at minimum these fields:
   `id` (string, slug), `title` (string), `instructions` (markdown string),
   `check_type` (enum — e.g. `project_exists`, `resource_exists`,
   `deployment_ready`, `manual`), and `expected` (type-specific value map).
   Each field has a description and example value.
6. `labs/schema.md` includes a complete YAML example of at least one step that
   uses a non-manual `check_type`.
7. `docs/architecture.md` specifies the verifier's OpenShift RBAC: a
   `ServiceAccount` in a dedicated namespace (e.g. `lab-verifier`), a
   `ClusterRole` granting `get`/`list` on the resource kinds needed by the
   allowed check types, and a `ClusterRoleBinding` — all read-only, no `watch`,
   no `create`/`delete`/`patch`.
8. Both documents are under one printed page each (≤ 600 words / file, excluding
   code blocks and tables).

## Open questions

1. **Port conflict policy** — the verifier defaults to `localhost:8000`; should
   `docs/architecture.md` mandate a fixed port or let `mise` expose it as an env
   var (`VERIFIER_PORT`)?
2. **CORS** — the guide is a `file://` or `localhost` origin; does the verifier
   need an explicit `Access-Control-Allow-Origin` header, or will we always serve
   the guide through `mise guide:serve` on a known port?
3. **Token storage** — the verifier reads the SNO cluster via a ServiceAccount
   token; should it read `~/.kube/config` (dev convenience) or require an explicit
   `KUBECONFIG` / `SA_TOKEN` env var (more explicit, safer for sharing)?
4. **`manual` check fallback** — for steps the verifier cannot automate (e.g.
   "compare the output with your notes"), the schema allows `check_type: manual`.
   Should `POST /api/verify/<step-id>` for a manual step always return `pass: true`,
   or should the guide skip the API call entirely and resolve client-side?
