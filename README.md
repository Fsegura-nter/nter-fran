# nterfran — Personal Claude Code Marketplace

Private plugin marketplace by Francisco Segura. Install plugins via Claude Code marketplace URL.

## Install

```
/plugin marketplace add https://gitlab.nfqsolutions.es/francisco.segura/nter-fran
```

## Plugins

| Plugin | Description |
|---|---|
| [nter-nexus](./plugins/nter-nexus/) | PRD-to-tasks pipeline for brownfield projects |

---

# nter-nexus

Multi-agent pipeline that reads a PRD document from Google Drive and produces structured backend, frontend, and QA tasks in Treyit.

## Stack support

| Layer | Supported |
|---|---|
| Backend | Spring Boot (Maven/Gradle), NestJS, FastAPI, Django, Flask, Go |
| Frontend | Angular, Next.js, React (Vite), Vue, Svelte |
| Design | draw.io (via Google Drive), Figma (MCP), Excalidraw (Playwright MCP) |

## Commands

### `/nter-nexus:init`

One-time project setup. Run once per repository.

- Detects backend/frontend paths and stack
- Configures Google Drive folder for PRD documents
- Optionally configures Figma API key
- Runs baseline codebase scan (Arco-Scan + Estela-Scan)
- Writes `.nter-nexus/config.json` (gitignored)

### `/nter-nexus:poc`

Processes a PRD document and creates tasks in Treyit.

- Requests Treyit auth token at runtime (never stored)
- Picks document from Google Drive folder
- Warm start: reuses scan artifacts if git commits haven't changed
- Cold start: re-scans codebase when commits differ
- Requires user confirmation before publishing anything to Treyit

## Pipeline

```
PRD document
 │
 ├──────────────────────────────────────────────────┐
 ▼                                                  ▼
[Clara]          PO review, risks, READINESS    [Canvas-Scan]   Extract design URLs
 │                                               │               draw.io / Figma / Excalidraw
 ▼ (+ reuse or refresh ──────────────────────────┘)
[Arco-Scan + Estela-Scan]  Codebase + design token scan (warm/cold)
 │
 ▼
[Arco]           Architecture decision record (ADR)
 │
 ▼
[Forge]          Backend breakdown + CONTRACT LOCK (hard gate)
 │
 ▼
[Estela]         Design validation + DESIGN ASSET MAP
 │
 ▼
[Pixel]          Frontend breakdown (Contract Lock + Design Asset Map)
 │
 ▼
[Finally]        E2E scenarios + Treyit QA task definitions
 │
 ▼
Treyit tasks (Back → Front → QA, after user confirmation)
```

**Marco** can be invoked by any agent when a human decision is needed. The pipeline pauses and resumes after the user responds.

## Artifacts

All pipeline outputs are saved to `.nter-nexus/state/` (gitignored):

```
.nter-nexus/
├── config.json              ← project config (gitignored)
└── state/
    ├── snapshot-meta.json   ← git SHAs for warm/cold start
    ├── 00-workspace.md
    ├── 00-arco-scan.md
    ├── 00-estela-scan.md
    ├── 00-design.md
    ├── design/              ← exported images or generated mockups
    ├── 01-clara-review.md
    ├── 02-arco-adr.md
    ├── 03-forge-breakdown.md
    ├── 03-contract-lock.md  ← gate: Pixel cannot start without this
    ├── 04-pixel-breakdown.md
    ├── 05-estela-validation.md
    └── 06-finally-qa.md
```

## Requirements

- Claude Code with MCP support
- Google Drive MCP (`@modelcontextprotocol/server-gdrive`) — for document fetching
- Playwright MCP — for Excalidraw screenshots (optional)
- Figma MCP (`figma-developer-mcp`) — for Figma frame export (optional, API key required)
- context7 MCP — for framework-specific library docs (Arco, Forge, Pixel)
- Treyit access — token requested at runtime

## File structure

```
plugins/nter-nexus/
├── .claude-plugin/
│   └── plugin.json          ← plugin manifest
├── .mcp.json                ← MCP server config
├── agents/
│   ├── arco-scan.md         ← Codebase structure scanner
│   ├── arco.md              ← Solution architect
│   ├── canvas-scan.md       ← Design asset extractor
│   ├── clara.md             ← PO review
│   ├── estela-scan.md       ← Design token scanner
│   ├── estela.md            ← UX/UI validation
│   ├── finally.md           ← QA scenarios
│   ├── forge.md             ← Backend breakdown
│   ├── marco.md             ← Human escalation
│   └── pixel.md             ← Frontend breakdown
└── skills/
    ├── nexus-init.md        ← /nter-nexus:init
    └── nexus-poc.md         ← /nter-nexus:poc
```
