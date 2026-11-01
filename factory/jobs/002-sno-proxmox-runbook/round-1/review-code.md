VERDICT: CHANGES

## Acceptance-criteria re-run (independent, on branch `factory/002-sno-proxmox-runbook`)

| # | Command | Result |
| --- | --- | --- |
| AC1 | `test -f docs/sno-proxmox.md` | ✅ PASS |
| AC2 | `grep -i "vm sizing\|sizing"` | ✅ PASS — matches `## 1. VM Sizing` and prereq note |
| AC3 | `grep -E "12\|32 GB\|150"` | ✅ PASS — all three values in sizing table |
| AC4 | `grep -E "\b8\b.*vCPU\|16 GB\|120 GB"` | ✅ PASS — one matching row in sizing table |
| AC5 | `grep -iE "assisted installer\|agent.based\|agent-based"` | ✅ PASS — "Assisted Installer" appears 14+ times |
| AC6 | `grep -E "api\."` + `grep -E "\*\.apps\."` | ✅ PASS — both match in DNS table and examples |
| AC7 | `grep -i "kubeconfig"` | ✅ PASS — full section 5 plus task checklist |
| AC8 | `grep -i "mise"` | ✅ PASS — section 9 "Planned Mise Tasks" |
| AC9 | count of `(password\|secret)\s*=` lines not filtered by exclusions | ✅ PASS — 0 |
| AC10 | count of `kubeconfig` lines without `<`, `auth/`, or `KUBECONFIG` | ✅ PASS — 0 |
| AC11 | `npx markdownlint-cli2 docs/sno-proxmox.md` | ✅ PASS — exit 0, 0 issues |
| AC12 | count of `:[0-9]{4}` lines not `:8080` or `:8081` | ✅ PASS — 0 (port 6443 appears as prose `(port 6443)`, not a URL pattern) |

All 12 acceptance criteria pass.

---

## Findings

### F1 — REQUIRED — Planned task names conflict with the backlog (docs/sno-proxmox.md:331–332)

`docs/sno-proxmox.md` section 9 documents these planned tasks:

```
lab:serve    — guide static server on port 8080
lab:verify   — verifier API on port 8081
```

`factory/backlog.md:12` names them differently:

```
guide:serve
verifier:serve
```

The runbook also documents `cluster:login` (matches backlog) but omits `lab:reset <n>` (in backlog).
`cluster:snapshot` and `cluster:reset` are reasonable additions not yet in the backlog, but the naming
divergence on the two primary serve tasks (`lab:serve` vs `guide:serve`, `lab:verify` vs `verifier:serve`)
will cause confusion when the mise-tasks backlog item is picked up.

**Fix:** Align section 9 task names with `factory/backlog.md:12`:
- `lab:serve` → `guide:serve`
- `lab:verify` → `verifier:serve`
- Add `lab:reset <n>` to the checklist (it is already in the backlog)

### F2 — REQUIRED — dnsmasq wildcard directive is subtly wrong (docs/sno-proxmox.md:124)

Line 124 of the dnsmasq block reads:

```ini
address=/.apps.sno.lab.local/192.168.1.50
```

The correct dnsmasq wildcard prefix for "all subdomains of `apps.sno.lab.local`" is
`address=/.apps.sno.lab.local/…` — this is actually correct dnsmasq syntax (leading `.`
matches all subdomains). However the DNS table at line 91 uses `*.apps.<cluster>.<domain>` (the
standard wildcard notation), while the dnsmasq block uses `.apps.sno.lab.local`. These are
equivalent in effect, but the mismatch between the wildcard notation in the DNS table and the
dnsmasq prefix notation in the code block is confusing to readers. Add a brief inline comment
clarifying that the leading `.` in dnsmasq matches all subdomains (equivalent to `*.apps.sno.lab.local`):

```ini
address=/.apps.sno.lab.local/192.168.1.50   # leading dot = all *.apps.sno.lab.local
```

### F3 — MINOR — SNO docs link points to OCP 4.13, not 4.22 (docs/sno-proxmox.md:341)

```
- [SNO install docs (Red Hat)](https://docs.redhat.com/…/openshift_container_platform/4.13/…)
```

The build notes acknowledge this gap ("should be updated to 4.22 docs once Red Hat publishes them"),
but there is no inline callout in the document itself. A reader following the link will land on
older docs without warning. Add a short parenthetical: `(4.13 docs — update to 4.22 once published)`.

### F4 — MINOR — `lab:verify` task name may collide with future naming (docs/sno-proxmox.md:332)

Separate from F1: if the backlog name `verifier:serve` is adopted, the name `lab:verify`
("start the verifier API") is ambiguous — it reads as "run the verifier once" rather than
"serve the verifier daemon". Addressed by F1's fix to `verifier:serve`.

---

## Other checks

### .markdownlint.yaml — justified and minimal

The file adds three keys: `MD013.line_length: 80`, `MD013.tables: false`, `MD013.code_blocks: false`.
This is the minimal config needed to suppress MD013 only inside tables and code blocks while keeping
the 80-character prose limit. No other rules are relaxed. Justified addition.

### .gitignore coverage — adequate

Patterns `kubeconfig*`, `*.kubeconfig`, and `auth/` cover both recommended paths (`auth/kubeconfig`
and `kubeconfig-<cluster-name>`). `pull-secret*` covers `pull-secret.json`. No changes needed.

### Technical accuracy — Assisted Installer / SNO flow

- Assisted Installer online flow is correctly described (discovery ISO, host inventory, operator
  progress, 40–60 min estimate). ✅
- Three required DNS records (`api.`, `api-int.`, `*.apps.`) are all present and correct. ✅
- DHCP reservation documented as primary path, static IP as alternative — matches spec. ✅
- Proxmox VM settings (q35 machine type, OVMF UEFI, VirtIO disk/NIC, EFI disk, `host` CPU type)
  are all correct for SNO on Proxmox. ✅
- AC5 note: the spec says "Assisted Installer or agent-based ISO"; the runbook documents the
  online Assisted Installer only. The agent-based ISO (offline/disconnected install) is a
  distinct flow. This is acceptable given the spec's open question 1 defaulted to online,
  but the runbook heading (`## 4. Assisted Installer Walkthrough`) could mention that the
  agent-based ISO is the equivalent offline path. Low severity; not blocking.
- Port 6443 appears correctly in the troubleshooting table as prose only (does not trigger AC12). ✅

### Verifier port references in section 9

`lab:serve` / `lab:verify` lines reference ports 8080 and 8081 correctly per `docs/architecture.md`.
The naming issue (F1) does not affect the port values.
