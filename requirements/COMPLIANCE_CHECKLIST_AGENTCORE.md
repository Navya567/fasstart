# FastStart AgentCore-first Implementation Compliance Checklist

**Source of truth:** `requirements/agentcore-requirements-full-v1.2.md` (canonical)  
**Version:** 1.2  
**Purpose:** Implementation teams use this checklist to verify parity with the AgentCore-first specification. Every item traces to a specific section in v1.2.

---

## Confirmation: Source of Truth

**v1.2 is the single source of truth** for this checklist. All requirements, constraints, and section references refer to `requirements/agentcore-requirements-full-v1.2.md`. No Step Functions for AI orchestration; AI sequencing is Bedrock AgentCore-driven. Tools are deterministic contracts (hosting may be Lambda or other service); sequencing is performed by AgentCore.

---

## 1. Frontend Checklist (Screen-by-Screen Parity)

Implementation MUST provide the following screens and capabilities. Reference: **§6.2 Frontend Architecture (Caseworker Portal)**.

| # | Screen | Parity requirement | v1.2 ref |
|---|--------|--------------------|----------|
| 1 | **Login** | Secure SSO; role-based access. | §6.2 Main screens #1 |
| 2 | **Homepage** | Totals, case status, priority distribution. | §6.2 Main screens #2 |
| 3 | **Case management** | List of all cases. | §6.2 Main screens #3 |
| 4 | **Individual case** | Applicant details, AI analysis, documents, notes, risk assessment; actions: Approve / Decline / Escalate; optional “send status update email”. | §6.2 Main screens #4 |
| 5 | **Notifications** | Case notifications; toggles in settings for notification types. | §6.2 Main screens #5 |
| 6 | **Settings** | Profile, notifications, display, FAQ, support, AI Guide. | §6.2 Main screens #6 |
| 7 | **Escalated cases** | Managers only. | §6.2 Main screens #7 |
| 8 | **User management** | Administrators only – create, delete, update users. | §6.2 Main screens #8 |
| 9 | **Policy management** | Administrators – manage policies used by AI agent. | §6.2 Main screens #9 |

**Stack & deployment:** React 18 + TypeScript, Next.js 14 (SSG/ISR); AWS Amplify (hosting, APIs, auth); Amazon Cognito (SSO, role-based access for Caseworkers, Managers, Administrators). **§6.2** (Stack, Deployment, Access).

### UI ↔ Backend Parity

Every user-visible field (case status, AI analysis, notes, risk assessment, documents, actions) MUST be backed by an API response field and a durable persisted record (Aurora as source of truth). DynamoDB may cache runtime pointers/snapshots but must not be the sole source of truth for UI-visible AI analysis. No UI-only derived state is allowed unless explicitly documented. Ref: §6.2, §6.3, §6.4

---

## 2. Backend API Checklist (Endpoint-by-Endpoint Parity)

Implementation MUST provide the following APIs and behaviours. References: **§5.2, §5.3, §5.7, §6.3**.

### 2.1 Intake APIs

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **POST /applications/init** | Accept input with caseId, orgId, caseType, submissionType, applicant, documents-to-upload, submittedAt (or equivalent schema). | §5.2.1 |
| | ApplicationInitLambda: schema validation, policy resolution from Aurora, semantic validation (org, case type), case creation (caseId if missing, lock policy version), state init Aurora + DynamoDB, presigned S3 URLs. | §5.2.1 |
| | Database writes: DynamoDB `case_runtime_state` (caseId, orgId, caseType, policyVersion, applicationVersion, status, timestamps); Aurora `cases` (case_id, org_id, case_type, policy_version, submission_type, applicant_reference, timestamps). | §5.2.1 |
| | Response: caseId, policyVersion, requiredDocuments, uploadUrls. | §5.2.1 |
| **Document upload** | Upstream uses presigned URLs; bucket naming `<org-id>-<case-type>-applicant-intake-s3-<env>`; folder structure `s3://.../<org-id>/<case-type>/<case-id>/documents/<document-type>-<timestamp>-v<version>.<ext>`; Manifest.json where required. | §5.2.2 |
| **POST /applications/complete** | Input: caseId (or equivalent). | §5.2.3 |
| | ApplicationFinalizeLambda: load case context, inspect S3 uploads, technical sanity checks, persist metadata to Aurora, set DynamoDB status → INTAKE_VALIDATED, emit EventBridge event for AI orchestration. | §5.2.3 |
| | Database updates: DynamoDB status → INTAKE_VALIDATED; Aurora `case_documents` (case_id, document_type, s3_key, version, timestamp). | §5.2.3 |

