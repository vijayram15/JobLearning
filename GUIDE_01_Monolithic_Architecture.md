# 📌 Section 1: Monolithic Architecture

## Overview

The **Excess Management System** started as a **monolithic application** – a single codebase handling all business logic from data ingestion to final reporting. This section explains:
- Original system design
- How it worked
- Why it succeeded initially
- When and why it started failing

---

## 🏗️ Original Monolithic Design

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                   SINGLE MONOLITH APPLICATION                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────  Layer 1: Data Ingestion ──────────────┐       │
│  │ Scheduler → Kafka Consumer → Raw Message Parser      │       │
│  └───────────────────────────────────────────────────────┘       │
│                            ↓                                     │
│  ┌──────────────  Layer 2: Data Persistence ───────────┐        │
│  │ Oracle DB: Store "NEW" status messages in table     │        │
│  │ → EXCESS_MESSAGE table                              │        │
│  └───────────────────────────────────────────────────────┘       │
│                            ↓                                     │
│  ┌──────────────  Layer 3: Workflow Orchestration ─────┐        │
│  │ JBPM Process Invocation                             │        │
│  │ → Create instances for each "NEW" message           │        │
│  │ → For ~1,000–2,000 records: 10–15 minutes          │        │
│  └───────────────────────────────────────────────────────┘       │
│                            ↓                                     │
│  ┌──────────────  Layer 4: User Task Processing ───────┐        │
│  │ Web Application (same codebase)                      │        │
│  │ → UI: AngularJS + JSP templates (server-rendered)    │        │
│  │ → End-users login and process JBPM instances        │        │
│  │ → Update instance status in same JBPM table/DB      │        │
│  └───────────────────────────────────────────────────────┘       │
│                            ↓                                     │
│  ┌──────────────  Layer 5: Reporting ────────────────┐          │
│  │ Scheduled Job (End of Day)                          │         │
│  │ → Query JBPM for "COMPLETED" instances             │         │
│  │ → Generate CSV/Excel/PDF reports                    │         │
│  │ → Email to stakeholders                             │         │
│  │ → Log to Audit table                                │         │
│  └───────────────────────────────────────────────────────┘       │
│                                                                   │
│                   ↓↓↓ SINGLE DATABASE ↓↓↓                        │
│                                                                   │
│  ┌──────────────  Oracle Database ──────────────┐               │
│  │ • EXCESS_MESSAGE (raw messages)              │               │
│  │ • Audit tables (reporting logs)              │               │
│  │ • All other business data                     │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
│  ┌──────────────  JBPM Database ────────────────┐               │
│  │ • Workflow instances                          │               │
│  │ • Task definitions                            │               │
│  │ • Process history & state                     │               │
│  │ • Session information                         │               │
│  └──────────────────────────────────────────────┘               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 Data Flow (Monolithic)

```
Step 1: INGESTION
  Kafka Message → Scheduler Job → Java Consumer Code
                                    ↓
                            Extract & Parse Message
                                    ↓
Step 2: STORAGE
  Save to Oracle with status=NEW
  ├─ Table: EXCESS_MESSAGE
  ├─ Columns: id, raw_data, status, created_date, etc.
  └─ Ready for processing

Step 3: WORKFLOW CREATION
  Oracle Query (WHERE status='NEW')
                    ↓
  For each NEW message:
  ├─ Invoke JBPM Process Definition
  ├─ Create JBPM Process Instance
  ├─ Assign tasks to users
  └─ Store instance state in JBPM DB

Step 4: TASK EXECUTION
  User logs into UI
                    ↓
  User sees their assigned tasks (from JBPM)
                    ↓
  User processes task:
  ├─ View message details (from Oracle)
  ├─ Make business decision
  ├─ Update task status in JBPM (COMPLETED/REJECTED)
  └─ Save back to Oracle if needed

Step 5: REPORTING
  Scheduled Job (e.g., 5 PM daily):
  ├─ Query Oracle: SELECT * WHERE status='COMPLETED' AND date=TODAY
  ├─ Generate Report (CSV/Excel/PDF)
  ├─ Send Email to Stakeholders
  └─ Log email sent event to AUDIT table
```

---

## ✅ Why This Design Worked (Initially)

### 1. **Simple & Easy to Understand**
- Single codebase, single database
- Linear flow: Ingest → Process → Report
- No complex service-to-service communication

### 2. **ACID Transactions**
- Oracle transactions ensured consistency
- All updates in a single DB = no distributed transaction problems
- Easy rollback if something fails

### 3. **Tight Integration**
- No latency between components
- Direct function calls (no network hops)
- Good performance for initial load (1,000–1,500 records/day)

### 4. **Easy Debugging**
- Everything in one process
- Can attach debugger to entire system
- Stack traces show full context

### 5. **Shared Data Model**
- All components understand the same object structure
- No serialization/deserialization overhead
- Natural code reuse

