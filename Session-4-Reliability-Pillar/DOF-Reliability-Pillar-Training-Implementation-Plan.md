# DOF Reliability Pillar — Training & Implementation Plan
**AWS Well-Architected Framework — Reliability Pillar**

---

## Overview

The Reliability pillar focuses on ensuring a workload performs its intended function correctly and consistently. This plan covers training sessions to educate DOF on reliability concepts and implementation activities to address the gaps identified in the WAF assessment.

> **Reference:** [AWS Well-Architected — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)

---

## Current State (as of April 6, 2026)

| # | WAF Question | Status | Why |
|---|-------------|--------|-----|
| REL 1 | Manage service quotas and constraints | Done | Control Tower multi-account setup manages quotas per account, SCPs limit what can be provisioned |
| REL 2 | Plan network topology | Done | VPC with public/private subnets, Security Groups, ALB already exist in DOF's environment |
| REL 3 | Design architecture for high availability | Partial | Multi-account structure done, but web and DB are still single instances with no Multi-AZ or ASG |
| REL 4 | Design interactions to prevent failures | Not done | No timeouts, connection limits, or rate limiting configured |
| REL 5 | Design interactions to mitigate failures | Not done | No maintenance page, no graceful degradation, no CDN |
| REL 6 | Monitor workload resources | Not done | No CloudWatch alarms or dashboards — March 27 incident went undetected for days |
| REL 7 | Adapt to changes in demand | Not done | No Auto Scaling Group — single instance handles all traffic |
| REL 8 | Implement change management | Not done | No staging environment — updates applied directly to production (caused March 27 incident) |
| REL 9 | Back up data | Not done | No automated backups — only manual snapshots before changes |
| REL 10 | Use fault isolation | Not done | Web and DB are separate (good), but no RDS Multi-AZ failover, no redundancy per tier |
| REL 11 | Design for withstanding component failures | Not done | If one instance dies, the whole service goes down — no auto-replacement |
| REL 12 | Test reliability | Not done | No failover tests, no load tests, no backup restore tests conducted |
| REL 13 | Plan for disaster recovery | Not done | No DR plan, no cross-region backups, no documented RPO/RTO |

---

## Session Plan

### Session 4: Monitoring & Change Management
**Topics:** REL 6, REL 8
**Duration:** 2-3 hours

#### Training Component

**REL 6 — Monitor Workload Resources**

What DOF needs to understand:
- Why monitoring matters — the March 27 incident (DB CPU at 99% for days before detection) is a real example of what happens without monitoring
- CloudWatch metrics: CPU, memory, disk, network, EBS IOPS
- CloudWatch alarms: automated notifications when thresholds are breached
- CloudWatch dashboards: real-time visibility without SSH

> **Best Practice:** "Monitor all components of the workload to detect failures. Use CloudWatch to set alarms for key metrics and receive notifications."
>
> Reference: [REL 6 — Monitor workload resources](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_monitor_monitor_workload.html)

**REL 8 — Implement Change Management**

What DOF needs to understand:
- Why staging environments are critical — the March 27 incident was caused by updating production directly
- Change management process: test in staging → validate → promote to production
- Runbooks for standard operations (deployments, updates, maintenance)

> **Best Practice:** "Use runbooks for standard activities such as deployment. Test changes in a non-production environment before deploying to production."
>
> Reference: [REL 8 — Implement change](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_manage_change_implement_change.html)

#### Implementation Activities

| # | Activity | Details |
|---|----------|---------|
| 1 | Create CloudWatch Dashboard | Dashboard for DOF website: EC2 CPU, memory, EBS IOPS, ALB health, DB metrics |
| 2 | Set up CloudWatch Alarms | CPU > 80%, EBS queue > 1, disk > 85%, ALB unhealthy count > 0 |
| 3 | Configure SNS Notifications | Email alerts to DOF and SSI teams when alarms trigger |
| 4 | Create Staging Environment | AMI of production web server → launch smaller instance for staging |
| 5 | Replicate Database to Staging | mysqldump from DB server → import to staging DB |
| 6 | Document Change Management Process | Runbook: how to test updates in staging before production |

---

### Session 5: Backup & Data Protection
**Topics:** REL 9
**Duration:** 2-3 hours

#### Training Component

