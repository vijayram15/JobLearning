# 🔐 Section 6: Security & Best Practices

## Overview

Security isn't an afterthought—it's foundational to any production system. This section covers how we secured the Excess Management System and the security principles behind each decision. This is critical interview territory because security is increasingly expected in system design discussions.

---

## 🎯 Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEFENSE IN DEPTH                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: NETWORK & TRANSPORT                                   │
│  ├─ VPC: Private subnet for sensitive services                 │
│  ├─ TLS/SSL: All traffic encrypted in transit                  │
│  ├─ API Gateway: WAF (Web Application Firewall) rules          │
│  └─ Security Groups: Restrict inbound/outbound traffic         │
│                                                                  │
│  Layer 2: AUTHENTICATION & AUTHORIZATION                        │
│  ├─ API Gateway: OAuth2/JWT validation                         │
│  ├─ IAM Roles: Least privilege per Lambda function             │
│  ├─ Secrets Manager: Encrypted credentials                     │
│  └─ MFA: For administrative access                             │
│                                                                  │
│  Layer 3: DATA SECURITY                                         │
│  ├─ KMS: Encryption keys managed by AWS                        │
│  ├─ DB Encryption: Oracle TDE, RDS encrypted, DynamoDB KMS     │
│  ├─ S3: Server-side encryption, no public access               │
│  └─ Secrets: Never hardcoded, always in Secrets Manager        │
│                                                                  │
│  Layer 4: APPLICATION LOGIC                                     │
│  ├─ Input Validation: Sanitize all inputs vs. SQL injection    │
│  ├─ RBAC: Role-Based Access Control on API endpoints           │
│  ├─ Audit Logging: Every action logged for compliance          │
│  └─ Rate Limiting: Prevent brute force, DDoS                   │
│                                                                  │
│  Layer 5: MONITORING & RESPONSE                                 │
│  ├─ CloudWatch: Log all security events                        │
│  ├─ GuardDuty: Anomaly detection, threat analysis              │
│  ├─ Alerts: Real-time security event notifications             │
│  └─ Incident Response: Playbooks for security breaches         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ AUTHENTICATION: OAuth2 + JWT

### Concept

Users authenticate once, receive a token (JWT), and use the token for subsequent API calls. Token is validated at API Gateway, not application.

### Implementation

```
Flow:

1. USER LOGIN (OAuth2 Provider)
   ┌─────────────────────────────────┐
   │ User enters credentials         │
   │ Username: john@company.com      │
   │ Password: xxxxxxxxx             │
   └────────────┬────────────────────┘
                ↓
   ┌─────────────────────────────────┐
   │ OAuth2 Server                   │
   │ • Verify credentials            │
   │ • Check MFA                     │
   │ • Create JWT token              │
   └────────────┬────────────────────┘
                ↓
   Token: eyJhbGc....... (signed by OAuth server)

2. TOKEN USAGE (Subsequent API Calls)
   ┌─────────────────────────────────┐
   │ GET /tasks                      │
   │ Header: Authorization: Bearer eyJhbGc... │
   └────────────┬────────────────────┘
                ↓
   ┌─────────────────────────────────┐
   │ API Gateway                     │
   │ • Extract token from header     │
   │ • Validate signature (public key)
   │ • Check expiration             │
   │ • Verify claims (roles)         │
   └────────────┬────────────────────┘
     ✓ Valid: Forward to backend
     ✗ Invalid: Return 401 Unauthorized

3. BACKEND PROCESSING
   ┌─────────────────────────────────┐
   │ Task Execution Service          │
   │ • Receives JWT claims (user ID) │
   │ • No need to re-authenticate    │
   │ • Check claims for permissions  │
   │ • Process request               │
   └─────────────────────────────────┘
```

### JWT Claims

```json
{
  "sub": "john@company.com",        // Subject (user ID)
  "roles": ["user", "task_executor"], // User roles
  "client_id": "ACME_CORP",         // Client organization
  "permissions": [                   // Specific permissions
    "read:tasks",
    "write:tasks"
  ],
  "exp": 1740825600,                // Expiration (Unix timestamp)
  "iat": 1740739200                 // Issued at
}
```

### Benefits

