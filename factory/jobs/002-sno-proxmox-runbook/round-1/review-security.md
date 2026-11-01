VERDICT: PASS

## Security Review — job 002-sno-proxmox-runbook, round 1

Reviewer: reviewer-security  
Branch: factory/002-sno-proxmox-runbook  
Files reviewed: `docs/sno-proxmox.md`, `.markdownlint.yaml`, `build.md`

---

### Credential handling

**kubeadmin password** — `docs/sno-proxmox.md:213–217`  
The runbook correctly instructs the reader to store the kubeadmin password in a
password manager and explicitly prohibits writing it into a repository file,
shell export, or `.env` file. No literal password value appears anywhere in the
diff. ✅

**Pull secret** — `docs/sno-proxmox.md:141–145`  
The reader is told to save `pull-secret.json` *outside* the repository, and the
note at line 144 correctly cross-references the `.gitignore` `pull-secret*` glob.
No pull-secret value or JSON blob appears in the diff. ✅

**KUBECONFIG paths** — `docs/sno-proxmox.md:228–264`  
Every kubeconfig reference either:
- uses the gitignored path `auth/kubeconfig` (covered by `.gitignore`:`auth/`), or
- uses the placeholder `kubeconfig-<cluster-name>` (covered by `.gitignore`:`kubeconfig*`), or
- uses the environment variable name `KUBECONFIG` (the uppercase marker, not a
  path).

No bare path outside the gitignored prefixes is used. ✅

**`.gitignore` coverage** — `.gitignore` (on `main`, unchanged by this diff)  
Confirmed patterns present: `kubeconfig*`, `*.kubeconfig`, `auth/`, `pull-secret*`,
`.env`, `.env.*`. These cover all paths the runbook recommends. ✅

---

### Secret-like strings

No literal tokens, passwords, certificates, private keys, or base64-encoded
blobs appear in any added line. The IP addresses (`192.168.1.50`) are generic
RFC-1918 lab examples, not real infrastructure credentials. ✅

---

### Shell commands and command injection

Shell snippets in the runbook are limited to:

- `export KUBECONFIG=<path-to-kubeconfig>` — placeholder, not evaluated.
- `oc get clusterversion` — read-only, no user input interpolated.
- `oc get clusteroperators` — read-only, no user input interpolated.
- `dig api.sno.lab.local` — DNS query, no user input interpolated.

No command substitution (`$(...)`), `eval`, piped `curl | sh`, or unsanitized
variable interpolation appears. ✅

---

### API and ingress exposure

The API endpoint (`https://api.<cluster-name>.<base-domain>:6443`) and ingress
wildcard (`*.apps.<cluster-name>.<base-domain>`) are described using
placeholders throughout. No real cluster hostname or routable IP outside the
RFC-1918 example block is exposed. The lab topology is explicitly private
(local DNS, home/lab network DHCP). ✅

---

### `.markdownlint.yaml`

This file contains only linting configuration (line-length rules). No
credentials, no sensitive paths. ✅

---

### `build.md`

No changes to `build.md` in this diff. Out of scope. ✅

---

### Summary

No findings require changes. All credential paths are gitignored, no literal
secrets appear, all shell examples are read-only or use placeholders, and API/
ingress references are scoped to a private lab network with placeholder
hostnames.
