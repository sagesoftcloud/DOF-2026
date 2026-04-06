# DOF AWS Reliability Pillar — Session 4 Presentation
## Monitoring & Change Management
**REL 6: Monitor Workload Resources | REL 8: Implement Change Management**

---

## Agenda

1. What is the Reliability Pillar?
2. Why Monitoring Matters (REL 6)
3. How to Set Up Monitoring — Step by Step
4. Why Change Management Matters (REL 8)
5. How to Set Up a Staging Environment — Step by Step
6. Best Practices Summary
7. Next Steps

---

## 1. What is the Reliability Pillar?

The Reliability pillar ensures your workload **performs its intended function correctly and consistently**. It covers:

- Can your system recover from failures?
- Can you detect problems before users do?
- Can you make changes safely without breaking production?

**DOF's current reliability gaps:**

| Area | Current Risk |
|------|-------------|
| Monitoring | No alarms — CPU was at 99% for days before anyone noticed (March 27 incident) |
| Change Management | No staging — updates applied directly to production, causing downtime |
| Backups | No automated backups — data loss risk |
| High Availability | Single instance per tier — one failure = full outage |

**Today's focus:** Monitoring (REL 6) and Change Management (REL 8) — the two highest priority items.

---

## 2. Why Monitoring Matters

### REL 6: How do you monitor workload resources?

> **AWS Best Practice:** "Monitor all components of the workload to detect failures. Use Amazon CloudWatch to set alarms for key metrics and receive notifications when thresholds are breached."
>
> Reference: [AWS Well-Architected — REL 6](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_monitor_monitor_workload.html)

### What Happened Without Monitoring (March 27 Incident)

```
March 22: DB server CPU starts hitting 99%
March 23: Still at 99% — nobody knows
March 24: Still at 99% — nobody knows
March 25: Still at 99% — nobody knows
March 26: Still at 99% — nobody knows
March 27: Users report wp-admin is down — emergency meeting called
```

**5 days of degraded performance before detection.**

### What Would Happen With Monitoring

```
March 22: DB server CPU hits 80%
           → CloudWatch alarm triggers
           → Email sent to DOF and SSI: "CPU HIGH on PROD-DOF-EC2-DOFWEB-DB-01"
           → Team investigates immediately
           → Issue resolved same day
```

### What to Monitor

| Metric | What It Tells You | Alarm Threshold |
|--------|------------------|-----------------|
| **CPUUtilization** | How busy the server is | > 80% for 5 minutes |
| **EBSReadOps / EBSWriteOps** | Disk I/O activity | Approaching IOPS limit |
| **VolumeQueueLength** | Disk I/O backlog | > 1 for 5 minutes |
| **NetworkIn / NetworkOut** | Network traffic | Unusual spikes |
| **StatusCheckFailed** | Instance health | Any failure |
| **disk_used_percent** | Disk space remaining | > 85% |
| **mem_used_percent** | Memory usage | > 85% |

### Three Components of Monitoring

```
CloudWatch Metrics     →  Raw data (CPU %, disk I/O, etc.)
         ↓
CloudWatch Alarms      →  "Alert me when CPU > 80%"
         ↓
SNS Notifications      →  Email/SMS sent to your team
```

---

## 3. How to Set Up Monitoring — Step by Step

### Step 1: Install CloudWatch Agent (for memory and disk metrics)

By default, CloudWatch only collects CPU and network metrics. To get **memory** and **disk** metrics, you need the CloudWatch Agent installed on each EC2 instance.

**On the Web Server (PROD-EC2-NEW-DOFWEB-APP-01):**