```
✅ Stateless: No session lookup needed (faster)
✅ Scalable: Multiple backends can validate same token
✅ Standard: OAuth2/JWT widely recognized
✅ Flexible: Can include custom claims (roles, permissions)
✅ Revokable: Expiration + token blacklist if needed
```

---

## 2️⃣ AUTHORIZATION: Role-Based Access Control (RBAC)

### Concept

Different roles (user, admin, auditor) have different permissions. Enforce at API Gateway and application levels.

### Implementation in Gold Report

```
Roles & Permissions:

ROLE: task_executor
├─ POST /tasks/{id}/complete (process a task)
├─ GET  /tasks (view assigned tasks)
├─ GET  /tasks/{id} (view task details)
└─ Cannot: DELETE, generate reports, delete other users' tasks

ROLE: manager
├─ All task_executor permissions
├─ GET  /analytics/team-metrics
├─ GET  /reports/daily
└─ Cannot: System administration, configuring JBPM

ROLE: admin
├─ All permissions
├─ POST /admin/configure-workflow
├─ DELETE /data/messages (full access)
├─ SET /system/pause-processing
└─ Can: Everything (with audit logging)

ROLE: auditor
├─ GET  /audit/all-emails-sent
├─ GET  /audit/user-actions
├─ GET  /reports/compliance
└─ Cannot: Modify data (read-only)
```

### API Gateway Authorization

```yaml
# API Gateway Policy

GET /tasks:
  • Require role: task_executor
  • Claim validation: roles contains "task_executor"
  
POST /admin/configure:
  • Require role: admin
  • Additionally: Check IP whitelisted (if external)
  • Log access (admin actions are sensitive)

GET /audit:
  • Require role: auditor
  • Additionally: Filter by user's organization
  • Cannot access other client's audit logs
```

### Example Code

```java
// Spring Boot - Method-level Authorization

@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    
    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('TASK_EXECUTOR', 'MANAGER', 'ADMIN')")
    public Task getTask(@PathVariable Long id) {
        // User must have one of these roles
        return taskService.getTask(id);
    }
    
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteTask(@PathVariable Long id) {
        // Only admin can delete
        taskService.delete(id);
    }
    
    @GetMapping("/all")
    @PreAuthorize("hasAnyRole('ADMIN', 'AUDITOR')")
    public List<Task> getAllTasks() {
        // Admin or auditor can see all tasks
        return taskService.getAll();
    }
}
```

---

## 3️⃣ DATA SECURITY: Encryption at Rest & in Transit

### Encryption in Transit (TLS/SSL)

```
REQUIREMENT: All communication encrypted

Implementation:
  • HTTPS everywhere (not HTTP)
  • TLS 1.2 or higher (TLS 1.3 preferred)
  • Valid certificates (not self-signed in production)
  
API Gateway → Lambda: Encrypted by AWS
API Gateway → JBPM DB: TLS connection
API Gateway → Task Execution: HTTPS only
Task Execution → JBPM DB: Encrypted connection
Lambda → SQS: AWS-managed encryption
Lambda → DynamoDB: AWS-managed encryption
```

### Encryption at Rest

```
REQUIREMENT: Data stored encrypted on disk/database

Implementation by Component:

1. Oracle Database
   • Transparent Data Encryption (TDE)
   • Database encryption key managed by Oracle
   • All tables encrypted automatically
   
2. JBPM Database (RDS)
   • RDS Encryption enabled at provisioning
   • AWS KMS manages encryption key
   • Cannot decrypt without KMS permission
   
3. DynamoDB
   • Server-side encryption with KMS
   • Default: AWS-managed key (good enough)
   • Optional: Customer-managed key (higher control)
   • Data encrypted before written to disk
   
4. S3 Bucket (Report Storage)
   • Server-side encryption (SSE-S3 or SSE-KMS)
   • Block public access (enabled)
   • Only authorized services can read
   
5. Audit RDS
   • Encryption enabled (same as JBPM DB)
   • Contains sensitive audit info (who did what)
   • Must be encrypted
   
6. CloudWatch Logs
   • Customer-managed KMS encryption optional
   • Encrypts logs to prevent tampering
```

---

## 4️⃣ SECRETS MANAGEMENT

### The Problem

