# FastStart – AgentCore-Driven Case Assessment System  
## Technical Specification – Architecture, Data, and Design (AgentCore-first)

**Version:** 1.2  
**Last updated:** February 2026


---

## 1. Introduction

### 1.1 Overview

The **FastStart AI-Powered Case Triage & Caseworker Platform** is a cloud-native, AWS-hosted solution designed to automate the processing of citizen applications for public-sector caseworkers. The platform reduces manual effort by providing AI-generated insights while maintaining human decision-making oversight.

### 1.2 Problem Statement

Public sector caseworkers face growing workloads in manually reviewing applications, extracting key data, and interpreting policies. This leads to:

- Backlogs and delays
- Inconsistent decisions
- Limited scalability of the current process

### 1.3 Objectives

- **Reduce manual effort** via automated document intake and AI-assisted decision support
- **Improve consistency, traceability, and auditability** of case assessments
- **Accelerate application processing** while preserving human oversight
- **Enable reuse** across multiple councils or organisations with configurable eligibility rules
- **Provide a scalable, AWS Marketplace–ready** solution

### 1.4 Target Users

| User type | Description |
|----------|-------------|
| **Public-sector organisations** | Councils, agencies deploying the platform |
| **Caseworkers** | Staff reviewing AI outputs and making final decisions |
| **Managers** | Staff reviewing escalated cases |
| **Administrators** | Staff managing users, policies, and configuration |

---

## 2. Scope

### 2.1 In Scope

- Secure caseworker portal (login, dashboard, case list, case detail, decisions, notifications, settings)
- Document intake and automated classification
- AI workflow orchestration (multi-agent via Amazon Bedrock Agent Core)
- Configurable eligibility rules and policy evaluation
- Structured case summaries and decision support
- Decision publication API for upstream/citizen portal (GET decision)
- Policy customisation and runtime usage (YAML/JSON upload, versioning)
- Audit trails and immutability of decisions

### 2.2 Out of Scope

- Fully automated approvals without human review
- Workflow automation beyond triage/recommendation
- Hybrid on-premise or multi-cloud hosting outside AWS
- Citizen-facing application portal (assumed upstream)

---

## 3. High-Level Solution Description

### 3.1 System Summary

Event-driven platform: **intake → AI processing → structured outputs → human review → decision publication**.

- Supports organisation-specific policies and reusable AI configurations
- All state in Aurora (authoritative) and DynamoDB (runtime workflow)
- No policy logic in code; policies loaded from versioned configuration

### 3.2 Differentiators

- **Caseworker-centric** – AI assists, does not replace
- **Multi-agent AI orchestration** (Bedrock Agent Core)
- **Configurable policies and eligibility rules** (YAML/JSON, versioned)
- **Scalable AWS-native deployment**
- **Structured, audit-ready decision support**
- **Decision retrieval API** for upstream systems

---

## 4. Requirements

### 4.1 Business Requirements

- Reduce manual effort
- Improve consistency and traceability
- Accelerate processing while retaining human oversight
- Configurable eligibility rules
- Secure handling of citizen data

### 4.2 Functional Requirements

| Area | Requirements |
|------|--------------|
| **Caseworker portal** | Secure login (SSO), view AI outputs, audit logs, policy management, notifications, settings |
| **Backend services** | Document ingestion, AI workflow orchestration, API endpoints (intake, complete, decision, caseworker APIs) |
| **Database** | Aurora PostgreSQL (relational, authoritative), DynamoDB (runtime state), S3 (documents) |
| **AI agents** | Classification, extraction, eligibility evaluation, structured summaries (no auto-approval) |

### 4.3 Non-Functional Requirements

- **Availability:** High availability; Multi-AZ where applicable
- **Scalability:** Serverless, event-driven; horizontal scaling
- **Security:** Encryption at rest and in transit, RBAC, no PII in logs
- **Auditability:** Immutable audit logs, policy version pinned per case
- **Latency:** Low latency for portal APIs; async for AI pipeline

### 4.4 Assumptions

