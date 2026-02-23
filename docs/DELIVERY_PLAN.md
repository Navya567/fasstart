# FastStart – Implementation Delivery Plan

**Spec version:** 1.2  
**Plan version:** 1.0  
**Date:** February 2026

---

## A) Delivery Plan Summary (1-page)

**System:** FastStart AI-Powered Case Triage & Caseworker Platform  
**Stack:** Next.js 14 / Amplify / Cognito / API Gateway / Lambda / Bedrock AgentCore / Aurora PostgreSQL / DynamoDB / S3 / EventBridge / SQS  
**Duration estimate:** 16–20 weeks across 6 phases with 2-week sprints  
**Team shape:** 2 backend, 2 frontend, 1 AI/ML, 1 platform/infra, 1 QA, 1 tech lead (shared)

| Phase | Weeks | Summary | Gate |
|-------|-------|---------|------|
| **0 – Foundation** | 1–3 | Terraform IaC, VPC, Aurora, DynamoDB, Cognito, S3 buckets, CI/CD pipelines, Amplify scaffold, event schema registry | Infra deploys to dev; Cognito login works; Aurora migrated with base tables |
| **1 – Intake Pipeline** | 4–6 | `POST /init`, presigned upload, `POST /complete`, S3 validation, EventBridge emission, DLQ | End-to-end intake: init → upload → complete → event on bus with valid schema |
| **2 – AI Orchestration** | 7–10 | AgentCore session, tools 1–5, lock semantics, agent_executions recording, case_ai_summaries, idempotent Tool #5 | Case progresses INTAKE_VALIDATED → READY_FOR_CASEWORKER_REVIEW; agent_executions populated; lock released |
| **3 – Portal MVP** | 8–13 | Login, dashboard, case list, case detail, notes, self-claim, decisions (approve/decline/escalate/pending), decision retrieval API | Caseworker can log in, view AI output, make a decision, decision retrievable via GET |
| **4 – Manager/Admin + Extras** | 12–16 | Escalation resolution, user CRUD, policy upload/validate/activate, notifications, notification prefs, optional AI email, internal SES email | Manager resolves escalation; admin manages users+policies; notifications flow |
| **5 – Ops Hardening & UAT** | 15–20 | Alarms, dashboards, stuck-case view, replay tooling, WAF/Shield tuning, performance testing, UAT, security review | All alarms firing correctly; load test passed; UAT signed off; pen-test clear |

Phases 3 and 4 overlap with Phase 2 (frontend work can start on mocks while AI pipeline stabilises).

**Critical path:** Phase 0 → Phase 1 → Phase 2 (tool contracts block portal showing real AI output). Frontend scaffolding starts in Phase 0 and runs parallel.

---

## 1) Implementation Phase Plan

### Phase 0 – Foundation (Weeks 1–3)

**Scope:**
- Terraform modules: networking, data, auth, security, observability, dns, frontend
- VPC with public/private subnets (≥2 AZs), NAT, VPC endpoints (S3, DynamoDB, Bedrock, SQS, EventBridge, Secrets Manager, CW Logs)
- Aurora PostgreSQL Serverless v2 (Multi-AZ, IAM auth, PITR, deletion protection)
- DynamoDB tables: `case_runtime_state`, `user_notifications`, `notification_preferences` (on-demand, KMS, PITR)
- S3 buckets: intake documents, policy definitions, decision bundles, server access logs (KMS, versioning, object lock, lifecycle rules, block public access)
- Cognito User Pool with OIDC/SAML federation stub, app client, role claims (Caseworker/Manager/Administrator)
- API Gateway with Cognito/Lambda authorizer, WAF WebACL, custom domain placeholder
- Amplify app scaffold (Next.js 14, branch-based environments, Cognito integration)
- Aurora schema migration v001: all §7.1 tables + `case_ai_summaries` + indexes from Appendix D
- EventBridge event bus + JSON Schema registry (Appendix H schemas)
- CI/CD: backend pipeline (lint → test → scan → plan → apply) + Amplify auto-build
- Secrets Manager entries for DB credentials; SSM parameters for config
- CloudTrail, CloudWatch log groups, base alarms

**Prerequisites:** AWS accounts provisioned (dev/tst/prd or at minimum dev), Route 53 hosted zone delegated, IdP configured for Cognito federation.

**Deliverables:**
- [ ] Terraform `plan` clean for all modules
- [ ] Aurora reachable from Lambda in private subnet
- [ ] Cognito login produces JWT with role claim
- [ ] Amplify serves Next.js hello-world on `faststart-dev.<domain>`
- [ ] EventBridge bus accepts and validates test events
- [ ] CI/CD pipeline runs end-to-end on merge to develop

**Acceptance criteria:**
1. `terraform apply` succeeds in dev with no manual steps.
2. Lambda can connect to Aurora via IAM auth and run a SELECT.
3. Cognito hosted UI login redirects back with valid JWT containing `custom:role`.
4. Amplify preview deployment works on PR.
5. Event published to bus passes JSON Schema validation.
6. All S3 buckets have public access blocked, KMS encryption, and lifecycle rules.

**Key risks:**
- Bedrock AgentCore regional availability — verify early in dev account.
- IdP integration delays (Azure AD/Okta config can take time).
- ACM certificate validation timing if DNS delegation is pending.

---

### Phase 1 – Intake Pipeline (Weeks 4–6)

**Scope:**
- `POST /applications/init` Lambda: schema validation, policy resolution, semantic validation, case creation, policy version lock, presigned URL generation, Aurora + DynamoDB writes
- `POST /applications/complete` Lambda: S3 inspection (manifest, mandatory docs, version limits, lookback, MIME types), Aurora case_documents write, DynamoDB status → INTAKE_VALIDATED, EventBridge event emission with §12.1 schema
- Presigned URL generation for S3 document upload
- Submission type handling (NEW duplicate rejection, UPDATE version increment)
- Policy version immutability enforcement
- Correlation ID generation (init) and propagation
- SQS queue for AI pipeline trigger (EventBridge → SQS → consumer Lambda)
- DLQ for intake SQS queue
- EventBridge schema contract tests in CI
- Audit log entries: CASE_INITIATED, INTAKE_COMPLETED

**Prerequisites:** Phase 0 complete. Seed data: at least 1 org, 1 case_type, 1 active policy with policy_documents.

**Deliverables:**
- [ ] Init endpoint creates case, returns presigned URLs
- [ ] Complete endpoint validates all S3 rules and emits event
- [ ] Event on SQS queue with valid schema
- [ ] Duplicate NEW rejected (409); UPDATE increments version
- [ ] DLQ captures failed events
- [ ] Integration test: init → upload docs to S3 → complete → event visible

**Acceptance criteria:**
1. Init with submissionType=NEW for new caseId returns 201 with presigned URLs.
2. Init with submissionType=NEW for existing active caseId returns 409.
3. Complete with all docs present returns 200 and emits CASE_INTAKE_VALIDATED.
4. Complete with missing mandatory doc returns 422 with VALIDATION_ERROR.
5. EventBridge event payload passes Appendix H schema validation.
6. Correlation ID present in Aurora, DynamoDB, EventBridge event, and CloudWatch logs.

**Key risks:**
- S3 presigned URL expiry window — set to 15 min default, document in API response.
- Policy seed data must be realistic enough to test all validation paths.

---

### Phase 2 – AI Orchestration (Weeks 7–10)

**Scope:**
- AgentCore session bootstrap Lambda (SQS consumer → invoke AgentCore)
- DynamoDB lock acquire/release/heartbeat per Appendix C.3
- Tool #1: Document validation (Textract + policy rules → DOCS_TECHNICALLY_VALIDATED)
- Tool #2: Data extraction (Textract + policy extraction fields → extracted_case_data → DATA_EXTRACTED)
- Tool #3: Policy evaluation (policy_rules against extracted data → rule_evaluations → POLICY_VALIDATED)
- Tool #4: Case summary & recommendation (Bedrock reasoning → case_ai_summaries with JSONB risk_assessment → SUMMARY_READY)
- Tool #5: Mark ready (pre-condition check, Aurora+DynamoDB status, event emission, lock release, strict idempotency → READY_FOR_CASEWORKER_REVIEW)
- agent_executions recording for every tool invocation (all §5.9.3 fields)
- State transition enforcement per Appendix C.1 (reject invalid transitions)
- Retry logic: exponential backoff, max 3 attempts, failure classification (infra vs business)
- Partial output persistence (extracted data saved even if later tool fails)
- BLOCKED status handling
- Correlation ID propagation through entire session
- GET /applications/{caseId}/decision Lambda

