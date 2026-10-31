# Factory backlog

Run the first unchecked item with `/factory next`.

- [x] Lab architecture decision: choose guide + verifier stack (static HTML + small Python API vs. alternatives), define the step schema (id, title, instructions, check type, expected result) and the verifier's read-only ServiceAccount/RBAC. Output: `docs/architecture.md` and `labs/schema.md`.
- [ ] SNO on Proxmox runbook: VM sizing, Assisted Installer / agent-based ISO flow, networking/DNS (`api.`, `*.apps.`), post-install `kubeconfig` handling. Output: `docs/sno-proxmox.md` plus a `mise` task checklist.
- [ ] Guide shell: single-file HTML guide with step cards, "I did it" button, persistent progress (localStorage), and an SVG diagram that updates as steps complete. Verification is stubbed (always pass).
- [ ] Verifier service: small API with allow-listed checks (e.g. `project_exists`, `resource_exists`, `deployment_ready`) that query the cluster using a read-only token; `POST /api/verify/<step-id>` returns pass/fail plus a hint. Wire the guide's "I did it" to it.
- [ ] Lab 01 - Projects and access: create a project, set quotas/limit ranges, grant a user a role. Includes step definitions and checks.
- [ ] Lab 02 - Authentication and RBAC: HTPasswd identity provider, users/groups, roles and bindings, remove kubeadmin.
- [ ] Study map: map EX280 objectives to labs, track coverage gaps in `docs/ex280-coverage.md`.
- [ ] `mise` tasks: `guide:serve`, `verifier:serve`, `cluster:login`, `lab:reset <n>`.
- [ ] Multi-node labs (last): topics SNO cannot cover (node maintenance/drain, scheduling across nodes, machine configs/MachineConfigPools, HA behavior). Run on an IBM Cloud multi-node OpenShift cluster; document how to point the verifier at that cluster.
