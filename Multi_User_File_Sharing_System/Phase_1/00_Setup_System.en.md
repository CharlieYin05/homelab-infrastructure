# System Setup

**2026-07-27**

---

## Hardware Configuration

- CPU: Intel N95
- Memory: SAMSUNG 12 GB LPDDR5 4000 MT/s
- Storage: WD SN740 256 GB
- Ethernet: Realtek RTL8111 × 2
- Wi-Fi: Realtek RTL8822CE

## Operating System

Debian GNU/Linux 13 (trixie)

## Install System Tools

- `samba` — File sharing service
- `acl` — File permission management
- `nftables` — Host firewall and port access control
- `vnstat` — Network traffic statistics
- `curl` — Command-line network tool, used to install Tailscale
- `htop` — Linux process and resource monitor
- `tree` — Display directory structures

## Plan the Filesystem Layout

### Planned Three-Disk Server Layout

```text
256 GB SSD — System Disk
├── Debian
├── Docker / Podman
├── Samba
├── Tailscale
├── nftables
├── /etc                ← All configuration files
├── /var                ← Runtime data, databases, caches and system logs
├── /opt                ← Application files (e.g. Immich Compose files)
└── Other system files

2 TB SSD — Data Disk
└── /srv
    ├── storage         ← Persistent service data
    │   ├── shares
    │   └── docker
    ├── logs            ← Long-term audit, security and network logs
    │   ├── samba
    │   ├── firewall
    │   ├── traffic
    │   └── ids         ← Future
    └── backups         ← Locally generated or temporarily stored backups

2 TB HDD — Cold Backup
├── /srv                ← Full data backup
├── /etc                ← Configuration backup
├── ACL exports
└── Recovery scripts
```

### Current Single-Disk File Server Layout

```text
256 GB SSD
│
├── /
│
├── /etc
├── /opt
├── /var
│
└── /srv
    ├── storage
    │   ├── shares
    │   └── docker
    │
    ├── logs
    │   ├── samba
    │   ├── firewall
    │   └── traffic
    │
    └── backups
```

## Router Configuration

Assign a static IP address to the FSS server.

## Issues Encountered

| Stage | Issue | Cause | Final Solution |
|---|---|---|---|
| **1. Creating the Installation USB** | The USB drive was not detected by the mini PC BIOS | The USB creation method or boot mode (UEFI/Legacy) was incompatible | Used balenaEtcher instead of GNU `dd` |
| **2. USB Peripherals** | The system failed to recognise three USB devices when they were connected at the same time | Unknown | Connecting only two devices unexpectedly resolved the issue |
| **3. Debian Installation** | Accidentally installed the GNOME desktop environment | `Desktop Environment` was selected during installation | Removed GNOME afterwards and kept only SSH and the command-line environment |
| **4. Suspected PATH Issue** | `modprobe` and `modinfo` appeared to be unavailable | `/usr/sbin` is not included in a normal user's default `PATH` | Confirmed this is normal Debian behaviour rather than a system fault |
| **5. Wi-Fi Disconnections** | Wi-Fi consistently disconnected after unplugging the USB keyboard | Unknown; possibly a compatibility issue between the system and wireless adapter | Disabled Wi-Fi using `nmcli radio wifi off` and kept the server on wired Ethernet only |

## Lessons Learned

- Besides avoiding firmware flashing late at night, avoid installing Linux late at night as well. Unexpected problems can easily turn into an all-night troubleshooting session without necessarily being solved.
- Prefer wired networking and do not build a server around the stability of Wi-Fi.
- When troubleshooting, verify the hardware first before blaming the software. Do not immediately assume the operating system is broken.
- Many behaviours that initially look like bugs are simply Debian defaults, such as `/usr/sbin` not being included in a normal user's `PATH`.