```bash
# Download and install CloudWatch Agent
sudo yum install -y amazon-cloudwatch-agent

# Create configuration file
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

**Configuration choices:**
- OS: Linux
- Collect metrics: Yes
- Metrics to collect: CPU, memory, disk
- Collection interval: 60 seconds
- Log files to collect: Apache access/error logs (optional)

**Repeat for the DB Server (PROD-DOF-EC2-DOFWEB-DB-01).**

> **Best Practice:** Install the CloudWatch Agent on every EC2 instance to get full visibility into memory and disk usage — metrics that are not available by default.

---

### Step 2: Create a CloudWatch Dashboard

A dashboard gives you a **single screen** to see the health of your entire website.

**Via Console:**

1. Go to **CloudWatch** → **Dashboards** → **Create dashboard**
2. Name: `DOF-Website-Dashboard`
3. Add widgets:

**Widget 1: Web Server CPU**
- Type: Line graph
- Metric: EC2 → Per-Instance → CPUUtilization
- Instance: `PROD-EC2-NEW-DOFWEB-APP-01`

**Widget 2: DB Server CPU**
- Type: Line graph
- Metric: EC2 → Per-Instance → CPUUtilization
- Instance: `PROD-DOF-EC2-DOFWEB-DB-01`

**Widget 3: ALB Health**
- Type: Number
- Metrics:
  - HealthyHostCount
  - UnHealthyHostCount

**Widget 4: ALB Request Count & Latency**
- Type: Line graph
- Metrics:
  - RequestCount
  - TargetResponseTime

**Widget 5: EBS IOPS (Web Server)**
- Type: Line graph
- Metrics:
  - VolumeReadOps
  - VolumeWriteOps

**Widget 6: Memory & Disk (after CloudWatch Agent is installed)**
- Type: Line graph
- Metrics:
  - mem_used_percent
  - disk_used_percent

> **Best Practice:** Create a single dashboard that shows all critical metrics. Share the dashboard URL with the team so anyone can check system health without SSH access.

---

### Step 3: Create CloudWatch Alarms

Alarms automatically notify you when something goes wrong.

**Via Console:**

1. Go to **CloudWatch** → **Alarms** → **Create alarm**

**Alarm 1: High CPU — Web Server**
- Metric: CPUUtilization for `PROD-EC2-NEW-DOFWEB-APP-01`
- Condition: Greater than 80%
- Period: 5 minutes
- Datapoints: 1 out of 1
- Action: Send to SNS topic `DOF-Alerts`

**Alarm 2: High CPU — DB Server**
- Metric: CPUUtilization for `PROD-DOF-EC2-DOFWEB-DB-01`
- Condition: Greater than 80%
- Period: 5 minutes
- Action: Send to SNS topic `DOF-Alerts`

**Alarm 3: ALB Unhealthy Targets**
- Metric: UnHealthyHostCount
- Condition: Greater than 0
- Period: 1 minute
- Action: Send to SNS topic `DOF-Alerts`

**Alarm 4: EBS Queue Length (Web Server)**
- Metric: VolumeQueueLength for web server volume
- Condition: Greater than 1
- Period: 5 minutes
- Action: Send to SNS topic `DOF-Alerts`

**Alarm 5: Disk Space (after CloudWatch Agent)**
- Metric: disk_used_percent
- Condition: Greater than 85%
- Period: 5 minutes
- Action: Send to SNS topic `DOF-Alerts`

> **Best Practice:** Set alarms at warning thresholds (80%), not critical thresholds (99%). This gives you time to respond before users are affected.

---

### Step 4: Create SNS Topic for Notifications

SNS (Simple Notification Service) sends the alarm notifications to your team.

**Via Console:**

1. Go to **SNS** → **Topics** → **Create topic**
2. Type: Standard
3. Name: `DOF-Alerts`
4. Create topic
5. **Create subscription:**
   - Protocol: Email
   - Endpoint: DOF team email (e.g., distribution list)
   - Create subscription
6. **Add more subscriptions** for SSI team emails
7. Each subscriber must **confirm** via the email they receive

> **Best Practice:** Use a distribution list email so multiple people receive alerts. Don't rely on a single person's email.

---

## 4. Why Change Management Matters

### REL 8: How do you implement change?

> **AWS Best Practice:** "Use runbooks for standard activities such as deployment. Test changes in a non-production environment before deploying to production."
>
> Reference: [AWS Well-Architected — REL 8](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_manage_change_implement_change.html)

### What Happened Without Change Management (March 27 Incident)

```
DOF needs to apply WordPress security updates (VA compliance)
    ↓
No staging environment available
    ↓
Updates applied directly to production
    ↓
Database schema changes trigger heavy processing
    ↓
MariaDB CPU spikes to 99%
    ↓
wp-admin becomes inaccessible
    ↓
Developer runs mysqlcheck --optimize on production to recover
    ↓
Temporary relief → cycle repeats
```

### What Would Happen With Change Management

```
DOF needs to apply WordPress security updates (VA compliance)
    ↓
Updates applied to STAGING environment first
    ↓
Test: Does the site work? Is CPU normal? Any errors?
    ↓
If YES → Promote the same changes to production (safe)
If NO  → Fix the issue in staging, production is unaffected
```

---

## 5. How to Set Up a Staging Environment — Step by Step

### Step 1: Create AMI of Production Web Server

An AMI (Amazon Machine Image) is an exact copy of your server.

**Via Console:**

1. Go to **EC2** → **Instances**
2. Select `PROD-EC2-NEW-DOFWEB-APP-01`
3. **Actions** → **Image and templates** → **Create image**
4. Name: `DOF-Web-Staging-AMI-YYYYMMDD`
5. Click **Create image**
6. Wait for AMI to be available (~5-10 minutes)

> **Best Practice:** Always create an AMI before making any changes to production. This is your rollback point.

---

### Step 2: Launch Staging Web Server from AMI

**Via Console:**

1. Go to **EC2** → **AMIs**
2. Select the AMI you just created
3. Click **Launch instance from AMI**
4. Configure:
   - Name: `STG-EC2-DOFWEB-APP-01`
   - Instance type: `c5.large` (smaller than production — staging doesn't need full power)
   - VPC: Same VPC as production (or a separate staging VPC if available)
   - Security Group: Same as production
5. Launch

---

### Step 3: Replicate Database to Staging

**Option A: If staying on EC2 (current setup)**

```bash
# On the production DB server, export the database
mysqldump -u root -p newdofwebsite > /tmp/dofweb_staging_backup.sql

