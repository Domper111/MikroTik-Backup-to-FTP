# MikroTik Backup to FTP

Automated weekly backup of MikroTik RouterOS configuration (`.backup` + `.rsc`)
uploaded to an FTP server, organized into date-based folders (`YYYY-MM-DD`).

Supports **RouterOS 7.x** and **RouterOS 6.x** with dedicated scripts, plus a
PowerShell helper for the FTP server side.

---

## Table of contents

- [Features](#features)
- [Requirements](#requirements)
- [Repository layout](#repository-layout)
- [Which script to use](#which-script-to-use)
- [RouterOS 7.x script](#routeros-7x-script)
- [RouterOS 6.x script](#routeros-6x-script)
- [Server-side helper (PowerShell)](#server-side-helper-powershell)
- [Installation](#installation)
- [Schedule](#schedule)
- [Verification](#verification)
- [Security notes](#security-notes)
- [Known limitations](#known-limitations)
- [License](#license)

---

## Features

- Creates both binary `.backup` (password-protected) and text `.rsc` export
- Uploads both files to an FTP server under a `YYYY-MM-DD` folder
- Removes local files after successful upload
- Detailed logging with numbered steps (`BACKUP: 1 ... 10`)
- Weekly scheduler (every 7 days)
- NTP configuration included
- Idempotent server-side folder creation via PowerShell
- Never deletes old backups — retention is left to your own policy

---

## Requirements

- RouterOS 7.x → use `backup-to-fs-v7.rsc`
- RouterOS 6.x → use `backup-to-fs-v6.rsc`
- Script policy: `read,write,test,sensitive,ftp`
- **RouterOS 7.x only:** Device-mode must allow `fetch` (see [Security notes](#security-notes)).
- FTP server reachable from the router
- NTP configured (correct clock is required for date-based folders)

---

## Repository layout

```
mikrotik-backup/
├── README.md
├── LICENSE
├── .gitignore
├── scripts/
│   ├── backup-to-fs-v7.rsc        # RouterOS 7.x
│   ├── backup-to-fs-v6.rsc        # RouterOS 6.x
│   └── server/
│       └── create-backup-folder.ps1
└── examples/
    └── config.example.rsc
```

---

## Which script to use

| RouterOS version | Script                  | NTP method                                  | Filename format                  | Date folder parsing                          |
|------------------|-------------------------|---------------------------------------------|----------------------------------|----------------------------------------------|
| **7.x**          | `backup-to-fs-v7.rsc`   | `/system ntp client servers add/remove`     | `<identity>_<remoteDir>`         | String replace on `/system clock get date`   |
| **6.x**          | `backup-to-fs-v6.rsc`   | `primary-ntp` / `secondary-ntp`             | `FFFFFFFFFFFFFFF`                | Manual month name mapping (`jan`..`dec`)     |

> **Note:** The v6 script uses a fixed filename `FFFFFFFFFFFFFFF` instead of the identity name. This is a workaround for the RouterOS 6 FTP client bug where paths with nested folders fail if the filename contains dots (e.g., in a router identity like `cow.ny.112.W3`). If your identity has no dots, you can safely change `:local base "FFFFFFFFFFFFFFF";` to `:local devName [/system identity get name]; :local base ($devName . "_" . $remoteDir);`.

---

## RouterOS 7.x script

**File:** `scripts/backup-to-fs-v7.rsc`

**Purpose:** Backup RouterOS 7.x configuration and upload to FTP.

**Requires:** RouterOS 7.x, policy `read,write,test,sensitive,ftp`, NTP configured.

**Placeholders to replace before import:**

| Placeholder       | Meaning                            |
|-------------------|------------------------------------|
| `<FS_HOST>`       | FTP server IP address              |
| `<FS_USER>`       | FTP username                       |
| `<FS_PASS>`       | FTP password                       |
| `<BACKUP_PASS>`   | Password for the `.backup` archive |
| `<NTP_PRIMARY>`   | Primary NTP server                 |
| `<NTP_SECONDARY>` | Secondary NTP server               |

**Key behavior:**

- Uses `/system ntp client servers add` (multi-server syntax available in 7.x).
- Builds `remoteDir` by replacing `/` with `-` in the router's date string.
- Filename: `<identity>_<remoteDir>` (e.g. `MikroTik_2026-10-28`).

**Run:**

```
/system script run backup-to-fs
```

---

## RouterOS 6.x script

**File:** `scripts/backup-to-fs-v6.rsc`

**Purpose:** Backup RouterOS 6.x configuration and upload to FTP.

**Requires:** RouterOS 6.x, policy `read,write,test,sensitive,ftp`, NTP configured.

**Placeholders to replace before import:**

| Placeholder       | Meaning                            |
|-------------------|------------------------------------|
| `<FS_HOST>`       | FTP server IP address              |
| `<FS_USER>`       | FTP username                       |
| `<FS_PASS>`       | FTP password                       |
| `<BACKUP_PASS>`   | Password for the `.backup` archive |
| `<NTP_PRIMARY>`   | Primary NTP server                 |
| `<NTP_SECONDARY>` | Secondary NTP server               |

**Key behavior:**

- Uses legacy NTP syntax: `/system ntp client set enabled=yes primary-ntp=... secondary-ntp=...`
- Parses the date manually: extracts year, day, and maps `jan..dec` → `01..12` → builds `remoteDir` as `YYYY-MM-DD`.
- Uses a fixed filename `FFFFFFFFFFFFFFF` (see note above).

**Run:**

```
/system script run backup-to-fs
```

---

## Server-side helper (PowerShell)

**File:** `scripts/server/create-backup-folder.ps1`

**Purpose:** Pre-create the `YYYY-MM-DD` folder on the FTP server so the router's upload never fails due to a missing directory.

**Parameters:**

| Parameter    | Default | Meaning                            |
|--------------|---------|------------------------------------|
| `RootPath`   | —       | Root directory for date folders    |
| `DateOffset` | `0`     | Days offset from today (0 = today) |

**Examples:**

```powershell
# Create today's folder
.\create-backup-folder.ps1 -RootPath "D:\test"

# Pre-create tomorrow's folder
.\create-backup-folder.ps1 -RootPath "D:\test" -DateOffset 1
```

**Schedule example (Windows Task Scheduler, daily at 02:55):**

```text
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File "C:\scripts\create-backup-folder.ps1" -RootPath "D:\test"
```

**Notes:**

- Idempotent — safe to run repeatedly (`-Force` + `Test-Path` check).
- Never deletes old backups; retention is left to your own policy.
- Returns exit code `0` on success, `1` on failure.

---

## Installation

1. Choose the script for your RouterOS version (see table above).
2. Copy it to the router (WinBox → Files, or SCP/SFTP).
3. Replace all placeholders **before** importing.
4. Import it:
   ```
   /import file-name=backup-to-fs-v7.rsc
   ```
   (or `backup-to-fs-v6.rsc` for RouterOS 6.x)
5. Run once manually to verify (see [Verification](#verification)).
6. Set up the PowerShell helper on the FTP server if the target folder must exist before upload.

---

## Schedule

Both scripts create the same scheduler:

- **Name:** `weekly-backup-fs`
- **Start:** `oct/28/2026 03:00:00`
- **Interval:** `7d` (every Wednesday)
- **Comment:** `Backup every Wednesday starting 28 Oct 2026`

Adjust `start-date` / `start-time` as needed. Note that `start-date` pins the first run; the interval then repeats every 7 days from that point.

---

## Verification

After a manual run:

```
/log print where message~"BACKUP"
```

Expected sequence (for v7):

```
BACKUP: 1. Script started
BACKUP: 2. Filename: <identity>_<YYYY-MM-DD>
BACKUP: 3. Destination folder: <YYYY-MM-DD>
BACKUP: 4. Files are being created, waiting 3 seconds...
BACKUP: 5. Local file .backup created successfully
BACKUP: 6. Local file .rsc created successfully
BACKUP: 7. Uploading .backup to path: ...
BACKUP: 8. Uploading .rsc to path: ...
BACKUP: 9. Success! Removing local files
BACKUP: 10. Script completed successfully!
```

Check the FTP server: files should appear under `RootPath/YYYY-MM-DD/`:

```
<identity>_YYYY-MM-DD.backup
<identity>_YYYY-MM-DD.rsc
```

Check the scheduler:

```
/system scheduler print where name="weekly-backup-fs"
```

---

## Security notes

- **Never commit real credentials.** This repository uses placeholders only.
- Keep a local copy with real values outside the repo (see `.gitignore`).
- Use a dedicated FTP user with write access limited to the backup directory.
- Consider FTPS/SFTP if your RouterOS version and server support it.
- The `.backup` archive is password-protected via `<BACKUP_PASS>`; the `.rsc`
  export is **plain text** — treat it as sensitive (it contains config, and
  potentially secrets embedded in `/export`).
- **RouterOS 7.x Device-mode:** If `fetch` is blocked (`/system/device-mode/print` shows `fetch: no`), you must enable it physically:
  ```
  /system/device-mode/update fetch=yes
  ```
  Then press the Reset button or power-cycle the router within 5 minutes.

---

## Known limitations

- `:delay 3s` is an empirical wait for file creation; not a guarantee.
- FTP upload result is not checked — failed uploads are not logged as errors.
- Date folder format in the 7.x script depends on the router's date format
  setting; the 6.x script's manual parsing is format-independent.
- `start-date` pins the schedule to a fixed date; verify it is still relevant.
- PowerShell helper creates folders only — it does not manage retention.
- **RouterOS 6.x FTP bug:** The built-in FTP client in RouterOS 6 may ignore nested directories in `dst-path` or fail when the filename contains dots. The v6 script uses a fixed filename and a leading slash (`dst-path="/YYYY-MM-DD/file.backup"`) as a workaround.

---

## License

MIT — see [LICENSE](LICENSE).