- All applications submitted via existing upstream channels
- Caseworkers trained on the portal
- Policies provided in compatible format (YAML/JSON)
- Upstream system can poll GET /applications/{caseId}/decision for decision retrieval

### 4.5 Dependencies

- AWS services (Bedrock, Aurora, DynamoDB, S3, API Gateway, EventBridge, SQS, Cognito, Amplify)
- Upstream document/case management system integration

### 4.6 Risks

| Risk | Mitigation |
|------|------------|
| Misconfigured policies or incomplete documents impacting AI accuracy | Validation on policy upload; human review for low-confidence cases |
| AWS service outages | Multi-AZ, retries, DLQs, stateless compute |
| Security/compliance breaches | IAM least privilege, KMS, CloudTrail, no PII in logs |
| Resistance to adoption by caseworkers | Clear UI, AI as assist not replace, training |

---

## 5. Architecture Overview

The platform is built as a **cloud-native, event-driven, serverless** architecture on AWS.

### 5.1 Conceptual Workflow (Summary)

| Step | Name | Description |
|------|------|-------------|
| 1 | Document ingestion | API-driven intake (init → S3 upload → complete) → EventBridge |
| 2 | Event-driven trigger | EventBridge emits CASE_INTAKE_VALIDATED → AI orchestration |
| 3 | Policy-governed AI orchestration | Agent Core: validation → extraction → policy evaluation → case summary |
| 4 | Caseworker review & human decision | Portal: view case, AI output, then Approve / Decline / Escalate |
| 5 | Policy customisation & runtime usage | YAML/JSON policies → S3 → validation → Aurora → agents read from DB |
| 6 | Decision publication / retrieval | GET /applications/{caseId}/decision → GetDecisionLambda → Aurora → JSON |

![Conceptual workflow](images/workflow.png)

![Conceptual workflow (detail)](images/workflow-2.png)

![Conceptual workflow (detail)](images/government-triage-workflow-agentcore.png)


---

### 5.2 Step 1 – Document Ingestion

#### 5.2.1 Application initialisation – `POST /applications/init`

**Input (example):**

```json
{
  "caseId": "HF-2025-000123",
  "orgId": "councilA",
  "caseType": "hardship-fund",
  "submissionType": "NEW",
  "applicant": {
    "firstName": "John",
    "lastName": "Smith",
    "dob": "1985-03-22",
    "nationalInsurance": "AB123456C"
  },
  "documents-to-upload": [
    { "fileName": "passport.pdf", "documentType": "passport", "version": 1 },
    { "fileName": "payslip_jan.pdf", "documentType": "payslip", "month": "January-2026", "version": 1 }
  ],
  "submittedAt": "2026-02-06T15:00:00Z"
}
```

**Lambda: ApplicationInitLambda**

1. Schema validation (required fields, submission type)
2. Policy resolution from Aurora (which policy version applies)
3. Semantic validation (org exists, case type allowed)
4. Case creation (generate caseId if missing, lock policy version)
5. State initialisation: Aurora + DynamoDB
6. Presigned S3 URLs for documents

**Database writes:**

- **DynamoDB** `case_runtime_state`: caseId, orgId, caseType, policyVersion, applicationVersion, status, timestamps
- **Aurora** `cases`: case_id, org_id, case_type, policy_version, submission_type, applicant_reference, timestamps

**Output to upstream:**

```json
{
  "caseId": "HF-2025-000123",
  "policyVersion": 3,
  "requiredDocuments": [...],
  "uploadUrls": { ... }
}
```

#### 5.2.2 Document upload (direct to S3)

- Upstream uploads documents using the provided presigned URLs
- **Bucket naming:** `<org-id>-<case-type>-applicant-intake-s3-<env>`
- **Folder structure:** `s3://.../<org-id>/<case-type>/<case-id>/documents/<document-type>-<timestamp>-v<version>.<ext>`
- Manifest.json must be included where required

#### 5.2.3 Intake finalisation – `POST /applications/complete`

**Input:**

```json
{
  "caseId": "HF-2025-000123"
}
```

**Lambda: ApplicationFinalizeLambda**

