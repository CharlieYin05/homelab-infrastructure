# Create Directory Structure and Configure User File Permissions

**2026-07-27**

---

## Permission Model

### Layer 1 (Fixed): RBAC

| Shared Directory | Role Group | Permission |
| ---------------- | ---------- | ---------- |
| `public` | `cy_public_ro` | Read-only |
| `public` | `cy_public_rw` | Read/write |
| `restriction` | `cy_restriction_ro` | Read-only |
| `restriction` | `cy_restriction_rw` | Read/write |
| `private` | `cy_private_rw` | Read/write (Administrator) |

#### Groups

```text
cy_public_ro
    ├── guest1
    ├── guest2
    └── ...

cy_public_rw
    ├── friendA
    └── cyin026

cy_restriction_ro
    ├── friendB
    └── friendC

cy_restriction_rw
    ├── friendD
    └── cyin026

cy_private
    └── cyin026
```

### Layer 2 (Flexible): ACL

Use ACLs to manage special subdirectories inside shared directories.

For example:

```text
restriction/
├── Bob_and_Charles_only
├── Alice_only
├── Finance
└── Project_X
```

---

## Process

### 1. Create Five User Groups and Verify

```bash
sudo groupadd cy_public_ro
sudo groupadd cy_public_rw

sudo groupadd cy_restriction_ro
sudo groupadd cy_restriction_rw

sudo groupadd cy_private_rw

getent group | grep "^cy_"
```

### 2. Create Shared Directories

```bash
sudo mkdir -p /srv/storage/shares/public
sudo mkdir -p /srv/storage/shares/restriction
sudo mkdir -p /srv/storage/shares/private
```

### 3. Set Owners

#### Define Directory Owners

| Directory | Owner | Group |
| --------- | ----- | ----- |
| public | root | cy_public_rw |
| restriction | root | cy_restriction_rw |
| private | me | cy_private_rw |

```bash
sudo chown root:cy_public_rw /srv/storage/shares/public
sudo chown root:cy_restriction_rw /srv/storage/shares/restriction
sudo chown cyin026:cy_private_rw /srv/storage/shares/private
```

### 4. Configure Basic `chmod` Permissions

New files and subdirectories created under `public_rw`, `restriction_rw`, and `private` will automatically inherit the directory's Group.

```text
2770
│││└── Other
││└── Group
│└── Owner
└── Special Permission (SGID)
```

#### When a User Accesses a File, Linux Roughly Evaluates Permissions in This Order

```text
1. Owner
   Is the user the Owner?
   │
   No
   ▼

2. ACL
   Is there an ACL entry for this user?
   │
   No
   ▼

3. Group
   Does the user belong to the Group?
   │
   No
   ▼

4. Other
   Use Other permissions
```

```bash
sudo chmod 2770 /srv/storage/shares/public
sudo chmod 2770 /srv/storage/shares/restriction
sudo chmod 2770 /srv/storage/shares/private
```

### 5. Configure ACLs

#### 5.1 Grant `cy_public_ro` and `cy_restriction_ro` Read + Directory Traversal Permissions

```bash
sudo setfacl -m g:cy_public_ro:rx /srv/storage/shares/public
sudo setfacl -m g:cy_restriction_ro:rx /srv/storage/shares/restriction
```

#### 5.2 Configure Default ACLs for New Content in `public` and `restriction`

New files and subdirectories created under `public` will automatically inherit the `cy_public_ro` ACL entry.

Apply and verify:

```bash
sudo setfacl -d -m g:cy_public_ro:rx /srv/storage/shares/public
sudo setfacl -d -m g:cy_restriction_ro:rx /srv/storage/shares/restriction
```

### 6. Add the First User `cyin026` (Administrator) to the Groups

Add the user:

```bash
sudo usermod -aG \
cy_public_ro,cy_public_rw,\
cy_restriction_ro,cy_restriction_rw,\
cy_private_rw \
cyin026
```

Log out and reconnect through SSH, then check:

```bash
groups
```

If the output contains the following groups, the configuration was successful:

```text
cy_public_ro cy_public_rw cy_restriction_ro cy_restriction_rw cy_private_rw
```

### 7. Validation

```text
               public
                  │
                  ▼
         Group = cy_public_rw
                  │
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
        SGID          Default ACL
         │                 │
         ▼                 ▼
 New files inherit    RO permission
 cy_public_rw         automatically inherits
                      cy_public_ro
```

---

## Linux User File Permission System

### Basic Concept: Both Users and Files Have Groups

Run:

```bash
id cyin026
```

Output:

```text
uid=1000(cyin026)
gid=1000(cyin026)
groups=1000(cyin026),1002(cy_public_rw),1004(cy_restriction_rw),1005(cy_private_rw)
```

Explanation:

