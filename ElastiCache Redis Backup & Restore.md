

## 1. Overview

### Why Backup & Restore Matters for Redis in Finance

Redis often holds **session state, order books, risk flags, throttling counters, and hot-reference data**.
For financial workloads, the expectations are strict:

* **RPO:** ~0 (no data loss) or minutes at worst
* **RTO:** < 1 hour
* **Zero customer impact** during backup
* **Auditability & compliance** (change snapshots, retention, encryption)

Amazon ElastiCache provides **native snapshot-based backup and restore** for **Redis only** (not Memcached).

---

### Backup Types

| Backup Type             | Description                            | Typical Use                    |
| ----------------------- | -------------------------------------- | ------------------------------ |
| **Automatic Snapshots** | Daily snapshots retained 1–35 days     | Baseline DR, compliance        |
| **Manual Snapshots**    | User-triggered, retained until deleted | Pre-change, pre-deploy, audits |

> Snapshots are **stored in AWS-managed S3** and are **incremental** after the first full snapshot.

---

### Key Constraints (Know These Early)

* Redis-only feature
* **Max retention:** 35 days (automatic)
* **Node type matters** (avoid tiny nodes)
* **Backups create I/O pressure** if memory is tight
* **Snapshots restore only to a NEW cluster** (not in-place overwrite)

---

### RPO / RTO Targets (Recommended)

| Tier                     | RPO       | RTO          |
| ------------------------ | --------- | ------------ |
| Trading / Payments       | Near-zero | < 30–60 mins |
| Customer session / cache | < 1 hour  | < 2 hours    |
| Analytics cache          | 24 hours  | Same day     |

---

### Performance Optimization Levers

* **`reserved-memory-percent`** → prevents snapshot OOM
* **Backup from read replicas** → zero impact on primaries
* **Low-traffic snapshot window**
* **Right-sized nodes (memory headroom)**

---

## 2. Prerequisites & Reference Architecture

### IAM & Security

ElastiCache manages snapshots internally, but for **automation, export, and audit**, ensure:

```json
{
  "Effect": "Allow",
  "Action": [
    "elasticache:CreateSnapshot",
    "elasticache:CopySnapshot",
    "elasticache:DescribeSnapshots",
    "elasticache:DeleteSnapshot"
  ],
  "Resource": "*"
}
```

If exporting or copying:

* **KMS permissions** for encrypted snapshots
* **S3 bucket policies** (cross-account DR)

---

### Network & Platform

* Private **VPC-only clusters**
* **VPC Endpoints** for:

  * ElastiCache
  * S3 (for DR workflows)
* **EKS applications** connect via internal endpoints

---

### Monitoring & Observability

Minimum CloudWatch signals to wire:

* Snapshot status (`available`, `failed`)
* Memory fragmentation
* Evictions
* CPU steal
* Replica lag

---

## 3. Step-by-Step Implementation

---

### A. Automatic Backups (Baseline Safety Net)

#### Console

* Enable **Backup Retention**: `7–35 days`
* Set **Snapshot Window** during off-peak hours

#### CLI

```bash
aws elasticache modify-replication-group \
  --replication-group-id fin-redis-rg \
  --snapshot-retention-limit 14 \
  --snapshot-window 03:00-04:00 \
  --apply-immediately
```

#### Terraform

```hcl
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "fin-redis-rg"
  engine                     = "redis"
  node_type                  = "cache.r6g.large"
  snapshot_retention_limit   = 14
  snapshot_window            = "03:00-04:00"
  automatic_failover_enabled = true
}
```

---

### B. Manual Backups (Change & Audit Safety)

**Always take a snapshot before:**

* Application deploys
* Parameter changes
* Node type upgrades
* Redis version upgrades

```bash
aws elasticache create-snapshot \
  --replication-group-id fin-redis-rg \
  --snapshot-name pre-release-2026-01-13
```

**Final snapshot on deletion**

```bash
aws elasticache delete-replication-group \
  --replication-group-id fin-redis-rg \
  --final-snapshot-identifier fin-final-snap
```

---

