# Backup Strategies for Databases

Your database is the heart of your application. Lose it, and you lose everything—customer data, orders, session state, the works. Yesterday we talked about disaster recovery planning. Today we get into the nitty-gritty of how you actually back things up.

Here's the hard truth: most teams don't think about backups until they need one. By then, it's usually too late.

## What's Your RPO and RTO?

Before choosing a backup strategy, you need two numbers:

- **RPO (Recovery Point Objective)** — How much data can you afford to lose? measured in time. 1 hour? 5 minutes? Zero?
- **RTO (Recovery Time Objective)** — How fast do you need to be back online? 30 minutes? 4 hours? A day?

These numbers drive everything else. A cryptocurrency exchange needs RPO of seconds and RTO of minutes. Your internal analytics dashboard? Maybe an RPO of 6 hours and RTO of a day is fine.

```
      RPO ↓ (more frequent backups) → Higher cost
      RTO ↓ (faster recovery) → Higher complexity
      
      Pick your pain: data loss vs. recovery time vs. budget
```

## Backup Types

### Full Backup

The entire database, every single byte. Simple, reliable, slow as hell.

```
✅ Complete snapshot — restore from a single point
✅ Simple to verify — what you see is what you get
❌ Takes forever for large databases
❌ Huge storage cost per backup
```

Full backups are your foundation. You take them weekly or daily, then complement with lighter options.

### Incremental Backup

Only the data that changed since the last *any* backup (full or incremental).

```
✅ Fast — only changed blocks/rows
✅ Small storage footprint
❌ Restore is slow — you need full + all incrementals
❌ One corrupted incremental = chain broken
```

### Differential Backup

Everything that changed since the last *full* backup.

```
✅ Faster than full, slower than incremental
✅ Restore needs only full + latest differential
❌ Gets bigger each day until the next full backup
```

### Visual Comparison

```
Monday (Full):   ████████████████████████████████
Tuesday:         ██ (incremental)  vs  ██████ (differential)
Wednesday:       ██ (incremental)  vs  ████████████ (differential)
Thursday:        ██ (incremental)  vs  ██████████████████ (differential)

Restore Friday:
- Incremental: Full + inc(Tue) + inc(Wed) + inc(Thu) = 4 steps
- Differential: Full + diff(Thu) = 2 steps
```

**Trade-off:** Incrementals save space but slow recovery. Differentials trade space for speed.

## Transaction Log Backups

This is where things get interesting. Most major databases (PostgreSQL WAL, MySQL binlog, SQL Server transaction log) let you stream every single change as it happens.

A transaction log backup captures the log files that record all transactions:

```sql
-- PostgreSQL WAL archiving
archive_mode = on
archive_command = 'cp %p /backups/wal/%f'

-- MySQL binary log retention
SET GLOBAL expire_logs_days = 7;

-- SQL Server log backup
BACKUP LOG MyDatabase TO DISK = 'E:\Backups\mydb_log.trn';
```

**Why this matters:** With full backups + transaction logs, you can do point-in-time recovery (PITR). Lose data at 3:47 PM? Restore last full backup + replay logs up to 3:46 PM. You're back with nearly zero data loss.

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Full DB │────▶│  Log 1   │────▶│  Log 2   │────▶│  Log 3   │
│  Backup  │     │ 09:00    │     │ 10:00    │     │ 10:30    │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                                          │
                                                    Restore to
                                                    10:15:37
```

## Cloud-Native Approaches

If you're on a managed database service, you don't need to cobble together scripts:

| Service | Built-in Backup | PITR | Retention | Restore Time |
|---------|----------------|------|-----------|--------------|
| RDS (PostgreSQL/MySQL) | Automated daily | ✅ Up to 35 days | Configurable | ~1hr per 100GB |
| Aurora | Continuous to S3 | ✅ Up to 35 days | Automated | ~10 minutes (any size) |
| Cloud SQL | Automated daily | ✅ Up to 365 days (PITR) | Configurable | ~30m per 100GB |
| Cosmos DB | Continuous (every 100s) | ✅ Up to 30 days | Automatic | Configurable |
| DynamoDB | On-demand/continuous | ✅ Up to 35 days (PITR) | Configurable | Variable |

**Reality check:** Built-in backups are convenient but you're locked into the cloud provider. Restoring 5TB on RDS can take hours, and you're paying for the restored instance while it warms up.

## Physical vs. Logical Backups

### Physical Backup
Copy the raw database files:

```bash
# PostgreSQL - pg_basebackup (physical)
pg_basebackup -D /backups/full_$(date +%Y%m%d) -X stream -v