**Prerequisites:** Phase 1 complete. Bedrock AgentCore access verified. Textract access verified.

**Deliverables:**
- [ ] Full pipeline: INTAKE_VALIDATED → ... → READY_FOR_CASEWORKER_REVIEW
- [ ] agent_executions has records for all 5 tool invocations
- [ ] case_ai_summaries populated with risk_assessment JSONB
- [ ] Lock acquired at session start, released by Tool #5
- [ ] Tool #5 re-invocation is a no-op (no duplicate event/writes)
- [ ] BLOCKED case when mandatory document extraction fails
- [ ] Decision retrieval API returns 404 when no decision yet

**Acceptance criteria:**
1. Case transitions through all 5 stages with agent_executions for each.
2. case_ai_summaries.risk_assessment contains risk_level, risk_factors, mitigations.
3. DynamoDB lock_owner set during processing; NULL after Tool #5.
4. Repeated SQS delivery (simulated) does not create duplicate agent_executions or events.
5. Tool invocation on already-advanced case is a no-op.
6. BLOCKED case has agent_executions with outcome=BLOCKED and case status=BLOCKED.
7. Correlation ID traceable end-to-end from EventBridge to all agent_executions records.

**Key risks:**
- Bedrock AgentCore SDK maturity — may need Strands SDK fallback.
- Textract accuracy on diverse document types — early testing essential.
- Lock TTL tuning — too short causes spurious lock expiry; too long blocks recovery.

---

### Phase 3 – Caseworker Portal MVP (Weeks 8–13)

**Scope:**
- Login screen with Cognito SSO (OIDC/SAML redirect)
- Homepage dashboard (ISR 60s): case count by status, priority distribution
- Case management list (ISR 30s): paginated, sortable, filterable, role-based visibility (§12.2)
- Individual case detail (SSR + SWR): applicant info, documents (presigned URLs), AI summary, risk assessment, extracted fields, rule evaluations, decision history, notes
- Add note (POST, append-only, audit: NOTE_ADDED)
- Self-claim (POST, atomic 409 handling, audit: CASE_CLAIMED)
- Decision submit (POST: APPROVED/DECLINED/PENDING/ESCALATED, audit: DECISION_*)
- Notifications list (client-side fetch/polling, mark read/unread)
- Settings stub (profile, notification preferences)
- RBAC guards: UI-level (hide/disable) + API-level (403)
- Standard error handling with correlation ID display
- Mutation-triggered refetch (§12.4)
- Pagination component (reusable: items, totalCount, page, pageSize, hasMore)

**Prerequisites:** Phase 0 Cognito + Amplify working. Phase 2 producing AI outputs (can use mocked data initially and switch to live when Phase 2 is complete).

**Deliverables:**
- [ ] Caseworker logs in, sees assigned + unassigned cases
- [ ] Case detail shows AI summary, risk assessment, documents, notes
- [ ] Caseworker can claim, add note, and approve/decline/escalate
- [ ] Decision retrieval API returns the decision
- [ ] Notifications appear and can be marked read
- [ ] 409 shown gracefully when claim race occurs

**Acceptance criteria:**
1. CW sees only own-assigned + unassigned in own org; MGR sees all in org.
2. Risk assessment on case detail matches case_ai_summaries.risk_assessment 1:1.
3. Self-claim on already-claimed case shows user-friendly "already assigned" message.
4. Decision submit writes case_decisions + audit_logs + emits EventBridge event.
5. Notes appear in reverse chronological order; no edit/delete controls visible.
6. After decision submit, `GET /applications/{caseId}/decision` returns the decision.
7. ISR pages revalidate at configured intervals; mutation triggers immediate refresh.

**Key risks:**
- AI output format may evolve during Phase 2 — use typed interfaces with optional fields.
- Presigned URL expiry for document viewing — generate on demand per case detail load.

---

### Phase 4 – Manager/Admin + Extended Features (Weeks 12–16)

**Scope:**
- Manager escalation list (ISR 30s), escalation detail with full decision chain, manager resolution (new case_decisions record, original preserved)
- Admin user management: list, create (Aurora + Cognito), activate/deactivate, soft delete, audit for each action
- Admin policy management: list versions, upload (S3 → EventBridge → validation Lambda), validation status display, activate version
- Policy validation Lambda: schema validation, semantic validation, fairness constraint enforcement, normalisation, persist to Aurora
- Notification preferences UI (Settings screen: toggle per notification type)
- Profile image upload (S3 presigned upload)
- Optional AI-drafted citizen email: draft (POST /email/draft → Bedrock), review in modal, confirm send (POST /email/send → SES + audit)
- Internal notification emails via SES (§12.5): org-level enablement, eligible types, non-blocking, INTERNAL_EMAIL_SENT audit
- Optional notification_email_deliveries tracking table

**Prerequisites:** Phase 3 complete for portal shell and auth. Phase 2 complete for policy tables to be populated.

**Deliverables:**
- [ ] Manager views escalation list, opens case, sees full decision chain, submits new resolution
- [ ] Admin creates user (appears in Cognito + Aurora), deactivates, soft-deletes
- [ ] Admin uploads policy YAML, sees validation status, activates
- [ ] Policy validation rejects contradictory rules and fairness violations
- [ ] Notification preferences toggleable; changes persist
- [ ] AI email draft → review → send → audit_logs entry
- [ ] SES internal email fires for CASE_ASSIGNED (when org enabled + user opted in)

**Acceptance criteria:**
1. Manager resolution creates NEW case_decisions row; original caseworker decision unchanged.
2. Escalation history view shows all decisions in order with performer identity.
3. User create in admin writes USER_CREATED to audit_logs.
4. Policy with conflicting rules rejected at upload with VALIDATION_ERROR.
5. Policy activation changes which policy version init Lambda resolves.
6. Citizen email sent only after explicit caseworker confirmation; EMAIL_SENT in audit.
7. Internal SES email failure does not fail the case assignment action.

**Key risks:**
- Cognito user sync with Aurora (eventual consistency) — sync on create, verify on login.
- Policy YAML schema complexity — start with the simplest supported policy structure.
- SES sandbox limits in dev — request production access early.

---

### Phase 5 – Ops Hardening & UAT (Weeks 15–20)

**Scope:**
- CloudWatch alarms per §5.9.5 + Appendix E.9 (tool errors, DLQ growth, API 5xx, Lambda errors, Aurora CPU, EventBridge failures)
- Operational dashboard: case pipeline throughput, stage durations, error rates by tool, DLQ depth, API latency percentiles
- Stuck-case view (query or lightweight dashboard) per §5.9.4
- Replay tooling (MAY): re-emit CASE_INTAKE_VALIDATED, resume endpoint, or re-run single tool
- Stale lock detection + manual clear procedure
- WAF rule tuning (rate limits per env)
- Shield verification
- Performance/load testing: 100 concurrent intakes, 50 AI pipelines, 200 portal users
- Failure injection tests: DLQ, duplicate events, stale locks, Textract throttling
- Security review: IAM policies, KMS key access, no PII in logs, pen-test
- UAT with representative users (caseworkers, managers, admins)
- Runbook documentation for ops team
- Alarm-to-owner matrix populated

**Prerequisites:** Phases 0–4 functionally complete. Test environment (tst) stable.

**Deliverables:**
- [ ] All alarms configured and tested (each fired at least once in test)
- [ ] Dashboard shows real-time pipeline health
- [ ] Stuck-case view returns expected results for simulated stuck cases
- [ ] Load test passes with <500ms p95 API latency under target load
- [ ] No PII found in CloudWatch logs (grep audit)
- [ ] UAT sign-off from product owner
- [ ] Runbooks published for top-8 operational scenarios

