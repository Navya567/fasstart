# Export AgentCore Diagrams to PNG (Downloadable Images)

**Pre-generated PNGs:** Five diagram images for the AgentCore design have been generated and saved in the project **assets** folder:

- `workflow-agentcore.png`
- `workflow-detail-agentcore.png`
- `aws-architecture-agentcore.png`
- `database-workflow-agentcore.png`
- `government-triage-workflow-agentcore.png`

If an `assets` folder appears in your workspace, open it to copy or download these PNGs. Otherwise they may be under Cursor’s project assets path (e.g. `.cursor/projects/.../assets/`); you can copy them into `requirements/diagrams/agentcore/images/` for use in docs.

---

The Mermaid source files in this folder can also be exported to PNG so you get **downloadable images** (e.g. exact Mermaid rendering) as below.

## Option 1: Mermaid CLI (recommended)

From the **project root** (or this folder), run:

```bash
# One-time: install mermaid-cli (if not global)
npm install -g @mermaid-js/mermaid-cli

# Create output folder
mkdir -p requirements/diagrams/agentcore/images

# Export each diagram to PNG
mmdc -i requirements/diagrams/agentcore/workflow-agentcore.mmd -o requirements/diagrams/agentcore/images/workflow-agentcore.png -b transparent
mmdc -i requirements/diagrams/agentcore/workflow-detail-agentcore.mmd -o requirements/diagrams/agentcore/images/workflow-detail-agentcore.png -b transparent
mmdc -i requirements/diagrams/agentcore/aws-architecture-agentcore.mmd -o requirements/diagrams/agentcore/images/aws-architecture-agentcore.png -b transparent
mmdc -i requirements/diagrams/agentcore/database-workflow-agentcore.mmd -o requirements/diagrams/agentcore/images/database-workflow-agentcore.png -b transparent
mmdc -i requirements/diagrams/agentcore/government-triage-workflow-agentcore.mmd -o requirements/diagrams/agentcore/images/government-triage-workflow-agentcore.png -b transparent
```

**Windows PowerShell** (from project root):

```powershell
npx --yes @mermaid-js/mermaid-cli mmdc -i requirements/diagrams/agentcore/workflow-agentcore.mmd -o requirements/diagrams/agentcore/images/workflow-agentcore.png -b transparent
npx --yes @mermaid-js/mermaid-cli mmdc -i requirements/diagrams/agentcore/workflow-detail-agentcore.mmd -o requirements/diagrams/agentcore/images/workflow-detail-agentcore.png -b transparent
npx --yes @mermaid-js/mermaid-cli mmdc -i requirements/diagrams/agentcore/aws-architecture-agentcore.mmd -o requirements/diagrams/agentcore/images/aws-architecture-agentcore.png -b transparent
npx --yes @mermaid-js/mermaid-cli mmdc -i requirements/diagrams/agentcore/database-workflow-agentcore.mmd -o requirements/diagrams/agentcore/images/database-workflow-agentcore.png -b transparent
npx --yes @mermaid-js/mermaid-cli mmdc -i requirements/diagrams/agentcore/government-triage-workflow-agentcore.mmd -o requirements/diagrams/agentcore/images/government-triage-workflow-agentcore.png -b transparent
```

Output will be in `requirements/diagrams/agentcore/images/` as downloadable PNGs.

## Option 2: Open index.html and export

1. Open `requirements/diagrams/agentcore/index.html` in a browser.
2. Right-click each diagram → “Save image as…” (or use browser screenshot / Print to PDF for the full page).

## Option 3: Mermaid Live Editor

1. Go to https://mermaid.live
2. Paste the contents of each `.mmd` file.
3. Use “Export” → PNG or SVG to download.

---

**Diagram list (AgentCore-first, no Step Functions):**

| File | Output image | Description |
|------|----------------|-------------|
| workflow-agentcore.mmd | workflow-agentcore.png | Conceptual flow: Intake → EventBridge → AgentCore → Tools → Caseworker |
| workflow-detail-agentcore.mmd | workflow-detail-agentcore.png | Swimlane-style: Citizen, Intake, Orchestration, AI Services, Data, Portal, Audit |
| aws-architecture-agentcore.mmd | aws-architecture-agentcore.png | 3 layers: EXTERNAL, API & ORCHESTRATION, DATA & AI |
| database-workflow-agentcore.mmd | database-workflow-agentcore.png | Aurora groups, DynamoDB, S3; AgentCore + Tools |
| government-triage-workflow-agentcore.mmd | government-triage-workflow-agentcore.png | Government case triage end-to-end (AgentCore orchestrator) |
