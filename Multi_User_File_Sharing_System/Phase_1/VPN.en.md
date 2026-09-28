# Set Up VPN

**2026-08-09**

---

## Objectives

- Allow both administrators and file users to remotely access the FSS server.
- Redesign the existing Tailnet policy to support multiple users and improve security.
- Implement Zero Trust within the Tailnet by using explicit Grants only, following the Principle of Least Privilege.

---

## Architecture

```text
                         ┌─────────────────────┐
                         │   cy-server-fss     │
                         │        FSS          │
                         │     tag:fss         │
                         │  192.168.XX.XX      │
                         │  100.105.xxx.xxx    │
                         └─────────┬───────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                 TCP/445       TCP/22        TCP/22
                   SMB           SSH            SSH
                    │             │             │
          ┌─────────▼──────┐ ┌────▼───────┐ ┌──▼───────────────┐
          │ group:file-user│ │ group:admin│ │     cy-server    │
          │   File Users   │ │   Admins   │ │ Network Hub      │
          └─────────┬──────┘ └────┬───────┘ │ 192.168.XX.XX   │
                    │             │         │ 100.65.xxx.xxx   │
                    │             │         │ Subnet Router    │
                    │             │         └────────┬─────────┘
                    │             │                  │
                    └─────────────┼──────────────────┘
                                  │
                         ┌────────▼─────────┐
                         │     Tailnet      │
                         │ Tailscale Grants│
                         │ Least Privilege │
                         └────────┬─────────┘
                                  │
             ┌────────────────────┼──────────────────────┐
             │                    │                      │
             ▼                    ▼                      ▼
         tag:fss          tag:network-hub         192.168.50.0/24
      FSS Services        Network Hub Services          Home LAN
             │                    │                      │
       ┌─────┴─────┐       ┌──────┴──────┐        ┌─────┴──────┐
       │           │       │             │        │            │
     :445        :22    SSH :22      Hosted      Router      Other
     SMB         SSH                 Services   192.168.50.1 Devices
       │           │
file-user ✅   file-user ❌                      file-user ❌
admin     ✅   admin     ✅                      admin     ✅
```

## Tailnet Policy

```text
group:file-user
    └──→ tag:fss
          └── TCP/445      SMB            ✅
    └──→ host:home-router
          └── TCP/UDP 53   DNS            ✅


group:admin
    ├──→ tag:fss
    │     ├── TCP/445      SMB            ✅
    │     └── TCP/22       SSH            ✅
    │
    ├──→ tag:network-hub
    │     ├── TCP/22       SSH            ✅
    │     └── TCP/XXX      Hosted Service ✅
    │
    └──→ host:home-router
          ├── TCP/80       Router Web     ✅
          └── TCP/UDP 53   DNS            ✅


All other Tailnet traffic not explicitly authorised
    └──→ DROP                             ❌
```

---

## Process

### 1. Add the Tailscale Repository

Create the keyring directory:

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
```

Download the official repository signing key:

```bash
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg \
  | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
```

Add the Trixie stable repository:

```bash
curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.tailscale-keyring.list \
  | sudo tee /etc/apt/sources.list.d/tailscale.list
```

### 2. Install Tailscale

```bash
sudo apt update
sudo apt install tailscale
```

Check whether the Tailscale network interface is present:

```bash
ip -br addr
```

Check Tailscale service status:

```bash
systemctl status tailscaled --no-pager
```

### 3. Join the FSS Server to the Tailnet

Log in:

```bash
sudo tailscale up
```

### 4. Configure Tailnet Rules (Zero Trust)

#### 4.1 Create the `admin` Group

#### 4.2 Create the `file-user` Group

#### 4.3 Create a Dedicated Tag for `cy-server-fss` and Assign It Manually

Create tag:

```text
tag:fss
```

Tag owner:

```text
group:admin
```

#### 4.4 Create a Dedicated Tag for `cy-server` and Assign It Manually

#### 4.5 Write Grants Based on the Tailnet Policy Above

#### 4.6 Invite Friends to Join the Tailnet as `file-user`

#### 4.7 Manually Add Friends to the `file-user` Group

#### 4.8 Ask the File User to Test the Services

```bash
# SMB — should succeed
nc -vz 100.105.XXX.XXX 445

