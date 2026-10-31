# Initial questions

## 1. Recommended specs for an SNO VM

Red Hat's documented **minimum** for single-node OpenShift is **8 vCPU, 16 GB RAM, 120 GB storage**
([SNO install docs, 4.13](https://docs.redhat.com/en/documentation/openshift_container_platform/4.13/html/installing_on_a_single_node/install-sno-installing-sno)).
Adding Operators raises the minimum, and OpenShift Virtualization needs a second 50 GiB+ disk.

That minimum is tight once you start doing admin labs (operators, monitoring, image registry, quotas). Practical sizing for this lab:

| Profile | vCPU | RAM | Disk | Use |
|---|---|---|---|---|
| Red Hat minimum | 8 | 16 GB | 120 GB | Boots, little headroom |
| **Recommended for this lab** | **12** | **32 GB** | **150-200 GB** (SSD/NVMe-backed) | EX280 labs plus a few workloads |
| Comfortable | 16 | 48-64 GB | 250 GB+ | Adds logging, ODF-lite, virtualization experiments |

The recommended row is our judgement, not a Red Hat figure; revisit it once we see real usage.

Proxmox notes:
- CPU type `host`, machine `q35`, UEFI (OVMF) works well; VirtIO disk and NIC.
- Fast storage matters most: etcd is latency-sensitive. Avoid spinning disk or heavily shared NFS for the root disk.
- Need DNS for `api.<cluster>.<domain>` and `*.apps.<cluster>.<domain>` pointing at the node IP, with a static IP or DHCP reservation.
- Take a Proxmox snapshot after a clean install so labs can be reset quickly.

## 2. OpenShift certifications and a suggested order

Source: [Red Hat certifications](https://www.redhat.com/en/services/certifications), [EX280](https://www.redhat.com/en/services/training/red-hat-certified-openshift-administrator-exam), [skill paths datasheet](https://www.redhat.com/en/resources/openshift-skill-paths-datasheet).

| Exam | Name | Notes |
|---|---|---|
| EX180 | Certified Technologist in OpenShift | Entry level, pairs with DO180 |
| **EX280** | **Certified System Administrator in OpenShift** | **Our target.** Hands-on, performance-based; based on OCP 4.22 per the exam page |
| EX380 | Certified Advanced System Administrator in OpenShift | Follow-up to EX280 (DO380) |
| Specialist exams | EX282 networking, EX316 virtualization, EX336 VM automation, EX370 data foundation, EX430 ACS, EX432 ACM, EX229 ROSA, EX480 multicluster | Take any; optional |

Pathway: EX200 (RHCSA, strongly recommended, not required) -> EX180 (optional) -> **EX280** -> EX380 -> specialists. EX280 + EX380 gives Red Hat Certified Engineer in OpenShift; adding specialist exams leads toward Certified Architect in OpenShift.

**Recommendation:** this project targets **EX280**. Check the current EX280 objectives page before building labs and map them in `docs/ex280-coverage.md` (a backlog item). Note SNO cannot cover everything on the exam (e.g. multi-node scheduling, node maintenance, machine configs behave differently on one node), so we should flag those gaps explicitly.
