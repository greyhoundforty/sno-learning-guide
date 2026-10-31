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
address=/.apps.sno.lab.local/192.168.1.50
```

Restart `dnsmasq` and confirm with `dig api.sno.lab.local`.

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
   - **OpenShift version:** select **4.22** (or the newest version in the
     dropdown that matches the EX280 exam objectives — check the current
     [EX280 exam page](https://www.redhat.com/en/services/training/ex280) to
     confirm the exam version).
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
3. Return to the Assisted Installer — within a few minutes the host should
   appear under **Host inventory** with status **Ready** or **Known**.

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

---

## 5. Post-Install: Credentials and KUBECONFIG

### 5.1 Retrieve kubeadmin credentials

At the end of the installation the Assisted Installer console shows a
**`kubeadmin` password**. This is the only time it is shown in the console.

- Store it immediately in a **password manager** (e.g. 1Password, Bitwarden,
  macOS Keychain).
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

1. In the Proxmox web UI, select the SNO VM.
2. Navigate to **Snapshots** → **Take Snapshot**.
3. Name it `post-install-clean` (or similar).
4. Leave **Include RAM** unchecked (the VM should be powered off or suspended
   first for a consistent snapshot).

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
| Host not appearing in Assisted Installer | Network/firewall | Ensure the VM can reach `api.openshift.com` |
| Install stalls at bootstrap | DNS misconfigured | Confirm `api.<cluster>.<domain>` resolves to the node IP |
| `oc` command not found | OpenShift CLI not installed | Download `oc` from console.redhat.com/openshift/downloads |
| `oc login` fails | Wrong API URL | Use `https://api.<cluster-name>.<base-domain>` (port 6443) |
| Operators degraded after boot | Slow node start | Wait 10–15 min; run `oc get clusteroperators` to check |

---

## 9. Planned Mise Tasks

The following `mise` tasks are **planned** for this project. They are
documented here as a checklist; implementation is a separate backlog item.

- [ ] `cluster:login` — authenticate to the cluster using `KUBECONFIG` and
  verify connectivity with `oc whoami`.
- [ ] `cluster:status` — run `oc get clusterversion` and
  `oc get clusteroperators` and surface a summary.
- [ ] `cluster:snapshot` — trigger a Proxmox snapshot of the SNO VM via the
  Proxmox API.
- [ ] `cluster:reset` — revert the SNO VM to the `post-install-clean` snapshot
  via the Proxmox API.
- [ ] `lab:serve` — start the guide static server on port 8080.
- [ ] `lab:verify` — start the verifier API on port 8081.

See [`mise.toml`](../mise.toml) for the currently implemented tasks.

---

## Reference Links

- [Red Hat Assisted Installer](https://console.redhat.com/openshift/assisted-installer/clusters)
- [SNO install docs (Red Hat)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.13/html/installing_on_a_single_node/install-sno-installing-sno)
- [EX280 exam objectives](https://www.redhat.com/en/services/training/ex280)
- [OpenShift CLI download](https://console.redhat.com/openshift/downloads)
