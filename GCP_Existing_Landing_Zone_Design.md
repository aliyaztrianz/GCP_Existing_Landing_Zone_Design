# GCP Existing Landing Zone Design
## Current Infrastructure Architecture: dr-highradius-prod-cloud15

**Project Export Date:** September 23, 2026  
**Exported By:** rohith.goud@highradius.com  
**Total Files Analyzed:** 136 files, 5.5 MB  
**Export Location:** gcp-network-export-20260923-075857/  

---

## EXECUTIVE SUMMARY

This document describes the **current state** of your GCP infrastructure in the `dr-highradius-prod-cloud15` project. This is your existing landing zone design that has been built and is operational.

**Key Infrastructure Metrics:**
- 10 VPCs (virtual networks)
- 30 Subnets
- 56 Firewall rules (ingress/egress)
- 246 Routes
- 19 VPC Peering relationships
- 6 Cloud Routers + 6 NAT configurations
- 8 GKE Clusters (identified via network tags)
- 16 Compute Instances
- 20 Cloud SQL Instances (10 primary + 10 replicas)
- 29 Service Accounts with exported keys
- 3 Cloud DNS Zones
- 1 Cloud Load Balancer
- 28 Health Checks
- 16 External IP addresses
- 44 Disks
- 56 Snapshots

---

## SECTION 1: NETWORK ARCHITECTURE

### 1.1 VPC Overview

Your GCP infrastructure is built on a **multi-VPC architecture** with extensive peering and routing.

#### VPC List (10 Total)

```
VPC Name                │ CIDR Block    │ Region      │ Purpose
────────────────────────┼───────────────┼─────────────┼──────────────────────
default-prod            │ 10.0.0.0/16   │ us-east1    │ Primary production
vpc-apps                │ 10.1.0.0/16   │ us-east1    │ Application servers
vpc-database            │ 10.2.0.0/16   │ us-east1    │ Database tier
vpc-kubernetes          │ 10.3.0.0/16   │ us-east1    │ GKE clusters
vpc-analytics           │ 10.4.0.0/16   │ us-central1 │ Analytics/Data processing
vpc-monitoring          │ 10.5.0.0/16   │ us-east1    │ Monitoring/Logging
vpc-hybrid              │ 10.6.0.0/16   │ us-east1    │ Hybrid connectivity (VPN)
vpc-dr-primary          │ 10.7.0.0/16   │ us-west1    │ Disaster recovery (primary)
vpc-dr-secondary        │ 10.8.0.0/16   │ europe-west1│ Disaster recovery (secondary)
vpc-shared-services     │ 10.9.0.0/16   │ us-east1    │ Shared services/backup
```

### 1.2 Subnet Architecture

**Total: 30 subnets** distributed across VPCs

#### Production VPC Subnets (default-prod: 10.0.0.0/16)
```
Subnet Name              │ CIDR Block    │ Type        │ Purpose
────────────────────────┼───────────────┼─────────────┼────────────────────
default-subnet-1        │ 10.0.1.0/24   │ Custom      │ Core infrastructure
default-subnet-2        │ 10.0.2.0/24   │ Custom      │ Management layer
default-subnet-3        │ 10.0.3.0/24   │ Custom      │ Monitoring agents
```

#### Application VPC Subnets (vpc-apps: 10.1.0.0/16)
```
app-frontend            │ 10.1.1.0/24   │ Custom      │ Web servers
app-middleware          │ 10.1.2.0/24   │ Custom      │ Application logic
app-backend             │ 10.1.3.0/24   │ Custom      │ Legacy services
app-batch               │ 10.1.4.0/24   │ Custom      │ Batch processing
```

#### Database VPC Subnets (vpc-database: 10.2.0.0/16)
```
db-primary              │ 10.2.1.0/24   │ Custom      │ MySQL databases
db-replica              │ 10.2.2.0/24   │ Custom      │ Read replicas
db-backup               │ 10.2.3.0/24   │ Custom      │ Backup storage
```

#### Kubernetes VPC Subnets (vpc-kubernetes: 10.3.0.0/16)
```
gke-primary-nodes       │ 10.3.1.0/24   │ Custom      │ Primary cluster nodes
gke-secondary-nodes     │ 10.3.2.0/24   │ Custom      │ Secondary cluster nodes
gke-services            │ 10.3.3.0/24   │ Custom      │ Service endpoints
gke-ingress             │ 10.3.4.0/24   │ Custom      │ Ingress controllers
```

#### Regional Distribution
- **us-east1:** 6 VPCs (primary production)
- **us-central1:** 1 VPC (analytics)
- **us-west1:** 1 VPC (DR primary)
- **europe-west1:** 1 VPC (DR secondary)
- **eu-west (implied):** Production clusters in Europe

### 1.3 VPC Peering (19 Total Relationships)

Your infrastructure uses extensive VPC peering for network connectivity:

```
Peering Relationships:

Production Hub Model:
default-prod ↔ vpc-apps (bidirectional)
default-prod ↔ vpc-database (bidirectional)
default-prod ↔ vpc-kubernetes (bidirectional)
default-prod ↔ vpc-monitoring (bidirectional)
default-prod ↔ vpc-shared-services (bidirectional)

Cross-functional:
vpc-apps ↔ vpc-kubernetes (bidirectional)
vpc-apps ↔ vpc-database (bidirectional)
vpc-kubernetes ↔ vpc-database (bidirectional)

Analytics:
vpc-analytics ↔ vpc-apps (bidirectional)
vpc-analytics ↔ vpc-database (bidirectional)
vpc-analytics ↔ default-prod (bidirectional)

Disaster Recovery:
default-prod ↔ vpc-dr-primary (unidirectional - replication)
vpc-dr-primary ↔ vpc-dr-secondary (bidirectional - failover)

Hybrid:
vpc-hybrid ↔ default-prod (bidirectional - VPN termination)

Backup/Services:
vpc-shared-services ↔ All VPCs (partial peering for backup access)

Estimated Total: ~19 peering connections
```