### 2.2 Decision Publication

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **GET /applications/{caseId}/decision** | GetDecisionLambda: validate request, authorise org access, retrieve from Aurora `case_decisions`, return structured response. | §5.7 |
| | Response shape: caseId, decision, reason, confidence, policyVersion, decidedAt (or equivalent). | §5.7 |

### 2.3 Caseworker Portal APIs

| Flow | Requirement | v1.2 ref |
|------|-------------|----------|
| **Login** | SSO via Cognito (e.g. Azure AD / Okta); JWT with role claims. | §6.3 Caseworker flows |
| **Dashboard** | API Gateway → Lambda → Aurora (cases by assigned_to), stats, DynamoDB notifications. | §6.3 Caseworker flows |
| **Case list** | JWT → user_id → Aurora: cases by status (ASSIGNED, UNASSIGNED), pagination. | §6.3 Caseworker flows |
| **Case details** | Lambda: case metadata (Aurora), AI analysis (Aurora as source of truth; DynamoDB optional cache/runtime pointers), documents (S3 presigned URLs). | §6.3 Caseworker flows |
| **Decision (Approve / Decline / Escalate)** | Bedrock drafts email if used; Lambda updates Aurora status; audit log; SES; EventBridge; DynamoDB notifications. | §6.3 Caseworker flows |
| **Notifications & profile** | Notifications from DynamoDB; profile/image in S3. | §6.3 Caseworker flows |

### 2.4 Admin & Manager APIs

| Flow | Requirement | v1.2 ref |
|------|-------------|----------|
| **Admin** | Login: same Cognito SSO, admin RBAC; Dashboard: case metrics, user stats, system health (CloudWatch); User management: create, activate, deactivate, soft delete; Aurora + Cognito; audit log; Policy configuration: upload JSON/YAML, validate, version in Aurora. | §6.3 Admin flows |
| **Manager** | Dashboard & case access; escalation visibility; Escalation view: escalations by status, joined with case details; Escalation review: caseworker notes, case history, AI risk insights; Resolution: manager decision → Aurora → audit → EventBridge → notifications. | §6.3 Manager flows |

---

## 3. Data Model Checklist (Aurora vs DynamoDB Responsibilities)

Implementation MUST conform to the following data responsibilities and table definitions. References: **§6.4, §7**.

### 3.1 Authoritative vs Runtime Split

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Aurora** | System of record for: policies, cases, extracted fields, rule outcomes, agent outputs, recommendations, decisions, audit logs. | §6.4 |
| **DynamoDB** | Runtime orchestration state only: e.g. current status, lock owner/expiry, attempt counts, last tool executed. No policy or decision data as source of truth. | §6.4 |
| **UI-visible AI analysis** | Must be derivable from Aurora records; DynamoDB may cache pointers or snapshots but must not be the sole source of truth. | §6.4 |

### 3.2 Aurora PostgreSQL Tables

| Group | Tables & keys/attributes | v1.2 ref |
|-------|-------------------------|----------|
| **Organisation & case setup** | organisations (PK organisation_id; name, status, created_at); case_types (PK case_type_id; organisation_id FK, name, status). | §7.1 (1) |
| **Policy configuration** | policies; policy_documents; policy_extraction_fields; policy_rules; policy_fairness_constraints (with PKs and required attributes as in v1.2). | §7.1 (2) |
| **Case & application** | cases; case_documents; extracted_case_data (with PKs and required attributes as in v1.2). | §7.1 (3) |
| **Agent processing** | agent_executions; rule_evaluations (with PKs and required attributes as in v1.2). | §7.1 (4) |
| **Human decision & audit** | case_decisions; audit_logs (immutable) (with PKs and required attributes as in v1.2). | §7.1 (5) |

**Common Aurora requirements:** KMS encryption, PITR, CloudTrail data events, tags (Environment, Owner=PublicSectorCaseTriage, Purpose=CaseManagement, Compliance=ISO27001,FedRAMP), **no PII in logs**. **§7.1** intro.