**Acceptance criteria:**
1. DLQ alarm fires within 5 min of first DLQ message.
2. Stuck case appears in view within 1 SLA threshold window + polling interval.
3. Replay of a stuck case resumes from Aurora checkpoint and does not duplicate data.
4. Load test: 100 concurrent init requests complete within 2s p95.
5. Security scan (Checkov + Snyk) passes with no critical findings.
6. UAT: 3 representative user journeys completed by non-dev users without assistance.

**Key risks:**
- Performance bottleneck in Aurora under concurrent AI pipeline writes — monitor ACU usage.
- WAF false positives blocking legitimate traffic — tune in tst before prd.
- UAT schedule dependency on user availability.

---

## 2) Work Breakdown Structure

### Epic 1: Infrastructure Foundation

| ID | Feature / Story | Type | Acceptance Criteria |
|----|----------------|------|---------------------|
| E1-F1 | **VPC & Networking** | | |
| E1-S01 | Create VPC with 2 AZ public/private subnets, NAT gateways, route tables | Tech | `terraform apply` creates VPC; Lambda in private subnet can reach internet via NAT |
| E1-S02 | Provision VPC endpoints (S3, DynamoDB, Bedrock, SQS, EventBridge, Secrets Manager, CW Logs) | Tech | Lambda calls each service without traversing NAT |
| E1-S03 | Security groups: Lambda SG, Aurora SG (ingress from Lambda SG only), VPC endpoint SGs | Tech | Aurora reachable only from Lambda SG; no 0.0.0.0/0 ingress |
| E1-F2 | **Data Stores** | | |
| E1-S04 | Aurora PostgreSQL Serverless v2 cluster (Multi-AZ, IAM auth, PITR, deletion protection, Performance Insights) | Tech | Cluster accessible via IAM auth from Lambda; failover tested |
| E1-S05 | Aurora migration v001: all §7.1 tables + case_ai_summaries + indexes (Appendix D) | Tech | All tables created; `REVOKE UPDATE, DELETE` on audit_logs, case_notes from app role |
| E1-S06 | DynamoDB tables: case_runtime_state, user_notifications (+ GSI), notification_preferences | Tech | Tables created with KMS, PITR; GSI on user_notifications queryable |
| E1-S07 | S3 buckets: intake docs, policy defs, decision bundles, access logs (KMS, versioning, object lock, lifecycle, block public access) | Tech | All buckets pass security checklist; lifecycle rules verified |
| E1-F3 | **Auth** | | |
| E1-S08 | Cognito User Pool with OIDC/SAML IdP federation, custom:role attribute, app client | Tech | Login via hosted UI returns JWT with custom:role claim |
| E1-S09 | API Gateway Lambda/Cognito authorizer validating JWT and extracting role + orgId | Tech | Authorizer rejects expired/missing JWT (401); passes valid JWT with claims |
| E1-F4 | **CI/CD & Observability** | | |
| E1-S10 | Backend CI/CD pipeline: lint, test, scan, terraform plan/apply, Lambda deploy | Tech | Pipeline runs on merge to develop; blocks on test failure |
| E1-S11 | Amplify app with branch-based envs (main→prd, develop→dev), Next.js scaffold, Cognito integration | Tech | Amplify preview deploy on PR; develop auto-deploys |
| E1-S12 | CloudTrail (mgmt + data events), CloudWatch log groups (structured JSON), base alarms | Tech | CloudTrail events visible in console; log groups created for all Lambdas |
| E1-S13 | Secrets Manager (DB creds) + SSM Parameter Store (config: ISR intervals, SLA thresholds) | Tech | Lambda reads DB connection from Secrets Manager; SSM params accessible |
| E1-F5 | **EventBridge & Schemas** | | |
| E1-S14 | EventBridge custom bus + JSON Schema registry (Appendix H schemas) | Tech | Test event published and validated against schema |
| E1-S15 | EventBridge schema contract tests in CI (validate producer output matches schema) | Tech | CI fails if schema is broken; passes on valid schema |
| E1-F6 | **DNS & Edge** | | |
| E1-S16 | Route 53 hosted zone, ACM certificates (us-east-1 for CloudFront + region) | Tech | Certificates issued and validated via DNS |
| E1-S17 | WAF WebACL (AWS Managed Rules + rate limiting) on API Gateway + CloudFront | Tech | WAF associated; test request blocked by SQL injection rule |
| E1-S18 | Shield Standard enabled on CloudFront + API Gateway | Tech | Shield status confirmed in console |
| E1-S19 | Custom domains: `faststart-dev.<domain>`, `api-faststart-dev.<domain>` | Tech | Frontend and API reachable on custom domains |

### Epic 2: Intake Pipeline

| ID | Story | Type | Acceptance Criteria |
|----|-------|------|---------------------|
| E2-S01 | POST /applications/init: schema validation, policy resolution, case creation, DynamoDB + Aurora writes, presigned URLs | Tech | 201 with presigned URLs; Aurora `cases` row exists; DynamoDB `case_runtime_state` exists |
| E2-S02 | Policy version lock at init (immutable for case lifetime) | Tech | Subsequent UPDATE preserves original policyVersion |
| E2-S03 | Submission type NEW: reject duplicate caseId (409 Conflict) | Tech | Second init with same caseId (non-terminal) returns 409 |
| E2-S04 | Submission type UPDATE: increment applicationVersion, preserve policyVersion | Tech | applicationVersion incremented; policyVersion unchanged |
| E2-S05 | Correlation ID generation at init; propagated to all downstream writes and logs | Tech | correlationId in Aurora cases, DynamoDB, and CloudWatch logs |
| E2-S06 | POST /applications/complete: S3 validation (manifest, mandatory docs, version limits, lookback, MIME types against policy) | Tech | 200 when all docs present; 422 with details when validation fails |
| E2-S07 | Complete: emit CASE_INTAKE_VALIDATED event with §12.1 mandatory fields | Tech | Event on bus passes Appendix H schema validation |
| E2-S08 | Complete: Aurora case_documents insert, DynamoDB status → INTAKE_VALIDATED | Tech | case_documents rows match uploaded docs; DynamoDB status = INTAKE_VALIDATED |
| E2-S09 | SQS queue for AI pipeline (EventBridge rule → SQS) with DLQ (maxReceiveCount=3) | Tech | Event flows to SQS; after 3 failures lands in DLQ |
| E2-S10 | Audit log entries: CASE_INITIATED (init), INTAKE_COMPLETED (complete) | Tech | audit_logs rows exist with correct action, performed_by, timestamp |

### Epic 3: AI Orchestration