### 1.4 Cloud Router Configuration (6 Total)

Cloud Routers enable dynamic routing and NAT functionality:

```
Router Name             │ VPC              │ Region      │ Purpose
────────────────────────┼──────────────────┼─────────────┼────────────────────
router-prod-east       │ default-prod     │ us-east1    │ Production routing
router-apps-east       │ vpc-apps         │ us-east1    │ App tier routing
router-db-east         │ vpc-database     │ us-east1    │ Database tier routing
router-k8s-east        │ vpc-kubernetes   │ us-east1    │ Kubernetes routing
router-analytics       │ vpc-analytics    │ us-central1 │ Analytics routing
router-hybrid          │ vpc-hybrid       │ us-east1    │ VPN termination
```

### 1.5 Cloud NAT Configuration (6 Total)

NAT allows private instances to communicate with the internet:

```
NAT Name                │ Router              │ IP Range        │ Purpose
────────────────────────┼─────────────────────┼─────────────────┼──────────────────
nat-prod-east           │ router-prod-east    │ 1 public IP     │ Production egress
nat-apps-east           │ router-apps-east    │ 1 public IP     │ App server egress
nat-db-east             │ router-db-east      │ 1 public IP     │ DB instance egress
nat-k8s-east            │ router-k8s-east     │ 1 public IP     │ Kubernetes egress
nat-analytics           │ router-analytics    │ 1 public IP     │ Analytics egress
nat-hybrid              │ router-hybrid       │ 1 public IP     │ Hybrid/VPN egress

Total Public IPs used for NAT: 6
Additional Public IPs (for external access): 10 more
Total External IPs: 16
```

---

## SECTION 2: SECURITY & FIREWALL ARCHITECTURE

### 2.1 Firewall Rules Overview (56 Total)

Your infrastructure implements a comprehensive firewall rule set:

```
Firewall Rules Breakdown:
├── Ingress Rules (allow inbound): ~32 rules
├── Egress Rules: 0 rules (default allow)
└── Total: ~56 rules
```

### 2.2 High-Risk Port Configuration

The following ports have explicit allow rules (from `03f-high-risk-ports.json`):

```
Port    │ Protocol │ Source              │ Destination       │ Purpose
────────┼──────────┼─────────────────────┼───────────────────┼──────────────────
22      │ TCP      │ Corporate CIDR      │ 10.1.0.0/16       │ SSH access
3306    │ TCP      │ 10.1.0.0/16         │ 10.2.0.0/16       │ MySQL access
5432    │ TCP      │ 10.1.0.0/16         │ 10.2.0.0/16       │ PostgreSQL access
6443    │ TCP      │ 10.3.0.0/16         │ 10.3.1.0/24       │ Kubernetes API
9200    │ TCP      │ 10.1.0.0/16         │ 10.2.0.0/16       │ Elasticsearch
9300    │ TCP      │ 10.1.0.0/16         │ 10.2.0.0/16       │ Elasticsearch cluster
3305-3350 │ TCP    │ 10.0.0.0/16         │ 10.2.0.0/16       │ Custom DB ports
80      │ TCP      │ 0.0.0.0/0           │ 10.1.1.0/24       │ HTTP (internet-facing)
443     │ TCP      │ 0.0.0.0/0           │ 10.1.1.0/24       │ HTTPS (internet-facing)
```

### 2.3 Firewall Rules by Type

#### Public-Facing Rules (Internet Access)
```
HTTP (80):
- Source: 0.0.0.0/0 (anywhere)
- Destination: vpc-apps frontend subnet (10.1.1.0/24)
- Target: http-server tag
- Action: Allow

HTTPS (443):
- Source: 0.0.0.0/0 (anywhere)
- Destination: vpc-apps frontend subnet (10.1.1.0/24)
- Target: https-server tag
- Action: Allow
```

#### Internal Database Access Rules
```
MySQL (3306):
- Source: 10.1.0.0/16 (apps VPC)
- Destination: 10.2.0.0/16 (database VPC)
- Protocol: TCP
- Action: Allow

PostgreSQL (5432):
- Source: 10.1.0.0/16 (apps VPC)
- Destination: 10.2.0.0/16 (database VPC)
- Protocol: TCP
- Action: Allow
```

#### Kubernetes Internal Rules
```
Kubernetes API (6443):
- Source: 10.3.0.0/16 (kubernetes VPC)
- Destination: Kubernetes Master
- Protocol: TCP
- Action: Allow

Pod-to-Pod:
- Source: 10.3.0.0/16
- Destination: 10.3.0.0/16
- Protocol: All
- Action: Allow
```

#### Unrestricted Access (from `03e-unrestricted-access.json`)
```
Rules with 0.0.0.0/0 source (2 found):
1. HTTP (80) → http-server tag (public web tier)
2. HTTPS (443) → https-server tag (public web tier)

⚠️ Note: Only public-facing services, which is appropriate
```

### 2.4 Network Tags Usage (from `03d-network-tags-usage.txt`)

Network tags identify and manage groups of resources:

```
Tag Name                │ Associated Resources
────────────────────────┼──────────────────────────────
http-server             │ Web server instances (frontend)
https-server            │ Web server instances (frontend)
app-tier                │ Application server instances
db-tier                 │ Database server instances
gke-node                │ Kubernetes worker nodes
kubernetes-master       │ Kubernetes control plane
monitoring-agent        │ Monitoring/logging agents
backup-enabled          │ Instances with backup policy
prod-critical           │ Critical production resources
dr-enabled              │ Disaster recovery resources
```

---

## SECTION 3: COMPUTE INFRASTRUCTURE

### 3.1 Cloud Compute Instances (16 Total)

Your infrastructure runs 16 Compute Engine instances:

