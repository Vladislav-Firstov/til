# Automate Backups with Cron and Tar

## Why This Matters

Clients run services on Linux servers. If a database crashes, a config gets corrupted, or a deployment goes wrong — they need a way to restore fast. A manual `cp` is not a backup strategy. A **scripted, scheduled, and rotated** backup is.

---

## The Stack

| Tool | Role |
|------|------|
| `tar` | Archive files/folders into a compressed `.tar.gz` |
| `cron` | Schedule the backup to run automatically |
| `find` | Delete archives older than N days (rotation) |

---

## 1. The Backup Script

Create the script at `/usr/local/bin/backup.sh`:

```bash
#!/bin/bash

# === Configuration ===
SOURCE_DIR="/var/www/myapp"          # What to backup
BACKUP_DIR="/backups"                # Where to store
RETENTION_DAYS=7                     # Keep last 7 days
DATE=$(date +%Y-%m-%d_%H-%M-%S)    # Timestamp: 2026-08-14_15-30-00

# === Create archive ===
echo "[$(date)] Starting backup of $SOURCE_DIR..."
tar -czf "$BACKUP_DIR/backup_$DATE.tar.gz" "$SOURCE_DIR"

# === Verify success ===
if [ $? -eq 0 ]; then
    echo "[$(date)] ✅ Backup created: backup_$DATE.tar.gz"
else
    echo "[$(date)] ❌ Backup failed!"
    exit 1
fi

# === Rotate old backups ===
echo "[$(date)] Removing backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] ✅ Done."
```

### Make it executable

```bash
chmod +x /usr/local/bin/backup.sh
```

### Test it manually

```bash
sudo /usr/local/bin/backup.sh
```

---

## 2. Schedule with Cron

Open the system crontab:

```bash
sudo crontab -e
```

Add this line to run daily at 2:00 AM and log output:

```cron
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

| Cron field | Value | Meaning |
|------------|-------|---------|
| Minute | `0` | At minute 0 |
| Hour | `2` | At 2 AM |
| Day | `*` | Every day |
| Month | `*` | Every month |
| Weekday | `*` | Every day of week |

**Redirection explained:**
- `>>` — append output to log file (don't overwrite)
- `2>&1` — redirect stderr (errors) to the same log as stdout

---

## 3. Verify Everything Works

```bash
# Check scheduled jobs
sudo crontab -l

# View backup logs
tail -f /var/log/backup.log

# List recent backups
ls -lh /backups | tail -5

# Check disk usage of backup folder
du -sh /backups
```

---

## 4. Quick Reference: tar Flags

| Flag | Meaning |
|------|---------|
| `-c` | Create archive |
| `-x` | Extract archive |
| `-z` | Compress with gzip |
| `-v` | Verbose (show files) |
| `-f` | Specify filename |
| `-t` | List contents |

---

## 5. Real-World Tips

1. **Always test restores.** A backup you can't restore is useless. Periodically extract an archive and verify the files.
2. **Backup to external storage.** For production, copy archives to S3, rsync to another server, or use `scp`. Local-only backups die with the server.
3. **Use absolute paths** in scripts. Cron runs with a minimal `$PATH`, so always use full paths like `/usr/local/bin/backup.sh`.
4. **Monitor log size.** If something breaks and the script runs every minute, logs can fill the disk. Consider `logrotate` for `/var/log/backup.log`.
5. **Exclude heavy files.** If backing up a web app, exclude `node_modules`, logs, and temp files:
   ```bash
   tar -czf backup.tar.gz --exclude='node_modules' --exclude='*.log' /var/www/myapp
   ```