| ID | Story | Type | Acceptance Criteria |
|----|-------|------|---------------------|
| E3-S01 | SQS consumer Lambda: invoke AgentCore session with caseId + correlationId | Tech | AgentCore session starts; correlation ID in session context |
| E3-S02 | DynamoDB lock acquire (conditional write: lock_owner WHERE NULL or expired) | Tech | Lock set on session start; second session aborts |
| E3-S03 | Lock heartbeat (extend expiry during long-running tools) | Tech | lock_expiry updated during execution; no premature expiry |
| E3-S04 | Tool #1 – Document validation: Textract + policy rules, DOCS_TECHNICALLY_VALIDATED | Tech | agent_executions row with outcome=SUCCESS; Aurora status advanced |
| E3-S05 | Tool #2 – Data extraction: Textract fields + policy extraction_fields → extracted_case_data, DATA_EXTRACTED | Tech | extracted_case_data rows match expected fields; confidence scores present |
| E3-S06 | Tool #3 – Policy evaluation: policy_rules against extracted data → rule_evaluations, POLICY_VALIDATED | Tech | rule_evaluations rows with pass/fail/conditional results |
| E3-S07 | Tool #4 – Case summary: Bedrock reasoning → case_ai_summaries (JSONB risk_assessment), SUMMARY_READY | Tech | case_ai_summaries row with risk_level, risk_factors, mitigations; confidence_score populated |
| E3-S08 | Tool #5 – Mark ready: pre-condition (SUMMARY_READY), Aurora+DynamoDB update, EventBridge CASE_AI_READY_FOR_REVIEW, lock release | Tech | Status = READY_FOR_CASEWORKER_REVIEW; event emitted; lock_owner = NULL |
| E3-S09 | Tool #5 strict idempotency: repeated invocation is no-op (no dup writes/events/lock releases) | Tech | Second invocation produces no new agent_executions, no new event, returns success |
| E3-S10 | All tools: agent_executions recording (all §5.9.3 fields) | Tech | Every tool invocation has agent_executions row with correlation_id, timestamps, outcome |
| E3-S11 | State transition enforcement: reject invalid transitions per Appendix C.1 | Tech | Tool invoked on wrong status returns no-op (idempotent) |
| E3-S12 | Retry logic: exponential backoff, max 3 attempts, failure classification | Tech | 3 agent_executions rows for retried tool (attempt 1-3); 4th failure → DLQ |
| E3-S13 | Business-blocked handling: BLOCKED status + agent_executions outcome=BLOCKED | Tech | Missing doc → status=BLOCKED; agent_executions records reason |
| E3-S14 | Partial output persistence: extracted data saved even if later tool fails | Tech | Tool #2 succeeds, Tool #3 fails → extracted_case_data rows preserved |
| E3-S15 | Correlation ID propagation: from SQS message through all tool invocations to agent_executions | Tech | Same correlationId in SQS message, all agent_executions, CloudWatch logs |
| E3-S16 | GET /applications/{caseId}/decision: retrieve latest decision from Aurora | Tech | Returns 200 with decision after approval; 404 before decision |

### Epic 4: Caseworker Portal

| ID | Story | Type | Acceptance Criteria |
|----|-------|------|---------------------|
| E4-S01 | Login page (SSG): Cognito SSO redirect, JWT storage, role extraction | User | User clicks login → redirected to IdP → returned with session |
| E4-S02 | Auth session management: token refresh, session expiry, logout | Tech | Silent refresh before token expiry; logout clears session |
| E4-S03 | RBAC UI guards: hide/disable actions based on role | Tech | CW cannot see user management or assign buttons |
| E4-S04 | Homepage dashboard (ISR 60s): case count by status, priority chart | User | Dashboard shows aggregated stats; refreshes within 60s |
| E4-S05 | Case list (ISR 30s): paginated table, sort by created_at/status/updated_at, filter by status, search by caseId | User | 20 cases per page; sort/filter works; search returns match |
| E4-S06 | Case list: role-based visibility (CW: own+unassigned in org, MGR: all in org) | User | CW does not see cases assigned to others; MGR sees all |
| E4-S07 | Case detail (SSR + SWR): applicant info, documents (presigned URLs), AI summary, risk assessment | User | All sections render; risk assessment matches Aurora 1:1 |
| E4-S08 | Case detail: extracted fields display, rule evaluation results | User | Each extracted field shown with confidence; each rule shown with pass/fail |
| E4-S09 | Case detail: decision history (reverse chronological, immutable display) | User | All decisions shown with performer, timestamp; no edit/delete controls |
| E4-S10 | Add note (POST, append-only): form, submit, immediate refetch | User | Note appears in list after submit; audit_logs has NOTE_ADDED |
| E4-S11 | Self-claim case (POST /claim): atomic, 409 handling with user-friendly message | User | Unassigned case claimed successfully; race returns "already assigned" |
| E4-S12 | Assignment/unassignment/reassignment by manager (POST /assign, /unassign, /reassign) | User | Manager assigns CW; audit_logs has CASE_ASSIGNED |
| E4-S13 | Decision submit (POST /decision): approve/decline/pending/escalate with justification | User | Decision persisted; case status updated; event emitted; redirect to list |
| E4-S14 | Decision idempotency: double-click prevention | Tech | Second click within window returns same decisionId |
| E4-S15 | Notifications list (client-side fetch): show unread count in nav, list with mark read/unread | User | Notifications load; unread badge updates; mark-read is optimistic |
| E4-S16 | Notification preferences (Settings): toggle per notification type | User | Toggle persists to DynamoDB; controls which notifications appear |
| E4-S17 | Profile image upload (Settings): presigned upload to S3 | User | Image uploads; displayed in nav after refresh |
| E4-S18 | Pagination component (reusable): items/totalCount/page/pageSize/hasMore | Tech | Used on case list, escalation list, notifications, user list, policy list |
| E4-S19 | Error handling: standard error display with correlation ID | Tech | API error shows message + correlation ID for support reference |
| E4-S20 | Mutation-triggered refresh: router.refresh() / SWR mutate() after all mutations | Tech | ISR data refreshes immediately after decision/note/claim/assignment |

### Epic 5: Manager & Admin Features

| ID | Story | Type | Acceptance Criteria |
|----|-------|------|---------------------|
| E5-S01 | Escalation list (ISR 30s): escalated cases, paginated, filterable | User | MGR sees escalated cases in own org |
| E5-S02 | Escalation detail: full decision chain, caseworker notes, AI risk insights | User | All prior decisions displayed; original caseworker decision clearly marked |
| E5-S03 | Manager resolution: submit new decision (new case_decisions row; original preserved) | User | New decision row in Aurora; original row unchanged; MANAGER_DECISION audit |
| E5-S04 | Admin user list (SSR + client): paginated, sortable | User | Admin sees all users with status |
| E5-S05 | Admin create user: Aurora INSERT + Cognito create, USER_CREATED audit | User | User appears in list and can log in |
| E5-S06 | Admin activate/deactivate user: Aurora + Cognito update, audit logged | User | Deactivated user cannot log in; USER_DEACTIVATED audit |
| E5-S07 | Admin soft delete user: Aurora set deleted_at + Cognito disable, USER_DELETED audit | User | User hidden from list; cannot log in |
| E5-S08 | Admin policy list (SSR + client): all versions with status (draft/active/retired) | User | Policies shown with version and status |
| E5-S09 | Admin policy upload: S3 presigned upload → EventBridge → validation Lambda | User | Upload starts; validation status displayed (pending → valid/invalid) |
| E5-S10 | Policy validation Lambda: schema + semantic + fairness constraint enforcement | Tech | Invalid policy shows detailed errors; valid policy persisted to Aurora |
| E5-S11 | Admin policy activate: Aurora UPDATE policies.status=active, POLICY_ACTIVATED audit | User | Next intake uses new policy version |
| E5-S12 | Optional AI email: POST /email/draft (Bedrock), review modal, POST /email/send (SES), EMAIL_SENT audit | User | Email sent only after confirm; audit_logs entry with action=EMAIL_SENT |
| E5-S13 | Internal SES email for eligible notification types (§12.5): org-level + user-level prefs, non-blocking | Tech | SES email fires for CASE_ASSIGNED when enabled; SES failure does not fail assignment |

### Epic 6: Ops Hardening

| ID | Story | Type | Acceptance Criteria |
|----|-------|------|---------------------|
| E6-S01 | CloudWatch alarms: tool error rate, DLQ growth, API 5xx, Lambda errors, Aurora CPU, EventBridge failures | Tech | Each alarm tested (triggered + resolved) |
| E6-S02 | Operational dashboard: pipeline throughput, stage durations, error rates, DLQ depth, API latency | Tech | Dashboard renders with live data |
| E6-S03 | Stuck-case view: query or dashboard listing cases where updated_at > SLA threshold | Tech | Simulated stuck case appears within 1 polling cycle |
| E6-S04 | DLQ alarm + triage procedure documented | Tech | Alarm fires; runbook followed successfully |
| E6-S05 | Stale lock detection + manual clear runbook | Tech | Stale lock identified; cleared; case resumes |
| E6-S06 | Replay tooling (MAY): re-emit event or invoke resume endpoint | Tech | Replayed case resumes from Aurora checkpoint; no duplicate data |
| E6-S07 | Alarm-to-owner matrix populated | Tech | Every alarm has owner, escalation path, runbook reference |
| E6-S08 | X-Ray tracing enabled on Lambda + API Gateway | Tech | Traces visible in X-Ray console; linked to correlationId |

---

## 3) Repo / Monorepo Structure