```
DON'T DO THIS:

# .env file
DB_PASSWORD=SuperSecret123
API_KEY=sk_live_abcd1234efgh5678

Problem:
  ✗ If code is committed, secret is exposed
  ✗ If developer machine is hacked, secret is there
  ✗ Can't rotate secrets without code change
  ✗ Can't have different secrets per environment
```

### The Solution: AWS Secrets Manager

```
Architecture:

┌───────────────────────────────┐
│ AWS Secrets Manager           │
├───────────────────────────────┤
│ Secret: jbpm-db-password      │
│ Value: (encrypted, rotated)   │
│ Rotation: Every 90 days       │
│ Access: Only Lambda with IAM  │
│ Audit: All accesses logged    │
└───────────────────────────────┘
         ↑
         │ (retrieves at runtime)
         │
┌───────────────────────────────┐
│ Workflow Lambda               │
├───────────────────────────────┤
│ On startup:                   │
│  password = sm.getSecret      │
│            ("jbpm-db-pass")  │
│                               │
│ Connect to JBPM DB with pwd  │
└───────────────────────────────┘
```

### Implementation

```java
// Workflow Lambda - Retrieve secret at runtime

import software.amazon.awssdk.services.secretsmanager.*;

public class WorkflowLambda {
    
    private String getJBPMPassword() {
        SecretsManagerClient client = 
            SecretsManagerClient.builder().build();
        
        GetSecretValueRequest request = 
            GetSecretValueRequest.builder()
                .secretId("jbpm-db-password")
                .build();
        
        GetSecretValueResponse response = 
            client.getSecretValue(request);
        
        return response.secretString();
    }
    
    public void handleRequest() {
        String password = getJBPMPassword();
        // Use password to connect to JBPM DB
        // Never log the password!
    }
}
```

### Benefits

```
✅ No secrets in code
✅ Automatic rotation (configurable)
✅ IAM control (only specific services can access)
✅ Audit trail (CloudTrail logs all accesses)
✅ Encryption (secrets stored encrypted)
✅ Different secrets per environment (dev, staging, prod)
```

---

## 5️⃣ LEAST PRIVILEGE: IAM Roles & Policies

### Concept

Each service has minimal permissions needed to function. Not "admin on everything."

### Implementation

```
Lambda Execution Roles:

INGESTION LAMBDA Role:
  Permissions:
    • s3:GetObject → Read from S3 (Kafka certs if needed)
    • sqs:SendMessage → Publish to SQS queue
    • logs:CreateLogGroup, CreateLogStream → CloudWatch logs
    • dynamodb:GetItem → (Maybe, for idempotency check)
  
  BLOCKED:
    • dynamodb:DeleteItem (✗ Cannot delete)
    • kms:Decrypt (✗ But decrypt via service)
    • iam:* (✗ Cannot modify IAM)
    • s3:DeleteObject (✗ Cannot delete)

WORKFLOW LAMBDA Role:
  Permissions:
    • sqs:ReceiveMessage, DeleteMessage → Consume from SQS
    • logs:CreateLogGroup, CreateLogStream → CloudWatch
    • dynamodb:PutItem, GetItem → DynamoDB access
    • rds-db:connect → Connect to JBPM DB (RDS)
    • kms:Decrypt → Decrypt RDS password from Secrets Manager
  
  BLOCKED:
    • dynamodb:DeleteTable (✗)
    • sqs:SendMessage (✗ This is for Ingestion Lambda only)
    • s3:* (✗ Email Lambda handles S3)

EMAIL LAMBDA Role:
  Permissions:
    • dynamodb:Query, Scan → Read from DynamoDB
    • rds-db:connect → Query Audit RDS
    • s3:PutObject → Store reports
    • ses:SendBulkEmail → Send emails
    • logs:CreateLogGroup, CreateLogStream → CloudWatch
  
  BLOCKED:
    • dynamodb:PutItem (✗ Read-only)
    • sqs:* (✗ Doesn't use SQS)
    • lambda:InvokeFunction (✗)
```

### IAM Policy Example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSQSOperations",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:workflow-queue"
    },
    {
      "Sid": "AllowDynamoDBRead",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/instances"
    },
    {
      "Sid": "AllowSecretsAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:jbpm-db-password"
    },
    {
      "Sid": "AllowCloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789012:*"
    }
  ]
}
```

### Benefits

```
✅ Containment: If Lambda is compromised, damage is limited
✅ Auditing: Can see exactly what each service can do
✅ Compliance: Satisfies least-privilege requirements
✅ Debugging: Clear errors if permissions missing (not silent failures)
```

---

## 6️⃣ INPUT VALIDATION & INJECTION PREVENTION

### SQL Injection Prevention

```
DON'T DO THIS:

