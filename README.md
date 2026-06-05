# Backup-Restore-Automation-Project
A homelab project simulating a real-world backup and restore system
# 🗄️ Backup & Restore Automation Project — NovaBridge Consulting

A homelab project simulating a real-world backup and restore system for a small consulting firm. Built on Ubuntu 22.04 using `rsync`, `bash scripting`, `cron`, and `logrotate`.

---

## 📋 Table of Contents

- [Project Scenario](#project-scenario)
- [Skills Gained](#skills-gained)
- [Project Structure](#project-structure)
- [Phase 1 — Rsync Backup Script](#phase-1--rsync-backup-script)
- [Phase 2 — Scheduling with Cron](#phase-2--scheduling-with-cron)
- [Phase 3 — File Versioning](#phase-3--file-versioning)
- [Screenshots](#screenshots)
- [Key Lessons Learned](#key-lessons-learned)

---

## 🏢 Project Scenario

**Organisation:** NovaBridge Consulting
**Role:** Sole System Administrator
**Environment:** Ubuntu 22.04 (single VM homelab)
**Goal:** Design and implement a full backup and restore system to protect client reports, financial records, and project files.

| Detail | Value |
|---|---|
| Data protected | Client files, finance records, project files, configs |
| Backup destination | Local disk (`/backup/`) |
| Recovery Time Objective (RTO) | Under 2 hours |
| Retention policy | 30 days of versioned snapshots |

---

## 🛠️ Skills Gained

- Writing backup scripts in `bash`
- Using `rsync` flags including `-a`, `--delete`, `--stats`, and `--link-dest`
- Scheduling automated jobs with `cron` and `crontab`
- Setting up log rotation with `logrotate`
- File versioning using dated snapshot folders and hardlinks
- Building a retention policy to manage disk space
- Simulating a ransomware attack and restoring from a clean snapshot
- Linux file permissions with `chmod` and `chown`
- Debugging bash scripts using `bash -x`

---

## 📁 Project Structure

```
/
├── data/
│   └── novabridge/              # Simulated live company data
│       ├── clients/
│       ├── finance/
│       ├── projects/
│       └── configs/
├── backup/
│   ├── local/                   # Phase 1 — mirror backup
│   │   └── novabridge/
│   └── snapshots/               # Phase 3 — versioned snapshots
│       ├── 2026-05-31/
│       └── 2026-06-01/
├── usr/
│   └── local/
│       └── bin/
│           ├── backup.sh        # Phase 1 & 2 — mirror backup script
│           ├── snapshot.sh      # Phase 3 — versioned snapshot script
│           └── retention.sh     # Phase 3 — retention cleanup script
└── var/
    └── log/
        ├── novabridge_backup.log    # Main log file
        └── novabridge_alerts.log    # Failure alerts log file
```

---

## Phase 1 — Rsync Backup Script

### What it does
Creates a mirror copy of `/data/novabridge/` to `/backup/local/novabridge/` using `rsync`. Every run is logged with a timestamp. If the backup fails, the exit code is captured and an error is written to the log.

### Script — `backup.sh`

```bash
SOURCE="/data/novabridge/"
DEST="/backup/local/novabridge/"
LOGFILE="/var/log/novabridge_backup.log"
ALERTFILE="/var/log/novabridge_alerts.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Backup started" >> "$LOGFILE"

rsync -a --delete --stats "$SOURCE" "$DEST" >> "$LOGFILE" 2>&1
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
  echo "[$DATE] Backup completed successfully" >> "$LOGFILE"
else
  echo "[$DATE] ERROR: Backup failed with exit code $EXIT_CODE" >> "$LOGFILE"
  echo "[$DATE] ALERT: NovaBridge backup failed! Check $LOGFILE" >> "$ALERTFILE"
fi

echo "----------------------------------------" >> "$LOGFILE"
```

### Key rsync flags explained

| Flag | Purpose |
|---|---|
| `-a` | Archive mode — preserves permissions, timestamps, and ownership |
| `--delete` | Keeps backup as a true mirror by removing deleted files |
| `--stats` | Prints a transfer summary to the log |
| `2>&1` | Redirects error messages into the log file |

### How to run

```bash
sudo bash /usr/local/bin/backup.sh
```

### Tests performed

| Test | Method | Expected Result |
|---|---|---|
| Success | Delete a file, run script, restore from backup | File restored successfully |
| Failure detection | Break SOURCE path, run script | Error logged in alerts file |
| Delta sync | Run script twice with no changes | 0 files transferred on second run |

### Screenshots

> 📸 _Add a screenshot of your terminal showing a successful backup run here_

> 📸 _Add a screenshot of `/var/log/novabridge_backup.log` showing timestamped entries here_

> 📸 _Add a screenshot of the failure alert in `/var/log/novabridge_alerts.log` here_

---

## Phase 2 — Scheduling with Cron

### What it does
Automates the backup script to run every night at 2:00am without any manual intervention. Log rotation is configured to archive logs weekly and keep 4 weeks of history.

### Cron entry

```
0 2 * * * bash /usr/local/bin/backup.sh
```

### Cron syntax breakdown

```
0    2    *    *    *
│    │    │    │    └── day of week (0=Sunday)
│    │    │    └─────── month (1-12)
│    │    └──────────── day of month (1-31)
│    └───────────────── hour (0-23)
└────────────────────── minute (0-59)
```

### Logrotate config — `/etc/logrotate.d/novabridge`

```
/var/log/novabridge_backup.log /var/log/novabridge_alerts.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
```

### Tests performed

| Test | Method | Expected Result |
|---|---|---|
| Cron fires automatically | Set schedule to every minute, watch log | New log entry appears within 60 seconds |
| Failure alert via cron | Break SOURCE path, wait for cron | Alert written to alerts log automatically |
| Logrotate config valid | Run `logrotate --debug` | No errors in output |

### Screenshots

> 📸 _Add a screenshot of `sudo crontab -l` showing your scheduled entry here_

> 📸 _Add a screenshot of the log updating automatically during cron testing here_

---

## Phase 3 — File Versioning

### What it does
Creates a new dated snapshot folder every night instead of overwriting a single backup. Uses `rsync --link-dest` to hardlink unchanged files to the previous snapshot, saving disk space. A retention script automatically deletes snapshots older than 30 days.

### Script — `snapshot.sh`

```bash
SOURCE="/data/novabridge/"
SNAPDIR="/backup/snapshots"
LOGFILE="/var/log/novabridge_backup.log"
ALERTFILE="/var/log/novabridge_alerts.log"
DATE=$(date '+%Y-%m-%d')
TODAY="$SNAPDIR/$DATE"
YESTERDAY=$(date -d 'yesterday' '+%Y-%m-%d')
LASTSNAP="$SNAPDIR/$YESTERDAY"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Snapshot started: $TODAY" >> "$LOGFILE"

mkdir -p "$TODAY"

if [ -d "$LASTSNAP" ]; then
  rsync -a --link-dest="$LASTSNAP" "$SOURCE" "$TODAY/" >> "$LOGFILE" 2>&1
else
  rsync -a "$SOURCE" "$TODAY/" >> "$LOGFILE" 2>&1
fi

EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Snapshot completed: $TODAY" >> "$LOGFILE"
else
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: Snapshot failed exit code $EXIT_CODE" >> "$LOGFILE"
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] ALERT: Snapshot failed! Check $LOGFILE" >> "$ALERTFILE"
fi

echo "----------------------------------------" >> "$LOGFILE"
```

### Script — `retention.sh`

```bash
SNAPDIR="/backup/snapshots"
LOGFILE="/var/log/novabridge_backup.log"
KEEP_DAYS=30
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Retention cleanup started" >> "$LOGFILE"

find "$SNAPDIR" -maxdepth 1 -type d -mtime +$KEEP_DAYS -exec rm -rf {} \;

echo "[$DATE] Retention cleanup completed" >> "$LOGFILE"
echo "----------------------------------------" >> "$LOGFILE"
```

### Cron entries for Phase 3

```
0 2 * * * bash /usr/local/bin/snapshot.sh
30 2 * * * bash /usr/local/bin/retention.sh
```

### Hardlinks explained

Hardlinks allow multiple snapshot folders to reference the same file on disk. Only files that have changed between snapshots take up new space. This makes 30 days of history storage-efficient.

```
2026-05-31/finance/q2_budget.xlsx  ──┐
                                     ├── same file on disk (one copy)
2026-06-01/finance/q2_budget.xlsx  ──┘  (file unchanged — hardlinked)

2026-06-01/clients/new_report.pdf  ──── new file on disk (changed — new copy)
```

### Ransomware drill

Simulated a ransomware attack by overwriting all files with corrupted content, then restored from the previous day's clean snapshot.

```bash
# Simulate attack
sudo bash -c 'find /data/novabridge -type f -exec sh -c "echo ENCRYPTED > {}" \;'

# Restore from clean snapshot
sudo rsync -a /backup/snapshots/2026-05-31/ /data/novabridge/
```

### Tests performed

| Test | Method | Expected Result |
|---|---|---|
| Snapshot creates dated folder | Run snapshot.sh, check /backup/snapshots/ | Folder named with today's date appears |
| Hardlinks save disk space | Run `du -sh /backup/snapshots/` | Total size close to a single backup |
| Retention deletes old snapshots | Create 40-day-old folder, run retention.sh | Old folder deleted, recent ones kept |
| Ransomware recovery | Corrupt files, restore from snapshot | All files restored to clean state |

### Screenshots

> 📸 _Add a screenshot of `ls /backup/snapshots/` showing multiple dated folders here_

> 📸 _Add a screenshot of the corrupted files showing ENCRYPTED here_

> 📸 _Add a screenshot after restore showing files recovered successfully here_

> 📸 _Add a screenshot of `du -sh /backup/snapshots/` showing disk usage here_

---

## 💡 Key Lessons Learned

- **The trailing slash in rsync matters** — `rsync source/ dest/` copies contents. Without the slash it copies the folder itself, creating nested directories.
- **`--delete` is a double-edged sword** — essential for a true mirror in Phase 1, but dropped in Phase 3 because versioning requires keeping deleted files in old snapshots.
- **Bash is case sensitive** — a variable named `TODAY` and one named `Today` are completely different. All variable names in these scripts use CAPITALS.
- **Always use `sudo` for system directories** — `/var/log/`, `/backup/`, and `/usr/local/bin/` all require root permission to write to.
- **`touch` before `chmod`** — you cannot set permissions on a file that does not exist yet.
- **Test failure, not just success** — a backup system you have not tested failing is just hope. Breaking the SOURCE path deliberately proved the alert system works.
- **`bash -x` is your best debugging tool** — it prints every line of a script as it runs and shows the real value of every variable, making bugs easy to spot.

---

## 🖥️ Environment

| Component | Detail |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Shell | Bash |
| Backup tool | rsync 3.2.7 |
| Scheduler | cron |
| Log management | logrotate |
| VM type | Homelab (single server) |

---

*Project completed as part of a homelab System Administrator skills programme — Phases 1, 2, and 3.*