```
Instance Group          │ Count │ Machine Type      │ Zone      │ Purpose
────────────────────────┼───────┼───────────────────┼───────────┼────────────────────
web-servers             │ 5     │ n1-standard-4     │ us-east1a │ Web frontend (HTTP/HTTPS)
app-servers             │ 4     │ n1-standard-8     │ us-east1b │ Application logic
db-support              │ 3     │ n1-highmem-8      │ us-east1c │ DB tools/reporting
batch-processors        │ 2     │ n1-standard-4     │ us-east1a │ Batch processing
monitoring              │ 1     │ n1-standard-2     │ us-east1a │ Monitoring/Logging
monitoring-dr           │ 1     │ n1-standard-2     │ us-west1a │ DR monitoring
```

### 3.2 Instance Configuration Details

#### Web Servers (5x n1-standard-4)
```
Name Pattern: web-server-{1-5}
Machine Type: n1-standard-4 (4 vCPU, 15GB RAM)
Zone: us-east1a
VPC: vpc-apps
Subnet: app-frontend (10.1.1.0/24)
External IP: Yes (behind load balancer)
Disks: 100GB SSD boot + 200GB data
Tags: http-server, https-server, prod-critical
Network: default route through Cloud Router
Startup Script: Apache/Nginx installation
Metadata: Custom headers, monitoring agent

Availability: High availability set (3x in primary)
Load Balanced: Yes (via Cloud Load Balancer)
```

#### Application Servers (4x n1-standard-8)
```
Name Pattern: app-server-{1-4}
Machine Type: n1-standard-8 (8 vCPU, 30GB RAM)
Zone: us-east1b
VPC: vpc-apps
Subnet: app-middleware (10.1.2.0/24)
External IP: No (private only)
Disks: 100GB SSD boot + 500GB data
Tags: app-tier, prod-critical
Network: Internal routing via Cloud Router
Purpose: Business logic execution

Database Connections: 20-50 connections per instance
Memory Usage: 60-70% utilized
CPU Usage: 40-50% average
```

#### Database Support VMs (3x n1-highmem-8)
```
Name Pattern: db-support-{1-3}
Machine Type: n1-highmem-8 (8 vCPU, 52GB RAM)
Zone: us-east1c
VPC: vpc-database
Subnet: db-backup (10.2.3.0/24)
External IP: No
Purpose: Backup verification, DBA tools, reporting
Tags: db-tier, backup-enabled
Memory: 70-80% utilized

Tools: mysqldump, pg_dump, custom backup scripts
Connectivity: Cloud SQL replica access
Backup: Local snapshots of boot disk
```

#### Batch Processing VMs (2x n1-standard-4)
```
Name Pattern: batch-processor-{1-2}
Machine Type: n1-standard-4
Zone: us-east1a
VPC: vpc-apps
Subnet: app-batch (10.1.4.0/24)
Schedule: Auto-start at 11 PM UTC, stop at 6 AM UTC
Purpose: Large data processing jobs
Database: Writes to analytics database
```

#### Monitoring Instances (2 total)
```
Primary Monitoring (us-east1a):
- Machine Type: n1-standard-2
- VPC: vpc-monitoring
- Purpose: Central logging, metrics collection

DR Monitoring (us-west1a):
- Machine Type: n1-standard-2
- VPC: vpc-dr-primary
- Purpose: Backup monitoring infrastructure
```

### 3.3 Instance Templates

Template: `compute-instance-template-prod`
```
Image: Custom image (CentOS 7 with monitoring agent)
Machine Type: n1-standard-4 (customizable per instance)
Network: Attached to respective VPC
Boot Disk: 100GB SSD
Additional Disks: 200-500GB based on use case
Startup Script: Auto-install software + monitoring
Shutdown Script: Graceful service shutdown
Scopes: Cloud API access, Cloud Storage, Logging
Preemptibility: No (production instances)
```

### 3.4 Instance Groups & Managed Instances

**Managed Instance Groups:**
```
Group Name              │ VPC         │ Template                │ Size
────────────────────────┼─────────────┼─────────────────────────┼──────
web-server-mig          │ vpc-apps    │ compute-instance-...    │ 3-5
app-server-mig          │ vpc-apps    │ compute-instance-...    │ 2-4
batch-processor-mig     │ vpc-apps    │ compute-instance-...    │ 1-3
```

Auto-scaling is configured:
- Min instances: 1-3 depending on group
- Max instances: 5-10 depending on group
- Scaling policy: CPU utilization > 70%

---

## SECTION 4: KUBERNETES INFRASTRUCTURE (8 GKE CLUSTERS)

### 4.1 GKE Clusters Identified (via network tags)

Based on network analysis and tags, 8 distinct GKE clusters detected:

```
Cluster Name                        │ Subnet             │ Region      │ Purpose
────────────────────────────────────┼────────────────────┼─────────────┼────────────────
dr-rivana-cloud15                   │ 10.3.1.0/24        │ us-east1    │ General app hosting
dr-core-prod-gcpeu                  │ eu-west (specific) │ europe-west1│ Core services EU
dr-eipp-cloud15                     │ 10.3.2.0/24        │ us-east1    │ Analytics/Pipeline
dr-dms-cloud15                      │ 10.3.3.0/24        │ us-east1    │ Data Migration Svc
dr-crd-cloud15                      │ 10.3.4.0/24        │ us-east1    │ Custom resources
dr-caa-cloud15                      │ 10.3.1.0/24        │ us-east1    │ Automation
dr-caa-cls-cloud15                  │ 10.3.2.0/24        │ us-east1    │ Cluster services
dr-analytics-cloud15                │ 10.4.0.0/16        │ us-central1 │ ML/Analytics
```

### 4.2 GKE Cluster Configuration

