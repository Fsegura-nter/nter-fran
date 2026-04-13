# nter-nexus:poc

Main pipeline orchestrator. Processes a PRD document from Google Drive and produces structured tasks for Treyit.

Run after `nter-nexus:init`. Requires an initialized `.nter-nexus/config.json`.

---

## Pre-flight

### P1: Read config

```bash
cat .nter-nexus/config.json
```

If missing: stop. Output:
```
nter-nexus is not initialized for this project.
Run /nter-nexus:init first.
```

Extract: `project`, `back`, `front`, `stack_backend`, `stack_frontend`, `drive_folder_id`, `treyit_board_id`.

Set `STATE_DIR` = `.nter-nexus/state/`

### P2: Request Treyit token

```
Paste your Treyit auth token (it won't be stored anywhere):
```

Wait for token. If blank: stop.

Store in memory as `TREYIT_TOKEN` for this session only.

### P3: Validate Treyit connection

```bash
curl -s -H "Authorization: Bearer {TREYIT_TOKEN}" https://yiyit.nter.es/api/employees/logged
```

If response lacks `id`: stop. `Invalid token — paste a fresh one and re-run.`

Extract `employeeId` from response.

### P4: Select document from Drive

If Drive MCP is configured:
```
Use the gdrive MCP to list files in folder {drive_folder_id}.
Show the list as:
  [N] {filename} — last modified {date}
```

Ask:
```
Which document do you want to process? Enter the number.
```

If Drive MCP is not configured:
```
Drive MCP is not configured. Please paste the document content below, then type END on a new line.
```

Read the document content (Google Doc text or pasted content). Store as `PRD_CONTENT`.

If document is a Google Doc via Drive MCP:
```
Use the gdrive MCP to read the content of file {file_id}.
```

Store document name as `PRD_NAME`.

### P5: Warm/cold start check

```bash
cat {STATE_DIR}/snapshot-meta.json 2>/dev/null
```

```bash
BACK_NOW=$(cd {back} && git rev-parse HEAD 2>/dev/null || echo "none")
FRONT_NOW=$(cd {front} && git rev-parse HEAD 2>/dev/null || echo "none")
```

Compare `back_commit` and `front_commit` from snapshot vs. current SHAs:

- Both match → **WARM START**: reuse `00-arco-scan.md` and `00-estela-scan.md`. Skip parallel scan agents.
- Either differs or snapshot missing → **COLD START**: run Arco-Scan and Estela-Scan in parallel.

Report:
```
Start mode: {WARM — scan artifacts reused | COLD — running fresh scan}
```

### P6: Write workspace.md

Write `{STATE_DIR}/00-workspace.md` before launching any agents:
```
WORKSPACE: {current directory absolute path}
BACK:      {absolute back path}
FRONT:     {absolute front path}
STACK_BACKEND:  {stack_backend}
STACK_FRONTEND: {stack_frontend}
STATE_DIR: {absolute STATE_DIR path}
```

All agents read their STATE_DIR from this file. It must exist before Phase 1 starts.

---

## Phase 1 — Parallel analysis

Run all of the following in parallel (as subagents or inline, as fast as possible):

### 1A: Clara — PO review

**Input:** PRD_CONTENT, STATE_DIR

Invoke Clara agent from `agents/clara.md`.

Clara reads the PRD, raises functional risks, proposes questions, produces READINESS signal.

Output: `{STATE_DIR}/01-clara-review.md`

### 1B: Arco-Scan — Codebase scan (COLD only)

**Skip if WARM START** — reuse existing `00-arco-scan.md`.

**Input:** `back` path, `front` path, STATE_DIR

Invoke Arco-Scan agent from `agents/arco-scan.md`.

Output: `{STATE_DIR}/00-arco-scan.md`

### 1C: Estela-Scan — Design token scan (COLD only)

**Skip if WARM START** — reuse existing `00-estela-scan.md`.

**Input:** `front` path, STATE_DIR

Invoke Estela-Scan agent from `agents/estela-scan.md`.

Output: `{STATE_DIR}/00-estela-scan.md`

### 1D: Canvas-Scan — Design asset extraction

**Input:** PRD_CONTENT, STATE_DIR

Invoke Canvas-Scan agent from `agents/canvas-scan.md`.

Canvas-Scan extracts Figma/Excalidraw/draw.io URLs from the PRD and captures design assets.

Output: `{STATE_DIR}/00-design.md` and images in `{STATE_DIR}/design/`

---

## Phase 1 gate: check all outputs

```bash
ls {STATE_DIR}/01-clara-review.md {STATE_DIR}/00-arco-scan.md {STATE_DIR}/00-estela-scan.md {STATE_DIR}/00-design.md 2>/dev/null
```

Verify each file ends with its signature line (`Firmado: Clara ✓`, etc.). If any is missing or unsigned: stop and report which agent failed.

