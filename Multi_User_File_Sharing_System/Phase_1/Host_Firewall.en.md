# Configure the FSS Host Firewall

**2026-08-01**

---

## Objective

Implement the host-level protection layer of **Defense in Depth**.

Even if the VPN access-control rules fail, unauthorized users should still be unable to access unauthorized ports on the FSS server, providing an additional layer of authorization.

The intended result is approximately:

```text
FSS Host Firewall

LAN 192.168.50.0/24
        │
        ├── SMB 445 ────────── Allow as required
        └── Other ports ────── Deny by default

Tailscale / tailscale0
        │
        ├── SMB 445 ────────── Allow
        └── Other traffic ──── Restricted

cy-server 192.168.50.13
        │
        ├── SMB 445 ────────── Allow
        └── SSH 22 ─────────── Allow

Established / Related ───────── Allow
Loopback ────────────────────── Allow

All other unauthorized inbound traffic ── DROP
```

---

## Preparation

In the previous chapter, FSS could be accessed through two paths:

- Native Tailnet path
- Subnet Router path

Therefore, the traffic path needed to be observed first.

### 1. Install a Packet Capture Tool

Install `tcpdump`:

```bash
sudo apt update
sudo apt install tcpdump

tcpdump --version
```

### 2. Capture SSH and SMB TCP Traffic on FSS

Run on FSS:

```bash
sudo tcpdump -ni enp3s0 'tcp port 445 or tcp port 22'
```

Then run the following on the Mac:

```bash
nc -vz 100.105.XXX.XXX 445
nc -vz fss.cy-server.com 445
nc -vz fss.cy-server.com 22
```

#### The Output Shows the Following

##### Direct Access Through the Tailscale IP

```text
Mac
 ↓
Tailscale
 ↓
tailscale0
 ↓
FSS
```

FSS can see that the traffic enters through:

```text
iifname = tailscale0
```

##### Access Through `fss.cy-server.com`

```text
Mac / File User
        ↓
Tailscale Grants
        ↓
cy-server
        ↓ SNAT
192.168.50.13
        ↓
enp3s0
        ↓
FSS
```

From the perspective of FSS `nftables`, the traffic only appears as:

```text
iifname = enp3s0
ip saddr = 192.168.50.13
```

This means that after passing through the Subnet Router, different users appear to FSS as the same LAN IP address.

#### Ideal `file-user` Traffic Path

```text
             User
               │
        fss.cy-server.com
               │
               ▼
   100.105.XXX.XXX (FSS Tailscale IP)
               │
           Tailscale
               │
        Tailscale Grants
               │
          tailscale0
               │
           nftables
               │
          Samba :445
```

### 3. Modify the Architecture

#### 3.1 Modify Router LAN DNS

Change only:

```text
fss.cy-server.com
```

from the local LAN IP to the FSS Tailscale IP.

#### 3.2 Remove the Old FSS Subnet Router Path

Change:

```json
{
    "src": ["group:admin"],
    "dst": ["tag:fss", "host:fss-lan"],
    "ip":  ["tcp:22", "tcp:445"]
},
{
    "src": ["group:file-user"],
    "dst": ["tag:fss", "host:fss-lan"],
    "ip":  ["tcp:445"]
}
```

to:

```json
{
    "src": ["group:admin"],
    "dst": ["tag:fss"],
    "ip":  ["tcp:22", "tcp:445"]
},
{
    "src": ["group:file-user"],
    "dst": ["tag:fss"],
    "ip":  ["tcp:445"]
}
```

#### New Architecture

FSS service traffic is now unified so that it enters through `tailscale0`.

This also benefits the traffic-monitoring work planned for the next stage.

```text
Internet / LAN
     │
   enp3s0
     │
     ├── Tailscale UDP transport → ALLOW
     ├── SSH 22                  → DROP
     ├── SMB 445                 → DROP
     ├── NetBIOS 137/138/139     → DROP
     └── Other inbound traffic   → DROP by default


Tailnet
     │
 tailscale0
     │
     ├── SSH 22  → ALLOW
     ├── SMB 445 → ALLOW
     └── Other   → DROP by default
```

