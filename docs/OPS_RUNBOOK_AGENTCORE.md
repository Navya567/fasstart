# FastStart AgentCore Ops Runbook

**Source of truth:** `requirements/agentcore-requirements-full-v1.2.md` (v1.2) and `requirements/COMPLIANCE_CHECKLIST_AGENTCORE.md`
**Audience:** Operations engineers responsible for monitoring, triaging, and recovering stuck or failed cases.

---

## Quick Checklist

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

## 1. Purpose and Scope

This runbook covers operational detection, diagnosis, and recovery of cases that stop progressing through the FastStart AI processing pipeline. It applies to the stages between `INTAKE_VALIDATED` and `READY_FOR_CASEWORKER_REVIEW` -- the window where Bedrock AgentCore orchestrates deterministic tool execution. (§5.4, §5.9)

**Out of scope:** Caseworker portal UI issues, Cognito/SSO problems, policy authoring errors, upstream intake failures before `POST /applications/complete`.

---

## 2. Key Concepts

- **Aurora PostgreSQL** is the **authoritative checkpoint and system of record** for case status, extracted data, rule outcomes, agent outputs, decisions, and audit logs. Resume decisions are always based on Aurora status. (§6.4, §5.8, §5.9.6)
- **DynamoDB** (`case_runtime_state`) holds **runtime state only**: current stage, lock_owner, lock_expiry, attempt counts, last tool executed. It is not a checkpoint for resume. (§6.4, §7.2)
- **Tools are idempotent, deterministic contracts.** AgentCore sequences them; policy logic lives in the tools, not in the agent. Repeated invocations must not duplicate side effects. (§5.4, §5.9.6)
- **No Step Functions** are used for AI sequencing; orchestration is Bedrock AgentCore-driven. (§5.4)

---

## 3. Detecting Problems: STUCK / FAILED / BLOCKED

### 3.1 Definitions (§5.9.1)

| Category | Meaning |
|----------|---------|
| **STUCK** | Case is in a non-terminal status and has **not advanced within the stage SLA threshold** (measured by `updated_at` or equivalent in Aurora). |
| **FAILED (Infrastructure)** | A tool invocation failed due to timeouts, throttling, dependency outage, or service errors, and the case did not progress after the configured retry strategy. |
| **BLOCKED (Business)** | A mandatory document, field, or policy requirement was not met; progression is intentionally halted until action is taken. |

### 3.2 Stage SLA Thresholds (§5.9.2)

Published thresholds (time-to-progress) must exist for each stage:

| Stage | Status on entry |
|-------|-----------------|
| Validation | `INTAKE_VALIDATED` |
| Data Extraction | `DOCS_TECHNICALLY_VALIDATED` |
| Policy Evaluation | `DATA_EXTRACTED` |
| Recommendation/Summary | `POLICY_VALIDATED` |
| Mark Ready / readiness event | transition to `READY_FOR_CASEWORKER_REVIEW` |

A case is **STUCK** when `NOW() - updated_at > SLA threshold` for its current status.

### 3.3 Stuck Case View (§5.9.4)

Use the operational view (query, report, or dashboard) that lists stuck cases. The view must show:

- Current `status` (Aurora) and time-in-status
- Last tool executed + last error code/time
- Runtime lock state (DynamoDB `lock_owner`)

Filter by: status/stage, orgId, time window.

---

## 4. Where to Find "What Failed"

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

## 5. Standard Triage Steps

1. **Identify the case.** Obtain `caseId` from the alert or stuck-case view.
2. **Read Aurora status.** Query `cases` for current `status` and `updated_at`. Confirm whether it exceeds the stage SLA.
3. **Pull tool invocation records.** Query Aurora `agent_executions` (or equivalent) for the case's last tool, outcome, attempt number, and error fields.
4. **Classify the failure.** Infrastructure error codes (timeouts, 5xx, throttling) = FAILED. Missing document/field = BLOCKED. No error but no progress = STUCK (possible lock or event-delivery issue).
5. **Check DynamoDB runtime state.** Look at `lock_owner` -- a stale lock may be preventing the next tool from executing.
6. **Search CloudWatch.** Use the correlation ID to find Lambda/AgentCore error logs for root cause detail.
7. **Check DLQs.** If retry exhaustion is suspected, inspect the relevant SQS DLQ for the failed message.

---

## 6. Recovery Actions (Safe Resume / Replay)

All recovery actions **must** resume from the last durable checkpoint based on **Aurora status** and remain **idempotent** -- no duplicate writes or events. (§5.8, §5.9.6)

### 6.1 Stale Lock Release

If DynamoDB `lock_owner` is stale (the owning Lambda/AgentCore session no longer running), clear the lock. The next invocation will re-acquire it.

### 6.2 Re-drive Options (§5.9.6)

The system may support one or more of the following manual replay mechanisms:

| Option | When to use |
|--------|-------------|
| **Re-emit trigger event** (e.g. `CASE_INTAKE_VALIDATED` via EventBridge) | Case stuck at the very first AI stage; no tool has run yet. |
| **Invoke resume endpoint** | Case stopped mid-pipeline; endpoint reads Aurora status and picks up from the last completed tool. |
| **Re-run a specific tool invocation** | A single tool failed after retries; infrastructure issue is resolved; re-execute that tool only. |

### 6.3 Idempotency Rules

- Tools **must validate current Aurora status** before applying transitions. If the status already reflects completion, the tool is a no-op.
- Repeated invocations **must not** duplicate database writes or event emissions.
- The readiness step that transitions to `READY_FOR_CASEWORKER_REVIEW` must also release the workflow lock and emit `CASE_AI_READY_FOR_REVIEW` idempotently. (§5.4, §5.8, §5.9.6)

### 6.4 BLOCKED (Business) Cases

These are not infrastructure failures. Route to the appropriate team:

- Missing mandatory document: notify upstream / applicant to re-submit.
- Field extraction failure: review document quality; may require manual data entry or policy exception.

---

## 7. Alerts and Alarms Mapping

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

## 8. Escalation and Audit Evidence

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

*All section references (§) refer to `requirements/agentcore-requirements-full-v1.2.md`. This runbook does not add or modify any requirements defined in v1.2.*
