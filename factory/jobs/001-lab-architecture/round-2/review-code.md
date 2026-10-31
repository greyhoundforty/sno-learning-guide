VERDICT: CHANGES

Reviewed `git diff main...factory/001-lab-architecture` (docs/architecture.md, labs/schema.md, build.md; HEAD `00a1a81`), spec.md, build.md including Round 1, and my round-1 review. Round 1 must-fix (broken fence) is resolved. One small new defect remains, in the Example column of the schema field table.

## Must fix

1. `labs/schema.md:19` and `labs/schema.md:25` - The Example cells show broken inline code. They contain `` `Run \`oc new-project demo\`.` ``. Backslash escapes do not work inside a CommonMark code span, so the span ends at the first inner backtick. markdown-it renders line 19 as `<code>Run \</code>oc new-project demo`.` and line 25 the same way with `oc delete project demo`. Stray backticks and a trailing backslash show up in the table. AC5 requires each field to have a readable example value. Fix: use a double-backtick span, e.g. ``` ``Run `oc new-project demo`.`` ```, or write the example without nested backticks. The YAML example at `:77` and `:81` is unaffected.

## Acceptance criteria (independently re-verified)

1. PASS - `docs/architecture.md` is present in the diff. `git diff --check` is clean and the file ends with a newline.
2. PASS - Stack is at `architecture.md:7-13`: static HTML plus `python -m http.server`, FastAPI, and `fetch()`. Rejected alternatives are at `:155-164`: Flask, Node, shell-out to `oc`, SPA, direct cluster access, and `~/.kube/config`.
3. PASS with a note - Flow is at `:24-32`. The route is now `POST http://127.0.0.1:<VERIFIER_PORT>/api/verify/<lab-id>/<step-id>`. The response is `{status: pass|fail|error, hint}` (`schema.md:55-67`). The spec text says `POST /api/verify/<step-id>`, `localhost`, and `{pass: bool, hint}`. Each difference came from a round-1 review item (lab-scoped route, loopback binding, a distinct error outcome). Confirm the orchestrator accepts these deviations from the spec's literal wording. I do not block on them.
4. PASS - `labs/schema.md` is present in the diff.
5. PASS, apart from the example cells in Must fix #1 - Step fields at `schema.md:15-25` include `id`, `title`, `instructions`, `check_type`, `expected`, plus `requires`, `diagram_node`, `hint`, `cleanup`. Each has a type, description, and example. The `check_type` enum is at `:34-39`.
6. PASS - I extracted the fence with markdown-it. The four-backtick `yaml` fence spans `schema.md:70-107`, and the inner triple-backtick block stays inside the `instructions:` scalar. PyYAML parses it as one lab with 3 steps: `create-project` (`project_exists`), `deploy-hello` (`deployment_ready`, requires `create-project`), and `compare-notes` (`manual`). Every `requires` id points to an earlier step.
7. PASS - The `yaml` fence at `architecture.md:108-151` parses as 4 documents: Namespace, ServiceAccount, ClusterRole, ClusterRoleBinding. Every rule has verbs exactly `[get, list]`. There are no wildcards and no `secrets`. `watch`, `create`, `patch`, `update`, and `delete` appear only in the exclusion sentence at `:107` and in the `oc create token` command at `:98`, which is not an RBAC verb. The ServiceAccount sits in the dedicated `lab-verifier` namespace, and the binding's roleRef and subjects match.
8. PASS - Re-counted with a markdown-it token walk that excludes fenced blocks and table blocks. `architecture.md` has 475 words and `schema.md` has 152, matching build.md. Both are ≤ 600. Counting table text as well gives 1085 and 569, but the spec excludes tables.

## ClusterRole coverage

| Check type or kind | Needs | Role rule | Result |
| --- | --- | --- | --- |
| `project_exists`, kind `Project` | `project.openshift.io/projects` | `architecture.md:126-128` | covered |
| kind `Namespace`, `Pod`, `Service`, `ConfigMap` | core `namespaces`, `pods`, `services`, `configmaps` | `:129-131` | covered |
| `deployment_ready`, kind `Deployment` | `apps/deployments` | `:132-134` | covered |
| kind `Route` | `route.openshift.io/routes` | `:135-137` | covered |
| `manual` | none | none | n/a |
| `GET /api/health` | list one namespace | `:130` | covered |

The seven kinds in the allow-list (`schema.md:45-53`) map one to one onto the role resources. API groups are correct, and `architecture.md:153` ties allow-list changes to role changes.

## Doc consistency

- Route: `<lab-id>/<step-id>` is used in `architecture.md:28` and `schema.md:17`, and `lab:<lab-id>:progress` at `:62` matches `schema.md:9`.
- Ports: 8080 and 8081 appear only in `architecture.md:19-20`. The loopback address `127.0.0.1` is used at `:28`, `:51`, `:83`, and `:84`. `architecture.md:11` still says "localhost" in prose, which is harmless.
- States: `locked`, `pending`, `checking`, `pass`, `fail`, `error` at `architecture.md:38-43`. The response statuses `pass`, `fail`, `error` at `schema.md:57` are consistent with them. `schema.md` does not redefine the states.
- Field names: `requires`, `diagram_node`, `hint`, and `cleanup` match across the documents (`architecture.md:38`, `:45`, `:62`, `:68`; `schema.md:22-25`). `error_kind` values `invalid_expected` and `cluster_unreachable` in `schema.md:43`, `:64` match `architecture.md:51-56`. Hint order at `architecture.md:45` is consistent with the optional step `hint` at `schema.md:24`.
- `check_type` names match across `architecture.md:79` and `schema.md:34-39`. A `manual` step makes no API call, which is stated in both documents.
- Tables: every row has the same column count as its header (checked with markdown-it).

## Minor (non-blocking)

- `architecture.md:58` - Health returns `cluster` as `ok | cluster_unreachable | auth`, but its namespace list could also return 403, which the Errors table maps to `forbidden` (`:54`). Add `forbidden`, or state that health only reports these three.
- `architecture.md:22` - Port injection into the guide remains high level. This is acceptable because `mise` tasks are out of scope.
