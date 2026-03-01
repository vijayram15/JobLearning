# 🚀 Section 2: Microservices Architecture

## Overview

After identifying monolithic scalability issues, the team redesigned **Excess Management System** using a **microservices** approach. This section explains:
- The new architecture
- Why each service exists
- How they communicate
- Benefits over monolithic design
- Trade-offs introduced

---

## 🏛️ Microservices Architecture Design

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MICROSERVICES ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│                   ┌──────── AWS Cloud Ecosystem ────────┐                   │
│                   │                                      │                   │
│  ┌─────────────────────────────────────────────────────────────━┐           │
│  │  Kafka (On-Prem) → EventBridge (AWS) → SQS                  │           │
│  │  (Message ingestion orchestration)                           │           │
│  └─────────────────────────────────────────────────────────────┘           │
│                            ↓                                                │
│  ┌──────────────────────────────────────────────────────────┐             │
│  │ SERVICE 1: INGESTION LAMBDA                              │             │
│  │ ┌─────────────────────────────────────────────────────┐  │             │
│  │ │ • Trigger: EventBridge (scheduled)                  │  │             │
│  │ │ • Action: Connect to Kafka, consume messages        │  │             │
│  │ │ • Persist: Store in Oracle ("NEW" status)          │  │             │
│  │ │ • Publish: Send to SQS for workflow service         │  │             │
│  │ │ • Benefit: Serverless, pay-per-invocation, no idle  │  │             │
│  │ └─────────────────────────────────────────────────────┘  │             │
│  └──────────────────────────────────────────────────────────┘             │
│                            ↓                                                │
│  ┌──────────────────────────────────────────────────────────┐             │
│  │ SERVICE 2: WORKFLOW LAMBDA (JBPM Orchestrator)           │             │
│  │ ┌─────────────────────────────────────────────────────┐  │             │
│  │ │ • Trigger: SQS (message queue)                      │  │             │
│  │ │ • Reserved Concurrency: 20 executions (controlled)  │  │             │
│  │ │ • Action: Create JBPM process instances            │  │             │
│  │ │ • Dual Persist:                                     │  │             │
│  │ │   - JBPM DB (workflow state)                        │  │             │
│  │ │   - DynamoDB (metadata, correlation IDs)           │  │             │
│  │ │ • Benefit: Horizontal scaling, queue-based backlog  │  │             │
│  │ │           management                                │  │             │
│  │ └─────────────────────────────────────────────────────┘  │             │
│  └──────────────────────────────────────────────────────────┘             │
│                            ↓                                                │
│  ┌──────────────────────────────────────────────────────────┐             │
│  │ SERVICE 3: TASK EXECUTION (Spring Boot API)              │             │
│  │ ┌─────────────────────────────────────────────────────┐  │             │
│  │ │ • Access: Web UI (users login via OAuth2/JWT)       │  │             │
│  │ │ • Stateful: Maintains user session                  │  │             │
│  │ │ • Action: Fetch tasks from JBPM, present to users   │  │             │
│  │ │ • Update: Write task completion to:                 │  │             │
│  │ │   - JBPM DB (workflow state)                        │  │             │
│  │ │   - DynamoDB (payload modifications)                │  │             │
│  │ │ • Benefit: User-facing, responsive, independently   │  │             │
│  │ │           scalable                                  │  │             │
│  │ └─────────────────────────────────────────────────────┘  │             │
│  └──────────────────────────────────────────────────────────┘             │
│                            ↓                                                │
│  ┌──────────────────────────────────────────────────────────┐             │
│  │ SERVICE 4: EMAIL LAMBDA (Scheduled Reporter)             │             │
│  │ ┌─────────────────────────────────────────────────────┐  │             │
│  │ │ • Trigger: Scheduled rule (5 PM daily)              │  │             │
│  │ │ • Action: Query DynamoDB for COMPLETED instances    │  │             │
│  │ │ • Generate: CSV/Excel/PDF reports                   │  │             │
│  │ │ • Persist: Store in S3 (durable storage)            │  │             │
│  │ │ • Send: Email to stakeholders via SES               │  │             │
│  │ │ • Audit: Log in Audit RDS                           │  │             │
│  │ │ • Benefit: Independent scaling, isolated failure    │  │             │
│  │ └─────────────────────────────────────────────────────┘  │             │
│  └──────────────────────────────────────────────────────────┘             │
│                                                                               │
│  ═══════════════════ DATA STORES (Polyglot Persistence) ════════════════  │
│                                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Oracle     │  │  JBPM DB     │  │  DynamoDB    │  │  Audit RDS   │  │
│  │              │  │              │  │              │  │              │  │
│  │ • Raw msgs   │  │ • Workflow   │  │ • Metadata   │  │ • Email logs │  │
│  │   (status)   │  │   instances  │  │ • Payloads   │  │ • Audit      │  │
│  │ • Audit      │  │ • Task       │  │ • Correls    │  │   trails     │  │
│  │              │  │   state      │  │              │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘


                          KEY DIFFERENCE: EVENT-DRIVEN
                    ↓↓  No direct service-to-service calls  ↓↓
                    Services communicate via messages/queues
