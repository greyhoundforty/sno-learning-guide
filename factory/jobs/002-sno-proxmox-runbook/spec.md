# Spec — 002-sno-proxmox-runbook

## Goal

Produce `docs/sno-proxmox.md`: a step-by-step runbook for installing Single Node OpenShift (SNO) on Proxmox, targeting OpenShift 4.22 / EX280 labs. The runbook covers VM sizing, the Assisted Installer / agent-based ISO flow, DNS requirements, and post-install kubeconfig handling. Also document a planned `mise` task checklist (description only; implementation is a separate backlog item).

## Scope

**In scope:**
- `docs/sno-proxmox.md` covering: VM sizing table, Proxmox VM configuration, DNS setup (`api.<cluster>.<domain>` and `*.apps.<cluster>.<domain>`), Assisted Installer / agent-based ISO walkthrough, post-install `kubeconfig` retrieval, and a documented mise task checklist section.
- Placeholder conventions for pull secrets and credentials (`<pull-secret>`, `<cluster-name>`, `<base-domain>`); no literal secrets or secret-like strings.
- Sizing recommendations: recommended 12 vCPU / 32 GB RAM / 150–200 GB SSD; minimum 8 vCPU / 16 GB RAM / 120 GB per Red Hat docs.

**Out of scope:**
- Implementing the `mise` tasks (future backlog item).
- EX280 lab content or verifier integration.
- Multi-node or cloud-provider OpenShift installs.

## Acceptance criteria

1. `docs/sno-proxmox.md` exists: `test -f docs/sno-proxmox.md`
2. Document contains a VM sizing section: `grep -i "vm sizing\|sizing" docs/sno-proxmox.md`
3. Document contains recommended values (12 vCPU, 32 GB, 150): `grep -E "12|32 GB|150" docs/sno-proxmox.md`
4. Document contains Red Hat minimum values (8 vCPU, 16 GB, 120): `grep -E "\b8\b.*vCPU|16 GB|120 GB" docs/sno-proxmox.md`
5. Document addresses Assisted Installer or agent-based ISO: `grep -iE "assisted installer|agent.based|agent-based" docs/sno-proxmox.md`
6. Document covers DNS records for both API and apps wildcard: `grep -E "api\." docs/sno-proxmox.md && grep -E "\*\.apps\." docs/sno-proxmox.md`
7. Document covers post-install kubeconfig retrieval: `grep -i "kubeconfig" docs/sno-proxmox.md`
8. Document contains a mise task checklist section: `grep -i "mise" docs/sno-proxmox.md`
9. No literal pull-secret values or secret-like assignments: `grep -vE "placeholder|example|<pull-secret>" docs/sno-proxmox.md | grep -iE "(password|secret)\s*=" | wc -l` must output `0`
10. Kubeconfig paths reference placeholders, not live paths: `grep "kubeconfig" docs/sno-proxmox.md | grep -vE "<|auth/|KUBECONFIG" | wc -l` must output `0` (all references use placeholder or gitignored path conventions)
11. File passes markdown lint with no errors: `markdownlint docs/sno-proxmox.md` exits `0` (or equivalent `mdl` / `npx markdownlint-cli2`)
12. Verifier ports documented correctly where referenced: if port numbers appear, they are 8080 (guide) and 8081 (verifier): `grep -E ":[0-9]{4}" docs/sno-proxmox.md | grep -vE ":8080|:8081" | wc -l` outputs `0`

## Open questions

1. **Pull-secret source**: should the runbook link to `console.redhat.com` for pull-secret download, or describe an offline/disconnected flow? Assuming online (default) until told otherwise.
2. **Static IP vs. DHCP reservation**: the runbook will document both options; confirm which the lab host Proxmox uses so we can recommend one primary path.
3. **Snapshot guidance**: include Proxmox snapshot step post-install? (Mentioned in `docs/initial-questions.md` as useful for lab resets — defaulting to yes.)
4. **OpenShift version pin**: Assisted Installer UI defaults to latest; should the runbook specify how to select 4.22 explicitly, or note that 4.22 may not yet be in the UI and fall back to 4.17 LTS?