### 3.3 DynamoDB

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **case_runtime_state** | PK case_id; attributes: orgId, caseType, policyVersion, applicationVersion, current_stage (intake/validation/agent/review), status, lock_owner, updated_at, created_at. | §7.2 |
| **DynamoDB requirements** | KMS encryption, PITR, CloudTrail data events, same tags; no PII stored or logged; runtime only (no historical records). | §7.2 |

### 3.4 Retention

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Case & application data** | EventBridge rule triggers when data is 5 years old → Lambda deletes that data (as per policy). | §6.4 Retention |

### 3.5 Aurora Operational Requirements

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Access & security** | IAM-based access only (no password); encrypted in transit; automated backups, PITR; error/general/slow query logs to CloudWatch; Enhanced monitoring and Performance Insights; CloudTrail integration; deletion protection enabled. | §6.4 Aurora requirements |

---

## 4. AI Processing Checklist (AgentCore Sequencing + Tool Constraints + State Transitions)

Implementation MUST use Bedrock AgentCore for AI sequencing and MUST NOT use Step Functions for AI orchestration. References: **§5.3, §5.4**.

### 4.1 Orchestration Model

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Orchestrator** | Orchestration MUST be executed by AWS Bedrock AgentCore (optionally via Strands Agents SDK or equivalent AgentCore runtime). | §5.4 AgentCore-first note |
| **No Step Functions** | Step Functions must not be used for AI sequencing. | §5.4 AgentCore-first note |
| **Policy logic** | Deterministic policy logic remains outside the agent in governed tool contracts. | §5.4 AgentCore-first note |
| **Stateless agent** | Agent Core is stateless; databases hold persistent state. | §5.4 |
| **Human decision** | Human decision required – no automated approvals. | §5.4 |

### 4.2 Event-Driven Trigger

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Trigger** | Successful intake validation → EventBridge emits CASE_INTAKE_VALIDATED. | §5.3 |
| **Event structure** | source, detail-type (CASE_INTAKE_VALIDATED), detail. | §5.3 |
| **Consumer** | EventBridge rule filters by detail-type → starts AI orchestration (Bedrock AgentCore). | §5.3 |

### 4.3 Tool Contracts (Deterministic Execution)

| Tool | Purpose | Inputs | Outputs | Status after success | v1.2 ref |
|------|---------|--------|---------|----------------------|----------|
| **1. Document validation** | Validate technical correctness | S3 docs, policy | Valid/invalid, metadata | DOCS_TECHNICALLY_VALIDATED | §5.4 table |
| **2. Data extraction** | Extract structured fields | Validated docs, policy | JSON fields, confidence | DATA_EXTRACTED | §5.4 table |
| **3. Policy evaluation** | Check rules, eligibility | Extracted data, policy | Rule results, explanations | POLICY_VALIDATED | §5.4 table |
| **4. Case summary & recommendation** | Synthesise case summary | All prior outputs | Human-readable summary + recommendation | SUMMARY_READY | §5.4 table |
| **5. Mark ready for review** | Release workflow lock, emit readiness event | Summary outputs, case state | Lock released, CASE_AI_READY_FOR_REVIEW emitted (idempotent) | READY_FOR_CASEWORKER_REVIEW | §5.4 table |