1. Load case context (policy, required documents)
2. Inspect S3 uploads (presence, formats, version limits)
3. Technical sanity checks (size, MIME type)
4. Persist metadata to Aurora
5. Update DynamoDB case state → **INTAKE_VALIDATED**
6. Emit EventBridge event for AI orchestration

**Database updates:**

- DynamoDB: status → INTAKE_VALIDATED
- Aurora: `case_documents` (case_id, document_type, s3_key, version, timestamp)

---

### 5.3 Step 2 – Event-Driven Triggering

- **Trigger:** Successful intake validation → EventBridge emits **CASE_INTAKE_VALIDATED**
- **Event structure:**

```json
{
  "source": "case.intake",
  "detail-type": "CASE_INTAKE_VALIDATED",
  "detail": { ... }
}
```

- EventBridge rule filters by detail-type → starts AI orchestration (Bedrock AgentCore)
- Loose coupling, fan-out, observability

---

### 5.4 Step 3 – Policy-Governed AI Orchestration (Agent Core)

**AgentCore-first note:** Orchestration MUST be executed by **AWS Bedrock AgentCore** (optionally via the **Strands Agents SDK** or equivalent AgentCore runtime). Deterministic policy logic remains outside the agent in governed tool contracts. Step Functions (not used for AI orchestration) must not be used for AI sequencing.

- **Agent Core** orchestrates deterministic, multi-stage, policy-governed workflows
- Stateless agents; databases hold persistent state
- **Human decision required** – no automated approvals
- Uses **Amazon Bedrock Agent Core**, **Amazon Textract** (document introspection), and Bedrock reasoning (classification)

| Agent | Purpose | Inputs | Outputs | Status |
|-------|---------|--------|---------|--------|
| 1. Document validation | Validate technical correctness | S3 docs, policy | Valid/invalid, metadata | DOCS_TECHNICALLY_VALIDATED |
| 2. Data extraction | Extract structured fields | Validated docs, policy | JSON fields, confidence | DATA_EXTRACTED |
| 3. Policy evaluation | Check rules, eligibility | Extracted data, policy | Rule results, explanations | POLICY_VALIDATED |
| 4. Case summary & recommendation | Synthesise case summary | All prior outputs | Human-readable summary + recommendation | READY_FOR_CASEWORKER_REVIEW |

**Failure handling:**

- Each agent handles failures explicitly; stops or flags for review
- Partial outputs preserved
- Workflow resumable (checkpoints in DynamoDB)

---

### 5.5 Step 4 – Caseworker Review & Human Decision

- Caseworker logs in via portal (Cognito SSO), sees list of cases assigned
- Selects case → sees case information, documents, AI summary and recommendation
- **Actions:** Approve, Decline, Pending, Escalate, or send email to citizen (e.g. request more info)
- AI outputs are **read-only**; caseworker takes final action
- **Entry criteria:** status READY_FOR_CASEWORKER_REVIEW

| Sub-area | Description |
|----------|-------------|
| Authentication & authorisation | Cognito SSO, IAM, Aurora role mapping |
| Case listing & notification | Read-only display of relevant cases, status, AI completion |
| Individual case view | Read-only: applicant data, documents, extracted fields, policy evaluation, AI summary |
| Human decision stage | Actions: Approve, Decline, Pending, Escalate → writes Aurora + DynamoDB (decision, justification, timestamp) |
| Escalation workflow | Senior review if ambiguous; original decision locked, history preserved |
| Notification & downstream | EventBridge / SES / optional downstream notifications |
| Audit & immutability | AI outputs frozen, policy version preserved, all actions timestamped and traceable |

---

### 5.6 Step 5 – Policy Customisation & Runtime Usage

**Policy definition & storage**

- YAML-based (or JSON) human-friendly files
- Uploaded via **Admin UI** in caseworker portal → **S3** (e.g. `policy-definition-bucket-prd`)
- Versioned; old versions immutable

**Processing flow**

1. Upload → S3 → ObjectCreated → EventBridge → Policy Lambda
2. Schema & semantic validation (structure, fairness, no contradictions)
3. Normalisation → machine-readable canonical form
4. Persist in Aurora → runtime policy execution
5. Activation → policy available for intake workflows

