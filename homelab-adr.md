# Homelab OptiPlex: Architecture Decision Records (ADR)

*Last updated: 2026-11-05.*

An ADR (Architecture Decision Record) explains **why** a choice was made. Records are numbered and never deleted: when a decision changes, a new record supersedes the old one, which is marked "superseded". The architecture itself is described in [homelab-architecture.md](homelab-architecture.md).

## Index

| No. | Decision | Status |
|---|---|---|
| ADR-001 | Proxmox VE 9.2 hypervisor, x86-64 build | Accepted |
| ADR-002 | Media stack in a VM with Docker | Accepted |
| ADR-003 | Keep Secure Boot enabled | Accepted |
| ADR-004 | Start with 8 GB of RAM | Accepted |
| ADR-005 | ext4 filesystem with LVM-thin | Accepted |
| ADR-006 | Hostname pve1.home.arpa | Accepted |
| ADR-007 | SSD for the system, HDD for media | Accepted |
| ADR-008 | Host network addressing | Accepted |
| ADR-009 | SATA controller in AHCI mode | Accepted |
| ADR-010 | SSH access with an ed25519 key | Accepted |
| ADR-011 | Address ranges and server addressing | Accepted |
| ADR-012 | Prod VM specification | Accepted |

## Template

```
### ADR-NNN: title
- Status: proposed / accepted / superseded by ADR-XXX
- Context: the problem or constraint
- Decision: what is chosen
- Alternatives considered: what else was evaluated, and why not
- Consequences: what it brings, what it costs
- Reversibility: easy / hard
```

---

### ADR-001: Proxmox VE 9.2 hypervisor, x86-64 build

- **Status**: accepted.
- **Context**: the goal is to separate environments (prod, test), have snapshots and learn virtualization as in a company, on a single machine with an Intel x86-64 CPU.
- **Decision**: install Proxmox VE 9.2 in place of Windows, from the official x86-64 ISO (version 9.2-1).
- **Alternatives considered**:
  - Debian or Ubuntu Server with Docker directly: simpler and lighter, and the iGPU works without passthrough, but does not meet the goals of separating prod and test or learning virtualization.
  - Debian with KVM/libvirt, XCP-ng, TrueNAS, Unraid, Incus: not chosen; Proxmox is the accessible open-source reference for a homelab, with a web UI, VMs, containers, snapshots and backups.
  - Proxmox VE arm64 ISO: officially released in August 2026, but built for ARM processors; it does not boot on an i5-9500.
- **Consequences**: an extra layer of complexity (iGPU passthrough) and host RAM usage on an 8 GB machine. Without a subscription, the default repositories are the paid ones: reconfigured after installation.
- **Reversibility**: hard once VMs exist, but deploying with `docker-compose` keeps the stack portable to another platform.

---

### ADR-002: Media stack in a VM with Docker

- **Status**: accepted.
- **Context**: Jellyfin, Seerr and the \*arr stack are deployed with Docker. Hardware transcoding on the iGPU (Quick Sync) is a requirement. An LXC container would share the iGPU more easily, since it shares the host kernel, but offers weaker isolation.
- **Decision**: a Debian VM running Docker, deployed with a `docker-compose` file versioned in Git.
- **Alternatives considered**:
  - Docker in an LXC container: simpler iGPU access and lighter on resources, but weaker isolation and dependency on the host kernel. As far as we know, the Proxmox documentation advises against this combination: *to be checked and cited before relying on it.*
  - Docker directly on the Proxmox host: mixes the hypervisor and the applications.
  - Jellyfin alone in an LXC and the rest in a VM: middle ground, more complex for sharing files between the two.
- **Consequences**: the iGPU must be passed exclusively to the VM (PCI passthrough). This requires VT-d (enabled in the BIOS) and can be finicky with an Intel iGPU. The iGPU is then no longer available to the host.
- **Plan B**: if passthrough fails within a time limit set in advance, move the stack to an LXC with the same `docker-compose` file, or to plain Debian with Docker. *Time limit to be set.*
- **Reversibility**: easy, thanks to `docker-compose`.

---

### ADR-003: Keep Secure Boot enabled

- **Status**: accepted.
- **Context**: Secure Boot is enabled in "Deployed" mode. Proxmox must boot, and the kernel modules needed for passthrough must be accepted.
- **Decision**: keep Secure Boot enabled and install from the official ISO.
- **Rationale (from the Proxmox documentation)**:
  - since Proxmox VE 8.1, the bootloader is signed with Microsoft's Secure Boot certificate, so the official ISO boots as is;
  - since kernel 6.2.16-8, the modules shipped with the kernel are signed and trusted by it.
