# Homelab OptiPlex 7070 SFF: architecture

*Last updated: 2026-11-06. Status: Proxmox host installed, secured and running headless. Next: prod VM.*

Decisions and their rationale are in [homelab-adr.md](homelab-adr.md). This document describes the **target state** and the **current state**.

> Scope: the OptiPlex only. The network's DNS/DHCP server is an external dependency, out of scope.

Legend: ✅ done or decided · 🟡 proposed, to be confirmed · ⬜ to do

---

## 1. Goals

- Host Jellyfin, Seerr and the \*arr stack, on the local network only (nothing exposed to the Internet).
- Work as in a professional environment: key-based SSH access, prod/test separation, up-to-date documentation.
- Produce a project worth showing on a CV.

---

## 2. Overview (target state)

```
Local network 192.168.1.0/24
        │
        │ Ethernet
┌───────┴──────────────────────────────────────────┐
│ Dell OptiPlex 7070 SFF                           │
│  Proxmox VE 9.2 (x86-64), host pve1.home.arpa    │
│  IP 192.168.1.10                                 │
│                                                  │
│   ┌────────────────────────┐   ┌──────────────┐  │
│   │ Prod VM .20 (Debian)   │   │ Test VM .21  │  │
│   │  Docker                │   │ (later)      │  │
│   │   Jellyfin, Seerr, *arr│   │              │  │
│   │  iGPU passthrough      │   │              │  │
│   └────────────────────────┘   └──────────────┘  │
│                                                  │
│  SSD: system, VMs, configuration                 │
│  HDD: media (to be purchased)                    │
└──────────────────────────────────────────────────┘
```

---

## 3. Hardware

| Item | Value |
|---|---|
| Machine | Dell OptiPlex 7070 SFF (bought second-hand) |
| CPU | Intel Core i5-9500, 3.00 GHz |
| Integrated graphics | Intel UHD Graphics 630 (Quick Sync) |
| RAM | 8 GB DDR4 DIMM 2666 MT/s, 1 module (DIMM1) |
| RAM slots | 4 in total, 3 free, 64 GB maximum supported |
| Current storage | 1 × 512 GB NVMe SSD (`/dev/nvme0n1`, shown as 476 GiB by the installer) |
| Planned storage | 1 HDD for media, not purchased yet |
| BIOS | Dell 1.35.0, up to date as of 2026-11-05 |
| Virtualization (BIOS) | Enabled |
| VT for Direct I/O (BIOS) | Enabled (required for iGPU passthrough) |
| Secure Boot | Enabled, "Deployed" mode |
| SATA Operation (BIOS) | AHCI (was "RAID On": the SSD was invisible to the installer) |
| AC Recovery (BIOS) | Power On: the server restarts by itself after a power outage (verified) |
| Keyboard Errors (BIOS) | Disabled: no POST warning when booting without a keyboard (verified) |
| UEFI boot order | `proxmox` (`\EFI\proxmox\shimx64.efi` on partition 2) first, then onboard NIC network boot (IPv4, IPv6); verified with `efibootmgr -v` |
| Network | Ethernet, IPv4 only |
| Previous OS | Windows, erased by the installation |

---

## 4. Hypervisor

| Parameter | Value | Status |
|---|---|---|
| Product | Proxmox VE 9.2 | ✅ |
| Architecture | x86-64 (ISO "9.2-1", not the arm64 build) | ✅ |
| Boot | UEFI, Secure Boot enabled (verified: `mokutil --sb-state` → enabled) | ✅ |
| Kernel | 7.0.14-28-pve (as of 2026-11-05) | ✅ |
| Filesystem | ext4 with LVM-thin | ✅ |
| Hostname | pve1.home.arpa (verified with `hostname -f`) | ✅ |
| Administration access | Web UI (port 8006, root password) and SSH with ed25519 key only | ✅ |
| Root account | Passphrase stored in a password manager (never in the docs) | ✅ |
| Notification e-mail | A personal address that is actually read | ✅ |

