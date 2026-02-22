# FastStart AgentCore-first Diagrams

Diagrams for the **AgentCore-driven** case assessment flow (no Step Functions for AI sequencing).  
Source: `requirements/agentcore-requirements-full-v1.2.md`.

## Downloadable images (PNG)

Pre-rendered PNGs for the new design (same style as the reference workflow/architecture images) are available:

| Image | Description |
|-------|-------------|
| `workflow-agentcore.png` | Conceptual flow: Intake → EventBridge → AgentCore → Tools → Caseworker |
| `workflow-detail-agentcore.png` | Swimlane: Citizen, Intake, Orchestration (AgentCore), AI Services, Data, Portal, Audit |
| `aws-architecture-agentcore.png` | Three layers: EXTERNAL, API & ORCHESTRATION, DATA & AI |
| `database-workflow-agentcore.png` | Database flow: Aurora groups, DynamoDB, S3; AgentCore + Tools |
| `government-triage-workflow-agentcore.png` | Government case triage end-to-end (AgentCore orchestrator) |

- **Location:** These PNGs are in the project **assets** folder (e.g. `assets/workflow-agentcore.png`, etc.). Open the folder in File Explorer or Cursor to copy or download.
- **Alternative:** To generate PNGs from the Mermaid sources (e.g. into `images/`), see **Export to PNG** below.

## Contents

| File | Description |
|------|-------------|
| `workflow-agentcore.mmd` | Conceptual workflow: Intake → EventBridge → AgentCore → Tools → Caseworker |
| `workflow-detail-agentcore.mmd` | Detailed swimlane: Citizen, Intake, Orchestration, AI Services, Data, Portal, Audit |
| `aws-architecture-agentcore.mmd` | Three-layer AWS view: EXTERNAL, API & ORCHESTRATION, DATA & AI |
| `database-workflow-agentcore.mmd` | Database flow: Aurora groups, DynamoDB, S3; AgentCore + Tools |
| `government-triage-workflow-agentcore.mmd` | Government case triage workflow (AgentCore as orchestrator) |
| `index.html` | Renders all diagrams in the browser (Mermaid); open locally or host to view/export |
| `export-to-png.md` | Commands to export Mermaid sources to PNG (Mermaid CLI, browser, or Mermaid Live) |

## How to view

1. **HTML (easiest):** Open `index.html` in a browser. Diagrams render via Mermaid.js (CDN). You can screenshot or use browser “Print to PDF” to export.
2. **Mermaid files:** Use any Mermaid-compatible viewer (VS Code Mermaid extension, Mermaid Live Editor, or `npx @mermaid-js/mermaid-cli mmdc -i file.mmd -o file.png`) to export PNG/SVG.

## Export to PNG

See **`export-to-png.md`** in this folder for:
- Mermaid CLI commands to export each `.mmd` to `images/*.png`,
- Using `index.html` or Mermaid Live Editor to save PNGs.

## Differences from Step Functions version

- **Orchestration:** EventBridge `CASE_INTAKE_VALIDATED` starts **Bedrock AgentCore** (Case Assessment Agent), not a Step Functions state machine.
- **Sequence:** AgentCore selects and invokes tools (Document Validation → Data Extraction → Policy Evaluation → Case Summary); tools are deterministic (e.g. Lambda).
- **No:** Step Functions, SQS step queues for tech-validation/extraction as the main sequencer.
- **Yes:** Same data stores (Aurora, DynamoDB, S3), same intake (ApplicationInit, ApplicationFinalize), same caseworker portal and decision API.