```text
Username:
    cyin026

Primary Group (gid):
    cyin026
    ← Files/directories created by cyin026 normally belong to the
      cyin026 file Group.

Supplementary Groups (groups):
    cy_public_rw
    ← cy_public_rw is both a user group and a file group.
      The same group acts like a label referenced by both users
      and files.

    cy_restriction_rw
    cy_private_rw
```

### Layer 1: `chmod`

It only understands:

```text
Owner
Group
Other
```

For example, `drwxrwx---` means:

```text
Owner  → rwx
Group  → rwx
Other  → ---
```

### Layer 2: SGID — Automatically Keep Group Ownership Consistent

Assume the directory is:

```text
public
```

Its Group is:

```text
cy_public_rw
```

Without SGID:

Charlie creates `movie.mp4`.

By default:

```text
Owner = charlie
Group = charlie
        (Charlie's Primary Group)
```

Even if Bob has `cy_public_rw` in his Supplementary Groups, he cannot read or write `movie.mp4`, because the file's Group only inherits Charlie's Primary Group.

With SGID:

Charlie creates `movie2.mp4`.

By default:

```text
Owner = charlie
Group = cy_public_rw
```

Bob can open it.

### Layer 3: ACL — When Basic Permissions Are Not Enough

`chmod` limits each file to a single Group.

ACLs allow permissions to be assigned separately to multiple users or groups.

Example of a file without ACL:

```text
Owner = Alice
Group = cy_public_rw

Permissions:

Owner : rw-
Group : rw-
Other : ---
```

If `cy_public_rw` already has read/write permission but additional users or groups also need their own permissions, `chmod` alone cannot provide this flexibility.

Example of a file with ACL:

```text
report.docx

Owner = Alice
Group = cy_public_rw

ACL =
    group:cy_public_ro
    group:auditor
    user:jack
```

This allows additional users and groups to have their own permissions on the file.

### Layer 4: Default ACL

A directory itself may have ACL entries, but newly created files and subdirectories inside it also need to inherit ACL rules.

Default ACLs provide this inheritance behaviour.

---

## Useful Commands

### View Existing Groups

```bash
getent group
```

Or view the users inside a specific group:

```bash
getent group GROUP_NAME
```

### Create a New Group

```bash
sudo groupadd GROUP_NAME
```

### Create a New User

```bash
sudo useradd -m -s /bin/bash USERNAME
sudo passwd USERNAME
```

Explanation:

- `-m` — Create a Home Directory
- `-s` — Specify the login Shell

### Add a User to a Supplementary Group

```bash
sudo usermod -aG GROUP_NAME USERNAME
```

Explanation:

- `-a` — Append. This is important; without it, existing supplementary groups may be overwritten.
- `-G` — Supplementary Groups

### View a User's Groups

```bash
groups USERNAME
```

### View Basic File/Directory Information

This includes Owner, Group, permissions, and whether ACLs are present.

Input:

```bash
ls -l /SRV/storage/shares/XXX/XXXX
```

Example output:

```text
-rw-rw----+ 1 cyin026 cy_public_rw test.txt
```

Explanation:

```text
-             File type (regular file in this example)

1             Number of hard links

rw-rw----     Permissions:
              Owner → rw-
              Group → rw-
              Other → ---

cyin026       Owner

cy_public_rw  Group

+             Indicates that ACL entries exist
```

#### View File/Directory ACLs

Input:

```bash
getfacl /srv/storage/shares/XXX/XXX
```

Example output using `public`:

```text
Object Information:

# file: srv/storage/shares/public
# owner: root
# group: cy_public_rw
# flags: -s-                         ← SGID enabled


Access ACL:

user::rwx                            ← Owner permissions
group::rwx                           ← Permissions for the owning Group
                                       (cy_public_rw)
group:cy_public_ro:r-x               ← Permissions for the additional Group
mask::rwx                            ← Maximum effective ACL permissions
other::---                           ← Permissions for Other


Default ACL:
                                      ← Default ACL applied to newly created
                                         files and subdirectories

default:user::rwx
default:group::rwx
default:group:cy_public_ro:r-x
default:mask::rwx
default:other::---
```

---

## Troubleshooting (After Samba Authentication)

### Permission Denied

1. Does the user belong to the correct Group?

```bash
id
```

2. Which Group owns the file, and are the ACLs correct?

```bash
getfacl
```

3. Is SGID enabled?

```bash
ls -ld
```

4. Log out and log back in.

---

## Naming Conventions

### Usernames

1. Use lowercase English letters or numbers.
2. Do not use spaces; use `-` instead.

### Group Names

1. Use lowercase English letters or numbers.
2. Do not use spaces; use `_` instead.

### Directory Names

1. Use lowercase English letters or numbers.
2. Do not use spaces; use `_` instead.

### File Names

1. Use lowercase English letters or numbers.
2. Do not use spaces; use `-` instead.
