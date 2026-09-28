# Set Up Samba File Auditing

**2026-08-01**

---

## Objective

Give the FSS server auditability and establish a foundation for future learning in SIEM, log analysis, and incident response.

## Expected Result

Example:

```text
Windows
   │
   │ Create test.txt
   ▼
Samba
   │
   ▼
/srv/logs/samba/audit.log

2026-08-01 15:23:11
user=cyin026
share=public
operation=create_file
path=test.txt
```

## About Audit Logs

An audit log records **what a user did**.

This is implemented using Samba's VFS (Virtual File System) plugin:

```text
Windows
   │
   ▼
Samba
   │
   ▼
full_audit plugin
   │
   ├── Write audit log
   │
   └── Continue file operation
```

---

## Process

### 1. Check Whether `full_audit` Is Installed

Note that a normal Debian user's `PATH` does not include `/usr/sbin`, so `/usr/sbin/smbd` may not be found without elevated privileges.

Check whether the module is available:

```bash
sudo /usr/sbin/smbd -b | grep VFS
```

### 2. Enable `full_audit`

#### 2.1 Back Up the Samba Configuration File

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.audit.bak
```

#### 2.2 Edit the Configuration

Add the following under `global`, `public`, `restriction`, and `private`:

```ini
vfs objects = full_audit                                           # Enable auditing

full_audit:prefix = %u|%I|%S                                       # username|client IP|share name
full_audit:success = mkdir rmdir rename unlink open create_file    # Successful operations to record
full_audit:failure = none                                          # Do not record failed operations
full_audit:facility = LOCAL7                                       # Send logs to syslog facility LOCAL7
full_audit:priority = NOTICE                                       # Log priority
```

Save the configuration and check the syntax.

### 3. Write Audit Logs to `/srv/logs/samba`

Flow:

```text
Windows
      │
      ▼
    Samba
      │
      ▼
 full_audit
      │
      ▼
   rsyslog
      │
      ▼
  audit.log
```

On Debian, `rsyslog` is responsible for actually writing the log file.

#### 3.1 Check Whether Debian Has `rsyslog`

```bash
systemctl status rsyslog
```

If it is not installed:

```bash
sudo apt update
sudo apt install rsyslog

systemctl status rsyslog
```

#### 3.2 Create an `rsyslog` Rule

Note: Debian does not automatically create a `syslog` user in this setup. The following operations are therefore performed as `root`.

Create the configuration file:

```bash
sudo nano /etc/rsyslog.d/30-samba-audit.conf
```

Add:

```text
if ($programname == "smbd_audit" and $msg contains ".DS_Store") then stop
if ($programname == "smbd_audit" and $msg contains "/._") then stop

local7.notice    /srv/logs/samba/audit.log
```

Create and configure the log file:

```bash
sudo touch /srv/logs/samba/audit.log
sudo chown root:adm /srv/logs/samba/audit.log
sudo chmod 640 /srv/logs/samba/audit.log
```

Verify:

```bash
ls -l /srv/logs/samba
```

Restart `rsyslog`:

```bash
sudo systemctl restart rsyslog
```

Restart `smbd`:

```bash
sudo systemctl restart smbd
```

Confirm that neither service reports errors:

```bash
sudo systemctl status rsyslog
sudo systemctl status smbd
```

---

## Issue 1

After creating the `rsyslog` rule, SMB access stopped working.

### Troubleshooting

Install the Samba client locally on the server. If `localhost` access works, the problem is likely on the client side.

```bash
sudo apt install smbclient
```

Remove the VFS plugin from `[global]`, then temporarily add a higher Samba log level for troubleshooting:

```ini
log level = 3
```

Test SMB access through localhost:

```bash
smbclient -L localhost -U cyin026
```

Try accessing `public`:

```bash
smbclient //localhost/public -U cyin026
```

Output:

```text
tree connect failed: NT_STATUS_UNSUCCESSFUL
```

| Test | Result |
|---|---|
| `full_audit` disabled | ✅ SMB works normally |
| `full_audit` under `[global]` | ❌ Share connection fails with `NT_STATUS_UNSUCCESSFUL` |
| `full_audit` configured only under a Share | ✅ Share can be listed, but cannot be entered |

The problem was therefore narrowed down to the `full_audit` configuration itself rather than Samba, permissions, ACLs, or `rsyslog`.

Remove `open` and `create_file` from the successful operations under `[public]`:

```ini
full_audit:success = mkdir rmdir rename unlink
```

---

## 2026/08/02 11:58 AM

Removing only `open` and `create_file` still failed.

I suspected that the operation names under `success` might belong to an older version of the API, although I was not yet sure whether this was caused by Debian or Samba.

First, apply the official `full_audit` template to `[private]`:

```ini
vfs objects = full_audit