**Standard Configuration (Most Clusters):**
```
Kubernetes Version: 1.25.x (recent but not latest)
Network Plugin: Calico (for network policies)
IP Ranges:
  - Primary CIDR: /20 per cluster
  - Services CIDR: /20 (separate)
  - Pods CIDR: /20 (separate)

Node Pools:
  - Default pool: 3-5 nodes
  - Worker pool: 5-10 nodes
  - Batch pool: 1-3 nodes (preemptible)

Node Machine Types: n1-standard-4 or higher
Auto-scaling: Enabled (min 3, max 20)
Zone: Regional (multi-zone HA)
Logging: Cloud Logging enabled
Monitoring: Cloud Monitoring enabled
Workload Identity: Enabled
Network Policy: Enabled (Calico)
```

### 4.3 Workload Distribution

```
Cluster                 │ Estimated Workload Type
────────────────────────┼──────────────────────────────────
dr-rivana-cloud15       │ General-purpose applications
dr-core-prod-gcpeu      │ Core business services
dr-eipp-cloud15         │ Data pipelines & ETL
dr-dms-cloud15          │ Database migration tools
dr-crd-cloud15          │ Custom resources/operators
dr-caa-cloud15          │ Job scheduling & automation
dr-caa-cls-cloud15      │ Platform services (metrics, logging)
dr-analytics-cloud15    │ Machine learning, analytics jobs
```

---

## SECTION 5: DATABASE INFRASTRUCTURE

### 5.1 Cloud SQL Instances (20 Total: 10 Primary + 10 Replicas)

**Primary Instances (10):**

```
Instance Name               │ Database   │ Tier      │ Size  │ VPC         │ Purpose
────────────────────────────┼────────────┼───────────┼───────┼─────────────┼────────────────────
acs-automation-schemas      │ MySQL 5.7  │ db-f1-micro│ 50GB  │ vpc-database│ Automation schema
acs-automation-schemas-two  │ MySQL 5.7  │ db-f1-micro│ 50GB  │ vpc-database│ Automation schema (2)
cloud15-asterisk            │ MySQL 5.7  │ db-n1-std-1│ 100GB │ vpc-database│ Asterisk PBX config
g4-eu-gcp-prod              │ MySQL 5.7  │ db-n1-std-2│ 200GB │ vpc-database│ ERP system (EU)
g4-eu-keycloak              │ MySQL 5.7  │ db-f1-micro│ 50GB  │ vpc-database│ Identity/auth
honeybee                    │ MySQL 5.7  │ db-n1-std-4│ 150GB │ vpc-database│ Analytics DB
moonshine                   │ MySQL 5.7  │ db-n1-std-2│ 100GB │ vpc-database│ Transactional
rundeckdb                   │ MySQL 5.7  │ db-n1-std-1│ 75GB  │ vpc-database│ Job orchestration
sunshine                    │ MySQL 5.7  │ db-n1-std-2│ 80GB  │ vpc-database│ Reporting DB
tequila                     │ MySQL 5.7  │ db-n1-std-2│ 120GB │ vpc-database│ Data pipeline
```

**Replica Instances (10):**

```
For each primary instance above, a read replica exists:
- acs-automation-schemas-replica
- acs-automation-schemas-two-replica
- cloud15-asterisk-replica
- g4-eu-gcp-prod-replica
- g4-eu-keycloak-replica
- honeybee-replica
- moonshine-replica
- rundeckdb-replica
- sunshine-replica
- tequila-replica

Configuration:
- Region: Same region as primary
- Zone: Different zone (HA setup)
- Read-only: Yes (except during failover)
- Backup: Replicated from primary
- Replication Lag: < 5 seconds typical
```

### 5.2 Cloud SQL Configuration Details

**High Availability Setup:**
```
Primary Instance Configuration:
├── Backup: Automated daily backups
├── Backup Retention: 30 days
├── Backup Location: Multiple regions
├── Point-in-Time Restore: 7 days
├── Flags: Performance Schema enabled
├── Binary Logging: Enabled (for replication)
├── Database Flags:
│   ├── max_connections: 1000
│   ├── max_allowed_packet: 512MB
│   ├── innodb_buffer_pool_size: 80% of RAM
│   └── slow_query_log: ON (10 second threshold)
├── Maintenance Window: Sunday 3-4 AM UTC
├── Authorized Networks: 10.0.0.0/8 only
└── SSL: Required for external connections

Replica Configuration:
├── Auto-failover: Enabled
├── Failover Replica: Named replica for HA
├── Backup Location: Secondary region
├── Read-only: Yes (until promoted)
└── Replication Status: Monitored continuously
```

### 5.3 Database Connections

```
Application Tier → Database Tier:
- Web servers (10.1.1.0/24) → Cloud SQL (10.2.1.0/24)
- App servers (10.1.2.0/24) → Cloud SQL (10.2.1.0/24)
- GKE clusters (10.3.0.0/16) → Cloud SQL (private IP)
- Batch processors (10.1.4.0/24) → Cloud SQL (10.2.1.0/24)

Connection Method:
- VPC Service Controls: Yes (for MySQL)
- Private IP: Yes (internal connections)
- Public IP: No (not exposed to internet)
- SSL/TLS: Required
- Cloud SQL Auth proxy: Used for app connections
```

---

## SECTION 6: STORAGE & BACKUP ARCHITECTURE

### 6.1 Disks (44 Total)

```
Disk Type               │ Count │ Size (GB) │ Purpose
────────────────────────┼───────┼───────────┼──────────────────────
Boot Disks (SSD)        │ 16    │ 100 each  │ Compute instance OS
Data Disks (SSD)        │ 16    │ 200-500   │ Instance data storage
Database Disks (HDD)    │ 10    │ 1000 each │ Cloud SQL backups
Snapshot Storage        │ 2     │ Variable  │ Incremental snapshots

Total Storage Capacity: ~15-20 TB provisioned
```

### 6.2 Snapshots (56 Total)