```

---

## 🔄 Data Flow (Microservices)

```
STEP 1: INGESTION LAMBDA (EventBridge Trigger)
  ┌─────────────────────────┐
  │ EventBridge Rule        │
  │ Time: Business hours    │
  │ (Scheduled event)       │
  └──────────┬──────────────┘
             ↓
  ┌─────────────────────────────────────────────────┐
  │ Ingestion Lambda                                 │
  │ • Connect to Kafka (on-prem)                     │
  │ • Consume excess messages                        │
  │ • Parse and validate                             │
  └──────────┬──────────────┬──────────────┬─────────┘
             ↓              ↓              ↓
   ┌──────────────┐  ┌──────────────┐  ┌─────────┐
   │ Oracle DB    │  │ SQS Queue    │  │ CloudWatch
   │ INSERT NEW   │  │ PUBLISH      │  │ METRICS
   │ message      │  │ message ID   │  │
   │ status=NEW   │  │              │  │
   └──────────────┘  └────┬─────────┘  └─────────┘
                          ↓
STEP 2: WORKFLOW LAMBDA (SQS Trigger)
  ┌─────────────────────────────────────────────────┐
  │ SQS Queue                                        │
  │ • Messages waiting to be processed               │
  │ • Reserved Concurrency: 20 instances max        │
  │ • Auto-retry with visibility timeout             │
  │ • Dead-letter queue for failures                 │
  └──────────┬──────────────────────────────────────┘
             ↓
  ┌─────────────────────────────────────────────────┐
  │ Workflow Lambda                                  │
  │ For each SQS message:                            │
  │ • Read message details from Oracle               │
  │ • Validate idempotency (already processed?)      │
  │ • Create JBPM process instance                   │
  │ • Store correlation data                         │
  └──────────┬──────────────┬──────────────┬─────────┘
             ↓              ↓              ↓
   ┌──────────────┐  ┌──────────────┐  ┌─────────┐
   │ JBPM DB      │  │ DynamoDB     │  │ CloudWatch
   │ INSERT       │  │ PUT metadata │  │ METRICS
   │ workflow     │  │ correlation  │  │
   │ instance     │  │ ID mapping   │  │
   └──────────────┘  └──────────────┘  └─────────┘

STEP 3: TASK EXECUTION (User Interaction)
  ┌─────────────────────────────────────────────────┐
  │ End-User                                         │
  │ • Logs into Web UI (Spring Boot Backend)        │
  │ • Authenticate via OAuth2/JWT                   │
  └──────────┬──────────────────────────────────────┘
             ↓
  ┌─────────────────────────────────────────────────┐
  │ Task Execution Service (Spring Boot)            │
  │ • Query JBPM DB for user's assigned tasks       │
  │ • Query DynamoDB for payload details            │
  │ • Render UI with task information               │
  └──────────┬──────────────────────────────────────┘
             ↓
  User processes task (approve/reject)
             ↓
  ┌─────────────────────────────────────────────────┐
  │ Task Completion Update                          │
  │ • Call JBPM REST API: Update task status        │
  │ • Update DynamoDB: Payload modifications        │
  │ • Log to CloudWatch                             │
  └──────────┬──────────────┬──────────────┬─────────┘
             ↓              ↓              ↓
   ┌──────────────┐  ┌──────────────┐  ┌─────────┐
   │ JBPM DB      │  │ DynamoDB     │  │ CloudWatch
   │ UPDATE       │  │ PUT updated  │  │ METRICS
   │ task status  │  │ state/payload│  │
   └──────────────┘  └──────────────┘  └─────────┘