full_audit:prefix = %u|%I
full_audit:success = open opendir
full_audit:failure = all !open

full_audit:facility = LOCAL7
full_audit:priority = NOTICE
```

---

## 2026/08/02 12:06 PM

Even the official template still failed.

Three major possibilities remained:

1. A regression bug in Samba 4.22 + Debian 13 `full_audit`.
2. A Debian Samba packaging issue.
3. One unverified possibility: whether `full_audit` depends on certain default VFS modules such as `acl_xattr`.

---

## 2026/08/02 12:14 PM

Increase the Samba log level to 11, attempt a connection, and immediately inspect the logs.

Client:

```bash
smbclient //localhost/private -U cyin026 -d 11
```

Server:

```bash
sudo grep -R "init_bitmap" /var/log/samba
```

```bash
sudo grep -R "Invalid success" /var/log/samba
```

---

## 2026/08/02 12:27 PM

The logs revealed the error.

Originally:

```text
init_bitmap: Could not find opname mkdir
```

After switching to the official example:

```ini
full_audit:success = open opendir
```

The log showed:

```text
smb_full_audit_connect: Invalid success operations list. Failing connect
```

The log directly indicated that `full_audit` failed while parsing the `full_audit:success` list during initialization, causing the entire Share connection to be rejected.

This suggested that the `full_audit` module itself was not broken.

Instead, the operation names configured in this version of Samba 4.22.10 might no longer exist.

---

## 2026/08/02 12:38 PM

The official documentation bundled with this Samba installation, `vfs_full_audit.8.gz`, appeared to be outdated.

Many online tutorials also used older API operation names, and those names no longer worked.

Even the official example:

```ini
full_audit:success = open opendir
```

was documented as incorrect by Samba developers.

First, configure a minimal VFS configuration under `[private]`, then add operations one by one and test which operation names are valid.

Minimal configuration:

```ini
vfs objects = full_audit

full_audit:prefix = %u|%I
full_audit:success = none
full_audit:failure = none

full_audit:facility = LOCAL7
full_audit:priority = NOTICE
```

The `private` Share passed the test.

The next step was to inspect the Samba source code directly and determine which `full_audit` operations actually exist.

---

## 2026/08/02 12:50 PM

### 1. Locate the Official Source File

Find the official Samba repository and inspect:

```text
source3/modules/vfs_full_audit.c
```

### 2. Trace `init_bitmap()`

Based on the log message, inspect the inputs and outputs of `init_bitmap()`:

```c
static struct bitmap *init_bitmap(TALLOC_CTX *mem_ctx, const char **ops)
{
	struct bitmap *bm;

	if (ops == NULL) {
		DBG_ERR("init_bitmap, ops list is empty (logic error)\n");
		return NULL;
	}

	bm = bitmap_talloc(mem_ctx, SMB_VFS_OP_LAST);
	if (bm == NULL) {
		DBG_ERR("Could not alloc bitmap\n");
		return NULL;
	}

	// The error I encountered is generated inside this section ↓
	for (; *ops != NULL; ops += 1) {
		int i;
		bool neg = false;
		const char *op;

		if (strequal(*ops, "all")) {
			for (i=0; i<SMB_VFS_OP_LAST; i++) {
				bitmap_set(bm, i);
			}
			continue;
		}

		if (strequal(*ops, "none")) {
			break;
		}

		op = ops[0];
		if (op[0] == '!') {
			neg = true;
			op += 1;
		}

		for (i=0; i<SMB_VFS_OP_LAST; i++) {
			if ((vfs_op_names[i].name == NULL)
			 || (vfs_op_names[i].type != i)) {
				smb_panic("vfs_full_audit.c: name table not "
					  "in sync with vfs_op_type enums\n");
			}
			if (strequal(op, vfs_op_names[i].name)) {
				if (neg) {
					bitmap_clear(bm, i);
				} else {
					bitmap_set(bm, i);
				}
				break;
			}
		}

		if (i == SMB_VFS_OP_LAST) {
			DBG_ERR("Could not find opname %s\n", *ops);
			TALLOC_FREE(bm);
			return NULL;
		}
	}