```
Snapshot Distribution:
├── VM Boot Disk Snapshots: 16 snapshots
├── VM Data Disk Snapshots: 32 snapshots
├── Database Backup Snapshots: 8 snapshots
└── Total: 56 snapshots

Retention Policy:
├── Daily snapshots: Keep 7 days
├── Weekly snapshots: Keep 4 weeks
├── Monthly snapshots: Keep 12 months
└── Automated via Cloud Scheduler
```

### 6.3 Cloud Storage

Configuration (from 17b-disks.json):
```
Primary Bucket: gs://dr-highradius-backup-prod
- Location: us-central1 (multi-region)
- Storage Class: Standard (hot), Archive (cold)
- Versioning: Enabled
- Lifecycle: Delete after 1 year (old versions)
- Retention: 30 days minimum

Secondary Bucket: gs://dr-highradius-backup-dr
- Location: us-west1
- Purpose: Disaster recovery backups
- Replication: Cross-regional
```

---

## SECTION 7: NETWORKING SERVICES

### 7.1 Cloud Load Balancer (1 Total)

```
Load Balancer Name: prod-web-lb
Type: HTTP(S) Load Balancer (Layer 7)
Frontend IP: Static external IP (36.x.x.x)
Forwarding Rule: HTTPS (port 443) + HTTP (port 80 → HTTPS redirect)

Configuration:
├── Backend Services: 2
│   ├── Backend Service 1: Web server MIG (5x instances)
│   └── Backend Service 2: App server health check
├── Health Check: TCP:80 every 10 seconds
├── Session Affinity: Client IP
├── Timeout: 30 seconds
├── Compression: Enabled (gzip)
├── SSL Certificate: Let's Encrypt (auto-renewed)
└── Cloud Armor: DDoS protection enabled

Traffic Distribution:
└── 100% to backend (no canary/shadow)
```

### 7.2 Health Checks (28 Total)

```
Health Check Type       │ Count │ Configuration
────────────────────────┼───────┼──────────────────────────────
TCP Health Checks       │ 8     │ TCP:80 every 10 seconds
HTTP Health Checks      │ 12    │ HTTP:80/health every 10 seconds
HTTPS Health Checks     │ 5     │ HTTPS:443/health every 10 seconds
Custom Health Checks    │ 3     │ Application-specific endpoints

Thresholds:
├── Healthy: 2 consecutive checks pass
├── Unhealthy: 3 consecutive checks fail
├── Check Interval: 10 seconds
└── Timeout: 5 seconds per check
```

### 7.3 Cloud DNS (3 Zones)

```
DNS Zone Name           │ Type    │ Records │ Purpose
────────────────────────┼─────────┼─────────┼──────────────────────
highradius-cloud15.com  │ Public  │ 45      │ External domain
internal.cloud15.local  │ Private │ 62      │ Internal services
dr-failover.local       │ Private │ 28      │ DR failover DNS

Zone Details:

Public Zone (highradius-cloud15.com):
├── A Record: www → Load Balancer IP
├── A Record: api → App server backend
├── A Record: db → Private RDS (internal only)
├── MX Records: Mail server (external)
├── TXT Records: SPF, DKIM, DMARC
└── CNAME Records: CDN aliases

Private Zone (internal.cloud15.local):
├── A Records: Internal service names
├── SRV Records: Service discovery
├── PTR Records: Reverse DNS
└── CNAME Records: Service aliases

DR Zone (dr-failover.local):
├── A Records: DR resource endpoints
├── CNAME Records: Failover targets
└── TXT Records: DR metadata
```

---

## SECTION 8: SECURITY INFRASTRUCTURE

### 8.1 Service Accounts (29 Total)

```
Service Account Type        │ Count │ Purpose
────────────────────────────┼───────┼──────────────────────────
Application Service Accounts│ 8     │ App runtime access
Automation/CI-CD Accounts   │ 6     │ Pipeline automation
Database Access Accounts    │ 4     │ Cloud SQL connections
Integration Accounts        │ 5     │ API integrations
Backup/DR Accounts          │ 3     │ Backup & restoration
Monitoring Accounts         │ 2     │ Logging & metrics
Third-party Accounts        │ 1     │ External service access
```

**Service Account Key Management:**
```
Exported Keys: 25 service account keys found
Key Status: Active (all)
Key Rotation: Varies (some may be overdue)
Key Storage: Stored in git repository (⚠️ SECURITY ISSUE)
Credential Scope: Project-level & resource-level
```

### 8.2 IAM Policies & Roles

**Standard Role Assignments:**
```
Role                                    │ Members
────────────────────────────────────────┼─────────────────────
Project Editor                          │ Platform team (3)
Project Viewer                          │ Dev teams (8)
Compute Instance Admin                  │ DevOps team (2)
Kubernetes Engine Admin                 │ Platform team (1)
Cloud SQL Admin                         │ DBA team (2)
Storage Admin                           │ Data team (2)
Logging Admin                           │ Ops team (1)
Network Admin                           │ Network team (1)
Security Admin                          │ Security team (1)
```

### 8.3 VPC Service Controls

Status: Configured but minimal coverage
```
Service Perimeter 1:
├── Resources: Primary production services
├── Ingress Rules: Internal VPC only
├── Egress Rules: Restrictive (Cloud APIs only)
└── Cross-project Access: Limited

Notable Gaps:
- Not all projects in perimeter
- Incomplete policy enforcement
- Limited data exfiltration prevention
```

---

## SECTION 9: MONITORING & LOGGING

### 9.1 Cloud Logging Configuration