STEP 4: EMAIL SERVICE (Scheduled Daily)
  ┌─────────────────────────────────────────────────┐
  │ EventBridge Rule                                 │
  │ Time: 5:00 PM daily                            │
  └──────────┬──────────────────────────────────────┘
             ↓
  ┌─────────────────────────────────────────────────┐
  │ Email Lambda                                     │
  │ • Query DynamoDB: All COMPLETED instances      │
  │ • Generate report (CSV/Excel/PDF)              │
  │ • Store temp file in S3                        │
  │ • Send via AWS SES                             │
  │ • Log audit entry                              │
  └──────────┬──────────────┬──────────────┬─────────┘
             ↓              ↓              ↓
   ┌──────────────┐  ┌──────────────┐  ┌──────────┐
   │ S3 Bucket    │  │ Audit RDS    │  │ SES Email
   │ STORE        │  │ INSERT       │  │ SEND
   │ report       │  │ email log    │  │
   └──────────────┘  └──────────────┘  └──────────┘
```

---

## 📊 Service Responsibilities

### Service 1: Ingestion Lambda

| Aspect | Details |
|--------|---------|
| **Trigger** | AWS EventBridge (scheduled job, business hours) |
| **Input** | Kafka messages from on-premises |
| **Processing** | Parse, validate, add unique message ID |
| **Output** | Oracle (raw message, status=NEW) + SQS (message sent) |
| **Concurrency** | Auto-scaling (unlimited) |
| **Cost Model** | Pay per invocation (~$0.0000002 per call) |
| **Failure Handling** | EventBridge retries; logs to CloudWatch |

### Service 2: Workflow Lambda (JBPM Orchestrator)

| Aspect | Details |
|--------|---------|
| **Trigger** | Amazon SQS queue (messages from Ingestion) |
| **Reserved Concurrency** | 20 concurrent executions (prevents overwhelming JBPM/DB) |
| **Processing** | Create JBPM workflow instance for each message |
| **Dual Persistence** | JBPM DB (workflow state) + DynamoDB (metadata) |
| **Idempotency Check** | Prevent duplicate workflow creation on retries |
| **Scale Capacity** | 1,000–2,000 instances in 10–15 min ✓ |
| **Failure Handling** | SQS retry + dead-letter queue |

### Service 3: Task Execution Service (Spring Boot on EC2)

| Aspect | Details |
|--------|----------|
| **Deployment** | Spring Boot application on EC2 instance |
| **Hosting** | EC2 also hosts JBPM workflow engine + its database |
| **Authentication** | OAuth2/JWT via API Gateway |
| **Input** | End-user requests via web UI/browser |
| **Processing** | Fetch tasks from JBPM; present to users; process actions |
| **Persistence** | Updates to JBPM DB (on same EC2) + DynamoDB |
| **Session Management** | Stateful sessions (user login, task context) |
| **Scaling** | Manual vertical scaling; or auto-scaling group with ALB |
| **Why EC2** | Requires persistent state, session management, always-on for business hours |
| **JBPM DB Connection** | Separate database instance (not shared with Oracle) - Database Per Service pattern |

### Service 4: Email Lambda (Reporter)

| Aspect | Details |
|--------|---------|
| **Trigger** | EventBridge scheduled rule (5 PM daily) |
| **Input** | Query DynamoDB for completed instances |
| **Processing** | Generate CSV/Excel/PDF reports |
| **Persistence** | Reports stored in S3; audit log in RDS |
| **Output** | Email via AWS SES to stakeholders |
| **Failure Handling** | Lambda retries; CloudWatch alerts |
| **Scalability** | Independent from other services |

---

## ⚖️ Architectural Trade-Offs: Why We Still Use Kafka & Oracle

### Context: "Isn't this architecture hybrid? Shouldn't it be all-AWS?"

This is a great question that shows you're thinking critically. Here's the actual reasoning:

### Decision 1: Keep On-Premises Kafka (No Choice)

**The Situation**:
```
Client Infrastructure (On-Premises):
  • Clients manage their own Kafka brokers
  • Clients send excess trade messages to their Kafka
  • This is their system of record (not ours to decide)