String query = "SELECT * FROM tasks WHERE id = " + userId;
// If userId = "1 OR 1=1", returns ALL tasks!

DO THIS:

String query = "SELECT * FROM tasks WHERE id = ?";
PreparedStatement stmt = db.prepareStatement(query);
stmt.setInt(1, userId);  // Parameter binding (safe)
ResultSet results = stmt.executeQuery();
```

### Request Validation

```java
// Spring Boot - Validate input on API endpoint

@RestController
public class TaskController {
    
    @PostMapping("/tasks/{id}/complete")
    public void completeTask(
        @PathVariable @NotNull Long id,
        @RequestBody @Valid CompleteTaskRequest request
    ) {
        // @NotNull ensures id is not null
        // @Valid triggers validation on request object
    }
}

// Validation class
@Data
public class CompleteTaskRequest {
    @NotBlank(message = "Decision cannot be blank")
    @Pattern(regexp = "^(APPROVE|REJECT)$", 
             message = "Decision must be APPROVE or REJECT")
    private String decision;
    
    @NotBlank
    @Length(min = 1, max = 500, message = "Comments too long")
    private String comments;
}
```

### XSS (Cross-Site Scripting) Prevention

```
Scenario:
  User submits comment: "<script>alert('hacked')</script>"
  If stored unescaped and displayed on web page:
    Script runs in browser = security breach

Prevention:
  • Escape output when rendering (HTML entities)
  • Use templating engines that auto-escape (Spring Thymeleaf, etc.)
  • Content Security Policy (CSP) headers
  • Never use innerHTML with user data
```

---

## 7️⃣ AUDIT LOGGING & COMPLIANCE

### What to Log

```
Security-Relevant Events:

1. AUTHENTICATION
   • User login (success/failure)
   • Token generation
   • Token refresh
   • MFA verification
   
2. AUTHORIZATION
   • Permission denied (access attempt to restricted resource)
   • Role changes
   • IAM policy changes
   
3. DATA ACCESS
   • Read sensitive data (who, what, when)
   • Update sensitive data
   • Delete data
   
4. SYSTEM CHANGES
   • Configuration changes
   • Database schema changes
   • Security group modifications

5. SECURITY EVENTS
   • Failed intrusion attempts
   • Rate limit exceeded
   • Suspicious activity
```

### Implementation in Gold Report

```java
// Audit Logging

@Component
public class AuditLogger {
    
    public void logUserAction(
        String userId,
        String action,
        String resource,
        String outcome
    ) {
        auditRDS.insert(
            "INSERT INTO audit_log " +
            "(user_id, action, resource, outcome, timestamp) " +
            "VALUES (?, ?, ?, ?, NOW())",
            userId, action, resource, outcome
        );
    }
}

// Example Usage

auditLogger.logUserAction(
    "john@company.com",
    "COMPLETED_TASK",
    "task-123",
    "SUCCESS"
);

auditLogger.logUserAction(
    "jane@company.com",
    "UNAUTHORIZED_ACCESS",
    "/admin/configure",
    "DENIED"
);
```

### Compliance Benefits

```
✅ Regulatory Compliance: Prove you log security events
✅ Forensics: Investigate security incidents
✅ Internal Audits: Demonstrate controls
✅ User Attribution: Know who did what (non-repudiation)
```

---

## 8️⃣ MONITORING & INCIDENT RESPONSE

### CloudWatch Alarms

```
Critical Alarms:

1. AUTHENTICATION FAILURES
   Alarm: Failed login count > 10 in 5 minutes
   Action: Alert security team, rate-limit IP

2. UNAUTHORIZED ACCESS
   Alarm: 401/403 errors > threshold
   Action: Investigate potential attack

3. DynamoDB THROTTLING
   Alarm: Throttle count > 0
   Action: Scale table or investigate traffic spike

4. Lambda ERRORS
   Alarm: Error rate > 1%
   Action: Page on-call engineer