	return bm;
	// The error I encountered is generated inside this section ↑
}
```

Input:

```text
TALLOC_CTX *mem_ctx
const char **ops
```

Output:

```text
return bm
```

means success.

```text
NULL
```

means failure.

### 3. Identify the Failure Condition

The failing section is the `for` loop that checks whether each configured operation exists in `vfs_op_names[]`.

### 4. Inspect the Actual Operations in `vfs_op_names[]`

```c
static struct {
	vfs_op_type type;
	const char *name;
} vfs_op_names[] = {
	{ SMB_VFS_OP_CONNECT,	"connect" },
	{ SMB_VFS_OP_DISCONNECT,	"disconnect" },
	{ SMB_VFS_OP_OPEN_SHARE_ROOT,	"open_share_root" },
	{ SMB_VFS_OP_DISK_FREE,	"disk_free" },
	{ SMB_VFS_OP_GET_QUOTA,	"get_quota" },
	{ SMB_VFS_OP_SET_QUOTA,	"set_quota" },
	{ SMB_VFS_OP_GET_SHADOW_COPY_DATA,	"get_shadow_copy_data" },
	{ SMB_VFS_OP_STATVFS,	"statvfs" },
	{ SMB_VFS_OP_FSTATVFS,	"fstatvfs" },
	{ SMB_VFS_OP_FS_CAPABILITIES,	"fs_capabilities" },
	{ SMB_VFS_OP_GET_DFS_REFERRALS,	"get_dfs_referrals" },
	{ SMB_VFS_OP_CREATE_DFS_PATHAT,	"create_dfs_pathat" },
	{ SMB_VFS_OP_READ_DFS_PATHAT,	"read_dfs_pathat" },
	{ SMB_VFS_OP_FDOPENDIR,	"fdopendir" },
	{ SMB_VFS_OP_READDIR,	"readdir" },
	{ SMB_VFS_OP_REWINDDIR, "rewinddir" },

	{ SMB_VFS_OP_MKDIRAT,	"mkdirat" },       // Create directory — operation I want to audit

	{ SMB_VFS_OP_CLOSEDIR,	"closedir" },

	{ SMB_VFS_OP_OPEN,	"open" },           // Open file/directory — operation I want to audit
	{ SMB_VFS_OP_OPENAT,	"openat" },

	{ SMB_VFS_OP_CREATE_FILE, "create_file" }, // Create/open file — operation I want to audit

	{ SMB_VFS_OP_CLOSE,	"close" },
	{ SMB_VFS_OP_READ,	"read" },
	{ SMB_VFS_OP_PREAD,	"pread" },
	{ SMB_VFS_OP_PREAD_SEND,	"pread_send" },
	{ SMB_VFS_OP_PREAD_RECV,	"pread_recv" },
	{ SMB_VFS_OP_WRITE,	"write" },
	{ SMB_VFS_OP_PWRITE,	"pwrite" },
	{ SMB_VFS_OP_PWRITE_SEND,	"pwrite_send" },
	{ SMB_VFS_OP_PWRITE_RECV,	"pwrite_recv" },
	{ SMB_VFS_OP_LSEEK,	"lseek" },
	{ SMB_VFS_OP_SENDFILE,	"sendfile" },
	{ SMB_VFS_OP_RECVFILE,  "recvfile" },

	{ SMB_VFS_OP_RENAMEAT,	"renameat" },       // Rename/move — operation I want to audit

	{ SMB_VFS_OP_RENAME_STREAM,	"rename_stream" },
	{ SMB_VFS_OP_FSYNC_SEND,	"fsync_send" },
	{ SMB_VFS_OP_FSYNC_RECV,	"fsync_recv" },
	{ SMB_VFS_OP_STAT,	"stat" },
	{ SMB_VFS_OP_FSTAT,	"fstat" },
	{ SMB_VFS_OP_LSTAT,	"lstat" },
	{ SMB_VFS_OP_FSTATAT,	"fstatat" },
	{ SMB_VFS_OP_GET_ALLOC_SIZE,	"get_alloc_size" },

	{ SMB_VFS_OP_UNLINKAT,	"unlinkat" },       // Delete file/directory entry — operation I want to audit

	{ SMB_VFS_OP_FCHMOD,	"fchmod" },
	{ SMB_VFS_OP_FCHOWN,	"fchown" },
	{ SMB_VFS_OP_LCHOWN,	"lchown" },
	{ SMB_VFS_OP_CHDIR,	"chdir" },
	{ SMB_VFS_OP_NTIMES,	"ntimes" },
	{ SMB_VFS_OP_FNTIMES,	"fntimes" },
	{ SMB_VFS_OP_FTRUNCATE,	"ftruncate" },
	{ SMB_VFS_OP_FALLOCATE,"fallocate" },
	{ SMB_VFS_OP_LOCK,	"lock" },
	{ SMB_VFS_OP_FILESYSTEM_SHAREMODE,	"filesystem_sharemode" },
	{ SMB_VFS_OP_FCNTL,	"fcntl" },
	{ SMB_VFS_OP_LINUX_SETLEASE, "linux_setlease" },
	{ SMB_VFS_OP_GETLOCK,	"getlock" },
	{ SMB_VFS_OP_SYMLINKAT,	"symlinkat" },
	{ SMB_VFS_OP_READLINKAT,"readlinkat" },
	{ SMB_VFS_OP_LINKAT,	"linkat" },
	{ SMB_VFS_OP_MKNODAT,	"mknodat" },
	{ SMB_VFS_OP_REALPATH,	"realpath" },
	{ SMB_VFS_OP_FCHFLAGS,	"fchflags" },
	{ SMB_VFS_OP_FILE_ID_CREATE,	"file_id_create" },
	{ SMB_VFS_OP_FS_FILE_ID,	"fs_file_id" },
	{ SMB_VFS_OP_FSTREAMINFO,	"fstreaminfo" },
	{ SMB_VFS_OP_GET_REAL_FILENAME, "get_real_filename" },
	{ SMB_VFS_OP_GET_REAL_FILENAME_AT, "get_real_filename_at" },
	{ SMB_VFS_OP_BRL_LOCK_WINDOWS,  "brl_lock_windows" },
	{ SMB_VFS_OP_BRL_UNLOCK_WINDOWS, "brl_unlock_windows" },
	{ SMB_VFS_OP_STRICT_LOCK_CHECK, "strict_lock_check" },
	{ SMB_VFS_OP_TRANSLATE_NAME,	"translate_name" },
	{ SMB_VFS_OP_PARENT_PATHNAME,	"parent_pathname" },
	{ SMB_VFS_OP_FSCTL,		"fsctl" },
	{ SMB_VFS_OP_OFFLOAD_READ_SEND,	"offload_read_send" },
	{ SMB_VFS_OP_OFFLOAD_READ_RECV,	"offload_read_recv" },
	{ SMB_VFS_OP_OFFLOAD_WRITE_SEND,	"offload_write_send" },
	{ SMB_VFS_OP_OFFLOAD_WRITE_RECV,	"offload_write_recv" },
	{ SMB_VFS_OP_FGET_COMPRESSION,	"fget_compression" },
	{ SMB_VFS_OP_SET_COMPRESSION,	"set_compression" },
	{ SMB_VFS_OP_SNAP_CHECK_PATH, "snap_check_path" },
	{ SMB_VFS_OP_SNAP_CREATE, "snap_create" },
	{ SMB_VFS_OP_SNAP_DELETE, "snap_delete" },
	{ SMB_VFS_OP_GET_DOS_ATTRIBUTES_SEND, "get_dos_attributes_send" },
	{ SMB_VFS_OP_GET_DOS_ATTRIBUTES_RECV, "get_dos_attributes_recv" },
	{ SMB_VFS_OP_FGET_DOS_ATTRIBUTES, "fget_dos_attributes" },
	{ SMB_VFS_OP_FSET_DOS_ATTRIBUTES, "fset_dos_attributes" },
	{ SMB_VFS_OP_FGET_NT_ACL,	"fget_nt_acl" },
	{ SMB_VFS_OP_FSET_NT_ACL,	"fset_nt_acl" },
	{ SMB_VFS_OP_SYS_ACL_GET_FD,	"sys_acl_get_fd" },
	{ SMB_VFS_OP_SYS_ACL_BLOB_GET_FD,	"sys_acl_blob_get_fd" },
	{ SMB_VFS_OP_SYS_ACL_SET_FD,	"sys_acl_set_fd" },
	{ SMB_VFS_OP_SYS_ACL_DELETE_DEF_FD,	"sys_acl_delete_def_fd" },
	{ SMB_VFS_OP_GETXATTRAT_SEND, "getxattrat_send" },
	{ SMB_VFS_OP_GETXATTRAT_RECV, "getxattrat_recv" },
	{ SMB_VFS_OP_FGETXATTR,	"fgetxattr" },
	{ SMB_VFS_OP_FLISTXATTR,	"flistxattr" },
	{ SMB_VFS_OP_REMOVEXATTR,	"removexattr" },
	{ SMB_VFS_OP_FREMOVEXATTR,	"fremovexattr" },
	{ SMB_VFS_OP_FSETXATTR,	"fsetxattr" },
	{ SMB_VFS_OP_AIO_FORCE, "aio_force" },
	{ SMB_VFS_OP_IS_OFFLINE,	"is_offline" },
	{ SMB_VFS_OP_SET_OFFLINE,	"set_offline" },
	{ SMB_VFS_OP_DURABLE_COOKIE,	"durable_cookie" },
	{ SMB_VFS_OP_DURABLE_DISCONNECT,	"durable_disconnect" },
	{ SMB_VFS_OP_DURABLE_RECONNECT,	"durable_reconnect" },
	{ SMB_VFS_OP_FREADDIR_ATTR,      "freaddir_attr" },
	{ SMB_VFS_OP_LAST, NULL }
};
```

---

## Issue 2

### Symptom

Opening a file produced no `full_audit` log entries.

---

## 2026-08-02 2:07 PM

Organising the evidence:

| Component | Status |
|---|---|
| `full_audit.so` exists | ✅ |
| `vfs objects = full_audit` is active | ✅ |
| `init_bitmap()` works | ✅ |
| Operation names come from `vfs_op_names[]` | ✅ |
| `mkdir` → `mkdirat` verified | ✅ |
| `open` is a valid operation | ✅ |
| `rsyslog` works | ✅ |
| LOCAL7 works | ✅ |
| `audit.log` is writable | ✅ |
| Share opens normally | ✅ |
| **No audit logs generated** | ❌ |

Suspicions:

1. `do_log()` is never executed.
2. `do_log()` executes, but `log_success()` filters the event out.

I need to continue reading the source code to locate the problem...

Too tired and hungry. Time to eat first.

---

## 2026-08-02 3:36 PM

Current evidence chain:

```text
full_audit.so          ✅ Exists
        │
        ▼
