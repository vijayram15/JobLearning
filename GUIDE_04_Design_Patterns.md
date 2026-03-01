# 🎨 Section 4: Design Patterns Used

## Overview

Design patterns are proven solutions to common architectural problems. For Excess Management System, we use several patterns to achieve scalability, reliability, and maintainability. This section explains each pattern, why we use it, and how it's applied.

---

## 🎯 Patterns at a Glance

```
Pattern Name                Where Applied              Problem Solved
─────────────────────────────────────────────────────────────────────
📌 API Gateway              Request entry point        Centralized security
🔊 Event-Driven             SQS, EventBridge          Loose coupling
📊 CQRS                      Read from DynamoDB        Query optimization
🛡️  Circuit Breaker         Lambda → DB calls         Fault tolerance
🎭 Saga                      Multi-step transactions   Distributed consistency
📦 Database per Service      Each service owns data    Independent scaling
🔄 Idempotency              Message processing        No duplicates
🗳️  Outbox                   Reliable messaging        Exactly-once semantics
```

---

## 1️⃣ API GATEWAY PATTERN

### Concept

All external requests route through a single entry point (API Gateway), which handles routing, security, rate limiting, and protocol translation.

```
Before (without API Gateway):
  User 1 → Task Execution Service (Spring Boot) → JBPM DB
  User 2 → Task Execution Service → JBPM DB
  User 3 → Task Execution Service → JBPM DB
  ??     → Direct calls bypass security

After (with API Gateway):
  User 1 ↘
  User 2 → API Gateway → Validate JWT ✓ → Route to Task Execution
  User 3 ↗               Rate limit ✓
           All requests go here first!
```

### Implementation in Gold Report

```
API Gateway Rules:
  1. All requests require JWT token
  2. Validate token signature (OAuth2)
  3. Rate limiting: 100 requests/minute per user
  4. Route /tasks → Task Execution Service
  5. Route /health → Health check endpoint

Flow:
  User sends HTTP request with JWT
    ↓
  API Gateway intercepts
    ↓
  Gateway validates JWT (signature, expiration)
    ↓
  Gateway applies rate limiting
    ↓
  If valid: Route to backend (Task Execution Service)
    ↓
  If invalid: Return 401 Unauthorized
```

### Benefits

```
✅ Centralized Security: One place to enforce OAuth2/JWT
✅ Rate Limiting: Prevent DDoS and abuse
✅ Routing: Route different URLs to different backends
✅ Protocol Translation: Could convert REST → gRPC if needed
✅ Logging: All requests logged in one place for auditing
✅ No Backend Changes: If auth changes, update only gateway
```

### Interview Question

**Q: Why use API Gateway instead of putting security in the Spring Boot app?**

> "Centralization is key. If security logic is in Spring Boot, then:
> - Hard to enforce consistently (each endpoint must remember)
> - If we add another backend service, must duplicate security code
> - Harder to change auth policy (must redeploy backend)
>
> With API Gateway:
> - Security enforced before request reaches any backend
> - Single choke point (easy to audit, change, monitor)
> - Backend trusts API Gateway has already validated request
> - Can add/remove backends without changing security rules"

---

## 2️⃣ EVENT-DRIVEN ARCHITECTURE PATTERN

### Concept

Services communicate by producing and consuming events (messages), rather than direct calls. Enables loose coupling and asynchronous processing.

```
Tight Coupling (Monolithic):
  Service A → calls Service B directly
  Service A → waits for Service B response
  If B is slow/down: A blocks and fails

Event-Driven (Microservices):
  Service A → publishes event to message queue
  Service A → returns immediately (doesn't wait)
  Service B → subscribes to event queue
  Service B → processes event asynchronously
  Service A doesn't care if B exists!
```

### Implementation in Excess Management System

```
Event Chain:
  
  1. INGESTION EVENT
     Event: IngestMessageReceived
     Data: { message_id, raw_payload, timestamp }
     Published to: SQS queue
     Consumers: Workflow Lambda

  2. WORKFLOW EVENT
     Event: WorkflowInstanceCreated
     Data: { message_id, jbpm_instance_id, timestamp }
     Published to: (could publish to EventBridge/SNS)
     Consumers: (future: notifications, analytics)

  3. TASK EXECUTION EVENT
     Event: TaskCompleted
     Data: { task_id, user_id, decision, timestamp }
     Published to: (stored in DynamoDB state)
     Consumers: Email reporting, audit logs

  4. EMAIL EVENT
     Event: ReportEmailSent
     Data: { recipients, record_count, timestamp }
     Published to: Audit RDS
     Consumers: (compliance team queries logs)
```

