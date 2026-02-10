# aws-wordpress-scalable-architecture

**Design-only reference architecture for a scalable, production-grade WordPress on AWS.**  
Edge: CloudFront + WAF + ALB · App: Auto Scaling EC2 (PHP 8.3) · Data: RDS (MariaDB) · Shared media: EFS · Cache: ElastiCache (Memcached).  
Optimized for **high traffic (10k+ concurrent)**, **low latency**, **SEO**, and **daily backups**.

> **Diagram:** `docs/aws_proje.drawio.pdf`
---

## Table of Contents
- [1. Purpose & Scope](#1-purpose--scope)
- [2. Goals & Assumptions](#2-goals--assumptions)
- [3. Architecture Overview](#3-architecture-overview)
- [4. Component Breakdown](#4-component-breakdown)
- [5. VPC & Network Topology](#5-vpc--network-topology)
- [6. Security & Compliance](#6-security--compliance)
- [7. Performance & SEO](#7-performance--seo)
- [8. Backups & Disaster Recovery](#8-backups--disaster-recovery)
- [9. Observability](#9-observability)
- [10. Cost Considerations](#10-cost-considerations)
- [11. Risks & Trade‑offs](#11-risks--trade-offs)
- [12. Future Enhancements](#12-future-enhancements)
- [13. Repository Structure](#13-repository-structure)
- [14. License](#14-license)

---

## 1. Purpose & Scope
This repository documents the **system design** of a scalable WordPress platform on AWS.  
It captures decisions, rationale, and reference topology. **It does not provide deployment code** (e.g., Terraform, CloudFormation, scripts).

**Out of scope:** step‑by‑step installation, CI/CD, OS hardening scripts, application theming/plugins beyond cache guidance.

---

## 2. Goals & Assumptions
**Goals**
- Serve **10k+ concurrent requests** with stable latency.
- Maximize **SEO performance** (TTFB, caching, image optimization).
- Ensure **security** at the edge and app layers (WAF, TLS, least privilege).
- Provide **daily backups** and clear **DR** path.
- Keep a pragmatic eye on **costs** (cache/CDN to reduce egress and origin load).

**Assumptions**
- WordPress (latest) on PHP **8.3**.
- Database: **MariaDB** managed via **Amazon RDS**.
- Shared media ≈ **30 GB**; stored on **EFS** for multi‑AZ app fleet.
- Memcached for object/session cache via **ElastiCache**.
- Multi‑AZ deployment in a single AWS Region.

---

## 3. Architecture Overview
- **Edge:** Route 53 → **CloudFront (CDN)** → **AWS WAF/Shield Standard**  
- **Entry:** **Application Load Balancer (ALB, HTTPS)** + HTTP→HTTPS redirect  
- **App:** **EC2 Auto Scaling** (PHP 8.3, Nginx/Apache, stateless instances)  
- **Shared Media:** **Amazon EFS** mounted on all app instances (`wp-content/uploads`)  
- **Data:** **Amazon RDS (MariaDB)** (+ optional **Read Replica** for read scaling/standby)  
- **Cache:** **Amazon ElastiCache (Memcached)** for object/fragment cache  
- **Objects & Logs:** **Amazon S3** (Versioning + Lifecycle policies)  
- **Ops:** CloudWatch/CloudTrail/GuardDuty, **AWS Backup**; access via Bastion or **SSM Session Manager**

---

## 4. Component Breakdown
**Route 53** – public DNS with ALIAS to CloudFront; optional health checks.  
**CloudFront** – global edge caching; TLS termination; shields origins; forwards minimal headers/cookies.  
**AWS WAF** – managed rules (SQLi/XSS), rate limiting, allow/deny lists.  
**ALB** – L7 routing, TLS from ACM, HTTP→HTTPS redirect, access logs to S3.  
**EC2 ASG** – stateless app tier; scale on CPU or ALB requests; immutable images recommended (AMI bake).  
**EFS** – shared media; one mount target per AZ; encrypted at rest; lifecycle to IA class where suitable.  
**RDS (MariaDB)** – Multi‑AZ for HA or Single‑AZ + Read Replica; encrypted; automated backups.  
**ElastiCache (Memcached)** – object/fragment cache; reduces DB and app load.  
**S3** – object storage, log archive, versioned backups; lifecycle to Glacier for cost control.  
**SSM** – parameter/secret storage, session‑based access (preferred over SSH).

---

## 5. VPC & Network Topology
- **VPC CIDR:** `10.0.0.0/16` (example)
- **Subnets per AZ:** Public (`10.0.0.0/24`, `10.0.1.0/24`), App (`10.0.10.0/24`, `10.0.11.0/24`), Data (`10.0.20.0/24`, `10.0.21.0/24`)
- **Routing:** Public → IGW; App/Data → NAT GW (egress only)
- **Access controls:** DB/Cache reachable **only** from App SG; EFS restricted to App SG; no public DB/EFS endpoints.

---

## 6. Security & Compliance
- **Edge security:** WAF managed + custom rules, rate limits; Shield Standard for L3/L4 DDoS.  
- **Identity:** IAM roles with **least privilege**; no long‑lived keys on instances.  
- **Encryption:** TLS in transit; KMS‑backed at rest for EBS/EFS/S3/RDS.  
- **Secrets:** SSM Parameter Store / Secrets Manager for DB creds and WP salts.  
- **Audit:** CloudTrail enabled; optional VPC Flow Logs; ALB/CloudFront logs to S3 with lifecycle.  
- **Access:** Prefer **SSM Session Manager**; avoid public SSH and open management ports.

---

## 7. Performance & SEO
- **CDN** to reduce TTFB and offload origin; long TTLs for static assets.  
- **Autoscaling** on request rate/CPU; AMI bake for faster scale‑out.  
- **Memcached** object/fragment caching; minimize cookie/header/query forwarding.  
- **HTTP/2**, gzip/brotli, proper `Cache-Control` and image optimization (WebP/AVIF, lazy‑load).  
- Consider S3 offload for media in future to cut EFS and egress costs.

---

## 8. Backups & Disaster Recovery
- **Daily** RDS snapshots (retention 7–30 days); automated backups enabled.  
- **EFS** daily backups via AWS Backup; S3 versioning + lifecycle to Glacier.  
- **Failover:** promote **RDS Read Replica** if primary fails; document RTO/RPO targets.  
- **Optional** cross‑region replication (S3 CRR, RDS RR) for higher resilience.

---

## 9. Observability
- **Metrics/Alarms:** ALB 5xx/latency/healthy targets; EC2 CPU/memory (agent); RDS CPU/free storage; ElastiCache evictions.  
- **Logs:** ALB & CloudFront access logs to S3; analyze via CloudWatch Logs Insights/Athena.  
- **Dashboards:** CloudWatch dashboards for edge, app, and data tiers.

---

## 10. Cost Considerations
Largest cost drivers: **NAT Gateway**, **ALB**, **EC2/EBS**, **EFS**, **RDS**, **ElastiCache**, **CloudFront**, **S3 transfer**, **WAF**.  
Reduce spend with: higher CDN/cache hit rates, right‑sized instances, low ASG min capacity, EFS IA, minimal header/cookie forwarding, consolidated NAT where safe.

---

## 11. Risks & Trade‑offs
- **EFS latency vs. simplicity:** convenient shared media but adds latency; evaluate S3 offload later.  
- **Memcached vs. Redis:** Memcached is simple/fast; Redis adds features (persistence, data structures) at higher cost/complexity.  
- **Single Region:** simpler ops; cross‑region DR increases resilience but adds cost.  
- **ASG scale‑out time:** consider AMI bake and warm pools.

---

## 12. Future Enhancements
- **Media offload to S3** + CloudFront signed URLs.  
- **Blue/Green or canary** deployments via ALB or CodeDeploy.  
- **Infrastructure as Code** (Terraform/CloudFormation) and **CI/CD**.  
- **WAF custom rules** tuned to app traffic patterns.  
- **Containerization** (ECS/EKS) if app complexity grows.

---

## 13. Repository Structure
```
.
├── docs/
│   └── aws_proje.drawio.pdf   # architecture diagram 
└── README.md                  

---

## 14. License
MIT