vfs objects            ✅ Loaded
        │
        ▼
init_bitmap()          ✅ Working
        │
        ▼
Operation names        ✅ Verified (mkdir → mkdirat)
        │
        ▼
do_log()               ❓
        │
        ▼
syslog()               Theoretically called
        │
        ▼
LOCAL7                  ✅ Working
        │
        ▼
rsyslog                 ✅ Working
        │
        ▼
audit.log               ✅ Working
```

From the source code:

```text
             do_log()
                │
       ┌────────┴────────┐
       │                 │
  syslog=true       syslog=false
       │                 │
   syslog()           DEBUG(1)
       │                 │
   rsyslog        /var/log/samba/log.*
```

Therefore, deliberately triggering `syslog=false` should help narrow the problem further.

Under `[private]`, change `syslog` from `true` to `false`.

Also change `success` to:

```text
connect openat create_file
```

Then observe the logs.

Server:

```bash
sudo tail -f /var/log/samba/log.smbd
```

Client:

Enter `private`, run `ls`, then `quit`.

The output only contained normal Samba debug logs.

This suggests the problem is not with `syslog()` itself.

According to the source:

```c
if (pd->do_syslog) {
    syslog(...);
} else {
    DEBUG(1, (...));
}
```

The `else` branch was triggered directly instead of `syslog()`.

That means the many `do_log()` functions I expected were apparently not being called at all.

This was extremely confusing.

---

## 2026-08-02 4:06 PM

If `do_log()` is really not being executed, inspect the Debian Samba source directly:

```bash
sudo apt update