```
faststart/
├── apps/
│   └── portal-web/                 # Next.js 14 + React 18 + TypeScript
│       ├── src/
│       │   ├── app/                # Next.js App Router pages
│       │   ├── components/         # Shared React components
│       │   ├── lib/                # API client, auth helpers, SWR hooks
│       │   ├── types/              # Shared TypeScript interfaces (imports from packages/shared-types)
│       │   └── middleware.ts       # Auth middleware (Cognito JWT validation)
│       ├── public/
│       ├── amplify.yml             # Amplify build spec
│       ├── next.config.js
│       ├── tailwind.config.ts
│       ├── tsconfig.json
│       └── package.json
├── services/
│   ├── api-lambdas/                # Portal + intake API handlers
│   │   ├── src/
│   │   │   ├── handlers/           # One file per Lambda entry point
│   │   │   │   ├── application-init.ts
│   │   │   │   ├── application-complete.ts
│   │   │   │   ├── get-decision.ts
│   │   │   │   ├── portal-dashboard.ts
│   │   │   │   ├── portal-cases.ts
│   │   │   │   ├── portal-case-detail.ts
│   │   │   │   ├── portal-case-claim.ts
│   │   │   │   ├── portal-case-assign.ts
│   │   │   │   ├── portal-case-decision.ts
│   │   │   │   ├── portal-case-notes.ts
│   │   │   │   ├── portal-notifications.ts
│   │   │   │   ├── portal-settings.ts
│   │   │   │   ├── portal-email.ts
│   │   │   │   ├── admin-users.ts
│   │   │   │   └── admin-policies.ts
│   │   │   ├── middleware/         # Auth, correlation ID, error handling, logging
│   │   │   ├── repositories/      # Aurora + DynamoDB data access (query builders)
│   │   │   ├── services/          # Business logic (validation, assignment, decisions)
│   │   │   └── utils/             # Shared utilities
│   │   ├── tsconfig.json
│   │   └── package.json
│   ├── ai-pipeline/                # AgentCore orchestration + tools
│   │   ├── src/
│   │   │   ├── orchestrator.ts     # SQS consumer → AgentCore session bootstrap
│   │   │   ├── tools/
│   │   │   │   ├── document-validation.ts    # Tool #1
│   │   │   │   ├── data-extraction.ts        # Tool #2
│   │   │   │   ├── policy-evaluation.ts      # Tool #3
│   │   │   │   ├── case-summary.ts           # Tool #4
│   │   │   │   └── mark-ready.ts             # Tool #5
│   │   │   ├── lock/               # DynamoDB lock acquire/release/heartbeat
│   │   │   ├── state/              # Status transition validation
│   │   │   └── recording/          # agent_executions writer
│   │   ├── tsconfig.json
│   │   └── package.json
│   └── event-handlers/             # EventBridge-triggered Lambdas
│       ├── src/
│       │   ├── notification-creator.ts   # Creates DynamoDB user_notifications
│       │   ├── internal-email-sender.ts  # SES for internal notifications
│       │   └── policy-validator.ts       # S3 ObjectCreated → validate → Aurora
│       ├── tsconfig.json
│       └── package.json
├── packages/
│   └── shared-types/               # Shared TypeScript types/interfaces
│       ├── src/
│       │   ├── api/                # Request/response DTOs
│       │   ├── events/             # EventBridge event types (from JSON schemas)
│       │   ├── db/                 # Aurora table row types
│       │   └── enums.ts            # CaseStatus, Decision, Role, etc.
│       ├── tsconfig.json
│       └── package.json
├── schemas/
│   ├── events/                     # EventBridge JSON Schemas
│   │   ├── case-intake-validated.v1.0.0.json
│   │   ├── case-ai-ready-for-review.v1.0.0.json
│   │   ├── case-decision-made.v1.0.0.json
│   │   ├── case-assigned.v1.0.0.json
│   │   └── ...
│   └── api/                        # OpenAPI or JSON Schema for API contracts
│       └── portal-api.v1.yaml
├── infra/
│   └── terraform/
│       ├── environments/           # Per-env tfvars
│       │   ├── dev.tfvars
│       │   ├── tst.tfvars
│       │   └── prd.tfvars
│       ├── modules/
│       │   ├── networking/
│       │   ├── data/
│       │   ├── compute/
│       │   ├── auth/
│       │   ├── frontend/
│       │   ├── security/
│       │   ├── observability/
│       │   └── dns/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── backend.tf
├── migrations/
│   └── aurora/
│       ├── V001__base_tables.sql
│       ├── V002__case_ai_summaries.sql
│       ├── V003__notification_email_deliveries.sql
│       ├── V004__indexes.sql
│       └── V005__immutability_grants.sql
├── tests/
│   ├── contract/                   # EventBridge schema validation tests
│   ├── integration/                # API integration tests
│   ├── e2e/                        # Playwright/Cypress end-to-end
│   └── load/                       # k6 / Artillery load tests
├── docs/
│   ├── runbooks/
│   ├── architecture/
│   └── DELIVERY_PLAN.md
├── turbo.json                      # Turborepo config (or nx.json)
├── package.json                    # Workspace root
└── tsconfig.base.json              # Shared TS config
```

**Naming conventions:**
- Lambda handlers: `FastStart-<env>-<kebab-name>` (e.g. `FastStart-dev-application-init`)
- S3 buckets: `<org-id>-<case-type>-applicant-intake-s3-<env>` or `faststart-<env>-<purpose>`
- DynamoDB: `FastStart-<env>-<table-name>` (e.g. `FastStart-dev-case-runtime-state`)
- Shared types imported as `@faststart/shared-types`

---

## 4) API Implementation Blueprint

### Endpoint Inventory

| Domain | Method | Path | Role | Idempotency | Spec ref |
|--------|--------|------|------|-------------|----------|
| **Intake** | POST | `/applications/init` | Upstream/API key | caseId+submissionType natural key | §5.2.1 |
| | POST | `/applications/complete` | Upstream/API key | caseId (reject if already validated) | §5.2.3 |
| | GET | `/applications/{caseId}/decision` | Upstream/API key | Read-only | §5.7 |
| **Portal – Cases** | GET | `/portal/dashboard` | CW/MGR/ADM | Read-only | App A #2 |
| | GET | `/portal/cases` | CW/MGR/ADM | Read-only | App A #3 |
| | GET | `/portal/cases/{caseId}` | CW(own)/MGR/ADM | Read-only | App A #4 |
| | POST | `/portal/cases/{caseId}/notes` | CW(own)/MGR | — | App A #5 |
| | POST | `/portal/cases/{caseId}/claim` | CW | Atomic (409) | App A #6 |
| | POST | `/portal/cases/{caseId}/assign` | MGR/ADM | — | App A #7 |
| | POST | `/portal/cases/{caseId}/unassign` | MGR/ADM | — | App A #8 |
| | POST | `/portal/cases/{caseId}/reassign` | MGR/ADM | — | App A #9 |
| | POST | `/portal/cases/{caseId}/decision` | CW(own)/MGR | Time-window dedup | App A #10-13,22 |
| | POST | `/portal/cases/{caseId}/email/draft` | CW(own) | — | App A #14 |
| | POST | `/portal/cases/{caseId}/email/send` | CW(own) | Idempotency-Key | App A #15 |
| **Portal – Notifications** | GET | `/portal/notifications` | CW/MGR/ADM | Read-only | App A #16 |
| | PATCH | `/portal/notifications/{id}` | CW/MGR/ADM | — | App A #17-18 |
| **Portal – Settings** | PUT | `/portal/settings/notifications` | CW/MGR/ADM | — | App A #19 |
| | POST | `/portal/settings/profile/image` | CW/MGR/ADM | — | App A #20 |
| **Portal – Escalations** | GET | `/portal/escalations` | MGR | Read-only | App A #21 |
| **Admin** | GET | `/portal/admin/users` | ADM | Read-only | App A #23 |
| | POST | `/portal/admin/users` | ADM | — | App A #24 |
| | PATCH | `/portal/admin/users/{userId}` | ADM | — | App A #25 |
| | DELETE | `/portal/admin/users/{userId}` | ADM | — | App A #26 |
| | GET | `/portal/admin/policies` | ADM | Read-only | App A #27 |
| | POST | `/portal/admin/policies/upload` | ADM | — | App A #28 |
| | POST | `/portal/admin/policies/{policyId}/activate` | ADM | — | App A #29 |