---

## Design the INPUT Processing Order

### INPUT Chain

```text
INPUT
 │
 ├─ 1. Loopback
 │      └─ ACCEPT
 │
 ├─ 2. Established / Related
 │      └─ ACCEPT
 │
 ├─ 3. Invalid
 │      └─ DROP
 │
 ├─ 4. Required ICMP / ICMPv6
 │      └─ ACCEPT
 │
 ├─ 5. Tailscale UDP on enp3s0
 │      └─ ACCEPT
 │
 ├─ 6. tailscale0
 │      ├─ TCP 22  → ACCEPT
 │      └─ TCP 445 → ACCEPT
 │
 ├─ 7. enp3s0 — cy-server
 │      └─ Source: 192.168.XX.XX
 │          TCP 22 → ACCEPT
 │
 └─ 8. Everything Else
        └─ DROP
```

Explanation:

```text
1. Loopback
   → Allow local host communication.

2. Established / Related
   → Allow inbound packets belonging to existing or related connections.

3. Invalid
   → Immediately drop packets that conntrack cannot classify properly.

4. Required ICMP / ICMPv6
   → Initially allow required ICMP traffic such as ping.

5. Tailscale UDP on enp3s0
   → Allow Tailscale to establish its own tunnel transport.

6. tailscale0
   → Allow only the required FSS services through the Tailscale interface.

7. enp3s0 — cy-server
   → Allow SSH on enp3s0 only when the source is cy-server.

8. Everything Else
   → Drop anything that does not match an explicit rule.
```

### INPUT Chain Logical Architecture

```text
                      INPUT
                        │
             ┌──────────┴───────────┐
             │                      │
          Allowed?               No Match
             │                      │
             ▼                      ▼
           ACCEPT                  DROP
             │
      ┌──────┼─────────┐
      │      │         │
     lo   Established  Required
          Connections  Network Protocols
                     │
              ┌──────┴───────┐
              │              │
       Tailscale Underlay  tailscale0
                             │
                         ┌───┴───┐
                         │       │
                       SSH 22  SMB 445
```

---

## Networking Notes

### L3 Packet Structure

```text
IP Packet
│
├── IP Header
│      ├── Source IP
│      ├── Destination IP
│      ├── TTL
│      ├── Protocol (TCP/UDP)
│      └── ...
│
└── Payload
       │
       ▼
   TCP Segment
      or
   UDP Datagram
```

### About Sockets

When `smbd` needs to use TCP port 445:

```text
Client
    │
Connect to port 445
    │
    ▼
systemd listens on port 22
    │
Detects an incoming connection
    │
    ▼
Starts smbd.service
    │
Passes the connection to smbd
    │
    ▼
smbd begins authentication
```

Port 445 is first handled by the systemd socket.

When a connection arrives, `smbd` is started.

A socket can be understood as a process's:

```text
"port placeholder + connection receiver"
```

### Data Flow

```text
                TCP Packet
                    │
                    ▼
            Network Card (NIC)
                    │
                    ▼
               Linux Kernel
                    │
             nftables check
                    │
         ┌──────────┴──────────┐
         │                     │
      ACCEPT                  DROP
         │                     │
         ▼                     ▼
 Check Socket Table        Discard Packet
         │
         ▼
 Is a program listening?
         │
     ┌───┴───┐
     │       │
    Yes      No
     │       │
     ▼       ▼
 Deliver   TCP RST
 to App    Connection Refused
(sshd /
 smbd)
```

### About the INPUT Chain

```text
NIC
 │
 ▼
Kernel
 │
 ▼
INPUT Chain
 │
 ├── ACCEPT → smbd / sshd / tailscaled
 │
 └── DROP   → Discard
```

---

## Common Operations

Check whether a network interface is up:

```bash
ip -br addr
```

View the server's currently listening network services:

```bash
sudo ss -tulpn
```