- **Alternatives considered**: disabling Secure Boot in the BIOS (simpler, but loses boot protection for no reason).
- **Consequences**: unsigned third-party modules would be rejected. None are planned. Verified after installation with `mokutil --sb-state`.
- **Reversibility**: easy, Secure Boot can be disabled in the BIOS. If that happens, record the reason in the architecture log.

---

### ADR-004: Start with 8 GB of RAM

- **Status**: accepted.
- **Context**: one 8 GB DDR4 module, 3 free slots, 64 GB maximum supported.
- **Decision**: start with the current RAM, without buying anything first.
- **Alternatives considered**: buying a module before starting (spending before measuring the need).
- **Consequences**: a permanently running test VM and ZFS (RAM cache) are not advisable in this configuration. Measured on 2026-11-05: the idle host uses about 1.6 GiB, leaving about 6 GiB for VMs.
- **Reversibility**: easy. Upgrade possible with an identical module (DDR4 2666). Dual-channel also benefits the iGPU; check the recommended slot in the Dell manual.

---

### ADR-005: ext4 filesystem with LVM-thin

- **Status**: accepted (applied at installation on 2026-11-05).
- **Context**: a single 476 GiB SSD, 8 GB of RAM.
- **Decision**: ext4, the installer default, which creates an LVM-thin pool for VM disks. Disk layout (hdsize, swapsize, maxroot, minfree, maxvz) left at the installer defaults.
- **Alternatives considered**:
  - ZFS: on a single disk, it detects data corruption but cannot repair it without redundancy, and its cache uses RAM by default.
  - A manual disk layout: no identified need to deviate from the defaults.
- **Consequences**: no data self-healing. VM snapshots remain available through LVM-thin. Swap roughly the size of RAM, useful with 8 GB to avoid a VM being killed when memory runs out, at the cost of SSD wear if heavily used.
- **Reversibility**: hard, changing it requires reinstalling.

---

### ADR-006: Hostname pve1.home.arpa

- **Status**: accepted (verified with `hostname -f`).
- **Context**: the hostname appears in the web UI, SSH prompts, DHCP leases and configuration files. Renaming a node after installation is tricky.
- **Decision**: descriptive name `pve1`, domain `home.arpa`, the suffix reserved for home networks (RFC 8375).
- **Alternatives considered**: a themed name (less meaningful with several machines), the owner's first name (a server is not a person, and it ages badly in documentation), `.local` (reserved for mDNS, a source of bugs).
- **Reversibility**: hard.

---

### ADR-007: SSD for the system, HDD for media

- **Status**: accepted.
- **Context**: the system, the services and their databases are small and accessed randomly. Media files are large and read sequentially.
- **Decision**: system, VMs and configuration on the SSD; media on the HDD, to be purchased.
- **Consequences**:
  - downloads and the media library must be on the same filesystem to allow hardlinks in the \*arr stack;
  - the media path must remain a configurable parameter, since the HDD is not there yet;
  - separating devices addresses performance and failure domains, not "write conflicts" between VMs.
- **Reversibility**: medium.

---

### ADR-008: Host network addressing

- **Status**: accepted (amended on 2026-11-05: static DHCP lease dropped, local DNS record added).
- **Context**: the network DHCP server hands out 192.168.1.100 to 192.168.1.200. The Proxmox installer expects a static configuration. The network DNS/DHCP server is a single point of failure.
- **Decision**: address 192.168.1.10/24 configured statically on the host, outside the dynamic range; gateway 192.168.1.254; primary DNS the network DNS server, secondary DNS the ISP router (192.168.1.254). The name `pve1.home.arpa` is published through a DNS rewrite on the network DNS server.
- **Rationale**: an address configured on the host keeps it reachable even if DHCP fails; choosing it outside the dynamic range means the DHCP server can never hand it to another device; a secondary DNS independent from the network DNS server keeps name resolution working during an outage. A statically configured host does not register itself in DNS, hence the explicit DNS record.
- **Alternatives considered**:
  - a static DHCP lease in addition to the host configuration: redundant (no conflict risk since the address is outside the range), and duplicating the address in two places risks them drifting apart;
  - a DHCP reservation only, without static configuration: the host would depend on the DHCP server at boot, which the Proxmox installer does not expect.
- **Consequences**: when the ISP router answers as secondary, the filtering DNS is bypassed for that query, and `pve1.home.arpa` does not resolve through it. Acceptable for a server. Changing the host address means updating the host configuration and the DNS record.
- **Reversibility**: easy.

---

### ADR-009: SATA controller in AHCI mode