# SSH — should time out
nc -vz -w 5 100.105.XXX.XXX 22

# Internal LAN services — should time out
nc -vz -w 5 100.65.XXX.XXX 22
nc -vz -w 5 100.65.XXX.XXX 8787
nc -vz -w 5 100.65.XXX.XXX 8000
nc -vz -w 5 100.65.XXX.XXX 4430
```

#### 4.9 Remove the Default Global Allow Rule After Testing Passes

### 5. Make FSS Accessible Through `fss.cy-server.com`

#### 5.1 Configure LAN DNS on the Router

One service maps to one IP address.

Edit the LAN DNS configuration:

```bash
vi /jffs/configs/dnsmasq.conf.add
```

Replace the contents with:

```text
# ==================================================
# Homelab Internal DNS
# ==================================================

# cy-server
host-record=cy-server.com,192.168.XXX.XXX
host-record=portainer.cy-server.com,192.168.XXX.XXX
host-record=kvm.cy-server.com,192.168.XXX.XXX
host-record=clipcascade.cy-server.com,192.168.XXX.XXX
host-record=npm.cy-server.com,192.168.XXX.XXX

# File Sharing Server
host-record=fss.cy-server.com,192.168.XXX.XXX
```

Restart DNS after saving:

```bash
service restart_dnsmasq
```

#### `cy-server.com` DNS Architecture

```text
                    cy-server.com
                         │
            ┌────────────┴────────────┐
            │                         │
      Private Services           Public Services
     LAN / Tailscale               Internet
            │                         │
     Router dnsmasq             Cloudflare DNS
            │                         │
    ├─ fss → .XX                └─ travelblog → OCI
    ├─ portainer → .XX
    ├─ kvm → .XX
    ├─ npm → .XX
    └─ clipcascade → .XX
```

---

## Issue 1

### Symptoms

After completing Step 4.9, enabling Tailscale on the Mac caused:

- Tailnet devices to remain reachable.
- SMB and other Tailscale services to continue working.
- Most normal Internet websites to stop loading.
- Internet access to recover immediately when Tailscale was disabled.
- The Tailscale GUI to report:

```text
DNS Unavailable

Tailscale can't reach the configured DNS servers.
Code: dns-forward-failing
```

At first, this looked like "the Internet breaks when Tailscale is enabled".

Further investigation showed that the actual problem was DNS resolution rather than Internet connectivity itself.

### Troubleshooting Process

#### 1. Basic Network Check

With Tailscale enabled:

```bash
ping -c 3 192.168.50.1
```

Succeeded:

```text
3 packets transmitted, 3 packets received
```

This confirmed that:

```text
Mac → Home Router
```

was working normally.

Then test a public IP address:

```bash
ping -c 3 1.1.1.1
```

This also succeeded.

However:

```bash
nslookup google.com
```

failed.

Initial diagnosis:

```text
DNS failure
```

#### 2. Check Tailscale DNS

Check DNS status:

```bash
tailscale dns status
```

Output:

```text
Tailscale DNS: enabled.
```

DNS configuration:

```text
MagicDNS: enabled

Resolvers:
(no resolvers configured, system default will be used)

Split DNS Routes:
cy-server.com -> 192.168.50.1         ← Internal services

System DNS:
192.168.50.1                          ← Normal public DNS also ultimately
                                        depends on the router
```

Test the Tailscale internal resolver:

```bash
tailscale dns query google.com
```

Failed:

```text
failed to query DNS:
waiting for response or error from [192.168.50.1]:
context deadline exceeded
```

Initial suspected failure path:

```text
Application
    ↓