### C. Cross-Region / DR Strategy

* Copy snapshots to DR region
* Restore into **warm-standby clusters**
* Keep DNS / service discovery ready

```bash
aws elasticache copy-snapshot \
  --source-snapshot-name pre-release-2026-01-13 \
  --target-snapshot-name dr-pre-release-2026-01-13 \
  --target-bucket dr-backups-bucket
```

---

### D. Restore Strategies

#### Restore to New Cluster (Standard)

```bash
aws elasticache create-replication-group \
  --replication-group-id fin-redis-restore \
  --snapshot-name pre-release-2026-01-13 \
  --node-type cache.r6g.large \
  --automatic-failover-enabled
```

#### Blue-Green Restore (Zero Downtime)

1. Restore snapshot → **new cluster**
2. Sync config
3. Warm traffic (shadow reads)
4. Switch endpoint / DNS
5. Decommission old cluster

---

## 4. Performance & Production Safety

### 1️ Backup from Read Replicas

* Multi-AZ replication groups automatically prefer replicas
* Avoids latency spikes on primary

---

### 2️ `reserved-memory-percent` (Critical)

**Recommended:** `25–50%` for financial workloads

```json
{
  "reserved-memory-percent": "30"
}
```

Why:

* Redis forks memory during snapshots
* Without headroom → snapshot failure or eviction storm

---

### 3️ Node Sizing Rules

| Bad Practice     | Correct Approach   |
| ---------------- | ------------------ |
| t1 / tiny nodes  | r6g / r7g families |
| 90% memory usage | Max 65–70% steady  |
| Single AZ        | Multi-AZ always    |

---

### 4️ Multi-AZ Testing Workflow

* Quarterly DR restore test
* Restore snapshot in DR region
* Run synthetic EKS load
* Validate latency + data consistency

---

## 5. Terraform & Event-Driven Automation

### Snapshot Monitoring with EventBridge

```json
{
  "source": ["aws.elasticache"],
  "detail-type": ["ElastiCache Snapshot Notification"]
}
```

Trigger:

* Slack / PagerDuty alert
* JIRA ticket for failures

---

### Cross-Account Snapshot Copy (Terraform Concept)

```hcl
resource "aws_elasticache_snapshot" "copy" {
  source_snapshot_name = "prod-snapshot"
  snapshot_name        = "dr-copy"
}
```

---

## 6. Disaster Recovery & Testing

### DR Checklist (Financial Grade)

* ✔ Automatic snapshots enabled
* ✔ Manual snapshot before every prod change
* ✔ Encrypted snapshots (KMS)
* ✔ Cross-region restore tested
* ✔ RTO < 1 hour validated
* ✔ Audit log of snapshot IDs retained

---

### AWS Backup Integration

* Centralized retention policy
* Cross-account vaults
* Compliance reporting (SOC, ISO)

---

## Text-Based Architecture (Production Pattern)

```
EKS Apps
   |
   | (read/write)
   v
ElastiCache Redis (Multi-AZ)
   |        |
 Primary   Replica  <-- Snapshots
               |
               v
         AWS-managed S3
               |
               v
         DR Region Restore
```

---

## Automatic vs Manual Snapshot – Quick Comparison

| Aspect    | Automatic   | Manual            |
| --------- | ----------- | ----------------- |
| Trigger   | AWS-managed | User-triggered    |
| Retention | 1–35 days   | Until deleted     |
| Use case  | Baseline DR | Pre-change, audit |
| Control   | Medium      | Full              |

---

## Final Production Checklist (Print This)

* [ ] Multi-AZ enabled
* [ ] Reserved memory ≥ 30%
* [ ] Snapshot window off-peak
* [ ] Pre-deploy snapshot mandatory
* [ ] Cross-region restore tested
* [ ] Monitoring & alerts wired
* [ ] DR runbook documented

---

### Closing Thought

> **Redis backups are not about copying data.
> They are about buying certainty under stress.**

When markets spike, traffic surges, or mistakes happen —
your snapshot strategy decides whether recovery is calm or chaotic.

---