### Pagination Contract (reusable)

```typescript
interface PaginatedRequest {
  page?: number;      // 1-based, default 1
  pageSize?: number;  // default 20, max 100
  sortBy?: string;    // column name
  sortOrder?: 'asc' | 'desc';
}

interface PaginatedResponse<T> {
  items: T[];
  totalCount: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}
```

### Error Mapping Matrix

| HTTP Status | error.code | When |
|-------------|-----------|------|
| 400 | `VALIDATION_ERROR` | Malformed request, missing required fields, invalid data types |
| 403 | `FORBIDDEN` | Valid JWT but insufficient role or org mismatch |
| 404 | `NOT_FOUND` | Case/user/policy/notification not found |
| 409 | `CONFLICT` | Duplicate case init (NEW), self-claim race, case not in expected status |
| 422 | `BUSINESS_BLOCKED` | S3 validation failure, policy validation failure |
| 500 | `INTERNAL_ERROR` | Unhandled exception (log full error, return safe message) |

### Endpoints Not Fully Specified in Appendix B (Proposed Contracts)

These endpoints appear in Appendix A but lack detailed contracts in Appendix B:

| Endpoint | Proposed Request | Proposed Response 200 |
|----------|-----------------|----------------------|
| `GET /portal/dashboard` | — (query params: none) | `{ caseCounts: { total, byStatus: {} }, priorityDistribution: [], recentActivity: [] }` |
| `GET /portal/cases` | PaginatedRequest + `?status=&q=&orgId=` | `PaginatedResponse<CaseListItem>` where CaseListItem = `{ caseId, orgId, caseType, status, assignedTo, createdAt, updatedAt }` |
| `GET /portal/cases/{caseId}` | — | `{ case, documents[], aiSummary, extractedFields[], ruleEvaluations[], decisions[], notes[] }` |
| `POST /portal/cases/{caseId}/assign` | `{ assignTo: userId }` | `{ caseId, assignedTo }` |
| `POST /portal/cases/{caseId}/reassign` | `{ assignTo: userId }` | `{ caseId, assignedTo }` |
| `POST /portal/cases/{caseId}/email/draft` | `{ context?: string }` | `{ draftId, subject, body, generatedAt }` |
| `GET /portal/notifications` | PaginatedRequest + `?status=read\|unread` | `PaginatedResponse<Notification>` |
| `PUT /portal/settings/notifications` | `{ preferences: { [notificationType]: boolean } }` | `{ updated: true }` |

---

## 5) Database Implementation Plan

### Aurora Migration Order

| Version | Description | Dependencies |
|---------|-------------|-------------|
| V001 | `organisations`, `case_types` | None |
| V002 | `policies`, `policy_documents`, `policy_extraction_fields`, `policy_rules`, `policy_fairness_constraints` | V001 |
| V003 | `cases`, `case_documents`, `extracted_case_data` | V001, V002 |
| V004 | `agent_executions`, `rule_evaluations` | V003 |
| V005 | `case_decisions`, `case_notes`, `audit_logs` | V003 |
| V006 | `case_ai_summaries` | V003 |
| V007 | `notification_email_deliveries` (optional) | None |
| V008 | All indexes (Appendix D.3) | V003–V006 |
| V009 | Immutability grants: `REVOKE UPDATE, DELETE ON audit_logs, case_notes, case_decisions, case_ai_summaries FROM app_role` | V005, V006 |

### DynamoDB Table Definitions

| Table | PK | SK | GSI | TTL | Notes |
|-------|----|----|-----|-----|-------|
| `case_runtime_state` | `case_id` | — | — | `lock_expiry` (for stale lock auto-cleanup) | On-demand, KMS, PITR |
| `user_notifications` | `notification_id` | — | `user_notifications_by_user` (PK: user_id, SK: created_at DESC) | Optional: 90 days on `created_at` | On-demand, KMS, PITR |
| `notification_preferences` | `user_id` | — | — | — | On-demand, KMS, PITR |

### Seed Data Strategy

| Table | Seed data | Purpose |
|-------|-----------|---------|
| `organisations` | 1 org: `council-a`, status=active | Required for all operations |
| `case_types` | 1 type: `hardship-fund` linked to council-a | Required for intake |
| `policies` | 1 active policy (v1) with policy_documents (passport=mandatory, payslip=mandatory with lookback=3), policy_rules (income < threshold), policy_extraction_fields, policy_fairness_constraints | Required for AI pipeline testing |
| Cognito | 3 test users: 1 CW, 1 MGR, 1 ADM with appropriate role claims | Required for portal testing |

### Rollback Considerations

- Each migration MUST be paired with a down-migration script.
- Indexes can be dropped without data loss.
- Immutability grants can be reverted with `GRANT`.
- Table drops should only happen in dev; tst/prd use column-add-only approach.
- Aurora PITR provides point-in-time recovery for emergency rollback.

---

## 6) AgentCore Tool Contracts

### Tool #1 – Document Validation

| Aspect | Detail |
|--------|--------|
| **Purpose** | Validate technical correctness of uploaded documents against policy requirements |
| **Inputs** | `caseId`, `correlationId`, S3 document paths (from `case_documents`), policy rules (from `policy_documents`) |
| **Pre-condition** | Aurora `cases.status` = `INTAKE_VALIDATED` |
| **Logic** | For each doc: Textract DetectDocumentText → verify readability, page count, format consistency. Cross-reference against `policy_documents` for completeness. |
| **Outputs** | Per-document: valid/invalid + metadata (page count, readability score, format) |
| **Aurora writes** | UPDATE `cases.status` → `DOCS_TECHNICALLY_VALIDATED`, UPDATE `cases.updated_at` |
| **DynamoDB writes** | UPDATE `case_runtime_state`: status, current_stage, last_tool_executed, updated_at |
| **agent_executions** | `tool_name=document_validation`, `tool_outcome=SUCCESS/FAILURE/BLOCKED`, timestamps, correlation_id, error fields if applicable |
| **Failure: infra** | Textract timeout/throttle → outcome=FAILURE, retry with backoff |
| **Failure: business** | Unreadable/corrupt document → outcome=BLOCKED, `cases.status`=BLOCKED |
| **Idempotency** | If status already `DOCS_TECHNICALLY_VALIDATED` or later: no-op, return success |
| **Log fields** | correlationId, caseId, toolName, documentCount, validCount, invalidCount, duration |

### Tool #2 – Data Extraction

| Aspect | Detail |
|--------|--------|
| **Purpose** | Extract structured fields from validated documents |
| **Inputs** | caseId, correlationId, validated S3 docs, `policy_extraction_fields` |
| **Pre-condition** | Aurora `cases.status` = `DOCS_TECHNICALLY_VALIDATED` |
| **Logic** | Textract AnalyzeDocument → extract fields per `policy_extraction_fields`. Map to field_name/value/confidence. |
| **Outputs** | Per-field: field_name, value, confidence_score |
| **Aurora writes** | INSERT `extracted_case_data` rows; UPDATE `cases.status` → `DATA_EXTRACTED` |
| **Failure: business** | Required field extraction fails (confidence below threshold) → BLOCKED |
| **Idempotency** | If status already `DATA_EXTRACTED` or later: no-op. On re-run from BLOCKED: delete prior extracted data, re-extract. |

### Tool #3 – Policy Evaluation