### Clara READINESS check

Read `01-clara-review.md` and extract `READINESS`:

- `✅` — proceed
- `⚠️` — show warnings to user, ask: `Continue anyway? (yes / no)`
- `❌` — stop. Show what is missing. Do not proceed until user resolves.
- `🚨` — invoke Marco immediately.

---

## Phase 2 — Architecture (Arco)

Invoke Arco agent from `agents/arco.md` as subagent.

Arco reads:
- `00-workspace.md`
- `00-arco-scan.md`
- `01-clara-review.md`

Arco produces the ADR with: domain model, API contract proposal, module mapping, cross-cutting concerns.

Output: `{STATE_DIR}/02-arco-adr.md`

Verify signature `Firmado: Arco ✓`.

If Arco escalates to Marco: pause pipeline, invoke Marco inline, resume after Marco resolves.

---

## Phase 3 — Backend breakdown (Forge)

Invoke Forge agent from `agents/forge.md` as subagent.

Forge reads:
- `00-workspace.md`
- `02-arco-adr.md`

Forge produces: full backend task breakdown + Contract Lock.

Outputs:
- `{STATE_DIR}/03-forge-breakdown.md`
- `{STATE_DIR}/03-contract-lock.md`

Verify both files exist and are signed (`Firmado: Forge ✓`).

**CONTRACT LOCK GATE**: If `03-contract-lock.md` is missing or unsigned, pipeline stops here. Report:
```
BLOCKED: Contract Lock not produced by Forge.
Pipeline cannot continue until Forge completes the contract.
```

---

## Phase 4 — UX/UI validation (Estela)

Invoke Estela agent from `agents/estela.md` as subagent.

Estela reads:
- `00-workspace.md`
- `00-estela-scan.md`
- `00-design.md`
- `01-clara-review.md`

Estela produces: design validation + DESIGN ASSET MAP.

Output: `{STATE_DIR}/05-estela-validation.md`

Verify signature `Firmado: Estela ✓`.

If Estela escalates to Marco: pause, invoke Marco, resume after resolution.

---

## Phase 5 — Frontend breakdown (Pixel)

Invoke Pixel agent from `agents/pixel.md` as subagent.

Pixel reads:
- `00-workspace.md`
- `03-contract-lock.md`
- `05-estela-validation.md`

Pixel produces: full frontend task breakdown with design references.

Output: `{STATE_DIR}/04-pixel-breakdown.md`

Verify signature `Firmado: Pixel ✓`.

---

## Phase 6 — QA scenarios (Finally)

Invoke Finally agent from `agents/finally.md` inline (has access to full context).

Finally reads:
- `00-workspace.md`
- `01-clara-review.md`
- `03-contract-lock.md`
- `04-pixel-breakdown.md`
- `05-estela-validation.md`

Finally produces: E2E scenarios + Treyit QA task definitions.

Output: `{STATE_DIR}/06-finally-qa.md`

Verify signature `Firmado: Finally ✓`.

---

## Phase 7 — Treyit board setup (if needed)

If `treyit_board_id` is null in config:

```bash
curl -s -H "Authorization: Bearer {TREYIT_TOKEN}" https://yiyit.nter.es/api/board
```

List boards. Ask user:
```
Which board should tasks be created in? (paste the board ID)
```

Then fetch columns:
```bash
curl -s -H "Authorization: Bearer {TREYIT_TOKEN}" \
  "https://yiyit.nter.es/api/board-card-status?boardCardId={board_id}"
```

Show columns. Ask:
```
Which column (status ID) should new tasks land in?
```

Save `board_id` and `status_id` for task creation. Update `config.json`.

---

## Phase 8 — Task preview and confirmation

Show a summary of all tasks to be created:

```
Tasks to create in Treyit:

Backend (Forge):
  BACK-1: {title}
  BACK-2: {title}
  ...

Frontend (Pixel):
  FRONT-1: {title} → parent: BACK-{N}
  FRONT-2: {title} → parent: BACK-{N}
  ...

QA (Finally):
  QA-RF{N}: {title}
  ...

Total: {N} tasks

Publish to Treyit? (yes / no)
```

Do not create anything until the user confirms.

---

## Phase 9 — Task creation

Create in order:

### 9A: Backend tasks (Forge)

For each `BACK-N` block in `03-forge-breakdown.md`:

```bash
cat > .nter-nexus/state/treyit_card.json << 'EOF'
{
  "title": "{SCOPE of BACK-N}",
  "description": "{HTML description — see format below}",
  "status": {status_id},
  "employee": {employeeId}
}
EOF

curl -s -X POST \
  -H "Authorization: Bearer {TREYIT_TOKEN}" \
  -H "Content-Type: application/json" \
  --data @.nter-nexus/state/treyit_card.json \
  https://yiyit.nter.es/api/board-cards
```