# Copy to staging DB server
scp /tmp/dofweb_staging_backup.sql user@staging-db-server:/tmp/

# On staging DB server, import
mysql -u root -p newdofwebsite < /tmp/dofweb_staging_backup.sql
```

**Option B: If migrated to RDS (recommended future state)**

1. Go to **RDS** → Select production instance
2. **Actions** → **Create read replica** or **Restore to point in time**
3. Name: `stg-dofweb-db`
4. Instance class: `db.t3.medium` (smaller for staging)

---

### Step 4: Update Staging WordPress Configuration

On the staging web server, update `wp-config.php` to point to the staging database:

```php
// Change database host to staging DB
define('DB_HOST', 'staging-db-host-here');
```

---

### Step 5: Test in Staging

Before applying any changes to production, test in staging:

```
Staging Test Checklist:
☐ Website loads correctly
☐ wp-admin is accessible
☐ All pages render properly
☐ CPU stays below 80% after changes
☐ No database connection errors
☐ Forms and uploads work
☐ Wait 24 hours to confirm stability
```

---

### Step 6: Promote to Production

Only after staging passes all tests:

1. Schedule a maintenance window with DOF
2. Create a fresh AMI of production (rollback point)
3. Apply the same changes to production
4. Monitor CloudWatch dashboard for 1 hour after changes
5. Confirm with DOF that everything works

> **Best Practice:** Never apply changes to production that haven't been tested in staging first. Always have a rollback plan (AMI snapshot).

---

## 6. Change Management Runbook

A runbook is a documented procedure for standard operations. DOF should follow this for every update:

```
RUNBOOK: WordPress / Plugin Update Process

PRE-CHANGE:
1. Create AMI of production web server
2. Create database backup (mysqldump or RDS snapshot)
3. Notify DOF and SSI teams of planned change

STAGING:
4. Apply updates to staging web server
5. Run staging test checklist
6. Wait 24 hours for stability
7. If tests fail → fix in staging, do NOT touch production

PRODUCTION:
8. Schedule maintenance window
9. Apply same updates to production
10. Monitor CloudWatch dashboard for 1 hour
11. Verify wp-admin and public site

POST-CHANGE:
12. Confirm with DOF that everything works
13. Keep AMI backup for 7 days (rollback point)
14. Document what was changed and when
```

---

## 7. Best Practices Summary

### REL 6 — Monitoring Best Practices

| # | Best Practice | DOF Action |
|---|-------------|------------|
| 1 | Monitor all components of the workload | Install CloudWatch Agent on all EC2 instances |
| 2 | Set alarms at warning thresholds, not critical | CPU alarm at 80%, not 99% |
| 3 | Use dashboards for real-time visibility | Create DOF-Website-Dashboard |
| 4 | Send notifications to multiple people | Use distribution list for SNS alerts |
| 5 | Enable detailed monitoring (1-minute intervals) | Enable in EC2 instance settings |
| 6 | Monitor application-level metrics | Track ALB request count, latency, error rates |

### REL 8 — Change Management Best Practices

| # | Best Practice | DOF Action |
|---|-------------|------------|
| 1 | Never update production directly | Always test in staging first |
| 2 | Use runbooks for standard operations | Follow the WordPress Update Runbook |
| 3 | Always have a rollback plan | Create AMI + DB backup before changes |
| 4 | Schedule maintenance windows | Coordinate with DOF for off-hours changes |
| 5 | Document all changes | Record what changed, when, and by whom |
| 6 | Monitor after every change | Watch CloudWatch for 1 hour post-change |

---

## 8. Next Steps

| # | Action | Owner |
|---|--------|-------|
| 1 | Install CloudWatch Agent on web and DB servers | SSI |
| 2 | Create CloudWatch Dashboard (DOF-Website-Dashboard) | SSI |
| 3 | Create CloudWatch Alarms (5 alarms) | SSI |
| 4 | Create SNS Topic (DOF-Alerts) + subscribe DOF/SSI emails | SSI + DOF |
| 5 | Create staging web server from AMI | SSI |
| 6 | Replicate database to staging | SSI |
| 7 | Test staging environment | SSI + DOF |
| 8 | Distribute WordPress Update Runbook to DOF | SSI |

**Next Session (Session 5):** Backup & Data Protection (REL 9)

---

## References

- [AWS Well-Architected — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [REL 6 — Monitor workload resources](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_monitor_monitor_workload.html)
- [REL 8 — Implement change](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_manage_change_implement_change.html)
- [Amazon CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [CloudWatch Agent Installation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [Amazon SNS User Guide](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)

---

**Prepared by:** Sagesoft Solutions
**Date:** April 6, 2026