5. DATA DELETION
   Alarm: DELETE operations on audit_log or critical table
   Action: Immediate investigation
```

### Incident Response Playbook

```
If Security Breach Detected:

1. CONTAIN (Immediate)
   • Revoke compromised credentials
   • Isolate affected service
   • Enable detailed logging
   
2. INVESTIGATE (Minutes to Hours)
   • Analyze CloudWatch logs, CloudTrail
   • Determine: What data was accessed?
   • Determine: How did attacker get in?
   
3. REMEDIATE (Hours to Days)
   • Patch vulnerabilities
   • Rotate all potentially-exposed secrets
   • Deploy security fixes
   
4. COMMUNICATE
   • Internal notifications
   • Customer notifications (if PII exposed)
   • Regulatory notifications (if required)
   
5. POST-MORTEM
   • Root cause analysis
   • Preventive measures
   • Update security policies
```

---

## 📋 Security Checklist

- [ ] All communication uses TLS/HTTPS
- [ ] Database encryption enabled (at rest)
- [ ] Secrets stored in Secrets Manager, not in code
- [ ] IAM roles follow least privilege principle
- [ ] API Gateway enforces OAuth2/JWT
- [ ] Sensitive inputs validated and sanitized
- [ ] Audit logging for all security events
- [ ] CloudWatch alarms for suspicious activity
- [ ] Incident response plan documented
- [ ] Security patches applied regularly
- [ ] MFA enabled for administrative access
- [ ] S3 buckets have public access blocked
- [ ] DynamoDB encryption enabled
- [ ] Lambda functions run in VPC (if accessing databases)
- [ ] Regular security scanning (OWASP Dependency Check, Snyk)

---

## 🎓 Interview Questions on Security

### Q1: How did you secure your API?

**Answer**:
> "We implemented multiple layers of security:
>
> **1. Network Level**: All traffic uses TLS 1.2+. API Gateway sits as the entry point.
>
> **2. Authentication**: OAuth2 with JWT tokens. Users log in once, get a token with claims (user ID, roles), and use it for subsequent calls. Token validated at API Gateway before reaching backend.
>
> **3. Authorization**: Role-Based Access Control (RBAC). Different roles (task_executor, manager, admin) have different permissions. Validated at both API Gateway and application level.
>
> **4. Least Privilege IAM**: Each Lambda has minimal permissions. Ingestion Lambda can't access S3, Email Lambda can't modify JBPM DB, etc. Breach of one service doesn't compromise others.
>
> **5. Secrets Management**: Database passwords, API keys stored in AWS Secrets Manager (encrypted), not in code. Rotated automatically.
>
> **6. Audit Logging**: All security events logged (auth failures, permission denials, data access) for compliance and forensics."

### Q2: How did you prevent SQL injection?

**Answer**:
> "We used parameterized queries (prepared statements) throughout. Never concatenate user input into SQL queries. Example:
>
> Bad: String query = 'SELECT * FROM tasks WHERE id = ' + userId;
>
> Good: String query = 'SELECT * FROM tasks WHERE id = ?'
>         stmt.setInt(1, userId);
>
> This ensures user input is treated as data, not SQL code.
>
> Additionally, we validate input on API endpoints (non-null, expected format), use Spring Boot's parameterized query support, and ran OWASP dependency scans."

### Q3: How do you handle secrets (passwords, API keys) securely?

**Answer**:
> "Secrets are NEVER in code or environment variables. We use AWS Secrets Manager:
>
> - All database passwords, API keys stored encrypted in Secrets Manager
> - Each Lambda retrieves securely at runtime via IAM
> - Automatic rotation (configurable frequency)
> - Audit trail: All accesses logged for compliance
> - Different secrets per environment (dev vs. production)
>
> This way, even if a developer's machine is compromised or code is leaked, the actual production secrets remain safe."

---

**← Previous**: [05_Real_World_Challenges.md](./GUIDE_05_Real_World_Challenges.md) | **→ Next**: [07_Interview_Q&A_Comprehensive.md](./GUIDE_07_Interview_Q&A_Comprehensive.md)

---

**Key Takeaway**: Security is not a feature—it's a requirement. Interviewers expect you to think about security proactively, not just as an afterthought. "We handle X because..." shows maturity.