---

## 5. Network

| Parameter | Value | Status |
|---|---|---|
| Network DHCP dynamic range | 192.168.1.100 to 192.168.1.200 | ✅ |
| Gateway (ISP router) | 192.168.1.254 | ✅ |
| Host address | 192.168.1.10/24, configured statically on the host | ✅ |
| Primary DNS | 192.168.1.30 (network DNS server) | ✅ |
| Secondary DNS | 192.168.1.254 (ISP router), independent from the network DNS server | ✅ |
| Management interface | nic0, Intel NIC (e1000e driver) | ✅ |
| Interface naming | Pinning enabled (installer default), name bound to the MAC address | ✅ |
| NIC MAC address | e4:54:e8:6b:0c:01 | ✅ |
| Static DHCP lease | Not used: address configured on the host and outside the DHCP range (see ADR-008) | ✅ |
| Local DNS record | `pve1.home.arpa` → 192.168.1.10, DNS rewrite on the network DNS server | ✅ |

### Addressing plan

Address ranges (see ADR-011):

| Range | Use |
|---|---|
| .1 to .9 | Reserved |
| .10 to .19 | Physical hosts |
| .20 to .29 | Virtual machines (VMID 1xx ↔ address .xx) |
| .30 to .39 | Network services |
| .100 to .200 | DHCP dynamic range (clients) |
| .254 | ISP router (gateway) |

| Address | Name | VMID | Role |
|---|---|---|---|
| 192.168.1.10 | pve1.home.arpa | — | Proxmox host |
| 192.168.1.20 | prod.home.arpa | 120 | Prod VM (media stack) |
| 192.168.1.21 | test.home.arpa | 121 | Test VM (later) |
| 192.168.1.30 | — | — | Network DNS/DHCP server |
| 192.168.1.254 | — | — | ISP router |

Rule: servers (hosts and VMs) have a static address configured on the machine, outside the dynamic range, published by name through local DNS records. Clients use DHCP. Services will be published under their own names (for example `jellyfin.home.arpa`), so that users never depend on a machine name.

### Dependency on the network DNS/DHCP server

- A DHCP outage does not affect the host: its address is configured on the host itself.
- A DNS outage only partly affects the host: the ISP router takes over as secondary DNS.
- VMs are servers too: they get static addresses configured on the machine (ADR-011), so a DHCP outage does not affect them either.

---

## 6. Storage

| Volume | Device | Content | Status |
|---|---|---|---|
| System and VMs | SSD | Proxmox, VM disks, service configuration | ✅ |
| Media | HDD | Jellyfin library and downloads | ⬜ (disk not purchased) |

### SSD layout (Proxmox installer)

All advanced fields left empty: the installer applies its defaults.

| Parameter | Role | Value |
|---|---|---|
| hdsize | SSD space used | 476 GiB (whole disk) |
| swapsize | Swap, RAM overflow on disk | Default (RAM size, between 4 and 8 GiB) |
| maxroot | System volume (Proxmox, ISOs, templates) | Default (about a quarter of the disk, capped) |
| minfree | Free space kept in the LVM volume group | Default (capped at 16 GiB) |
| maxvz | LVM-thin `data` volume, VM disks | Default (the rest) |

Actual values recorded on 2026-11-05 (`lsblk`, `lvs`, `pvesm status`):

| Item | Size | Role |
|---|---|---|
| nvme0n1 | 476.9 GiB | Whole NVMe SSD |
| nvme0n1p1 | 1 MiB | BIOS boot partition (legacy compatibility) |
| nvme0n1p2 | 1 GiB | EFI system partition, mounted on `/boot/efi` (bootloader) |
| nvme0n1p3 | 475 GiB | LVM physical volume, volume group `pve` |
| pve/swap | 7.54 GiB | Swap |
| pve/root | 96 GiB | System, ext4, mounted on `/` (Proxmox storage `local`) |
| pve/data | 348.35 GiB | LVM-thin pool (Proxmox storage `local-lvm`), 3.6 GiB of metadata |
| Free space in `pve` group | about 16 GiB | `minfree` reserve, to be confirmed with `vgs` |

