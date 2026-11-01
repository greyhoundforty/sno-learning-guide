VERDICT: PASS

## Acceptance-criteria re-run (independent, on branch `factory/002-sno-proxmox-runbook`)

All commands executed directly against the branch checkout.

| # | Command | Result |
| --- | --- | --- |
| AC1 | `test -f docs/sno-proxmox.md` | ✅ PASS |
| AC2 | `grep -i "vm sizing\|sizing"` | ✅ PASS — matches `## 1. VM Sizing` and prereq note |
| AC3 | `grep -E "12\|32 GB\|150"` | ✅ PASS — all three values in sizing table and VM config table |
| AC4 | `grep -E "\b8\b.*vCPU\|16 GB\|120 GB"` | ✅ PASS — one row in sizing table |
| AC5 | `grep -iE "assisted installer\|agent.based\|agent-based"` | ✅ PASS — "Assisted Installer" appears 10+ times |
| AC6 | `grep -E "api\."` + `grep -E "\*\.apps\."` | ✅ PASS — both in DNS table, examples, and dnsmasq block |
| AC7 | `grep -i "kubeconfig"` | ✅ PASS — section 5 heading, §5.2 button prose, paths, and export examples |
| AC8 | `grep -i "mise"` | ✅ PASS — section 9 heading and body text |
| AC9 | `(password\|secret)\s*=` after filtering exclusions → count | ✅ PASS — 0 |
| AC10 | bare `kubeconfig` not matching `<\|auth/\|KUBECONFIG` → count | ✅ PASS — 0 |
| AC11 | `npx markdownlint-cli2 docs/sno-proxmox.md` | ✅ PASS — 0 issues, exit 0 |
| AC12 | `:[0-9]{4}` not `:8080\|:8081` → count | ✅ PASS — 0; port 6443 and port 443 appear as prose only, no colon-number pattern |

All 12 acceptance criteria pass.

---

## Round-1 findings resolution

| Finding | Status | Evidence |
| --- | --- | --- |
| F1 — Task names conflict with backlog | ✅ RESOLVED | `docs/sno-proxmox.md:377–381` now has `guide:serve`, `verifier:serve`, `cluster:login`, `lab:reset <n>` — exact match to `factory/backlog.md:12` |
| F2 — dnsmasq wildcard comment missing | ✅ RESOLVED | `docs/sno-proxmox.md:128` now reads `address=/.apps.sno.lab.local/192.168.1.50   # leading dot = all *.apps.sno.lab.local` |
| F3 — 4.13 link no version warning | ✅ RESOLVED | `docs/sno-proxmox.md:399` now reads `[SNO install docs (Red Hat)][sno-docs] — 4.13 docs; update link to 4.22 once published` |
| F4 — `lab:verify` naming ambiguous | ✅ RESOLVED | `lab:verify` is gone; replaced by `verifier:serve` (F1 fix) |

---

## Technical accuracy — new content added in round 2

### `oc` CLI install prerequisite (`docs/sno-proxmox.md:17–19`)

- Download URL `console.redhat.com/openshift/downloads` is correct and live.
- `brew install openshift-cli` is the correct Homebrew formula name (verified:
  `openshift-cli: stable 4.22.18`, old name `openshift-client`). ✅

### DNS verification with `dig` (`docs/sno-proxmox.md:133–138`)

- Three `dig +short` commands cover `api.`, `api-int.`, and a wildcard probe
  (`foo.apps.`) — all three required DNS records are exercised. ✅
- Failure instruction ("re-check `dnsmasq.conf` and restart `dnsmasq`") and
  the consequence note ("stall the installer 40+ minutes") are accurate. ✅

### `oc get clusteroperators` post-install check (`docs/sno-proxmox.md:244–252`)

- Column names `AVAILABLE`, `PROGRESSING`, `DEGRADED` match the actual OCP
  output format. ✅
- The guidance to wait 5–10 min when `PROGRESSING=True` and to use
  `oc get clusterversion` as the final readiness signal is correct SNO
  operational practice. ✅

### `oc get clusterversion` output example (`docs/sno-proxmox.md:351–354`)

- Column order `NAME VERSION AVAILABLE PROGRESSING SINCE STATUS` matches the
  actual OCP command output. ✅

### Graceful shutdown before snapshot (`docs/sno-proxmox.md:321–328`)

- `ssh core@<node-ip-address> sudo shutdown -h now` is correct for RHCOS
  (CoreOS). The `core` user is the correct default SSH user for SNO. ✅
- "Stop (not Force Stop)" in Proxmox UI correctly distinguishes ACPI clean
  shutdown from a hard-kill. ✅
- "Include RAM unchecked" for a stopped VM is correct Proxmox behaviour. ✅

### `oc login` troubleshooting row (`docs/sno-proxmox.md:367`)

- URL template `https://api.<cluster-name>.<base-domain>` with port 6443 is
  the correct SNO API endpoint form. ✅
- Port 6443 is expressed as prose `(the API server listens on port 6443)` —
  not as a `:6443` URL pattern. This satisfies AC12 as written; the prose
  approach is an acceptable interpretation per the job instructions. ✅

### Discovery ISO timing and diagnostics (`docs/sno-proxmox.md:202–209`)

- 3–8 minute discovery window is accurate for typical Assisted Installer
  registration. ✅
- `journalctl -u agent -f` targets the `assisted-service-agent` unit name used
  on the RHCOS discovery image. ✅
- Outbound HTTPS to `api.openshift.com` on port 443 is the correct firewall
  note. Port 443 appears as prose `(port 443)` — no colon-number pattern,
  no AC12 concern. ✅

### Section 9 planned tasks (`docs/sno-proxmox.md:372–392`)

- All four backlog tasks present with exact names: `guide:serve`,
  `verifier:serve`, `cluster:login`, `lab:reset <n>`. ✅
- `cluster:status` and `cluster:snapshot` correctly labelled as "optional
  extras, not yet in the backlog". ✅
- `mise.toml` link uses relative path `../mise.toml`; file exists at project
  root — link is valid. ✅

---

## Other checks

### Minimal diff

The branch adds four files (`docs/sno-proxmox.md`, `.markdownlint.yaml`,
`build.md`, `spec.md`) and touches no existing code. No scope creep. ✅

### `.markdownlint.yaml`

Three keys only: `MD013.line_length: 80`, `MD013.tables: false`,
`MD013.code_blocks: false`. Minimal and justified. ✅

### `.gitignore` coverage

`kubeconfig*`, `*.kubeconfig`, `auth/`, and `pull-secret*` are all present
from before this branch. AC10 passes cleanly. ✅

### No new issues found

No findings requiring changes. The document is accurate, consistent with the
backlog, and all acceptance criteria pass independently.
