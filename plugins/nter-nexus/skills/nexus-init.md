# nter-nexus:init

One-time setup for a project that will use the nter-nexus pipeline. Run this once per repository. Safe to re-run — it overwrites the config, not the pipeline outputs.

---

## What this does

1. Detects backend and frontend paths in the current workspace
2. Configures the Google Drive folder for PRD documents
3. Optionally configures Figma API key
4. Configures Treyit board and columns
5. Writes `.nter-nexus/config.json` (gitignored)
6. Runs a cold scan (Arco-Scan + Estela-Scan) and saves snapshot artifacts
7. Reports what was configured and what is optional/missing

---

## Step 0: Detect workspace

```bash
pwd
ls -la
```

From the current directory, identify:
- Is there a `src/`, `backend/`, `api/`, or similar backend folder?
- Is there a `frontend/`, `client/`, `web/`, or `src/app/` folder?
- Or are back and front in the same repo (monorepo)?

If the structure is ambiguous, ask the user:
```
I found the following directories: {list}
Which is the backend path? (relative or absolute)
Which is the frontend path? (relative or absolute)
```

---

## Step 1: Ask for configuration

Ask these questions upfront in a single message:

```
Setting up nter-nexus for this project. A few things:

1. Backend path: {detected or "please specify"}
2. Frontend path: {detected or "please specify"}
3. Project name (used in task titles): {basename of current dir or specify}
4. Google Drive folder: Using the company-wide folder (1BVZQKF61uvBKIpnFUHRQK4GSzR8r8Eh3)
   — Press Enter to confirm, or paste a different folder ID.
5. Treyit board: Do you have an existing board ID? (or leave blank to skip for now)
6. Figma API key: Optional. Needed only if your team uses Figma for designs.
   — Paste your key, or press Enter to skip.
```

Wait for answers before proceeding.

---

## Step 2: Validate paths

```bash
BACK={backend-path}
FRONT={frontend-path}

# Verify backend
[ -f "$BACK/pom.xml" ] && echo "Backend: Spring Boot (Maven)" && STACK_BACK="spring-maven"
[ -f "$BACK/build.gradle" ] && echo "Backend: Spring Boot (Gradle)" && STACK_BACK="spring-gradle"
[ -f "$BACK/build.gradle.kts" ] && echo "Backend: Spring Boot (Gradle KTS)" && STACK_BACK="spring-gradle-kts"
[ -f "$BACK/package.json" ] && grep -q "@nestjs/core" "$BACK/package.json" && echo "Backend: NestJS" && STACK_BACK="nestjs"
[ -f "$BACK/requirements.txt" ] && echo "Backend: Python" && STACK_BACK="python"
[ -f "$BACK/go.mod" ] && echo "Backend: Go" && STACK_BACK="go"

# Verify frontend
[ -f "$FRONT/angular.json" ] && echo "Frontend: Angular" && STACK_FRONT="angular"
[ -f "$FRONT/next.config.js" ] || [ -f "$FRONT/next.config.ts" ] && echo "Frontend: Next.js" && STACK_FRONT="nextjs"
[ -f "$FRONT/vite.config.ts" ] && grep -q "react" "$FRONT/vite.config.ts" 2>/dev/null && echo "Frontend: React (Vite)" && STACK_FRONT="react-vite"
grep -q '"vue"' "$FRONT/package.json" 2>/dev/null && echo "Frontend: Vue" && STACK_FRONT="vue"
```

If a path cannot be validated, report it and ask the user to correct it.

---

## Step 3: Verify Google Drive MCP

```bash
# Check if gdrive MCP is available
cat ~/.claude/settings.json 2>/dev/null | grep -i "gdrive\|google"
```

If not configured:
```
Google Drive MCP is not configured. To enable document fetching from Drive:

1. Follow the setup guide: https://github.com/modelcontextprotocol/servers/tree/main/src/gdrive
2. Add your credentials to ~/.claude/settings.json under "mcpServers"
3. Or configure via: claude mcp add gdrive

The pipeline will ask you to paste document content manually if Drive MCP is unavailable.
```

If configured, test access:
```
Use the gdrive MCP to list files in folder 1BVZQKF61uvBKIpnFUHRQK4GSzR8r8Eh3.
Confirm: can files be listed? (yes / access denied / folder not found)
```

---

## Step 4: Configure Figma (optional)

If user provided a Figma API key:

```bash
# Check if figma MCP entry exists in project .mcp.json
cat .mcp.json 2>/dev/null | grep -i figma
```

If not present, add to project `.mcp.json`:
```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-developer-mcp", "--stdio"],
      "env": {
        "FIGMA_API_KEY": "{API_KEY}"
      }
    }
  }
}
```

Note: API key is written only to the local `.mcp.json` which must be gitignored.

---

## Step 5: Write config

Create `.nter-nexus/config.json`:

```json
{
  "project": "{PROJECT_NAME}",
  "back": "{absolute-backend-path}",
  "front": "{absolute-frontend-path}",
  "stack_backend": "{STACK_BACK}",
  "stack_frontend": "{STACK_FRONT}",
  "drive_folder_id": "{DRIVE_FOLDER_ID}",
  "treyit_board_id": "{BOARD_ID | null}",
  "figma_configured": {true | false},
  "initialized_at": "{ISO datetime}"
}
```

Create `.nter-nexus/state/` directory for pipeline artifacts.

Ensure `.gitignore` contains:
```
.nter-nexus/state/
.nter-nexus/config.json
```

---

## Step 6: Cold scan

Run initial Arco-Scan and Estela-Scan to build the baseline snapshot. This is what warm-start comparisons are made against.

**Arco-Scan**: scan backend and frontend structure → write to `.nter-nexus/state/00-arco-scan.md`
**Estela-Scan**: scan design tokens and UI patterns → write to `.nter-nexus/state/00-estela-scan.md`

Then capture the current git commit SHA:

```bash
cd {backend-path} && git rev-parse HEAD 2>/dev/null
cd {frontend-path} && git rev-parse HEAD 2>/dev/null
```

Write `.nter-nexus/state/snapshot-meta.json`:
```json
{
  "back_commit": "{sha | null}",
  "front_commit": "{sha | null}",
  "scanned_at": "{ISO datetime}"
}
```

---

## Step 7: Report

```
nter-nexus initialized ✓

Project:          {name}
Backend:          {path} ({stack})
Frontend:         {path} ({stack})
Drive folder:     {id} ({accessible | not configured})
Figma MCP:        {configured | skipped}
Treyit board:     {id | not configured — set later}

Baseline scan:    .nter-nexus/state/00-arco-scan.md
                  .nter-nexus/state/00-estela-scan.md
Snapshot:         .nter-nexus/state/snapshot-meta.json

Next step: run /nter-nexus:poc to process a PRD document.
```

If anything was skipped (Drive MCP, Figma, Treyit board), list what was skipped and how to configure it later.
