VERDICT: CHANGES

## Summary

The runbook is structurally sound and covers the right territory. A first-time learner can follow the numbered sections from VM creation through a working `oc get clusterversion`. However, several gaps create friction or silent failure points that will stop a learner cold: the `oc` CLI is never installed, the version-selection path is ambiguous enough to produce a mismatched cluster, the discovery ISO boot has no failure signal, and the troubleshooting table is too thin for the real failure modes a first-timer hits. These are fixable; no structural rework is needed.

---

## Findings

### F1 — `oc` CLI is never installed (blocking)

The runbook assumes `oc` is present from step 5.2 onward but never tells the learner how to install it. The troubleshooting table mentions "OpenShift CLI not installed" as a known symptom, which means the author knows this can happen — yet there is no install step.

A learner who hits `oc: command not found` has to leave the runbook and self-rescue. For an EX280 candidate this is a common first failure.

**Fix:** Add a prerequisite bullet — or a small §0 "Before you start" section — that covers installing `oc` (download from console.redhat.com/openshift/downloads or `brew install openshift-cli`). Alternatively add it as §5.0 immediately before the first `oc` command.

---

### F2 — OpenShift 4.22 version guidance is ambiguous (blocking for exam alignment)

Step 4.2 says: select **4.22** "or the newest version in the dropdown that matches the EX280 exam objectives — check the current EX280 exam page to confirm the exam version."

This is contradictory. If 4.22 is the target, say so and stop. If the learner should check the exam page, the runbook should tell them *what to look for* on that page and *what to do* when 4.22 is not listed (is 4.21 acceptable? is 4.23 acceptable?). The fallback in §7 ("select the highest available 4.x version and note the gap") is buried three screens after the decision point.

**Fix:** Consolidate the version-selection guidance into one place (step 4.2). State explicitly: "Select 4.22. If 4.22 is not in the dropdown, select the highest available 4.22.x. If no 4.22 build is listed, see §7 for the fallback procedure." Remove the open-ended "check the exam page" aside — it shifts the cognitive burden to the learner at the worst moment.

---

### F3 — No verification step after DNS setup (blocking)

Section 3 specifies DNS records but ends without a command to confirm they resolve. A learner who has a typo in `dnsmasq.conf` or who saved `/etc/hosts` to the wrong file will not find out until the Assisted Installer stalls at bootstrap — a 40-minute wait before the error.

Section 3.2 mentions `dig api.sno.lab.local` in passing, but only as part of the restart instruction, not as an explicit verification step.

**Fix:** End §3 with an explicit check box:

```bash
# Verify all three records resolve before proceeding
dig +short api.sno.lab.local
dig +short api-int.sno.lab.local
dig +short foo.apps.sno.lab.local   # tests the wildcard
```

Expected output: the node IP on each line. Failure action: re-check dnsmasq or /etc/hosts.

---

### F4 — Discovery ISO boot: no time estimate or failure signal (high friction)

Step 4.5 says "within a few minutes the host should appear … with status Ready or Known." There is no guidance on:
- How long "a few minutes" means (typically 3–8 min).
- What the VM console looks like while it is phoning home (so the learner knows it is working, not hung).
- What to do if the host does not appear after 10 minutes.

The troubleshooting table covers "Host not appearing" with "Ensure the VM can reach api.openshift.com" — but not how to verify that from the Proxmox console.

**Fix:** Add a note after step 4.5.2:
> The discovery image prints log lines to the console as it contacts the Assisted Installer. If you see repeated connection-refused messages or no console activity after 5 minutes, SSH into the node (`ssh core@<node-ip>`) and run `journalctl -u agent -f` to watch the registration attempt. The most common cause is a firewall blocking outbound HTTPS to `api.openshift.com` (port 443).

Update the troubleshooting row to include this diagnostic command.

---

### F5 — `kubeadmin` password: "only time it is shown" is not accurate (medium)