sudo apt install dpkg-dev

apt source samba
```

Then inspect Debian's version of:

```text
vfs_full_audit.c
```

Narrow the question down further:

Why does even a basic connection through `smb_full_audit_connect()` not generate output from `do_log()`?

Why would this function not be registered as a VFS hook?

The `static struct vfs_fn_pointers vfs_full_audit_fns` structure contains:

```c
.connect_fn = smb_full_audit_connect,
```

so the VFS hook registration appears to be correct.

Call chain:

```text
smb.conf
    │
    ▼
vfs objects = full_audit
    │
    ▼
vfs_full_audit_init()
    │
    ▼
smb_register_vfs(...)
    │
    ▼
vfs_full_audit_fns
    │
    ├── connect_fn      -> smb_full_audit_connect
    ├── openat_fn       -> smb_full_audit_openat
    ├── create_file_fn  -> smb_full_audit_create_file
    └── ...
```

---

## 2026-08-02 4:49 PM

The question is now even narrower:

Why is the Hook registered correctly but still producing no audit logs?

First, Debian 4.22's `vfs_full_audit.c` differs from the Samba GitHub upstream source.

The upstream version uses:

```c
do_log(op, NULL, ...)
```

while Debian 4.22 uses:

```c
do_log(op, true, ...)
```

However, this difference does not appear relevant to the current problem.

---

## 2026-08-02 5:16 PM

This is getting frustrating.

Could the troubleshooting approach itself have been wrong from the beginning?

Maybe I should not have treated `audit.log` as the only possible log destination.

Search every log under `/var/log` for entries containing `ok|` and `connect`.

```bash
sudo grep -R "ok|" /var/log
sudo grep -R "connect" /var/log
```

The result revealed:

### 1. `vfs_full_audit` Was Working Normally

For example:

```text
/var/log/samba/log.cy-server-fss.old:

