# Enhance Backups Crons

On your backup server you can add cron to server to create backup

for example open crontab by the following

```bash
sudo -u pgbackrest crontab -e
```

and add the following commands

```bash
######################################################################################
# Weekly full backup: Thursday at 11:00 PM UTC (Friday 2:00 AM EEST)
# +3:00 CAIRO TIME
#0 23 * * 4 pgbackrest --stanza=cn-backup backup --type=full >> /var/log/pgbackrest/cron_full.log 2>&1
# +2:00 CAIRO TIME
0 0 * * 5 pgbackrest --stanza=cn-backup backup --type=full >> /var/log/pgbackrest/cron_full.log 2>&1

######################################################################################
# Daily incremental backup: Friday–Wednesday at 11:00 PM UTC (Saturday–Thursday 2:00 AM EEST)
# +3:00 CAIRO TIME
#0 23 * * 0-3,5-6 pgbackrest --stanza=cn-backup backup --type=incr >> /var/log/pgbackrest/cron_incr.log 2>&1
# +2:00 CAIRO TIME
0 0 * * 6,0-4 pgbackrest --stanza=cn-backup backup --type=incr >> /var/log/pgbackrest/cron_incr.log 2>&1

######################################################################################
# Intra-day incremental backup: Daily at 11:00 AM UTC (Daily 2:00 PM EEST)
# +3:00 CAIRO TIME
#0 11 * * * pgbackrest --stanza=cn-backup backup --type=incr >> /var/log/pgbackrest/cron_intra.log 2>&1
# +2:00 CAIRO TIME
0 12 * * * pgbackrest --stanza=cn-backup backup --type=incr >> /var/log/pgbackrest/cron_intra.log 2>&1

######################################################################################
# Daily expire: Daily at 12:00 AM UTC (Daily 3:00 AM EEST)
# +3:00 CAIRO TIME
#0 0 * * * pgbackrest --stanza=cn-backup expire >> /var/log/pgbackrest/cron_expire.log 2>&1
# +2:00 CAIRO TIME
0 1 * * * pgbackrest --stanza=cn-backup expire >> /var/log/pgbackrest/cron_expire.log 2>&1
```

---

## Enhance backup

target: Increase backup frequently to decrease data lose on restore

I will do the following

- Full backup every Friday 2:00 AM EEST.
- Daily differential backup at 2:00 AM EEST, except Friday.
- Incremental every 15 minutes, avoiding conflict with full/diff.
- Daily expire run at 3:00 AM EEST.

I will run backup using script instead of running command directly to add custom logic.

I will create custom script

```bash
touch /usr/local/bin/pgbackrest-backup-safe.sh
sudo chown root:root /usr/local/bin/pgbackrest-backup-safe.sh
sudo chmod 755 /usr/local/bin/pgbackrest-backup-safe.sh
chmod +x /usr/local/bin/pgbackrest-backup-safe.sh
```

/usr/local/bin/pgbackrest-backup-safe.sh

```bash
#!/bin/bash

STANZA="cn-backup"
TYPE="incr"

# read backup type if passed (--type=full, diff, incr)
for arg in "$@"; do
    if [[ $arg == --type=* ]]; then
        TYPE="${arg#--type=}"
    fi
done

LOCKFILE="/var/run/pgbackrest_backup.lock"

# If a backup is already running, skip safely
if pgrep -f "pgbackrest.*backup" > /dev/null; then
    echo "$(date '+%F %T') Backup in progress → skipping ($TYPE)" >> /var/log/pgbackrest/cron_skip.log
    exit 0
fi

# create lock file to prevent race condition
if [ -f "$LOCKFILE" ]; then
    echo "$(date '+%F %T') Lock exists → skipping ($TYPE)" >> /var/log/pgbackrest/cron_skip.log
    exit 0
fi

touch "$LOCKFILE"

# run the backup
echo "$(date '+%F %T') Starting $TYPE backup" >> /var/log/pgbackrest/run.log
pgbackrest --stanza="$STANZA" --type="$TYPE" backup >> /var/log/pgbackrest/cron_${TYPE}.log 2>&1

rm -f "$LOCKFILE"
```

now you can run backup script like the following

```bash
/usr/local/bin/pgbackrest-backup-safe.sh --type=incr
/usr/local/bin/pgbackrest-backup-safe.sh --type=full
/usr/local/bin/pgbackrest-backup-safe.sh --type=diff
```

then create crons

```bash
sudo -u pgbackrest crontab -e
```

add the following

```bash
######################################################################################
# WEEKLY FULL – Friday 02:00 EEST (00:00 UTC)
######################################################################################
0 0 * * 5 /usr/local/bin/pgbackrest-safe.sh --type=full >> /var/log/pgbackrest/cron_full.log 2>&1

######################################################################################
# DAILY DIFF – Except Friday – 02:00 EEST (00:00 UTC)
######################################################################################
0 0 * * 6,0-4 /usr/local/bin/pgbackrest-safe.sh --type=diff >> /var/log/pgbackrest/cron_diff.log 2>&1

######################################################################################
# INCREMENTAL EVERY 15 MIN – (skipping on conflicts with full/diff)
######################################################################################
0,15,30,45 * * * * /usr/local/bin/pgbackrest-safe.sh --type=incr >> /var/log/pgbackrest/cron_incr.log 2>&1

######################################################################################
# DAILY EXPIRE – 03:00 EEST (01:00 UTC)
######################################################################################
0 1 * * * pgbackrest --stanza=cn-backup expire >> /var/log/pgbackrest/cron_expire.log 2>&1

```