- **Status**: accepted.
- **Context**: on the first run, the Proxmox installer reported "No Hard Disk Found". The BIOS had SATA Operation set to "RAID On" (Intel RST), a mode Windows supports through its driver but the installer's Linux kernel does not.
- **Decision**: set SATA Operation to AHCI.
- **Alternatives considered**: keeping "RAID On" (the SSD stays invisible to Linux, installation impossible).
- **Consequences**: the installer detects the SSD. Windows no longer boots with this setting, which does not matter since the disk was wiped. No hardware RAID is used, so nothing is lost.
- **Reversibility**: easy on the BIOS side, but must not be changed once Proxmox is installed.

---

### ADR-010: SSH access with an ed25519 key

- **Status**: accepted. SSH password authentication disabled on 2026-11-05 (file `/etc/ssh/sshd_config.d/10-cle-uniquement.conf`). The web UI remains protected by the root password, handled by a separate service (`pveproxy`).
- **Context**: pve1 is administered remotely. A password is sent to the server at every login and can be guessed through repeated attempts.
- **Decision**: authentication with an ed25519 key pair, generated on the PC and dedicated to that device, protected by a passphrase. Only the public key is placed on the server.
- **Alternatives considered**:
  - keeping password login: the secret is sent to the server and exposed to brute force;
  - an RSA key: works, but ed25519 is more modern, faster and uses shorter keys;
  - a key without a passphrase: stealing the private key file would be enough to access the server.
- **Consequences**: the passphrase is requested at every connection (an SSH agent can cache it for the session). Losing the PC means removing its line from `authorized_keys`, identifiable by the key comment.
- **Reversibility**: easy.

---

### ADR-011: Address ranges and server addressing

- **Status**: accepted.
- **Context**: the network will host several kinds of machines (physical hosts, VMs, network services, clients). A flat sequence of addresses mixes them and breaks as soon as a new physical host or VM is added. ADR-008 set the rule for pve1; VMs need one too.
- **Decision**:
  - ranges by category: .10–.19 physical hosts, .20–.29 VMs, .30–.39 network services, .100–.200 DHCP dynamic range, .254 ISP router;
  - VMID mirrors the address (VMID 120 ↔ .20);
  - servers, VMs included, follow ADR-008: static address configured on the machine, outside the DHCP range, published through a local DNS record. Only clients use DHCP.
- **Rationale**: the criterion is the role, not physical versus virtual. A client fetches services and can change address; a server is reached by others and must stay reachable, including when the DHCP server is down. The prod VM hosts Jellyfin, so it is a server.
- **Alternatives considered**:
  - DHCP reservations for VMs: centralised, and the DNS server can learn their names automatically, but each VM would depend on the DHCP server at boot;
  - sequential addresses after the host (.11, .12…): simple, but no room for a second physical host without mixing categories.
- **Consequences**: each new server needs a manual DNS record. The VM range holds 10 addresses, enough for this homelab.
- **Reversibility**: easy, but changing an address means updating the machine, the DNS record and the documentation.

---

### ADR-012: Prod VM specification

- **Status**: accepted (VM not created yet).
- **Context**: the media stack (Jellyfin, Seerr, \*arr, download client) runs in one VM with Docker (ADR-002), with iGPU passthrough for hardware transcoding. About 6 GiB of RAM are available for VMs. The media HDD has not been purchased yet, but a few test movies (under 10 GiB each) should be playable before it arrives.
- **Decision**:
  - VMID 120, name `prod`, address 192.168.1.20 (ADR-011);
  - Debian, OVMF (UEFI) firmware with an EFI disk, q35 machine type;
  - 4 GiB of RAM;
  - 64 GiB system disk and a temporary 50 GiB media disk, both on `local-lvm` with Discard enabled; the media disk is mounted on `/mnt/media`, the path every container will use;
  - one Docker container per service.
- **Alternatives considered**:
  - SeaBIOS and the i440fx machine type: older emulated hardware, less suited to GPU passthrough than OVMF and q35 (PCIe);
  - one VM per service: stronger isolation, but each VM runs its own kernel and OS, unaffordable with 8 GB of RAM;
  - storing test movies on the system disk: mixes media and system (contrary to ADR-007) and would require a different media path later;
  - a larger temporary media disk: unnecessary for movies under 10 GiB; growing a disk later is easy.
- **Consequences**:
  - with PCI passthrough, the VM's 4 GiB are reserved and locked in RAM as soon as it starts, used or not;
  - all services share the VM: if it goes down, they all go down;
  - name `prod` describes the environment, not the function; acceptable with two VMs, ambiguous if another production VM is ever added. Users will reach services through service names (`jellyfin.home.arpa`), so renaming the VM would not affect them.
- **Reversibility**: RAM and disk growth are easy; firmware and machine type are hard to change after the OS is installed.