| Proxmox storage | Type | Content | Total | Used on 2026-11-05 |
|---|---|---|---|---|
| local | dir (`/var/lib/vz`) | ISOs, container templates, backups | 93.9 GiB | 6.38 % |
| local-lvm | lvmthin | VM and container disks | 348.35 GiB | 0 % |

Watch point: `local-lvm` is thin-provisioned. Monitor the actual fill rate (`Data%` and `Meta%` in `lvs`), not just the sum of the virtual disk sizes. Enable **Discard** on VM disks so that space freed inside a VM is returned to the pool.

Constraint for the \*arr stack: downloads and the library must be on the **same filesystem** to allow hardlinks. Folder layout: *TBD* (see open questions).

---

## 7. Virtual machines

### Prod VM (see ADR-012)

| Parameter | Value | Status |
|---|---|---|
| VMID / name | 120 / prod | 🟡 |
| OS | Debian 13.7.0 (netinst, amd64), ISO on `local`, SHA-256 verified | ✅ |
| Firmware / machine type | OVMF (UEFI) / q35, with an EFI disk | 🟡 |
| RAM | 4 GiB, fully reserved (required by PCI passthrough) | 🟡 |
| System disk | 64 GiB on `local-lvm`, Discard enabled | 🟡 |
| Media disk (temporary) | 50 GiB on `local-lvm`, Discard enabled, mounted on `/mnt/media`; replaced by the HDD later | 🟡 |
| Network | `vmbr0`, static 192.168.1.20/24, gateway 192.168.1.254 | 🟡 |
| iGPU | Passthrough of 00:02.0 (IOMMU group 0) | ⬜ |

### Services (target state)

| Service | Role | Where | Status |
|---|---|---|---|
| Jellyfin | Media playback, Quick Sync hardware transcoding | Prod VM, Docker | ⬜ |
| Seerr | Media requests | Prod VM, Docker | ⬜ |
| \*arr stack | Library management | Prod VM, Docker | ⬜ |

One Docker container per service, all in the prod VM, deployed with a `docker-compose` file versioned in Git. Media path `/mnt/media` in every container, whatever the underlying disk. Local access only for now; external access for family and friends is planned later (separate decision).

---

## 8. RAM budget ⬜

| Component | Estimated RAM | Source |
|---|---|---|
| Proxmox host | about 1.6 GiB (measured idle, no VM) | `free -h` on 2026-11-05 |
| Prod VM (Debian, Docker, Jellyfin, \*arr, download client) | 4 GiB (estimate, to be checked with `free -h` in the VM) | ADR-012 |
| Test VM | *to estimate* | |
| iGPU (shared memory) | *to estimate* | |
| **Total** | | |

Measurements on 2026-11-05 (`free -h`, host without VMs): 7.5 GiB visible to the OS out of 8 GiB installed (the rest is reserved by the firmware and the iGPU), 1.6 GiB used, 5.9 GiB available (`available`). That leaves about 6 GiB to split between VMs, keeping a margin. Swap unused, `swappiness` at 60 (default).

Possible upgrade: an identical module (DDR4 2666) in one of the 3 free slots.

---

## 9. Pre-installation checklist

- [x] Hardware survey (CPU, RAM, BIOS)
- [x] BIOS up to date
- [x] Virtualization and VT for Direct I/O enabled
- [x] Secure Boot decision
- [x] Nothing to recover from the existing Windows install
- [x] Official Proxmox VE x86-64 ISO downloaded (version 9.2-1)
- [ ] ISO SHA256 checksum verified against the official page (skipped; done for the Debian ISO)
- [x] USB stick ready, UEFI boot successful
- [x] SATA Operation switched to AHCI
- [x] 192.168.1.10 free (confirmed in practice: the host has used it without conflict since installation)
- [x] Network confirmed as /24