cyin026|::1|connect|ok|private

cyin026|::1|openat|ok|r|/srv/storage/shares/private

cyin026|::1|create_file|ok|0x81|dir|open|/srv/storage/shares/private
```

This is exactly the output format produced by `full_audit`.

The `do_log()` function:

```c
syslog(priority,
    "%s|%s|%s|%s\n",
    audit_pre,
    audit_opname(op),
    err_msg,
    op_msg);
```

matches the output:

```text
cyin026|::1|connect|ok|private
```

### 2. I Had Been Looking at the Wrong Log File

The logs were not being written to:

```text
/srv/logs/samba/audit.log
```

They were being written to:

```text
/var/log/samba/log.cy-server-fss.old
```

Why?

Why was `full_audit` bypassing `rsyslog` and being handled by Samba's own logging backend?

---

## 2026-08-02 5:29 PM

I finally realised the problem.

During earlier testing, `[private]` still had:

```ini
full_audit:syslog = false
```

According to the source code:

```c
if (pd->do_syslog) {
    syslog(...);
} else {
    DEBUG(1, ("%s|%s|%s|%s\n", ...));
}
```

When set to `false`:

```text
full_audit:syslog = false
        │
        ▼
pd->do_syslog = false
        │
        ▼
syslog() is NOT called
        │
        ▼