macOS DNS
    ↓
Tailscale DNS
    ↓
192.168.50.1
    ↓
TIMEOUT
```

### 3. Verify Whether Router DNS Works

Bypass Tailscale DNS directly:

```bash
dig @192.168.50.1 google.com

dig +tcp @192.168.50.1 google.com
```

Both succeeded.

This proved that the DNS service on:

```text
192.168.50.1
```

was working correctly.

The problem was narrowed further toward Tailscale DNS.

### 4. Temporarily Disable Tailscale DNS

Disable Tailscale DNS:

```bash
tailscale set --accept-dns=false
```

Afterwards, Internet tests worked normally:

```bash
dscacheutil -q host -a name google.com

ping google.com
```

Tailscale services also continued to work:

```text
nc -vz 100.105.169.81 445
→ Connection ... port 445 succeeded!
```

### 5. Investigate the Tailscale Client Version

The terminal was using:

```bash
which tailscale
```

Output:

```text
/opt/homebrew/bin/tailscale
```

This was the Homebrew installation:

```text
Tailscale 1.92.3
```

At the same time, the Mac also had:

```text
/Applications/Tailscale.app
```

This meant the CLI and backend could potentially come from different builds:

```text
CLI:
1.92.3-t9a08e8f1c

Backend:
1.92.3-ta17f36b9b-ga4dc88aac
```

After uninstalling the Homebrew version and upgrading the Tailscale app, this still failed:

```bash
tailscale dns query google.com
```

Therefore, the CLI/backend mismatch was indeed a configuration issue, but it was not the root cause of this DNS failure.

### 6. Discover a Problem With the Subnet Route

`cy-server` is a Tailscale Subnet Router advertising:

```text
192.168.50.0/24
```

The Mac was currently connected to the Home LAN using the same subnet:

```text
Local LAN
192.168.50.0/24
       ↑
       │ overlap
       ↓
Tailscale advertised route
192.168.50.0/24
```

Perform an A/B test on Subnet Route acceptance.

#### Disable Subnet Route Acceptance

```bash
tailscale set --accept-dns=true

tailscale set --accept-routes=false
```

Run:

```bash
tailscale dns query google.com
```

again.

It immediately succeeded.

This directly confirmed that the failure was related to accepting the Tailscale-advertised:

```text
192.168.50.0/24
```

Subnet Route.

### 7. Suspect Tailscale Access Control

The system had previously worked correctly until Zero Trust was implemented.

The old Allow All policy had just been removed and replaced with explicit Grants.

This suggested that an infrastructure dependency might have been missed:

```text
Router DNS
192.168.50.1:53
```

Perform another A/B test.

Re-enable Subnet Route and DNS acceptance:

```bash
tailscale set --accept-routes=true
tailscale set --accept-dns=true
```

Test A:

```bash
tailscale dns query google.com
```

hangs.

Test B:

Add the following outbound rule for Admin:

```json
{
    "src": ["group:admin"],
    "dst": ["192.168.50.1"],
    "ip": [
        "udp:53",
        "tcp:53"
    ]
}
```

After adding it:

```bash
tailscale dns query google.com
```

succeeded immediately, and normal websites became accessible again.

### 8. Root Cause

```text
Mac
 │
 │ accept-dns=true
 │ accept-routes=true
 ▼
Tailscale DNS Forwarder
 │
 │ DNS resolver = 192.168.50.1
 ▼
Tailscale Subnet Route
192.168.50.0/24
 │
 ▼
Tailnet Access Control
 │
 │ Missing UDP/TCP 53 Grant
 ▼
