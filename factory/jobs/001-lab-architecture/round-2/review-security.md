VERDICT: PASS

## Round 1 findings

- **F1 (guide bind address): resolved.** `docs/architecture.md:84` now requires both servers to bind to `127.0.0.1`. It gives the guide command as `python -m http.server --bind 127.0.0.1 "$GUIDE_PORT"`. The flow (line 28), the CORS rule (line 83) and the error text (line 51) all use `127.0.0.1`.
- **F2 (SA token lifetime): resolved.** `docs/architecture.md:95-101` requires TokenRequest tokens (`oc create token lab-verifier -n lab-verifier --duration=8h`) with a maximum of 8 hours. It forbids legacy `kubernetes.io/service-account-token` Secrets.
- **F3 (`resource_exists.kind` allow-list): resolved.** `labs/schema.md:41-53` lists the seven allowed kinds (`Project, Namespace, Pod, Service, ConfigMap, Deployment, Route`) as case-sensitive. Anything else is rejected with `invalid_expected` before any cluster call. `docs/architecture.md:81` links to the list, and `:153` ties each kind to a role rule. The list matches the ClusterRole (`:126-137`).

## New surface checked

- **`GET /api/health` (`docs/architecture.md:58`): acceptable.** It returns only the enums `ok`, `cluster_unreachable` and `auth`. It uses a `list namespaces` call, which the role already grants (`:130`), so no new RBAC is needed. It is read-only and sits behind the same `127.0.0.1` bind.
- **`error_kind` and hints: acceptable.** The guide messages (`:51-56`) are fixed strings. `error_kind` is a closed enum, so no API server URL, token, TLS detail or raw exception text appears in the design. The `fail` hint in `labs/schema.md:60` only echoes author-controlled parameters.
- **Route params `<lab-id>/<step-id>` (`docs/architecture.md:28`): acceptable.** `step-id` is restricted to lowercase letters, digits and hyphens (`labs/schema.md:17`), and `lab-id` is a slug (`labs/schema.md:9`). Both are used as lookup keys into the YAML, and unknown values return 404 `unknown_step` (`:55`) with no cluster call. Cross-field validation holds: `requires` must reference earlier steps (`labs/schema.md:29`) and unknown `expected` keys are rejected (`:30`).
- **CORS (`docs/architecture.md:83`): acceptable.** The origin is restricted to `http://127.0.0.1:${GUIDE_PORT}`, with no wildcard. A learner who opens `localhost:8080` instead will hit a CORS failure. That is a usability issue, not a security one.
- **RBAC (`docs/architecture.md:105-151`): unchanged and still least-privilege.** It grants only `get` and `list`, and there is no `secrets`, `pods/exec`, `pods/log`, `nodes` or token access.
- **No secrets, credential defaults or `~/.kube/config` fallback were introduced** (`docs/architecture.md:88, 101`). The diff contains no tokens or kubeconfigs.

## Advisory notes (non-blocking, for the implementation jobs)

- **Lab lookup by id.** Load the labs into a map at startup and look up by the validated slug. Do not build a file path such as `labs/<lab-id>.yaml` from the request parameter. Enforce the slug regex on `lab-id` the same way as `step-id`.
- **Unauthenticated localhost POSTs.** Any web page the learner visits can send a request to `127.0.0.1:8081`, and DNS rebinding could read the responses. Impact is limited to read-only existence and readiness checks. The implementation can cheaply tighten this by validating the `Host` header against `127.0.0.1:${VERIFIER_PORT}`.
- **Rendering `instructions`.** The Markdown comes from repo-controlled YAML. The guide's renderer should still escape or sanitize raw HTML. Lab authors are trusted, but learners may import third-party labs.
- **Error bodies.** In the implementation, return fixed messages and log exception detail server-side only.