Our Role:
  • We consume from their Kafka via Lambda
  • Connect via VPN/AWS PrivateLink (secure tunnel)
  • We don't control client infrastructure
```

**Alternatives Considered**:
- Replace with AWS SQS? ❌ Can't—clients are already using Kafka
- Replace with AWS Kinesis? ❌ Still doesn't help—clients send to Kafka
- Mirror Kafka to AWS? ❌ Extra complexity, no benefit

**Verdict**: **KEEP KAFKA** (Architectural Constraint)
- We don't control it—it's client infrastructure
- We CAN'T "go all-AWS" when clients have on-premises systems
- This is real-world hybrid infrastructure (common in enterprises)
- Saying "use only AWS services" ignores client reality

### Decision 2: Keep Oracle (Strategic Business Decision)

**The Situation**:
```
Current Usage:
  • Oracle DB stores raw excess messages (EXCESS_MESSAGE table)
  • ~2,000–5,000 records/day
  • Query pattern: Simple (~100ms queries)
  • Already licensed and maintained by IT team

Cost Today: ~$50/month
Cost if Migrated to DynamoDB: ~$5/month (would save $45/month)
```

**Why Oracle is Still There**:
| Reason | Impact |
|--------|--------|
| **Already Operational** | DB team manages it, backups configured, security approved |
| **Compliance** | Auditors pre-approved Oracle for financial data retention |
| **Team Expertise** | DBA team knows Oracle; DynamoDB requires retraining |
| **Zero Downtime Requirement** | Migration requires dual-write complexity or scheduled downtime |
| **Migration Risk** | 2-4 week project to safely migrate + retrain team |
| **Cost Benefit** | Saving $45/month doesn't justify migration effort/risk |

**Migration Scenario** (Why NOT to do it now):
```
Option: Dual-Write During Migration
  1. Write all new messages to BOTH Oracle + DynamoDB
  2. Verify DynamoDB data matches Oracle (hours of testing)
  3. Switch read queries from Oracle → DynamoDB
  4. Monitor for few weeks
  5. Decommission Oracle

Problems:
  • Increases Lambda processing time (dual writes slower)
  • Dual writes fail? Data inconsistency risk
  • DynamoDB schema different from Oracle—data shape mismatch
  • Bugs introduced during migration = customer impact
  • Not worth the risk for $45/month savings
```

**Practical Decision** (What We Actually Do):
```
TODAY: Keep Oracle
  • Cost: $50/month
  • Risk: None (proven system)
  • Effort: Zero

FUTURE-WHEN-WE-SCALE: Migrate to DynamoDB
  • Trigger: If scaling to 50,000+ records/day
  • ROI: $500+/month savings (worth migration effort)
  • Plan: Schedule maintenance window, controlled migration
  • New team: Hired DynamoDB expert by then