---

## ⚠️ The Monolithic Problems (Why It Failed)

### 1. **Scalability Bottleneck** (PRIMARY ISSUE)

**Problem**:
```
Original Capacity:  1,000–1,500 records/day ✓ Working
New Requirement:    1,000–2,000 records in 10–15 minutes ✗ Failing
```

**Why It Failed**:
- **Single Process**: All operations (ingest, workflow creation, task processing) ran in a single application instance
- **No Horizontal Scaling**: To handle more load, you need bigger machines (vertical scaling) – expensive and limited
- **Sequential Processing**: Even though there were threads, bottlenecks existed at:
  - Database connection pool (limited connections)
  - JBPM engine (single instance couldn't handle rapid instance creation)
  - Memory constraints (garbage collection pauses)

**Example**:
```
Trying to create 2,000 JBPM instances in 15 minutes:
= ~133 instances/minute
= ~2.2 instances/second

Single JBPM engine on one server → Thread pool saturation
→ Queue backlog → Delayed instance creation → User complaints
```

### 2. **Technology Lock-In**

**Problem**:
- Entire system tied to:
  - Oracle database (expensive licensing)
  - JBPM (EOL version, hard to upgrade)
  - Single tech stack (Java monolith)

**Impact**:
- Can't replace JBPM with newer orchestration tool without rewriting entire system
- Oracle is expensive for this scale – no room for different DB per use case
- All developers must know monolithic codebase

### 3. **Deployment Risk**

**Problem**: All-or-nothing deployments
```
Need to fix bug in Email Service?
  ↓
Deploy entire monolith
  ↓
Risk: Any bug in ingestion/workflow/tasks affects deployment
  ↓
Outage possible
```

**Impact**:
- Long release cycles
- Risk of breaking unrelated features
- No independent scaling for particular components

### 4. **Team Scalability**

**Problem**:
- Multiple teams (ingestion, workflow, reporting, task execution) all editing same codebase
- Merge conflicts
- Coordination overhead
- Knowledge silos (JBPM team vs Data team vs Reporting team)

### 5. **Resource Utilization Inefficiency**

**Problem**:
- Ingestion runs only during business hours (9 AM – 5 PM)
- Reporting runs once per day (5 PM)
- Task processing is sporadic (user-dependent)
- But entire application must run 24/7 on EC2 → Idle cost

**Cost Impact**:
```
EC2 Instance (m5.2xlarge): ~$400/month × always-on
  vs
AWS Lambda (1M invocations): ~$20/month + per-compute charges
```

### 6. **Failure Isolation Issues**

**Problem**: One failing component brings down entire system

```
Scenario 1: Kafka connection fails
  → Ingestion fails
  → But JBPM instance creation freezes too (same process)
  → Users can't process tasks
  → Reporting doesn't run
  ↓
ENTIRE SYSTEM DOWN

Scenario 2: JBPM DB connection timeout
  → Workflow creation fails
  → Users can't process tasks (they need JBPM instances)
  → Reporting fails (depends on JBPM status)
  ↓
ENTIRE SYSTEM DOWN
```

### 7. **Data Consistency vs Scalability Trade-Off**

**Problem**:
- ACID transactions work well in single machine
- But cause contention at scale
- All reads/writes compete for same DB locks
- Blocking queries = slower system

### 8. **Operational Visibility**

**Problem**:
- One application = one log file
- Hard to isolate problems:
  - Is ingestion slow? Or is it JBPM?
  - Is the DB running out of connections? Or is the app?
  - Hard to measure individual component performance

---

## 📋 Specific Monolithic Drawbacks

### Drawback 1: Database Contention

```
When processing 2,000 instances in 15 minutes:

JBPM Thread 1: UPDATE process_instance SET status='IN_PROGRESS'
JBPM Thread 2: UPDATE process_instance SET status='COMPLETED'  
JBPM Thread 3: INSERT into task_instance ...
JBPM Thread 4: SELECT from process_instance ...

Result:
  → Lock contention on JBPM tables
  → Slow queries block fast queries
  → Cascading delays
  → Timeout exceptions
```

### Drawback 2: Memory Pressure

```
Creating 2,000 JBPM instances in memory:
  → Each instance = ~1-2 MB object graph
  → 2,000 instances = ~2-4 GB heap
  → Garbage collection pause: 500ms – 2s
  → During GC pause: NO PROGRESS on instance creation
  → Users experience "stuck" system

With vertical scaling (bigger server):
  → More costly
  → Still eventual limit (memory is finite)
```

### Drawback 3: Network Latency (Kafka + DB)

```
Monolith connects to:
  → On-premises Kafka (network latency + firewall rules)
  → Oracle on corporate network
  → JBPM DB (possibly another server)

Single slow connection = entire system slows
  (no other services can work around it)
```

### Drawback 4: Rigid Workflow

```
If business wants to change:
  → "Let's add a pre-processing step before JBPM"
  → "Let's use ML to classify messages before routing to JBPM"
  → "Let's cache reporting data"

With monolith:
  → Edit same codebase
  → Retest everything
  → Deploy whole app
  → Days/weeks of work

With microservices:
  → Add new service
  → Deploy independently
  → Days work, minimal risk
```

### Drawback 5: Outdated UI Technology (AngularJS + JSP)

```
Original UI Stack:
  • AngularJS (released 2012, deprecated since 2021)
  • JSP templates (server-side rendering, tightly coupled)
  • jQuery for DOM manipulation
  
Problems:
  ✗ AngularJS → deprecated (no longer maintained)
  ✗ JSP → couples frontend + backend (can't deploy UI independently)
  ✗ Server-rendering → harder to cache globally (no CDN support)
  ✗ No component-based architecture (AngularJS 1.x → pain)
  ✗ Performance: Full page reload on navigation

Migration Path (Microservices):
  ✅ React (modern, component-based, maintained)
  ✅ Decoupled from backend (served from S3/CloudFront, not JSP)
  ✅ Independent UI deployment (no backend restart needed)
  ✅ Better SEO + global caching (CloudFront ready)
  ✅ SPA (Single Page App → faster navigation within UI)
```

---

## 🎯 Monolithic Architecture Summary Table

| Aspect | Status | Issue |
|--------|--------|-------|
| **Initial Load (1,000–1,500/day)** | ✅ Works | N/A |
| **Scaled Load (2,000/15 min)** | ❌ Fails | Concurrency limits, DB contention |
| **Independent Scaling** | ❌ Not possible | Vertical only |
| **Deployment Risk** | ⚠️ High | All-or-nothing changes |
| **Technology Flexibility** | ❌ Rigid | Locked into Oracle + JBPM |
| **Failure Isolation** | ❌ Poor | One failure = system down |
| **Cost Efficiency** | ❌ Poor | Pay for always-on infrastructure |
| **Team Independence** | ❌ Limited | Shared codebase conflicts |
| **Operational Visibility** | ⚠️ Mixed | Single log file, hard to isolate |

---

## 🔄 Transition Decision

**When the first client requested 1,000–2,000 records/day:**
- Monolith hit scalability wall
- Vertical scaling: Too expensive + Limited gains
- **Decision**: Break into microservices

**Business Driver**:
- Multiple clients across the board wanted to adopt same system
- Current monolith couldn't handle multi-client / high-volume scenarios
- Need: Scalable, cost-effective, independent-deployment platform

---

## 🎓 Interview Questions About Monolithic Architecture

### Q1: Why did the monolithic approach fail at scale?

**Answer**:
> "The original monolithic design worked well for 1,000–1,500 records/day with a single application instance running on EC2. However, when the client introduced 1,000–2,000 records in just 10–15 minutes, we hit several bottlenecks:
>
> 1. **Concurrency Limits**: The JBPM engine running in a single process couldn't handle rapid instance creation. Thread pools were saturated, causing queue buildup.
> 2. **Database Contention**: All operations competed for the same Oracle and JBPM DB connections. Lock contention on workflow tables caused significant delays.
> 3. **Vertical Scaling Limitation**: We tried scaling up the EC2 instance, but there's a finite limit to vertical scaling, and cost increases exponentially.
> 4. **Memory Pressure**: Creating 2,000 JBPM instances (~2–4 GB heap) triggered frequent garbage collection pauses, causing the system to appear stuck.
> 5. **Monolithic Overhead**: All components (ingestion, workflow, tasks, reporting) ran in one process, so any failure cascaded system-wide.
>
> The real issue: **horizontal scaling wasn't possible** with a tightly coupled monolith. We needed independent services that could scale on-demand."

### Q2: What were the specific pain points you faced?

**Answer**:
> "Three major pain points:
>
> 1. **Deployment Risk**: Fixing a bug in the email service meant deploying the entire monolith, risking unrelated components.
> 2. **Cost Inefficiency**: The EC2 instance ran 24/7, but ingestion only happened during business hours. We wasted money on idle capacity.
> 3. **Team Coordination**: Different teams (ingestion, workflow, reporting) all modified the same codebase, leading to merge conflicts and coordination overhead.
>
> Each team wanted to deploy independently, but the monolith forced them to coordinate—slow and error-prone."

---

**← Previous**: [00_Project_Overview.md](./GUIDE_00_Project_Overview.md) | **→ Next**: [02_Microservices_Architecture.md](./GUIDE_02_Microservices_Architecture.md)

---

**Remember**: Monolithic architectures aren't "bad" – they're appropriate for small, well-scoped applications. The Gold Report started as a good fit but outgrew it quickly. Understanding why is crucial for architectural decision-making.