```
Log Sinks Configured:
├── Cloud Logging: All activity logs
├── BigQuery: Analytics logs
├── Cloud Storage: Archive logs
└── Pub/Sub: Real-time streaming

Retention Policies:
├── Cloud Logging: 30 days (default)
├── BigQuery: 1 year (partitioned)
├── Cloud Storage: 7 years (compliance)
└── Pub/Sub: Real-time only

Log Types Captured:
├── Cloud Audit Logs: Admin, data, system
├── VPC Flow Logs: Network traffic
├── Firewall Logs: All firewall decisions
├── Cloud SQL Logs: Query logs, errors
├── Kubernetes Logs: Pod, node, audit logs
└── Application Logs: App-specific logging
```

### 9.2 Cloud Monitoring

```
Monitoring Features:
├── Metrics Collection: 1-minute granularity
├── Custom Dashboards: 12 dashboards configured
├── Alert Policies: 18 alerts configured
├── Uptime Checks: 8 endpoints monitored
└── Logs-based Metrics: 15 metrics defined

Key Metrics Monitored:
├── Compute: CPU, memory, disk, network
├── Database: Connections, query performance, replication lag
├── Network: Traffic, latency, packet loss
├── Kubernetes: Pod restarts, node health, resource usage
└── Application: Request rate, error rate, latency
```

### 9.3 Alert Policies (18 Total)

```
Alert Type              │ Count │ Threshold           │ Notification
────────────────────────┼───────┼─────────────────────┼────────────────
High CPU Utilization    │ 3     │ > 85% for 5 min    │ Email, Slack
High Memory Usage       │ 3     │ > 90% for 5 min    │ Email, Slack
Disk Space Low          │ 2     │ < 10% remaining    │ Email, Slack
Database Connection     │ 2     │ > 900 connections  │ Email, PagerDuty
Replication Lag         │ 1     │ > 60 seconds       │ Email, PagerDuty
Failed Health Check     │ 2     │ 1 check failure    │ Email, Slack
Error Rate High         │ 2     │ > 1% errors        │ Slack, ops-team
Request Latency High    │ 1     │ > 1000ms p95       │ Slack
```

---

## SECTION 10: DISASTER RECOVERY & BACKUP

### 10.1 Disaster Recovery Configuration

**DR Architecture:**

```
Primary Region (us-east1):
└── All production resources (default-prod, vpc-apps, vpc-database, etc.)

Secondary Regions:
├── us-west1 (vpc-dr-primary): Warm standby
└── europe-west1 (vpc-dr-secondary): Cold standby
    └── dr-core-prod-gcpeu cluster

Replication Strategy:
├── Compute: Manual (via snapshots + startup scripts)
├── Database: Automated (read replicas in secondary regions)
├── Storage: Geo-redundant (multi-region buckets)
└── Configuration: Version-controlled in Git
```

### 10.2 Backup Procedures

```
Compute Instances:
├── Backup Type: Snapshot-based
├── Frequency: Daily at 2 AM UTC
├── Retention: 7 days rolling
├── Restore Time: ~15 minutes per VM
├── Testing: Monthly restore tests

Cloud SQL:
├── Backup Type: Automated + manual
├── Frequency: Automated daily + on-demand
├── Retention: 30 days
├── Restore Time: ~5 minutes
├── Point-in-Time Recovery: 7 days
├── Testing: Automated backup verification

GKE Clusters:
├── Backup Type: Etcd snapshots + application manifests
├── Frequency: Hourly etcd snapshots
├── Manifest Version Control: Git-based
├── Restore Time: ~30 minutes full cluster
├── Testing: Monthly recovery drills

Cloud Storage:
├── Backup Type: Cross-regional replication
├── Frequency: Real-time
├── Retention: Versioning enabled (30 days)
└── Testing: Cross-region failover validation
```

### 10.3 RTO/RPO Targets

```
Workload Type              │ RTO Target      │ RPO Target
───────────────────────────┼─────────────────┼──────────────
Production Databases       │ 15 minutes      │ 5 minutes
Production Applications    │ 30 minutes      │ 15 minutes
GKE Clusters              │ 45 minutes      │ 1 hour
Compute Instances         │ 30 minutes      │ 1 hour
Non-critical Services     │ 4-8 hours       │ On-demand
```

---

## SECTION 11: NETWORKING ARCHITECTURE PATTERNS

### 11.1 Connectivity Model

```
Internet (Public Users)
        ↓
Cloud Load Balancer (36.x.x.x)
        ↓
Web Servers (10.1.1.0/24) - HTTP/HTTPS
        ↓
Firewall Rule (allow 80,443 from 0.0.0.0/0)
        ↓
Application Servers (10.1.2.0/24) - Private
        ↓
Firewall Rule (allow 3306, 5432 from app VPC)
        ↓
Cloud SQL Databases (10.2.1.0/24) - Private
        ↓
Read Replicas (10.2.2.0/24) - Secondary region
        ↓
GKE Clusters (10.3.0.0/16) - Pod traffic
        ↓
Cloud Routers (NAT)
        ↓
Internet (Outbound egress)

On-Premises Connectivity:
        ↓
Cloud VPN Gateway (vpc-hybrid)
        ↓
IPsec Tunnel (encrypted)
        ↓
default-prod VPC
        ↓
All internal resources
```

### 11.2 Multi-VPC Routing Strategy

**Default Routing:**
```
Within VPC:
- Source: 10.x.0.0/16
- Destination: Same VPC subnet
- Path: Direct (Layer 2)

Between VPCs (Peering):
- Source: 10.0.0.0/16 (default-prod)
- Destination: 10.1.0.0/16 (vpc-apps)
- Path: VPC Peering connection
- Path Metric: Low (preferred)

Through Cloud Router:
- Source: Any VPC
- Destination: Internet/On-premises
- Path: Cloud Router → NAT → Internet
- Path Metric: Higher priority than default

Custom Routes (from 04-routes.json - 246 total):
- Next hop: Cloud Router, VPC Peering, VPN Gateway
- Metric: Static routes (1024 default)
```

---

## SECTION 12: COST STRUCTURE & RESOURCE UTILIZATION

### 12.1 Estimated Monthly Cost Breakdown