---

## 10. Post-installation

- [x] Access the web UI (`https://192.168.1.10:8006`, root account, Linux PAM realm)
- [x] Add a DNS rewrite on the network DNS server: `pve1.home.arpa` → 192.168.1.10, and test it
- [ ] Configure an SMTP relay so Proxmox e-mail alerts are actually delivered
- [x] SSH access with an ed25519 key (key dedicated to the PC, passphrase-protected)
- [x] Disable SSH password authentication (`/etc/ssh/sshd_config.d/10-cle-uniquement.conf`)
- [x] Enterprise repositories disabled, pve-no-subscription repository added (Debian trixie)
- [x] Update the host (Upgrade) and reboot to activate the new kernel
- [x] Check Secure Boot status from Linux
- [x] Check that the IOMMU is active in the kernel (DMAR active, interrupt remapping enabled)
- [x] Confirm devices in IOMMU groups 0, 9 and 10 with `lspci`
- [x] Add the ISP router (192.168.1.254) as the host's secondary DNS
- [x] Documentation versioned in a Git repository
- [x] Headless setup: AC Recovery, Keyboard Errors, UEFI boot order
- [ ] Test AC Recovery: unplug power, plug it back, check that pve1 restarts on its own
- [ ] Test iGPU passthrough
- [x] Download the Debian 13.7.0 netinst ISO to `local`, with SHA-256 verification
- [ ] Create the prod VM (ADR-012), install Debian and Docker
- [ ] Deploy Jellyfin, Seerr and \*arr with a versioned `docker-compose`
- [ ] Create the test VM
- [ ] Set up backups and test a restore

---

## 11. Success criteria

- Jellyfin transcodes a 4K HEVC movie in real time, with hardware transcoding shown in its statistics.
- SSH access by key only.
- The test environment can be rebuilt from versioned files, without manually copying prod.
- Documentation kept up to date, with the reason behind each choice.

---

## 12. Open questions

- Codecs the UHD 630 iGPU handles in hardware (decode and encode): check the Intel ARK page and the Jellyfin documentation.
- Folder layout on the HDD for hardlinks.
- What must stay identical between prod and test, and what must differ (data, secrets, ports).
- Backup strategy: what, where, how often. A backup on the same SSD as what it protects does not survive a disk failure.
- Plan B if iGPU passthrough fails, with a time limit set in advance.
- External access method for family and friends (VPN or public exposure behind a reverse proxy).
- Prod VM CPU: number of cores (vCPUs are shared, not reserved) and CPU type (`x86-64-v2-AES` or `host`).
- Prod VM "Start at boot": should it start automatically when pve1 reboots?

---

## 13. Log