### Benefits

```
✅ Loose Coupling: Services don't depend on each other
✅ Async Processing: Ingestion doesn't wait for workflow
✅ Scalability: Each service scales independently
✅ Buffers Spikes: Queue absorbs 2,000 messages/15min
✅ Can Add Consumers: New reporting needs? Add new consumer
✅ Failure Isolation: If Workflow fails, Ingestion still works
```

### Code Example (Pseudo)

```java
// Ingestion Lambda
public void handleIngestRequest() {
    Message msg = kafkaConsumer.poll();
    oracle.insert(msg, status="NEW");
    
    // Publish to SQS (fire and forget)
    sqsClient.publishMessage(
        queue="workflow-queue",
        message=msg
    );
    // Don't wait for response!
}

// Workflow Lambda
public void handleSQSMessage(SQSEvent event) {
    for (SQSMessage msg : event.getRecords()) {
        Message fullMsg = oracle.getById(msg.messageId);
        
        // Create JBPM workflow
        WorkflowInstance instance = 
            jbpmClient.createInstance(fullMsg);
        
        // Store correlation
        dynamodb.put(
            key=msg.messageId,
            value=instance
        );
        
        // Acknowledge to SQS (remove from queue)
        sqs.deleteMessage(msg);
    }
}
```

### Interview Question

**Q: How does event-driven prevent cascading failures?**

> "Good question. Consider a failure scenario:
>
> **Monolithic (Direct Calls)**:
> - User clicks 'Process Task'
> - UI calls Task Execution Service
> - Service queries JBPM DB (timeout occurs)
> - User sees error immediately
> - If JBPM is down, entire system appears broken
>
> **Event-Driven (Our Setup)**:
> - User clicks 'Process Task'
> - UI calls Task Execution Service
> - Service updates JBPM DB (local transaction, fast)
> - Service publishes event to queue (fire & forget)
> - User sees success immediately
> - If downstream Email Lambda fails to process events:
>   • Reporting delayed, but
>   • Core task processing still works
>   • Events sit safely in queue, retried later
>   • No cascading failure to user experience
>
> Events isolate failures. Downstream problems don't affect user-facing services."

---

## 3️⃣ CQRS (Command Query Responsibility Segregation) PATTERN

### Concept

Separate read operations (queries) from write operations (commands) into different services/data stores.

```
Traditional (Read and Write Same):
  User Request → Service → Query/Update DB → Return
  Problem: Read optimization conflicts with write optimization

CQRS (Separate Read & Write):
  Write Path: User updates data → Service → Update (Oracle + JBPM DB)
  Read Path: User queries data → Service → Read (DynamoDB cached copy)
  Problem: Eventually consistent, but reads are fast!
```

### Implementation in Gold Report

```
COMMAND (Write Path):
  Task Execution Service
    1. User completes task
    2. Call JBPM REST API: Update task status
    3. Save to JBPM DB (workflow state)
    4. Save to DynamoDB (payload + status)
    5. Write to audit log
    6. Return success to user

Read Path from COMMAND (as a side effect):
    Both stores updated immediately

EMAIL SERVICE (Read Path):
  1. Query ONLY DynamoDB (very fast)
    SELECT * FROM instances WHERE status='COMPLETED'
    Result: < 100ms (no joins, simple schema)
  
  2. Generate report
  3. Send email

Why Two Places?
  • JBPM DB: Source of truth for workflow state (consistency)
  • DynamoDB: Optimized for reporting reads (performance)
  
  Trade-off:
    Accept: DynamoDB is ~100ms behind JBPM DB
    Gain: Report generation 50x faster
```

### Benefits

```
✅ Read Performance: DynamoDB queries < 100ms vs. JBPM > 1s
✅ Write Flexibility: Can optimize writes independently
✅ Scalability: Read replicas can scale independently
✅ Caching: DynamoDB acts as cache layer for reports
✅ Reporting Isolation: Report queries don't hit JBPM
```

### Interview Question

**Q: Why is CQRS applicable here but not everywhere?**