Section 5.1 says the kubeadmin password is shown "at the end of the installation" and "this is the only time it is shown in the console." In practice the Assisted Installer keeps the credentials accessible in the cluster detail view until the cluster is deleted. Telling a learner they have one shot creates unnecessary anxiety and can cause them to rush and make a storage mistake.

**Fix:** Revise to: "The kubeadmin password is shown in the Assisted Installer cluster detail page under **Credentials**. Store it in a password manager now; you can retrieve it from the console again before the cluster is deleted, but do not rely on that."

---

### F6 — Boot-order change after install is not mentioned (medium)

Step 4.4 says to set boot order: CD/DVD first, then VirtIO disk. The tip in §4.7 notes the VM reboots mid-install. What the tip does not say is that after the install completes, the VM will attempt to boot from the ISO again on any subsequent restart unless the boot order is changed.

A learner who snapshots (§6) or reboots post-install and sees the discovery image re-appear will be confused.

**Fix:** Add to the §4.7 tip (or as its own step after §4.7): "After installation completes, remove the discovery ISO from the CD/DVD drive or change the boot order to disk-first. This prevents the node from re-booting into the discovery image on future restarts."

---

### F7 — Post-install operator degradation check is missing (medium)

The install "completes" at §4.7 when the progress bar hits 100%. However, on a resource-constrained node it is common for some cluster operators to still be progressing or degraded immediately after the installer reports success. The learner has no instruction to wait or verify.

Section 5.2 jumps straight to downloading KUBECONFIG and running `oc get clusterversion`, but if operators are still converging, `oc get clusterversion` will show `PROGRESSING=True`, which can look like a failure.

**Fix:** Add a step between §4.7 and §5 (or as §5.0):

```bash
oc get clusteroperators
```

> Expected: all operators show `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False`. If any operator shows `PROGRESSING=True`, wait 5–10 minutes and re-run. This is normal on nodes with 32 GB RAM.

---

### F8 — Snapshot section (§6) says to power off but the workflow is unclear (low)

Section 6 says "the VM should be powered off or suspended first for a consistent snapshot." It does not say *how* to power off cleanly — `oc adm node-maintenance`, Proxmox shutdown button, or `sudo shutdown -h now` via SSH. Using the Proxmox "force off" while the cluster is running can corrupt etcd.

**Fix:** Add: "Shut down cleanly: `ssh core@<node-ip> sudo shutdown -h now`, or use **Stop** in Proxmox web UI (not **Force Stop**). Wait for the VM status to show **stopped** before taking the snapshot."

---

### F9 — Reference link to SNO docs points to 4.13, not 4.22 (low)

The Reference Links section links to the SNO install docs for OCP 4.13. The build.md notes this is a known gap. For a learner who clicks through to verify something version-specific, this will give outdated information.

The build.md says "update to 4.22 docs once Red Hat publishes them." That is fine as a deferral, but the link should at minimum warn the learner that it points to a prior version.

**Fix:** Append `(4.13 — update when 4.22 docs are published)` to the link label, so learners know the version mismatch is intentional.

---

### F10 — `oc login` troubleshooting row missing the port (low)

The troubleshooting table says: `oc login` fails → use `https://api.<cluster-name>.<base-domain>` — but does not include `:6443`. A learner following this hint without the port will get a connection refused. The port is mentioned in prose earlier in the same table row, but it is easy to miss.

**Fix:** Change the Fix cell to: `Use https://api.<cluster-name>.<base-domain>:6443` (the API server listens on port 6443).

---

## What works well

- The three-tier sizing table with a clear **Recommended** callout is easy to scan and makes the right choice obvious.
- DHCP-reservation-first with static-IP fallback is the correct ordering for a home lab.
- The gitignore / credential storage guidance is specific and actionable — no vague "be careful with secrets."
- The dnsmasq wildcard approach is the right default for *.apps and the example config is copy-paste ready.
- §9 "Planned Mise Tasks" is honest about what is not implemented and sets clear expectations.
- The troubleshooting table exists at all — most first-time runbooks omit it entirely.