**Policy use at runtime**

- Intake Lambda resolves **active policy version**
- Agents **read from Aurora**; never directly from YAML or S3 at execution time
- Caseworker UI shows version, rule outcomes, fairness guarantees

**Aurora tables:** `policies`, `policy_documents`, `policy_extraction_fields`, `policy_rules`, `policy_fairness_constraints`

*(Sample YAML policy structure is in section 6.4 / Appendix if needed.)*

---

### 5.7 Step 6 – Decision Publication / Retrieval

The **Citizen Portal / Upstream system** polls for the decision.

**Endpoint:** `GET /applications/{caseId}/decision`

**Lambda: GetDecisionLambda**

- Validate request
- Authorise org access
- Retrieve decision from Aurora (`case_decisions`)
- Return structured response

**Example response:**

```json
{
  "caseId": "HF-2025-000123",
  "decision": "APPROVED",
  "reason": "Household income below hardship threshold",
  "confidence": 0.87,
  "policyVersion": 3,
  "decidedAt": "2026-02-10T14:32:00Z"
}
```

---

### 5.8 Workflow Consistency & Recovery

**Design principles**

1. Stateless compute
2. Externalised state (DynamoDB + Aurora)
3. Idempotent steps
4. Resume from last successful checkpoint
5. All failures observable and auditable

**Recovery services**

- SQS + DLQs (retry isolation)
- CloudWatch (monitoring/alerts)
- Lambda (stateless, retry-safe)
- DynamoDB (runtime state)
- Aurora (authoritative metadata)

**Failure handling**

- Partial processing tracked; retry from checkpoint
- DLQs capture permanent failures; manual replay possible
- Multi-AZ Aurora, Lambda auto-retry, Textract retries for resilience

---


---

### 5.9 Ops Observability & Operational Recovery

The system MUST support operational detection, diagnosis, and safe recovery of cases that stop progressing due to either business validation blocks or infrastructure/runtime failures.

#### 5.9.1 Definitions: STUCK, FAILED (Infrastructure), BLOCKED (Business)

A case is **STUCK** when:
- The case is in a **non-terminal** status (e.g. `INTAKE_VALIDATED`, `DOCS_TECHNICALLY_VALIDATED`, `DATA_EXTRACTED`, `POLICY_VALIDATED`, `SUMMARY_READY`), AND
- The case has not advanced to the next valid status within the **stage SLA threshold**, measured using `updated_at` (or an equivalent “last progressed” timestamp) in Aurora.

A case is **FAILED (Infrastructure)** when:
- A tool invocation fails due to infrastructure/runtime conditions (timeouts, throttling, dependency outage, service errors), AND
- The case does not progress beyond the current stage after the configured retry strategy.

A case is **BLOCKED (Business)** when:
- A mandatory document/field/policy requirement fails, and progression is intentionally halted until action is taken (e.g. missing mandatory document, required field extraction failure).

#### 5.9.2 Stage SLA Thresholds

The system MUST define and publish stage-level SLA thresholds (time-to-progress) for:
- Validation
- Data Extraction
- Policy Evaluation
- Recommendation/Summary
- Mark Ready / readiness event emission

These thresholds MUST be used for stuck-case detection and operational alerting.

#### 5.9.3 Tool-Level Failure Recording (Per Case)

Each deterministic tool invocation MUST record, in a durable and queryable form (e.g. Aurora execution/audit tables and/or structured logs linked by correlation ID), at minimum:

- `caseId`
- `tool_name`
- `tool_attempt_number` (or equivalent retry count)
- `tool_started_at`
- `tool_ended_at`
- `tool_outcome` (`SUCCESS`, `FAILURE`, `RETRYING`, `BLOCKED`)
- `last_error_code` (when failure occurs; standardized categories recommended)
- `last_error_time`
- `last_error_summary` (short, non-PII)

The system MUST ensure Ops can identify, for any stuck/failed case: the last tool executed, the most recent error code/time, and whether the failure is infrastructure vs business-blocking.

