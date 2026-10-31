VERDICT: CHANGES

Reviewed `git diff main...factory/001-lab-architecture` (commits 767ec6f, 92a9643; files: `docs/architecture.md`, `labs/schema.md`, `build.md`).

## Must fix

1. `labs/schema.md:37-60` - Broken markdown fence. The `yaml` example (opened at line 37) embeds a plain triple-backtick block inside the `instructions: |` scalar (lines 43-45). In CommonMark the inner "```" at line 43 closes the outer fence, so the example renders truncated and the rest (lines 46-60, including the `deployment_ready` `expected` map and the `manual` step) spills out as plain text. This undermines AC6 ("complete YAML example"). Fix: open the outer fence with four backticks (`` ```` ``) or four tildes, or drop the inner fence and use an indented command. Note: build.md's "YAML parses" claim only holds when the block is extracted by line range, not by fence. A fence-based extractor parses just lines 38-42.

## Acceptance criteria

1. PASS - `docs/architecture.md` exists and is committed.
2. PASS - Stack at `architecture.md:7-11` (static HTML via `python -m http.server`, FastAPI). Rejected alternatives at `:111-120` (Flask, Node, `oc` shell-out, SPA, direct cluster access, `~/.kube/config`).
3. PASS - Flow at `:24-38`: `POST http://localhost:<VERIFIER_PORT>/api/verify/<step-id>`, `{pass, hint}`, step card and SVG node update.
4. PASS - `labs/schema.md` exists and is committed.
5. PASS - `schema.md:7-14` has `id`, `title`, `instructions`, `check_type`, `expected` (plus optional `hint`), each with type, description, and example. Enum values are at `:20-25`.
6. CHANGES - Content is a complete non-manual example (`deploy-hello`, `deployment_ready`), but see Must fix #1: it renders broken.
7. PASS - YAML at `architecture.md:65-107`: 4 documents parse with PyYAML (Namespace, ServiceAccount, ClusterRole, ClusterRoleBinding). The SA is in the dedicated `lab-verifier` namespace. Every rule has verbs exactly `["get","list"]`. `watch`, `create`, `patch`, `update`, and `delete` appear only in the exclusion sentence at `:63`. There are no wildcards and no secrets access.
8. PASS - Recounted excluding fenced blocks and table rows: `architecture.md` 353 words, `schema.md` 76 words (both ≤ 600). The recount is not affected by the fence bug, and either way is far under the limit.

## Consistency checks

- Check types: `architecture.md:42` (`project_exists`, `resource_exists`, `deployment_ready`, `manual`) matches `schema.md:20-25` exactly.
- ClusterRole coverage: `project_exists` -> `projects` (`:82-84`); `deployment_ready` -> `deployments` (`:88-90`); `resource_exists` kinds -> projects, namespaces, pods, services, configmaps, deployments, routes (`:82-93`). Every check type is covered, and `schema.md:27` ties the `kind` allow-list to the ClusterRole. `manual` needs no RBAC.
- Ports: 8080 (`GUIDE_PORT`) and 8081 (`VERIFIER_PORT`) are used consistently in `architecture.md:19-20`, `:28`, and `:45`. `schema.md` does not reference ports, so there is no conflict.
- OpenShift API groups: `projects` -> `project.openshift.io` (`:82`); `routes` -> `route.openshift.io` (`:91`); `deployments` -> `apps` (`:88`); core kinds -> `""` (`:85`). All correct.
- ServiceAccount, ClusterRoleBinding `roleRef` and `subjects` names and namespaces match.

## Minor (non-blocking)

- `architecture.md:22`: how the verifier port is injected into the guide at serve time is vague. build.md lists it as a known gap, and it is acceptable because `mise` tasks are out of scope.
- `architecture.md:42` omits the `resource_exists` kind list, which lives only in the ClusterRole (`:86`, `:89`, `:92`). This is acceptable because `schema.md:27` points to the ClusterRole.
- `docs/` and `labs/` are new directories with no existing conventions or tests to follow. No mise task or test applies to this job.
