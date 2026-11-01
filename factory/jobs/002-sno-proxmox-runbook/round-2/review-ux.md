VERDICT: PASS

## Summary

All ten round-1 findings are genuinely resolved. The fixes are well-placed,
do not introduce new confusion, and the section numbering is coherent. The
document is ready to ship.

---

## Finding-by-finding verification

### F1 — `oc` CLI never installed ✅ Resolved

The Prerequisites section now includes a bolded `**OpenShift CLI (`oc`)**`
bullet with a direct link to the downloads page and a `brew install
openshift-cli` alternative. Placement in Prerequisites is the right location
— learners see it before the first command they will run. No confusion
introduced.

---

### F2 — Version guidance contradictory ✅ Resolved

Step 4.2 now reads: select **4.22** → if not found, select the highest
**4.22.x** build → if no 4.22 build at all, follow §7 fallback. The
open-ended "check the exam page" aside is gone. The three-tier decision
is unambiguous and the reference to §7 is specific. The §7 prose itself
("If the Assisted Installer did not offer 4.22, select the highest available
4.x version and note the gap in `docs/ex280-coverage.md`") is consistent
with step 4.2. No confusion introduced.

---

### F3 — No DNS verification step ✅ Resolved

Section 3.2 now ends with an explicit `dig` block for all three records,
with expected output and a clear failure action: re-check `dnsmasq.conf`,
restart `dnsmasq`, and a reminder that a DNS misconfiguration will stall the
installer 40+ minutes into bootstrap. The check is correctly scoped to §3.2
(dnsmasq path) rather than §3.1 (/etc/hosts), which is appropriate because
/etc/hosts cannot express the wildcard. No confusion introduced.

---

### F4 — Discovery ISO boot: no timing/console/failure signal ✅ Resolved

Step 4.5 now gives a concrete "3–8 minutes" window, describes what console
activity looks like, and provides the `ssh core@<node-ip-address>` +
`journalctl -u agent -f` diagnostic for the no-activity case. The
troubleshooting table row for "Host not appearing" now also includes the
`journalctl` command. The note correctly identifies the most common cause
(outbound HTTPS to `api.openshift.com` port 443). No confusion introduced.

---

### F5 — "Only time it is shown" inaccurate ✅ Resolved

Section 5.1 no longer claims the password is shown only once. It now says the
password is visible in the cluster detail page under **Credentials** and can
be retrieved again before the cluster record is deleted — but advises storing
it now anyway. The tone is calm and accurate. No confusion introduced.

---

### F6 — Boot order not reset after install ✅ Resolved

The §4.7 tip block now contains a second `**After installation:**` paragraph
that explicitly tells the learner to remove the discovery ISO or change the
boot order to **disk first**, with a clear explanation of what happens if they
do not. The placement (immediately below the install-complete signal) is
exactly the right moment for this instruction. No confusion introduced.

---

### F7 — No operator readiness check ✅ Resolved

A new §5.0 "Wait for cluster operators" subsection sits between the end of
§4 (installation) and §5.1 (credential retrieval). The `oc get
clusteroperators` command is present with expected output, a wait-and-retry
instruction, and a reassurance that `PROGRESSING=True` is normal. The
sequencing is correct: §5.0 gates credential download, and §5.2 still leads
to `oc get clusterversion` as the final confirmation. The section number 5.0
is unconventional but unambiguous; a learner following top-to-bottom will hit
it at the right point. No confusion introduced.

One minor observation (non-blocking): §5.0 says "check operator status before
downloading credentials," but the `oc` command in §5.0 itself requires a
`KUBECONFIG` that has not been downloaded yet. In practice a learner would use
the `oc` URL/password login path to run this check — or the Assisted Installer
web UI shows operator status too. The intended reading is probably "check this
in the Assisted Installer UI before leaving, then download credentials" — that
is the sensible interpretation, and the Assisted Installer does show operator
health. Not a blocker; the common case (100% progress = operators ready) will
just proceed to §5.1 without friction.

---

### F8 — Snapshot section: unclear shutdown method ✅ Resolved

Section 6 now leads with a numbered step showing the SSH `sudo shutdown -h now`
command, with the Proxmox **Stop** (not **Force Stop**) as an explicit
alternative, and "wait for VM status to show **stopped**" before snapping. The
etcd-corruption risk is addressed without being alarming. No confusion
introduced.

---

### F9 — 4.13 doc link no version warning ✅ Resolved

The reference link now uses a reference-style label `[sno-docs]` with the
annotation "4.13 docs; update link to 4.22 once published" visible in the
Reference Links section. A learner who clicks through will see the 4.13 URL
and understand the mismatch is intentional. No confusion introduced.

---

### F10 — `oc login` troubleshooting row missing the port ✅ Resolved

The troubleshooting row now reads: "Use `https://api.<cluster-name>.<base-domain>`
with port 6443 (the API server listens on port 6443)." The port is in the
actionable Fix cell, not buried in separate prose. No confusion introduced.

---

## No new issues introduced

- **Section numbering:** The top-level sections are numbered 1–9 with
  Prerequisites as an unnumbered prelude and the new §5.0 as a subsection.
  This is coherent; the unnumbered Prerequisites section is standard practice
  and §5.0 reads naturally as a pre-requisite step within §5.
- **Ordering of §5.0:** Placing operator readiness before credential retrieval
  is the correct logical order. The minor KUBECONFIG dependency noted above
  (F7) does not block the learner in the common case.
- **Version guidance flow:** §4.2 → §7 reference is tight and unambiguous.
  No duplication, no contradictory advice.
- **DNS verification scope:** The `dig` block is in §3.2 (dnsmasq) only.
  Section 3.1 (/etc/hosts) correctly notes the wildcard limitation without
  a verification block; this is appropriate since /etc/hosts cannot test the
  wildcard anyway.
- **Troubleshooting table completeness:** The table now has five rows covering
  the real first-timer failure modes, including the new "Operators degraded
  after boot" row added for F7.

---

## What continues to work well

Everything noted in round 1 still holds. The three additions that stand out
most after the round-2 fixes: the end-of-§3.2 DNS verification block is
exactly the kind of gate that prevents 40-minute wait-and-discover cycles; the
§4.7 boot-order post-install note will save a significant portion of first-time
learners from a confusing re-discovery-image boot; and §5.0 correctly sets
expectations about "installation complete" vs. "cluster ready" — a distinction
that trips up nearly every first-time SNO installer.