#### 5.9.4 Ops Reporting: Stuck Case View

The system MUST provide an operational view (query/report/dashboard) listing STUCK cases, including:
- current `status` (Aurora) and time-in-status
- last tool executed + last error code/time
- runtime lock state (if applicable)

The view MUST support filtering by status/stage, organisation/orgId (where relevant), and time window.

#### 5.9.5 Alerting

The system MUST support alerting for:
- Tool error-rate spikes (per tool)
- Tool timeout spikes and duration anomalies
- AgentCore orchestration/session failures
- Retry exhaustion / repeated failures for the same case+tool
- DLQ growth (if queues/DLQs are used)
- EventBridge delivery/dispatch failures for critical events (e.g. readiness event)

Alerts MUST include correlation identifiers and case identifiers sufficient to locate impacted cases.

#### 5.9.6 Recovery and Replay

The system MUST support safe recovery by ensuring state transitions are idempotent, tools validate current status before applying transitions, and repeated invocations do not duplicate side effects (writes/events).

The system MAY support one or more manual replay mechanisms for Ops (e.g. re-emitting the trigger event for a case, invoking a resume endpoint, or re-running tool invocations). Any replay MUST resume from the last durable checkpoint based on Aurora status and MUST remain idempotent.


## 6. Application Architecture

### 6.1 Architecture Summary

Modular, cloud-native, event-driven design on AWS.

### 6.2 Frontend Architecture (Caseworker Portal)

- **Stack:** React 18 + TypeScript, Next.js 14 (SSG/ISR)
- **Deployment:** AWS Amplify (hosting, APIs, auth, backend environments)
- **Access:** Amazon Cognito (SSO, role-based access for Caseworkers, Managers, Administrators)

**Main screens**

| # | Screen | Description |
|---|--------|-------------|
| 1 | Login | Secure SSO; role-based access |
| 2 | Homepage | Totals, case status, priority distribution |
| 3 | Case management | List of all cases |
| 4 | Individual case | Applicant details, AI analysis, documents, notes, risk assessment; Approve / Decline / Escalate; optional “send status update email” |
| 5 | Notifications | Case notifications; toggles in settings for notification types |
| 6 | Settings | Profile, notifications, display, FAQ, support, AI Guide |
| 7 | Escalated cases | Managers only |
| 8 | User management | Administrators only – create, delete, update users |
| 9 | Policy management | Administrators – manage policies used by AI agent |

**Login page**

![Login page](images/loginpage.png)

**Homepage**

![Homepage](images/homepage.png)

**The Case Management Screen**

![Case management screen](images/case-management-screen.png)

**Individual case screen**

![Individual case](images/individual-case.png)
![Individual case 2](images/individual-case2.png)
![Individual case 3](images/individualscreen-3.png)
![Individual case 4](images/individual-case4.png)

**Notifications**

![Notifications](images/notification.png)

**Settings**

![Settings](images/settings.png)

**Escalated Cases**

*(Add screenshot when available.)*

**User Management**

![User management](images/user-management.png)

**Policy Management**

![Policy management 1](images/policymanagement1.png)
![Policy management 2](images/policymanagement2.png)
![Policy management 3](images/policymanagement3.png)

### 6.3 Backend Architecture

**Categories**

1. **Application intake, Agent Core / AI processing, decision retrieval** – as in section 5
2. **APIs that power the Caseworker Portal**

**Caseworker flows**

| Flow | Description |
|------|-------------|
| Login | SSO via Cognito (e.g. Azure AD / Okta); JWT with role claims |
| Dashboard | API Gateway → Lambda → Aurora (cases by assigned_to), stats, DynamoDB notifications |
| Case list | JWT → user_id → Aurora: cases by status (ASSIGNED, UNASSIGNED), pagination |
| Case details | Lambda: case metadata (Aurora), AI analysis (Aurora as source of truth; DynamoDB optional cache/runtime pointers), documents (S3 presigned URLs) |
| Decision (Approve / Decline / Escalate) | Bedrock drafts email if used; Lambda updates Aurora status; audit log; SES; EventBridge; DynamoDB notifications |
| Notifications & profile | Notifications from DynamoDB; profile/image in S3 |

