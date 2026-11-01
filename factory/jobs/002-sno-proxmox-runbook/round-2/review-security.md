VERDICT: PASS

## Security Review — job 002-sno-proxmox-runbook, round 2

Reviewer: reviewer-security  
Branch: factory/002-sno-proxmox-runbook  
Files reviewed: `docs/sno-proxmox.md` (full diff), `build.md` (Round 2 section),
`factory/jobs/002-sno-proxmox-runbook/round-1/review-security.md`

---

### Scope of this round

Round 1 issued a PASS. This review re-examines the three areas added or changed
in Round 2:

1. `ssh core@<node-ip-address>` commands (two new sites)
2. `sudo shutdown -h now` command
3. `journalctl -u agent -f` command
4. Credential-handling prose changes (§5.1 rewrite, §5.0 new section)
5. Secret-like string check over the full diff

---

### New commands — `ssh core@<node-ip-address>`

**`docs/sno-proxmox.md:206`** (§4.5 diagnostic tip)

```text
ssh core@<node-ip-address>
```

The host is a template placeholder (`<node-ip-address>`). No IP address,
hostname, or credential is hardcoded. The `core` user is the standard
read-only/debug RHCOS user — no elevated privilege is granted by SSH alone.
No `-i <keyfile>` flag with a real path, no password flag. ✅

**`docs/sno-proxmox.md:364`** (§8 troubleshooting table)

```text
ssh core@<node-ip-address>
```

Same placeholder form, same RHCOS user. ✅

---

### New command — `sudo shutdown -h now`

**`docs/sno-proxmox.md:323`** (§6 snapshot preparation)

```bash
ssh core@<node-ip-address> sudo shutdown -h now
```

The command is a standard OS-level halt issued to the lab VM before taking a
Proxmox snapshot — appropriate and expected for this runbook's purpose. The
target host is a placeholder. No destructive scope beyond the local lab VM.
No user-supplied input is interpolated into the command; the reader types it
verbatim. ✅

---

### New command — `journalctl -u agent -f`

**`docs/sno-proxmox.md:206`** (§4.5 diagnostic tip) and
**`docs/sno-proxmox.md:364`** (§8 troubleshooting table)

```text
journalctl -u agent -f
```

Read-only log tail for the named unit `agent`. No unit name comes from user
input; it is the literal string `agent` (the Assisted Installer discovery
agent service). No log redirection to a file, no pipe to a shell, no sensitive
output stored anywhere in the runbook. ✅

---

### Credential handling — §5.0 and §5.1 rewrites

**§5.0 — `oc get clusteroperators`** (`docs/sno-proxmox.md:244`)  
New section; read-only `oc` command, no argument interpolation. ✅

**§5.1 — kubeadmin credential prose** (`docs/sno-proxmox.md:254–265`)  
The rewrite corrects the earlier inaccuracy ("only shown once") without
relaxing the security guidance. The prohibition on writing the password into
a file, shell export, or `.env` that could be committed is preserved verbatim.
No literal password value appears. ✅

---

### KUBECONFIG paths — re-verified

The two new `$HOME`-relative example paths introduced in Round 2:

**`docs/sno-proxmox.md:292`**

```bash
export KUBECONFIG="$HOME/sno-learning-guide/auth/kubeconfig"
```

**`docs/sno-proxmox.md:310`**

```bash
export KUBECONFIG="$HOME/sno-learning-guide/auth/kubeconfig"
```

Both resolve to `auth/kubeconfig` under the repo root — a path already covered
by the `.gitignore` `auth/` pattern. The `$HOME` prefix makes it clear this is
a shell session export, not a file write. These paths satisfy the AC10 check:
every `kubeconfig` reference includes `auth/` or `KUBECONFIG`. ✅

---

### Pull secret

No change from Round 1. `pull-secret.json` remains outside the repo,
`.gitignore` `pull-secret*` is unchanged. ✅

---

### Secret-like strings

Full scan of all added lines for tokens, passwords, certificates, private
keys, or base64 blobs:

- IP addresses (`192.168.1.50`, `8.8.8.8`) — RFC-1918 / public DNS examples,
  not credentials.
- No base64-encoded strings of meaningful length.
- No bearer tokens, API keys, or certificate PEM blocks.
- `(password|secret)\s*=` pattern → 0 matches. ✅

---

### Command injection / input interpolation

The three new shell snippets are all of the form:

- `ssh core@<placeholder> <literal-command>`
- `journalctl -u agent -f`

No shell variable interpolation, command substitution (`$(...)`), `eval`,
or piped `curl | sh` patterns. The reader cannot supply arbitrary input that
reaches a shell unsanitized — all commands are static literals with
placeholder tokens that are replaced by the reader before typing. ✅

---

### Summary

All Round 2 additions are clean. The new `ssh`, `shutdown`, and `journalctl`
commands use placeholder targets, operate on the local lab VM only, and do
not expose credentials or allow injection. Credential-handling prose was
tightened, not relaxed. No literal secrets appear anywhere in the diff.
All Round 1 PASS findings remain valid.