> "CQRS shines when reads and writes have different optimization needs:
>
> **Our Use Case** (Perfect for CQRS):
> - Writes: Task completion (infrequent, ~100/min)
> - Reads: Report generation (frequent queries on entire dataset)
> - WRITE optimization: ACID, workflow state consistency
> - READ optimization: Fast scans, no joins
> - Solution: Write to both, read from DynamoDB ✓
>
> **When CQRS is Overkill**:
> - Reads = Writes (CRUD app, balanced)
> - Real-time consistency required (financial transactions)
> - Small dataset (no performance difference)
>
> **Key Insight**:
> CQRS isn't always needed. Use only when read/write patterns are significantly different."

---

## 4️⃣ CIRCUIT BREAKER PATTERN

### Concept

When calling external services (DB, APIs), detect failures and automatically stop sending requests temporarily ("open circuit"), then gradually retry ("half-open").

```
Normal Operation (Circuit Closed):
  Request → Service → Success ✓

Failures Detected (Circuit Open):
  Request → Circuit Breaker says "STOP" ✗
  Requests blocked, not sent to failing service
  Wait 30 seconds...

Partial Recovery (Circuit Half-Open):
  Send single test request
  If successful: Close circuit, resume normal flow
  If fails: Stay open, wait again
```

### Implementation in Gold Report

```
Lambda → JBPM DB Connections

Without Circuit Breaker:
  If JBPM DB down:
  • Workflow Lambda tries to call it
  • Waits 5 seconds for timeout
  • Fails, retries immediately
  • 20 concurrent lambdas all timeout
  • System appears hung for 5 seconds

With Circuit Breaker:
  If JBPM DB down:
  • Lambda 1: Fails, Circuit records failure
  • Lambda 2: Detects open circuit, fails fast (1ms)
  • Lambda 3: Fails fast (1ms)
  • ...
  • No cascading timeouts
  • Clear error messages
  • SQS retries automatically
```

### Benefits

```
✅ Fail Fast: Don't wait for timeouts
✅ Prevent Cascades: Stop hammering failing service
✅ Recovery Aware: Gradually retry as service recovers
✅ Observable: Circuit breaker state visible in metrics
✅ Automatic Mitigation: No manual intervention needed
```

### Interview Question

**Q: How does Circuit Breaker enable resilience?**

> "Circuit Breaker prevents cascading failures by stopping requests to failing services:
>
> Scenario: JBPM DB (on-prem) has network connectivity issue
>
> Without Circuit Breaker:
> - Each Lambda invocation tries to call JBPM
> - Each times out after 5 seconds
> - 20 concurrent Lambdas × 5 second timeout = system stalled
> - Queue backs up, messages wait indefinitely
> - Looks like system is hung
>
> With Circuit Breaker:
> - First few calls fail, circuit opens
> - Subsequent calls get fast failure (1ms)
> - Operations team alerted immediately
> - Messages stay safely in SQS (visible in queue metrics)
> - When JBPM recovers, circuit half-opens, one request succeeds, circuit closes
> - System auto-recovers without manual intervention
>
> Key: Don't waste resources on guaranteed failures. Fail fast and retry intelligently."

📌 **2026 SUGGESTION:**  
Modern systems implement Circuit Breaker via **Resilience4j** library (Java standard 2026) or **Service Mesh (Istio)** for infrastructure-level resilience. These automate the pattern without manual coding. Watch for: "Tell me about resilience strategies you've implemented." Be ready to discuss both application-level (Resilience4j) and infrastructure-level (Istio) approaches. See `ADVANCED_2026_Technology_Standards.md` for modern resilience evolution.

---

## 5️⃣ SAGA PATTERN (Distributed Transactions)

### Concept

For operations spanning multiple services, execute steps sequentially and maintain consistency through compensating actions if anything fails.

```
Distributed Transaction: Create Workflow Instance

Step 1: Oracle → Store message               ✓
Step 2: JBPM DB → Create workflow instance ✓
Step 3: DynamoDB → Store metadata          ❌ FAILS!

What Now?
  Saga Option 1 (Orchestration): Central coordinator rolls back
    → Delete from JBPM
    → Mark in Oracle as FAILED
  Saga Option 2 (Choreography): Services emit events, others listen
```

### Implementation in Gold Report

```
Orchestration Approach (More Explicit):
  
  Saga Coordinator (Workflow Lambda):
    
    Step 1: Try to create JBPM instance
      if success
        Step 2: Try to write to DynamoDB
          if success
            Saga complete, commit
          else
            Compensate:
              Delete from JBPM
              Return error to SQS (message will retry)
      else
        Fail immediately (no changes)
```

