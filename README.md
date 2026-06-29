# AWS Chaos Engineering & Incident Recovery

**Topics:** AWS, Chaos Engineering, Fault Injection, Site Reliability Engineering, Incident Response, Postmortem, DevOps

[![AWS](https://img.shields.io/badge/Cloud-AWS-orange)](https://aws.amazon.com/) [![Chaos Engineering](https://img.shields.io/badge/Focus-Chaos%20Engineering-red)](https://en.wikipedia.org/wiki/Chaos_engineering) [![SRE](https://img.shields.io/badge/Focus-Site%20Reliability%20Engineering-green)](https://sre.google/) [![Incident Response](https://img.shields.io/badge/Practice-Incident%20Response-blue)](https://aws.amazon.com/cloudwatch/) [![Postmortem](https://img.shields.io/badge/Output-Blameless%20Postmortem-lightgrey)](https://sre.google/sre-book/postmortem-culture/)

---

> **Note:** This is **Part 3** of a three-part cloud SRE engineering project.
> While Part 1 built a highly available AWS infrastructure and Part 2 instrumented it with production-grade observability, this phase focuses on **deliberately injecting failures, measuring detection and recovery, and producing a blameless postmortem**.

---

## 📌 Project Overview

This project demonstrates real-world **chaos engineering and incident recovery** practices applied to a live AWS multi-tier architecture.

Inspired by Netflix's Chaos Monkey and Google's DiRT (Disaster Recovery Testing) program, real failures were deliberately injected into the production-grade infrastructure deployed in Part 1 and monitored by the observability stack built in Part 2.

The goal was to validate resilience, expose observability gaps, measure detection and recovery times, and document findings in a professional blameless postmortem — the same format used by SRE teams at Google, Amazon, and Netflix.

Key engineering areas explored:

- Chaos Engineering & Controlled Fault Injection
- Time to Detect (TTD) and Time to Mitigate (TTM) measurement
- Cascading Failure Analysis
- Blameless Postmortem Writing
- Observability Gap Identification
- Production Incident Diagnosis



## 🔧 Skills Demonstrated

- AWS Fault Injection (Manual Chaos Engineering)
- Incident Detection & Response
- Root Cause Analysis (RCA)
- Cascading Failure Diagnosis
- CloudWatch Alarm Evaluation
- EC2 Instance Connect Live Diagnosis
- MySQL Connection Pool Troubleshooting
- Blameless Postmortem Authorship
- SLO Compliance Assessment



## 💥 Fault Injection Summary

Two real failures were injected into the live system from different failure categories, following the SRE chaos engineering methodology.

| Fault | Category | TTD | TTM | Alarm Fired? |
|---|---|---|---|---|
| EC2 Auto Scaling Instance Termination | Compute | 1 min (visual) / 8 min (alarm) | 31 minutes | ✅ Yes (after recovery) |
| RDS Instance Reboot | Database | Never detected | ~18 minutes | ❌ No |



## 🔴 Fault 1 — EC2 Instance Termination (Compute Failure)

### What Was Injected

The Auto Scaling Instance (`i-080d88a8e860f23ab`) was terminated via the EC2 console while under active synthetic load, simulating an unexpected compute failure.

### What Happened

The Auto Scaling Group detected the terminated instance and launched a replacement within approximately 2 minutes — demonstrating the self-healing capability of the ASG design. However, the abrupt termination caused the Node.js MySQL connection pool to close database connections uncleanly.

Stale connections from the terminated instance, combined with new connections from the replacement instance and ongoing synthetic traffic, exhausted the `db.t3.micro` RDS connection limit (~66 max connections), triggering `ERROR 1040: Too many connections`.

### Impact

- Error rate peaked at **13.57%** — 13× the 1.0% SLO error budget
- `/students` endpoint returned error pages throughout the secondary failure window
- Total incident duration: **31 minutes**

### Detection

| Detection Method | Time |
|---|---|
| Dashboard visual (Error Rate widget) | ~1 minute |
| `SLO-Breach-5XX-Error-Rate` alarm fired | 8 minutes |
| Secondary database failure detected | ~16 minutes (manual only) |

**Key Finding:** The alarm fired 6 minutes *after* apparent recovery — providing notification value but zero actionable response value. The secondary cascading database failure generated **zero monitoring signals** and was discovered only through manual browser testing.

### Recovery

1. Connected to replacement EC2 instance via EC2 Instance Connect
2. Confirmed Node.js process running
3. Verified Secrets Manager returning the correct RDS endpoint
4. Ran direct MySQL connection → received `ERROR 1040: Too many connections`
5. Confirmed 59 active connections against ~66 max limit via RDS console
6. Rebooted `CapstoneDB` to clear all stale connections
7. Confirmed `/students` loading correctly — full recovery at **15:45**

**TTM: 31 minutes**



## 🟡 Fault 2 — RDS Instance Reboot (Database Failure)

### What Was Injected

The RDS instance `CapstoneDB` was rebooted via the AWS console to simulate a database restart event — common during patching, parameter changes, or failover scenarios.

### What Happened

The Node.js connection pool automatically reconnected to RDS after the reboot without surfacing any HTTP 500 errors to the ALB. The entire database restart was **invisible to the observability stack**.

### Impact

- No HTTP 500 errors generated at the ALB level
- No CloudWatch alarm fired
- No dashboard metric spiked
- `/students` degradation discovered only through manual browser testing
- Total incident duration: **~18 minutes** across three RDS reboots

### Detection

**TTD: Never detected by any monitoring system.**

The only observable signal was a minor dip in the Traffic Volume widget — insufficient to trigger any alert. A complete database restart was entirely invisible to the ALB-centric monitoring stack.

### Recovery

Three RDS reboots were required. The first reboot completed silently with auto-reconnection, but residual connection exhaustion from Fault 1 kept `/students` broken. The second reboot briefly restored service, but the still-running synthetic traffic immediately re-exhausted connections. The synthetic curl loop was stopped before the third reboot, allowing RDS to return with a clean connection count.

**TTM: ~18 minutes**



## 📋 Blameless Postmortem

A complete blameless postmortem was written for Fault 1 — the most significant failure — following the SRE Workbook Appendix D template.

**[View Full Postmortem (PDF)](./Postmortem_Cascading_Compute_and_Database_Failure.pdf)**

### Postmortem Highlights

**What Went Well:**
- ASG automatically replaced the terminated instance within ~2 minutes without manual intervention
- `HealthyHostCount` never dropped to 0 — replacement was fast enough to maintain continuous ALB registration
- Dashboard Error Rate widget clearly showed the spike within 1 minute of injection
- `SLO-Breach-5XX-Error-Rate` alarm correctly identified the breach and delivered SNS notification
- Secrets Manager correctly provided RDS credentials to the replacement instance automatically

**What Went Wrong:**
- Alarm fired 8 minutes after injection and 6 minutes after apparent recovery — too late for actionable response
- No alarm existed for RDS connection count — database connection exhaustion was completely invisible to monitoring
- ALB health check path `/` has no database dependency — a healthy EC2 instance masked a complete database failure
- Dashboard showed `HealthyHostCount: 1` throughout despite `/students` being broken for 31 minutes
- No runbook existed for `ERROR 1040` diagnosis, adding ~10 minutes of unstructured investigation



## 🛠️ Action Items & Improvement Proposals

| Action | Type | Priority |
|---|---|---|
| Add synthetic probe hitting `/students` every 60 seconds with alarm on non-200 response | Detect | Critical |
| Add `HealthyHostCount < 1` alarm with 1-minute evaluation period | Detect | High |
| Add RDS `DatabaseConnections < 1` alarm with SNS notification | Detect | High |
| Reduce `SLO-Breach-5XX-Error-Rate` evaluation period from 5 min to 1 min | Detect | High |
| Configure MySQL `wait_timeout` on RDS parameter group to reduce stale connection retention | Prevent | High |
| Update ALB health check path from `/` to `/students` | Prevent | Medium |
| Write runbook for `ERROR 1040` covering diagnosis commands and recovery steps | Process | High |
| Update on-call handoff document to include RDS connection exhaustion as known failure mode | Process | Medium |
| Establish protocol to stop synthetic traffic generators before executing recovery procedures | Process | Medium |



## 🎯 SLO Compliance Assessment

| SLO | Target | Fault 1 Result | Fault 2 Result |
|---|---|---|---|
| Availability | > 99.9% | ❌ Breached — 13.57% error rate at peak | ✅ No HTTP errors generated |
| p95 Latency | < 1500ms | ✅ Not breached | ✅ Not breached |
| Database Health | > 99.0% | ❌ Breached — ERROR 1040 on all DB queries | ❌ Breached silently — undetected |

**Error Budget Impact (Fault 1):**
The 31-minute incident consumed **70.8%** of the monthly 43.8-minute error budget in a single fault injection exercise.



## 🔍 Key Observability Findings

### Finding 1 — ALB-Centric Monitoring is Insufficient for Multi-Tier Architectures

The entire observability stack was built around ALB metrics. Database-layer failures that don't produce HTTP 500 errors at the load balancer are completely invisible. A healthy EC2 instance masked a total database failure for 31 minutes.

### Finding 2 — Alarm Evaluation Windows Must Match Fault Duration

The `SLO-Breach-5XX-Error-Rate` alarm requires 2 out of 3 datapoints to breach threshold across a 5-minute evaluation period. Short-duration compute failures that self-heal via ASG will consistently fire alarms *after* recovery, providing notification but no response value.

### Finding 3 — Connection Pool Auto-Reconnection Masks Database Outages

The Node.js MySQL connection pool automatically reconnects after a database restart without surfacing errors to the application layer. A complete RDS reboot was undetectable without a synthetic end-to-end health probe.

### Finding 4 — Fault Injection Tooling Can Become a Contributing Factor

The synthetic curl loop used for traffic generation during fault injection re-exhausted the RDS connection pool after every reboot attempt, extending the total incident duration significantly. Monitoring tools must be treated as part of the system under test.



## 🧠 Lessons Learned

A single compute fault can cascade into a multi-layer failure through indirect resource exhaustion — a pattern common in production but rarely anticipated in monitoring design. The most significant lesson is that an ALB-centric observability stack is insufficient for a multi-tier architecture: the database layer requires independent monitoring coverage.

The 31-minute incident was largely driven not by the original fault, but by the gap between what the dashboard showed (healthy) and what users experienced (broken). Defense in depth in observability is as important as defense in depth in infrastructure.



## ⚙️ Technology Stack

### Cloud Infrastructure
- AWS EC2 + Auto Scaling Group
- Application Load Balancer (ALB)
- Amazon RDS (MySQL — db.t3.micro)
- AWS Secrets Manager
- AWS Cloud9 IDE

### Observability
- Amazon CloudWatch
- CloudWatch Alarms (Metric Math)
- Amazon SNS

### Fault Injection
- Manual fault injection via AWS Console
- Synthetic load generation via curl loop (Cloud9)
- EC2 Instance Connect for live diagnosis
- MySQL CLI for database-layer diagnosis



## 📖 SRE Documentation

- **[Full Blameless Postmortem (PDF)](./Postmortem_Cascading_Compute_and_Database_Failure.pdf):** Complete incident postmortem following the SRE Workbook Appendix D template, covering timeline, root cause analysis, detection analysis, response and recovery, action items, and lessons learned.



[![⬅️ Back to Part 2](https://img.shields.io/badge/Back%20to-Part%202-blue)](https://github.com/georgecyberli/FCM790-SRE-Project2) [![⬅️ Back to My Portfolio](https://img.shields.io/badge/Back%20to-My%20Portfolio-blue)](https://github.com/georgecyberli)