| Aspect | Detail |
|--------|--------|
| **Purpose** | Evaluate extracted data against policy rules |
| **Inputs** | caseId, correlationId, extracted_case_data, `policy_rules`, `policy_fairness_constraints` |
| **Pre-condition** | Aurora `cases.status` = `DATA_EXTRACTED` |
| **Logic** | For each `policy_rules` row: compare extracted field value against operator+comparison_value. Record pass/fail/conditional with explanation. Verify no fairness constraint violations. Deterministic evaluation order per §5.6. |
| **Outputs** | Per-rule: result (pass/fail/conditional), explanation |
| **Aurora writes** | INSERT `rule_evaluations` rows; UPDATE `cases.status` → `POLICY_VALIDATED` |
| **Failure: business** | Critical rule cannot be evaluated (missing data) → BLOCKED |
| **Idempotency** | If status already `POLICY_VALIDATED` or later: no-op. Re-run: delete prior rule_evaluations, re-evaluate. |

### Tool #4 – Case Summary & Recommendation

| Aspect | Detail |
|--------|--------|
| **Purpose** | Synthesise human-readable summary, recommendation, and risk assessment |
| **Inputs** | caseId, correlationId, extracted_case_data, rule_evaluations, policy context |
| **Pre-condition** | Aurora `cases.status` = `POLICY_VALIDATED` |
| **Logic** | Bedrock LLM reasoning: generate summary_text, recommendation (APPROVE/DECLINE/ESCALATE/REVIEW), risk_assessment JSONB (risk_level, risk_factors[], mitigations[]), confidence_score. Policy data read from Aurora (never from prompt). |
| **Outputs** | Summary text, recommendation, structured risk assessment, confidence score |
| **Aurora writes** | INSERT `case_ai_summaries` row; UPDATE `cases.status` → `SUMMARY_READY` |
| **Failure: infra** | Bedrock timeout/throttle → FAILURE, retry |
| **Idempotency** | If status already `SUMMARY_READY` or later: no-op. Re-run: creates new summary (UI shows most recent by created_at). |
| **Log fields** | correlationId, caseId, recommendation, riskLevel, confidenceScore, bedrockLatencyMs |

### Tool #5 – Mark Ready for Review