- **2026-11-05**: hardware survey, BIOS check, choice of Proxmox, of a VM with Docker for the media stack, of keeping Secure Boot and of the x86-64 ISO. Installation preparation.
- **2026-11-05, 14:00**: first installer run, "No Hard Disk Found" error. Cause: SATA controller in "RAID On" mode (Intel RST), not supported by the installer's Linux kernel. Fix: SATA Operation set to AHCI in the BIOS (Windows no longer boots, no consequence). Installation restarted with ext4 and the default SSD layout (476 GiB).
- **2026-11-05, 14:37**: installation complete, first access to the web UI. "No valid subscription" message expected (no subscription). Next step: update repositories.
- **2026-11-05, 14:41**: enterprise repositories disabled, pve-no-subscription repository added. `apt-get update` without errors (Debian trixie, trixie-security, trixie-updates, Proxmox no-subscription).
- **2026-11-05, 14:45**: full host upgrade ("Your System is up-to-date"), including a new kernel and the Secure Boot loader (`shim-signed`). Reboot required to activate the kernel.
- **2026-11-05, 14:51**: reboot. Kernel moved from 7.0.2-6-pve to 7.0.14-28-pve. Verified: hostname pve1.home.arpa, Secure Boot enabled.
- **2026-11-05, 14:56**: IOMMU active (`DMAR: Intel(R) Virtualization Technology for Directed I/O`, interrupt remapping enabled). 11 IOMMU groups (0 to 10). Device 00:02.0 (iGPU) is alone in group 0, which suits passthrough. Group 9 holds the chipset devices at 00:1f.x, including the NIC (00:1f.6): never pass it to a VM. Group 10: NVMe SSD (01:00.0).
- **2026-11-05, 15:06**: IOMMU group mapping confirmed with `lspci -nn` (iGPU 00:02.0 in group 0, NIC 00:1f.6 in group 9, NVMe 01:00.0 in group 10).
- **2026-11-05, 15:08**: ISP router added as secondary DNS (`/etc/resolv.conf`: search home.arpa, nameserver 192.168.1.30 then 192.168.1.254). Resolution tested with `getent hosts`.
- **2026-11-05, 15:23**: actual SSD layout recorded (root 96 GiB, swap 7.54 GiB, data pool 348.35 GiB) and Proxmox storages documented.
- **2026-11-05, 15:38**: idle RAM measurement: Proxmox host about 1.6 GiB, 5.9 GiB available for VMs. Swap at 0, `swappiness` 60.
- **2026-11-05, 15:57**: ed25519 SSH key generated on the PC (with passphrase), public key added to `/root/.ssh/authorized_keys`, key login tested successfully. pve1 ED25519 host key fingerprint: `SHA256:0iq8vD4uTFYXwX9q5aHt95VLuthxGBEBxKRx3PrSMRA` (verified server-side with `ssh-keygen -lf`).
- **2026-11-05, 16:05**: SSH password authentication disabled via `/etc/ssh/sshd_config.d/10-cle-uniquement.conf` (`PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin prohibit-password`). Effective configuration checked with `sshd -T`, applied with `systemctl reload ssh`. Tested: login without key refused (`Permission denied (publickey)`), key login accepted.
- **2026-11-05, 16:32**: documentation pushed to a public GitHub repository.
- **2026-11-05, 16:35**: documentation translated to English.
- **2026-11-05, 16:48**: static DHCP lease dropped (redundant with the host's static configuration, the address being outside the DHCP range). Name resolution to be provided by a DNS rewrite on the network DNS server.
- **2026-11-05, 16:52**: DNS rewrite `pve1.home.arpa` → 192.168.1.10 added. Name resolution confirmed by SSH (`pve1.home.arpa (192.168.1.10)`), same host key fingerprint as under the IP address.
- **2026-11-05, 17:21**: address ranges defined (ADR-011) and prod VM specification decided (ADR-012): VMID 120, 192.168.1.20, OVMF/q35, 4 GiB RAM, 64 GiB system disk, temporary 50 GiB media disk. One VM for all services, one container per service.
- **2026-11-05, 17:46**: Debian 13.7.0 netinst ISO (amd64) downloaded to `local` from cdimage.debian.org via "Download from URL", SHA-256 checksum verified by Proxmox. Prod VM creation postponed to the next session.
- **2026-11-05, 20:03**: BIOS AC Recovery set to Power On, so pve1 restarts automatically after a power outage. Keyboard Errors disabled. Preparation for headless operation (no screen or keyboard).
- **2026-11-05, 20:11**: stale "Windows Boot Manager" UEFI entry removed (left over in the motherboard NVRAM after Windows was wiped). "Onboard NIC (IPV6)" network boot entry deleted by mistake, then back after exiting the BIOS without saving. AC Recovery and Keyboard Errors settings to be re-checked for the same reason.
- **2026-11-05, 20:14**: AC Recovery (Power On) and Keyboard Errors (disabled) re-checked in the BIOS. `efibootmgr -v`: BootOrder 0004 (proxmox, signed shim on the EFI partition), 0001 and 0005 (onboard NIC network boot); no Windows entry left. Ready for headless operation.