### Benefits

```
✅ Distributed Consistency: Multiple services act as one transaction
✅ Compensating Actions: Clearly defined rollback logic
✅ Observability: Can see saga state in logs
✅ No Complex 2PC: Avoid distributed locks and coordination overhead
```

### Interview Question

**Q: Why not use traditional database transactions?**

> "Traditional transactions require all services to share one database:
>
> Old Monolithic Approach:
> - Everything in Oracle
> - BEGIN TRANSACTION
> - Update all tables atomically
> - COMMIT or ROLLBACK
> - ✓ Simple, strong consistency
>
> Microservices Reality:
> - JBPM DB is separate (on-prem)
> - DynamoDB is separate (AWS)
> - Oracle is separate
> - Can't use database ACID across different systems
> 
> Alternative Approach (Global XA Transactions):
> - Prepare all changes in each DB
> - Coordinator says commit/rollback
> - Very slow, complex, prone to failure
> - [Not recommended]
>
> Saga Pattern (Better):
> - Execute steps sequentially
> - If any step fails, call compensating actions (rollback)
> - Simpler than 2-phase commit, more reliable
> - Eventual consistency (not immediate)
> - ✓ Practical for distributed systems"

---

## 6️⃣ DATABASE PER SERVICE PATTERN

### Concept

Each microservice owns its own database(s), preventing tight coupling and enabling independent scaling.

```
Before (Single Shared DB):
  All services share Oracle
    • Schema changes require coordination
    • Resource contention (connections, locks)
    • Scaling limited to scaling entire DB
    • One team controls all data

After (Database per Service):
  Ingestion owns: Oracle (raw messages)
  Workflow owns: JBPM DB + DynamoDB metadata
  Email owns: Audit RDS table
    ✓ No coordination needed
    ✓ Each service scales independently
    ✓ Teams own their data
```

### Implementation in Gold Report

```
Ingestion Service:
  Owns: Oracle (EXCESS_MESSAGE table)
  Cannot access: JBPM DB, DynamoDB, Audit RDS

Workflow Service:
  Owns: JBPM DB (instances), DynamoDB (metadata)
  Cannot access: Raw messages except via Oracle query
  Cannot access: Audit RDS directly

Task Execution Service:
  Owns: JBPM DB updates, DynamoDB updates
  Cannot access: Audit RDS directly

Email Service:
  Owns: Audit RDS (email log entries)
  Query-only: DynamoDB (for report data)
```

### Benefits

```
✅ Independent Scaling: Scale Workflow without scaling Task Execution
✅ Technology Flexibility: Each service picks best DB for its needs
✅ Team Autonomy: Each team owns their data schema
✅ Polyglot Persistence: Oracle for compliance, DynamoDB for speed, RDS for audits
```

### Challenges

```
⚠️ Data Consistency: Changes in one service may take time to reflect in others
⚠️ Duplication: Some data exists in multiple places (JBPM + DynamoDB)
⚠️ Query Complexity: Can't join across databases easily
```

---

## 7️⃣ IDEMPOTENCY PATTERN

### Concept

Ensure that executing the same operation multiple times produces the same result as executing it once.

```
Without Idempotency:
  Retry scenario: SQS message retried after timeout
    1. Create JBPM instance (succeeds)
    2. Lambda crashes after updating DynamoDB but before SQS acknowledgment
    3. SQS redelivers message (thinks it failed)
    4. Create JBPM instance again (duplicate! ❌)
    5. Two instances, one message, reporting corrupted

With Idempotency:
    1. Create JBPM instance (succeeds)
    2. Lambda crashes
    3. SQS redelivers message
    4. Check: Does instance already exist for this message? (YES)
    5. Skip creation, return success
    6. No duplicates ✓
```

### Implementation in Gold Report

```
Workflow Lambda (Idempotency):

public void processMessage(String messageId) {
    // Check if already processed
    WorkflowInstance existing = jbpmDB.queryByMessageId(messageId);
    
    if (existing != null) {
        // Already processed, return success (idempotent!)
        return success;
    }
    
    // First time processing
    WorkflowInstance instance = jbpmClient.createInstance(messageId);
    dynamodb.put(messageId, instance);
    
    return success;
}

Key:
  • messageId is idempotency key
  • If same message comes again, result is same
  • No side effects from duplicate processing
```

### Benefits