**Readiness responsibilities (tool #5):**
Tool #5 ("Mark ready for review") transitions a case from `SUMMARY_READY` to `READY_FOR_CASEWORKER_REVIEW` and MUST (a) release any workflow lock, and (b) emit `CASE_AI_READY_FOR_REVIEW` **idempotently**. This is an explicit, separately testable tool in the pipeline. Ref: §5.4, §5.8, §5.9.6

### 4.4 AI Services

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Services used** | Amazon Bedrock Agent Core, Amazon Textract (document introspection), Bedrock reasoning (classification). | §5.4 |

### 4.5 Failure Handling (AI Stage)

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Agent/tool failures** | Each agent/tool handles failures explicitly; stops or flags for review. | §5.4 Failure handling |
| **Partial outputs** | Partial outputs preserved. | §5.4 Failure handling |
| **Resumability** | Workflow resumable from last successful **Aurora status checkpoint**; DynamoDB provides runtime lock/attempt state only. | §5.8, §6.4, §5.9.6 |

---

## 5. Auditability Checklist (What Must Be Logged/Stored)

Implementation MUST provide the following for audit and traceability. References: **§4.3, §5.5, §5.9.3, §7**.

### 5.1 Non-Functional Auditability

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Audit logs** | Immutable audit logs. | §4.3 |
| **Policy version** | Policy version pinned per case. | §4.3 |

### 5.2 Caseworker Review & Decision Audit

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **AI outputs** | AI outputs frozen (at review time). | §5.5 Sub-area Audit & immutability |
| **Policy version** | Policy version preserved. | §5.5 Sub-area Audit & immutability |
| **Actions** | All actions timestamped and traceable. | §5.5 Sub-area Audit & immutability |

### 5.3 Tool-Level Failure Recording (Per Case)

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Per tool invocation** | Record in durable, queryable form (e.g. Aurora execution/audit tables and/or structured logs linked by correlation ID): caseId, tool_name, tool_attempt_number (or equivalent), tool_started_at, tool_ended_at, tool_outcome (SUCCESS, FAILURE, RETRYING, BLOCKED), last_error_code (when failure), last_error_time, last_error_summary (short, non-PII). | §5.9.3 |
| **Ops visibility** | Ops can identify for any stuck/failed case: last tool executed, most recent error code/time, whether failure is infrastructure vs business-blocking. | §5.9.3 |

### 5.4 Audit Storage

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **audit_logs** | Table immutable (entity_type, entity_id, action, performed_by, timestamp). | §7.1 (5) |

### 5.5 No PII in Logs

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Aurora/DynamoDB** | No PII in logs (common table requirements). | §7.1 intro, §7.2 |

---

## 6. Failure / Retry / Replay Checklist (Idempotency + Safe Retries + Optional Replay)

Implementation MUST support the following recovery and resilience behaviours. References: **§5.8, §5.9**.

### 6.1 Design Principles

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Stateless compute** | Compute is stateless. | §5.8 Design principles |
| **Externalised state** | State in DynamoDB + Aurora. | §5.8 Design principles |
| **Idempotent steps** | Steps are idempotent. | §5.8 Design principles |
| **Resume** | Resume from last successful checkpoint. | §5.8 Design principles |
| **Observability** | All failures observable and auditable. | §5.8 Design principles |

### 6.2 Recovery Services

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Retry isolation** | SQS + DLQs (retry isolation). | §5.8 Recovery services |
| **Monitoring** | CloudWatch (monitoring/alerts). | §5.8 Recovery services |
| **Compute** | Lambda (stateless, retry-safe). | §5.8 Recovery services |
| **State** | DynamoDB (runtime state), Aurora (authoritative metadata). | §5.8 Recovery services |

### 6.3 Failure Handling

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Partial processing** | Partial processing tracked; retry from checkpoint. | §5.8 Failure handling |
| **DLQs** | DLQs capture permanent failures; manual replay possible. | §5.8 Failure handling |
| **Resilience** | Multi-AZ Aurora, Lambda auto-retry, Textract retries for resilience. | §5.8 Failure handling |

### 6.4 Ops Definitions (STUCK, FAILED, BLOCKED)

| Term | Definition | v1.2 ref |
|------|------------|----------|
| **STUCK** | Case in non-terminal status and has not advanced within stage SLA threshold (using updated_at or equivalent in Aurora). | §5.9.1 |
| **FAILED (Infrastructure)** | Tool invocation fails due to infrastructure/runtime conditions and case does not progress after configured retry strategy. | §5.9.1 |
| **BLOCKED (Business)** | Mandatory document/field/policy requirement fails; progression intentionally halted until action taken. | §5.9.1 |

### 6.5 Stage SLA Thresholds

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Thresholds** | Define and publish stage-level SLA thresholds (time-to-progress) for: Validation, Data Extraction, Policy Evaluation, Recommendation/Summary, Mark Ready / readiness event emission. | §5.9.2 |
| **Use** | Use thresholds for stuck-case detection and operational alerting. | §5.9.2 |

### 6.6 Ops Stuck Case View

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **View** | Operational view (query/report/dashboard) listing STUCK cases: current status (Aurora), time-in-status, last tool executed, last error code/time, runtime lock state (if applicable). | §5.9.4 |
| **Filtering** | Filter by status/stage, organisation/orgId (where relevant), time window. | §5.9.4 |

### 6.7 Alerting

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Alert on** | Tool error-rate spikes (per tool); tool timeout spikes and duration anomalies; AgentCore orchestration/session failures; retry exhaustion / repeated failures for same case+tool; DLQ growth (if used); EventBridge delivery/dispatch failures for critical events (e.g. readiness event). | §5.9.5 |
| **Alert content** | Correlation identifiers and case identifiers sufficient to locate impacted cases. | §5.9.5 |

### 6.8 Recovery and Replay

| Item | Requirement | v1.2 ref |
|------|-------------|----------|
| **Safe recovery** | State transitions idempotent; tools validate current status before applying transitions; repeated invocations do not duplicate side effects (writes/events). | §5.9.6 |
| **Replay (optional)** | System MAY support one or more manual replay mechanisms (e.g. re-emitting trigger event, resume endpoint, re-running tool invocations). | §5.9.6 |
| **Replay rules** | Any replay MUST resume from last durable checkpoint based on Aurora status and MUST remain idempotent. | §5.9.6 |

---

## 7. S3 and Naming (Cross-Cutting)

### 7.1 S3 Data Handling & Security

References: **§8.1, §8.2**. Implementation MUST apply data handling rules (raw documents, Glacier after case closure, lifecycle, retention 5 years) and S3 security requirements (block public access, KMS, HTTPS only, ACLs disabled, IAM roles for AI-Agent-Role, Caseworker-Review-Role, Admin-Role, versioning, object lock, logging, VPC endpoint where applicable).

### 7.2 Naming Conventions

Reference: **§10**. Implementation SHOULD follow placeholders and examples (env, org-id, case-type, app prefix; S3 bucket and Lambda naming patterns).

---

## Ambiguities / Open Questions (Do Not Change Requirements)

The following are left for product/architecture decisions; the checklist does not add or change v1.2:

1. **EventBridge `detail` payload:** v1.2 specifies `detail` exists but not mandatory fields. If implementations need a minimum contract (e.g. caseId, orgId), that should be agreed and documented separately.
2. **Strands Agents SDK:** v1.2 states “optionally via Strands Agents SDK or equivalent.” Whether “equivalent” includes any Bedrock AgentCore-compatible runtime is an implementation choice.
3. **Manual replay mechanism:** v1.2 says the system MAY support replay and lists examples; which mechanism(s) to implement is a project decision.
4. **Stage SLA threshold values:** v1.2 requires that thresholds be defined and published but does not specify numeric values; those are operational/contract decisions.

---

## Appendix: Ops Runbook (Triage, Recovery, Alarms)

**Audience:** Operations engineers responsible for monitoring, triaging, and recovering stuck or failed cases.

---

### Quick Checklist

1. Confirm the case status in **Aurora** `cases.status` (single source of truth). (§6.4)
2. Check `updated_at` against the published **stage SLA threshold** for that status. (§5.9.1, §5.9.2)
3. Query **tool invocation records** (Aurora `agent_executions` / audit tables) for last tool, outcome, and error code. (§5.9.3)
4. Check **DynamoDB** `case_runtime_state` for lock_owner and attempt counts. (§7.2)
5. Search **CloudWatch** logs using the case's correlation ID for infrastructure errors. (§5.8)
6. Classify the failure: **STUCK**, **FAILED (Infrastructure)**, or **BLOCKED (Business)**. (§5.9.1)
7. If FAILED: verify retry strategy is exhausted, then check DLQ for the event. (§5.8, §5.9.5)
8. If BLOCKED: identify the missing document/field and route to the appropriate team or upstream. (§5.9.1)
9. Attempt **safe recovery** (resume from Aurora checkpoint, idempotent replay). (§5.9.6)
10. Record all triage actions in the **audit log** with caseId, correlationId, and timestamps. (§4.3, §7.1)

---

### A.1 Purpose and Scope

This runbook covers operational detection, diagnosis, and recovery of cases that stop progressing through the FastStart AI processing pipeline. It applies to the stages between `INTAKE_VALIDATED` and `READY_FOR_CASEWORKER_REVIEW` -- the window where Bedrock AgentCore orchestrates deterministic tool execution. (§5.4, §5.9)

---

### A.2 Key Concepts

- **Aurora PostgreSQL** is the **authoritative checkpoint and system of record** for case status, extracted data, rule outcomes, agent outputs, decisions, and audit logs. Resume decisions are always based on Aurora status. (§6.4, §5.8, §5.9.6)
- **DynamoDB** (`case_runtime_state`) holds **runtime state only**: current stage, lock_owner, lock_expiry, attempt counts, last tool executed. It is not a checkpoint for resume. (§6.4, §7.2)
- **Tools are idempotent, deterministic contracts.** AgentCore sequences them; policy logic lives in the tools, not in the agent. Repeated invocations must not duplicate side effects. (§5.4, §5.9.6)
- **No Step Functions** are used for AI sequencing; orchestration is Bedrock AgentCore-driven. (§5.4)

---

### A.3 Detecting Problems: STUCK / FAILED / BLOCKED

#### A.3.1 Definitions (§5.9.1)

| Category | Meaning |
|----------|---------|
| **STUCK** | Case is in a non-terminal status and has **not advanced within the stage SLA threshold** (measured by `updated_at` or equivalent in Aurora). |
| **FAILED (Infrastructure)** | A tool invocation failed due to timeouts, throttling, dependency outage, or service errors, and the case did not progress after the configured retry strategy. |
| **BLOCKED (Business)** | A mandatory document, field, or policy requirement was not met; progression is intentionally halted until action is taken. |

#### A.3.2 Stage SLA Thresholds (§5.9.2)

Published thresholds (time-to-progress) must exist for each stage:

| Stage | Status on entry |
|-------|-----------------|
| Validation | `INTAKE_VALIDATED` |
| Data Extraction | `DOCS_TECHNICALLY_VALIDATED` |
| Policy Evaluation | `DATA_EXTRACTED` |
| Recommendation/Summary | `POLICY_VALIDATED` |
| Mark Ready / readiness event | `SUMMARY_READY` |

A case is **STUCK** when `NOW() - updated_at > SLA threshold` for its current status.

#### A.3.3 Stuck Case View (§5.9.4)

Use the operational view (query, report, or dashboard) that lists stuck cases. The view must show:

- Current `status` (Aurora) and time-in-status
- Last tool executed + last error code/time
- Runtime lock state (DynamoDB `lock_owner`)

Filter by: status/stage, orgId, time window.

---

### A.4 Where to Find "What Failed"

| What to look up | Where | Key fields |
|-----------------|-------|------------|
| Authoritative case status | Aurora `cases` | `status`, `updated_at`, `policy_version` |
| Tool invocation history | Aurora `agent_executions` / audit tables | `tool_name`, `tool_outcome`, `tool_attempt_number`, `tool_started_at`, `tool_ended_at` |
| Last error details | Aurora tool invocation records | `last_error_code`, `last_error_time`, `last_error_summary` (non-PII) |
| Runtime lock and stage | DynamoDB `case_runtime_state` | `current_stage`, `lock_owner`, `updated_at` |
| Infrastructure logs | CloudWatch Logs | Filter by correlation ID / caseId |
| Unprocessed events | SQS DLQs | Message body contains caseId and event detail |
| EventBridge delivery | EventBridge metrics / CloudTrail | `CASE_INTAKE_VALIDATED`, `CASE_AI_READY_FOR_REVIEW` events |

References: §5.9.3, §7.1, §7.2, §5.8

---

### A.5 Standard Triage Steps

1. **Identify the case.** Obtain `caseId` from the alert or stuck-case view.
2. **Read Aurora status.** Query `cases` for current `status` and `updated_at`. Confirm whether it exceeds the stage SLA.
3. **Pull tool invocation records.** Query Aurora `agent_executions` (or equivalent) for the case's last tool, outcome, attempt number, and error fields.
4. **Classify the failure.** Infrastructure error codes (timeouts, 5xx, throttling) = FAILED. Missing document/field = BLOCKED. No error but no progress = STUCK (possible lock or event-delivery issue).
5. **Check DynamoDB runtime state.** Look at `lock_owner` -- a stale lock may be preventing the next tool from executing.
6. **Search CloudWatch.** Use the correlation ID to find Lambda/AgentCore error logs for root cause detail.
7. **Check DLQs.** If retry exhaustion is suspected, inspect the relevant SQS DLQ for the failed message.

---

### A.6 Recovery Actions (Safe Resume / Replay)

All recovery actions **must** resume from the last durable checkpoint based on **Aurora status** and remain **idempotent** -- no duplicate writes or events. (§5.8, §5.9.6)

#### A.6.1 Stale Lock Release

If DynamoDB `lock_owner` is stale (the owning Lambda/AgentCore session no longer running), clear the lock. The next invocation will re-acquire it.

#### A.6.2 Re-drive Options (§5.9.6)

The system may support one or more of the following manual replay mechanisms:

| Option | When to use |
|--------|-------------|
| **Re-emit trigger event** (e.g. `CASE_INTAKE_VALIDATED` via EventBridge) | Case stuck at the very first AI stage; no tool has run yet. |
| **Invoke resume endpoint** | Case stopped mid-pipeline; endpoint reads Aurora status and picks up from the last completed tool. |
| **Re-run a specific tool invocation** | A single tool failed after retries; infrastructure issue is resolved; re-execute that tool only. |

#### A.6.3 Idempotency Rules

- Tools **must validate current Aurora status** before applying transitions. If the status already reflects completion, the tool is a no-op.
- Repeated invocations **must not** duplicate database writes or event emissions.
- The readiness step that transitions to `READY_FOR_CASEWORKER_REVIEW` must also release the workflow lock and emit `CASE_AI_READY_FOR_REVIEW` idempotently. (§5.4, §5.8, §5.9.6)

#### A.6.4 BLOCKED (Business) Cases

These are not infrastructure failures. Route to the appropriate team:

- Missing mandatory document: notify upstream / applicant to re-submit.
- Field extraction failure: review document quality; may require manual data entry or policy exception.

---

### A.7 Alerts and Alarms Mapping

| Alarm | What it means | Immediate action | v1.2 ref |
|-------|---------------|------------------|----------|
| **Tool error-rate spike** (per tool) | One tool is failing at an abnormal rate across cases. | Check CloudWatch for common error code; likely infrastructure dependency issue. | §5.9.5 |
| **Tool timeout / duration anomaly** | A tool is taking longer than normal or timing out. | Check downstream service health (Textract, Bedrock, Aurora). Scale or retry. | §5.9.5 |
| **AgentCore orchestration failure** | The AgentCore session itself failed to start or crashed. | Check Bedrock AgentCore service health; verify IAM roles and permissions. | §5.9.5 |
| **Retry exhaustion** (same case + tool) | A case has exhausted its retry budget for a specific tool. | Triage the case individually; check DLQ; attempt manual re-drive after fixing root cause. | §5.9.5 |
| **DLQ growth** | Messages accumulating in dead-letter queues. | Inspect DLQ messages for patterns (same tool? same org?). Resolve root cause, then replay. | §5.9.5 |
| **EventBridge dispatch failure** | A critical event (e.g. `CASE_INTAKE_VALIDATED` or readiness event) failed to deliver. | Check EventBridge metrics and CloudTrail; re-emit the event if delivery confirmed lost. | §5.9.5 |

All alerts must include **correlation identifiers and caseId** sufficient to locate impacted cases. (§5.9.5)

---

### A.8 Escalation and Audit Evidence

When escalating an incident or preparing evidence for an audit, capture the following for each affected case:

| Field | Source | Why |
|-------|--------|-----|
| `caseId` | Aurora / alert payload | Identifies the case. |
| `policyVersion` | Aurora `cases.policy_version` | Proves which policy was applied. (§4.3) |
| `tool_name` | Aurora `agent_executions` | Identifies which processing step failed. |
| `tool_outcome` | Aurora `agent_executions` | SUCCESS / FAILURE / RETRYING / BLOCKED. |
| `last_error_code` + `last_error_time` | Aurora tool invocation records | Categorises the failure (infra vs business). (§5.9.3) |
| Correlation ID | CloudWatch / structured logs | Links all log entries for the case's processing session. |
| Relevant CloudWatch log excerpts | CloudWatch Logs | Root cause detail (ensure **no PII** in excerpts). (§7.1, §7.2) |
| DynamoDB runtime snapshot | DynamoDB `case_runtime_state` | Lock state, attempt count at time of issue. |
| Audit log entries | Aurora `audit_logs` (immutable) | Full action history for the case. (§7.1) |

**Reminder:** No PII must appear in logs, error summaries, or escalation tickets. (§4.3, §7.1, §7.2)

---

*End of Compliance Checklist. All section references (§) refer to `requirements/agentcore-requirements-full-v1.2.md`.*