Extract `id` from response → store as `back_ids[N]`.

**Back task description HTML:**
```html
<p>{1-sentence intro: what this task implements and why}</p>
<br>
<p><b>Contexto funcional</b><br>
{FUNCTIONAL CONTEXT from Forge's BACK-N block}</p>
<br>
<p><b>Detalle técnico backend</b><br>
{entity, DTO, service, controller breakdown from Forge}</p>
<br>
<p><b>Contract Lock</b><br>
{relevant endpoints from 03-contract-lock.md for this task}</p>
{only in BACK-1:}
<br>
<p><b>Riesgos identificados</b><br>
{risks from Clara's review}</p>
<br>
<p><i>Generado por IA — {today's date}</i></p>
```

### 9B: Frontend tasks (Pixel)

For each `FRONT-N` block in `04-pixel-breakdown.md`:

Resolve `PARENT: BACK-{N}` → look up `back_ids[N]`.

Prepare `DESIGN_IMAGE_BASE64` if `DESIGN_REF` is a local PNG:
```bash
base64 "{DESIGN_REF}" 2>/dev/null | tr -d '\n\r'
```

```bash
cat > .nter-nexus/state/treyit_card.json << 'EOF'
{
  "title": "{title from FRONT-N}",
  "description": "{HTML description}",
  "status": {status_id},
  "employee": {employeeId},
  "parent": {back_ids[N]}
}
EOF

curl -s -X POST \
  -H "Authorization: Bearer {TREYIT_TOKEN}" \
  -H "Content-Type: application/json" \
  --data @.nter-nexus/state/treyit_card.json \
  https://yiyit.nter.es/api/board-cards
```

**Front task description HTML:**
```html
<p>{1-sentence intro: what the user sees and what this task does}</p>
<br>
<p><b>Contexto funcional</b><br>
{functional context from Pixel's FRONT-N}</p>
<br>
<p><b>Qué incluye esta tarea</b><br>
{list of artifacts: files, routes, endpoints consumed}<br>
{one item per line with <br>}</p>
{if estela notes exist:}
<br>
<p><b>Validación UX/UI</b><br>
{Estela notes relevant to this task}</p>
<br>
<p><b>Diseño de referencia</b><br>
{if DESIGN_URL != "none": <a href="{DESIGN_URL}">{DESIGN_LABEL}</a><br>}
{if local PNG: <img src="data:image/png;base64,{base64}" style="max-width:100%">}
{if generated HTML mockup: inline the mockup HTML content}</p>
<br>
<p><i>Generado por IA — {today's date}</i></p>
```

### 9C: QA tasks (Finally)

For each `TREYIT QA TASK — RF{N}` block in `06-finally-qa.md`:

Apply labels if Treyit supports them:
```bash
# Get or create tags
curl -s -H "Authorization: Bearer {TREYIT_TOKEN}" \
  "https://yiyit.nter.es/api/board-tags?boardId={board_id}"
```

Create each QA task using the HTML description from Finally's output. No parent.

---

## Phase 10 — Snapshot update and summary

Update snapshot on successful completion:

```bash
BACK_NOW=$(cd {back} && git rev-parse HEAD 2>/dev/null || echo "none")
FRONT_NOW=$(cd {front} && git rev-parse HEAD 2>/dev/null || echo "none")

cat > {STATE_DIR}/snapshot-meta.json << EOF
{
  "back_commit": "${BACK_NOW}",
  "front_commit": "${FRONT_NOW}",
  "scanned_at": "$(node -e "process.stdout.write(new Date().toISOString())" 2>/dev/null || date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
```

Print final summary:
```
Pipeline complete ✓

Document: {PRD_NAME}
Tasks created:
  Backend:  {N} tasks
  Frontend: {N} tasks
  QA:       {N} tasks

Pipeline artifacts saved to: {STATE_DIR}

Scan snapshot updated — next run will use warm start if no new commits.
```

---

## Marco escalation (any phase)

If any agent invokes Marco, pause the pipeline and display:

```
ESCALATION — Marco
Reason:  {reason from agent}
Options: {options presented}
Blocked: {agent name}

Answer Marco's question to continue the pipeline.
```

After the user responds, resume from the point where the pipeline was paused.

---

## Rules

- Never create Treyit tasks without user confirmation.
- Never store the Treyit token anywhere — memory only.
- The Contract Lock is a hard gate. Pixel does not run without it.
- Warm start skips Arco-Scan and Estela-Scan only. Clara and Canvas-Scan always run fresh.
- All agents write their output to `{STATE_DIR}/` — never to `/tmp/` or other locations.
- Do not invent task content. Every task field maps to an agent output.
- If an agent signature is missing, treat that agent as failed and stop.