```
✅ Reliability: Retries don't cause corruption
✅ Simplicity: Don't need complex distributed transactions
✅ Observable: Can easily detect duplicate processing (metrics)
```

---

## 8️⃣ OUTBOX PATTERN (Reliable Messaging)

### Concept

Guarantee that an action and a message are either both performed or both rolled back (no partial states).

```
Without Outbox:
  Step 1: Write to Oracle
  Step 2: Publish to SQS
  
  Problem: If step 2 fails, message lost but data written

With Outbox:
  Step 1: Write to Oracle
  Step 2: Write to Oracle "outbox" table (same transaction!)
  Step 3: Background job polls outbox, publishes to SQS
  Step 4: On success, delete from outbox
  
  Guarantee: Either both happen or neither
```

### Implementation in Gold Report

```
Ingestion Lambda (Outbox Pattern):

Oracle Transaction:
  BEGIN
    INSERT INTO EXCESS_MESSAGE (id, payload, status)
    VALUES (msg-123, {...}, 'NEW');
    
    INSERT INTO OUTBOX (message_id, event_type, event_data)
    VALUES (msg-123, 'MessageIngested', {...});
  COMMIT

Background Job (runs every 10 sec):
  SELECT * FROM OUTBOX WHERE published=false;
  FOR each row:
    Publish to SQS
    UPDATE OUTBOX SET published=true
    
Benefit:
  • If SQS publish fails, it's retried by background job
  • Message will eventually reach SQS
  • Guaranteed exactly-once semantics (or at-least-once with idempotency)
```

### Benefits

```
✅ Guaranteed Delivery: Message won't be lost
✅ Exactly-Once (with Idempotency): Combine with idempotency for exactly-once
✅ No Message Loss: More reliable than:
     (write → publish) or (publish → write)
```

---

## 📊 Patterns Summary Table

| Pattern | Problem Solved | Cost | Complexity |
|---------|----------------|------|------------|
| **API Gateway** | Security, routing | Medium | Low |
| **Event-Driven** | Coupling, async processing | Medium | Medium |
| **CQRS** | Read/write optimization | Medium | High |
| **Circuit Breaker** | Cascading failures | Low | Medium |
| **Saga** | Distributed transactions | Medium | High |
| **Database per Service** | Scaling, autonomy | Medium | Medium |
| **Idempotency** | Retry safety | Low | Low |
| **Outbox** | Reliable messaging | Low | Medium |

---

## 🎓 Interview Pattern Questions

### Q: Describe a pattern you used to solve a specific problem.

**Answer**:
> "I used the **Saga Pattern** to handle distributed transactions in our microservices.
>
> Problem: When creating a workflow instance, we need to coordinate between JBPM DB and DynamoDB. If one fails, we can't leave data in an inconsistent state.
>
> Solution: Orchestration Saga
> 1. Workflow Lambda calls JBPM API to create instance
> 2. If success: Write metadata to DynamoDB
> 3. If DynamoDB fails: Compensating action—delete from JBPM
> 4. If JBPM fails: No compensation needed, SQS will retry entire message
>
> Result: Either both succeed or both fail. No partial states.
>
> Alternative: Choreography Saga
> - Publish WorkflowInstanceCreated event
> - Email service subscribes, generates report metadata
> - If report fails, Email service publishes WorkflowRollback event
> - More decoupled, but harder to trace
>
> We chose Orchestration for simplicity and observability."

### Q: Which pattern would you apply to a new requirement?

**Answer** (Example):
> "If we needed to add real-time notifications when a task is assigned:
>
> **Pattern: Event-Driven + API Gateway**
>
> 1. Task Execution Service publishes TaskAssigned event to SQS
> 2. New Notification Lambda subscribes to queue
> 3. Notification Lambda calls Notification API (WebSocket, push, email)
> 4. API Gateway secures the notification endpoint
>
> Why not: Direct service calls
> - Tight coupling
> - If Notification service is down, task processing blocked
>
> Why Event-Driven:
> - Task assignment proceeds immediately, notification async
> - Notification failures don't impact core workflow
> - Easy to add new notification channels later"

---

**← Previous**: [03_Technology_Stack.md](./GUIDE_03_Technology_Stack.md) | **→ Next**: [05_Real_World_Challenges.md](./GUIDE_05_Real_World_Challenges.md)

---

**Key Takeaway**: Patterns are solutions to known problems. Use them when you recognize the problem, not just because they sound cool. The mark of a good architect is knowing when to apply a pattern and when to keep things simple.