```

**Verdict**: **KEEP ORACLE** (Pragmatic Business Decision)
- Not for technical reasons (DynamoDB is technically fine)
- For practical reasons (effort/risk/cost analysis)
- Future-ready (can migrate when scale justifies it)

### The Real Architecture Principle

**NOT**: "Use only AWS" or "Use only managed services"
**BUT**: "Use the right tool, make pragmatic trade-offs, understand constraints"

| Constraint Type | Example | Action |
|-----------------|---------|--------|
| **External** | Clients use on-prem Kafka | Accept it (can't change) |
| **Business** | Oracle license already paid | Keep it (cost-benefit) |
| **Operational** | Team knows Oracle | Keep it (learning curve trade-off) |
| **Technical** | Need high consistency on raw messages | Choose Oracle (compliance) |

**Interview Takeaway**: Real architects balance ideological purity with practical constraints. Saying "we should use all-AWS" is naive if your clients have on-premises infrastructure. Saying "we should rip out Oracle for DynamoDB" is naive if the cost/risk isn't justified yet.

---

## ⓘ Clarification Needed: JBPM Database Architecture

**You asked an important question**: Is JBPM DB actually a separate Oracle instance, or is it just schemas within the same Oracle?

**Answer**: It depends on how your team set it up. Both are valid:

| Setup | Implementation | Trade-off |
|-------|----------------|-----------|
| **Schema Separation** | Same Oracle instance, different JBPM schema | Cost-efficient, risk of lock contention at scale |
| **Instance Separation** | Two Oracle instances or JBPM on RDS | Eliminates lock contention, higher cost |

**What to Find Out** (Ask your DBA team or AWS architect):
```
Run one of these commands:

If Oracle:
  SELECT OWNER, TABLE_NAME FROM DBA_TABLES 
  WHERE OWNER IN ('jbpm', 'JBPM_OWNER');
  
Result:
  • If returns rows: JBPM is in your Oracle (same instance)
  • If empty: JBPM is on different database

Check connection strings:
  • Business app: Connects to oracle_host:1521/EXCESS_DB
  • Workflow service: Connects to ???
    - Same host = Scenario A (same instance)
    - Different host = Scenario B (different instance)
```

**For Interview Preparation**:
- If you don't know, say: "JBPM tables are logically separated from business data (by schema or instance) to prevent lock contention"
- This covers both scenarios and shows you understand the principle

---

## 🎯 Benefits of Microservices Design

### 1. **Horizontal Scalability** ✅

**Before (Monolithic)**:
```
Need 3x capacity? Buy 3x bigger server
Cost: $400/month → $1,200+/month
Limitation: Physical server limit
```

**After (Microservices)**:
```
Need 3x capacity? Run 3x more Lambda instances
Cost: Pay only for what you use
Example: 2,000 instances/day = ~$2–5/day in Lambda costs
```

### 2. **Independent Service Scaling** ✅

```
High user traffic for task processing?
  → Scale only Task Execution Service
  → Ingestion/Workflow/Email unaffected

High reporting load?
  → Scale Email Lambda
  → Doesn't impact transaction processing

Perfect granularity!
```

### 3. **Cost Efficiency** ✅

```
Monolithic EC2 Cost:
  m5.2xlarge: $400/month (24/7)
  × 12 months = $4,800/year

Microservices Lambda Cost:
  Ingestion: ~$20/month (runs 8 hours/day on business days)
  Workflow: ~$30/month (triggered by SQS)
  Email: <$10/month (runs once/day)
  
  Total: ~$60/month = $720/year
  SAVINGS: ~$4,000/year (for 1 client)
```

### 4. **Independent Deployment** ✅

```
Bug in Email Service?
  → Deploy only Email Lambda
  → No risk to other services
  → Fast, isolated fix

New feature in Task Execution?
  → Update only Spring Boot service
  → Other microservices unaffected
```

### 5. **Technology Flexibility** ✅

```
Can now use:
  • Lambda (for lightweight, scheduled tasks)
  • Spring Boot (for stateful, user-facing features)
  • DynamoDB (where fast lookups matter)
  • RDS (for compliance/audit tables)
  • Oracle (where already integrated)

Monolith forced single tech = constraint
Microservices allow best-fit technology
```

### 6. **Better Failure Isolation** ✅

```
Scenario: JBPM connection fails

Monolithic:
  → Ingestion freezes ❌
  → Task processing freezes ❌
  → Reporting freezes ❌
  → ENTIRE SYSTEM DOWN

Microservices:
  → Workflow Lambda: Failed, queued messages sit in SQS ✓
  → Ingestion: Still running, buffering messages ✓
  → Task Execution: Still running, users can process old tasks ✓
  → Email: Still running, can generate reports from cached data ✓
  → System partially degraded, not completely down ✓
