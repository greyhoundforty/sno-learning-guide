# Installing Single Node OpenShift on Proxmox

This runbook walks through installing Single Node OpenShift (SNO) on a Proxmox
host using the Red Hat Assisted Installer. The target is OpenShift 4.22 for
EX280 lab practice.

---

## Prerequisites

- A Proxmox node with enough capacity (see sizing table below).
- DNS that you control, or the ability to add entries to `/etc/hosts` or a
  local `dnsmasq` instance.
- A Red Hat account at [console.redhat.com](https://console.redhat.com) with a
  pull secret downloaded.
- The Proxmox web UI or SSH access.
- **OpenShift CLI (`oc`)** installed on your workstation. Download it from
  [console.redhat.com/openshift/downloads](https://console.redhat.com/openshift/downloads)
  and place it on your `PATH`. On macOS you can also run `brew install openshift-cli`.

---

## 1. VM Sizing

> Red Hat's minimum is tight for real lab work. Use the recommended row if
> you can.

| Profile | vCPU | RAM | Disk | Notes |
| --- | --- | --- | --- | --- |
| Red Hat minimum | 8 | 16 GB | 120 GB | Boots; little headroom for operators |
| **Recommended (this lab)** | **12** | **32 GB** | **150–200 GB** SSD/NVMe | EX280 labs |
| Comfortable | 16 | 48 GB | 250 GB+ | Adds logging, ODF-lite, virtualization |

The recommended row is a lab judgment call, not a Red Hat figure. Revisit once
you see real usage. etcd is latency-sensitive; **avoid spinning disk or
heavily-shared NFS for the root volume**.

---

## 2. Proxmox VM Configuration

### 2.1 Create the VM

In the Proxmox web UI, create a new VM with these settings:

| Setting | Value |
| --- | --- |
| Machine type | `q35` |
| BIOS | OVMF (UEFI) — add an EFI disk when prompted |
| CPU type | `host` |
| vCPU | 12 (recommended) |
| RAM | 32 GB (recommended) |
| Disk bus | VirtIO Block (`virtio-scsi` or `virtio-blk`) |
| Disk size | 150 GB minimum |
| Network | VirtIO NIC, bridged to your LAN bridge (e.g. `vmbr0`) |
| Display | Default (SPICE or VNC) |

> **SeaBIOS alternative:** If you cannot use OVMF, SeaBIOS (BIOS mode) also
> works with the Assisted Installer. UEFI is preferred for SNO.
>
> **QEMU Guest Agent:** Not required for SNO installation. Skip it unless you
> want richer Proxmox integration post-install.

### 2.2 Network — DHCP reservation (recommended)

The Assisted Installer needs a stable IP for the node before it can configure
DNS and finish the install. The cleanest approach on a home or lab network is a
**DHCP reservation** tied to the VM's MAC address:

1. Note the MAC address Proxmox assigned to the VirtIO NIC.
2. In your router or DHCP server, create a reservation:
   `<node-mac-address>` → `<node-ip-address>` (e.g. `192.168.1.50`).
3. Confirm the VM receives the reserved IP at boot.

### 2.3 Network — static IP (alternative)

If DHCP reservation is not available, configure a static IP in the Assisted
Installer's **Networking** section:

- IP address: `<node-ip-address>`
- Gateway: `<gateway-ip>`
- DNS: `<dns-ip>` (can be the same host running `dnsmasq`, or `8.8.8.8`)

---

## 3. DNS Setup

SNO requires three DNS records pointing at the single node IP. Without them,
the API endpoint and application routes will not resolve.

| Record | Type | Value |
| --- | --- | --- |
| `api.<cluster-name>.<base-domain>` | A | `<node-ip-address>` |
| `api-int.<cluster-name>.<base-domain>` | A | `<node-ip-address>` |
| `*.apps.<cluster-name>.<base-domain>` | A | `<node-ip-address>` |

**Example** — if `<cluster-name>` is `sno` and `<base-domain>` is `lab.local`:

```text
api.sno.lab.local       A   192.168.1.50
api-int.sno.lab.local   A   192.168.1.50
*.apps.sno.lab.local    A   192.168.1.50
```

### 3.1 /etc/hosts fallback (single machine only)

If you only need resolution from your workstation and do not have a local DNS
server, add entries to `/etc/hosts`. Note that the wildcard `*.apps` record
**cannot** be expressed in `/etc/hosts`; you must add individual app hostnames
as they come up, or use the `dnsmasq` approach below.

```text
192.168.1.50  api.sno.lab.local
192.168.1.50  api-int.sno.lab.local
192.168.1.50  console-openshift-console.apps.sno.lab.local
192.168.1.50  oauth-openshift.apps.sno.lab.local
```

### 3.2 dnsmasq wildcard (recommended for lab hosts)

On a Linux host running `dnsmasq`, add a single wildcard file so all `*.apps`
routes resolve automatically:

```ini
# /etc/dnsmasq.d/sno.conf
address=/api.sno.lab.local/192.168.1.50
address=/api-int.sno.lab.local/192.168.1.50
address=/.apps.sno.lab.local/192.168.1.50   # leading dot = all *.apps.sno.lab.local
```

Restart `dnsmasq`, then verify all three records resolve before proceeding:

```bash
# Expected output: the node IP on each line
dig +short api.sno.lab.local
dig +short api-int.sno.lab.local
dig +short foo.apps.sno.lab.local   # tests the wildcard
```

If any command returns no output or a wrong address, re-check `dnsmasq.conf`
and restart `dnsmasq` before continuing. A DNS misconfiguration will stall the
installer 40+ minutes into the bootstrap phase.

---

## 4. Assisted Installer Walkthrough

The Assisted Installer at [console.redhat.com](https://console.redhat.com) is
the supported online path for SNO. It provides a discovery ISO that boots on
the VM, collects hardware information, and drives the install from the console.

### 4.1 Download your pull secret

1. Log in to [console.redhat.com](https://console.redhat.com).
2. Navigate to **OpenShift → Downloads** or the cluster creation wizard.
3. Click **Copy pull secret** or **Download pull secret** and save it as
   `pull-secret.json` somewhere outside this repository.

> `pull-secret.json` matches the `pull-secret*` glob in `.gitignore` and will
> not be committed. Keep it outside the repo to be safe.

### 4.2 Create the cluster in the Assisted Installer

1. Go to **OpenShift → Assisted Installer** on
   [console.redhat.com](https://console.redhat.com).
2. Click **Create cluster**.
3. Fill in the cluster details:
   - **Cluster name:** `<cluster-name>` (e.g. `sno`)
   - **Base domain:** `<base-domain>` (e.g. `lab.local`)
   - **OpenShift version:** select **4.22**. If 4.22 is not in the dropdown,
     select the highest available **4.22.x** build. If no 4.22 build is listed
     at all, select the highest available 4.x version and follow the fallback
     procedure in §7 to confirm what was installed.
   - **Host inventory:** choose **Single node OpenShift (SNO)**.
4. Paste your pull secret when prompted.
5. Configure SSH public key (optional but useful for node debugging).
6. Review networking. If using a static IP, switch to **Static IP** mode and
   enter `<node-ip-address>`, gateway, and DNS server.

### 4.3 Generate and download the discovery ISO

1. Proceed to the **Host discovery** step.
2. Click **Generate Discovery ISO**.
3. Download the ISO file (typically `discovery_image_<cluster-name>.iso`).

### 4.4 Upload the ISO to Proxmox

1. In the Proxmox web UI, navigate to your node → **local** (or another ISO
   storage) → **ISO Images**.
2. Click **Upload** and select the discovery ISO.
3. In the VM's hardware view, attach the ISO to the CD/DVD drive.
4. Set the VM boot order: CD/DVD first, then the VirtIO disk.

### 4.5 Boot the discovery image

1. Start the VM in Proxmox.
2. Watch the console (SPICE/VNC) — the discovery image boots, runs a minimal
   Linux environment, and phones home to the Assisted Installer.
3. Return to the Assisted Installer — within **3–8 minutes** the host should
   appear under **Host inventory** with status **Ready** or **Known**.

> **Console activity:** while the discovery image is running you will see log
> lines scrolling in the VM console as it contacts the Assisted Installer.
> This is normal. If the console goes quiet with no activity after 5 minutes
> and the host has not appeared, SSH into the node
> (`ssh core@<node-ip-address>`) and run `journalctl -u agent -f` to watch
> the registration attempt. The most common cause is a firewall blocking
> outbound HTTPS to `api.openshift.com` (port 443).

### 4.6 Set the installation disk

1. In the Assisted Installer, expand the discovered host.
2. Under **Installation disk**, select the VirtIO block device (e.g.
   `/dev/vda`).
3. Confirm the host's hostname, IP address, and hardware checks are all green.

### 4.7 Run the installation

1. Click **Install cluster**.
2. The console shows per-step progress: bootstrap, master, cluster operators.
3. Full installation typically takes **40–60 minutes** on SSD-backed storage.
4. When the progress bar reaches 100 % and all operators are available, the
   install is complete.

> **Tip:** The VM console will reboot part-way through. After reboot the node
> continues the install from the local disk; the discovery ISO is no longer
> needed.
>
> **After installation:** remove the discovery ISO from the CD/DVD drive or
> change the boot order to **disk first** in the VM hardware settings. If the
> ISO stays as the first boot device, the node will boot back into the
> discovery image on every future restart.

---

## 5. Post-Install: Credentials and KUBECONFIG

### 5.0 Wait for cluster operators

Installation "completes" when the progress bar reaches 100 %, but on a
resource-constrained node some cluster operators may still be converging. Check
operator status before downloading credentials:

```bash
oc get clusteroperators
```

Expected output: all operators show `AVAILABLE=True`, `PROGRESSING=False`,
`DEGRADED=False`. If any operator shows `PROGRESSING=True`, wait 5–10 minutes
and re-run. This is normal on 32 GB RAM nodes and does not indicate a failed
install. Once `oc get clusterversion` shows `PROGRESSING=False`, the cluster
is fully ready.

### 5.1 Retrieve kubeadmin credentials

At the end of the installation the Assisted Installer console shows the
**`kubeadmin` password** in the cluster detail page under **Credentials**.

- Store it immediately in a **password manager** (e.g. 1Password, Bitwarden,
  macOS Keychain).
- You can retrieve it again from the Assisted Installer console as long as the
  cluster record has not been deleted — but do not rely on this; store it now.
- **Never** write it into a file in this repository, a shell variable export,
  or a `.env` file that could be committed.

### 5.2 Download the cluster credential file

1. In the Assisted Installer, click the **Download KUBECONFIG** button to
   obtain the cluster credential file (the `KUBECONFIG` file).
2. Save it to a path covered by `.gitignore`. Both patterns below are already
   gitignored in this project — use either:

   ```text
   auth/kubeconfig
   ```

   or

   ```text
   kubeconfig-<cluster-name>
   ```

3. Export it in your shell session:

   ```bash
   export KUBECONFIG=<path-to-kubeconfig>
   ```

   For example:

   ```bash
   export KUBECONFIG="$HOME/sno-learning-guide/auth/kubeconfig"
   ```

4. Verify cluster access:

   ```bash
   oc get clusterversion
   ```

   The output should show the installed version (e.g. `4.22.x`).

### 5.3 Persist KUBECONFIG across sessions

Add the export to your shell profile (`.zshrc`, `.bashrc`) or to a local
`.env` file (gitignored):

```bash
# In ~/.zshrc or a gitignored .env — never in a tracked file
export KUBECONFIG="$HOME/sno-learning-guide/auth/kubeconfig"
```

---

## 6. Post-Install: Proxmox Snapshot

Taking a snapshot immediately after a clean install lets you reset the lab
environment quickly without re-running the full installer.

1. Shut down the node cleanly before snapshotting to avoid etcd corruption:

   ```bash
   ssh core@<node-ip-address> sudo shutdown -h now
   ```

   Alternatively, use **Stop** (not **Force Stop**) in the Proxmox web UI.
   Wait for the VM status to show **stopped** before continuing.

2. In the Proxmox web UI, select the SNO VM.
3. Navigate to **Snapshots** → **Take Snapshot**.
4. Name it `post-install-clean` (or similar).
5. Leave **Include RAM** unchecked (the VM is stopped).

> **Lab reset workflow:** Revert to `post-install-clean`, start the VM, wait
> for the cluster to stabilize (all operators ready), then run your lab.

---

## 7. Verifying the Version

After connecting with `oc`, confirm the installed version matches the exam
target:

```bash
oc get clusterversion
```

Expected output (example):

```text
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.22.1    True        False         2d      Cluster version is 4.22.1
```

If the Assisted Installer did not offer 4.22, select the highest available
4.x version and note the gap in `docs/ex280-coverage.md`.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Host not appearing in Assisted Installer | Network/firewall | Ensure the VM can reach `api.openshift.com`. From the VM console, `ssh core@<node-ip-address>`, then run `journalctl -u agent -f` to see registration errors |
| Install stalls at bootstrap | DNS misconfigured | Confirm `api.<cluster>.<domain>` resolves to the node IP with `dig +short api.<cluster>.<domain>` |
| `oc` command not found | OpenShift CLI not installed | Download `oc` from [console.redhat.com/openshift/downloads](https://console.redhat.com/openshift/downloads) or run `brew install openshift-cli` |
| `oc login` fails | Wrong API URL or port | Use `https://api.<cluster-name>.<base-domain>` with port 6443 (the API server listens on port 6443) |
| Operators degraded after boot | Slow node start | Wait 10–15 min; run `oc get clusteroperators` to check; `PROGRESSING=True` is normal during convergence |

---

## 9. Planned Mise Tasks

The following `mise` tasks are **planned** for this project. They are
documented here as a checklist; implementation is a separate backlog item.

- [ ] `guide:serve` — start the guide static server on port 8080.
- [ ] `verifier:serve` — start the verifier API on port 8081.
- [ ] `cluster:login` — authenticate to the cluster using `KUBECONFIG` and
  verify connectivity with `oc whoami`.
- [ ] `lab:reset <n>` — revert the lab environment to the clean snapshot for
  lab number `<n>`.

The following tasks are additional cluster helpers (**optional extras**, not
yet in the backlog — add them when the mise-tasks item is picked up):

- [ ] `cluster:status` — run `oc get clusterversion` and
  `oc get clusteroperators` and surface a summary.
- [ ] `cluster:snapshot` — trigger a Proxmox snapshot of the SNO VM via the
  Proxmox API.

See [`mise.toml`](../mise.toml) for the currently implemented tasks.

---

## Reference Links

- [Red Hat Assisted Installer](https://console.redhat.com/openshift/assisted-installer/clusters)
- [SNO install docs (Red Hat)][sno-docs] — 4.13 docs; update link to 4.22 once published
- [EX280 exam objectives](https://www.redhat.com/en/services/training/ex280)
- [OpenShift CLI download](https://console.redhat.com/openshift/downloads)

[sno-docs]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.13/html/installing_on_a_single_node/install-sno-installing-sno