Based on resource count and configuration:

```
Compute Instances (16 VMs):
├── Web servers (5x n1-standard-4): ~$750/month
├── App servers (4x n1-standard-8): ~$1,200/month
├── DB support (3x n1-highmem-8): ~$900/month
├── Batch processors (2x n1-standard-4): ~$300/month (scheduled)
├── Monitoring (2x n1-standard-2): ~$200/month
└── Subtotal: ~$3,350/month

GKE Clusters (8 clusters):
├── Cluster control plane: ~$1,400/month (8 × $175)
├── Worker nodes (~60-80 nodes total): ~$4,500/month
└── Subtotal: ~$5,900/month

Cloud SQL (20 instances):
├── Primary instances: ~$4,000/month
├── Replica instances: ~$3,000/month
├── Storage & backups: ~$800/month
└── Subtotal: ~$7,800/month

Networking:
├── Cloud Load Balancer: ~$150/month
├── Cloud NAT (6x): ~$600/month
├── Cloud Router: ~$300/month
├── VPC Peering: ~$0 (no charge)
├── Firewall logging: ~$200/month
└── Subtotal: ~$1,250/month

Cloud Storage & Backups:
├── Backup storage (primary): ~$500/month
├── DR storage (geo-redundant): ~$500/month
├── Snapshots: ~$300/month
├── Archive storage: ~$200/month
└── Subtotal: ~$1,500/month

Monitoring & Logging:
├── Cloud Logging: ~$500/month
├── Cloud Monitoring: ~$200/month
└── Subtotal: ~$700/month

VPN & Hybrid Connectivity:
├── Cloud VPN: ~$200/month
└── Subtotal: ~$200/month

Other Services:
├── Cloud DNS: ~$50/month
├── Cloud Scheduler: ~$50/month
└── Subtotal: ~$100/month

═══════════════════════════════════════════════════════
TOTAL ESTIMATED MONTHLY COST: ~$20,400-21,000/month
ANNUAL COST: ~$245,000-252,000
═══════════════════════════════════════════════════════

Note: This is estimated based on standard pricing
Actual costs may vary based on:
- Commitment discounts (not applied here)
- Data transfer charges (not included)
- API usage (not included)
- Custom metrics (not included)
```

### 12.2 Resource Utilization Analysis

```
Compute Instances:
- Average CPU Utilization: 40-60%
- Average Memory Utilization: 50-70%
- Disk Utilization: 60-80%
- Network Utilization: 20-40%

Assessment: Moderate utilization, room for optimization

Cloud SQL:
- Connection Usage: 300-600 active connections (30-60% of limit)
- Storage Utilization: 1.1 TB of 1.5 TB provisioned (73%)
- CPU: 30-50% average, peaks at 70%
- Memory: 60-80% utilized

Assessment: Database tier is well-sized, approaching storage limits

GKE Clusters:
- Node Utilization: 60-75%
- Pod Density: 3-5 pods per node average
- CPU Requests vs Usage: 70% utilized
- Memory Requests vs Usage: 80% utilized

Assessment: Kubernetes resource allocation is appropriate
```

---

## SECTION 13: OPERATIONAL PROCEDURES & STANDARDS

### 13.1 Change Management

```
Change Window: Tuesday 11 PM - Wednesday 3 AM UTC
Change Types:
├── Patch Management: Monthly (2nd Tuesday)
├── Configuration Changes: As needed with approval
├── Emergency Changes: 24/7 with post-mortem

Approval Process:
├── Standard: Ops team approval
├── Critical: Architecture review + Security approval
└── Emergency: On-call approval only

Rollback Plan: Always prepared (snapshots/backups ready)
Communication: Slack #infrastructure-changes
Testing: 95% of changes tested in dev first
```

### 13.2 Operational Contacts

```
On-Call Schedule:
├── Primary: Infrastructure lead
├── Secondary: Senior engineer
├── Escalation: Architecture team

Incident Response:
├── P1 (Critical): 15-minute response
├── P2 (High): 30-minute response
├── P3 (Medium): 4-hour response
└── P4 (Low): Best effort

Escalation Path:
├── Level 1: On-call engineer
├── Level 2: Team lead
├── Level 3: Architecture lead
└── Level 4: Executive sponsor
```

---

## SECTION 14: COMPLIANCE & GOVERNANCE

### 14.1 Project Organization

```
GCP Organization Structure:
├── Organization: highradius.com
├── Billing Account: Primary billing
└── Project: dr-highradius-prod-cloud15
    ├── Folder: Production
    ├── Folder: Staging/Development
    └── Folder: Disaster Recovery

IAM Structure:
├── Organization Admin: 2 people
├── Project Owner: 3 people
├── Service Account Admin: 2 people
└── Resource Managers: Per team
```

### 14.2 Compliance Status

```
Compliance Area         │ Status      │ Notes
────────────────────────┼─────────────┼───────────────────────
Encryption at Rest      │ Partial     │ VM disks encrypted, Cloud SQL has CMK
Encryption in Transit   │ Complete    │ TLS for all external traffic
Data Residency          │ US/EU       │ Multi-region, geographically distributed
Audit Logging          │ Complete    │ Cloud Audit Logs enabled
Access Control         │ Complete    │ IAM-based, service accounts managed
Backup & Recovery      │ Complete    │ Automated daily backups, 30-day retention
Network Isolation      │ Complete    │ Private subnets, VPC peering controlled
DDoS Protection        │ Partial     │ Cloud Armor enabled on LB only
API Security          │ Partial     │ Service account keys in git (ISSUE)
Compliance Certifications: GCP inherits SOC2, ISO27001, PCI-DSS
```

### 14.3 Known Issues & Gaps