DEBUG(1, ...) is called
```

Therefore, the logs never passed through `syslog()`.

Instead, they were written to:

```text
/var/log/samba/log.cy-server-fss.old
```

Change the setting back to `true` and test again.

Finally, `audit.log` started working correctly.

---

## 4. Configure Log Rotation

### Objective: Use `logrotate` to Retain 12 Months of Logs

#### 4.1 Create a Rule for `/srv/logs/samba/audit.log`

Edit the `logrotate` configuration:

```bash
sudo nano /etc/logrotate.d/samba-audit
```

Follow the format used by Debian's existing rules:

```text
/srv/logs/samba/audit.log
{
    rotate 12

    monthly

    missingok

    notifempty

    compress

    delaycompress

    sharedscripts

    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

Explanation:

```text
rotate 12
→ Keep 12 rotated logs

monthly
→ Rotate once per month

missingok
→ Do not report an error if the log file does not exist

notifempty
→ Do not rotate an empty log file

compress
→ Compress older rotated logs

delaycompress
→ Do not compress the newest rotated log until the next rotation

sharedscripts
→ Run the postrotate script once

postrotate
→ Execute the following command after rotation

/usr/lib/rsyslog/rsyslog-rotate
→ Tell rsyslog to reopen audit.log instead of continuing
  to write to the old rotated audit.log.1
```

---

## Log Lifecycle

```text
Samba
  │
  │ Producer
  ▼
rsyslog
  │
  │ Collector
  ▼
logrotate
  │
  │ Rotation / Retention
  ▼
Archived Logs
```

- Samba — produces the audit events
- `rsyslog` — collects and writes the logs
- `logrotate` — manages the log lifecycle

---

## VFS Operation Reference — Source Code Version

| VFS Operation | Meaning | Example User Action |
|---|---|---|
| **connect** | Client connects to a Share | Open `\\server\private`, enter `private` from a mobile client |
| **disconnect** | Client disconnects from a Share | Close SMB session, exit file manager, network disconnects |
| **openat** | Open a file or directory using Linux `openat()` | Browse a directory, open an image/PDF, view file properties |
| **create_file** | SMB CREATE request for opening or creating files/directories | Create a file, open a file, create a directory; some clients may trigger this while browsing |
| **mkdirat** | Create a directory | Create a new folder |
| **renameat** | Rename or move | Rename a file or move it to another directory |
| **unlinkat** | Remove a directory entry | Delete a file and some directory-removal operations |

---

## Troubleshooting Open-Source Projects

1. Use logs to locate the relevant function.
2. Read the relevant function instead of reading the entire project.
3. Form and test hypotheses.
4. Eliminate possible causes one by one.
5. Let the evidence determine the next step instead of changing configuration based on guesses.

---

## Typical Structure of a Large C Open-Source Project

```text
Configuration File
        │
        ▼
Parser (`lp_parm_*`)
        │
        ▼
Module Init / Registration
        │
        ▼
Function Pointer Table / Hook
        │
        ▼
Actual Function
(`smb_full_audit_openat`)
        │
        ▼
Logs
```

---

## Samba Configuration File Reference

### View the Configuration

```bash
cat /etc/samba/smb.conf
```

### Edit the Configuration

```bash
sudo nano /etc/samba/smb.conf
```

### Validate and Restart After Changes

```bash
sudo testparm
sudo systemctl restart smbd
```

---

## Samba `audit.log` Reference

### View All Records

```bash
sudo cat /srv/logs/samba/audit.log
```

### View the Last 20 Lines

```bash
sudo tail -20 /srv/logs/samba/audit.log
```

### Search for a Specific User

```bash
sudo grep "USERNAME" /srv/logs/samba/audit.log
```

### Search for a Specific IP Address

```bash
sudo grep "IP_ADDRESS" /srv/logs/samba/audit.log
```

### Search for Failed Operations

```bash
sudo grep "fail" /srv/logs/samba/audit.log
```

### View Only Delete Operations

```bash
sudo grep "unlinkat" /srv/logs/samba/audit.log
```

### Monitor in Real Time

```bash
sudo tail -f /srv/logs/samba/audit.log
```

---

## Samba `rsyslog.d` Reference

Usually used to add log filtering rules.

Edit:

```bash
sudo nano /etc/rsyslog.d/30-samba-audit.conf
```

Restart:

```bash
sudo systemctl restart rsyslog
```

---

## `logrotate` Configuration Reference

### View Existing `logrotate` Rules

```bash
ls /etc/logrotate.d/
```

### View Generated Samba Logs

```bash
ls -lh /srv/logs/samba
```

### View the `rsyslog` Rotation Rule

```bash
cat /etc/logrotate.d/rsyslog
```

### Edit the Samba Audit Rotation Rule

```bash
sudo nano /etc/logrotate.d/samba-audit
```

### View the Samba Audit Rotation Configuration

```bash
sudo cat /etc/logrotate.d/samba-audit
```