**REL 9 — Back Up Data**

What DOF needs to understand:
- Current risk: no automated backups — if data is lost, it's gone
- Backup strategies: snapshots (EBS), automated backups (RDS), S3 versioning
- Recovery Point Objective (RPO): how much data can you afford to lose?
- Recovery Time Objective (RTO): how fast do you need to recover?
- AWS Backup: centralized backup management across services
- 3-2-1 backup rule: 3 copies, 2 different media, 1 offsite

> **Best Practice:** "Back up data, applications, and configuration to meet your recovery time objectives (RTO) and recovery point objectives (RPO)."
>
> Reference: [REL 9 — Back up data](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_data_backup_back_up_data.html)

#### Implementation Activities

| # | Activity | Details |
|---|----------|---------|
| 1 | Enable EBS Snapshots | Automated daily snapshots for web server and DB server volumes |
| 2 | Configure AWS Backup | Backup plan: daily backups, 30-day retention |
| 3 | Enable S3 Versioning | For any S3 buckets with DOF content |
| 4 | Database Backup Strategy | Automated mysqldump schedule (if staying on EC2) or RDS automated backups (if migrated) |
| 5 | Test Backup Restoration | Restore from snapshot to verify backups work |
| 6 | Document RPO/RTO | Define acceptable data loss and recovery time with DOF |

---

### Session 6: High Availability & Fault Isolation
**Topics:** REL 3, REL 7, REL 10, REL 11
**Duration:** 3-4 hours

#### Training Component

**REL 3 — Design Architecture for High Availability**

What DOF needs to understand:
- Current risk: single instance per tier — any failure = complete outage
- Availability Zones (AZs): physically separate data centers within a region
- Multi-AZ deployments: spread resources across AZs for redundancy

**REL 7 — Adapt to Changes in Demand**

What DOF needs to understand:
- Auto Scaling Groups (ASG): automatically add/remove instances based on demand
- Why ASG alone doesn't fix everything (March 18 EBS incident — Launch Template must have correct IOPS)
- Scaling policies: target tracking, step scaling
- DOF prerequisite: WordPress must be stateless (shared storage for uploads)

**REL 10 — Use Fault Isolation**

What DOF needs to understand:
- Fault isolation: if one component fails, others keep running
- Separate web and database tiers (DOF already has this)
- Use managed services (RDS) to isolate database failures from web tier
- Multi-AZ for automatic failover

**REL 11 — Design for Withstanding Component Failures**

What DOF needs to understand:
- Health checks: ALB detects unhealthy instances and routes traffic away
- Auto-replacement: ASG launches new instances when old ones fail
- Database failover: RDS Multi-AZ automatically switches to standby

> **References:**
> - [REL 3 — Design for HA](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_ha_design_ha.html)
> - [REL 7 — Adapt to demand](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_adapt_to_demand_adapt.html)
> - [REL 10 — Fault isolation](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_fault_isolation_use_fault_isolation.html)
> - [REL 11 — Withstand failures](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_withstand_component_failures_design_withstand.html)

#### Implementation Activities

| # | Activity | Details |
|---|----------|---------|
| 1 | Migrate Database to RDS | MariaDB → RDS Multi-AZ (automated failover, backups, patching) |
| 2 | Deploy ElastiCache Redis | Object cache to reduce DB load |
| 3 | Set up EFS | Shared file storage for WordPress uploads (required for ASG) |
| 4 | Create Launch Template | Correct instance type, EBS IOPS/throughput, user data script |
| 5 | Implement Auto Scaling Group | Min 2 instances across 2 AZs behind ALB |
| 6 | Configure ALB Health Checks | Proper health check path, intervals, thresholds |

---

### Session 7: Failure Prevention & Mitigation
**Topics:** REL 4, REL 5
**Duration:** 2 hours

#### Training Component

**REL 4 — Design Interactions to Prevent Failures**

What DOF needs to understand:
- Throttling and rate limiting: protect services from being overwhelmed
- Timeouts and retries: don't let one slow service bring everything down
- Queue-based architecture: decouple components so failures don't cascade
- Connection pooling: manage database connections efficiently

**REL 5 — Design Interactions to Mitigate Failures**

What DOF needs to understand:
- Graceful degradation: serve cached/static content when database is down
- Circuit breaker pattern: stop sending requests to a failing service
- Maintenance pages: ALB fixed response when backend is down

