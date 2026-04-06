# Career-Ops — Knowledge Transfer Document

> **Version:** 1.1.0 | **Stack:** Claude Code · Node.js · Playwright · Go · Markdown

This document is a complete technical reference for the career-ops system: every component, every data flow, every decision, and every execution path — with flowcharts and execution diagrams.

---

## Table of Contents

1. [What is career-ops](#1-what-is-career-ops)
2. [High-Level System Architecture](#2-high-level-system-architecture)
3. [Directory Map](#3-directory-map)
4. [Data Contract: Two-Layer Model](#4-data-contract-two-layer-model)
5. [Session Startup Flow](#5-session-startup-flow)
6. [Core Skill Modes](#6-core-skill-modes)
7. [Auto-Pipeline: Full Execution Flow](#7-auto-pipeline-full-execution-flow)
8. [Single Offer Evaluation (oferta mode)](#8-single-offer-evaluation-oferta-mode)
9. [PDF Generation Pipeline](#9-pdf-generation-pipeline)
10. [Portal Scanner (scan mode)](#10-portal-scanner-scan-mode)
11. [Batch Processing](#11-batch-processing)
12. [Pipeline Mode (URL Inbox)](#12-pipeline-mode-url-inbox)
13. [Tracker Management](#13-tracker-management)
14. [Utility Scripts Reference](#14-utility-scripts-reference)
15. [Dashboard TUI](#15-dashboard-tui)
16. [Scoring System Deep Dive](#16-scoring-system-deep-dive)
17. [Data Schemas](#17-data-schemas)
18. [Canonical States Machine](#18-canonical-states-machine)
19. [Update System](#19-update-system)
20. [Ethical Use Guardrails](#20-ethical-use-guardrails)
21. [Customization Map](#21-customization-map)
22. [Tech Stack Summary](#22-tech-stack-summary)

---

## 1. What is career-ops

Career-ops is an **AI-powered job search pipeline** built on top of Claude Code. It turns the Claude Code agent into a full command center that:

- Evaluates job offers with a structured A-F scoring system (10 weighted dimensions)
- Generates tailored, ATS-optimized PDF CVs per job description
- Scans 45+ company career portals and job boards automatically
- Processes offers in batch with parallel sub-agents
- Tracks every evaluated offer in a canonical markdown table
- Provides integrity checks, deduplication, and status normalization

The system was built and battle-tested on a real job search: 740+ offers evaluated, 100+ tailored CVs generated, resulting in a Head of Applied AI role.

> **Core philosophy:** quality over quantity. Career-ops is a filter, not a spray tool. It strongly discourages applying to anything below 4.0/5.

---

## 2. High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Claude Code Agent                          │
│              Reads CLAUDE.md + modes/*.md at runtime                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
   ┌──────▼──────┐    ┌────────▼──────┐   ┌────────▼──────────┐
   │ Single Eval  │    │  Portal Scan  │   │  Batch Process    │
   │ (auto-pipe)  │    │  (scan.md)    │   │  (batch-runner)   │
   └──────┬──────┘    └────────┬──────┘   └────────┬──────────┘
          │                    │                    │
          │            ┌───────▼───────┐     ┌──────▼──────────┐
          │            │  pipeline.md  │     │   N × workers    │
          │            │  (URL inbox)  │     │   (claude -p)    │
          │            └───────────────┘     └──────┬──────────┘
          │                                          │
   ┌──────▼──────────────────────────────────────────▼────────────┐
   │                        Output Pipeline                        │
   │  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────┐│
   │  │  Report .md  │  │  PDF            │  │  Tracker TSV     ││
   │  │  (A-F eval)  │  │  (HTML→Puppeteer)│  │  (tracker-add/.) ││
   │  └──────────────┘  └─────────────────┘  └──────────────────┘│
   └──────────────────────────────┬───────────────────────────────┘
                                  │  merge-tracker.mjs
                       ┌──────────▼──────────┐
                       │  data/applications.md│
                       │  (canonical tracker) │
                       └─────────────────────┘
```

---

## 3. Directory Map

```
career-ops/
│
├── CLAUDE.md                   ← Agent instructions (system layer)
├── DATA_CONTRACT.md            ← Which files are user vs system
├── README.md                   ← Public-facing documentation
├── VERSION                     ← Semantic version (e.g., 1.1.0)
├── package.json                ← Node.js deps (playwright)
│
├── cv.md                       ← [USER] Your CV in markdown (source of truth)
├── article-digest.md           ← [USER] Proof points from portfolio (optional)
├── portals.yml                 ← [USER] Scanner config (companies + queries)
│
├── config/
│   └── profile.example.yml     ← Template → copy to profile.yml
│   └── profile.yml             ← [USER] Your identity, targets, comp range
│
├── modes/                      ← Agent skill modes (system layer)
│   ├── _shared.md              ← Scoring system, archetypes, global rules
│   ├── _profile.md             ← [USER] Your archetypes, negotiation scripts
│   ├── _profile.template.md    ← Template for _profile.md
│   ├── auto-pipeline.md        ← Full auto-pipeline instructions
│   ├── oferta.md               ← Single offer evaluation (A-F blocks)
│   ├── ofertas.md              ← Multi-offer comparison matrix
│   ├── pdf.md                  ← PDF generation pipeline
│   ├── scan.md                 ← Portal scanner workflow
│   ├── batch.md                ← Batch processing architecture
│   ├── pipeline.md             ← URL inbox processor
│   ├── apply.md                ← Live application form assistant
│   ├── contacto.md             ← LinkedIn outreach message generator
│   ├── deep.md                 ← Deep company research prompt
│   ├── tracker.md              ← Tracker read/update
│   ├── training.md             ← Course/cert evaluation
│   ├── project.md              ← Portfolio project evaluation
│   └── de/                     ← German language mode variants
│       ├── _shared.md
│       ├── angebot.md
│       ├── bewerben.md
│       └── pipeline.md
│
├── templates/
│   ├── cv-template.html        ← ATS-optimized HTML CV template
│   ├── portals.example.yml     ← Template for portals.yml
│   └── states.yml              ← Canonical status definitions (source of truth)
│
├── batch/
│   ├── batch-prompt.md         ← Self-contained worker prompt (system layer)
│   ├── batch-runner.sh         ← Orchestrator shell script
│   ├── batch-input.tsv         ← [gitignored] Input URLs for batch run
│   ├── batch-state.tsv         ← [gitignored] Progress tracking
│   ├── logs/                   ← [gitignored] Per-offer worker logs
│   └── tracker-additions/      ← [gitignored] TSV files awaiting merge
│
├── dashboard/                  ← Go TUI application
│   ├── main.go
│   ├── internal/               ← TUI components (Bubble Tea)
│   ├── go.mod
│   └── go.sum
│
├── data/                       ← [USER, gitignored]
│   ├── applications.md         ← Canonical application tracker table
│   ├── pipeline.md             ← URL inbox (pending + processed)
│   └── scan-history.tsv        ← Dedup history for scanner
│
├── reports/                    ← [USER, gitignored] Evaluation reports
│   └── {###}-{company}-{date}.md
│
├── output/                     ← [USER, gitignored] Generated PDFs
│   └── cv-candidate-{company}-{date}.pdf
│
├── jds/                        ← [USER] Saved job descriptions (local: prefix)
│
├── interview-prep/
│   └── story-bank.md           ← [USER] Accumulated STAR+R stories
│
├── fonts/                      ← Self-hosted fonts (Space Grotesk + DM Sans)
│
├── examples/                   ← Sample CV, report, proof points
│
├── docs/                       ← Documentation
│   ├── SETUP.md
│   ├── CUSTOMIZATION.md
│   ├── ARCHITECTURE.md
│   └── KNOWLEDGE_TRANSFER.md   ← This file
│
├── generate-pdf.mjs            ← HTML → PDF via Playwright/Chromium
├── merge-tracker.mjs           ← Merge TSV additions into applications.md
├── verify-pipeline.mjs         ← Health check for pipeline integrity
├── normalize-statuses.mjs      ← Canonicalize status values
├── dedup-tracker.mjs           ← Remove duplicate tracker entries
├── cv-sync-check.mjs           ← Validate setup consistency
└── update-system.mjs           ← Check/apply/rollback system updates
```

---

## 4. Data Contract: Two-Layer Model

The system distinguishes between **user data** (never auto-updated) and **system files** (safe to replace on update).

```
┌─────────────────────────────────────────────────────────────┐
│                      USER LAYER                             │
│           (NEVER touched by updates or Claude)              │
├─────────────────────────┬───────────────────────────────────┤
│ File                    │ Purpose                           │
├─────────────────────────┼───────────────────────────────────┤
│ cv.md                   │ Your CV (source of truth)         │
│ config/profile.yml      │ Identity, targets, comp range     │
│ modes/_profile.md       │ Archetypes, narrative, negotiation│
│ article-digest.md       │ Proof points from portfolio       │
│ interview-prep/story-bank.md │ STAR+R story accumulation   │
│ portals.yml             │ Customized company/query list     │
│ data/applications.md    │ Application tracker               │
│ data/pipeline.md        │ URL inbox                         │
│ data/scan-history.tsv   │ Scanner dedup history             │
│ reports/*               │ Evaluation reports                │
│ output/*                │ Generated PDFs                    │
│ jds/*                   │ Saved job descriptions            │
└─────────────────────────┴───────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     SYSTEM LAYER                            │
│              (safe to replace on updates)                   │
├─────────────────────────┬───────────────────────────────────┤
│ CLAUDE.md               │ Agent instructions                │
│ modes/_shared.md        │ Scoring + archetypes + rules      │
│ modes/oferta.md + ...   │ All skill mode instructions       │
│ *.mjs                   │ Utility scripts                   │
│ batch/batch-prompt.md   │ Worker prompt                     │
│ batch/batch-runner.sh   │ Orchestrator script               │
│ templates/*             │ CV template, states, portals      │
│ dashboard/*             │ Go TUI source                     │
│ fonts/*                 │ Self-hosted web fonts             │
│ VERSION                 │ Version number                    │
│ DATA_CONTRACT.md        │ This contract                     │
└─────────────────────────┴───────────────────────────────────┘
```

**The cardinal rule:** customizations always go into `modes/_profile.md` or `config/profile.yml`, never into `_shared.md`. This ensures updates don't overwrite personal data.

---

## 5. Session Startup Flow

Every Claude Code session begins with this silent startup sequence:

```
Session starts
     │
     ▼
┌─────────────────────────────────────────────┐
│  node update-system.mjs check               │
│  (silently checks GitHub for new version)   │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
   update-available      up-to-date / dismissed / offline
        │                     │
   Tell user:            Say nothing
   "v{old} → v{new}       │
    Update?"              │
        │                  │
    ┌───┴───┐              │
   Yes     No              │
    │       │              │
  apply   dismiss          │
    │       │              │
    └───┬───┘              │
        └──────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│  Onboarding checks (silent):                       │
│  1. cv.md exists?                                  │
│  2. config/profile.yml exists?                     │
│  3. modes/_profile.md exists?  (auto-copy if not)  │
│  4. portals.yml exists?                            │
└───────────────────┬────────────────────────────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    All exist            Missing files
         │                     │
    Ready to use         Onboarding mode
         │                     │
         ▼                     ▼
  Normal operation       Step-by-step setup:
                          1. CV collection
                          2. Profile fill
                          3. Portals setup
                          4. Tracker init
                          5. Context gathering
                          6. Confirm ready
```

### Onboarding Steps Detail

| Step | File Created | Method |
|------|-------------|--------|
| 1 – CV | `cv.md` | Paste CV / LinkedIn URL / dictate experience |
| 2 – Profile | `config/profile.yml` | Copy from example, fill interactively |
| 3 – Portals | `portals.yml` | Copy from template, customize queries |
| 4 – Tracker | `data/applications.md` | Create empty table |
| 5 – Context | `config/profile.yml` + `article-digest.md` | Conversational extraction |

---

## 6. Core Skill Modes

The agent selects modes based on user intent:

```
User action / input
         │
         ▼
┌────────────────────────────────────────────────────────────────┐
│                     Mode Routing Table                         │
├──────────────────────────────┬─────────────────────────────────┤
│ User action                  │ Mode loaded                     │
├──────────────────────────────┼─────────────────────────────────┤
│ Paste URL or JD text         │ auto-pipeline.md                │
│ /career-ops (no args)        │ Show all commands               │
│ /career-ops scan             │ scan.md                         │
│ /career-ops pdf              │ pdf.md                          │
│ /career-ops batch            │ batch.md                        │
│ /career-ops tracker          │ tracker.md                      │
│ /career-ops apply            │ apply.md                        │
│ /career-ops pipeline         │ pipeline.md                     │
│ /career-ops contacto         │ contacto.md                     │
│ /career-ops deep             │ deep.md                         │
│ /career-ops training         │ training.md                     │
│ /career-ops project          │ project.md                      │
│ Compare multiple offers      │ ofertas.md                      │
└──────────────────────────────┴─────────────────────────────────┘
```

Every mode reads `_shared.md` first (global context), then `_profile.md` (user customizations), before executing its specific instructions.

---

## 7. Auto-Pipeline: Full Execution Flow

This is the primary workflow — triggered whenever a URL or JD text is pasted.

```
User pastes URL or JD text
           │
           ▼
   ┌───────────────┐
   │  Step 0:      │
   │  Extract JD   │◄────── If URL:
   └───────┬───────┘        1st: Playwright (browser_navigate + snapshot)
           │                2nd: WebFetch (static pages)
           │                3rd: WebSearch (last resort)
           │                Fail: Ask user to paste manually
           ▼
   ┌───────────────┐
   │  Step 1:      │  Reads: cv.md, _profile.md, article-digest.md
   │  Evaluation   │  Runs: Blocks A → F (see §8)
   │  A-F          │  Outputs: Score 1–5, archetype, gaps, STAR stories
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │  Step 2:      │  Creates: reports/{###}-{slug}-{date}.md
   │  Save Report  │  Contains: All 6 blocks + keywords + (optional) Section G
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐  Reads: cv-template.html, cv.md, profile.yml
   │  Step 3:      │  Injects keywords, rewrites summary, reorders bullets
   │  Generate PDF │  Writes: /tmp/cv-candidate-{company}.html
   │               │  Runs: node generate-pdf.mjs → output/{company}-{date}.pdf
   └───────┬───────┘
           │
           ▼
   ┌─────────────────────────────────────┐
   │  Step 4: Draft Application Answers  │  Only if score >= 4.5
   │  (Section G of report)              │  Extracts form questions via Playwright
   │                                     │  Tone: "I'm choosing you" framework
   └───────┬─────────────────────────────┘
           │
           ▼
   ┌───────────────┐
   │  Step 5:      │  Writes TSV to: batch/tracker-additions/{num}.tsv
   │  Update       │  NEVER edits applications.md directly
   │  Tracker      │  merge-tracker.mjs handles the actual merge
   └───────────────┘
```

---

## 8. Single Offer Evaluation (oferta mode)

The evaluation engine produces 6 blocks (A–F):

```
         Input: JD text + cv.md + _profile.md + article-digest.md
                               │
              ┌────────────────▼─────────────────┐
              │  Paso 0: Archetype Detection      │
              │  Classify into 1 of 6 types:      │
              │  • AI Platform / LLMOps           │
              │  • Agentic / Automation            │
              │  • Technical AI PM                │
              │  • AI Solutions Architect         │
              │  • AI Forward Deployed            │
              │  • AI Transformation              │
              └────────────────┬─────────────────┘
                               │ archetype → determines framing for all blocks
              ┌────────────────▼─────────────────┐
              │  Block A: Role Summary            │
              │  • Archetype, domain, function    │
              │  • Seniority, remote policy       │
              │  • Team size, TL;DR sentence      │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼─────────────────┐
              │  Block B: CV Match                │
              │  • JD requirement → exact CV line │
              │  • Gaps + mitigation strategy     │
              │  • Hard blocker vs. nice-to-have  │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼─────────────────┐
              │  Block C: Level Strategy          │
              │  • JD level vs. candidate level   │
              │  • "Sell senior without lying"    │
              │  • Downlevel fallback plan        │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼─────────────────┐
              │  Block D: Comp Research           │
              │  • WebSearch: Glassdoor, Levels   │
              │  • Market rate vs. offer          │
              │  • Company comp reputation        │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼─────────────────┐
              │  Block E: Personalization Plan    │
              │  • Top 5 CV changes               │
              │  • Top 5 LinkedIn changes         │
              │  • Keyword injection targets      │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼─────────────────┐
              │  Block F: Interview Prep          │
              │  • 6–10 STAR+R stories            │
              │  • Mapped to JD requirements      │
              │  • Case study recommendation      │
              │  • Red-flag question responses    │
              │  → Append new stories to          │
              │    interview-prep/story-bank.md   │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼─────────────────┐
              │  Post-evaluation:                 │
              │  • Save reports/{###}-slug-date.md│
              │  • Write TSV to tracker-additions/│
              └──────────────────────────────────┘
```

### Archetype-Specific Framing

Each archetype shifts which proof points to emphasize in Block B:

| Archetype | Emphasized in Block B |
|-----------|----------------------|
| FDE (Forward Deployed) | Fast delivery, client-facing, prototyping |
| SA (Solutions Architect) | System design, integrations, enterprise |
| PM (Product Manager) | Discovery, roadmap, metrics, stakeholders |
| LLMOps | Evals, observability, pipelines, cost |
| Agentic | Multi-agent, HITL, orchestration, error handling |
| Transformation | Change management, adoption, scaling |

---

## 9. PDF Generation Pipeline

```
   Input: cv.md + profile.yml + JD (keywords)
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │ 1. Extract 15-20 keywords from JD      │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 2. Detect:                             │
   │    • JD language → CV language         │
   │    • Company location → paper format   │
   │      US/Canada → letter (8.5in)        │
   │      Elsewhere → A4 (210mm)            │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 3. Rewrite Professional Summary        │
   │    • Inject top 5 JD keywords          │
   │    • Bridge exit narrative             │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 4. Select top 3-4 relevant projects    │
   │    • Ranked by JD match                │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 5. Reorder experience bullets          │
   │    • Most JD-relevant bullets first    │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 6. Build competency grid               │
   │    • 6-8 keyword phrases from JD       │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 7. Fill cv-template.html placeholders  │
   │    • {{NAME}}, {{EMAIL}}, etc.         │
   │    • {{SUMMARY_TEXT}}, {{EXPERIENCE}}  │
   │    • {{COMPETENCIES}}, {{PROJECTS}}    │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 8. Write to /tmp/cv-{company}.html     │
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 9. node generate-pdf.mjs               │
   │    Chromium headless → PDF render      │
   │    Saves to output/{company}-{date}.pdf│
   └────────────────┬───────────────────────┘
                    │
   ┌────────────────▼───────────────────────┐
   │ 10. Update tracker: PDF ❌ → ✅         │
   └────────────────────────────────────────┘
```

### PDF Design Tokens

| Property | Value |
|----------|-------|
| Heading font | Space Grotesk 600–700 |
| Body font | DM Sans 400–500 |
| Primary color | `hsl(187, 74%, 32%)` (cyan) |
| Accent color | `hsl(270, 70%, 45%)` (purple) |
| Gradient header | cyan → purple (2px line) |
| Margins | 0.6in |
| Background | white |
| Body size | 11px, line-height 1.5 |

### CV Section Order (Optimized for 6-Second Scan)

1. Header (name, contact, portfolio link)
2. Professional Summary (keyword-dense, 3–4 lines)
3. Core Competencies (flex-grid, 6–8 phrases)
4. Work Experience (reverse chronological)
5. Projects (top 3–4 most relevant)
6. Education & Certifications
7. Skills

---

## 10. Portal Scanner (scan mode)

```
   Input: portals.yml + scan-history.tsv + applications.md + pipeline.md
                    │
                    ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Discovery: 3 levels run in parallel (additive)            │
   │                                                            │
   │  Level 1 — Playwright (PRIMARY)                           │
   │  ┌──────────────────────────────────────────────────────┐ │
   │  │ For each tracked_company with careers_url:           │ │
   │  │   browser_navigate(careers_url)                      │ │
   │  │   browser_snapshot() → extract all job listings      │ │
   │  │   If pagination → navigate additional pages          │ │
   │  │   Output: {title, url, company}                      │ │
   │  └──────────────────────────────────────────────────────┘ │
   │                                                            │
   │  Level 2 — Greenhouse API (COMPLEMENTARY)                 │
   │  ┌──────────────────────────────────────────────────────┐ │
   │  │ For each company with api: field:                    │ │
   │  │   WebFetch boards-api.greenhouse.io/v1/boards/{slug} │ │
   │  │   Parse JSON → extract {title, url, company}         │ │
   │  └──────────────────────────────────────────────────────┘ │
   │                                                            │
   │  Level 3 — WebSearch (BROAD DISCOVERY)                    │
   │  ┌──────────────────────────────────────────────────────┐ │
   │  │ For each search_query with enabled: true:            │ │
   │  │   WebSearch(query) → results                         │ │
   │  │   Extract {title, url, company} from result titles   │ │
   │  └──────────────────────────────────────────────────────┘ │
   └──────────────────────────────┬─────────────────────────────┘
                                  │ combined candidate list
                                  ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Title Filtering (portals.yml → title_filter)              │
   │  ✅ At least 1 positive keyword in title                   │
   │  ❌ Zero negative keywords in title                        │
   │  ⭐ seniority_boost keywords = priority flag              │
   └──────────────────────────────┬─────────────────────────────┘
                                  │
                                  ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Deduplication (3 sources)                                 │
   │  1. scan-history.tsv  → URL exact match                   │
   │  2. applications.md   → company + role normalized match   │
   │  3. pipeline.md       → URL exact match                   │
   └──────────────────────────────┬─────────────────────────────┘
                                  │
                                  ▼
   ┌────────────────────────────────────────────────────────────┐
   │  For each NEW offer that passes filters:                   │
   │  → Append to pipeline.md:                                 │
   │    - [ ] {url} | {company} | {title}                      │
   │  → Record in scan-history.tsv: status=added               │
   │                                                            │
   │  For filtered offers: status=skipped_title                │
   │  For duplicates:      status=skipped_dup                  │
   └──────────────────────────────────────────────────────────┘
```

### portals.yml Structure

```yaml
title_filter:
  positive: ["AI", "ML", "LLM", ...]   # Must match ≥1
  negative: ["Junior", "Intern", ...]   # Must match 0
  seniority_boost: ["Staff", "Lead"]    # Priority boost

search_queries:
  - name: "Ashby — AI PM"
    query: "site:jobs.ashbyhq.com AI product manager"
    enabled: true

tracked_companies:
  - name: "Anthropic"
    platform: "greenhouse"
    careers_url: "https://job-boards.greenhouse.io/anthropic"
    api: "https://boards-api.greenhouse.io/v1/boards/anthropic/jobs"
    enabled: true
```

---

## 11. Batch Processing

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│           Claude Conductor (--chrome --dangerously-skip-permissions)  │
│                                                                 │
│   Chrome visible: navigates portals, reads DOM live            │
│   Reads batch-input.tsv → spawns N parallel workers            │
└──────────────────────────────┬──────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    ┌─────▼─────┐        ┌─────▼─────┐       ┌─────▼─────┐
    │ Worker 1  │        │ Worker 2  │       │ Worker N  │
    │ claude -p │        │ claude -p │       │ claude -p │
    └─────┬─────┘        └─────┬─────┘       └─────┬─────┘
          │                    │                    │
          ▼                    ▼                    ▼
    report .md          report .md          report .md
    PDF .pdf            PDF .pdf            PDF .pdf
    tracker TSV         tracker TSV         tracker TSV
          │                    │                    │
          └────────────────────┴────────────────────┘
                               │
                    node merge-tracker.mjs
                               │
                    data/applications.md
```

### Batch State Machine

```
┌─────────┐    start      ┌─────────────┐   success   ┌───────────┐
│ pending │──────────────►│ in_progress │────────────►│ completed │
└─────────┘               └──────┬──────┘             └───────────┘
                                  │ failure
                                  ▼
                           ┌─────────┐   --retry-failed   ┌─────────┐
                           │ failed  │──────────────────► │ pending │
                           └─────────┘                    └─────────┘
```

### batch-state.tsv Schema

```
id  url         status     started_at  completed_at  report_num  score  error  retries
1   https://...  completed  2026-...   2026-...      002         4.2    -      0
2   https://...  failed     2026-...   2026-...      -           -      msg    1
3   https://...  pending    -          -             -           -      -      0
```

### Batch Runner Options

| Flag | Effect |
|------|--------|
| `--dry-run` | List pending items, no execution |
| `--retry-failed` | Re-run only failed items |
| `--start-from N` | Skip items with id < N |
| `--parallel N` | Run N workers simultaneously |
| `--max-retries N` | Max retry attempts per item (default: 2) |

---

## 12. Pipeline Mode (URL Inbox)

The pipeline is a lightweight "second brain" for accumulating URLs to evaluate later.

```
   User adds URLs to data/pipeline.md:
   - [ ] https://jobs.example.com/...
   - [ ] https://boards.greenhouse.io/...
           │
           ▼ /career-ops pipeline
   ┌─────────────────────────────────────────────┐
   │ 1. Read pipeline.md → find all - [ ] items  │
   │ 2. If 3+ URLs → launch parallel agents      │
   │    If 1-2 URLs → process sequentially       │
   └─────────────────────┬───────────────────────┘
                         │ for each URL
                         ▼
   ┌─────────────────────────────────────────────┐
   │  a. Calculate next REPORT_NUM               │
   │     (max in reports/ + 1, zero-padded)      │
   └─────────────────────┬───────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────┐
   │  b. Extract JD:                             │
   │     1st: Playwright                         │
   │     2nd: WebFetch                           │
   │     3rd: WebSearch                          │
   │     local: → read jds/{file} directly       │
   │     fail: mark - [!] with note, continue    │
   └─────────────────────┬───────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────┐
   │  c. Run full auto-pipeline (Eval→PDF→Track) │
   └─────────────────────┬───────────────────────┘
                         │
   ┌─────────────────────▼───────────────────────┐
   │  d. Move item from Pending to Processed:    │
   │  - [x] #NNN | URL | Company | Role | Score  │
   └─────────────────────────────────────────────┘
```

### pipeline.md Format

```markdown
## Pendientes
- [ ] https://jobs.example.com/posting/123
- [ ] https://boards.greenhouse.io/company/jobs/456 | Company Inc | Senior PM
- [!] https://private.url/job — Error: login required

## Procesadas
- [x] #143 | https://jobs.example.com/posting/789 | Acme Corp | AI PM | 4.2/5 | PDF ✅
```

---

## 13. Tracker Management

### Tracker Addition Flow (ALWAYS use TSV, never direct edit)

```
Evaluation completes
         │
         ▼
Write batch/tracker-additions/{num}-{slug}.tsv
         │
         │  TSV format (9 columns, tab-separated):
         │  num | date | company | role | status | score | pdf | report | notes
         │
         ▼
node merge-tracker.mjs
         │
         ▼
Appends row to data/applications.md
(handles column swap: TSV has status before score,
 applications.md has score before status)
```

### applications.md Schema

```markdown
| # | Date | Company | Role | Score | Status | PDF | Report | Notes |
|---|------|---------|------|-------|--------|-----|--------|-------|
| 001 | 2026-01-01 | Acme Corp | Senior AI PM | 4.2/5 | Evaluated | ✅ | [001](reports/001-acme-2026-01-01.md) | Strong match |
```

**Column order in applications.md:** `# | Date | Company | Role | Score | Status | PDF | Report | Notes`
**Column order in TSV:** `num | date | company | role | status | score | pdf | report | notes`  
(note: status/score swap — `merge-tracker.mjs` handles this automatically)

### Pipeline Integrity Commands

```bash
node verify-pipeline.mjs      # Health check (statuses, dupes, broken links)
node normalize-statuses.mjs   # Map aliases to canonical states
node dedup-tracker.mjs        # Remove duplicate company+role entries
node merge-tracker.mjs        # Merge pending TSVs into applications.md
node cv-sync-check.mjs        # Check setup consistency
```

---

## 14. Utility Scripts Reference

### generate-pdf.mjs

Converts an HTML file to PDF using Playwright/Chromium.

```bash
node generate-pdf.mjs <input.html> <output.pdf> [--format=letter|a4]
```

- Default format: `a4`
- Reads HTML, launches Chromium headless, prints to PDF
- Handles relative font paths via `file://` base URL

### merge-tracker.mjs

Reads all `.tsv` files in `batch/tracker-additions/`, converts each to a markdown table row, and appends to `data/applications.md`. After merging, moves processed TSVs to `batch/tracker-additions/merged/`.

**Duplicate guard:** If company+role already exists in `applications.md`, the merge script skips the insert and logs a warning.

### verify-pipeline.mjs

Checks pipeline integrity across 7 dimensions:

1. All statuses are canonical (per `templates/states.yml`)
2. No duplicate company+role entries
3. All report links point to existing files
4. Scores match format `X.X/5` (or `N/A` / `DUP`)
5. All rows have proper pipe-delimited format
6. No pending TSVs in `tracker-additions/` awaiting merge
7. `states.yml` canonical IDs for cross-system consistency

### normalize-statuses.mjs

Maps status aliases (e.g., `evaluada`, `aplicado`, `enviada`) to canonical English values (e.g., `Evaluated`, `Applied`). Source of truth: `templates/states.yml` aliases field.

### dedup-tracker.mjs

Finds duplicate entries in `applications.md` where company + role match (case-insensitive, normalized). Keeps the most recent entry by date. Marks duplicates with score `DUP` before removing.

### cv-sync-check.mjs

Validates that the setup is coherent:
- `cv.md` exists and is non-empty
- `config/profile.yml` exists
- `modes/_profile.md` exists (copies from template if not)
- `portals.yml` exists
- Reports any warnings to be shown to the user before evaluation

### update-system.mjs

```bash
node update-system.mjs check     # Returns JSON: status, local, remote, changelog
node update-system.mjs apply     # Downloads and applies latest system layer files
node update-system.mjs dismiss   # Snoozes update notification
node update-system.mjs rollback  # Reverts to previous version
```

Outputs JSON status: `up-to-date` | `update-available` | `dismissed` | `offline`

---

## 15. Dashboard TUI

A standalone Go terminal application to browse the pipeline visually.

```bash
cd dashboard
go build -o career-dashboard .
./career-dashboard
```

### Features

| Feature | Detail |
|---------|--------|
| Filter tabs | All, Evaluated, Applied, Interview, Top ≥4, SKIP |
| Sort modes | Score, Date, Company, Status |
| View modes | Grouped by status, flat list |
| Report preview | Lazy-loaded markdown preview of selected entry |
| Inline status change | Pick new status with keyboard |

### Tech Stack

- **Framework:** Bubble Tea (TUI framework)
- **Styling:** Lipgloss (Catppuccin Mocha theme)
- **Data source:** Reads `data/applications.md` directly
- **Status states:** Read from `templates/states.yml`

---

## 16. Scoring System Deep Dive

### Global Score (1–5)

Calculated as a weighted average across multiple dimensions. Dimensions tracked in `_shared.md`:

| Dimension | What it measures |
|-----------|-----------------|
| Match con CV | Skills, experience, proof points alignment |
| North Star alignment | Fit to the user's target archetypes |
| Comp | Salary vs. market quartile |
| Cultural signals | Culture, growth, stability, remote policy |
| Red flags | Blockers, warnings (negative adjustments) |

### Score Interpretation

| Score | Recommendation |
|-------|---------------|
| 4.5+ | Strong match — apply immediately |
| 4.0–4.4 | Good match — worth applying |
| 3.5–3.9 | Decent but not ideal — apply only with specific reason |
| < 3.5 | Not recommended — system actively discourages |

### Multi-Offer Comparison Matrix (ofertas mode)

10 weighted dimensions for side-by-side comparison:

| Dimension | Weight |
|-----------|--------|
| North Star alignment | 25% |
| CV match | 15% |
| Level (senior+) | 15% |
| Comp estimate | 10% |
| Growth trajectory | 10% |
| Remote quality | 5% |
| Company reputation | 5% |
| Tech stack modernity | 5% |
| Speed to offer | 5% |
| Cultural signals | 5% |

---

## 17. Data Schemas

### Report File: `reports/{###}-{company}-{YYYY-MM-DD}.md`

```markdown
# Evaluación: {Company} — {Role}

**Fecha:** YYYY-MM-DD
**Arquetipo:** {detected}
**Score:** X.X/5
**URL:** https://...
**PDF:** output/cv-{company}-{date}.pdf

---
## A) Resumen del Rol
## B) Match con CV
## C) Nivel y Estrategia
## D) Comp y Demanda
## E) Plan de Personalización
## F) Plan de Entrevistas
## G) Draft Application Answers   ← only if score >= 4.5
---
## Keywords extraídas
```

### Tracker TSV: `batch/tracker-additions/{num}-{slug}.tsv`

```
{num}\t{YYYY-MM-DD}\t{company}\t{role}\t{status}\t{X.X/5}\t{✅|❌}\t[{num}](reports/...)\t{note}
```

9 tab-separated columns. Column order: num, date, company, role, **status**, **score**, pdf, report, notes.
(In `applications.md` the order is: num, date, company, role, **score**, **status**, pdf, report, notes)

### Scan History: `data/scan-history.tsv`

```
url  first_seen  portal  title  company  status
https://...  2026-02-10  Ashby — AI PM  Senior PM  Acme  added
https://...  2026-02-10  Greenhouse     Junior Dev  BigCo  skipped_title
```

Status values: `added` | `skipped_title` | `skipped_dup`

---

## 18. Canonical States Machine

Source of truth: `templates/states.yml`

```
                       ┌──────────┐
                       │Evaluated │  ← Initial state after report
                       └────┬─────┘
                            │ apply
                       ┌────▼─────┐
                       │ Applied  │
                       └────┬─────┘
                    ┌───────┴───────┐
                    │               │
               ┌────▼─────┐   ┌────▼──────┐
               │Responded │   │ Rejected  │  ← Company rejects
               └────┬─────┘   └───────────┘
                    │
               ┌────▼─────┐
               │Interview │
               └────┬─────┘
                    │
          ┌─────────┴──────────┐
          │                    │
     ┌────▼─────┐        ┌─────▼──────┐
     │  Offer   │        │  Rejected  │
     └──────────┘        └────────────┘

  Side states (can set at any time):
  ┌──────────┐    ┌──────────┐
  │ Discarded│    │   SKIP   │
  └──────────┘    └──────────┘
  (candidate out)  (don't apply)
```

| State | Canonical | Aliases |
|-------|-----------|---------|
| Evaluated | `Evaluated` | `evaluada` |
| Applied | `Applied` | `aplicado`, `aplicada`, `enviada`, `sent` |
| Responded | `Responded` | `respondido` |
| Interview | `Interview` | `entrevista` |
| Offer | `Offer` | `oferta` |
| Rejected | `Rejected` | `rechazado`, `rechazada` |
| Discarded | `Discarded` | `descartado`, `descartada`, `cerrada`, `cancelada` |
| SKIP | `SKIP` | `no_aplicar`, `no aplicar`, `skip`, `monitor` |

**Rules:**
- No markdown bold (`**`) in status field
- No dates in status field
- No extra text (use Notes column)

---

## 19. Update System

```
node update-system.mjs check
         │
         ▼
  Compare VERSION vs GitHub release tag
         │
    ┌────┴────┐
    │         │
 update    up-to-date
 available       │
    │         silent
    │
 Output JSON:
 {
   "status": "update-available",
   "local": "1.0.0",
   "remote": "1.1.0",
   "changelog": "..."
 }
         │
   User confirms
         │
node update-system.mjs apply
         │
  ┌──────▼──────────────────────────────────────────┐
  │  Downloads system layer files from GitHub        │
  │  Replaces: CLAUDE.md, modes/*.md, *.mjs,         │
  │            templates/*, batch/*, dashboard/*     │
  │  NEVER touches: user layer files                 │
  └──────────────────────────────────────────────────┘
         │
node update-system.mjs rollback  ← Reverts if needed
```

---

## 20. Ethical Use Guardrails

Built-in constraints that cannot be disabled:

| Rule | Enforcement |
|------|-------------|
| Never submit application | Always stops before clicking Submit/Send/Apply |
| Discourage low-fit | Explicitly recommends against applying if score < 4.0/5 |
| Never invent metrics | Reads cv.md + article-digest.md at eval time, never hardcodes |
| Never share phone | Phone excluded from generated messages |
| No parallel Playwright | Only one agent may use Playwright at a time |
| Human-in-the-loop | Every recommendation requires user confirmation before action |

---

## 21. Customization Map

| What to change | File to edit | Method |
|---------------|-------------|--------|
| Target roles / archetypes | `modes/_shared.md` | Edit archetype table |
| Personal narrative | `modes/_profile.md` | Edit framing per archetype |
| Salary targets | `config/profile.yml` → compensation | Update range, minimum |
| Negotiation scripts | `modes/_profile.md` | Edit negotiation section |
| Companies to scan | `portals.yml` → tracked_companies | Add/remove entries |
| Job board queries | `portals.yml` → search_queries | Add/disable queries |
| CV design | `templates/cv-template.html` | Edit CSS tokens |
| Fonts | `fonts/` + `templates/cv-template.html` | Replace font files, update @font-face |
| Scoring weights | `modes/_shared.md` | Edit scoring tables |
| Language (German) | `config/profile.yml` → language.modes_dir: modes/de | Switch modes directory |
| New canonical states | `templates/states.yml` | Add state + update normalize script |
| Session hooks | `.claude/settings.json` | Add Claude Code hooks |

**The golden rule:** User data (`_profile.md`, `profile.yml`) is never overwritten by system updates. Always customize there, not in `_shared.md`.

---

## 22. Tech Stack Summary

| Component | Technology | Version / Notes |
|-----------|-----------|-----------------|
| Agent runtime | Claude Code | Custom skill modes |
| PDF generation | Playwright / Chromium | `^1.58.1` |
| Portal scraping | Playwright | Same instance |
| Dashboard TUI | Go + Bubble Tea + Lipgloss | Go 1.21+ |
| CV template | HTML + CSS | Self-hosted fonts (Space Grotesk, DM Sans) |
| Config | YAML | `profile.yml`, `portals.yml`, `states.yml` |
| Data storage | Markdown tables | `applications.md`, `pipeline.md` |
| Batch history | TSV | `scan-history.tsv`, `batch-state.tsv` |
| Node scripts | ESM (`.mjs`) | Node.js 18+ |
| Language modes | Markdown | `modes/*.md` + `modes/de/*.md` |
| Update mechanism | GitHub API | Compares VERSION tag |

---

## Quick Reference: npm Scripts

```bash
npm run verify       # node verify-pipeline.mjs
npm run normalize    # node normalize-statuses.mjs
npm run dedup        # node dedup-tracker.mjs
npm run merge        # node merge-tracker.mjs
npm run pdf          # node generate-pdf.mjs
npm run sync-check   # node cv-sync-check.mjs
npm run update:check # node update-system.mjs check
npm run update       # node update-system.mjs apply
npm run rollback     # node update-system.mjs rollback
```

---

*Generated from source: CLAUDE.md, DATA_CONTRACT.md, modes/*, *.mjs, templates/*, docs/*.*
