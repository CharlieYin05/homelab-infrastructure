# Set Up Samba File Sharing Service

**2026-07-29**

---

## Why Use Samba

SMB is currently one of the most compatible, widely supported, and commonly used **LAN file-sharing protocols**.

It was developed by Microsoft and became widespread because of Windows' large market share. Since the SMB protocol is publicly documented, it is supported by most operating systems.

| Protocol | Windows | Linux | macOS | iOS | Android | NAS | Primary Use Case |
|---|---|---|---|---|---|---|---|
| **SMB** | ✅ Native | ✅ | ✅ Native | ✅ Native | ✅ | ✅ | General-purpose file sharing |
| NFS | ⚠️ Optional | ✅ Native | ✅ | ❌ | Limited | ✅ | Linux / Unix |
| AFP | ❌ | ❌ | ⚠️ Deprecated | ⚠️ | ❌ | Limited | Older Apple systems |
| FTP | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | File transfer, not file sharing |
| WebDAV | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Cloud storage / remote collaboration |

When Windows accesses a shared folder, it normally uses SMB.

Linux does not provide a Windows-style SMB file server by default, so Samba is used to make Linux behave as an SMB file server.

```text
Samba = SMB server on Linux
```

## Why Not Deploy Samba in Docker Like Immich?

- Samba behaves more like a system service than an application such as Immich, so running it directly on the host provides better integration with the operating system.
- Samba depends heavily on Linux users, groups, and filesystem permissions.
- UID/GID mismatches between the host and container can make permission mapping more difficult.
- Samba needs direct access to server users, permissions, and storage resources, so container isolation provides limited benefit in this case.

## File Sharing Flow

```text
Windows Explorer
        │
        ▼
SMB Protocol
        │
        ▼
smbd (Samba Service)
        │
        ├─ Authenticate SMB username/password
        │  using the Samba password database
        │
        ▼
Map to Linux User (UID/GID)
        │
        ├─ Check Samba share configuration
        │  such as `valid users`
        │
        ▼
Linux Kernel
        │
        ├─ Check Unix permission bits
        ├─ Check ACLs
        └─ Check SELinux/AppArmor if enabled
        │
        ▼
Disk (/srv/storage)
```

---

## Process

### 1. Install Samba

Install:

```bash
sudo apt update
sudo apt install samba
```

Check the default listening ports:

```bash
sudo ss -tulpn | grep smbd
```

View the configuration directory:

```bash
ls -l /etc/samba/
```

View the Samba user database:

```bash
sudo pdbedit -L
```

### 2. Create Samba Users

Before creating a Samba user, the corresponding Linux user must already exist and be assigned to the correct groups.

A Samba user consists of:

```text
Linux User + Samba Password
```

Samba ultimately needs to find the corresponding Linux user and map the authenticated Samba account to that Linux identity.

- Samba account → **Authentication**
- Linux account → **Authorization**

#### 2.1 Confirm That the Linux User Exists

```bash
id USERNAME
```

#### 2.2 Create a Samba User and Set a Password

```bash
sudo smbpasswd -a USERNAME
```

View the Samba users in the database:

```bash
sudo pdbedit -L
```

### 3. Share Files

A Samba Share is a network entry point that maps to a Linux filesystem path.

The Share itself is not the actual Linux path. The path must be explicitly defined in the Samba configuration.

#### 3.1 Back Up the Samba Configuration File

Backup:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

Restore if necessary:

```bash
sudo cp /etc/samba/smb.conf.bak /etc/samba/smb.conf
```

#### 3.2 Edit the Configuration File

```bash
sudo nano /etc/samba/smb.conf
```

Add the path mapping at the end of the file:

```ini
[public]                                                # Share name shown to clients

    path = /srv/storage/shares/public                   # Filesystem path

    browseable = yes                                    # Allow the Share to appear
                                                        # in client browsing lists
                                                        # (not a security control)

    read only = no                                      # Samba does not prevent
                                                        # users from writing

    valid users = @cy_public_rw @cy_public_ro           # Users/groups allowed to
                                                        # enter the Share.
                                                        # Does not determine whether
                                                        # they can write.

    create mask = 0660                                  # Limit newly created files
                                                        # to a maximum of 0660
                                                        # (-rw-rw----), except Owner

    directory mask = 2770                               # Limit newly created
                                                        # directories to 2770
                                                        # (drwxrwx---) and preserve SGID

    inherit permissions = yes                           # New files/directories
                                                        # inherit parent-directory
                                                        # permission characteristics
                                                        # where possible
```

Check the configuration syntax:

```bash
testparm
```

#### 3.3 Restart the Samba Service

```bash
sudo systemctl restart smbd
```

Check whether the service is active:

```bash
systemctl status smbd --no-pager
```

#### 3.4 Test SMB Access on Android, iOS, Windows and macOS

#### Issue 1: iOS Files App Detects the Share as Read-Only

Devices where the issue was reproduced:

- iPad mini 6 — iPadOS 18.6.2
- iPhone 13 — iOS 26.5.2

Symptoms:

- SMB authentication works normally.
- Share enumeration works normally.
- The Files app displays the Share as `Read Only`.
- Android 16, Windows 11 24H2, and macOS 15.6.1 all have correct permissions.

Status:

Deferred for later investigation.

The issue is currently believed to be related to the iOS client.

### 4. Samba Hardening

#### 4.1 Disable Unnecessary Shares

View the currently configured Shares:

```bash
testparm -s
```

Then comment out or remove:

```text
[homes]
[printers]
[print$]
```

#### 4.2 Change the Guest Policy

Prevent unknown users from being mapped to Guest:

```ini
map to guest = Never
```

Prevent normal users from creating Guest-accessible usershares:

```ini
usershare allow guests = No
```

#### 4.3 Set the Minimum SMB Protocol Version

Set SMB3 as the minimum protocol:

```ini
server min protocol = SMB3
```

#### 4.4 Disable Printing-Related RPC Services

```ini
disable spoolss = yes
```

#### 4.5 Disable NetBIOS

```ini
disable netbios = yes
```

---

## Troubleshooting Method 2.0

```text
① Network
   │
   ├─ ping
   ├─ TCP 445
   └─ Is smbd running?
        │
        ▼

② Share
   │
   ├─ testparm
   └─ smbclient -L localhost
        │
        ▼

③ User Authentication
   │
   ├─ pdbedit -L
   ├─ smbpasswd
   └─ journalctl
        │
        ▼

④ Samba Configuration
   │
   ├─ valid users
   ├─ write list
   └─ browseable
        │
        ▼

⑤ Linux Permissions
   │
   ├─ ls -l
   ├─ getfacl
   └─ id
        │
        ▼

⑥ SELinux / AppArmor
```