**Admin flows**

- Login: same Cognito SSO, admin RBAC
- Dashboard: case metrics, user stats, system health (CloudWatch)
- User management: create, activate, deactivate, soft delete; Aurora + Cognito; audit log
- Policy configuration: upload JSON/YAML, validate, version in Aurora

**Manager flows**

- Dashboard & case access; escalation visibility
- Escalation view: escalations by status, joined with case details
- Escalation review: caseworker notes, case history, AI risk insights
- Resolution: manager decision → Aurora → audit → EventBridge → notifications

### 6.4 Database Architecture

**Authoritative vs runtime state:** Aurora PostgreSQL is the **system of record** for policies, cases, extracted fields, rule outcomes, agent outputs, recommendations, decisions, and audit logs. DynamoDB stores **runtime orchestration state only** (e.g., current status, lock owner/expiry, attempt counts, last tool executed). UI-visible “AI analysis” must be derivable from Aurora records; DynamoDB may cache pointers or latest snapshots but must not be the sole source of truth.


![Database workflow](images/database-workflow.png)

**Dual-database model**

- **Amazon Aurora PostgreSQL** – system of record: policies, cases, document metadata, extracted data, rule evaluations, agent outputs, human decisions, audit trail
- **Amazon DynamoDB** – runtime workflow state only: stage, status, retry, locks (no policy or decision data)

**Table groups** (detailed in section 7)

1. Organisation & case setup (Aurora)
2. Policy configuration – versioned (Aurora)
3. Case & application (Aurora)
4. Agent processing (Aurora)
5. Human decision & audit (Aurora)
6. Runtime workflow (DynamoDB)

**Aurora requirements**

- IAM-based access only (no password)
- Encrypted in transit
- Automated backups, PITR
- Error/general/slow query logs to CloudWatch
- Enhanced monitoring and Performance Insights
- CloudTrail integration
- Deletion protection enabled

**Retention**

- EventBridge rule triggers when Case & Application data is **5 years old** → Lambda deletes that data (as per policy).

---

## 7. Database Tables

### 7.1 Aurora PostgreSQL

**Common requirements for all tables:** KMS encryption, PITR, CloudTrail data events, tags (Environment, Owner=PublicSectorCaseTriage, Purpose=CaseManagement, Compliance=ISO27001,FedRAMP), **no PII in logs**.

#### 1. Organisation & case setup

| Table | Primary key | Required attributes |
|-------|-------------|---------------------|
| **organisations** | organisation_id (String) | name, status (active/suspended), created_at |
| **case_types** | case_type_id (String) | organisation_id (FK), name, status |

#### 2. Policy configuration (versioned)

| Table | Primary key | Required attributes |
|-------|-------------|---------------------|
| **policies** | policy_id (String) | organisation_id (FK), case_type_id (FK), version, status (draft/active/retired), effective_from, created_at |
| **policy_documents** | policy_document_id (String) | policy_id (FK), document_type, mandatory, accepted_formats, max_versions, lookback_period_months (optional) |
| **policy_extraction_fields** | extraction_field_id (String) | policy_id (FK), document_type, field_name, datatype |
| **policy_rules** | rule_id (String) | policy_id (FK), field_name, operator (<, <=, >, >=, =), comparison_value, description |
| **policy_fairness_constraints** | fairness_id (String) | policy_id (FK), prohibited_attribute, enforcement_level (strict/advisory) |

#### 3. Case & application

| Table | Primary key | Required attributes |
|-------|-------------|---------------------|
| **cases** | case_id (String) | organisation_id (FK), case_type_id (FK), policy_id (FK), policy_version, submission_type, applicant_reference, status, created_at, intake_completed_at |
| **case_documents** | case_document_id (String) | case_id (FK), document_type, s3_object_path, version, upload_timestamp |
| **extracted_case_data** | extracted_data_id (String) | case_id (FK), field_name, value, confidence_score |

#### 4. Agent processing