# MySQL - direct file copy with LOCK TABLES
mysql -e "FLUSH TABLES WITH READ LOCK;"
tar czf /backups/mysql_$(date +%Y%m%d).tar.gz /var/lib/mysql
mysql -e "UNLOCK TABLES;"
```

**Pros:** Fast, compact, includes indexes and config.
**Cons:** Only restores to same DB version, same platform.

### Logical Backup
Export data as SQL or other format:

```bash
# PostgreSQL - pg_dump (logical)
pg_dump -Fc mydb > /backups/mydb_$(date +%Y%m%d).dump

# MySQL - mysqldump
mysqldump --single-transaction mydb | gzip > /backups/mydb_$(date +%Y%m%d).sql.gz

# MongoDB
mongodump --db mydb --gzip --out /backups/$(date +%Y%m%d)
```

**Pros:** Portable across versions and platforms, can restore single tables.
**Cons:** Slower for large databases, larger output.

## The 3-2-1 Rule

If you remember one thing from this article, remember this:

> **3 copies of your data, on 2 different media types, with 1 copy off-site.**

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  3 Copies                                           │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │ Primary  │   │ Local    │   │ Off-site │        │
│  │ Database │   │ Backup   │   │ Backup   │        │
│  └──────────┘   └──────────┘   └──────────┘        │
│                        ↓              ↓             │
│                     2 Media         1 Off-site      │
│                    (SSD + S3)      (S3/Different    │
│                                      Region)        │
└─────────────────────────────────────────────────────┘
```

**Real-world example for a startup on AWS:**
1. **Copy 1:** RDS running production (live data)
2. **Copy 2:** RDS automated snapshots + WAL (stored in S3, same region)
3. **Copy 3:** Cross-region replication of snapshots to `us-west-2` + daily `pg_dump` to S3 Glacier

Now when `us-east-1` goes down (it happens), you can spin up from the `us-west-2` snapshot.

## Automate and Monitor

A backup that hasn't been tested is not a backup. It's a security blanket.

```bash
#!/bin/bash
# Simple backup automation (not production-grade, but shows the pattern)
set -euo pipefail

DB_NAME="myapp"
BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
S3_BUCKET="s3://myapp-backups"

# Step 1: Take the backup
pg_dump -Fc "$DB_NAME" > "$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump"

# Step 2: Upload to S3
aws s3 cp "$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump" "$S3_BUCKET/daily/"

# Step 3: Clean up old local backups (keep 7 days)
find "$BACKUP_DIR" -name "${DB_NAME}_*.dump" -mtime +7 -delete

# Step 4: Alert if something went wrong
if [ $? -eq 0 ]; then
  curl -H "Content-Type: application/json" \
       -d '{"text":"Backup completed successfully"}' \
       "$SLACK_WEBHOOK_URL"
else
  curl -H "Content-Type: application/json" \
       -d '{"text":"⚠️ BACKUP FAILED - check the server"}' \
       "$SLACK_WEBHOOK_URL"
fi
```

### What to Monitor

| Metric | Why It Matters | Alert When |
|--------|---------------|------------|
| Backup duration | Slower = possible issues | >2x normal time |
| Backup file size | Drastic changes = data loss or corruption | ±20% from avg |
| WAL/log shipping lag | Missing logs = no PITR | Lag > 30 min |
| Restore test success | The backup might be corrupt | Any failure |
| Storage capacity | Can't write backups | <20% free space |

## Restore Testing

Here's the dirty secret: most teams say they have backups. Few actually test their restore process end-to-end.

**Once a quarter:** spin up a fresh database instance, restore from the oldest full backup you have, replay logs, and verify the data is consistent. Record the time it takes. That's your real RTO.

```bash
# Restore test checklist
□ Create new DB instance (same version as production)
□ Restore latest full backup
□ Apply transaction logs up to a specific timestamp
□ Run integrity checks: ANALYZE, CHECK constraints, data sample
□ Verify application connects and queries work
□ Document actual time taken
□ Document any failures and fix them
```

## Which Strategy Should You Pick?

| Scenario | RPO | RTO | Recommended Strategy |
|----------|-----|-----|-------------------|
| Fintech/payments | Seconds | Minutes | Synchronous replication + continuous WAL + cross-region |
| SaaS (standard) | 1 hour | 4 hours | Daily full + differential + hourly logs |
| Internal tool | 24 hours | 24 hours | Daily full backup only |
| Analytics/warehouse | 6 hours | 12 hours | Periodic snapshot + data source replay |
| Dev/test | None | 48 hours | Weekly full backup |

## The Bottom Line

- Know your RPO and RPO before picking tools
- Full + incremental + transaction logs gives you the best flexibility
- 3-2-1 rule isn't optional — that off-site copy saves you when the office floods
- Cloud provider backups are convenient but test your restore anyway
- Automate the process and monitor it
- **Test your restores.** Seriously. Do it this week.

Backups are boring until they're not. When production goes down at 2 AM, you'll be very glad you spent the time getting this right.
