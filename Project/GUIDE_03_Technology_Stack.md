# ⚙️ Section 3: Technology Stack Explanation

## Overview

This section explains **why each technology was chosen** for specific roles in the Excess Management System microservices architecture. Understanding the "why" is crucial for system design interviews—you need to justify technology choices based on requirements, not just pick familiar tools.

---

## 🏗️ Technology Stack at a Glance

```
┌─────────────────────────────────────────────────────┐
│         EXCESS MANAGEMENT SYSTEM TECH STACK         │
├─────────────────────────────────────────────────────┤
│                                                     │
│ MESSAGING & SCHEDULING                              │
│  • Kafka (on-prem): Source of truth for messages    │
│  • EventBridge (AWS): Scheduled triggers            │
│  • SQS (AWS): Queue-based decoupling                │
│                                                     │
│ COMPUTE                                             │
│  • AWS Lambda: Serverless functions (3 services)    │
│  • Spring Boot: Stateful task processing API        │
│                                                     │
│ DATA STORES                                         │
│  • Oracle: Raw message persistence                  │
│  • JBPM DB: Workflow orchestration state            │
│  • DynamoDB: Fast metadata lookups                  │
│  • RDS: Audit & compliance logging                  │
│  • S3: Report storage (durable)                     │
│                                                     │
│ SECURITY & ACCESS                                   │
│  • API Gateway: Request routing & OAuth2/JWT        │
│  • IAM: Least privilege for Lambda functions        │
│  • KMS: Encryption keys                             │
│                                                     │
│ MONITORING                                          │
│  • CloudWatch: Metrics, logs, alarms                │
│  • X-Ray: Distributed tracing                       │
│  • VPC: Network isolation                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 🎯 Technology Decisions Deep Dive

---

## 1️⃣ AWS LAMBDA (vs. EC2, vs. Kubernetes)

### The Decision

**Chosen**: AWS Lambda for Ingestion, Workflow, and Email services

**Alternatives Considered**:
- EC2 (Virtual Machines)
- Kubernetes
- On-premises servers

### Why Lambda?

| Factor | Lambda | EC2 | Kubernetes |
|--------|--------|-----|-----------|
| **Cost Model** | $0.0000002/invocation | $0.96/day (24/7 minimum) | $0.125/core-hour (minimum 1 node) |
| **Scaling** | Automatic, milliseconds | Manual setup, minutes | Automatic, seconds |
| **Cold Starts** | Yes (100–500ms) | No | No |
| **Execution Limit** | 15 minutes | Unlimited | Unlimited |
| **Use Case Fit** | ⭐⭐⭐⭐⭐ Perfect | ⭐⭐ Poor | ⭐⭐⭐ Good |
| **Operational Overhead** | None | High | Very High |

### Detailed Analysis

#### Cost Comparison

```
Excess Management System Monthly Cost

LAMBDA (Actual):
  Ingestion runs:   8 hours/day × 20 business days = 160 hours/month
  Price: 160 hours = 576,000 seconds
  @ $0.0000002/100ms = $0.005/month ✓ Negligible
  Plus: Memory cost ($0.000006667 per GB-second)
  Total estimate: ~$5-10/month

EC2 (if used):
  m5.2xlarge: $0.384/hour
  24 hours × 30 days = $276/month (minimum)
  Auto-scaling for peaks: $300-400/month
  Total: ~$350-400/month

SAVINGS WITH LAMBDA: $350-390/month
  = $4,200–4,680/year
  × Multi-client deployment = $20,000+ annual savings company-wide
```

#### Our Use Case Fits Lambda

```
✅ Event-Driven: Triggered by schedules (EventBridge) & queues (SQS)
✅ Stateless: Each invocation is independent
✅ Bursty: Not constant load (spike at 9 AM ingestion)
✅ Short-running: Ingestion takes < 5 minutes, Workflow < 1 minute
✅ No 24/7 requirement: Runs business hours only

❌ NOT Suitable for Lambda:
  • Long-running batch jobs (> 15 min limit)
  • Constant background processes
  • Stateful, persistent connections
  • Very frequent invocations (> 100,000/sec = cost spike)
```

#### Why NOT Kubernetes?

```
Kubernetes is powerful but overkill for Excess Management System:

Why overkill:
  • Need: Schedule 4 jobs daily
  • Kubernetes: Container orchestration, pod scheduling, networking
  • Overhead: Deploy control plane, manage nodes, monitor clusters
  • Team size: Would need DevOps engineer just to manage K8s

Real Kubernetes value:
  ✓ Multiple services with complex communication
  ✓ Need persistent storage, pet containers
  ✓ Team already has K8s expertise
  ✓ Scale to millions of requests/day