| Table | Primary key | Required attributes |
|-------|-------------|---------------------|
| **agent_executions** | agent_execution_id (String) | case_id (FK), agent_name, status (success/failed), executed_at |
| **rule_evaluations** | evaluation_id (String) | case_id (FK), rule_id (FK), result (pass/fail/conditional), explanation |

#### 5. Human decision & audit

| Table | Primary key | Required attributes |
|-------|-------------|---------------------|
| **case_decisions** | decision_id (String) | case_id (FK), decision, decided_by, justification, decided_at |
| **audit_logs** | audit_id (String) | entity_type, entity_id, action, performed_by, timestamp — **immutable** |

### 7.2 DynamoDB

| Table | Primary key | Required attributes |
|-------|-------------|---------------------|
| **case_runtime_state** | case_id (String) | orgId, caseType, policyVersion, applicationVersion, current_stage (intake/validation/agent/review), status, lock_owner, updated_at, created_at |

**Requirements:** KMS encryption, PITR, CloudTrail data events, same tags; **no PII stored or logged**; runtime only (no historical records).

---

## 8. S3 Data Handling & Security

### 8.1 Data handling rules

**Raw documents (applicant uploads)**

- PDF, DOCX, ID proofs, bank statements, utility bills, letters
- Stored in raw-documents bucket (Standard)

**Glacier (after case closure)**

- Closed case raw documents
- Final case packs / decision bundles (PDF summaries, decision letters)

**Lifecycle**

- Day 0 → S3 Standard  
- Day 30 → S3 Standard-IA  
- Case closed +30–90 days → Glacier Flexible Retrieval  
- Retention 5 years → then delete

### 8.2 S3 security requirements

- Block all public access
- Encryption at rest (AWS KMS)
- Deny non-HTTPS (`aws:SecureTransport = false`)
- Disable ACLs
- IAM restricted to: **AI-Agent-Role** (read-only raw), **Caseworker-Review-Role** (read-only raw), **Admin-Role** (full)
- Versioning enabled
- Object lock (WORM) in Governance mode
- Server access logging to dedicated logging bucket
- S3 VPC endpoint where applicable

---

## 9. Infrastructure Architecture
![AWS architecture](images/architecture.png)

![AWS architecture](images/aws-architecture.png)

![AWS architecture (detail)](images/complete-architecture.png)

- **IAC** – Terraform; CI/CD for all components needed
- **VPC** – segmented; private subnets for application and data; NAT for outbound
- **Public access** – Frontend and API via CloudFront and API Gateway; WAF and Shield
- **DNS & TLS** – Route 53, ACM (custom domains, HTTPS)
- **Caseworker portal** – Amplify + CloudFront
- **Auth** – Cognito with enterprise IdP (e.g. Azure AD, Okta), SSO, RBAC
- **Compute** – Lambda (intake, validation, decision, portal APIs); Bedrock AgentCore; EventBridge; SQS + DLQs
- **AI** – Bedrock Agent Core, Textract
- **Data** – Aurora PostgreSQL (Multi-AZ), DynamoDB (encryption, PITR), S3 (KMS, versioning, object lock, lifecycle)
- **Security** – IAM least privilege, KMS, TLS, Secrets Manager; VPC endpoints for S3, DynamoDB, Bedrock, etc.
- **Observability** – CloudWatch, CloudTrail, Aurora Performance Insights
- **HA & DR** – Multi-AZ, automated backups, PITR, versioned S3

---

## 10. Naming Conventions

| Placeholder | Values / format |
|-------------|-----------------|
| `<env>` | `dev` \| `tst` \| `prd` |
| `<org-id>` | Short, lowercase, hyphenated (e.g. `council-a`) |
| `<case-type>` | Short slug (e.g. `hardship-fund`, `housing-benefit`) |
| **App prefix** | **PS-FastStart** (or `PS` or `ct` in code) |

**Examples**

- S3 bucket: `<org-id>-<case-type>-applicant-intake-s3-<env>`
- Lambda: `FastStart-<env>-application-init`, `FastStart-<env>-get-decision`, etc.

---

*End of FastStart Technical Specification*