> **References:**
> - [REL 4 — Prevent failures](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_prevent_interaction_failure_prevent.html)
> - [REL 5 — Mitigate failures](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_mitigate_interaction_failure_mitigate.html)

#### Implementation Activities

| # | Activity | Details |
|---|----------|---------|
| 1 | Configure ALB Maintenance Page | Fixed response rule for when all targets are unhealthy |
| 2 | Set Database Connection Limits | Configure max_connections appropriately in MariaDB/RDS |
| 3 | Implement CloudFront | CDN for static content — serves cached pages even if origin is down |
| 4 | Configure Request Timeouts | ALB idle timeout, PHP max execution time |

---

### Session 8: Reliability Testing & Disaster Recovery
**Topics:** REL 12, REL 13
**Duration:** 2-3 hours

#### Training Component

**REL 12 — Test Reliability**

What DOF needs to understand:
- Test failover: simulate instance failure, verify ASG replaces it
- Test backup restoration: restore from snapshot, verify data integrity
- Load testing: verify the system handles expected traffic
- Game days: scheduled exercises to practice incident response

**REL 13 — Plan for Disaster Recovery**

What DOF needs to understand:
- DR strategies (from cheapest to fastest recovery):
  1. Backup & Restore — cheapest, slowest (hours)
  2. Pilot Light — minimal resources running, faster (minutes to hours)
  3. Warm Standby — scaled-down copy running (minutes)
  4. Multi-Site Active/Active — full copy running (seconds)
- For DOF's current size: Backup & Restore is sufficient
- Cross-region backups: copy snapshots to another region for disaster protection

> **References:**
> - [REL 12 — Test reliability](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_test_reliability_test.html)
> - [REL 13 — Plan for DR](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html)

#### Implementation Activities

| # | Activity | Details |
|---|----------|---------|
| 1 | Conduct Failover Test | Terminate one ASG instance, verify auto-replacement |
| 2 | Conduct Backup Restore Test | Restore EBS snapshot, verify data |
| 3 | Load Test | Use Apache Bench or similar to test under load |
| 4 | Create DR Plan Document | Document RPO, RTO, recovery steps, responsible parties |
| 5 | Set Up Cross-Region Backups | Copy critical snapshots to us-west-2 (Oregon) as DR |

---

## Proposed Session Schedule

| Session | Topics | Prerequisites | Priority |
|---------|--------|---------------|----------|
| **Session 4** | Monitoring & Change Management (REL 6, REL 8) | None | High — do first |
| **Session 5** | Backup & Data Protection (REL 9) | None | High |
| **Session 6** | High Availability & Fault Isolation (REL 3, 7, 10, 11) | Staging environment ready (Session 4) | High |
| **Session 7** | Failure Prevention & Mitigation (REL 4, REL 5) | ASG implemented (Session 6) | Medium |
| **Session 8** | Reliability Testing & DR (REL 12, REL 13) | All above completed | Medium |

---

## After All Sessions — Expected Reliability State

| # | WAF Question | Status |
|---|-------------|--------|
| REL 1 | Manage service quotas | Done |
| REL 2 | Plan network topology | Done |
| REL 3 | Design for high availability | Done (Multi-AZ, ASG) |
| REL 4 | Prevent failures | Done (timeouts, connection limits, CDN) |
| REL 5 | Mitigate failures | Done (maintenance page, graceful degradation) |
| REL 6 | Monitor workload resources | Done (CloudWatch dashboards, alarms, SNS) |
| REL 7 | Adapt to changes in demand | Done (ASG) |
| REL 8 | Implement change management | Done (staging environment, runbooks) |
| REL 9 | Back up data | Done (AWS Backup, EBS snapshots, DB backups) |
| REL 10 | Use fault isolation | Done (RDS Multi-AZ, separate tiers) |
| REL 11 | Withstand component failures | Done (ASG auto-replacement, RDS failover) |
| REL 12 | Test reliability | Done (failover tests, load tests) |
| REL 13 | Plan for disaster recovery | Done (DR plan, cross-region backups) |

**Target: 13/13 Reliability questions addressed.**

---

**Prepared by:** Sagesoft Solutions
**Date:** April 6, 2026
