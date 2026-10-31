VERDICT: CHANGES

## Findings

### F1 — Guide server bind address not constrained to 127.0.0.1 (docs/architecture.md:8)
**Severity: medium**

`docs/architecture.md` line 8 specifies `python -m http.server` for the guide. By default that
command binds to `0.0.0.0`, exposing the guide (and its injected `VERIFIER_PORT` value) to the
local network. The document mandates `127.0.0.1` only for the verifier (line 46) but is silent on
the guide server bind address. The `mise` task invocation must pass `-b 127.0.0.1` (or equivalent),
and `docs/architecture.md` must state this explicitly alongside the verifier rule.

**Required change:** add a bullet to the "Verifier rules" section (or a new "Guide server" section)
stating the guide static server also binds to `127.0.0.1` only, e.g.
`python -m http.server --bind 127.0.0.1 <GUIDE_PORT>`.

---

### F2 — ServiceAccount token lifetime / rotation not addressed (docs/architecture.md:50-58)
**Severity: medium**

The "Cluster access" section (lines 50–58) documents two credential modes (`KUBECONFIG` and
`SA_TOKEN`) but says nothing about token lifetime, expiry, or how to rotate credentials. In
Kubernetes 1.22+ a `TokenRequest`-issued SA token has a default TTL of 1 hour; a long-lived
legacy `Secret`-backed token never expires. Using a non-expiring token is a security risk if the
credential leaks.

**Required change:** add a note specifying that the SA token should be obtained via `TokenRequest`
(or `oc create token lab-verifier -n lab-verifier --duration=…`) and state a recommended maximum
lifetime (e.g. 8 hours for a lab session). Warn that legacy non-expiring `Secret`-backed SA tokens
should not be used.

---

### F3 — `resource_exists.kind` allow-list is implicit, not enumerated (labs/schema.md:23-27 / docs/architecture.md:81-93)
**Severity: low**

`labs/schema.md` line 27 states "`resource_exists.kind` must be one of the kinds granted in the
verifier ClusterRole" without listing them. An implementer reading only the schema has no explicit
enumeration to validate against, and could accept arbitrary kind strings without checking them
against a hardcoded list. The ClusterRole grants six groups of resources
(`projects`, `namespaces`, `pods`, `services`, `configmaps`, `deployments`, `routes`); these should
be the literal allow-list enforced by the verifier.

**Required change:** in `labs/schema.md` (or `docs/architecture.md` under "Verifier rules") add an
explicit enumeration of the allowed `kind` values for `resource_exists`, e.g.
`Project, Namespace, Pod, Service, ConfigMap, Deployment, Route`. The verifier must reject any
kind not on this list before making a cluster API call.

---

## Passing checks

- **ClusterRole scope justified** — `projects` (openshift) and `routes` are cluster-scoped
  resources; `namespaces` is cluster-scoped; a `Role` cannot grant access to these kinds, so
  `ClusterRole`/`ClusterRoleBinding` is the correct choice. (docs/architecture.md:77–107)

- **No credential-bearing resources in ClusterRole** — `secrets`, `serviceaccounts/token`,
  `pods/exec`, `pods/log`, `nodes`, and `events` are absent from all rules. (docs/architecture.md:82–93)

- **`configmaps` at cluster scope — acceptable** — granted at cluster scope to support
  `resource_exists` checks across any namespace. Sensitivity depends on what labs create; the
  allow-list constraint (F3 above, once fixed) limits the attack surface to kinds explicitly
  enumerated by the architecture.

- **No `watch`/`create`/`patch`/`update`/`delete` verbs** — confirmed absent from the ClusterRole
  and called out in prose. (docs/architecture.md:63, 82–93)

- **Injection: no shell-out, allow-listed check types** — verifier rules explicitly prohibit
  arbitrary `oc` or shell strings; parameters are schema-validated; `step-id` is a slug
  (lowercase letters, digits, hyphens). (docs/architecture.md:42–43, labs/schema.md:9)

- **CORS restricted to known origin** — `Access-Control-Allow-Origin` is set only for
  `http://localhost:${GUIDE_PORT}`, not `*`. (docs/architecture.md:45)

- **Verifier binds to 127.0.0.1** — explicitly stated. (docs/architecture.md:46)

- **No `~/.kube/config` fallback** — explicitly prohibited; verifier exits on missing credentials.
  (docs/architecture.md:50, 57)

- **Credential files gitignored** — `.gitignore` covers `kubeconfig*`, `*.kubeconfig`, `auth/`,
  `pull-secret*`, `.env`, `.env.*`; `SA_TOKEN` is an env var (not a file), so no gap.

- **No secrets committed** — diff contains no tokens, kubeconfigs, or pull secrets.