DNS Denied
```

The problem did not exist before because the old policy contained:

```json
{
    "src": ["*"],
    "dst": ["*"],
    "ip": ["*"]
}
```

After removing Allow All and implementing Least Privilege, I only considered services explicitly accessed by users:

```text
Router HTTP :80
```

but missed:

```text
Router DNS UDP :53
Router DNS TCP :53
```

As a result, the Tailscale DNS Forwarder could no longer reach the DNS server.

## 9. Lessons Learned

When moving from Allow All to Least Privilege, it is not enough to inventory only the application services that users access directly.

The infrastructure dependencies behind those services must also be identified.

For example:

```text
Services explicitly used by users:
├── SMB 445
├── SSH 22
├── NPM 443
└── Router HTTP 80

Infrastructure dependencies that are easy to overlook:
├── DNS 53       ← Failure in this incident
├── DHCP
├── NTP
├── ICMP
└── Other internal resolution / authentication services
```

---

### FSS Access Paths

#### Administrator

- SSH: User Device → Tailscale MagicDNS → FSS
- SSH: User Device → Tailscale Split DNS → Router DNS → FSS
- SSH: User Device → `cy-server` → FSS
- SMB: User Device → Tailscale MagicDNS → FSS
- SMB: User Device → Tailscale Split DNS → Router DNS → FSS

#### File User

- SMB: User Device → Tailscale MagicDNS → FSS
- SMB: User Device → Tailscale Split DNS → Router DNS → FSS

---

## Final Grants Configuration

```jsonc
{
    "grants": [
        // Administrators can access FSS through either the Tailscale IP
        // or fss.cy-server.com using SSH and SMB
        {
            "src": ["group:admin"],
            "dst": ["tag:fss"],
            "ip":  ["tcp:22", "tcp:445"],
        },

        // File users can access the FSS SMB service through either
        // the Tailscale IP or fss.cy-server.com
        {
            "src": ["group:file-user"],
            "dst": ["tag:fss"],
            "ip":  ["tcp:445"],
        },

        // Administrators can access services hosted on cy-server through
        // either its Tailscale IP or *.cy-server.com
        {
            "src": ["group:admin"],
            "dst": ["tag:network-hub", "host:cy-server-lan"],
            "ip":  ["tcp:22", "tcp:443"],
        },

        // Administrators can access the home router
        {
            "src": ["group:admin"],
            "dst": ["host:home-router"],
            "ip":  ["tcp:80"],
        },

        // Allow administrators and file users to use the router DNS service
        // to resolve *.cy-server.com
        {
            "src": ["group:admin", "group:file-user"],
            "dst": ["host:home-router"],
            "ip":  ["tcp:53", "udp:53"],
        },
    ],

    "ssh": [
        {
            "action": "check",
            "src":    ["autogroup:member"],
            "dst":    ["autogroup:self"],
            "users":  ["autogroup:nonroot", "root"],
        },
    ],

    "groups": {
        "group:admin": ["admin@XXX.com"],
        "group:file-user": ["user0@XXX.com"],
    },

    "tagOwners": {
        // Locate FSS through its native Tailnet IP
        "tag:fss": ["group:admin"],

        // Locate cy-server through its native Tailnet IP
        "tag:network-hub": ["group:admin"],
    },

    "hosts": {
        // Reach the home router through the Subnet Router
        "home-router": "192.168.50.1",

        // Reach the LAN IP of cy-server through the Subnet Router
        "cy-server-lan": "192.168.XXX.XXX",

        // Reach the LAN IP of FSS through the Subnet Router
        "fss-lan": "192.168.XXX.XXX",
    },
}
```

---

## Common Tailscale Operations

View device status in the Tailnet:

```bash
tailscale status
```

Test latency and check whether traffic is being relayed:

```bash
tailscale ping <PEER_IP>
```

---

## Notes

1. When a new user joins the Tailnet, remember to manually add them to the appropriate user group under Definitions.
2. If a service requires special access, create an explicit rule for it. Remember that access rules are directional.
3. When writing rules, usually define:
   - **Source:** Which Group
   - **Destination:** Which Tag / Host
   - **Protocol:** `TCP:<PORT>` or `UDP:<PORT>`