Excess Management System: Simple event-driven functions
  → Lambda is perfect fit
  → Kubernetes is over-engineered
```

### Lambda Drawback: Cold Starts

```
What's a cold start?

First invocation of Lambda:
  1. AWS spins up container
  2. Runs initialization code
  3. Executes function
  Time: 100–500ms

Subsequent invocations (warm):
  Time: 10–50ms

For Gold Report:
  • Ingestion: Runs every morning → 1 cold start acceptable
  • Email: Runs daily at 5 PM → 1 cold start acceptable
  • Workflow: Sustained SQS traffic → Keeps Lambda warm ✓

Mitigation:
  • Provisioned concurrency: Keep Lambdas warm 24/7 (but costs more)
  • For our workload: Not needed
```

---

## 2️⃣ AWS SQS (Message Queue)

### The Decision

**Chosen**: SQS for decoupling Ingestion → Workflow

### Why SQS?

#### Requirement: Decouple Services

```
Without SQS (Direct Lambda Calls):
  Ingestion Lambda calls Workflow Lambda directly
    → Tight coupling (must know Workflow Lambda exists)
    → No buffering (sudden spike = failures)
    → No retries (message lost on failure)

With SQS:
  Ingestion publishes to SQS
  Workflow consumes from SQS
    ✓ Loose coupling (services don't know each other)
    ✓ Buffering (queue absorbs spikes)
    ✓ Built-in retries (messages NOT lost)
    ✓ Dead-letter queue (failed messages captured for analysis)
```

#### Why NOT Direct Lambda Calls?

```
Scenario: Ingestion publishes 2,000 messages in 2 minutes

Direct Lambda Calls:
  Ingestion → calls Workflow Lambda 2,000 times
    → Workflow Lambda can't handle 1,000/minute
    → Many invocations fail with "too many requests"
    → Messages lost; need manual retry

SQS:
  Ingestion → publishes 2,000 to SQS in 2 minutes ✓
  SQS queue: Buffers messages safely
  Workflow Lambda: Picks up at its own pace (20 concurrent)
    → All messages eventually processed
    → No message loss
    → Automatic retries
```

#### Why NOT Kafka?

```
Gold Report uses both Kafka and SQS—why?

Kafka (On-Premises):
  ✓ Source of truth for upstream systems
  ✓ High throughput (millions/sec)
  ❌ Operational overhead (cluster management)
  ❌ Not ideal for within-AWS service communication

SQS (AWS-Managed):
  ✓ Perfect for within-AWS async communication
  ✓ Fully managed (no ops overhead)
  ✓ Simple, reliable queue semantics
  ✓ Tight integration with Lambda

Decision:
  Kafka → On-prem messages from clients
  SQS → Internal service communication
```

### SQS Features We Leverage

```
1. Reserved Concurrency + SQS = Load Control
   Workflow Lambda reserved at 20 concurrent executions
   SQS automatically throttles: Only 20 messages per second
   Result: JBPM DB never overwhelmed

2. Visibility Timeout
   Message picked by Lambda, hidden from queue for 5 minutes
   If Lambda fails → message becomes visible again → retry
   Prevents duplicate processing (mostly)

3. Dead-Letter Queue (DLQ)
   Messages that fail 3 times → sent to DLQ
   Operations team can investigate why they failed
   Critical for detecting bugs

4. Long Polling
   Lambda polls SQS efficiently (10-second wait)
   No wasted API calls
   Cloud-native, cost-effective
```

📌 **2026 SUGGESTION:**  
Modern systems (2026 standards) often layer **RabbitMQ** alongside SQS for advanced routing, or **Apache Kafka** for event streaming at scale. SQS remains perfect for AWS-native systems, but enterprises evaluate multi-cloud flexibility. View `ADVANCED_2026_Technology_Standards.md` for modern messaging evolution.

---

## 3️⃣ AWS EVENTBRIDGE (Scheduling)

### The Decision

**Chosen**: EventBridge for scheduled triggers (Ingestion & Email services)

### Why EventBridge?

#### Requirement: Schedule Jobs at Specific Times

```
Business Requirements:
  • Ingestion: Run at 9:00 AM (business hours start)
  • Email: Run at 5:00 PM daily (end of day reporting)
  • Skip weekends (already handled by cron rules)

Options:
  1. EC2 Cron Job: Manual setup, not cloud-native
  2. Lambda + CloudWatch Events: Old way (now EventBridge)
  3. EventBridge (Recommended): Modern, serverless, powerful
```

#### EventBridge Advantages

```
✅ Serverless: No infrastructure to manage
✅ Routing: Rules-based routing (can fan out to multiple targets)
✅ Cross-Service: Trigger Lambda, SQS, SNS, etc.
✅ Filtering: Can filter events before sending
✅ DLQ Support: Failed invocations go to DLQ
✅ Retry Policy: Automatic retries with exponential backoff

Example Rule:
  Trigger: 0 9 ? * MON-FRI (9 AM weekdays)
  Target: Ingestion Lambda
  Retry policy: 3 times, exponential backoff
  DLQ: SNS topic (alert if Lambda fails)
```

#### Why NOT Cron + EC2?

```
Cron (Unix Scheduler):
  ❌ Requires EC2 instance running 24/7 just to schedule
  ❌ No visibility into whether job ran
  ❌ No built-in retry or monitoring
  ❌ Hard to debug failures

EventBridge:
  ✓ Managed by AWS (no EC2 needed)
  ✓ Full visibility in CloudWatch
  ✓ Built-in retry, DLQ, filtering
  ✓ Scales automatically
```

---

## 4️⃣ JBPM (Workflow Engine)

### The Decision

**Chosen**: JBPM for workflow orchestration and task management

### Why JBPM?

#### Context: Legacy Constraint

```
Gold Report inherited JBPM (End-of-Life version)
  ❌ No longer actively developed
  ❌ Limited community support
  ❌ Hard to find expertise

BUT:
  ✓ Already deeply integrated in the original system
  ✓ Business logic expressed in BPMN (visual workflows)
  ✓ Ripping it out would require massive rewrite
  ✓ Still functional for this use case

Decision: Keep JBPM, refactor around it
```

#### JBPM's Role in Gold Report

```
JBPM Responsibilities:
  1. Store workflow instance state
     (which tasks completed, who approved, audit trail)

  2. Manage task lifecycle
     (created → assigned → in-progress → completed)

  3. Enforce business logic
     (rules, approvals, conditional routing)

  4. Provide task lists to end-users
     (what tasks are assigned to me?)

Why JBPM ?
  • Business people designed workflows in BPMN
    (visual, non-technical)
  • Execution engine handles complexity
    (state machine, conditional logic, retries)
```

#### JBPM + Microservices Coordination

```
Old Approach (Monolithic):
  All JBPM operations in same process
  Problem: Scalability bottleneck

New Approach (Microservices):
  Workflow Lambda interacts with JBPM via REST API
  Workflow state stored in JBPM DB
  Metadata mirrored to DynamoDB for fast queries

Benefit:
  • JBPM remains stable (doesn't change)
  • Can scale Workflow Lambda horizontally
  • Can improve queries via DynamoDB
  • Separate JBPM DB prevents lock contention with business data
```

#### JBPM Database: Separate Instance (Database Per Service)

**Microservices Best Practice**: Each service owns its database.

**Our Implementation**:
```
Ingestion Service → Oracle DB (raw messages)
Workflow Service → JBPM DB (workflow instances) ← SEPARATE DATABASE
Reporting Service → DynamoDB (reporting cache)
```

**Why JBPM DB is Separate**:

| Reason | Impact |
|--------|--------|
| **Database Per Service** | Microservices pattern—each service owns its data |
| **Lock Contention Prevention** | JBPM rapid writes (2,000 instances/10min) won't block business data reads |
| **Independent Scaling** | If workflow load grows to 10K instances/day, can upgrade JBPM DB without touching Oracle |
| **Technology Independence** | Can replace JBPM engine (with Temporal.io) without modifying Oracle or business data |
| **Compliance Separation** | Business data archived 7 years; JBPM data transient (5-min retention) |

**JBPM Database Options** (AWS Context):
- **PostgreSQL RDS**: JBPM officially supports PostgreSQL (most common in AWS)
- **Separate Oracle Instance**: Your team may maintain second Oracle (costlier)
- **MySQL/Aurora**: JBPM supports MySQL-compatible backends

**No Ambiguity in Microservices**:
```
Monolithic (Old):
  • Single Oracle instance, multiple schemas (BUSINESS + JBPM)
  • Problem: Lock contention at scale

Microservices (New):
  • Ingestion → Oracle DB
  • Workflow → JBPM DB (separate)
  • No sharing = no contention
```

#### JBPM Challenges in Microservices

```
Challenge 1: JBPM Exposes REST API
  Solution: Workflow Lambda calls REST API for instance creation

Challenge 2: Need Both JBPM & DynamoDB
  Reason: JBPM is slow for "get all completed tasks"
  Solution: DynamoDB maintains fast index, eventual consistency OK

Challenge 3: Legacy Tech, Limited Support
  Mitigation: Encapsulate in Workflow Lambda
  Future: Can replace workflow engine without re-architecting
```

---

## 5️⃣ ORACLE DATABASE

### The Decision

**Chosen**: Oracle for raw message persistence

### Why Oracle?

#### Requirement: Store Incoming Excess Messages

```
Functional Requirement:
  Store raw Kafka messages (JSON, large payloads)
  Status tracking (NEW → PROCESSED → COMPLETED)
  Audit trail (created_by, created_date, etc.)
  Compliance: Keep records for 7 years

Oracle Chosen Because:
  ✓ Already enterprise standard in organization
  ✓ Strong ACID: Guarantees data consistency
  ✓ Compliance: Mature audit logging
  ✓ Existing integrations: Other systems already query Oracle
```

#### Oracle's Role

```
Oracle Table: EXCESS_MESSAGE
  ├─ ID (unique message ID, idempotency key)
  ├─ RAW_PAYLOAD (JSON message from Kafka)
  ├─ STATUS (NEW, IN_PROGRESS, COMPLETED, FAILED)
  ├─ CREATED_DATE
  ├─ PROCESSED_DATE
  ├─ JBPM_INSTANCE_ID (cross-reference to workflow)
  └─ AUDIT_INFO

Usage:
  Ingestion Lambda: INSERT (status=NEW)
  Workflow Lambda: UPDATE (status=IN_PROGRESS, store JBPM_INSTANCE_ID)
  Task Execution: UPDATE (status based on user action)
  Reporting: SELECT WHERE status=COMPLETED
```

#### Why Not Just DynamoDB?

```
DynamoDB is great, but not right here:

DynamoDB Mismatch:
  ❌ No transactions (if you need ACID)
  ❌ Compliance teams prefer relational DB for audits
  ❌ Already licensed (Oracle); why duplicate?
  ❌ Hard joins (if messages need related data)

Oracle Fit:
  ✓ ACID guarantees ensure no lost messages
  ✓ SQL queries for auditing and compliance
  ✓ Already integrated with other systems
  ✓ Team familiar with Oracle
```

---

## 6️⃣ DYNAMODB (Fast Lookups)

### The Decision

**Chosen**: DynamoDB for message metadata and payloads

### Why DynamoDB?

#### Requirement: Fast Lookups for Reporting

```
Monolithic Problem:
  JBPM DB query: SELECT * FROM process_instance WHERE status='COMPLETED'
  Result: ~5 seconds for 2,000 records (joins, complex schema)

New Requirement (Microservices):
  Email service needs to generate report at 5 PM
  Users can't wait 5 seconds for UI
  Need sub-100ms response

Solution: DynamoDB

DynamoDB Advantages:
  ✓ Flat schema: Key → Value (very simple)
  ✓ Fast: < 10ms for single item, < 100ms for scans
  ✓ Scalable: Auto-scales to millions of requests
  ✓ No locking: Eventual consistency acceptable for metadata
```

#### DynamoDB Schema for Excess Management System

```
Table: goldreport-instances
Partition Key: message_id (unique identifier)
Sort Key: created_date (enables range queries)

Attributes:
  {
    "message_id": "MSG-123",
    "created_date": "2026-03-01",
    "status": "COMPLETED",
    "jbpm_instance_id": "1234567",
    "payload": {...raw message...},
    "completed_date": "2026-03-01",
    "completed_by": "user@company.com",
    "decision": "APPROVED"
  }

Queries:
  ✓ Get single message: O(1) → Key lookup
  ✓ Get all completed today: O(N) → Scan with filter (fast)
  ✓ Get by user: O(N) → Single index scan (fast)
```

#### Why Not Relational Database for This?

```
Relational (Oracle or RDS):
  Pros:
    ✓ Structured schema
    ✓ ACID transactions
    ✓ Complex joins

  Cons:
    ❌ Slower for simple lookups (indexes needed)
    ❌ Scale-up (we'd hit CPU limits)
    ❌ Requires DBA tuning

DynamoDB:
  Pros:
    ✓ Simple lookups (< 10ms)
    ✓ Infinite scale (if you pay)
    ✓ No tuning needed (managed service)

  Cons:
    ❌ No complex joins
    ❌ Eventual consistency (OK for our case)
    ❌ More expensive at extreme scale

For this use case: DynamoDB wins
```

📌 **2026 SUGGESTION:**  
Modern cloud systems (2026 standard) add a **Redis/ElastiCache layer in front of DynamoDB** for multi-tier caching. Benefits: L1 cache (Redis) serves 80% of reads in <5ms, L2 (DynamoDB) handles rest. This pattern dramatically reduces database load under peak traffic. Interview take-away: Understand caching layering and when to add it. See `ADVANCED_2026_Technology_Standards.md` Section 2 for complete Redis implementation.

---

## 7️⃣ RDS (Audit & Compliance)

### The Decision

**Chosen**: RDS (Relational Database Service) for audit logging

### Why RDS?

#### Requirement: Compliance & Audit Trails

```
Business Requirement:
  "Prove that emails were sent to stakeholders"
  "Who processed which task and when?"
  "Full audit trail for regulatory compliance"

Solution: Audit RDS

RDS Schema:
  Table: email_audit_log
    ├─ ID (primary key)
    ├─ EMAIL_SENT_DATE
    ├─ RECIPIENTS (stakeholders)
    ├─ REPORT_DATE
    ├─ STATUS (success/failed)
    ├─ RECORD_COUNT (how many instances in report)
    └─ VERIFICATION_CODE (unique, for traceability)

Usage:
  Email Lambda: INSERT audit record after sending email
  Compliance team: Query for proof of delivery
```

#### Why Not DynamoDB for Audit?

```
Audit logs need:
  ✓ Structured schema (columns, data types)
  ✓ SQL queries ("show me all emails from March")
  ✓ Compliance standards (immutable append-only)
  ✓ Easy export to compliance tools

RDS provides all, DynamoDB doesn't
  (DynamoDB is good for fast app data, not compliance logs)
```

---

## 8️⃣ S3 (Report Storage)

### The Decision

**Chosen**: Amazon S3 for storing generated reports

### Why S3?

```
Requirement:
  Store CSV/Excel/PDF reports
  Make available for download by stakeholders
  Durable storage (99.999999999% durability)

Why S3:
  ✓ Durable: 11 nines (reports won't get lost)
  ✓ Cheap: ~$0.023/GB/month
  ✓ Simple: Just put file and URL
  ✓ Scalable: Unlimited storage
  ✓ Integrated: Email service can generate pre-signed URLs

Why NOT local filesystem:
  ❌ Lambda is ephemeral (file system deleted after execution)
  ❌ No shared storage (multiple Lambda instances can't access)
  ❌ Not durable (no backup)

Why NOT RDS/Database:
  ❌ BLOBs in databases are slow
  ❌ Wastes database capacity for file storage
  ❌ Not designed for large files
```

---

## 🔐 SECURITY TECHNOLOGIES

### API Gateway + OAuth2/JWT

```
Requirement: Secure access to Task Execution Service

Approach:
  1. All requests go through API Gateway
  2. API Gateway validates JWT token (OAuth2)
  3. Only authenticated requests reach backend
  4. Spring Boot app trusts JWT (already validated)

Flow:
  User logs in → Gets JWT token
  User sends API request → Includes token in header
  API Gateway → Validates token signature
  JWT Contains: user ID, roles, permissions
  Backend → Uses claims for authorization

Why This Architecture:
  ✓ Centralized security (not in each service)
  ✓ Token-based (stateless, scalable)
  ✓ Standard (OAuth2 widely recognized)
  ✓ Enables 3rd-party integrations
```

---

## 9️⃣ EC2 (Compute for Task Execution Service)

### The Decision

**Chosen**: EC2 instance for Task Execution Service (Spring Boot + JBPM hosting)

**Alternatives Considered**:
- Lambda (no – state/session management issues)
- ECS/Kubernetes (possible, but EC2 simpler for this workload)
- On-premises (not considered – cloud-native preference)

### Why EC2?

#### Requirement: Stateful User Application with Session Management

```
User Experience Flow:
  1. User logs in → Session created
  2. Assigned tasks fetched from JBPM
  3. User opens task detail → State maintained in Spring Boot
  4. User modifies and saves → Session context used for audit
  5. Next request: Session still active → No re-authentication
  
This requires:
  ✓ Persistent connections
  ✓ In-memory session state
  ✓ JBPM running on same server (low-latency)
```

#### Why Not Lambda?

```
Lambda Limitation:
  Each cold start = new container
  Session lost between invocations
  Users see: login again? (frustrating)
  
EC2 Solution:
  Always-running instance
  Sessions persist across requests
  Same container for entire user session
  Result: Smooth UX
```

#### JBPM Co-location Benefit

```
Separate Servers:
  Spring Boot → REST call → JBPM Server (over network)
  Latency: 10-50ms per call
  Complexity: Cross-server coordination

Same EC2 Instance:
  Spring Boot → Direct JBPM (in-process)
  Latency: < 1ms per call
  Simplicity: Local transactions, same Java process
```

### EC2 Configuration for Excess Management System

```
Instance Type: t3.large or t3.xlarge
  CPU: 2-4 vCPU (handles 50-100 concurrent users)
  Memory: 8-16 GB (for JBPM cache + Spring Boot heap)
  
Storage: 100 GB EBS (for JBPM database + logs)
  Backup: Daily snapshots
  
Network: 
  VPC: Private subnet (access via API Gateway)
  Security Group: Only inbound on 8080 (Spring Boot)
  
Monitoring: CloudWatch (CPU, memory, disk, application metrics)
```

### Trade-offs: EC2 vs Other Options

| Factor | EC2 | ECS | Lambda |
|--------|-----|-----|--------|
| **State Management** | ✅ Excellent | ✅ Good | ❌ Poor |
| **Session Support** | ✅ Native | ✅ With ALB | ❌ Requires Redis |
| **JBPM Co-location** | ✅ Perfect | ⚠️ Possible | ❌ Can't host |
| **Cost (50 users concurrent)** | ✅ ~$100/mo | ✅ ~$150/mo | ❌ ~$300+/mo |
| **Ops Overhead** | ⚠️ Moderate | ✅ Lower | ✅ None |
| **Cold Starts** | ✅ None | ✅ None | ❌ 500ms+ |
| **Complexity** | ✅ Simple | ⚠️ Moderate | ✅ Simple |

**For this use case**: EC2 wins for cost and simplicity.

### 2026 Evolution: Consider ECS with ALB

📌 **2026 SUGGESTION:**  
If scaling to 500+ concurrent users becomes necessary (2026+), consider **migrating to ECS**:
- Containerize Spring Boot + JBPM as single Docker image
- Run on ECS with Application Load Balancer (ALB)
- ALB provides sticky sessions (routes user to same ECS task)
- Auto-scaling: Add/remove tasks based on CPU
- Benefit: Managed scaling without manual intervention

But for current load (50-100 users): **EC2 is optimal**.

---

## � React UI Deployment (S3 + CloudFront)

### The Decision (Current Implementation)

**Chosen Today**: React UI served directly from EC2 instance alongside Spring Boot

**Future Consideration** (if 200+ global concurrent users): S3 + CloudFront CDN

### Why EC2 Today?

**Current State Works Well For**:
- Cost: $0 additional (React files served by existing EC2)  
- Simplicity: No separate deployment infrastructure
- Current users: 50-100 concurrent US-based
- Acceptable latency: 20ms for US users

**When To Upgrade To S3 + CloudFront**:
- User base expands to 200+ concurrent
- Geographic distribution across regions (USA + EU + APAC)
- EU users complain about 150ms latency
- Performance becomes a business requirement

---

### Current Implementation: React on EC2

**Architecture**:
```
┌────────────────────────────────────────────┐
│ EC2 Instance (t3.large, $100/month)        │
│ ├─ Port 80/443                             │
│ ├─ React UI (static files: HTML, JS, CSS)  │
│ ├─ Spring Boot Backend (REST API)          │
│ └─ JBPM Workflow Engine                    │
└────────────────────────────────────────────┘
         ↓
    User Browser
    • US user: 20ms latency ✓
    • HTML/JS/CSS served by Spring Boot
    • API calls to same server
```

**How It Works**:
```bash
# React build
npm run build
# Output: dist/ with index.html, js/*, css/*, assets/

# Deployed to EC2
Spring Boot serves:
  GET  /          → index.html
  GET  /static/*  → js, css, images  
  GET  /api/*     → REST endpoints
  POST /api/*     → REST endpoints
  
# Single restart = both UI and API update
```

**Cost & Trade-offs**:
- Monthly cost: $100 (EC2 only, no additional charges)
- UI latency for US: 20ms (good)
- UI latency for EU: 150ms (acceptable for current user base)
- UI latency for APAC: 250ms (acceptable for current user base)
- Backend restart = UI downtime (but rare in production)

---

### 2026 Evolution: S3 + CloudFront (If Global Scale Needed)

**When this becomes necessary**:
- Concurrent users grow to 200+
- Geographic expansion: USA + EU + APAC users
- EU users report slow load times (150ms becomes unacceptable)
- Performance SLA required by customers

**Future Architecture**:
```
┌─────────────────────────────────────────┐
│ S3 Bucket (React static files only)     │
│ ├─ index.html                           │
│ ├─ js/main.abc.chunk.js                 │
│ ├─ css/main.xyz.chunk.css               │
│ └─ assets/                              │
└─────────────┬───────────────────────────┘
              │ (origin)
              ↓
┌─────────────────────────────────────────┐
│ CloudFront Distribution (200+ edges)    │
│ NYC → US users (15ms)                   │
│ London → EU users (20ms)                │
│ Tokyo → APAC users (18ms)               │
└─────────────┬───────────────────────────┘
              ↓
        User Browser (nearest edge)
        • US: 15ms (vs 20ms today)
        • EU: 20ms (vs 150ms today)
        • APAC: 18ms (vs 250ms today)

React app makes REST calls to EC2 backend:
              ↓
┌─────────────────────────────────────────┐
│ EC2 (Spring Boot Backend only)          │
│ • REMOVE: Serving static files          │
│ • KEEP: REST API (business logic)       │
│ • KEEP: Session management              │
│ • KEEP: JBPM orchestration              │
└─────────────────────────────────────────┘
```

**Cost Comparison**:
```
CURRENT (React on EC2):
  • EC2 instance: $100/month
  • Total: $100/month

IF UPGRADED TO S3 + CloudFront:
  • EC2 instance: $100/month (backend only, lighter load)
  • S3 storage (1 GB React files): ~$0.02/month
  • CloudFront (30 GB/month traffic): ~$2.50/month
  • Route53 (DNS): ~$1/month
  • Total: ~$103.50/month

Decision Matrix:
  • Current approach: SAVES $3.50/month vs upgrade
  • Benefit of upgrade: 10x better latency globally
  • When to upgrade: Only if global latency becomes a business requirement
```

**Implementation (when ready)**:
```bash
# Deploy to S3
npm run build
aws s3 sync dist/ s3://app-ui-bucket/ --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E1234EXAMPLE \
  --paths "/*"

# CORS configuration needed
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.company.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .maxAge(3600);
    }
}
```

**Cache Strategy (2026)**:
```
• index.html: TTL 0 (always check for updates)
• js/css chunks: TTL 1 year (versioned filenames: main.a1b2c3d4.js)
• assets/: TTL 30 days
• CloudFront monitoring:
  - Cache hit ratio (target: > 85%)
  - Latency by region
  - Error rates (alert if > 1%)
```

---

## �📊 Technology Stack Summary Table

| Component | Technology | Why | Alternative Rejected |
|-----------|-----------|-----|---------------------|
| **Scheduling** | EventBridge | Serverless, managed, retry support | Cron on EC2 |
| **Messages In** | Kafka (On-Prem) | External source (clients push to their Kafka), no AWS replacement | SQS (can't consume proprietary Kafka) |
| **Queue Decoupling** | SQS | Buffering, retry, DLQ | Direct Lambda calls |
| **Serverless Compute** | Lambda (Ingestion, Workflow, Email) | Cost ($60/mo), scaling, no ops | EC2, K8s |
| **Stateful API** | Spring Boot | User-facing, stateful, performance | Lambda (cold starts) |
| **Raw Messages** | Oracle | Compliance + existing investment (could migrate to DynamoDB in future) | DynamoDB (risk of dual-write complexity) |
| **Workflow Engine** | JBPM | Legacy but functional | Rebuild from scratch |
| **Fast Metadata** | DynamoDB | Sub-100ms queries, scale | RDS (slower) |
| **Audit Logs** | RDS | SQL compliance, structured | DynamoDB (wrong fit) |
| **Report Storage** | S3 | Durable, cheap, simple | Local FS (ephemeral) |
| **Task Execution** | EC2 instance | Stateful app + JBPM hosting, session management | Lambda (no state) |
| **Frontend UI** | React | Modern framework, component-based, independent deployment | AngularJS + JSP (legacy, tightly coupled) |
| **React UI** | EC2 (served by Spring Boot) | Current: Cost-efficient ($0 extra), simple deployment | S3 + CloudFront (future, if global scale needed) |

---

## 🎓 Interview Questions About Technology Stack

### Q1: Why AWS Lambda and not EC2?

**Answer**:
> "Lambda is the right choice for our event-driven, bursty workload because:
>
> 1. **Cost**: Lambda costs ~$60/month for our ingestion, workflow, and email services combined. EC2 would cost $300-400/month for 24/7 always-on capacity. We save ~$4,000/year company-wide.
>
> 2. **Automatic Scaling**: Lambda scales instantly to handle 2,000 messages/15 minutes without manual intervention. EC2 requires load balancers and auto-scaling policies—more operational overhead.
>
> 3. **Maintenance-Free**: No patching, no OS updates, no security configurations. AWS manages the infrastructure. EC2 requires ongoing DevOps management.
>
> 4. **Perfect Fit for Our Use Case**: Our functions are stateless (no persistent connections), short-running (< 5 minutes), and triggered by events (Kafka messages, schedules). This is exactly what Lambda is designed for.
>
> The only Lambda drawback is cold starts (~100-500ms), but that's acceptable because Ingestion runs once daily and Email Once daily."

### Q2: Why use both SQS and Kafka?

**Answer**:
> "They serve different roles:
>
> **Kafka** (On-Premises):
> - Source of truth for upstream systems at client locations
> - Business requirement: Stay integrated with existing Kafka infrastructure
> - High throughput for external message broker
>
> **SQS** (AWS):
> - Internal microservice communication within our AWS infrastructure
> - Decouples Ingestion Lambda from Workflow Lambda (loose coupling)
> - Buffers burst load (2,000 messages → SQS queue → gradual processing)
> - Built-in retry and dead-letter queue for reliability
>
> Using both is the pattern: **External events → Kafka → Ingestion Lambda → SQS → Workflow Lambda**
>
> This separates concerns and uses each tool for its intended purpose."

### Q3: Why DynamoDB alongside JBPM DB instead of just using JBPM?

**Answer**:
> "JBPM DB has the workflow state and complex schema, but it's slow for reporting queries:
>
> **JBPM DB Query**: 'Get all completed tasks today'
> - Complex joins across multiple tables
> - JBPM enforces locking for consistency
> - Results: 5-10 seconds for 2,000 records
>
> **DynamoDB Query** (same logic):
> - Simple flat schema: message_id → status, payload
> - No locking (eventual consistency acceptable)
> - Results: < 100ms
>
> **Design Pattern**: Database per Service
> - JBPM DB: Workflow orchestration (strong consistency needed)
> - DynamoDB: Metadata cache (eventual consistency acceptable)
> - Accept data sync lag (milliseconds to seconds)
> - Middleware: Workflow Lambda writes to both after instance creation
>
> This is a trade-off between consistency and performance. For reporting, performance wins."

### Q4: How is React UI deployed and why?

**Answer**:
> "Currently, React UI is served directly from the EC2 instance alongside Spring Boot backend.
>
> **Technical Implementation**:
> - React built to static files (npm run build → dist/)
> - Spring Boot serves index.html and bundles from /static routes
> - REST API routes (/api/*) go to backend logic
> - Single deployment: EC2 starts both services
>
> **Why EC2 Today**:
> - Cost: $0 additional (files served by existing EC2)
> - Simplicity: One deployment unit, one restart handles both
> - Acceptable for current 50-100 concurrent US-based users
> - 20ms latency for US users is acceptable
>
> **Future Consideration** (if global scale required):
> - 200+ concurrent users across USA + EU + APAC
> - EU users see 150ms latency from US EC2 (problematic)
> - Solution: S3 + CloudFront
>   - S3: Store React static files
>   - CloudFront: 200+ edge locations globally
>   - Latency: 15-50ms everywhere (vs 150-250ms from single EC2)
>   - Cost: +$3.50/month (negligible for 10x performance improvement)
>
> **Interview Angle**: 'We made the right tradeoff for our current scale. Architecture can evolve as requirements change. That's the beauty of microservices—we can deploy UI independently when needed.'"

### Q5: Why migrate from AngularJS + JSP to React?

**Answer**:
> "The original monolithic architecture used AngularJS (1.x) with JSP templates for the UI. While functional for small scale, it had several limitations:
>
> **AngularJS + JSP Problems**:
> - **Deprecation**: AngularJS has been unsupported since 2021 (no security fixes, no maintenance)
> - **Tight Coupling**: JSP templates are server-rendered, requiring backend restart to change UI
> - **No Independent Deployment**: Can't update UI without redeploying entire backend
> - **Performance**: Full page reloads on navigation (not a SPA - Single Page App)
> - **Caching**: Hard to cache globally (JSP is dynamic, CDN-unfriendly)
> - **Developer Experience**: AngularJS 1.x lacks modern tooling and practices
>
> **React Advantages**:
> - **Modern & Maintained**: React actively maintained, up-to-date security patches
> - **Component-Based**: Reusable UI components, easier to maintain and test
> - **Decoupled from Backend**: React is just static HTML/JS/CSS (can be served from S3+CDN if needed)
> - **Independent Deployment**: Update UI without touching backend
> - **SPA Experience**: Smooth navigation, no full page reloads
> - **Developer Productivity**: Rich ecosystem, better tooling (Create React App, TypeScript support)
> - **Global Scaling Ready**: React SPA is CDN-friendly (static files can be cached globally)
>
> **Migration Strategy**:
> - Keep React on EC2 today (cost-efficient)
> - Already decoupled, easy to move to S3+CloudFront later if needed
> - REST API bridge between React and backend unchanged"

---

**← Previous**: [02_Microservices_Architecture.md](./GUIDE_02_Microservices_Architecture.md) | **→ Next**: [04_Design_Patterns.md](./GUIDE_04_Design_Patterns.md)

---

**Key Takeaway**: Technology decisions are driven by requirements, not preference. For each component, we asked: "What problem are we solving? What's the simplest, most cost-effective technology that solves it?" This tradeoff analysis is what interviewers want to see.