```

### 7. **Team Independence** ✅

```
Team 1: Owns Ingestion Lambda
Team 2: Owns Workflow Lambda  
Team 3: Owns Task Execution Service
Team 4: Owns Email Lambda

Benefits:
  ✓ No merge conflicts on shared code
  ✓ Each team deploys independently
  ✓ Different tech stacks allowed
  ✓ Faster feature development
  ✓ Clear ownership & accountability
```

### 8. **Observable & Debuggable** ✅

```
Each Lambda function logs independently:
  • Ingestion metrics: Messages/sec, parse errors, DB latency
  • Workflow metrics: Instance creation rate, JBPM latency
  • Task metrics: User engagement, update latency
  • Email metrics: Report generation time, SES failures

CloudWatch dashboards show:
  ✓ Where bottlenecks are
  ✓ Which service is slow
  ✓ Individual service health
  ✓ Not a monolithic black box
```

---

## ⚠️ Trade-Offs & Challenges

### Challenge 1: Increased Operational Complexity

**Cost**: Managing 4 services, multiple databases, queue infrastructure
- **Monitoring**: Need dashboards for each service + overall health
- **Debugging**: Harder to trace requests across services
- **Deployment**: More orchestration needed (CI/CD pipelines)

**Mitigation**: Use AWS-managed services (Lambda, SQS, EventBridge reduce overhead)

### Challenge 2: Data Consistency

**Issue**: JBPM DB and DynamoDB may go out of sync
- **Dual Writes**: Writing to both DBs isn't atomic
- **Network Failures**: Some writes might fail while others succeed
- **Eventual Consistency**: Data is eventually consistent, not immediately

**Mitigation**: Implement idempotency checks, use Outbox Pattern, accept eventual consistency

### Challenge 3: Distributed Transactions

**Issue**: Transaction spanning multiple services is complex
```
Example: Workflow creation fails mid-process
  → JBPM DB was updated but DynamoDB wasn't
  → Partially completed transaction
```

**Mitigation**: Use Saga pattern for distributed transactions (covered in next sections)

### Challenge 4: Network Latency

**Before**: Direct function calls (no network latency)
**After**: Messages go through SQS (adds ~100ms latency)

**Impact**: End-to-end processing slightly slower, but parallelization compensates

### Challenge 5: Testing Complexity

**Monolithic**: Can test entire flow in one test
**Microservices**: Must mock external services, test edges cases

**Mitigation**: Contract testing, integration test environments

---

## 🔑 Key Architectural Decisions

### Decision 1: Reserved Concurrency (20)

```
Why limit Workflow Lambda to 20 concurrent executions?

Without limit:
  SQS has 10,000 messages
    → Lambda scales to 1,000 concurrent instances
      → Each hits JBPM DB
        → DB connection pool (50 connections) exhausted
          → All queries timeout
            → Cascading failures

With limit (20):
  SQS has 10,000 messages
    → Lambda scales to max 20 concurrent
      → Each hits JBPM DB
        → DB connection pool: 20 connections in use
          → Remaining 30 for other services
            → Stable, controlled processing
              → Messages queue in SQS safely until ready
```

### Decision 2: Dual Persistence (JBPM DB + DynamoDB)

```
Why not just JBPM DB?

JBPM DB Issues:
  • Complex schema (workflow internals)
  • Slow for lookups by message ID
  • Joins expensive across tables
  • Locking contention

DynamoDB Benefits:
  • Simple, flat schema (message ID → payload)
  • Fast lookups (< 10ms)
  • Multi-item transactions supported
  • No locking (eventual consistency OK for reporting)

Compromise:
  • Keep workflow state in JBPM DB (already required)
  • Keep message payloads & status in DynamoDB (fast for reporting)
  • Accept eventual consistency between them
```

### Decision 3: SQS Over Direct Lambda Calls

```
Why use SQS instead of Lambda invoking Lambda directly?

Direct Lambda Calls:
  ❌ Workflow Lambda must know Ingestion Lambda exists
  ❌ Tight coupling (changes break things)
  ❌ No buffering (sudden spikes = failures)
  ❌ No retries (if Lambda fails, message lost)