| Aspect | Detail |
|--------|--------|
| **Purpose** | Transition case to caseworker review; emit readiness event; release lock |
| **Inputs** | caseId, correlationId, summaryId (from Tool #4 output) |
| **Pre-condition** | Aurora `cases.status` = `SUMMARY_READY` |
| **Logic** | 1. Verify pre-condition (SUMMARY_READY). 2. UPDATE Aurora status → READY_FOR_CASEWORKER_REVIEW. 3. UPDATE DynamoDB stage/status/updated_at. 4. Emit CASE_AI_READY_FOR_REVIEW via EventBridge (with summaryId). 5. Release DynamoDB lock (lock_owner=NULL, lock_expiry=NULL). |
| **Aurora writes** | UPDATE `cases.status` → `READY_FOR_CASEWORKER_REVIEW`, UPDATE `cases.updated_at` |
| **DynamoDB writes** | UPDATE status, stage, lock_owner=NULL, lock_expiry=NULL, updated_at |
| **EventBridge** | Emit `CASE_AI_READY_FOR_REVIEW` with all §12.1 mandatory fields + summaryId |
| **agent_executions** | tool_name=mark_ready, full §5.9.3 fields |
| **Strict idempotency** | If status already READY_FOR_CASEWORKER_REVIEW: NO writes, NO event, NO lock change. Return success. This is the strongest idempotency guarantee in the pipeline. |
| **Log fields** | correlationId, caseId, priorStatus, newStatus, eventEmitted (bool), lockReleased (bool) |

---

## 7) Frontend Delivery Plan

### Route Map

| Route | Screen | Rendering | Auth | Role guard |
|-------|--------|-----------|------|------------|
| `/login` | Login | SSG | Public | — |
| `/` | Homepage dashboard | ISR 60s | Required | CW/MGR/ADM |
| `/cases` | Case management list | ISR 30s | Required | CW/MGR/ADM |
| `/cases/[caseId]` | Individual case detail | SSR + SWR | Required | CW(own)/MGR/ADM |
| `/notifications` | Notifications | Client fetch | Required | CW/MGR/ADM |
| `/settings` | Settings | SSG shell + client | Required | CW/MGR/ADM |
| `/settings/faq` | FAQ | SSG | Required | CW/MGR/ADM |
| `/settings/ai-guide` | AI Guide | SSG | Required | CW/MGR/ADM |
| `/escalations` | Escalated cases | ISR 30s | Required | MGR |
| `/admin/users` | User management | SSR + client | Required | ADM |
| `/admin/policies` | Policy management | SSR + client | Required | ADM |

### State Management

- **SWR** (stale-while-revalidate) for all data fetching: automatic caching, revalidation, error retry.
- Global SWR config: `revalidateOnFocus: true`, `dedupingInterval: 2000`.
- Mutation pattern: `useSWRMutation` → on success → `mutate()` to invalidate cache → `router.refresh()` for ISR pages.
- Auth state: Cognito Amplify Auth SDK; token stored in httpOnly cookie (Amplify default).
- Role state: decoded from JWT `custom:role` claim; cached in React context.

### Auth Session Flow

1. User navigates to `/` → middleware checks for valid session cookie.
2. No session → redirect to `/login` → Cognito hosted UI (OIDC/SAML).
3. IdP authentication → callback to Amplify → session cookie set.
4. Middleware extracts JWT claims → injects role into request context.
5. Token refresh: Amplify Auth handles silent refresh before expiry.
6. Session expiry (24h refresh token): redirect to login.

### RBAC UI Guards

```typescript
// Hook usage
const { role, orgId } = useAuth();

// Component guard
<RoleGuard allowed={['MANAGER', 'ADMINISTRATOR']}>
  <AssignButton />
</RoleGuard>

// Route guard (middleware.ts)
if (pathname.startsWith('/admin') && role !== 'ADMINISTRATOR') {
  return NextResponse.redirect('/');
}
```

### Case Detail Page Component Breakdown

```
CaseDetailPage
├── CaseHeader (caseId, status badge, assignedTo, actions dropdown)
├── TabNavigation (Overview | Documents | AI Analysis | Notes | Decisions)
├── Tab: Overview
│   ├── ApplicantInfoCard
│   └── CaseTimelineCard
├── Tab: Documents
│   └── DocumentList (presigned URL links, type, version, upload date)
├── Tab: AI Analysis
│   ├── AISummaryCard (summary_text)
│   ├── RiskAssessmentCard (risk_level badge, risk_factors list, mitigations)
│   ├── RecommendationBadge (APPROVE/DECLINE/ESCALATE/REVIEW + confidence)
│   ├── ExtractedFieldsTable (field, value, confidence bar)
│   └── RuleEvaluationsTable (rule description, result badge, explanation)
├── Tab: Notes
│   ├── NotesList (reverse chronological; performer, timestamp, text)
│   └── AddNoteForm (textarea + submit)
├── Tab: Decisions
│   └── DecisionHistoryList (reverse chronological; decision, performer, justification, timestamp)
├── DecisionPanel (sticky bottom bar when case is decidable)
│   ├── ApproveButton, DeclineButton, PendingButton, EscalateButton
│   ├── JustificationTextarea
│   └── SubmitDecisionButton
└── AIEmailModal (draft → review → confirm send)
```

### Amplify Environment Variables

| Variable | Example | Purpose |
|----------|---------|---------|
| `NEXT_PUBLIC_API_BASE_URL` | `https://api-faststart-dev.example.gov.uk` | API Gateway base URL |
| `NEXT_PUBLIC_COGNITO_USER_POOL_ID` | `eu-west-2_abc123` | Cognito User Pool |
| `NEXT_PUBLIC_COGNITO_CLIENT_ID` | `xyz789` | Cognito app client |
| `NEXT_PUBLIC_COGNITO_DOMAIN` | `auth-faststart-dev.example.gov.uk` | Cognito hosted UI domain |
| `ISR_HOMEPAGE_REVALIDATE_SECONDS` | `60` | Homepage ISR interval |
| `ISR_CASE_LIST_REVALIDATE_SECONDS` | `30` | Case list ISR interval |

### Amplify Branch Strategy

| Branch | Amplify env | Domain | Auto-deploy |
|--------|------------|--------|-------------|
| `develop` | dev | `faststart-dev.<domain>` | Yes (on push) |
| `release/*` | tst | `faststart-tst.<domain>` | Manual approve |
| `main` | prd | `faststart-prd.<domain>` | Manual approve |
| PR branches | preview | `pr-<n>.faststart-dev.<domain>` | Yes (Amplify previews) |

---

## 8) Operations / SRE Runbook Starter Pack

### Runbook 1: Stuck Case Diagnosis

1. Query Aurora: `SELECT case_id, status, updated_at, NOW()-updated_at AS age FROM cases WHERE status NOT IN ('APPROVED','DECLINED') AND NOW()-updated_at > interval '<SLA_THRESHOLD>'`.
2. For each stuck case, query `agent_executions` for last tool + outcome.
3. Check DynamoDB `case_runtime_state` for `lock_owner` (stale lock?).
4. Classify: STUCK (no error, no progress) | FAILED (error code) | BLOCKED (business).
5. Action per classification (see below).

### Runbook 2: Replay / Re-drive

1. Verify case status in Aurora (authoritative checkpoint).
2. If stuck at INTAKE_VALIDATED (no tool has run): re-emit `CASE_INTAKE_VALIDATED` event via EventBridge PutEvents with original caseId + correlationId.
3. If stuck mid-pipeline: invoke resume Lambda with caseId. Lambda reads Aurora status and resumes from the next tool.
4. Verify: new `agent_executions` rows appear; status advances; no duplicate data.

### Runbook 3: DLQ Triage

1. Check DLQ depth alarm. Query DLQ for messages.
2. Inspect message body: extract `caseId`, `correlationId`, `detail-type`.
3. Check CloudWatch logs for the correlationId to find root cause.
4. If transient (throttling, timeout): re-drive message from DLQ to source queue.
5. If permanent (schema invalid, missing case): log, alert, and archive.

### Runbook 4: Event Schema Validation Failures

1. Consumer DLQ receives event that failed schema validation.
2. Extract event payload. Validate against registered JSON Schema manually.
3. Identify which field failed (missing? wrong type?).
4. Root cause: producer bug or schema version mismatch.
5. Fix producer, deploy, re-emit corrected event for affected case.

### Runbook 5: Bedrock/Textract Throttling

1. CloudWatch alarm: tool timeout/duration anomaly.
2. Check Bedrock/Textract service health dashboard.
3. If throttled: reduce Lambda reserved concurrency for AI pipeline temporarily.
4. If service degradation: wait for recovery; stuck cases will auto-resume on retry.
5. If persistent: request quota increase via AWS Support.

### Runbook 6: Stale Lock Handling

1. Query DynamoDB: `case_runtime_state` where `lock_owner IS NOT NULL` and `lock_expiry < NOW()`.
2. Verify the owning Lambda/AgentCore session is no longer running (check CloudWatch for the session ID).
3. Clear lock: `UPDATE case_runtime_state SET lock_owner=NULL, lock_expiry=NULL WHERE case_id=:caseId`.
4. Re-drive the case (Runbook 2).

### Alarm-to-Owner Matrix Template

| Alarm | Owner | Escalation | Runbook |
|-------|-------|------------|---------|
| Tool error-rate spike | AI team | → Platform team (15 min) | #5 |
| DLQ growth | Platform team | → Tech lead (30 min) | #3 |
| API 5xx > 1% | Backend team | → Tech lead (15 min) | CloudWatch logs |
| Lambda error > 5% | Backend team | → Platform team (15 min) | CloudWatch logs |
| Aurora CPU > 80% | Platform team | → DBA (30 min) | Scale ACU |
| EventBridge dispatch failure | Platform team | → Tech lead (15 min) | #4 |
| Stuck case SLA breach | Ops team | → Tech lead (1 hour) | #1 |

### Dashboard Widget List

1. Case pipeline throughput (cases/hour by stage)
2. Stage duration percentiles (p50, p95, p99 per tool)
3. Tool error rate (per tool, 5-min windows)
4. DLQ depth (per queue)
5. API latency percentiles (p50, p95, p99)
6. API error rate (4xx, 5xx)
7. Active locks count
8. Cases by status (stacked bar)
9. Aurora ACU usage
10. Lambda concurrent executions

---

## 9) Test Strategy

### Layer Summary

| Layer | Scope | Tools | Run when |
|-------|-------|-------|----------|
| **Unit** | Functions, services, validators, state machine | Jest (TS) | Every commit |
| **Contract** | EventBridge schemas, API request/response shapes | AJV (JSON Schema), jest | Every commit |
| **Integration** | Lambda + Aurora + DynamoDB + S3 (deployed to dev) | Jest + AWS SDK | Merge to develop |
| **E2E** | Full user journeys through portal | Playwright | Merge to release |
| **Load** | Throughput, latency, concurrency | k6 or Artillery | Pre-release |
| **Failure injection** | DLQ, retries, locks, duplicates | Custom scripts + AWS FIS | Pre-release |

### Top 15 Must-Pass Scenarios Before Production

| # | Scenario | Type | Covers |
|---|----------|------|--------|
| 1 | Init → upload → complete → event emitted with valid schema | Integration | Intake pipeline E2E |
| 2 | Full AI pipeline: INTAKE_VALIDATED → READY_FOR_CASEWORKER_REVIEW | Integration | All 5 tools |
| 3 | Tool #5 repeated invocation produces no duplicate writes or events | Integration | Idempotency |
| 4 | Caseworker login → view case list → view case detail → approve | E2E | Portal MVP journey |
| 5 | Self-claim race: two concurrent claims → one succeeds, one gets 409 | Integration | Concurrency |
| 6 | Escalate → manager resolves → original decision preserved | E2E | Escalation chain |
| 7 | Add note → note appears in list → note is immutable (no UPDATE/DELETE) | Integration | Append-only |
| 8 | EventBridge event with missing required field → routed to DLQ | Contract | Schema validation |
| 9 | Correlation ID traceable from init → event → agent_executions → logs | Integration | Observability |
| 10 | DLQ message after 3 SQS failures; alarm fires | Integration | Failure handling |
| 11 | Stale lock expires → case resumable | Integration | Lock semantics |
| 12 | Policy upload with contradictory rules → rejected | Integration | Policy validation |
| 13 | Role-based visibility: CW cannot see other CW's assigned cases | E2E | RBAC |
| 14 | Duplicate init (NEW) for active case → 409 | Integration | Idempotency |
| 15 | 100 concurrent intakes complete within 2s p95 | Load | Performance |

---

## 10) Proposed Clarifications

These are the only items not fully resolved by the spec that could block implementation. Each has a proposed default.

| # | Item | Why it matters | Proposed default | Impact if deferred |
|---|------|---------------|-----------------|-------------------|
| 1 | **Cognito user ↔ Aurora user sync model** | Spec defines Cognito for auth and Aurora for user data (assigned_to FK). Need to define how users are linked. | Aurora `users` table (not in §7.1) with `user_id` matching Cognito sub. Admin create writes both. Login syncs display_name on first access. | Cannot implement user management or assignment without this. **Must resolve Phase 0.** |
| 2 | **Stage SLA threshold numeric values** | §5.9.2 requires thresholds but defers values (§11.8). Needed for stuck-case detection alarms. | Validation: 5 min, Extraction: 10 min, Policy eval: 5 min, Summary: 10 min, Mark ready: 2 min. Configurable via SSM. | Alarms cannot fire without values. Defer to Phase 5 if needed, but configure placeholders in Phase 0. |
| 3 | **`users` table DDL** | §7.1 defines case-related tables but no `users` table. Admin user management (§6.3) and `assigned_to` FK need one. | Add Aurora `users` table: `user_id (PK), cognito_sub, email, display_name, role, organisation_id (FK), status (active/inactive/deleted), created_at, updated_at, deleted_at`. Migration V001a. | Blocks user management and assignment. **Must resolve Phase 0.** |
| 4 | **Bedrock AgentCore SDK choice** | §11.6 leaves open whether to use Strands SDK or native AgentCore API. Affects implementation of orchestrator. | Start with AWS Bedrock AgentCore native API. If SDK limitations emerge, fall back to Strands Agents SDK. Decision by end of Phase 2, Week 8. | Low impact if deferred; either path is spec-compliant. |