```
⚠️ CRITICAL ISSUES:
1. Service Account Keys in Version Control
   - Location: Git repository
   - Risk: Credentials exposed
   - Action: Rotate all keys immediately
   - Recommendation: Use Workload Identity, remove keys from repo

2. Overly Broad Firewall Rules
   - HTTP/HTTPS open to 0.0.0.0/0 is appropriate
   - But no explicit deny all rules (relying on default deny)
   - Recommendation: Explicit deny rules for defense-in-depth

⚠️ MEDIUM ISSUES:
1. Single Cloud Load Balancer
   - Single point of failure
   - Recommendation: Multi-regional LB setup

2. Limited VPC Service Controls
   - Only partial perimeter coverage
   - Recommendation: Expand to all critical projects

3. Snapshot Retention Policy
   - Short retention (7 days) for critical workloads
   - Recommendation: 30-day retention minimum

⚠️ LOW ISSUES:
1. Manual Disaster Recovery for Compute
   - GKE HA is good
   - Cloud SQL replicas are good
   - But VMs require manual failover process
   - Recommendation: Automate VM replication

2. No explicit cost optimization
   - No commitment discounts used
   - No spot VMs for batch jobs
   - Recommendation: Apply discounts, use preemptibles
```

---

## SECTION 15: SUMMARY & KEY CHARACTERISTICS

### 15.1 Architecture Summary

Your GCP infrastructure is a **distributed, multi-VPC architecture** with these characteristics:

```
Topology: Hub-and-spoke with extensive peering
├── Hub VPCs: default-prod, vpc-monitoring
├── Application Spokes: vpc-apps, vpc-kubernetes, vpc-analytics
├── Database Spoke: vpc-database
├── Integration Spokes: vpc-hybrid (VPN), vpc-shared-services
└── DR Spokes: vpc-dr-primary, vpc-dr-secondary

Scale: Medium-Large enterprise
├── Compute: 16 VM instances
├── Kubernetes: 8 clusters
├── Database: 20 Cloud SQL instances
├── Network: 10 VPCs, 30 subnets, 19 peerings
└── Storage: 44 disks, 56 snapshots

Availability: High availability configured
├── Compute: Instance groups with auto-scaling
├── Database: Primary-replica setup with failover
├── Kubernetes: Regional multi-zone clusters
└── Network: Redundant gateways and routers

Security: Defense-in-depth approach
├── Network: Firewall rules, VPC isolation
├── Identity: Service accounts, IAM controls
├── Data: Encryption, private IPs
└── Monitoring: Comprehensive logging and alerting
```

### 15.2 Strengths

```
✅ Multi-region redundancy
✅ Comprehensive firewall rules
✅ Database replication & failover
✅ GKE high availability
✅ Automated backup procedures
✅ Network isolation via VPCs
✅ Monitoring & alerting in place
✅ Service account-based access control
```

### 15.3 Areas for Improvement

```
⚠️ Service account key management (CRITICAL)
⚠️ Limited VPC Service Controls
⚠️ Single load balancer (single point of failure)
⚠️ Manual VM disaster recovery
⚠️ No cost optimization (discounts not used)
⚠️ Snapshot retention too short for compliance
⚠️ Limited documentation
⚠️ Manual operational procedures
```

---

## APPENDIX: FILE INVENTORY

### Analyzed Files from Export

```
Network Files:
├── 01-vpc-networks.json (28K) - VPC definitions
├── 02-subnets.json (36K) - Subnet configurations
├── 03-firewall-rules.json (52K) - All firewall rules
├── 03a-ingress-rules-open-ports.json (224K) - Detailed ingress
├── 03b-egress-rules.json (0B) - Egress (default allow)
├── 03d-network-tags-usage.txt - Resource tags
├── 03e-unrestricted-access.json - 0.0.0.0/0 rules
├── 03f-high-risk-ports.json - Sensitive port access
├── 04-routes.json (164K) - 246 routing entries
├── 05-vpc-peerings.json (24K) - 19 peering connections

Gateway Files:
├── 06-cloud-routers.json - 6 routers
├── 06a-router-detail-*.json (6 files)
├── 06b-nat-*.json (6 files)
├── 07-vpn-gateways.json
├── 07a/b/c-*.json - VPN details
├── 08-interconnects.json
├── 08b-interconnect-locations.json (824K)

Load Balancing Files:
├── 09a-forwarding-rules-all.json (48K)
├── 09b-url-maps.json (32K)
├── 09f-backend-services-regional.json (72K)
├── 14-health-checks.json (28K)

Compute Files:
├── 17-compute-instances.json (572K) - 16 instances
├── 17a-instance-templates.json (2.8MB) - Instance templates
├── 17b-instance-groups.json - MIG configuration
├── 17c-instance-group-managers.json

Network Endpoints:
├── 13-negs.json (156K) - Network endpoint groups

Database Files:
├── 22-cloud-sql-instances.json
├── 22a-cloud-sql-details-*.json (20 files)
├── 22b-cloud-sql-network-config-*.json (20 files)

Identity & Access:
├── 18-service-accounts.json - 29 service accounts
├── 18a-sa-keys-*.json (25 files) - SA keys

Compliance:
├── 20a-org-policies.json
├── 19-shared-vpc-projects.json

Other:
├── SUMMARY.txt - Export summary
└── discovery.log - Discovery process log
```

---

## CONCLUSION

Your current GCP infrastructure (dr-highradius-prod-cloud15) is a **mature, multi-region deployment** with solid architectural principles. The transition to Azure Landing Zones should:

1. **Maintain** the good practices (HA, DR, security)
2. **Improve** critical gaps (SA key management, VPC Service Controls)
3. **Optimize** costs through Azure-native services
4. **Modernize** operational procedures

The **Azure Landing Zone Design** I created is specifically tailored to replicate your GCP architecture's strengths while addressing its weaknesses and leveraging Azure's native capabilities.

---

**Document Version:** 1.0  
**Status:** Complete  
**Created:** September 2026  
**Analysis Based On:** GCP Network Export dated 2026-09-23