SQS Approach:
  ✓ Ingestion publishes; Workflow subscribes (loose coupling)
  ✓ Message queue buffers bursts (1,000/minute → spread over time)
  ✓ Built-in retries & dead-letter queue (no messages lost)
  ✓ Visibility timeout (prevents duplicate processing)
  ✓ Easy to add new consumers later
```

---

## 📋 Microservices Summary Table

| Aspect | Monolithic | Microservices |
|--------|-----------|---------------|
| **Scalability** | Vertical (limited) | Horizontal (unlimited) ✅ |
| **Load Capacity** | 1,000–1,500/day | 2,000/15 minutes ✅ |
| **Cost** | $400+/month (always-on) | $60/month (pay-per-use) ✅ |
| **Deployment** | All-or-nothing | Independent per service ✅ |
| **Failure Impact** | System-wide outage | Isolated degradation ✅ |
| **Team Independence** | Limited (shared code) | Full (separate services) ✅ |
| **Operational Complexity** | Low | Higher (but manageable) ⚠️ |
| **Data Consistency** | ACID (strong) | Eventual (weak) ⚠️ |
| **Debugging** | Simple (one process) | Complex (distributed) ⚠️ |

---

## 🎓 Interview Questions About Microservices Architecture

### Q1: Walk me through your microservices architecture.

**Answer**:
> "Excess Management System is built on 4 core microservices:
>
> 1. **Ingestion Lambda**: Scheduled daily to consume Kafka messages from on-premises. It stores raw messages in Oracle with status 'NEW' and publishes to SQS for downstream processing.
>
> 2. **Workflow Lambda**: Triggered by SQS, creates JBPM workflow instances. I use reserved concurrency (20) to prevent overwhelming the database. It stores workflow state in JBPM DB and metadata in DynamoDB for fast lookups.
>
> 3. **Task Execution Service**: Spring Boot API for end-users to process tasks. It fetches tasks from JBPM, allows user updates, and persists changes to both JBPM DB and DynamoDB.
>
> 4. **Email Lambda**: Scheduled daily at 5 PM to generate reports, query DynamoDB for completed instances, and email stakeholders.
>
> Each service is independently deployable and scalable. They communicate asynchronously via SQS and EventBridge, reducing tight coupling."

### Q2: Why did you choose Lambda over EC2?

**Answer**:
> "Three key reasons:
>
> 1. **Cost**: Lambda is pay-per-invocation ($0.0000002/call). For a workload that runs 8 hours/day on weekdays, this is roughly $60/month. EC2 m5.2xlarge runs 24/7 at ~$400/month—almost 7x more expensive.
>
> 2. **Automatic Scaling**: Lambda scales horizontally without infrastructure management. EC2 requires load balancers and autoscaling groups—more operational overhead.
>
> 3. **Maintenance**: No patching, no OS updates, no security patches for Lambda. Fully managed by AWS. EC2 requires ongoing system administration."

### Q3: How do you handle the capacity spike of 2,000 instances in 15 minutes?

**Answer**:
> "This is where reserved concurrency and SQS buffering come together:
>
> 1. **Ingestion Lambda**: Runs at fixed times, consumes all Kafka messages, publishes 2,000 messages to SQS queue in minutes.
>
> 2. **SQS Acts as Buffer**: Even though 2,000 messages arrive quickly, they're safely queued.
>
> 3. **Workflow Lambda Reserved Concurrency (20)**: Max 20 concurrent executions. Each creates a JBPM instance. So 2,000 instances take ~100 batches.
>
> 4. **Time to Process**: 2,000 instances / (20 concurrent × 60 instances/minute per Lambda) = ~1.67 hours. Acceptable, with SQS buffering load.
>
> This prevents the JBPM DB from being overwhelmed—a critical lesson from monolith failures."

---

**← Previous**: [01_Monolithic_Architecture.md](./GUIDE_01_Monolithic_Architecture.md) | **→ Next**: [03_Technology_Stack.md](./GUIDE_03_Technology_Stack.md)

---

**Key Takeaway**: Microservices solve scalability, cost, and operational issues but introduce complexity. The key is understanding why each technology was chosen and accepting the trade-offs.
