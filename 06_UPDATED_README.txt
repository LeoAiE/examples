# Complete Tool Bible

> **PURPOSE**: documentation for the AI Agent Platform V2. Every script, every connection, every configuration. Step-by-step instructions for changing ANY tool, agent, prompt, or report — with exact file paths and exact locations.

---

## TABLE OF CONTENTS

1. [Quick Start](#1-quick-start)
2. [Architecture Overview](#2-architecture-overview)
3. [The Living Dossier Pipeline (Current Architecture)](#3-the-living-dossier-pipeline)
4. [Directory Structure](#4-directory-structure)
5. [Every Script Explained](#5-every-script-explained)
6. [Per-Agent Walkthrough (What To Change & Where)](#6-per-agent-walkthrough)
7. [Per-Tool Walkthrough — quick lookup](#7-per-tool-walkthrough)
8. [Per-JSON Plan Walkthrough](#8-per-json-plan-walkthrough)
9. [How To Change Report Prompts](#9-how-to-change-report-prompts)
10. [How To Change Report Formatting](#10-how-to-change-report-formatting)
11. [API Endpoints Reference](#11-api-endpoints-reference)
12. [Configuration Reference](#12-configuration-reference)
13. [Dependency Map (If I Change X, What Breaks?)](#13-dependency-map)
14. [Workflow Modification — Step-by-Step](#14-workflow-modification--step-by-step)
15. [Editing an Existing Tool — Complete Walkthrough](#15-editing-an-existing-tool--complete-walkthrough)
16. [Adding a New Tool — Complete Walkthrough](#16-adding-a-new-tool--complete-walkthrough)
17. [Complete "If I Change X" Quick Reference](#17-complete-if-i-change-x-quick-reference)
18. [Plug-and-Play: Replacing Demo Tools with Real APIs](#18-plug-and-play-replacing-demo-tools-with-real-apis)
19. [Plug-and-Play: Worked Examples (Real APIs Per Agent)](#19-plug-and-play-worked-examples-real-apis-per-agent)
20. [**TOOL-BY-TOOL REFERENCE**](#20-tool-by-tool-reference-every-tool-what-it-does-how-to-change-it) — full detail per tool: what it does, how it works, how to change it
21. [**STRICT IDENTITY MATCHING** (Issue 1 — false-positive fix)](#21-strict-identity-matching-issue-1--the-false-positive-fix) — decision tree, two-tier output, phone-owner resolution, how to tune it
22. [**LE REPORT TEMPLATES** (Issue 2 — report-style fix)](#22-le-report-templates-issue-2--the-report-style-fix) — the 4 templates, the JSON schema, how to edit/add templates, how to turn them on/off
23. [Configuration — New Flags (added with Issues 1 & 2)](#23-configuration--new-flags-added-with-issues-1--2) — the 17 new config flags
24. [**CHANGELOG** — every fix, every detail](#24-changelog--every-fix-every-detail) — L1-L4, R1-R9, G1-G4, §3.x; test coverage; full file inventory
25. [**TROUBLESHOOTING & DIAGNOSTICS**](#25-troubleshooting--diagnostics--if-you-see-x-look-at-y) — "if you see X, look at Y"; the `eligibility_reason` breadcrumb; SQLite diagnostic queries; false-positive / false-negative / citation / rating / hedging diagnosis flows; recovery; performance knobs

---

## 1. QUICK START

```bash
cd V2/backend
pip install -r requirements.txt
python main.py
# Server: http://0.0.0.0:8000
# Frontend: open browser to http://localhost:8000
```

### Required `.env` File

```bash
# At least ONE LLM key required:
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=AIza...

# Optional overrides:
LLM_PROVIDER=openai          # openai | anthropic | google | local
LLM_MODEL=gpt-4o-mini        # Model name
LLM_BASE_URL=                 # Custom endpoint (for local/proxy)
LLM_API_KEY=                  # If using custom endpoint
```

### Verify

```bash
curl http://localhost:8000/          # Health check
curl http://localhost:8000/agents    # List all 15 agents
```

---

## 2. ARCHITECTURE OVERVIEW

```
+------------------------------------------------------------------+
|                       FRONTEND (index.html)                        |
|  Chat UI | Target Card | LEO | Targeting | Trace | Results        |
+------------------------------+-----------------------------------+
                               |
                          HTTP / SSE
                               |
+------------------------------v-----------------------------------+
|                  api_server.py (FastAPI, 60+ endpoints)           |
+--+--------+--------+--------+--------+--------+---------+--------+
   |        |        |        |        |        |         |
   v        v        v        v        v        v         v
+------+ +------+ +------+ +------+ +------+ +-------+ +------+
|Orch- | |LEO   | |Targ- | |Chat  | |Event | |Target | |Result|
|estr- | |Mana- | |eting | |Mana- | |Bus   | |Store  | |Store |
|ator  | |ger   | |Mana- | |ger   | |(SSE) | |(SQL)  | |(SQL) |
+--+---+ +--+---+ |ger   | +--+---+ +------+ +-------+ +------+
   |        |      +--+---+    |
   |        |         |        |
   v        v         v        v
+--+--------+---------+--------+---+
|       DossierPipeline            |
|  7 steps: INPUT→EXPAND→COLLECT→  |
|  ENRICH→COMPILE→ASSESS→FINAL    |
+--+-------------------------------+
   |
   v
+--+-------------------------------+
|         Living Dossier           |
|  Formatters, Sections, Summaries |
|  Assessments, Narrative, ExecSum |
+--+-------------------------------+
   |
   v
+--+-------------------------------+
|     14 Agents (+ SNA + Unified)  |
|  Each has: plan, tools, database |
+--+-------------------------------+
   |
   +---> Plan Executor (deterministic JSON steps)
   +---> Tool Calling Engine (autonomous LLM loop)
   +---> Sub-Agent Processor (parallel chunked workers)
   |
   v
+--+-------------------------------+
|           Tool Layer             |
|  Search tools, query tools,      |
|  analysis tools, utility tools   |
+--+-------------------------------+
   |
   v
+--+-------------------------------+
|        Data Layer                |
|  15 text DBs | voter.csv/SQLite  |
|  target_store.db | results.db    |
|  dossiers/ (file system)         |
+----------------------------------+
```

### Three Workflows

| Workflow | Agents Used | Output | Key Difference |
|----------|-------------|--------|----------------|
| **Targeting** | 7 data + 7 assessment + SNA | Full dossier + exec summary + narrative (11 sections from assessments) | Runs ALL 7 pipeline steps |
| **Trace** | 7 data agents only | Trace report (lighter) | Skips ASSESS step (step 6), narrative from data sources |
| **LEO** | 7 data agents only | Dossier + narrative (no assessments) | Skips ASSESS step, adds scheduling |

---

## 3. THE LIVING DOSSIER PIPELINE

### What Replaced What

The old architecture had:
- `report_synthesizer.py` (DELETED)
- `targeting_report_synthesizer.py` (DELETED)
- `agent_consolidator.py` (DELETED)
- `fact_extractor.py` (DELETED)
- `dossier_generator.py` (DELETED)
- `memory_engine.py` (DELETED)
- `large_data_processor.py` (DELETED)
- `generic_entity_extractor.py` (DELETED)

**ALL replaced by TWO files:**
- `tools/shared/living_dossier.py` (2024 lines) — Core engine
- `tools/shared/dossier_pipeline.py` (1160 lines) — Pipeline controller

### The 7 Steps

```
STEP 1: INPUT
    - Create dossier directory: data/dossiers/{target_id}/
    - Write identifiers/header.md (name, phone, passport, etc.)

STEP 2: EXPAND
    - name_variation_generator → 18+ Arabic/English transliterations
    - correlation_tool → find linked phones/IMEIs/IMSIs
    - Store expanded identifiers for all subsequent searches

STEP 3: COLLECT (parallel, 7 data agents)
    - For each data agent (sigint, travel, osint, intel_reports, fisa, cne, geoint):
      - agent.search(identifiers) via plan_executor
      - living_dossier.write_agent_section(results) → formats to markdown
      - living_dossier.summarize_section() → LLM summary (1-2K chars)
    - Writes: data/dossiers/{target_id}/sections/01_sigint.md ... 07_geoint.md

STEP 4: ENRICH (max 3 iterations)
    - contact_search → find new phone numbers in results
    - For each new identifier → search relevant agents again → append to sections
    - enrichment_tracker tracks iterations until convergence

STEP 5: COMPILE (zero LLM)
    - compile_cross_references() → Python regex across all sections
    - compile_contact_network() → phone extraction + name resolution
    - compile_voter_results() → voter DB lookup
    - compile_dossier() → concatenate all sections into versioned file
    - Writes: data/dossiers/{target_id}/sections/08_meta.md

STEP 6: ASSESS (7 assessment types — SKIPPED in trace/LEO)
    - For each assessment (access, accessibility, motivation, etc.):
      - Read compiled dossier
      - Chunk into 3K char segments
      - _is_pure_index_chunk() → skip table-of-contents chunks
      - Regex pre-scan with keywords → prioritize relevant chunks
      - For each relevant chunk: LLM scan for assessment-specific facts
      - Compile findings into assessment markdown
    - Writes: assess_access.md, assess_accessibility.md, etc.

STEP 7: FINAL
    - generate_executive_summary(dossier_text, identifiers) → 4000 tokens
    - generate_narrative_report(dossier_text, identifiers) → 11 sections × 2000 tokens
    - Store report in target_store
    - Stream final event via SSE
```

### Dossier Disk Structure

```
data/dossiers/{target_id}/
    identifiers/
        header.md              # Target identity (name, phone, passport)
    sections/
        01_sigint.md           # SIGINT formatted records + summary
        02_travel.md           # Travel formatted records + summary
        03_cne.md              # CNE formatted records + summary
        04_fisa.md             # FISA formatted records + summary
        05_osint.md            # OSINT formatted records + summary
        06_intel_reports.md    # Intel reports formatted + summary
        07_geoint.md           # GEOINT formatted records + summary
        08_meta.md             # Cross-references + contact network + voter
    assessments/               # (targeting workflow only)
        assess_access.md
        assess_accessibility.md
        assess_motivation.md
        assess_suitability.md
        assess_security.md
        assess_counter_intel.md
        assess_pattern_of_life.md
    dossier_v1.md              # First compiled version
    dossier_v2.md              # After enrichment
    ...
```

---

## 4. DIRECTORY STRUCTURE

```
V2/
├── frontend/
│   └── index.html              # Complete single-page UI (2694 lines)
│
├── backend/
│   ├── main.py                 # Entry point: uvicorn server
│   ├── api_server.py           # FastAPI routes, SSE, all CRUD (1640 lines)
│   ├── config.py               # All configuration (94 lines)
│   ├── llm_provider.py         # Multi-provider LLM with retry (324 lines)
│   │
│   ├── agents/
│   │   ├── orchestrator.py     # Central hub (1558 lines)
│   │   ├── base_agent.py       # Base class (615 lines)
│   │   ├── sigint_agent.py     # CDR/communications
│   │   ├── travel_agent.py     # Flight/border records
│   │   ├── osint_agent.py      # Open source intelligence
│   │   ├── intel_reports_agent.py # Intelligence reports
│   │   ├── fisa_agent.py       # FISA surveillance
│   │   ├── cne_agent.py        # Computer network exploitation
│   │   ├── geoint_agent.py     # Geospatial intelligence
│   │   ├── access_agent.py     # Assessment: access
│   │   ├── accessibility_agent.py # Assessment: accessibility
│   │   ├── motivation_agent.py # Assessment: motivation
│   │   ├── suitability_agent.py # Assessment: suitability
│   │   ├── security_agent.py   # Assessment: security
│   │   ├── counter_intel_agent.py # Assessment: counter-intel
│   │   ├── pattern_of_life_agent.py # Assessment: pattern of life
│   │   ├── sna_agent.py        # Social network analysis (167 lines)
│   │   ├── unified_agent.py    # Cross-database unified search
│   │   ├── leo_manager.py      # LEO workflow CRUD (215 lines)
│   │   ├── targeting_manager.py # Targeting workflow CRUD (379 lines)
│   │   └── chat_manager.py     # Chat session management (99 lines)
│   │
│   ├── tools/
│   │   ├── base_tool.py        # BaseTool abstract class
│   │   ├── tool_definitions.py # JSON schemas for LLM tool calling
│   │   ├── tool_registry.py    # Dynamic tool discovery
│   │   │
│   │   ├── shared/
│   │   │   ├── living_dossier.py         # Core dossier engine (2024 lines)
│   │   │   ├── dossier_pipeline.py       # 7-step pipeline (1160 lines)
│   │   │   ├── trace_report_synthesizer.py # Trace reports (523 lines)
│   │   │   ├── plan_executor.py          # JSON plan execution (1051 lines)
│   │   │   ├── tool_calling_engine.py    # LLM tool loop (844 lines)
│   │   │   ├── sub_agent_processor.py    # Parallel chunks (460 lines)
│   │   │   ├── target_store.py           # SQLite persistence (2195 lines)
│   │   │   ├── results_store.py          # Sub-agent results (427 lines)
│   │   │   ├── event_bus.py              # SSE streaming (133 lines)
│   │   │   ├── agent_query_tool.py       # Cross-agent queries (389 lines)
│   │   │   ├── contact_search.py         # Cross-DB phone lookup (258 lines)
│   │   │   ├── cross_target_linker.py    # Shared identifiers (801 lines)
│   │   │   ├── entity_resolver.py        # Duplicate detection (510 lines)
│   │   │   ├── enrichment_tracker.py     # Enrichment iterations (349 lines)
│   │   │   ├── identity_scorer.py        # Confidence scoring (372 lines)
│   │   │   ├── name_variation_generator.py # Arabic/English names (315 lines)
│   │   │   ├── voter_db_tool.py          # Voter registration (128 lines)
│   │   │   ├── translation_tool.py       # Translation (243 lines)
│   │   │   └── datetime_tool.py          # Date math (157 lines)
│   │   │
│   │   ├── agent_specific/
│   │   │   ├── base_agent_search_tool.py # Parent search class (205 lines)
│   │   │   ├── sigint/
│   │   │   │   ├── sigint_search_tool.py
│   │   │   │   ├── sigint_adapter.py
│   │   │   │   ├── cdr_query.py
│   │   │   │   └── correlation_tool.py
│   │   │   ├── travel/
│   │   │   │   ├── travel_search_tool.py
│   │   │   │   ├── travel_query.py
│   │   │   │   └── co_traveler_search.py
│   │   │   ├── fisa/
│   │   │   │   ├── fisa_search_tool.py
│   │   │   │   └── fisa_query.py
│   │   │   ├── cne/cne_search_tool.py
│   │   │   ├── geoint/geoint_search_tool.py
│   │   │   ├── osint/osint_search_tool.py
│   │   │   ├── intel_reports/intel_reports_search_tool.py
│   │   │   └── sna/
│   │   │       ├── build_network.py
│   │   │       └── analyze_network.py
│   │   │
│   │   ├── database_connectors/
│   │   │   ├── text_file_connector.py
│   │   │   └── json_connector.py
│   │   │
│   │   └── api_integrations/
│   │       ├── osint_tools.py
│   │       └── reverse_lookup.py
│   │
│   ├── plans/                  # 14 JSON execution plans
│   │   ├── sigint.json, travel.json, osint.json, intel_reports.json
│   │   ├── fisa.json, cne.json, geoint.json
│   │   ├── access.json, accessibility.json, motivation.json
│   │   ├── suitability.json, security.json
│   │   ├── counter_intel.json, pattern_of_life.json
│   │
│   ├── schemas/                # 9 database schema JSONs
│   │   ├── sigint_db.json, travel_db.json, osint_db.json
│   │   ├── intel_reports_db.json, fisa_db.json, cne_db.json
│   │   ├── geoint_db.json, correlation_db.json, voter_db.json
│   │
│   ├── data/                   # All databases
│   │   ├── sigint_db.txt, travel_db.txt, osint_db.txt
│   │   ├── intel_reports_db.txt, fisa_db.txt, cne_db.txt
│   │   ├── geoint_db.txt, correlation_db.txt
│   │   ├── access_db.txt, accessibility_db.txt, motivation_db.txt
│   │   ├── suitability_db.txt, security_db.txt, counter_intel_db.txt
│   │   ├── pattern_of_life_db.txt
│   │   ├── voter.csv, voter_registration.db
│   │   ├── leos.json, targeting_workflows.json, trace_workflows.json
│   │   ├── generate_databases.py
│   │   └── dossiers/           # Output dossier folders
│   │
│   ├── test_e2e.py             # End-to-end tests
│   └── test_reallife.py        # Real API tests
```

---

## 5. EVERY SCRIPT EXPLAINED

### Core (4 files)

| File | What It Does | Called By | Calls |
|------|-------------|-----------|-------|
| `main.py` | Starts uvicorn server | Shell | api_server |
| `api_server.py` | 60+ REST endpoints, CORS, SSE streaming | uvicorn | orchestrator, leo_manager, targeting_manager, chat_manager, event_bus, target_store |
| `config.py` | Loads .env, defines ALL paths and constants | EVERYONE | os, dotenv |
| `llm_provider.py` | Multi-provider LLM (OpenAI/Anthropic/Google/local), retry, thread-safe metrics | plan_executor, tool_calling_engine, sub_agent_processor, living_dossier, trace_report_synthesizer | openai, anthropic, google |

### Agents (20 files)

| File | What It Does | Called By |
|------|-------------|-----------|
| `orchestrator.py` | Registers 15 agents + tools + plans; runs pipelines | api_server |
| `base_agent.py` | Base class: search(), chat(), plan executor integration | All agent subclasses |
| `sigint_agent.py` | CDR/communications; overrides chat() for plan routing | orchestrator |
| `travel_agent.py` | Travel records | orchestrator |
| `osint_agent.py` | Open-source intel; custom search splitting | orchestrator |
| `intel_reports_agent.py` | Intelligence reports | orchestrator |
| `fisa_agent.py` | FISA surveillance records | orchestrator |
| `cne_agent.py` | Computer network exploitation | orchestrator |
| `geoint_agent.py` | Geospatial intelligence | orchestrator |
| `access_agent.py` | Assessment: access vectors | orchestrator |
| `accessibility_agent.py` | Assessment: physical accessibility | orchestrator |
| `motivation_agent.py` | Assessment: MICE framework | orchestrator |
| `suitability_agent.py` | Assessment: recruitment suitability | orchestrator |
| `security_agent.py` | Assessment: officer safety | orchestrator |
| `counter_intel_agent.py` | Assessment: CI risks | orchestrator |
| `pattern_of_life_agent.py` | Assessment: daily patterns | orchestrator |
| `sna_agent.py` | Social network analysis (no database) | orchestrator |
| `unified_agent.py` | Cross-database unified search | orchestrator |
| `leo_manager.py` | LEO workflow CRUD + pipeline execution | api_server |
| `targeting_manager.py` | Targeting/Trace workflow CRUD + pipeline execution | api_server |
| `chat_manager.py` | Multi-session chat with SQLite persistence | api_server |

### Shared Tools (19 files)

| File | What It Does | Called By |
|------|-------------|-----------|
| `living_dossier.py` | Core dossier engine: formatters, summaries, assessments, narrative, exec summary | dossier_pipeline |
| `dossier_pipeline.py` | 7-step pipeline controller | orchestrator, leo_manager, targeting_manager |
| `trace_report_synthesizer.py` | Trace workflow report with 5 narrative topics | dossier_pipeline |
| `plan_executor.py` | Deterministic JSON plan execution + sub-agent chunking | base_agent |
| `tool_calling_engine.py` | Autonomous LLM tool loop (max 10 iterations) | base_agent, chat_manager |
| `sub_agent_processor.py` | Concurrent chunked workers (up to 50 parallel) | plan_executor |
| `target_store.py` | SQLite: 20+ tables, thread-safe | orchestrator, api_server, dossier_pipeline |
| `results_store.py` | SQLite: sub-agent processing results | api_server, sub_agent_processor |
| `event_bus.py` | SSE event streaming | api_server, dossier_pipeline |
| `agent_query_tool.py` | Cross-agent query bridge + cache | plan_executor, tool_calling_engine |
| `contact_search.py` | Cross-DB phone contact discovery | dossier_pipeline |
| `cross_target_linker.py` | Shared identifiers across targets | living_dossier |
| `entity_resolver.py` | Jaccard similarity duplicate detection | orchestrator |
| `enrichment_tracker.py` | Enrichment iteration tracking (max 3) | dossier_pipeline |
| `identity_scorer.py` | Record confidence scoring (pure Python) | plan_executor |
| `name_variation_generator.py` | Arabic/English name expansion | plan_executor, tool_calling_engine |
| `voter_db_tool.py` | Iraq voter registration SQLite search | plan_executor, tool_calling_engine |
| `translation_tool.py` | Language detection + translation via LLM | plan_executor, tool_calling_engine |
| `datetime_tool.py` | Date math + data type ranges | plan_executor, tool_calling_engine |

---

## 6. PER-AGENT WALKTHROUGH (What To Change & Where)

### SIGINT AGENT — Complete Modification Guide

**I want to change what SIGINT searches for:**
1. Open `plans/sigint.json`
2. Find step with `"tool": "sigint_search"` (usually step_1)
3. Modify `params.query` — this is what gets searched in the database
4. To add a new search step, add a new object to the `steps` array

**I want to change how SIGINT results are analyzed:**
1. Open `plans/sigint.json`
2. Find steps with `"action": "analyze"` (usually step_3 and step_4)
3. Modify the `"prompt"` string — this is the LLM instruction

**I want to change how SIGINT data appears in the dossier:**
1. Open `tools/shared/living_dossier.py`
2. Search for `def _format_sigint` (around line 200)
3. Modify the markdown template inside this method
4. Output goes to `data/dossiers/{target_id}/sections/01_sigint.md`

**I want to change how SIGINT is assessed:**
1. The SIGINT data is assessed by ALL 7 assessment types (not just one)
2. To change what pattern_of_life looks for in SIGINT data, see assessment prompts below

**I want to change SIGINT's chat behavior:**
1. Open `agents/sigint_agent.py`
2. The `chat()` method override routes phone queries through the JSON plan
3. Modify `_extract_identifiers()` to change what triggers plan execution

**Files affected by SIGINT changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/sigint.json` |
| Search tool logic | `tools/agent_specific/sigint/sigint_search_tool.py`, `sigint_adapter.py` |
| CDR parsing | `tools/agent_specific/sigint/cdr_query.py` |
| Phone correlation | `tools/agent_specific/sigint/correlation_tool.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_sigint()` |
| Section order | `tools/shared/living_dossier.py` → `SECTION_ORDER` (index 0: "01") |
| Database file | `data/sigint_db.txt` |
| Schema | `schemas/sigint_db.json` |
| Agent class | `agents/sigint_agent.py` |

---

### TRAVEL AGENT — Complete Modification Guide

**I want to change what Travel searches for:**
1. Open `plans/travel.json`
2. Find step with `"tool": "travel_search"`
3. Modify `params.query`

**I want to change how Travel results are analyzed:**
1. Open `plans/travel.json`
2. Find `"action": "analyze"` steps
3. Modify the `"prompt"` strings

**I want to change how Travel data appears in the dossier:**
1. Open `tools/shared/living_dossier.py`
2. Search for `def _format_travel`
3. Modify the markdown template

**I want to change co-traveler detection:**
1. Open `tools/agent_specific/travel/co_traveler_search.py`
2. The `execute()` method contains the matching logic (flight + date)
3. Modify the matching criteria

**Files affected by Travel changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/travel.json` |
| Search tool logic | `tools/agent_specific/travel/travel_search_tool.py` |
| Record parsing | `tools/agent_specific/travel/travel_query.py` |
| Co-traveler logic | `tools/agent_specific/travel/co_traveler_search.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_travel()` |
| Section order | `tools/shared/living_dossier.py` → `SECTION_ORDER` (index 1: "02") |
| Database file | `data/travel_db.txt` |
| Schema | `schemas/travel_db.json` |
| Agent class | `agents/travel_agent.py` |

---

### OSINT AGENT — Complete Modification Guide

**I want to change what OSINT searches for:**
1. Open `plans/osint.json`
2. Modify the search step params

**I want to change how OSINT splits searches:**
1. Open `agents/osint_agent.py`
2. The agent overrides search() to split phone/name queries
3. Modify the splitting logic in the `search()` method

**I want to change how OSINT data appears in the dossier:**
1. Open `tools/shared/living_dossier.py`
2. Search for `def _format_osint`
3. Modify the markdown template

**Files affected by OSINT changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/osint.json` |
| Search splitting | `agents/osint_agent.py` → `search()` override |
| Search tool logic | `tools/agent_specific/osint/osint_search_tool.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_osint()` |
| Database file | `data/osint_db.txt` |
| Schema | `schemas/osint_db.json` |

---

### INTEL REPORTS AGENT — Complete Modification Guide

**Files affected by Intel Reports changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/intel_reports.json` |
| Search tool logic | `tools/agent_specific/intel_reports/intel_reports_search_tool.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_intel_reports()` |
| Database file | `data/intel_reports_db.txt` |
| Schema | `schemas/intel_reports_db.json` |
| Agent class | `agents/intel_reports_agent.py` |

---

### FISA AGENT — Complete Modification Guide

**I want to change FISA auto-expansion (which phones get searched):**
1. Open `agents/fisa_agent.py` or `plans/fisa.json`
2. FISA has special logic for expanding selectors from SIGINT correlation
3. Only PHONE numbers are expanded (not IMEIs/IMSIs — this was a bug fix)

**Files affected by FISA changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/fisa.json` |
| Search tool logic | `tools/agent_specific/fisa/fisa_search_tool.py` |
| FISA record parsing | `tools/agent_specific/fisa/fisa_query.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_fisa()` |
| Database file | `data/fisa_db.txt` |
| Schema | `schemas/fisa_db.json` |

---

### CNE AGENT — Complete Modification Guide

**Files affected by CNE changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/cne.json` |
| Search tool logic | `tools/agent_specific/cne/cne_search_tool.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_cne()` |
| Database file | `data/cne_db.txt` |
| Schema | `schemas/cne_db.json` |
| Agent class | `agents/cne_agent.py` (custom data_path) |

---

### GEOINT AGENT — Complete Modification Guide

**Files affected by GEOINT changes:**
| Change Type | Files |
|-------------|-------|
| Search behavior | `plans/geoint.json` |
| Search tool logic | `tools/agent_specific/geoint/geoint_search_tool.py` |
| Dossier formatting | `tools/shared/living_dossier.py` → `_format_geoint()` |
| Database file | `data/geoint_db.txt` |
| Schema | `schemas/geoint_db.json` |

---

### ACCESS ASSESSMENT AGENT — Complete Modification Guide

**I want to change what Access looks for:**
1. Open `tools/shared/living_dossier.py`
2. Find `ASSESSMENT_TYPES` dict (line ~54)
3. Find the `"access"` key
4. Modify the `"prompt"` string — this is what the LLM scans for in each chunk
5. Modify the `"keywords"` list — these prioritize which chunks get scanned first

**I want to change the Access plan execution:**
1. Open `plans/access.json`
2. This contains the steps for access assessment when using plan_executor mode
3. Add/remove `query_agent` steps to change which data agents are queried

**Actual prompt (in living_dossier.py):**
```python
"access": {
    "title": "ACCESS ASSESSMENT",
    "prompt": (
        "Evaluate what ACCESS this target can provide. Look for:\n"
        "1. POSITION — job title, organization, role, responsibilities\n"
        "2. INFORMATION ACCESS — classified data, trade secrets, government intel\n"
        "3. NETWORK ACCESS — contacts in government, military, organizations\n"
        "4. PHYSICAL ACCESS — facilities, restricted areas, infrastructure\n"
        "5. TECHNICAL ACCESS — systems, databases, communications\n"
        "If this chunk contains relevant evidence, extract with record IDs.\n"
        "If not, respond: NO_INDICATORS\n\n"
        "IMPORTANT: Only report SIGNIFICANT findings..."
    ),
    "keywords": ["position", "director", "ministry", "clearance", "access",
                 "department", "manager", "official", "government", "military",
                 "classified", "restricted", "authorized"],
},
```

**Files affected:**
| Change Type | Files |
|-------------|-------|
| What it scans for | `tools/shared/living_dossier.py` → `ASSESSMENT_TYPES["access"]["prompt"]` |
| Priority keywords | `tools/shared/living_dossier.py` → `ASSESSMENT_TYPES["access"]["keywords"]` |
| Display title | `tools/shared/living_dossier.py` → `ASSESSMENT_TYPES["access"]["title"]` |
| Plan execution | `plans/access.json` |
| Agent class | `agents/access_agent.py` |
| Output file | `data/dossiers/{target_id}/sections/assess_access.md` |
| Narrative section | `tools/shared/living_dossier.py` → `NARRATIVE_SECTIONS` tuple with `"assess_access.md"` |

---

### ACCESSIBILITY ASSESSMENT AGENT — Complete Modification Guide

**Same pattern as Access. Key differences:**
- Dict key: `"accessibility"` in `ASSESSMENT_TYPES`
- Scans for: travel patterns, daily routine, public venues, digital presence, security posture
- Keywords: `["travel", "flight", "hotel", "restaurant", "mosque", "routine", "daily", ...]`
- Plan: `plans/accessibility.json`
- Output: `assess_accessibility.md`

---

### MOTIVATION ASSESSMENT AGENT — Complete Modification Guide

**Same pattern. Key differences:**
- Dict key: `"motivation"` in `ASSESSMENT_TYPES`
- Uses MICE framework: Money, Ideology, Compromise, Ego, Family, Career
- Keywords: `["debt", "loan", "salary", "money", "divorce", "ideology", "affair", ...]`
- Plan: `plans/motivation.json`
- Output: `assess_motivation.md`

---

### SUITABILITY ASSESSMENT AGENT

- Dict key: `"suitability"` — Reliability, cooperation, risk tolerance, stability, discretion
- Plan: `plans/suitability.json`
- Output: `assess_suitability.md`

### SECURITY ASSESSMENT AGENT

- Dict key: `"security"` — Physical security, weapons, counter-surveillance, encrypted comms, hostile environment
- Plan: `plans/security.json`
- Output: `assess_security.md`

### COUNTER-INTEL ASSESSMENT AGENT

- Dict key: `"counter_intel"` — Loyalty, deception, hostile service connections, tradecraft, double agent indicators
- Plan: `plans/counter_intel.json`
- Output: `assess_counter_intel.md`

### PATTERN OF LIFE ASSESSMENT AGENT

- Dict key: `"pattern_of_life"` — Daily routine, weekly patterns, communication patterns, travel patterns, predictability rating
- Plan: `plans/pattern_of_life.json`
- Output: `assess_pattern_of_life.md`

---

### SNA AGENT (Social Network Analysis)

**Special: No database, no plan, no search tool.**
- Synthesizes from ALL other agent results
- Uses `tools/agent_specific/sna/build_network.py` + `analyze_network.py`
- Pure graph algorithms (centrality, clusters)

**To change SNA:**
1. `agents/sna_agent.py` — agent behavior
2. `tools/agent_specific/sna/build_network.py` — how the graph is constructed
3. `tools/agent_specific/sna/analyze_network.py` — graph algorithms

---

## 7. PER-TOOL WALKTHROUGH (What To Change & Where)

### SIGINT Search Tool

| What | File | Exact Location |
|------|------|----------------|
| Search logic | `tools/agent_specific/sigint/sigint_search_tool.py` | `execute()` method |
| Data adapter | `tools/agent_specific/sigint/sigint_adapter.py` | Adapts DB format for search |
| CDR parsing | `tools/agent_specific/sigint/cdr_query.py` | Parse CDR text → structured records |
| Phone correlation | `tools/agent_specific/sigint/correlation_tool.py` | Phone→IMEI→IMSI chains |
| If I change this | Also update: `plans/sigint.json` (if step references change) |

### Travel Search Tool

| What | File | Exact Location |
|------|------|----------------|
| Search logic | `tools/agent_specific/travel/travel_search_tool.py` | `execute()` method |
| Record parsing | `tools/agent_specific/travel/travel_query.py` | Parse travel text → trips |
| Co-traveler | `tools/agent_specific/travel/co_traveler_search.py` | Same flight+date matching |
| If I change this | Also update: `plans/travel.json` |

### FISA Search Tool

| What | File | Exact Location |
|------|------|----------------|
| Search logic | `tools/agent_specific/fisa/fisa_search_tool.py` | `execute()` method |
| FISA parsing | `tools/agent_specific/fisa/fisa_query.py` | Parse FISA orders |
| If I change this | Also update: `plans/fisa.json` |

### CNE / GEOINT / OSINT / Intel Reports Search Tools

All follow the same pattern:
- File: `tools/agent_specific/{agent}/{agent}_search_tool.py`
- Method: `execute()`
- Extends: `base_agent_search_tool.py`
- If changed: also update `plans/{agent}.json`

### Name Variation Generator

| What | File | Exact Location |
|------|------|----------------|
| Arabic/English mappings | `tools/shared/name_variation_generator.py` | Dictionaries at top of class |
| Transliteration rules | Same file | `_generate_arabic_variations()` |
| Max variations | Same file | Return list length |
| If I change this | Nothing else breaks — it's a leaf tool |

### Voter DB Tool

| What | File | Exact Location |
|------|------|----------------|
| Search logic | `tools/shared/voter_db_tool.py` | `execute()` method |
| Database path | `config.py` → `VOTER_CSV_PATH` | Points to `data/voter.csv` |
| If I change this | Nothing else breaks — it's a leaf tool |

### Contact Search Tool

| What | File | Exact Location |
|------|------|----------------|
| Phone discovery logic | `tools/shared/contact_search.py` | `execute()` method |
| Which agents to search | Same file | Agent list in search loop |
| If I change this | Affects enrichment (Step 4 of pipeline) |

### Agent Query Tool (Cross-Agent Bridge)

| What | File | Exact Location |
|------|------|----------------|
| Query routing | `tools/shared/agent_query_tool.py` | `execute()` method |
| Cache prepopulation | Same file | `prepopulate_cache()` method |
| If I change this | Affects ALL assessment agents (they use this to query data agents) |

### Identity Scorer

| What | File | Exact Location |
|------|------|----------------|
| Field weights | `tools/shared/identity_scorer.py` | Top of class (passport=0.95, phone=0.90, etc.) |
| Confidence thresholds | Same file | HIGH≥0.7, MEDIUM=0.4-0.7, LOW<0.4 |
| Name matching | Same file | Jaccard + Arabic variation lookup |
| If I change this | Affects record confidence tags in SQLite |

### Translation Tool

| What | File | Exact Location |
|------|------|----------------|
| Translation logic | `tools/shared/translation_tool.py` | `execute()` method |
| External API config | `config.py` | `TRANSLATION_API_URL`, `TRANSLATION_API_KEY` |
| If I change this | Nothing else breaks — it's a leaf tool |

### Plan Executor

| What | File | Exact Location |
|------|------|----------------|
| Step execution | `tools/shared/plan_executor.py` | `_execute_step()` method |
| Template variable substitution | Same file | `_substitute_templates()` |
| Sub-agent chunking threshold | Same file | `LARGE_PROMPT_THRESHOLD = 6000` |
| MAP prompt (chunk scanning) | Same file | `_chunked_llm_call()` |
| If I change this | Affects ALL agents that use JSON plans (all 14) |

### Tool Calling Engine

| What | File | Exact Location |
|------|------|----------------|
| LLM tool loop | `tools/shared/tool_calling_engine.py` | `run()` method |
| Max iterations | `config.py` | `TOOL_CALLING_MAX_ITERATIONS = 10` |
| Tool definitions | `tools/shared/tool_definitions.py` | JSON schemas list |
| If I change this | Affects chat mode + agents without plans |

### Sub-Agent Processor

| What | File | Exact Location |
|------|------|----------------|
| Worker logic | `tools/shared/sub_agent_processor.py` | `process_chunks()` method |
| Max parallel workers | `config.py` | `SUB_AGENT_MAX_WORKERS = 50` |
| Records per worker | `config.py` | `SUB_AGENT_RECORDS_PER_WORKER = 2` |
| If I change this | Affects large result processing speed |

---

## 8. PER-JSON PLAN WALKTHROUGH

### How Plans Work

Every agent has a JSON plan in `plans/{agent}.json`. When `agent.search()` is called, the `plan_executor.py` reads this JSON and executes steps in order.

### Plan JSON Structure

```json
{
  "version": "4.0",
  "agent_key": "sigint",
  "description": "SIGINT collection and analysis",
  "steps": [
    {
      "id": "step_1",
      "name": "Search CDR records",
      "action": "search",          // or "tool_call", "analyze", "for_each"
      "tool": "sigint_search",
      "params": {
        "query": "{{identifiers}}", // template variable from context
        "max_results": 100
      },
      "output_key": "raw_cdr"      // stored in context for later steps
    },
    {
      "id": "step_2",
      "action": "analyze",
      "prompt": "Analyze: {{raw_cdr}}",  // uses output from step_1
      "max_tokens": 2000,
      "output_key": "analysis"
    }
  ]
}
```

### Template Variables Available

| Variable | What It Contains |
|----------|-----------------|
| `{{identifiers}}` | Full identifiers JSON (name, phone, passport, etc.) |
| `{{phone}}`, `{{name}}`, `{{passport}}` | Individual identifier fields |
| `{{output_key_from_previous_step}}` | Output from any earlier step |
| `{{dossier_context}}` | Compiled dossier text (for assessment plans) |

### Step Actions

| Action | What It Does | Uses LLM? |
|--------|-------------|-----------|
| `search` | Execute a search tool against text DB | No |
| `tool_call` | Execute any registered tool | No |
| `analyze` | Send prompt to LLM for analysis | Yes |
| `for_each` | Iterate over array results | Depends on sub-steps |

### Per-Plan Modification Guide

#### `plans/sigint.json` — SIGINT Plan

**Steps:** correlate → CDR query → network analysis → compile
**To modify:**
1. Open `plans/sigint.json`
2. To change search queries: modify step_1 `params`
3. To change analysis prompts: modify step_3 and step_4 `prompt`
4. To add cross-reference: add a `tool_call` step with `"tool": "query_agent"`
5. To change output compilation: modify the last step's `prompt`

#### `plans/travel.json` — Travel Plan

**Steps:** search → extract routes → analyze patterns → co-travelers → compile
**To modify:**
1. Open `plans/travel.json`
2. To change what's searched: modify step_1 `params`
3. To disable co-traveler: remove the co_traveler step
4. To add a cross-reference step: add `{"action": "tool_call", "tool": "query_agent", "params": {"agent_key": "sigint", "query": "{{phone}}"}}`

#### `plans/osint.json` — OSINT Plan
#### `plans/intel_reports.json` — Intel Reports Plan
#### `plans/fisa.json` — FISA Plan
#### `plans/cne.json` — CNE Plan
#### `plans/geoint.json` — GEOINT Plan

All follow the same pattern: search → analyze → compile. Modify `params` to change search, modify `prompt` to change analysis.

#### `plans/access.json` — Access Assessment Plan

**Key difference from data agents:** Assessment plans query OTHER agents via `query_agent` tool, then receive `{{dossier_context}}` to analyze.

**Steps:** expand identifiers → query intel_reports → query osint → query sigint → query cne → assess → compile
**To change which data agents are queried:**
1. Add/remove `query_agent` steps
2. Modify `params.agent_key` to target different agents

#### `plans/accessibility.json` — Queries: travel, geoint, meta, osint
#### `plans/motivation.json` — Queries: intel_reports, osint, cne, sigint
#### `plans/suitability.json` — Queries: intel_reports, osint, cne, sigint
#### `plans/security.json` — Queries: intel_reports, sigint, geoint, osint, cne, fisa
#### `plans/counter_intel.json` — Queries: sigint, travel, fisa, intel_reports, cne
#### `plans/pattern_of_life.json` — Queries: sigint, travel, geoint, meta, cne

---

## 9. HOW TO CHANGE REPORT PROMPTS

### Executive Summary

**File:** `tools/shared/living_dossier.py`
**Method:** `generate_executive_summary()`

**Step-by-step:**
1. Open `tools/shared/living_dossier.py`
2. Search for `def generate_executive_summary`
3. Find the prompt string passed to `self.llm.chat()`
4. Modify the prompt text
5. Available context: `{target_identity}` (injected as "TARGET: Name (Phone: +xxx)")
6. The compiled dossier text is passed as the user message content
7. To change output length: modify `max_tokens=4000`

---

### Narrative Report (Targeting Mode — 11 sections from assessments)

**File:** `tools/shared/living_dossier.py`
**Method:** `generate_narrative_report()`
**Location:** Line ~1232 (the `if has_assessments:` branch)

**Structure:** List of tuples: `(section_key, title, assess_file, prompt)`

```python
NARRATIVE_SECTIONS = [
    ("executive_overview", "Executive Overview", None,
     "Write a 1-paragraph executive overview of this intelligence target. "
     "State who they are, why they matter, overall risk level, and key recommendation."),
    ("target_background", "Target Background", None,
     "Write 1-2 paragraphs describing who this target is..."),
    ("access", "Access Assessment", "assess_access.md",
     "Write 1-2 paragraphs analyzing what intelligence access..."),
    ("accessibility", "Accessibility Assessment", "assess_accessibility.md",
     "Write 1-2 paragraphs on how accessible this target is..."),
    ("motivation", "Motivation & Vulnerabilities", "assess_motivation.md",
     "Write 1-2 paragraphs analyzing motivation using MICE..."),
    ("suitability", "Suitability Assessment", "assess_suitability.md",
     "Write 1-2 paragraphs on suitability for engagement..."),
    ("security", "Security Assessment", "assess_security.md",
     "Write 1-2 paragraphs on security risks..."),
    ("counter_intel", "Counter-Intelligence Assessment", "assess_counter_intel.md",
     "Write 1-2 paragraphs on CI risks..."),
    ("pattern_of_life", "Pattern of Life", "assess_pattern_of_life.md",
     "Write 1-2 paragraphs describing daily routine..."),
    ("network", "Network Analysis", None,
     "Write 1 paragraph summarizing contact network..."),
    ("recommendations", "Risk Assessment & Recommendations", None,
     "Write 1-2 paragraphs with risk level and recommendations..."),
]
```

**Step-by-step to modify:**
1. To change section prompt: modify the 4th element (prompt string) in the tuple
2. To change section title: modify the 2nd element
3. To change which assessment file feeds it: modify the 3rd element (filename or None)
4. To add a section: add a new tuple to the list
5. To remove a section: remove the tuple
6. To reorder: move tuples up/down in the list

---

### Narrative Report (LEO/Trace Mode — 11 sections from data sources)

**File:** `tools/shared/living_dossier.py`
**Method:** `generate_narrative_report()`
**Location:** Line ~1270 (the `else:` branch — no assessments)

```python
NARRATIVE_SECTIONS = [
    ("executive_overview", "Executive Overview", None, "..."),
    ("target_background", "Target Background", None, "..."),
    ("communications", "Communications & SIGINT Analysis", "01_sigint.md", "..."),
    ("travel", "Travel & Movement Patterns", "02_travel.md", "..."),
    ("digital", "Digital Footprint & Device Data", "03_cne.md", "..."),
    ("fisa", "Legal Intercepts & FISA", "04_fisa.md", "..."),
    ("osint", "Open Source Intelligence", "05_osint.md", "..."),
    ("intel", "Intelligence Reporting", "06_intel_reports.md", "..."),
    ("geoint", "Geospatial Intelligence", "07_geoint.md", "..."),
    ("network", "Network Analysis", None, "..."),
    ("recommendations", "Risk Assessment & Recommendations", None, "..."),
]
```

Same modification pattern as targeting mode.

---

### Trace Report Narrative Topics

**File:** `tools/shared/trace_report_synthesizer.py`
**Method:** `_generate_narrative()`
**Location:** Line ~289

**Structure:** Tuples of `(topic_key, title, agent_keys_list, prompt)`

```python
NARRATIVE_TOPICS = [
    ("communications", "Communications & SIGINT Activity",
     ["sigint"],
     "Write 2-3 paragraphs about the target's communications activity..."),
    ("travel", "Travel & Movement Patterns",
     ["travel", "geoint"],
     "Write 2-3 paragraphs about the target's travel and movement patterns..."),
    ("digital", "Digital Footprint & Device Data",
     ["cne", "osint"],
     "Write 1-2 paragraphs about the target's digital footprint..."),
    ("intel_reports", "Intelligence Reporting",
     ["intel_reports"],
     "Write 1-2 paragraphs summarizing intelligence reports..."),
    ("legal_intercepts", "Legal Intercepts & FISA",
     ["fisa"],
     "Write 1-2 paragraphs about FISA or legal intercept records..."),
]
```

**Step-by-step to modify:**
1. To change which agents feed a topic: modify the 3rd element (agent_keys list)
2. To change the LLM prompt: modify the 4th element
3. To add a topic: add a new tuple
4. Topics generate in PARALLEL (ThreadPoolExecutor)

---

### Assessment Scan Prompts (7 types)

**File:** `tools/shared/living_dossier.py`
**Dict:** `ASSESSMENT_TYPES` (starts at line ~54)

**Step-by-step to modify ANY assessment:**
1. Open `tools/shared/living_dossier.py`
2. Find `ASSESSMENT_TYPES = {`
3. Find the key you want to change: `"access"`, `"accessibility"`, `"motivation"`, `"suitability"`, `"security"`, `"counter_intel"`, `"pattern_of_life"`
4. Modify the `"prompt"` string — numbered criteria that the LLM looks for
5. Modify the `"keywords"` list — chunks with these words get scanned FIRST
6. The `"title"` becomes the heading in the assessment output

**Assessment prompt format (all follow this pattern):**
```python
"prompt": (
    "Evaluate {WHAT} for this target. Look for:\n"
    "1. CRITERION_1 — what to look for\n"
    "2. CRITERION_2 — what to look for\n"
    "...\n"
    "If this chunk contains relevant evidence, extract with record IDs.\n"
    "If not, respond: NO_INDICATORS\n\n"
    "IMPORTANT: Only report SIGNIFICANT findings..."
),
```

---

### Section Summaries

**File:** `tools/shared/living_dossier.py`
**Methods:** `summarize_section()` → `_single_summary()` and `_chunked_summary()`

**Step-by-step:**
1. `_single_summary()` — used when section fits in one chunk
   - Find the prompt string asking for "concise summary"
   - Receives `identifiers` for target identity context
2. `_chunked_summary()` — MAP/REDUCE for large sections
   - MAP prompt: "Extract relevant facts from this chunk"
   - REDUCE prompt: "Synthesize into coherent summary"
   - Both accept `identifiers` to inject target identity

---

## 10. HOW TO CHANGE REPORT FORMATTING

### Dossier Section Formatters

**File:** `tools/shared/living_dossier.py`

Each data agent has a dedicated formatter converting raw records into markdown:

| Agent | Method | Output Format |
|-------|--------|---------------|
| SIGINT | `_format_sigint()` | `#### CONTACT: +phone (N interactions)\n- Date, duration, type` |
| Travel | `_format_travel()` | `#### Record: Departure→Arrival\n- Date, flight, passport` |
| FISA | `_format_fisa()` | `#### Report: SELECTOR\n- Content, dates` |
| CNE | `_format_cne()` | `#### Record: Operation\n- Files, tools, dates` |
| Intel Reports | `_format_intel_reports()` | `#### Report: Title\n- Source, content` |
| OSINT | `_format_osint()` | `#### Record: Source\n- Content, entities` |
| GEOINT | `_format_geoint()` | `#### Record: Location\n- Coordinates, imagery` |
| Generic | `_format_generic()` | Fallback key-value formatting |

**Step-by-step to change formatting:**
1. Open `tools/shared/living_dossier.py`
2. Search for `def _format_{agent}` (e.g., `_format_sigint`)
3. The method receives a list of raw records
4. Modify the markdown template/structure inside
5. Output goes to `data/dossiers/{target_id}/sections/{NN}_{agent}.md`
6. Changes affect ALL future dossier compilations

### Section Order

**File:** `tools/shared/living_dossier.py`
**Constant:** `SECTION_ORDER` (line ~39)

```python
SECTION_ORDER = [
    ("sigint", "01", "SIGINT INTELLIGENCE"),
    ("travel", "02", "TRAVEL INTELLIGENCE"),
    ("cne", "03", "CNE INTELLIGENCE"),
    ("fisa", "04", "FISA INTELLIGENCE"),
    ("osint", "05", "OPEN SOURCE INTELLIGENCE"),
    ("intel_reports", "06", "INTELLIGENCE REPORTS"),
    ("geoint", "07", "GEOSPATIAL INTELLIGENCE"),
    ("meta", "08", "METADATA INTELLIGENCE"),
]
```

**Step-by-step to change order:**
1. Swap the number strings (2nd element) between tuples
2. Files are named: `{number}_{agent_key}.md`

### Cross-Reference Format

**Method:** `compile_cross_references()` in `living_dossier.py`
- Pure Python regex (zero LLM)
- Scans all section files for shared identifiers (phones, IMEIs, emails, passports)
- Outputs markdown table: `| Identifier | Type | Found In |`

**Step-by-step to modify:**
1. Find `compile_cross_references()` or `_compile_cross_references()`
2. To add new identifier types: add regex patterns
3. To change output format: modify the markdown template at end of method
4. To exclude target's own identifiers: the method accepts `identifiers` param for filtering

### Contact Network Format

**Method:** `compile_contact_network()` in `living_dossier.py`
- Pure Python (zero LLM)
- Extracts phone numbers from all sections
- Resolves names via `_resolve_contact_names()` (regex proximity matching)
- Counts interactions per agent

**Output format:**
```markdown
## Contact Network

### Known Contacts
- **Ahmad Hassan** (+964-770-xxx) — 47 interactions (SIGINT: 35, FISA: 12)
- **Unknown** (+964-750-xxx) — 8 interactions (SIGINT: 8)
```

---

## 11. API ENDPOINTS REFERENCE

### Core Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/` | Health check |
| GET | `/agents` | List all 15 agents |
| POST | `/target-card` | Run full pipeline (SSE) |
| POST | `/single-agent` | Query one agent |
| POST | `/agentic-search` | Free-text tool-calling search |

### LEO Workflows

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/leo/create` | Create LEO |
| GET | `/leo/list` | List all LEOs |
| POST | `/leo/run/{leo_id}` | Execute LEO pipeline |
| GET | `/leo/{leo_id}/results` | Get results |
| DELETE | `/leo/delete/{leo_id}` | Delete LEO |
| POST | `/leo/schedule/{leo_id}` | Schedule periodic runs |

### Targeting Workflows

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/targeting/create` | Create targeting workflow |
| GET | `/targeting/list` | List all |
| POST | `/targeting/run/{wf_id}` | Execute targeting pipeline |
| GET | `/targeting/{wf_id}/results` | Get results |
| DELETE | `/targeting/{wf_id}` | Delete workflow |

### Trace Workflows

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/trace/create` | Create trace |
| GET | `/trace/list` | List all |
| POST | `/trace/run/{wf_id}` | Execute trace pipeline |
| GET | `/trace/{tr_id}/results` | Get trace report |

### Chat

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/chat/start` | Start chat session |
| POST | `/chat/message` | Send message |
| GET | `/chat/history/{session_id}` | Get history |

### Dossier

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/targets/{target_id}/dossier` | Compiled dossier text |
| GET | `/targets/{target_id}/dossier/sections` | List section files |
| GET | `/targets/{target_id}/dossier/sections/{name}` | Single section |
| GET | `/targets/{target_id}/dossier/versions` | Version history |
| GET | `/targets/{target_id}/dossier/narrative` | Narrative report |

### Events (SSE)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/events/{run_id}` | Live SSE stream |
| GET | `/events/{run_id}/history` | Past events |
| GET | `/pipeline/result/{run_id}` | Final result |

### Targets & Records

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/targets` | All targets |
| GET | `/targets/{target_id}/records` | All records |
| DELETE | `/targets/{target_id}/records/{record_id}` | Soft-delete |
| GET | `/targets/{target_id}/reports/latest` | Latest report |

---

## 12. CONFIGURATION REFERENCE

**File:** `config.py` (94 lines)

### All Settings

| Variable | Default | What It Controls |
|----------|---------|-----------------|
| **LLM** | | |
| `LLM_BASE_URL` | "" | Custom LLM endpoint |
| `LLM_API_KEY` | "" | API key for custom endpoint |
| `LLM_MODEL` | "" | Model name override |
| `LLM_PROVIDER` | "" | Provider type (openai schema) |
| `LLM_MAX_CALLS_PER_TARGET` | 300 | Safety cap per target run |
| `OPENAI_API_KEY` | — | OpenAI key |
| `ANTHROPIC_API_KEY` | — | Anthropic key |
| `GOOGLE_API_KEY` | — | Google AI key |
| `DEFAULT_LLM_PROVIDER` | "openai" | Which provider by default |
| `DEFAULT_MODEL` | "gpt-4o-mini" | Which model by default |
| `LLM_FAILOVER_ORDER` | openai,anthropic,google | Failover chain |
| `LLM_TIMEOUT_SECONDS` | 30 | Request timeout |
| **Paths** | | |
| `BASE_DIR` | auto-detected | Backend root |
| `DATA_DIR` | `{BASE_DIR}/data` | Data files |
| `DOSSIERS_DIR` | `{DATA_DIR}/dossiers` | Dossier output |
| `TOOLS_DIR` | `{BASE_DIR}/tools` | Tools directory |
| `GLOBAL_TOOLS_DIR` | `{TOOLS_DIR}/shared` | Shared tools |
| `AGENT_TOOLS_BASE` | `{TOOLS_DIR}/agent_specific` | Per-agent tools |
| **Databases** | | |
| `SIGINT_DB` | `{DATA_DIR}/sigint_db.txt` | SIGINT records |
| `TRAVEL_DB` | `{DATA_DIR}/travel_db.txt` | Travel records |
| `OSINT_DB` | `{DATA_DIR}/osint_db.txt` | OSINT records |
| `INTEL_REPORTS_DB` | `{DATA_DIR}/intel_reports_db.txt` | Intel reports |
| `FISA_DB` | `{DATA_DIR}/fisa_db.txt` | FISA records |
| `CNE_DB` | `{DATA_DIR}/cne_db.txt` | CNE records |
| `GEOINT_DB` | `{DATA_DIR}/geoint_db.txt` | GEOINT records |
| `CORRELATION_DB` | `{DATA_DIR}/correlation_db.txt` | Correlation data |
| `ACCESS_DB` | `{DATA_DIR}/access_db.txt` | Access data |
| `ACCESSIBILITY_DB` | `{DATA_DIR}/accessibility_db.txt` | Accessibility data |
| `MOTIVATION_DB` | `{DATA_DIR}/motivation_db.txt` | Motivation data |
| `SUITABILITY_DB` | `{DATA_DIR}/suitability_db.txt` | Suitability data |
| `SECURITY_DB` | `{DATA_DIR}/security_db.txt` | Security data |
| `COUNTER_INTEL_DB` | `{DATA_DIR}/counter_intel_db.txt` | Counter-intel data |
| `PATTERN_OF_LIFE_DB` | `{DATA_DIR}/pattern_of_life_db.txt` | Pattern of life data |
| `VOTER_CSV_PATH` | `{DATA_DIR}/voter.csv` | Voter CSV |
| `TARGET_STORE_DB_PATH` | `{DATA_DIR}/target_store.db` | Target SQLite |
| `RESULTS_DB_PATH` | `{DATA_DIR}/results_store.db` | Results SQLite |
| **Tool Calling** | | |
| `TOOL_CALLING_MAX_ITERATIONS` | 10 | Max tool-call loops in chat |
| `TOOL_CALLING_MAX_MISTAKES` | 3 | Max errors before abort |
| **Sub-Agent Processing** | | |
| `SUB_AGENT_RECORDS_PER_WORKER` | 2 | Records per sub-agent chunk |
| `SUB_AGENT_MAX_WORKERS` | 50 | Max parallel workers |
| **Living Dossier** | | |
| `DOSSIER_CHUNK_SIZE` | 3000 | Assessment chunk size (chars) |
| `DOSSIER_SUMMARY_THRESHOLD` | 6000 | When to use chunked summary |
| `DOSSIER_MAX_SUB_AGENT_WORKERS` | 4 | Parallel assessment workers |
| `LARGE_DATA_CHUNK_SIZE` | 3000 | Legacy chunk size |
| `MAX_SUB_WORKERS` | 10 | Legacy max workers |
| **Enrichment** | | |
| `ENRICHMENT_MAX_ITERATIONS` | 3 | Max enrichment loops |
| `ENRICHMENT_MAX_NEW_PER_ITERATION` | 10000 | Max new IDs per iteration |
| `ENRICHMENT_ENABLED` | true | Enable/disable enrichment |
| **Translation** | | |
| `TRANSLATION_API_URL` | "" | External translation API |
| `TRANSLATION_API_KEY` | "" | Translation API key |
| **Server** | | |
| `API_HOST` | "0.0.0.0" | Bind address |
| `API_PORT` | 8000 | Bind port |
| `MAX_TOKENS` | 2000 | Default LLM max tokens |
| `TEMPERATURE` | 0.3 | Default LLM temperature |

---

## 13. DEPENDENCY MAP (If I Change X, What Breaks?)

### Critical Files — Change These Carefully

| File Changed | Files That Break |
|--------------|-----------------|
| `config.py` | **ALL files** (everyone imports Config) |
| `llm_provider.py` | plan_executor, tool_calling_engine, sub_agent_processor, living_dossier, trace_report_synthesizer, base_agent, orchestrator, chat_manager |
| `base_agent.py` | ALL 14 agent files (they extend BaseAgent) |
| `orchestrator.py` | api_server, leo_manager, targeting_manager |
| `target_store.py` | orchestrator, api_server, dossier_pipeline, living_dossier, enrichment_tracker, agent_query_tool, cross_target_linker, entity_resolver, leo_manager, targeting_manager, chat_manager |
| `living_dossier.py` | dossier_pipeline (only caller) |
| `dossier_pipeline.py` | orchestrator, leo_manager, targeting_manager |
| `plan_executor.py` | base_agent (all agents use it) |
| `tool_calling_engine.py` | base_agent, chat_manager |
| `tool_definitions.py` | tool_calling_engine |
| `base_agent_search_tool.py` | ALL 7 data agent search tools |
| `event_bus.py` | api_server, dossier_pipeline |

### Safe to Change (Leaf Nodes — Nothing Imports Them)

- `main.py` — entry point only
- `index.html` — frontend, standalone
- `test_e2e.py`, `test_reallife.py` — tests
- Individual agent files (e.g., `sigint_agent.py`) — only orchestrator imports them
- Plan JSON files — only plan_executor reads them at runtime
- Schema JSON files — only search tools reference them
- `name_variation_generator.py` — leaf utility
- `voter_db_tool.py` — leaf utility
- `datetime_tool.py` — leaf utility
- `translation_tool.py` — leaf utility
- SNA tools (`build_network.py`, `analyze_network.py`) — only sna_agent imports

### Dependency Chain

```
api_server → orchestrator → base_agent → plan_executor → sub_agent_processor → llm_provider
                          → dossier_pipeline → living_dossier → llm_provider
                                            → trace_report_synthesizer → llm_provider
                          → target_store

api_server → leo_manager → dossier_pipeline (same chain)
api_server → targeting_manager → dossier_pipeline (same chain)
api_server → chat_manager → tool_calling_engine → llm_provider
```

---

## APPENDIX: AIR-GAP LOCAL LLM SETUP

For air-gapped deployment without external API access:

```bash
# In .env:
LLM_PROVIDER=local
LLM_BASE_URL=http://localhost:11434/v1   # Ollama example
LLM_MODEL=llama3.1:70b
LLM_API_KEY=not-needed
```

The platform uses OpenAI-compatible API format (`/v1/chat/completions`). Any server exposing this works.

Recommended: 70B+ parameters for assessment quality. Smaller models (7B-13B) work for MAP/chunking tasks.

---

## APPENDIX: HOW TO ADD A NEW DATA AGENT (Full Walkthrough)

1. **Create agent file** — `agents/new_data_agent.py`:
   ```python
   from agents.base_agent import BaseAgent
   class NewDataAgent(BaseAgent):
       agent_key = "new_data"
       agent_name = "New Data Agent"
   ```

2. **Create database** — `data/new_data_db.txt` with records

3. **Create schema** — `schemas/new_data_db.json` with field definitions

4. **Create search tool** — `tools/agent_specific/new_data/new_data_search_tool.py`:
   ```python
   from tools.agent_specific.base_agent_search_tool import BaseAgentSearchTool
   class NewDataSearchTool(BaseAgentSearchTool):
       # Point to your database file
       pass
   ```

5. **Add config** — `config.py`: `NEW_DATA_DB = os.path.join(DATA_DIR, "new_data_db.txt")`

6. **Create plan** — `plans/new_data.json` with execution steps

7. **Register in orchestrator** — `orchestrator.py` → `_register_agents()` method

8. **Add to pipeline** — `dossier_pipeline.py` → `DATA_AGENTS` list

9. **Add section order** — `living_dossier.py` → `SECTION_ORDER` list (add tuple)

10. **Add formatter** — `living_dossier.py` → create `_format_new_data()` method (or it uses `_format_generic`)

---

## APPENDIX: HOW TO ADD A NEW ASSESSMENT TYPE (Full Walkthrough)

1. **Add to ASSESSMENT_TYPES** — `living_dossier.py`:
   ```python
   "my_assessment": {
       "title": "MY ASSESSMENT",
       "prompt": "Evaluate the target for... Look for:\n1. ...\n2. ...\n"
                 "If this chunk contains relevant evidence, extract with record IDs.\n"
                 "If not, respond: NO_INDICATORS",
       "keywords": ["keyword1", "keyword2"],
   },
   ```

2. **Create agent** — `agents/my_assessment_agent.py`

3. **Create plan** — `plans/my_assessment.json` (copy from `access.json` as template)

4. **Register in orchestrator** — `orchestrator.py`

5. **Add to pipeline** — `dossier_pipeline.py` → `ASSESSMENT_AGENTS` list

6. **Add to narrative** — `living_dossier.py` → add tuple to `NARRATIVE_SECTIONS` (targeting mode)

---

## 14. WORKFLOW MODIFICATION — STEP-BY-STEP

### Understanding the 3 Workflows

All 3 workflows use the SAME 7-step pipeline (`dossier_pipeline.py`). The ONLY difference:

| Workflow | Who Calls Pipeline | `workflow_type` param | Step 6 (Assess) | Step 7 (Final) |
|----------|-------------------|----------------------|-----------------|----------------|
| **Targeting** | `targeting_manager.py` | `"targeting"` | RUNS 7 assessments | Narrative from assessment files |
| **Trace** | `targeting_manager.py` | `"trace"` | SKIPPED | `trace_report_synthesizer.py` |
| **LEO** | `leo_manager.py` | `"leo"` | SKIPPED | Narrative from data source files |

---

### I WANT TO: Change which agents run in a workflow

**File:** `tools/shared/dossier_pipeline.py` (line ~31)

```python
DATA_AGENTS = [
    "travel", "osint", "intel_reports", "sigint", "fisa", "cne", "geoint",
]
```

**Steps:**
1. Open `tools/shared/dossier_pipeline.py`
2. Find `DATA_AGENTS` list at top of class (line ~31)
3. Add or remove agent keys from this list
4. These agents run in Step 3 (COLLECT) for ALL workflows

**To change agents for ONE specific workflow only:**
1. The `run()` method accepts `agent_keys` parameter (optional subset)
2. This is passed from the calling manager
3. For targeting: `targeting_manager.py` → find where `pipeline.run()` is called
4. For LEO: `leo_manager.py` → find where `pipeline.run()` is called
5. Pass `agent_keys=["sigint", "travel"]` to limit which agents run

---

### I WANT TO: Change which assessments run

**File:** `tools/shared/dossier_pipeline.py` (line ~34)

```python
ASSESSMENT_AGENTS = [
    "access", "accessibility", "motivation", "suitability",
    "security", "counter_intel", "pattern_of_life",
]
```

**Steps:**
1. Open `tools/shared/dossier_pipeline.py`
2. Find `ASSESSMENT_AGENTS` list (line ~34)
3. Add or remove assessment type keys
4. These only run in Step 6 (targeting workflow only)
5. Must also have matching entry in `ASSESSMENT_TYPES` dict in `living_dossier.py`

---

### I WANT TO: Change Step 2 (EXPAND) — how identifiers are expanded

**File:** `tools/shared/dossier_pipeline.py`
**Method:** `_step_2_expand()` (line ~211)

**What this step does:**
- Takes user-provided identifiers (name, phone, passport)
- Generates 18+ Arabic/English name transliterations via `name_variation_generator`
- Runs phone correlation to find linked IMEIs/IMSIs
- Stores expanded identifiers for all subsequent searches

**Steps to modify:**
1. Open `tools/shared/dossier_pipeline.py`
2. Find `def _step_2_expand(self, identifiers)`
3. To change name expansion: modify the call to `self.orch._auto_expand_query()`
4. To change which identifier fields trigger expansion: modify the `meaningful` check (line ~220)
5. To add a new expansion type (e.g., email domain expansion): add logic after the phone correlation

**Related files:**
- `tools/shared/name_variation_generator.py` — actual Arabic/English dictionaries
- `tools/agent_specific/sigint/correlation_tool.py` — phone→IMEI→IMSI chains
- `agents/orchestrator.py` → `_auto_expand_query()` method

---

### I WANT TO: Change Step 3 (COLLECT) — how agents are run

**File:** `tools/shared/dossier_pipeline.py`
**Method:** `_step_3_collect()` (line ~278)

**What this step does:**
- Runs 7 data agents in parallel (ThreadPoolExecutor, max 5 workers)
- Each agent: `agent.search(identifiers)` via plan_executor
- Results written to dossier sections via `dossier.write_agent_section()`
- Sections summarized via `dossier.summarize_section()`

**Steps to modify:**
1. Open `tools/shared/dossier_pipeline.py`
2. Find `def _step_3_collect()`
3. To change concurrency: modify `max_workers = min(5, len(data_agents))` (line ~327)
4. To change what happens after agent runs: modify the post-processing after `_run_agent()`
5. To add a hook after each agent completes: add code after `dossier.write_agent_section()`

**To change how an individual agent searches:**
- Edit that agent's plan JSON file (see Section 8 above)
- OR override `search()` in the agent class file

---

### I WANT TO: Change Step 4 (ENRICH) — iterative discovery

**File:** `tools/shared/dossier_pipeline.py`
**Method:** `_step_4_enrich()` (line ~533)

**What this step does:**
- Scans existing results for new phone numbers (via `contact_search`)
- For each new identifier found, searches relevant agents again
- Appends new results to existing sections
- Loops until convergence or max iterations (default 3)

**Steps to modify:**
1. Open `tools/shared/dossier_pipeline.py`
2. Find `def _step_4_enrich()`
3. To change max iterations: modify `config.py` → `ENRICHMENT_MAX_ITERATIONS` (default 3)
4. To disable enrichment: set `ENRICHMENT_ENABLED=false` in .env
5. To change what's considered a "new identifier": modify `contact_search.py` → `execute()`
6. To change which agents get re-searched: modify the agent selection logic in this method

**Related files:**
- `tools/shared/enrichment_tracker.py` — tracks discovered identifiers, prevents infinite loops
- `tools/shared/contact_search.py` — finds new phone numbers across all databases
- `config.py` → `ENRICHMENT_MAX_ITERATIONS`, `ENRICHMENT_MAX_NEW_PER_ITERATION`, `ENRICHMENT_ENABLED`

---

### I WANT TO: Change Step 5 (COMPILE) — cross-references and contact network

**File:** `tools/shared/dossier_pipeline.py`
**Method:** `_step_5_compile()` (line ~718)

**What this step does (zero LLM, all Python):**
1. `compile_cross_references()` — regex scan for shared identifiers
2. `compile_contact_network()` — phone extraction + name resolution
3. Voter registration check
4. Entity resolution + cross-target links
5. Compile full dossier (concatenate all sections into versioned file)

**Steps to modify:**
1. To change cross-references: `living_dossier.py` → `compile_cross_references()` (see Section 10)
2. To change contact network: `living_dossier.py` → `compile_contact_network()` (see Section 10)
3. To add a new compilation sub-step: add code after step 5d in `_step_5_compile()`
4. To change voter lookup: modify `_check_voter_registration()` in same file
5. To change cross-target linking: modify `tools/shared/cross_target_linker.py`

---

### I WANT TO: Change Step 6 (ASSESS) — assessment scanning

**File:** `tools/shared/dossier_pipeline.py`
**Method:** `_step_6_assess()` (line ~801)

**What this step does:**
- Reads the compiled dossier from disk
- For each of 7 assessment types, calls `dossier.run_assessment(type)`
- Each assessment: chunks dossier into 3K segments → LLM scans each chunk
- Results saved to `assess_{type}.md`

**Steps to modify:**
1. To change assessment prompts: modify `ASSESSMENT_TYPES` in `living_dossier.py` (Section 9)
2. To change chunk size: `config.py` → `DOSSIER_CHUNK_SIZE` (default 3000)
3. To change which chunks are skipped: `living_dossier.py` → `_is_pure_index_chunk()`
4. To change keyword pre-filtering: modify `keywords` list in `ASSESSMENT_TYPES`
5. To run assessments in parallel vs serial: modify executor in `_step_6_assess()`
6. To skip specific assessments: remove from `ASSESSMENT_AGENTS` list in `dossier_pipeline.py`

**This step only runs for `workflow_type == "targeting"`.** Trace and LEO skip it.

---

### I WANT TO: Change Step 7 (FINAL) — report generation

**File:** `tools/shared/dossier_pipeline.py`
**Methods:**
- `_step_7_final()` (line ~923) — for targeting and LEO
- `_step_7_trace_report()` (line ~848) — for trace

**For targeting/LEO final:**
1. Calls `dossier.generate_executive_summary()` → LLM generates 4000-token summary
2. Calls `dossier.generate_narrative_report()` → LLM generates 11-section narrative
3. Stores report in target_store

**For trace final:**
1. Calls `trace_report_synthesizer.synthesize_trace_report()` → generates trace-specific report
2. Also generates executive summary and narrative

**Steps to modify:**
1. To change executive summary: see Section 9 → "Executive Summary"
2. To change narrative sections: see Section 9 → "Narrative Report"
3. To change trace report: see Section 9 → "Trace Report Narrative Topics"
4. To add a new step after the report: add code after `_step_7_final()` returns
5. To change what gets stored in SQLite: modify the `store.store_report()` call in the method

---

### I WANT TO: Make a workflow skip or add a step

**File:** `tools/shared/dossier_pipeline.py`
**Method:** `run()` (line ~53)

The `run()` method is the master controller. Steps are called sequentially:

```python
# In run() method:
dossier_dir = self.dossier.create_dossier(...)        # Step 1
identifiers = self._step_2_expand(identifiers)        # Step 2
raw_results, ... = self._step_3_collect(...)          # Step 3
enrichment_stats = self._step_4_enrich(...)           # Step 4
dossier_path = self._step_5_compile(...)              # Step 5
if workflow_type not in ("trace", "leo"):             # Step 6 (conditional)
    assessments = self._step_6_assess(target_id)
if workflow_type == "trace":                          # Step 7 (branched)
    ... = self._step_7_trace_report(...)
else:
    ... = self._step_7_final(...)
```

**To skip a step for a specific workflow:**
1. Add a condition: `if workflow_type != "my_type": self._step_X()`
2. Example: to skip enrichment for trace: wrap Step 4 in `if workflow_type != "trace":`

**To add a new step:**
1. Create a new method: `def _step_X_my_step(self, ...)`
2. Call it from `run()` at the desired position
3. Add SSE event emission: `self._emit("my_step_started", message="...")`

---

### I WANT TO: Change the API endpoint for a workflow

**File:** `api_server.py`

**Targeting workflow endpoints:** Search for `@app.post("/targeting/`
**LEO workflow endpoints:** Search for `@app.post("/leo/`
**Trace workflow endpoints:** Search for `@app.post("/trace/`

**Steps:**
1. Open `api_server.py`
2. Find the endpoint (e.g., `@app.post("/targeting/run/{wf_id}")`)
3. The endpoint handler calls the manager (e.g., `targeting_manager.run_workflow()`)
4. The manager calls `dossier_pipeline.run(workflow_type="targeting")`
5. To change request format: modify the endpoint function parameters
6. To change response format: modify the return dict

**Chain:**
```
Frontend → api_server.py (endpoint) → {leo,targeting}_manager.py → dossier_pipeline.run() → 7 steps
```

---

## 15. EDITING AN EXISTING TOOL — COMPLETE WALKTHROUGH

### What Is a "Tool"?

A tool is a Python class with an `execute(params: dict) -> dict` method. Tools are called by:
1. **Plan Executor** — deterministic JSON steps call tools by name
2. **Tool Calling Engine** — LLM autonomously decides which tools to call
3. **Pipeline** — some tools called directly by pipeline steps

### Step-by-Step: Edit an Agent Search Tool

**Example: I want to change how the SIGINT search works**

1. **Open the tool file:**
   ```
   tools/agent_specific/sigint/sigint_search_tool.py
   ```

2. **Find the `execute()` method** — this is the entry point. It receives `params` dict and returns results dict.

3. **Understand the inheritance:**
   - `SIGINTSearchTool` extends `BaseAgentSearchTool` (in `tools/agent_specific/base_agent_search_tool.py`)
   - `BaseAgentSearchTool` provides: text file loading, regex search, scoring, ranking
   - Override `execute()` to change behavior completely, OR modify parent class to change ALL search tools

4. **Modify the search logic:**
   - To change scoring: modify the scoring algorithm in `base_agent_search_tool.py`
   - To change what gets returned: modify the return dict structure
   - To add filtering: add filter logic after search results are gathered

5. **Test the change:**
   ```bash
   python -c "
   from tools.agent_specific.sigint.sigint_search_tool import SIGINTSearchTool
   tool = SIGINTSearchTool('data/sigint_db.txt')
   result = tool.execute({'query': '+964-770-555-1234'})
   print(result.keys(), len(str(result)))
   "
   ```

6. **Check downstream effects:**
   - The plan JSON (`plans/sigint.json`) references this tool by name
   - If you changed the return dict keys, update the plan's next step that uses `{{output_key}}`
   - The `_format_sigint()` method in `living_dossier.py` expects a certain input format

### Step-by-Step: Edit a Shared Tool

**Example: I want to change how name_variation_generator works**

1. **Open:** `tools/shared/name_variation_generator.py`

2. **Find the class** (usually one class per file, extending `BaseTool`)

3. **Find `execute()`** — entry point with params dict

4. **Modify:**
   - To add new Arabic transliterations: find the dictionaries/mapping tables
   - To change max variations: modify the return list length
   - To change the algorithm: modify the generation logic

5. **Nothing else breaks** — this is a leaf tool (nothing imports it except plan_executor and tool_calling_engine which call it generically by name)

6. **Verify compile:**
   ```bash
   python -c "import py_compile; py_compile.compile('tools/shared/name_variation_generator.py')"
   ```

### Step-by-Step: Edit the Tool Calling Engine

**WARNING: This affects ALL agents in chat mode and ALL agents without JSON plans.**

1. **Open:** `tools/shared/tool_calling_engine.py`

2. **Key methods:**
   - `run()` — main loop (up to 10 iterations)
   - The system prompt that tells LLM which tools exist
   - Tool execution dispatch

3. **To change max iterations:** `config.py` → `TOOL_CALLING_MAX_ITERATIONS`

4. **To change the system prompt for tools:**
   - Find where tool definitions are formatted into the prompt
   - Modify the instruction text

5. **To change how tool results are fed back:** modify the message construction after tool execution

### Step-by-Step: Edit the Plan Executor

**WARNING: This affects ALL 14 agents that use JSON plans.**

1. **Open:** `tools/shared/plan_executor.py`

2. **Key methods:**
   - `execute_plan()` — reads JSON, iterates steps
   - `_execute_step()` — dispatches each step by action type
   - `_substitute_templates()` — replaces `{{variables}}` in params/prompts
   - `_chunked_llm_call()` — sub-agent MAP scanning for large prompts

3. **To change template substitution:** modify `_substitute_templates()`

4. **To change the chunking threshold:** find `LARGE_PROMPT_THRESHOLD = 6000` (change this number)

5. **To change how MAP sub-agents work:** modify `_chunked_llm_call()` — the sub-agent receives the assessment prompt + a chunk, and must return SMALLER output than input

6. **To add a new action type:** add a new `elif action == "my_action":` block in `_execute_step()`

---

## 16. ADDING A NEW TOOL — COMPLETE WALKTHROUGH

> ⚠️ **Read Section 18 first** if you are *replacing* an existing demo tool with real-API code (Jupyter → production). Section 18 has the exact contract, debugging checklist, and the failure modes you'll hit. Section 16 is the basics.

### The Two Tool Types

| Type | Base class | Used by | Lives in |
|------|-----------|---------|----------|
| **Shared tool** | `BaseTool` | All agents | `tools/shared/` |
| **Agent-specific search tool** | `BaseAgentSearchTool` | One data agent (sigint, travel, fisa, cne, osint, geoint, intel_reports) | `tools/agent_specific/<agent>/` |

Both inherit from `BaseTool` ultimately. The difference: agent search tools auto-receive `query` + `search_terms` (pre-expanded by orchestrator); shared tools take whatever kwargs you define.

### The Contract (REQUIRED — non-negotiable)

Every tool **MUST**:

1. Subclass `BaseTool` (or `BaseAgentSearchTool`)
2. Implement `def execute(self, **kwargs) -> ToolResult:` — note `**kwargs`, NOT a `params` dict
3. Implement `def get_parameter_schema(self) -> Dict[str, Any]:` — abstract method, instantiation crashes without it (see [base_tool.py:28-30](backend/tools/base_tool.py#L28-L30))
4. Return a `ToolResult` dataclass — NOT a plain dict. Fields: `success: bool`, `data: Any`, `error: str`, `tool_name: str`, `execution_time_ms: float`
5. Call `super().__init__(name=, description=, category=)` with the named args

```python
from tools.base_tool import BaseTool, ToolResult


class WebScraperTool(BaseTool):
    """Scrape a URL and return text. Shared tool."""

    def __init__(self):
        super().__init__(
            name="web_scraper",
            description="Scrape a web page and return text content.",
            category="api_integrations",
        )

    def get_parameter_schema(self) -> dict:
        return {
            "required": ["url"],
            "properties": {
                "url": {"type": "string", "description": "Full URL"},
                "max_chars": {"type": "integer", "description": "Max chars"},
            },
        }

    def execute(self, url: str = "", max_chars: int = 5000, **kwargs) -> ToolResult:
        try:
            import requests
            resp = requests.get(url, timeout=10)
            text = resp.text[:max_chars]
            return ToolResult(success=True, data={
                "content": text,
                "chars": len(text),
            })
        except Exception as e:
            return ToolResult(success=False, error=str(e))
```

### Adding a Shared Tool — Step by Step

#### Step 1: Create the tool file

Create `tools/shared/web_scraper.py` using the contract above.

#### Step 2: Register in `orchestrator.py`

Open [`agents/orchestrator.py`](backend/agents/orchestrator.py), find `_init_tool_calling_engines()` (line ~130). Add a `ToolDefinition`:

```python
# --- Tool N: web_scraper ---
from tools.shared.web_scraper import WebScraperTool
self._web_scraper = WebScraperTool()  # instance kept on self

web_scraper_def = ToolDefinition(
    name="web_scraper",
    description="Scrape a web page and return its text content.",
    parameters=[
        {"name": "url", "required": True,
         "description": "Full URL (https://...)"},
        {"name": "max_chars", "required": False,
         "description": "Max characters to return (default 5000)"},
    ],
    handler=self._handle_web_scraper,
)
```

#### Step 3: Add the handler method

Add this method to the `Orchestrator` class (anywhere in the `_handle_*` block):

```python
def _handle_web_scraper(self, url: str = "", max_chars: int = 5000) -> Dict:
    """Handler for web_scraper tool calls. Returns {"result": ...} or {"error": ...}."""
    self._tool_log("web_scraper", f"Scraping {url[:60]}")
    result = self._web_scraper.execute(url=url, max_chars=max_chars)
    if result.success:
        return {"result": result.data.get("content", "")}
    return {"error": result.error}
```

> **Why this shape?** The tool-calling engine expects handlers to return `{"result": ...}` for success or `{"error": ...}` for failure. This is the engine contract, NOT the tool contract. The tool returns `ToolResult`; the handler unwraps it for the engine.

#### Step 4: Register with each agent's engine

Find the loop in `_init_tool_calling_engines()` (around line 341, `for agent_key in engine_agents:`). Add `engine.register_tool(web_scraper_def)` next to the other `register_tool(...)` calls (around line 386).

#### Step 5: (Plan-driven) Also register handler with PlanExecutor

If any agent's `plans/<agent>.json` will call this tool, also add it to `_init_plan_executors()` `tool_handlers` dict (line ~452):

```python
tool_handlers = {
    ...
    "web_scraper": self._handle_web_scraper,  # <-- ADD HERE
}
```

> ⚠️ **This is the #1 reason new tools "fault."** The engine and plan executor maintain *separate* handler tables. Register in one, miss the other → chat works but plan fails (or vice versa). See Section 18.5.

#### Step 6: Verify

```bash
cd V2/backend
python -c "import py_compile; py_compile.compile('tools/shared/web_scraper.py')"
python -c "import py_compile; py_compile.compile('agents/orchestrator.py')"
python -c "from tools.shared.web_scraper import WebScraperTool; t=WebScraperTool(); print(t.get_parameter_schema())"
```

Then start the server and check the boot log for `[ORCHESTRATOR] Plan executors initialized for N agents` and confirm N matches expectation.

---

### Adding an Agent-Specific Search Tool

**Example: a new "financial" data agent's search tool.**

#### Step 1: Create the file

`tools/agent_specific/financial/__init__.py` (empty) and `tools/agent_specific/financial/financial_search_tool.py`:

```python
from tools.agent_specific.base_agent_search_tool import BaseAgentSearchTool


class FinancialSearchTool(BaseAgentSearchTool):
    """Search financial records (text-file demo; swap to SQL/API in production)."""

    def __init__(self, db_path: str):
        super().__init__(
            name="search_financial_records",
            description=(
                "Search financial transaction records. Search by account number, "
                "transaction ID, name, date, or amount."
            ),
            db_path=db_path,
        )
```

That's the whole demo tool — `BaseAgentSearchTool` provides `execute()` that does text-file search out of the box (records separated by `\n\n`). To go to a real API, override `execute()` — see Section 18.

#### Step 2: Create the data file

`data/financial_db.txt` with records separated by **two newlines** (`\n\n`).

#### Step 3: Register the path in `config.py`

```python
FINANCIAL_DB = os.path.join(DATA_DIR, "financial_db.txt")
```

#### Step 4: Register in `orchestrator.py` — TWO places

In `_init_tool_calling_engines()`, line ~148:

```python
from tools.agent_specific.financial.financial_search_tool import FinancialSearchTool

self._agent_search_tools = {
    "travel": TravelSearchTool(Config.TRAVEL_DB),
    "sigint": SIGINTSearchTool(Config.SIGINT_DB),
    ...
    "financial": FinancialSearchTool(Config.FINANCIAL_DB),  # <-- ADD HERE
}
```

If you want this agent to participate in workflows, also add `"financial"` to `engine_agents` (line 334) AND `dossier_pipeline.py` `DATA_AGENTS` (line 31). See Section 18.4 for the full list of places.

#### Step 5: Done — agent's `tool_calling_engine` automatically gets the search tool

The loop at line 352-364 reads `self._agent_search_tools.get(agent_key)` and registers it. No further code in orchestrator.

#### Step 6: Add a plan JSON (optional but recommended)

Create `plans/financial.json` based on an existing one. Reference the search tool by its `name` field (`"search_financial_records"`).

---

### Adding a Tool to an Existing Agent's Plan

**Example: travel agent should cross-check financial records.**

Open `plans/travel.json`, add a step before the compile step:

```json
{
    "id": "xref_financial",
    "action": "tool_call",
    "tool": "query_agent",
    "params": {
        "agent_key": "financial",
        "query": "{{identifiers.first_name}} {{identifiers.family_name}}",
        "mode": "database"
    },
    "store_as": "financial_xref",
    "skip_if": "not identifiers.first_name",
    "description": "Cross-reference target in financial records"
}
```

Reference `{{financial_xref}}` in the final `compile_report` step's prompt. Plans are reloaded per search call — no restart needed.

---

### Removing a Tool — Complete Cleanup

To fully remove a tool (e.g., `web_scraper`), update **all** of these — missing any one leaves a phantom:

1. **`agents/orchestrator.py`** — remove the import, the `_web_scraper` instance, the `ToolDefinition`, the `_handle_web_scraper` method, and the `engine.register_tool(...)` call (line ~386)
2. **`agents/orchestrator.py`** — also remove from `_init_plan_executors()` `tool_handlers` dict (line ~452)
3. **`tools/shared/tool_definitions.py`** — remove the entry if listed
4. **All `plans/*.json`** — grep for the tool name (`grep -l web_scraper plans/`); remove or replace any step that calls it
5. **Delete the file** itself
6. **Restart the server** — Python module cache will hold the old class otherwise

Verify:
```bash
grep -rn "web_scraper" backend/  # Should return nothing except git-ignored bytecode
python -c "import py_compile; py_compile.compile('agents/orchestrator.py')"
```

---

## 18. PLUG-AND-PLAY: REPLACING DEMO TOOLS WITH REAL APIs

This section is the production-migration guide. Read it end-to-end before swapping any demo tool. The goal: take working Jupyter API code and drop it into the platform with **zero downstream breakage**.

### 18.1 The Promise

The platform is designed so each data agent's search tool is **the only thing that touches the data source.** Everything downstream — formatters, the dossier compiler, contact search, cross-target linker, enrichment tracker, assessment scanners — consumes a **stable text contract.** Keep that contract, and you can swap a text-file tool for a SQL/REST/gRPC tool without changing any other file.

The Living Dossier markdown architecture (one consolidated `dossier_v{N}.md` per target, built from per-agent sections) **stays exactly as is.** It's the thing that makes summarization-from-one-place work; replacing tools doesn't touch it.

### 18.2 The Output Contract Your Real-API Tool MUST Honor

Two layers of contract:

**Layer A — Python return type (rigid, code will crash if violated):**

```python
def execute(self, query: str = "", search_terms: str = "", **kwargs) -> ToolResult:
    return ToolResult(
        success=True,
        data={
            "result": "<TEXT BLOB — this is what downstream consumers parse>",
            "record_count": 42,
            "total_records": 1500,
        },
    )
```

- `success` False → `_handle_agent_search()` propagates as `{"error": ...}`, agent reports failure
- `data["result"]` is a **string** — the formatted text records (see Layer B)
- `data["record_count"]` and `data["total_records"]` are used for logging; non-fatal if missing
- Returning a plain dict instead of `ToolResult` → `AttributeError: 'dict' object has no attribute 'success'` at [orchestrator.py:732](backend/agents/orchestrator.py#L732)

**Layer B — The text format inside `data["result"]` (per-agent, parsed by `living_dossier.py` formatters):**

| Agent | Record separator | Required pattern | Where it's parsed |
|-------|------------------|------------------|-------------------|
| **sigint** | `\n\n` | Each block must START with `CDR_<digits>\n` then lines `FROM:`, `TO:`, `TYPE:`, `DATE:`, `TIME:`, `DURATION:`, `TOWER_FROM:` (any subset) | `_format_sigint()` line ~1490 |
| **travel** | `\n\n` | Free-form text per record | `_format_travel()` line ~1588 |
| **fisa** | `\n\n` | Free-form text per record | `_format_fisa()` line ~1600 |
| **cne** | `\n\n` | Free-form; include literal "DELETED" or "RECOVERED" if applicable (auto-tagged) | `_format_cne()` line ~1612 |
| **intel_reports** | `\n\n` | Free-form narrative per report | `_format_intel_reports()` line ~1627 |
| **osint** | `\n\n` | Free-form per record | `_format_osint()` line ~1639 |
| **geoint** | `\n\n` | Free-form per record | `_format_geoint()` line ~1651 |

**Critical rule for every agent:** records separated by exactly **two newlines** (`\n\n`). One newline → entire payload treated as one record → counts wrong → contact extraction broken.

**SIGINT-specific** is the strictest. Example of a single CDR record block:

```
CDR_00045
FROM: +9647701234567
TO: +9647809876543
TYPE: VOICE
DATE: 2025-09-12
TIME: 14:33:21
DURATION: 187
TOWER_FROM: BAGHDAD-N-12
```

If your real CDR API returns JSON, transform it to this text shape inside `execute()` before returning. Don't return JSON — the formatter expects this exact prefix-syntax to extract `from`/`to`/`tower` reliably; deviating means contact names show as "UNKNOWN" everywhere.

### 18.3 Step-by-Step: Replace SIGINT Demo Tool with a Real CDR API

This is the concrete walkthrough using a Jupyter-tested CDR REST API as the example.

#### Step 1 — Identify the file you replace

Only this one file: `tools/agent_specific/sigint/sigint_search_tool.py`. Nothing else in the codebase needs to change *if* your output text matches Layer B.

#### Step 2 — Lift your Jupyter code into a private method

```python
# tools/agent_specific/sigint/sigint_search_tool.py

import os
import requests
from typing import Dict, Any
from tools.agent_specific.base_agent_search_tool import BaseAgentSearchTool
from tools.base_tool import ToolResult


class SIGINTSearchTool(BaseAgentSearchTool):
    """SIGINT search tool — REAL CDR API (replaces text-file demo)."""

    def __init__(self, db_path: str = ""):
        # db_path kept for backward-compat with orchestrator constructor
        # but unused — we hit the real API instead
        super().__init__(
            name="search_sigint_records",
            description=(
                "Search live CDR/SMS/voice intercept database. "
                "Search by phone, IMEI, IMSI, name, date range, or tower."
            ),
            db_path=db_path,
        )
        self.api_url = os.environ.get("CDR_API_URL", "https://cdr.internal/v1/search")
        self.api_key = os.environ.get("CDR_API_KEY", "")

    def _call_cdr_api(self, terms: list[str]) -> list[dict]:
        """The Jupyter-proven API call. Returns list of CDR dicts."""
        resp = requests.post(
            self.api_url,
            json={"terms": terms, "limit": 5000},
            headers={"Authorization": f"Bearer {self.api_key}"},
            timeout=60,
        )
        resp.raise_for_status()
        return resp.json().get("records", [])

    @staticmethod
    def _record_to_text(rec: dict, idx: int) -> str:
        """Format ONE API record into the SIGINT text contract (Layer B)."""
        return (
            f"CDR_{idx:05d}\n"
            f"FROM: {rec.get('caller', '')}\n"
            f"TO: {rec.get('callee', '')}\n"
            f"TYPE: {rec.get('type', 'VOICE').upper()}\n"
            f"DATE: {rec.get('date', '')}\n"
            f"TIME: {rec.get('time', '')}\n"
            f"DURATION: {rec.get('duration_seconds', 0)}\n"
            f"TOWER_FROM: {rec.get('tower_id', '')}"
        )

    def execute(self, query: str = "", search_terms: str = "", **kwargs) -> ToolResult:
        # search_terms is a comma-separated, pre-expanded list from the orchestrator
        # (auto-expand names + auto-correlate phones already happened upstream)
        terms = [t.strip() for t in search_terms.split(",") if t.strip()] \
            if search_terms else [t for t in query.split() if t]

        if not terms:
            return ToolResult(success=True, data={
                "result": "No search terms provided.",
                "record_count": 0, "total_records": 0,
            })

        try:
            records = self._call_cdr_api(terms)
        except Exception as e:
            # Network/auth/timeout failures: return success=False so the
            # tool-calling engine surfaces it (not a silent empty result)
            return ToolResult(success=False, error=f"CDR API failed: {e}")

        if not records:
            return ToolResult(success=True, data={
                "result": f"No CDR records matching '{query}' found.",
                "record_count": 0, "total_records": 0,
            })

        text_blocks = [self._record_to_text(r, i + 1) for i, r in enumerate(records)]
        return ToolResult(success=True, data={
            "result": "\n\n".join(text_blocks),
            "record_count": len(records),
            "total_records": len(records),
        })
```

#### Step 3 — Verify in isolation BEFORE touching the orchestrator

```bash
cd V2/backend
python <<'PY'
from tools.agent_specific.sigint.sigint_search_tool import SIGINTSearchTool
t = SIGINTSearchTool()
print("Schema OK:", t.get_parameter_schema())
r = t.execute(query="+9647701234567", search_terms="+9647701234567")
assert hasattr(r, "success"), "Did not return ToolResult"
assert isinstance(r.data["result"], str), "result not a string"
blocks = [b for b in r.data["result"].split("\n\n") if b.strip()]
assert all(b.startswith("CDR_") for b in blocks), "Missing CDR_ prefix"
print(f"OK — {len(blocks)} records, first line: {blocks[0].splitlines()[0]}")
PY
```

If this script passes, the rest of the system will work — that's the whole point of the contract.

#### Step 4 — Verify the formatter accepts your output

```bash
python <<'PY'
from tools.shared.living_dossier import LivingDossier
ld_text = open("test_cdr_output.txt").read()  # paste your tool's output here
formatted = LivingDossier(llm=None, dossiers_dir="/tmp")._format_sigint(
    ld_text, identifiers={"phone": "+9647701234567"}
)
print(formatted[:500])
# Should show "**Total CDR Records:** N" + "#### CONTACT: ..." groupings
# If you see "Total CDR Records: 0", your CDR_ prefix is wrong
PY
```

#### Step 5 — Restart the server

`python main.py` (or however you launch). Watch the boot log:
- `[ORCHESTRATOR] Plan executors initialized for 14 agents` — sigint is one of them
- No `ImportError` / `TypeError` for the sigint tool

#### Step 6 — Test through chat

Open the SIGINT chat tab, send `"search for +9647701234567"`. Expected: dossier section written, contact summary appears. If chat says "Error: Unknown" — see Section 18.5.

#### Step 7 — Test through LEO / Targeting workflow

Run a targeting workflow. Inspect `data/dossiers/<target_id>/sections/01_sigint.md`:
- Header: `**Total CDR Records:** N`
- Body: contact groupings `#### CONTACT: <phone> (N interactions)` with table rows

Same for any other agent — only Step 2's record formatting changes per agent (per Layer B table).

### 18.4 Everything That Depends on Your Tool's Output

When you replace a tool, these consumers all keep working *only if* you honor Layer B:

| Consumer | What it reads | File |
|----------|---------------|------|
| `_handle_agent_search()` | `result.data["result"]`, `result.data["record_count"]` | `agents/orchestrator.py:710` |
| Tool-calling engine logging | `result.success`, `result.error` | `tools/shared/tool_calling_engine.py` |
| Plan executor `tool_call` step | Same shape as engine handler | `tools/shared/plan_executor.py` |
| Dossier section formatter | Splits text by `\n\n`, applies per-agent regex | `tools/shared/living_dossier.py:1490+` |
| Dossier record counter | Counts literal `#### CONTACT:` / `#### Order ` / `#### Record ` / `#### Report ` in formatted output | `tools/shared/living_dossier.py:283` |
| SIGINT adapter (structured records) | Parses `CDR_*` blocks for `from`/`to`/`tower` fields | `tools/agent_specific/sigint/sigint_adapter.py` |
| FISA query tool | Parses FISA orders from `\n\n`-split text | `tools/agent_specific/fisa/fisa_query.py` |
| Travel query tool | Parses travel trips from `\n\n`-split text | `tools/agent_specific/travel/travel_query.py` |
| Enrichment tracker | Regex-extracts new phones/emails/IMEIs from text | `tools/shared/enrichment_tracker.py` |
| Contact search | Iterates structured records (CDR contact_list, FISA all_contacts, Travel co_traveler_list) | `tools/shared/contact_search.py` |
| Cross-target linker | Reads structured records from TargetStore | `tools/shared/cross_target_linker.py` |

**The quick test:** if your formatted output, when split by `\n\n`, gives you N blocks, then `record_count` should be N, and the dossier section will show N records. If those numbers don't match, the format is wrong.

### 18.5 Why Tool Registration Faults — Common Failure Modes

Every one of these is a real symptom from the existing demo code:

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| `TypeError: Can't instantiate abstract class WebScraperTool with abstract method get_parameter_schema` | Forgot `get_parameter_schema()` | Add it (see contract above) |
| `AttributeError: 'dict' object has no attribute 'success'` somewhere in orchestrator | `execute()` returned a dict, not `ToolResult` | Wrap in `ToolResult(success=True, data={...})` |
| `TypeError: __init__() missing 1 required positional argument: 'name'` | `super().__init__(db_path)` called positionally | Use named args: `super().__init__(name=..., description=..., db_path=...)` |
| Boot log: `[ORCHESTRATOR] Plan executors initialized for 13 agents` (expected 14) | Plan JSON parse error or tool not registered in `_init_plan_executors()` `tool_handlers` dict | Check `plans/<agent>.json` validity AND add handler to line ~452 |
| Chat works but LEO/Targeting workflow shows "0 records" for the agent | Tool registered with engine but not with PlanExecutor | Add to BOTH `_init_tool_calling_engines()` AND `_init_plan_executors()` |
| Dossier section shows the raw API JSON / unformatted text | Output didn't follow Layer B (wrong separator or missing CDR_/etc. prefix) | Reformat inside `execute()` before returning |
| Contact network shows all "UNKNOWN" names | SIGINT records missing `FROM:`/`TO:` lines, so formatter can't pair contacts | Use the exact field names |
| `record_count` is right but dossier shows 0 records | Counter looks for `#### CONTACT:`/`#### Order `/etc.; you're using a different format | Don't change the formatter's section markers — only change `execute()` |
| Tool found by chat but agentic-search ignores it | Tool registered without a handler, OR handler returns `ToolResult` instead of `{"result": ...}` | Handler must unwrap to `{"result": str}` or `{"error": str}` |
| Old code keeps running after edits | Python `__pycache__` holds bytecode | `find . -name __pycache__ -exec rm -rf {} +` then restart |
| `ModuleNotFoundError` after restart | New tool file missing `__init__.py` in its directory | Add empty `__init__.py` |

### 18.6 Debugging Checklist (Run These IN ORDER When a Tool "Faults")

```bash
cd V2/backend

# 1. File-level syntax
python -c "import py_compile; py_compile.compile('tools/agent_specific/sigint/sigint_search_tool.py')"

# 2. Class instantiation (catches abstract-method errors)
python -c "from tools.agent_specific.sigint.sigint_search_tool import SIGINTSearchTool; t=SIGINTSearchTool('data/sigint_db.txt'); print(type(t).__mro__)"

# 3. Schema callable
python -c "from tools.agent_specific.sigint.sigint_search_tool import SIGINTSearchTool; print(SIGINTSearchTool('data/sigint_db.txt').get_parameter_schema())"

# 4. Execute returns ToolResult
python -c "
from tools.agent_specific.sigint.sigint_search_tool import SIGINTSearchTool
from tools.base_tool import ToolResult
r = SIGINTSearchTool('data/sigint_db.txt').execute(query='test', search_terms='test')
assert isinstance(r, ToolResult), f'got {type(r)}'
print('OK', r.success, type(r.data))
"

# 5. Output text follows Layer B (per agent)
python -c "
from tools.agent_specific.sigint.sigint_search_tool import SIGINTSearchTool
r = SIGINTSearchTool('data/sigint_db.txt').execute(query='+9647', search_terms='+9647')
text = r.data.get('result', '')
blocks = [b for b in text.split('\\n\\n') if b.strip()]
sigint_ok = all(b.startswith('CDR_') for b in blocks) if blocks else True
print(f'{len(blocks)} blocks, sigint_format={sigint_ok}')
"

# 6. Orchestrator boots clean
python -c "from agents.orchestrator import Orchestrator; o = Orchestrator(); print(list(o._agent_search_tools.keys()))"

# 7. Plan executor picks it up
python -c "
from agents.orchestrator import Orchestrator
o = Orchestrator()
agent = o.agents['sigint']
print('plan loaded:', agent.plan is not None)
print('engine:', agent.tool_calling_engine is not None)
print('search_tool:', o._agent_search_tools.get('sigint').name)
"

# 8. Live test through API
curl -X POST http://localhost:8000/single-agent \
    -H "Content-Type: application/json" \
    -d '{"agent": "sigint", "query": {"phone": "+9647701234567"}}'
```

If any of these fail, fix them in order — later steps depend on earlier ones.

### 18.7 What You MUST NOT Touch (Stable Contracts)

Don't change these unless you really mean to — they're the contract that makes plug-and-play work:

- `BaseTool` / `BaseAgentSearchTool` interfaces (`tools/base_tool.py`, `tools/agent_specific/base_agent_search_tool.py`)
- `_handle_agent_search()` in `orchestrator.py` (line 710)
- The Layer B record markers: `CDR_<n>`, `FROM:`, `TO:`, etc. for SIGINT; `\n\n` separator for all
- The dossier section markers: `#### CONTACT:`, `#### Order `, `#### Record `, `#### Report `
- `LivingDossier.SECTION_ORDER`, `ASSESSMENT_TYPES`
- `DossierPipeline.run()` step order — Steps 1-7 are linear and dependent

### 18.8 The Living Dossier Architecture is Preserved

This is intentional and unchanged:
- One `dossier_v{N}.md` per target — the single source of truth for the report
- 11 standardized section files (`01_sigint.md` … `08_voter.md`, `09_executive_summary.md`, etc.)
- Targeting → all 7 data agents + all 7 assessments + executive summary + narrative
- Trace → 7 data agents + trace report (no assessments)
- LEO → user-selected agents + executive summary + narrative (no assessments)

**Replacing demo tools with real APIs does NOT change which sections are written, the section-writing order, or the markdown structure.** Your tool produces the text; the formatter writes the section; the compiler stitches sections into the dossier; assessments scan the dossier. That whole chain is independent of where the data came from.

---

## 19. PLUG-AND-PLAY: WORKED EXAMPLES (Real APIs Per Agent)

Each subsection below is the full migration sketch — no LLM, no plan changes — for swapping the demo text-file tool with a real source. The output text format follows Section 18.2 Layer B.

### 19.1 SIGINT (CDR REST API)

See Section 18.3 for the complete walkthrough.

### 19.2 Travel (Airline GDS API)

```python
class TravelSearchTool(BaseAgentSearchTool):
    def __init__(self, db_path: str = ""):
        super().__init__(name="search_travel_records",
                         description="Search live airline/border-control records.",
                         db_path=db_path)

    def execute(self, query: str = "", search_terms: str = "", **kw) -> ToolResult:
        terms = [t.strip() for t in (search_terms or query).split(",") if t.strip()]
        try:
            trips = my_jupyter_gds_api(terms)  # your existing code
        except Exception as e:
            return ToolResult(success=False, error=f"GDS API: {e}")
        blocks = []
        for t in trips:
            blocks.append(
                f"FLIGHT: {t['airline']} {t['flight_no']}\n"
                f"PASSENGER: {t['name']}\n"
                f"PASSPORT: {t['passport']}\n"
                f"FROM: {t['origin']}\n"
                f"TO: {t['destination']}\n"
                f"DATE: {t['date']}\n"
                f"COMPANIONS: {', '.join(t.get('companions', []))}"
            )
        return ToolResult(success=True, data={
            "result": "\n\n".join(blocks),
            "record_count": len(trips), "total_records": len(trips),
        })
```

Travel formatter is permissive (free-form per record), so any consistent field layout works.

### 19.3 OSINT (Multiple Public APIs)

```python
def execute(self, query: str = "", search_terms: str = "", **kw) -> ToolResult:
    terms = [t.strip() for t in (search_terms or query).split(",") if t.strip()]
    results = []
    # Each API call wrapped in try/except — partial failure shouldn't kill the tool
    try: results.extend(my_linkedin_api(terms))
    except Exception as e: print(f"  LinkedIn: {e}")
    try: results.extend(my_twitter_api(terms))
    except Exception as e: print(f"  Twitter: {e}")
    try: results.extend(my_whois_api(terms))
    except Exception as e: print(f"  WHOIS: {e}")
    blocks = [f"SOURCE: {r['source']}\nFOUND: {r['summary']}\nURL: {r.get('url','')}"
              for r in results]
    return ToolResult(success=True, data={
        "result": "\n\n".join(blocks) if blocks else "No public records found.",
        "record_count": len(results), "total_records": len(results),
    })
```

### 19.4 FISA / CNE / Intel Reports / GEOINT — Same Pattern

Each one: subclass `BaseAgentSearchTool`, override `execute()`, call your real API, format output as `\n\n`-separated text blocks. The downstream formatters are all permissive (Section 18.2 Layer B) except SIGINT.

### 19.5 Mixing Demo + Real Sources During Migration

You can replace tools one at a time. The platform doesn't care if SIGINT hits a real API while Travel still reads `data/travel_db.txt`. The contract layer hides the source.

To run a hybrid: replace `tools/agent_specific/sigint/sigint_search_tool.py` only. Leave the others. Test. Then move on.

### 19.6 Plug-and-Play Checklist (Apply to EVERY Tool Replacement)

- [ ] `execute(self, query="", search_terms="", **kwargs) -> ToolResult` signature is exact
- [ ] `get_parameter_schema()` returns a dict with `required` and `properties`
- [ ] `__init__` calls `super().__init__(name=..., description=..., db_path=...)` with named args
- [ ] On success: `ToolResult(success=True, data={"result": str, "record_count": int, "total_records": int})`
- [ ] On API failure: `ToolResult(success=False, error="<msg>")` — NOT a try/except that swallows
- [ ] Output `data["result"]` is text, records separated by exactly `\n\n`
- [ ] For SIGINT: every block starts `CDR_<n>\n` with `FROM:`/`TO:`/etc. lines
- [ ] Verified locally with the Section 18.6 commands BEFORE restarting the server
- [ ] Restarted the server (Python module cache is stale otherwise)
- [ ] Confirmed boot log shows the agent loaded (no ImportError, plan executor count matches)
- [ ] Tested via `/single-agent` API endpoint (Section 18.6 step 8)
- [ ] Tested via chat tab — answer returns and dossier section gets written
- [ ] Tested via LEO/Targeting workflow — `data/dossiers/<id>/sections/01_<agent>.md` populated correctly

---

## 17. COMPLETE "IF I CHANGE X" QUICK REFERENCE

| I Want To Change... | Files To Modify |
|--------------------|-----------------|
| What SIGINT searches for | `plans/sigint.json` → step_1 params |
| How SIGINT results look in dossier | `living_dossier.py` → `_format_sigint()` |
| What the access assessment scans for | `living_dossier.py` → `ASSESSMENT_TYPES["access"]["prompt"]` |
| The executive summary prompt | `living_dossier.py` → `generate_executive_summary()` |
| Narrative report sections (targeting) | `living_dossier.py` → `NARRATIVE_SECTIONS` (has_assessments branch) |
| Narrative report sections (LEO/trace) | `living_dossier.py` → `NARRATIVE_SECTIONS` (else branch) |
| Trace report topics | `trace_report_synthesizer.py` → `NARRATIVE_TOPICS` |
| Which agents run in pipeline | `dossier_pipeline.py` → `DATA_AGENTS` list |
| Which assessments run | `dossier_pipeline.py` → `ASSESSMENT_AGENTS` list |
| Max enrichment loops | `config.py` → `ENRICHMENT_MAX_ITERATIONS` |
| Assessment chunk size | `config.py` → `DOSSIER_CHUNK_SIZE` |
| Max concurrent agents | `dossier_pipeline.py` → `max_workers` in `_step_3_collect()` |
| LLM provider/model | `.env` → `LLM_PROVIDER`, `LLM_MODEL`, `LLM_BASE_URL` |
| Contact network format | `living_dossier.py` → `compile_contact_network()` |
| Cross-reference format | `living_dossier.py` → `compile_cross_references()` |
| Section file order | `living_dossier.py` → `SECTION_ORDER` list |
| Name transliteration rules | `name_variation_generator.py` → dictionaries |
| Phone correlation logic | `tools/agent_specific/sigint/correlation_tool.py` |
| Co-traveler matching | `tools/agent_specific/travel/co_traveler_search.py` |
| Confidence score weights | `identity_scorer.py` → field weights at top |
| Max LLM calls per search | `config.py` → `TOOL_CALLING_MAX_ITERATIONS` |
| Add a new shared tool | `orchestrator.py` → `_init_tool_calling_engines()` AND `_init_plan_executors()` `tool_handlers` + new .py file (see Section 16 Step 5) |
| Add a new data agent | See APPENDIX above (10 steps) |
| Add a new assessment | See APPENDIX above (6 steps) |
| Replace a demo tool with a real API | See Section 18 (full migration + debugging guide) |
| Skip enrichment for trace | `dossier_pipeline.py` → `run()` → wrap Step 4 in condition |
| Change API endpoint format | `api_server.py` → find the endpoint decorator |
| Change frontend behavior | `frontend/index.html` — single file, all JS inline |
| Add a new workflow type | `dossier_pipeline.py` → `run(workflow_type=)` + new branch in `_step_7_*`; add new manager (e.g. `xyz_manager.py`); register endpoint in `api_server.py`; if it's user-facing, add UI tab in `frontend/index.html` |
| Restrict an agent so other agents cannot cross-query it | `orchestrator.py` → `_handle_agent_query()` allowlist; remove from `query_agent` description |
| Hide a tool from a specific agent | `orchestrator.py` → in `_init_tool_calling_engines()` loop, skip the `engine.register_tool(...)` call for that agent |
| Make a tool log every call | `orchestrator.py` → handler method; use `self._tool_log("tool_name", "msg")` |
| Verify all tools registered correctly | Run debug commands in Section 18.6 |

---

## 20. TOOL-BY-TOOL REFERENCE (every tool, what it does, how to change it)

This is the deep reference for every tool in the platform. **Each entry has the same structure:** what the tool does, how it works internally, where it's called, the exact methods you can override, and the most common modifications.

There are three categories:

| Category | Where they live | Who uses them |
|----------|----------------|----------------|
| **Framework** | `tools/` root + `tools/database_connectors/` | All agents — provide the contract every other tool inherits |
| **Shared tools** | `tools/shared/` | All agents (registered by orchestrator into every tool-calling engine) |
| **Agent-specific tools** | `tools/agent_specific/<agent>/` | One specific data agent (search tools, adapters, post-processors) |

The total tool inventory:

- **Framework**: 5 tools (BaseTool contract, tool registry, tool definitions, JSON connector, text-file connector)
- **Shared (global)**: 19 tools — orchestration (living_dossier, dossier_pipeline, plan_executor, tool_calling_engine, sub_agent_processor, trace_report_synthesizer), persistence (target_store, results_store), enrichment (enrichment_tracker, contact_search, cross_target_linker, entity_resolver, identity_scorer), utilities (name_variation_generator, voter_db_tool, datetime_tool, translation_tool, agent_query_tool, event_bus)
- **Agent-specific**: 13 tools — 7 search tools (one per data agent), 4 adapters (sigint_adapter, cdr_query, fisa_query, travel_query), 1 correlation tool (sigint), 2 SNA tools (build_network, analyze_network), 1 co-traveler search

---

### 20.1 Framework Tools

#### 20.1.1 `tools/base_tool.py` — The contract every tool obeys

**What it is:** abstract base class + `ToolResult` dataclass. **Every tool in the platform must subclass `BaseTool`.**

**The contract (do not violate or downstream code crashes):**

```python
@dataclass
class ToolResult:
    success: bool          # False = caller treats as failure, surfaces error
    data: Any = None       # On success, contains the tool's output (usually a dict)
    error: str = ""        # On failure, human-readable error message
    tool_name: str = ""    # Auto-set by some callers
    execution_time_ms: float = 0  # Auto-set by tool registry

class BaseTool(ABC):
    def __init__(self, name: str, description: str, category: str, version: str = "1.0.0"):
        # name = unique identifier ("search_sigint_records", "voter_db_search", etc.)
        # description = appears in the LLM system prompt — write it from the LLM's POV
        # category = "agent_search", "api_integrations", "database_connectors", etc.

    @abstractmethod
    def get_parameter_schema(self) -> Dict[str, Any]:
        """JSON-schema-style dict: {"required": [...], "properties": {...}}"""

    @abstractmethod
    def execute(self, **kwargs) -> ToolResult:
        """Do the work. Return ToolResult — NEVER a plain dict."""
```

**How to change:** don't, unless you really mean to. This is the stable contract that lets you swap any underlying tool implementation without changing callers (Section 18 plug-and-play depends on it).

**When you'd touch it:** only to add fields to `ToolResult` (e.g., `confidence: float`, `cached: bool`). Adding fields is backwards-compatible; renaming fields breaks every tool.

---

#### 20.1.2 `tools/tool_registry.py` — Tool discovery and execution metrics

**What it is:** in-process registry that holds tool instances by name and tracks every execution.

**Key methods:**
- `register(tool: BaseTool)` — add to registry
- `execute_tool(name, **kwargs) -> ToolResult` — invoke by name, time it, log it
- `list_tools(category=None)` — list all tools, optionally filtered
- `get_metrics()` — execution count, avg latency, success rate per tool

**Where it's called:** `api_server.py` exposes `/tools`, `/tools/execute/{name}`, and `/metrics/tools` endpoints that proxy directly to this registry.

**How to change behavior:**

| Want to... | Edit |
|-----------|------|
| Add per-tool rate limiting | `execute_tool()` — track timestamps per name, sleep if over threshold |
| Add tool-call audit log to disk | `execute_tool()` — append a JSON line to a log file after each call |
| Block dangerous tools | `register()` — raise if `tool.name` in deny-list |
| Auto-load tools from a directory | `load_tools_from_dir()` — already exists, walks `tools/` and instantiates BaseTool subclasses |

---

#### 20.1.3 `tools/tool_definitions.py` — JSON schema for the LLM

**What it is:** centralized dict mapping `tool_name → {description, parameters, category}`. The tool-calling engine reads this when building the system prompt for the LLM, so the LLM knows what tools exist and how to call them.

**Key constant:** `TOOL_DEFINITIONS` (large dict at module top). Each entry:

```python
"correlate_identifiers": {
    "description": "Two-stage SIGINT correlation: takes a phone, finds IMEI/IMSI...",
    "parameters": {
        "type": "object",
        "properties": {
            "phone": {"type": "string", "description": "Phone number"},
            "imei": {"type": "string", "description": "Optional IMEI"},
            "imsi": {"type": "string", "description": "Optional IMSI"},
        },
        "required": ["phone"],
    },
    "category": "database_query",
},
```

**How to change:**
- **Add a new tool definition** here, then create the matching handler in `orchestrator.py` (`_handle_<tool_name>`) and register both with the tool-calling engine. **You must do all three or the tool won't work.**
- **Edit a description** to change how the LLM thinks about the tool. The LLM's tool selection is driven by these descriptions — make them precise.

**Common pitfall:** if you add a tool here but forget to add the handler, the LLM will call it and get a "tool not found" error. The reverse (handler without definition) means the LLM never knows it exists.

---

#### 20.1.4 `tools/database_connectors/json_connector.py` — Generic JSON file search

**What it does:** opens a JSON file, walks it, returns records matching a query. Mostly used as a debug/utility tool — agent search tools have their own implementations.

**Key method:** `execute(db_path: str, query: str = "")` — loads JSON, returns matches.

**When to use:** if you have a structured JSON dataset that doesn't fit any agent (e.g., a vendor list, a code dictionary). Don't use for the agent databases — those use `BaseAgentSearchTool` for proper formatting.

#### 20.1.5 `tools/database_connectors/text_file_connector.py` — Generic text file search

**What it does:** searches a text file for records (separated by `\n\n`) matching a query. Same shape as `BaseAgentSearchTool` but standalone — exposed via the LLM tool `search_text_database` so agents can search any text DB by name.

**When to use:** when an agent's plan needs to search a sibling agent's database directly without going through `query_agent`. Used internally by some plans.

---

### 20.2 Shared Tools (global — available to every agent)

#### 20.2.1 `tools/shared/living_dossier.py` — The dossier engine

**What it is:** the single most important file in the platform. Builds, maintains, and reads the per-target markdown dossier.

**What it does:**
- Creates the dossier directory (`data/dossiers/<target_id>/sections/`)
- Writes per-agent section files (`01_sigint.md`, `02_travel.md`, etc.) — one per data agent
- Generates per-section summaries via sub-agent chunking
- Compiles cross-references across sections (shared identifiers)
- Builds the contact network section
- Writes voter results
- Runs the 7 chunk-by-chunk assessment scans
- Generates the executive summary + narrative report
- Stitches all sections into the final `dossier_v{N}.md`

**Key constants you'll touch most:**

| Constant | What it controls | Line ~ |
|----------|-----------------|--------|
| `SECTION_ORDER` | Order of sections in compiled dossier | 39 |
| `ASSESSMENT_TYPES` | Dict of 7 assessment types: `prompt`, `keywords`, `title` | 54 |
| `NARRATIVE_SECTIONS` | Two layouts: targeting (assessment-based) vs LEO/trace (data-based) | ~1232 |

**Per-agent formatters** (lines ~1490-1673):
- `_format_sigint(data, identifiers)` — parses CDR_*** records, groups by contact phone, builds interaction tables
- `_format_travel(data, identifiers)` — passenger trips, sorted chronologically
- `_format_fisa(data, identifiers)` — surveillance orders + intercepts
- `_format_cne(data, identifiers)` — device extractions; tags records with "DELETED/RECOVERED" if found
- `_format_intel_reports(data, identifiers)` — narrative reports preserved verbatim
- `_format_osint(data, identifiers)` — social media + public records
- `_format_geoint(data, identifiers)` — geo-points with lat/lon
- `_format_generic(data, identifiers)` — fallback for any unknown agent

**How to change behavior:**

| Want to... | Edit |
|-----------|------|
| Add a new section to the dossier | `SECTION_ORDER` (insert tuple `("name", "0X", "TITLE")`); add corresponding writer method |
| Change how SIGINT records are grouped | `_format_sigint()` — currently groups by contact phone, sorts by frequency |
| Change which fields are extracted from a CDR record | `_format_sigint()` — see the line-by-line `if line.startswith("FROM:")` block |
| Add a new assessment type | `ASSESSMENT_TYPES` dict (see APPENDIX) |
| Change assessment chunk size | `DOSSIER_CHUNK_SIZE` in `config.py` (default 3000 chars) |
| Change executive summary prompt | `generate_executive_summary()` |
| Change narrative report sections | `NARRATIVE_SECTIONS` — has two layouts, `if has_assessments` branch (targeting) vs else (LEO/trace) |
| Add target identity check (prevent wrong-person summaries) | `summarize_section()` — already accepts `identifiers`; injects target name into the prompt |
| Change cross-reference detection | `compile_cross_references()` — add identifier types to scan for |
| Change contact network format | `compile_contact_network()` |

**Critical invariants — don't change without thinking:**
- Section files are named `NN_<key>.md` where NN is from `SECTION_ORDER`
- The dossier file is named `dossier_v<N>.md` with `N` incrementing per pipeline run
- Record counters look for literal markers `#### CONTACT:`, `#### Order `, `#### Record `, `#### Report ` — see Section 18.2 Layer B

---

#### 20.2.2 `tools/shared/dossier_pipeline.py` — The 7-step pipeline orchestrator

**What it is:** the pipeline controller that calls all the other tools in the right order.

**The 7 steps:**

| Step | Method | What happens | LLM calls |
|------|--------|--------------|-----------|
| 1. INPUT | `create_dossier()` | Make `data/dossiers/<target_id>/sections/` | 0 |
| 2. EXPAND | `_step_2_expand()` | Name variations + phone correlation | 0 (regex + dictionary) |
| 3. COLLECT | `_step_3_collect()` | Run data agents in parallel, write sections | many (agents do their own) |
| 4. ENRICH | `_step_4_enrich()` | Discover contacts, search new identifiers, loop | many |
| 5. COMPILE | `_step_5_compile()` | Cross-refs, contact network, voter check | 0 |
| 6. ASSESS | `_step_6_assess()` | 7 assessment scans, chunk by chunk | many |
| 7. FINAL | `_step_7_final()` (or `_step_7_trace_report()`) | Executive summary + narrative + save report | 2 (summary + narrative) |

**Key constants:**
- `DATA_AGENTS` (line 31) — `["travel", "osint", "intel_reports", "sigint", "fisa", "cne", "geoint"]`
- `ASSESSMENT_AGENTS` (line 34) — the 7 assessment types

**The `run()` signature:**

```python
def run(self, identifiers: Dict, target_id: str, run_id: str,
        agent_keys: Optional[List[str]] = None,
        workflow_type: str = "targeting") -> Dict:
    # workflow_type: "targeting" (full), "trace" (skip assessments + use trace report),
    #                "leo" (skip assessments)
    # agent_keys: subset of DATA_AGENTS to run; None = all
```

**How to change behavior:**

| Want to... | Edit |
|-----------|------|
| Add a new data agent to the pipeline | `DATA_AGENTS` list; ensure it's also in `orchestrator._init_tool_calling_engines()` and has a section in `living_dossier.SECTION_ORDER` |
| Make a workflow type skip a step | `run()` — wrap the step's `if workflow_type not in (...)` |
| Change parallelism for data agents | `_step_3_collect()` — `max_workers = min(5, len(data_agents))`; raise/lower the cap |
| Change enrichment iteration limit | `_step_4_enrich()` — `MAX_ITERATIONS` from `EnrichmentTracker`; or pass `max_iterations=` |
| Add a new compile step (like voter, cross-ref) | `_step_5_compile()` — add a `5d` block similar to existing 5a/5b/5c |
| Restrict cross-agent queries to selected agents | already done — `set_active_agent_scope(data_agents)` for non-targeting workflows |

**Where the bug fixes live:**
- BUG #2 cross-agent gating: `set_active_agent_scope(data_agents)` for LEO/trace, `None` for targeting — line ~85
- BUG #2 enrichment scoping: `allowed_agents` passed into `tracker.get_search_queue()` — line ~589
- BUG #2 contact search scoping: `_run_secondary_search(... search_agents=allowed_agents)` — line ~688

---

#### 20.2.3 `tools/shared/plan_executor.py` — Deterministic JSON-plan runner

**What it is:** runs JSON plans (in `plans/<agent>.json`) step by step. Replaces the LLM-driven loop for predictable, repeatable agent behavior.

**Action types** in plans:
- `tool_call` — invoke a registered tool with templated parameters
- `llm_analysis` — send accumulated data to LLM with a constrained prompt
- `llm_synthesis` — final compile/synthesis step
- `for_each` — loop sub-steps over items from a prior step

**Template variable substitution:** `{{key}}` in any field gets replaced from the context dict. `{{identifiers.phone}}` reaches into nested dicts. `{{step_id.field}}` references prior step outputs.

**Key methods:**
- `run(plan, identifiers, auto_complete_large_results=False)` — execute a full plan
- `_chunked_llm_call(prompt, large_data)` — when input prompt > `LARGE_PROMPT_THRESHOLD` (6000 chars), uses sub-agent processor to chunk the data
- `_resolve_template(text, context)` — substitutes `{{...}}` references

**How to change behavior:**

| Want to... | Edit |
|-----------|------|
| Change the chunking threshold for large prompts | `LARGE_PROMPT_THRESHOLD` constant near top |
| Add a new action type to plans | `_dispatch_step()` — add an `elif step["action"] == "your_type"` branch |
| Add a new template helper | `_resolve_template()` — handle a new `{{custom_helper(...)}}` form |
| Change how `for_each` parallelizes | `_handle_for_each()` |
| Inject extra guardrails into every LLM prompt | top of `_run_llm_step()` — system prompt gets the plan's `guardrails` list |

**Critical invariant:** the keys filtered out before substitution are those starting with `_` (underscore). Internal pipeline metadata (like `_target_id`, `_search_groups`) is hidden from templates this way.

---

#### 20.2.4 `tools/shared/tool_calling_engine.py` — LLM-driven tool loop

**What it is:** the alternative to PlanExecutor — gives the LLM a list of tools and lets it decide what to call. Used as fallback for agents without a plan, and for chat mode.

**The loop:**
1. Build system prompt with role + tool descriptions (from `tool_definitions.py`)
2. Send user message to LLM
3. Parse XML tool calls from response: `<search_sigint_records><query>+964</query></search_sigint_records>`
4. Execute each tool call
5. Feed results back as next user message
6. Repeat until `<attempt_completion>` or `MAX_ITERATIONS`

**Key features:**
- **Sub-agent chunking**: tool results > 4000 chars get split into chunks, each chunk processed by a sub-agent for full record extraction (no truncation)
- **Auto-complete on large results** (BUG #9 fix): if `auto_complete_large_results=True` (chat mode), engine returns immediately after first big tool result instead of looping
- **No-data early exit**: after 2 consecutive empty tool results, forces completion to avoid infinite no-result loops
- **SSE event emission**: every tool call and LLM call emits `tool_called`, `tool_result`, `llm_call_started`, `llm_call_complete` for live UI streaming

**How to change:**

| Want to... | Edit |
|-----------|------|
| Change max iterations | `MAX_ITERATIONS` constant or pass `max_iterations=` to `run()` |
| Change max consecutive mistakes before stopping | `MAX_MISTAKES` constant |
| Change sub-agent chunking threshold | `_should_chunk_result()` — currently `> 4000 chars` |
| Disable XML tool parsing (use OpenAI function calling instead) | swap `_parse_tool_calls()` for OpenAI `tools=` parameter handling |
| Change the system prompt template | `_build_system_prompt()` |

**Critical methods to know:**
- `register_tool(tool_def: ToolDefinition)` — adds a tool to this engine instance
- `run(system_prompt, user_message, ...)` — the main entry point

---

#### 20.2.5 `tools/shared/sub_agent_processor.py` — Chunking + parallel sub-agents

**What it is:** when there's too much data for a single LLM call, this splits it into batches and runs concurrent sub-agent workers.

**Process:**
1. Split data into batches of N records (default 2)
2. Spawn `asyncio` workers, each gets the parent agent's `role_description` for domain context
3. Each worker returns structured findings with citations + confidence
4. Findings persist to `ResultsStore` (SQLite)
5. Returns deduplicated findings + `job_id` for user review

**Key methods:**
- `execute(data, query, parent_agent_role, parent_agent_name, source_label, records_per_worker, record_delimiter)` — the entry point
- `_extract_findings_from_chunk(chunk, query, role, ...)` — what each sub-agent does
- `_dedupe_findings(findings)` — collapse duplicates by content hash

**How to change:**

| Want to... | Edit |
|-----------|------|
| Change records per worker | `records_per_worker` parameter (default 2). Smaller = more workers, more LLM cost, finer granularity |
| Change parallelism cap | `MAX_WORKERS` constant — currently 4 |
| Change record splitter | `record_delimiter` parameter (default `\n\n`) |
| Add a new finding type | `_extract_findings_from_chunk()` prompt; update the JSON schema returned |
| Skip persistence | pass a `None` results_store to constructor |

---

#### 20.2.6 `tools/shared/trace_report_synthesizer.py` — Trace mode reports

**What it is:** when workflow_type is `trace`, this builds a "where did the target appear" report instead of an assessment.

**Structure:**
- Section 1: Summary
- Section 2.1: Name search results (per agent)
- Section 2.2: Phone number search results (with correlation data)
- Section 2.3: Other identifiers (email, passport, IMEI, locations)
- Section 3: Narrative prose

**Key constant:** `NARRATIVE_TOPICS` (line ~289) — list of `(topic_key, title, agent_keys, prompt)` tuples. 5 entries by default.

**How to change:**
- **Add a narrative topic** — append to `NARRATIVE_TOPICS`
- **Change report sections** — edit `synthesize_trace_report()` directly
- **Adjust narrative prompt** — edit each tuple's prompt field

---

#### 20.2.7 `tools/shared/target_store.py` — Per-target SQLite persistence

**What it is:** the durable, audit-trailed store of every record collected about every target.

**Schema (20 tables):**
- `targets` — one row per target, holds identifiers
- `runs` — one row per pipeline execution
- `records` — one row per individual record (content-hashed for dedup)
- `tombstones` — user-deleted records (never re-imported)
- `reports` — versioned final reports
- `kg_entities`, `kg_triples` — knowledge graph (Layer 2)
- `memory_log` — append-only memory drawers (Layer 1)
- `facts` — extracted facts (passport=X, email=Y, etc.)
- `target_links` — bidirectional cross-target connections
- `merge_candidates` — entity-resolution flags
- `chat_sessions`, `chat_messages` — chat persistence
- `identifier_searches` — audit log of which identifiers each agent searched
- ... and 6 more

**Key methods:**
- `get_or_create_target(identifiers)` — returns `(target_id, was_created)`
- `start_run(target_id, agents, leo_id=)` → `run_id`
- `complete_run(run_id, stats)` — finalizes the run
- `store_record(...)` — single record with dedup via content hash
- `store_records_batch(...)` — bulk insert, returns stats
- `tombstone_record(record_id)` — block re-import
- `save_report(target_id, run_id, ...)` — versioned report save
- `get_report_by_version(target_id, version)` — fetch
- All chat methods: `create_chat_session`, `add_chat_message`, etc.

**WAL mode** is enabled — concurrent reads while pipeline writes.

**How to change:**

| Want to... | Edit |
|-----------|------|
| Add a new table | Add `CREATE TABLE` to `_init_schema()` + add accessors |
| Change dedup logic | `_compute_content_hash()` — currently MD5 of JSON-serialized record |
| Add a new field to records | Add column to `_init_schema()` and update `store_record()` |
| Export to a different DB (Postgres, etc.) | Replace `sqlite3` calls — they're fenced behind a `_get_conn()` helper |

**Critical invariant:** `store_record()` is idempotent via content hash — calling it twice with the same record is safe.

---

#### 20.2.8 `tools/shared/results_store.py` — Sub-agent processing audit log

**What it is:** stores every sub-agent processing job, every worker's input/output, and every individual finding.

**Used by:** `sub_agent_processor.py` writes here; the API endpoint `/results/jobs/{job_id}` reads here so users can drill into what each sub-agent did.

**Why separate from TargetStore:** sub-agent jobs aren't always about a specific target (some are global searches). Mixing would muddy TargetStore.

**Schema:**
- `jobs` — one row per processing batch
- `workers` — one per sub-agent invocation, with raw input/output
- `findings` — extracted findings per worker

---

#### 20.2.9 `tools/shared/event_bus.py` — In-process pub/sub for SSE

**What it is:** a thread-safe, async-aware event bus that the pipeline emits to and the SSE endpoint streams from.

**No external dependencies** — no Redis, no Kafka. Pure `asyncio.Queue` per subscriber, protected by `threading.Lock`.

**Key methods:**
- `publish(event_type, **fields)` — fire-and-forget, called from pipeline code
- `subscribe(client_id) → asyncio.Queue` — called by the SSE endpoint
- `unsubscribe(client_id)` — clean up when client disconnects

**How to change:**
- **Add a new event type** — call `bus.publish("your_event", agent_key=..., data=...)` from anywhere; the SSE consumer picks it up
- **Filter events per subscriber** — wrap `subscribe()` to take a filter function

---

#### 20.2.10 `tools/shared/agent_query_tool.py` — Cross-agent database query

**What it is:** the tool that lets one agent query another agent's database. Registered as `query_agent` in every agent's tool-calling engine.

**Smart behavior:**
- Checks `TargetStore` FIRST (so already-collected data isn't re-fetched)
- Falls back to running the target agent's `tool_calling_engine` if not cached
- Recursion guard prevents `agent_A → agent_B → agent_A → ...` loops
- Cache populated by `prepopulate_cache(agent_key, result, variants)` after each data agent runs in step 3

**Key methods:**
- `set_agents(agents_dict)` — orchestrator calls this once at boot
- `set_target_store(store)` — same
- `set_target_identifiers(identifiers)` — set per-pipeline target context
- `prepopulate_cache(agent_key, result, variants)` — preload after step 3
- `clear_cache()` — between targets
- `execute(agent_key, query, mode="database")` — the actual query

**Cross-agent gating** (BUG #2 fix): `_handle_agent_query()` in `orchestrator.py` checks `_active_agent_keys` before calling this tool. The query_tool itself doesn't gate — orchestrator does.

---

#### 20.2.11 `tools/shared/contact_search.py` — Secondary contact search

**What it is:** after data agents extract phone contacts (CDR contacts, FISA all_contacts, Travel co-travelers), this tool searches every agent's database for those contact phones to find more intel about each contact.

**Zero LLM calls** — pure text matching.

**Key method:**
```python
execute(contacts, target_phone="", search_agents=None) -> ToolResult
# search_agents: None = search all agents (full Targeting); list = restrict to those (LEO)
```

**Skip-empty-DB optimization** (BUG #8 fix) — checks `db_path` size before loading; assessment agents with 0-byte DBs are skipped.

**How to change:**

| Want to... | Edit |
|-----------|------|
| Change name extraction from records | `_extract_name_from_records()` |
| Change risk classification | `_classify_risk()` — currently rule-based on # of databases found in |
| Change phone matching | `_phone_matches_in_record()` — handles country code normalization |
| Add a new contact source (e.g., social media handles) | extend `_search_database()` to accept a `username` field |

---

#### 20.2.12 `tools/shared/cross_target_linker.py` — Inter-target connections

**What it is:** when a new identifier is found for Target A, checks if any other Target B has the same identifier. Creates bidirectional `target_links` if found.

**Confidence weights** (line ~32):
- passport = 1.0
- phone = 0.95
- email = 0.9
- imsi = 0.9
- imei = 0.85
- address = 0.6

**Secondary link types:**
- `co_communication` — Target A frequently calls a number belonging to Target B
- `co_travel` — Target A and B share a flight + date

**Key methods:**
- `check_all_links_for_target(target_id, run_id)` — the main entry called from `_step_5_compile()`
- `_check_shared_identifiers(target_id)` — type-by-type ID match
- `_check_co_communication(target_id)` — CDR analysis
- `_check_co_travel(target_id)` — PNR matching

**How to change:**
- Add identifier types to the confidence weights dict
- Add new secondary link types — implement a `_check_<type>` method and call from `check_all_links_for_target()`

---

#### 20.2.13 `tools/shared/entity_resolver.py` — Same-person merge candidates

**What it is:** detects when Target A and Target B might actually be the same person (different intake records).

**Match rules** (line ~25):
| Score | Condition | Action |
|-------|-----------|--------|
| 1.0 | Same passport | auto-link + flag for merge |
| 0.95 | Same phone + similar name | auto-link + flag for merge |
| 0.7 | Same phone, different name | auto-link + flag for review |
| 0.5 | Similar name + same city | flag for review only |
| 0.4 | Shared associate + similar pattern | flag for review only |

**Zero LLM calls** — Jaccard name similarity + exact-match on normalized identifiers.

**Key methods:**
- `check_merge_candidates(target_id)` — runs all rules, writes to `merge_candidates` table
- `_compute_name_jaccard(name_a, name_b)` — token-overlap similarity
- `_normalize_phone(phone)` — strip dashes/spaces

**How to change:**
- Edit the rule weights in the constructor's threshold dict
- Add a new rule — write `_check_<rule>` method, call from `check_merge_candidates()`

---

#### 20.2.14 `tools/shared/enrichment_tracker.py` — Iterative enrichment bookkeeping

**What it is:** Step 4 of the pipeline calls this to track which identifiers have been discovered and which agents have searched them. Pure Python, zero LLM.

**Key constant:** `IDENTIFIER_AGENT_MAP` (line 29) — maps identifier type → list of agents that should search it:

```python
IDENTIFIER_AGENT_MAP = {
    "phone":    ["sigint", "fisa", "geoint", "cne", "travel", "osint", "intel_reports"],
    "name":     ["travel", "osint", "intel_reports", "fisa", "geoint", "cne"],
    "passport": ["travel"],
    "email":    ["cne", "osint"],
    "imei":     ["sigint"],
    "imsi":     ["sigint"],
    "ip":       ["cne", "fisa"],
    "username": ["osint", "cne"],
}
```

**Key methods:**
- `add_discovered(id_type, value, source, iteration)` — record a new finding
- `mark_searched(id_type, value, agent_key)` — record an agent's coverage
- `get_search_queue(allowed_agents=None)` — returns unsearched (type, value, agents) — **respects allowed_agents** (BUG #2 fix)
- `extract_from_results(raw_results, iteration)` — regex-extracts new IDs from agent outputs
- `seed_from_identifiers(identifiers)` — seed the queue with the user's input

**How to change:**

| Want to... | Edit |
|-----------|------|
| Add a new identifier type | Add to `IDENTIFIER_AGENT_MAP`; add regex to `extract_from_results()` |
| Change which agents handle a type | Edit the list in `IDENTIFIER_AGENT_MAP` |
| Tune iteration limit | `MAX_ITERATIONS` (default 3); `MAX_NEW_PER_ITERATION` (default 10000) |
| Change the priority order | `_PRIORITY` list — `["phone", "name", "passport", ...]` |

---

#### 20.2.15 `tools/shared/identity_scorer.py` — Confidence scoring per record

**What it is:** scores how confidently a stored record belongs to the target.

**Confidence levels:**
- HIGH (≥ 0.7): 2+ strong fields match (name+phone, name+passport)
- MEDIUM (0.4–0.7): Full name match but no corroborating fields
- LOW (< 0.4): Partial name or weak match

**Key method:** `score_record(target_identifiers, record) -> {confidence, score, reasoning}`

Every record stored via `_lightweight_store()` in dossier_pipeline gets tagged with `confidence_level`, `confidence_score`, and `reasoning`.

**How to change:**
- Edit field weights in `score_record()` (name = 0.4, phone = 0.3, passport = 0.4, etc.)
- Add a new field — extend the scoring logic

---

#### 20.2.16 `tools/shared/name_variation_generator.py` — Arabic/English name expansion

**What it is:** generates all known transliterations of Arabic names. `محمد` → `Mohammad, Mohammed, Muhammad, Muhammed, Mohamad`.

**Two paths:**
1. **Hardcoded dictionary** (`ARABIC_NAME_VARIATIONS`) — fast, no LLM call. Covers ~200 common names.
2. **LLM expansion** — for names not in the dictionary, calls the LLM to generate variations.

**Also handles:**
- Kunya (`أبو خالد` ↔ `Abu Khalid`)
- Phone variations (strip `+`, country code variants)

**Key method:** `execute(name, use_llm=True) -> ToolResult` returning `{search_terms, full_name_variations, total_part_variations}`.

**How to change:**

| Want to... | Edit |
|-----------|------|
| Add a hardcoded variation | `ARABIC_NAME_VARIATIONS` dict — add `"name_in_arabic": ["en1", "en2", ...]` |
| Disable LLM expansion globally | Pass `use_llm=False` (called from `_handle_expand_name()` in orchestrator) |
| Add Persian/Turkish/Urdu support | Add a parallel dictionary, branch on script detection |

---

#### 20.2.17 `tools/shared/voter_db_tool.py` — Iraq voter CSV search

**What it is:** searches an in-memory CSV (`data/voter.csv`) of Iraqi voter registrations. All columns are Arabic.

**Columns:** `first_name, father_name, grandfather_name, family_name, voter_id, national_id, city, mother_name, phone, date_of_birth, gender, registration_date`

**Key method:** `execute(name="", phone="", city="", national_id="")` — accepts any combination of fields.

**Important:** the LLM should call `expand_name_variations` first to convert English names to Arabic. The voter DB has NO English entries.

**How to change:**

| Want to... | Edit |
|-----------|------|
| Add a search field | `execute()` parameter list + matching logic |
| Change CSV path | `Config.VOTER_CSV_PATH` (in `config.py`) |
| Add fuzzy matching | `_match_field()` — currently exact substring match |

---

#### 20.2.18 `tools/shared/datetime_tool.py` — Date math for the LLM

**What it is:** the LLM doesn't know what year it is. This tool gives it current date and lets it compute ranges.

**Operations:**
- `now` — current date/time
- `range` — default range for a data type (`cdr`=1yr, `intel_reports`=10yr, `fisa`=5yr, `travel`=3yr)
- `subtract` — custom subtraction (`days_back`, `months_back`, `years_back`)

**Default time ranges per data type** (line ~30):
```python
DEFAULT_RANGES = {
    "cdr": 365, "intel_reports": 3650, "fisa": 1825, "travel": 1095, "osint": 730,
}
```

**How to change:**
- Edit `DEFAULT_RANGES` to change defaults
- Add a new operation in `execute()`

---

#### 20.2.19 `tools/shared/translation_tool.py` — External translation/OCR

**What it is:** integrates with an external translation/OCR API (URL + key in `config.py`).

**Operations:**
- `detect` — language detection
- `translate` — text translation
- `ocr` — extract text from an image/document
- `verify` — cross-check an LLM translation against the API

**Configuration:** `TRANSLATION_API_URL` and `TRANSLATION_API_KEY` in `config.py`. If unavailable, returns an error (no silent failure).

**How to change:**
- Replace the API endpoint — edit URL constant and request format in each operation method
- Add a new operation — add an `elif operation == "..."` branch in `execute()`

---

### 20.3 Agent-Specific Tools (per data agent)

#### 20.3.1 `tools/agent_specific/base_agent_search_tool.py` — The base for all 7 search tools

**What it is:** the parent class of every data agent's search tool. Provides text-file-search out of the box; subclasses just provide `name`, `description`, and `db_path`.

**Why it exists:** so all 7 search tools share search semantics (records split by `\n\n`, multi-word AND matching, phone-as-digit matching with country-code variants), and so the orchestrator handler doesn't need to know how each tool gets its data.

**Key built-ins:**
- `_load_data()` — reads the text file (cached, invalidate via `invalidate_cache()`)
- `_text_search(content, search_terms)` — multi-word word-boundary matching with phone-aware fallback
- `_phone_matches_in_record(search_norm, record_text)` — country-code-aware phone matching
- `_normalize(text)` — strip dashes/spaces/parens

**The `execute()` contract** (which subclasses inherit unless overridden):

```python
def execute(self, query: str = "", search_terms: str = "", **kwargs) -> ToolResult:
    # query: original user query (for display/logging)
    # search_terms: comma-separated, pre-expanded list (orchestrator handles this)
    # Returns: ToolResult with data["result"] (text), data["record_count"], data["total_records"]
```

**How to change for plug-and-play (real APIs):** override `execute()` in the subclass — that's the entire migration. See Section 18.3 for the SIGINT example.

---

#### 20.3.2 SIGINT tools (`tools/agent_specific/sigint/`)

##### `sigint_search_tool.py` — Main agent search tool

**What it does:** searches `data/sigint_db.txt` for CDR/SMS/IMEI/IMSI records. Inherits from `BaseAgentSearchTool`.

**To replace with real CDR API:** override `execute()` (Section 18.3 has the full walkthrough). The output text MUST follow the SIGINT layer-B format: blocks separated by `\n\n`, each starting with `CDR_NNN`, with `FROM:`, `TO:`, `TYPE:`, `DATE:`, `TIME:`, `DURATION:`, `TOWER_FROM:` lines.

##### `sigint_adapter.py` — Subscriber correlation table parser

**What it does:** parses the MSISDN/IMEI/IMSI correlation table format (pipe-delimited rows like `MSISDN: +964-... | IMEI: ... | IMSI: ... | CARRIER: ...`).

**Why it exists:** the SIGINT database has TWO types of records — CDR call records AND a correlation table. The adapter handles the correlation rows so they can be stored as structured records in TargetStore.

**Used by:** `dossier_pipeline._lightweight_store()` — line ~447 (`tool_map["sigint"] = (SIGINTAdapter(), ...)`)

**Output:** `{records: [{phone, imei, imsi, carrier, first_seen}, ...]}`

**How to change:**
- If your real CDR system uses a different correlation format, edit `_parse_record()` in this file
- If you have NO correlation table (only CDR), return `{records: []}` — the rest of the pipeline still works

##### `cdr_query.py` — Detailed CDR parser

**What it does:** parses CALL_LOG_### and SMS_LOG_### text records (the older CDR format). Extracts From/To/Time/Tower/Duration plus a deduplicated contact list sorted by frequency.

**Used by:** `tool_calling_engine` registers it as `cdr_query` (LLM can call directly). Plans use it for structured CDR analysis.

**Output:** `{records, contact_list, contact_summary}` — contact_list is sorted by interaction count.

**How to change:**
- If switching to a real CDR API, the API likely returns structured JSON already; either bypass this tool or rewrite `_parse_record()` to accept JSON

##### `correlation_tool.py` — Two-stage SIGINT correlation

**What it does:** the most powerful SIGINT tool. Given a phone, finds associated IMEIs/IMSIs from the correlation database, then searches AGAIN with the expanded set to find new phones (SIM swaps, device sharing, SIM cloning).

**Two stages:**
1. Phone → query correlation_db → get IMEI(s) + IMSI(s)
2. Expanded set (phone + IMEI + IMSI) → query again → discover NEW phones/IMEIs/IMSIs sharing identifiers

**Output structure:** `{primary_phone, related_phones, related_imeis, related_imsis, network_topology}`

**Used by:** `_handle_correlate()` in orchestrator. Called BEFORE `cdr_query` to expand the search scope.

**How to change:**
- Add stage 3 (re-search after stage 2) for deeper SIM-swap detection
- Tune the threshold for what counts as "shared identifier" (currently exact match)

---

#### 20.3.3 Travel tools (`tools/agent_specific/travel/`)

##### `travel_search_tool.py` — Main agent search tool

**What it does:** searches `data/travel_db.txt` for PNR/flight records. Inherits from `BaseAgentSearchTool`.

**Layer B output format:** records separated by `\n\n`. Each PNR is free-form (no required keys), but consistent fields like `NAME:`, `FLIGHT:`, `DATE:`, `FROM:`, `TO:`, `PHONE:`, `DOC_NUMBER:` make downstream parsing easier.

**To replace with real airline/border API:** see Section 19.2.

##### `travel_query.py` — PNR record adapter

**What it does:** parses both `PNR_###` (modern) and `TRAVEL_RECORD_###` (legacy) record formats. Extracts trips and builds a co-traveler list.

**Used by:** `_lightweight_store()` in `dossier_pipeline.py`.

**Output:** `{trips, co_traveler_list, route_summary}`

**How to change:**
- If your real source returns one PNR-per-record JSON, replace `_parse_record()` to accept JSON
- Add new fields to extract (frequent flier number, baggage tags, etc.) by extending the field-extraction loop

##### `co_traveler_search.py` — Find passengers on same flights

**What it does:** given the target's travel records, extracts each FLIGHT+DATE pair, then searches the full travel database for OTHER passengers on those same flights.

**Output:** `{co_travelers, trip_overlaps}` — co_travelers sorted by # of shared flights.

**Used by:** plans (e.g., `travel.json` calls `search_co_travelers`); orchestrator handler `_handle_co_traveler_search()`.

**How to change:**
- Tune the matching: currently exact FLIGHT + DATE match. Could relax to same flight number + ±1 day for flight-time tolerance.
- Add seat proximity: passengers within N seats of target = higher confidence

---

#### 20.3.4 FISA tools (`tools/agent_specific/fisa/`)

##### `fisa_search_tool.py` — Main agent search tool

**What it does:** searches `data/fisa_db.txt` for surveillance orders + intercepts. Inherits from `BaseAgentSearchTool`.

##### `fisa_query.py` — FISA order adapter

**What it does:** parses `FISA_ORDER_###` records, extracts structured fields (order number, judge, agency, target, selectors) plus unstructured narrative (justification, analyst notes).

**Output:** `{orders, all_contacts}` — all_contacts is the union of OTHER_PARTY phones across all intercepts.

**Used by:** `_lightweight_store()`. The `all_contacts` field feeds into `_run_secondary_search()`.

**How to change:**
- Tune which fields are "structured" (extracted to dict keys) vs "unstructured" (kept as raw text)
- Add support for new order types (e.g., Title VII §702 collections)

---

#### 20.3.5 CNE tools (`tools/agent_specific/cne/`)

##### `cne_search_tool.py` — Main agent search tool

**What it does:** searches `data/cne_db.txt` for device forensic extractions, emails, chat logs, browser history. Inherits from `BaseAgentSearchTool`.

**Output format note:** records containing literal "DELETED" or "RECOVERED" get auto-tagged `[DELETED/RECOVERED]` in the dossier section by `_format_cne()`. If your real forensics API has a deletion flag, format the output text to include those tokens so the dossier highlights them.

---

#### 20.3.6 OSINT, GEOINT, Intel Reports tools

All three follow the same pattern as the others: subclass `BaseAgentSearchTool`, set `name`/`description`/`db_path`, and the parent's text-search logic handles the rest. Replace `execute()` to call your real API (Section 19.3 has an OSINT multi-API example).

---

#### 20.3.7 SNA tools (`tools/agent_specific/sna/`)

##### `build_network.py` — Construct contact graph

**What it does:** takes the contact_intel (from `contact_search.py`) and builds a graph: target as central node, contacts as nodes, edges weighted by # of interactions or shared databases.

**Output:** `{nodes, edges, edge_count, node_count}` — adjacency-dict format, no external graph library.

**Used by:** SNA agent's plan + the API endpoint `/sna/network`.

##### `analyze_network.py` — Compute graph metrics

**What it does:** analyzes the built graph. No external deps (no NetworkX) — pure Python algorithms.

**Computes:**
- **Degree centrality** — how connected each node is
- **Clusters** — groups of densely connected nodes
- **Risk nodes** — nodes touching multiple high-risk databases (FISA + CNE = elevated)
- **Key connections** — bridges between clusters

**How to change:**
- Add betweenness centrality — implement BFS from each node and count shortest-path traversals
- Replace with NetworkX — change `analyze()` to use `nx.from_dict_of_dicts()` and `nx.betweenness_centrality()`. The output dict shape stays the same so downstream consumers don't break.

---

### 20.4 API Integration Tools (placeholders for real APIs)

These tools have stub implementations that return `{"status": "lookup_ready", "note": "Connect <vendor> API for live results"}`. They exist so the LLM tool-calling machinery has something to call. To make them functional:

| Tool | File | Real API to plug in |
|------|------|---------------------|
| `social_media_scan` | `api_integrations/osint_tools.py` | Pipl, Spokeo, Snusbase, IntelX, RocketReach |
| `whois_lookup` | `api_integrations/osint_tools.py` | WhoisXML API, DomainTools |
| `breach_check` | `api_integrations/osint_tools.py` | HaveIBeenPwned, DeHashed |
| `reverse_phone_lookup` | `api_integrations/reverse_lookup.py` | Twilio Lookup, NumVerify, IPQS |
| `reverse_email_lookup` | `api_integrations/reverse_lookup.py` | Hunter.io, Clearbit Reveal |
| `reverse_username_lookup` | `api_integrations/reverse_lookup.py` | Sherlock (open-source), Maigret |

**Pattern for each:** the file has a class subclassing `BaseTool`. Replace the body of `execute()` with your API call. Keep the return shape (`ToolResult.data` dict) consistent — downstream code expects `phone`, `email`, `username`, etc. as top-level keys.

---

### 20.5 Quick "Where do I edit X" Index for Tools

| I want to change... | Open this file |
|---------------------|----------------|
| What records SIGINT search returns | `tools/agent_specific/sigint/sigint_search_tool.py` (override `execute()`) |
| How CDR records are parsed | `tools/agent_specific/sigint/cdr_query.py` |
| What identifiers SIM-swap correlation finds | `tools/agent_specific/sigint/correlation_tool.py` |
| What records Travel search returns | `tools/agent_specific/travel/travel_search_tool.py` |
| What's a "co-traveler" | `tools/agent_specific/travel/co_traveler_search.py` |
| What records FISA returns | `tools/agent_specific/fisa/fisa_search_tool.py` (and `fisa_query.py` for parsing) |
| What records CNE returns | `tools/agent_specific/cne/cne_search_tool.py` |
| What records OSINT returns | `tools/agent_specific/osint/osint_search_tool.py` |
| What records GEOINT returns | `tools/agent_specific/geoint/geoint_search_tool.py` |
| What records Intel Reports returns | `tools/agent_specific/intel_reports/intel_reports_search_tool.py` |
| How dossier sections are formatted | `tools/shared/living_dossier.py` (`_format_<agent>()`) |
| Which agents/assessments run by default | `tools/shared/dossier_pipeline.py` (`DATA_AGENTS`, `ASSESSMENT_AGENTS`) |
| Cross-agent allowed agents (LEO scoping) | `agents/orchestrator.py` (`set_active_agent_scope`) |
| Iterative enrichment which agents handle which IDs | `tools/shared/enrichment_tracker.py` (`IDENTIFIER_AGENT_MAP`) |
| Confidence weights for cross-target links | `tools/shared/cross_target_linker.py` |
| Same-person merge thresholds | `tools/shared/entity_resolver.py` |
| Per-record confidence scoring | `tools/shared/identity_scorer.py` |
| Arabic name variations dictionary | `tools/shared/name_variation_generator.py` (`ARABIC_NAME_VARIATIONS`) |
| Default time ranges (CDR=1yr, FISA=5yr, etc.) | `tools/shared/datetime_tool.py` (`DEFAULT_RANGES`) |
| Translation/OCR API endpoint | `tools/shared/translation_tool.py` + `config.py` |
| Voter CSV path | `config.py` (`VOTER_CSV_PATH`) |
| Sub-agent batch size | `tools/shared/sub_agent_processor.py` (`records_per_worker` default) |
| Tool-calling engine max iterations | `tools/shared/tool_calling_engine.py` (`MAX_ITERATIONS`) |
| Plan executor large-prompt threshold | `tools/shared/plan_executor.py` (`LARGE_PROMPT_THRESHOLD`) |
| Assessment chunk size | `config.py` (`DOSSIER_CHUNK_SIZE`) |

---

### 20.6 Changing a Tool — The Universal Recipe

Every tool change follows the same 4 phases. Use this when the change is bigger than a one-line tweak:

**Phase 1 — Understand**
```bash
# Find every place the tool is registered
grep -rn "<tool_name>" backend/ --include="*.py"

# Find every plan that uses it
grep -l "<tool_name>" backend/plans/*.json

# Find downstream consumers of its output
grep -rn "<tool_name>" backend/tools/shared/ --include="*.py"
```

**Phase 2 — Verify the contract**
```python
# In a Python REPL inside V2/backend/
from tools.<path>.<file> import <ToolClass>
t = <ToolClass>(...)
print(t.get_parameter_schema())
result = t.execute(...)
print(type(result), result.success, type(result.data))
# Must be ToolResult, True/False, dict
```

**Phase 3 — Make the change** (edit `execute()`, or the formatter, or the constants)

**Phase 4 — Verify nothing downstream broke**
```bash
# Compile check
python3 -m py_compile <changed_file>

# Boot the orchestrator (catches import-time + registration errors)
python3 -c "from agents.orchestrator import Orchestrator; o = Orchestrator(); print('OK')"

# End-to-end check via the API
curl -X POST http://localhost:8000/single-agent \
     -H "Content-Type: application/json" \
     -d '{"agent": "<agent>", "query": {"phone": "+9647705551234"}}'

# Look at the dossier section that was written
cat data/dossiers/<target_id>/sections/0X_<agent>.md
```

If all four pass, your change is solid. If anything fails, the failure tells you exactly where to look (Phase 4 step 1 = compile error → syntax; step 2 = import error → missing import; step 3 = handler error → orchestrator wiring; step 4 = format error → Layer B compliance).

---

## 21. STRICT IDENTITY MATCHING (Issue 1 — the false-positive fix)

> **What this is:** the system that decides, for every record returned by every
> data source, whether the record is *verifiably about the target* or just a
> partial-name / loose-phone collision with someone else. Records that pass
> drive the assessments, executive summary, and narrative. Records that fail
> go to a **Data Dump** subsection — visible, retained, but never fed into the
> analysis.
>
> **Why it exists:** before this fix, a record matching only the first name
> (or only a 7-digit phone suffix, common in Iraqi/Saudi/Iranian dialing plans)
> could be tagged HIGH confidence and feed the assessment as if it were the
> target. Real-API testing showed records about *other* "Mohammed Ali"s bleeding
> into the report. This system stops that.

### 21.1 The decision tree — `tools/shared/strict_match.py`

Every record passes through `is_assessment_eligible(target_identifiers, record)`.
It returns `(eligible: bool, reason: str)`. The reason is stored alongside the
record so a reviewer can see *why* anything was set aside.

```
                      record + target identifiers
                              │
                              ▼
              ┌─────────────────────────────────┐
              │ Path A: strong-ID match?        │
              │ passport / IMEI / IMSI / email  │  ── yes ──►  ELIGIBLE
              │ (exact, case-insensitive)       │              reason: "strong-ID match: passport"
              └──────────────┬──────────────────┘
                             │ no
                             ▼
              ┌─────────────────────────────────┐
              │ Path B: every target name part  │
              │ present in record's name fields │  ── yes ──►  ELIGIBLE
              │ (variant-aware, prefix-stripped)│              reason: "all target name parts matched"
              └──────────────┬──────────────────┘
                             │ no (≥1 part missing)
                             ▼
              ┌─────────────────────────────────┐
              │ Path C: phone-owner resolution  │
              │ resolved owner name strict-      │  ── yes ──►  ELIGIBLE
              │ matches target?                 │              reason: "phone owner matches target"
              └──────────────┬──────────────────┘
                             │ no
                             ▼
              ┌─────────────────────────────────┐
              │ Path E: record's phone == one   │
              │ of target's KNOWN phones        │  ── yes ──►  ELIGIBLE
              │ (primary + SIM-swap-correlated) │              reason: "record phone matches target's known phone"
              └──────────────┬──────────────────┘
                             │ no
                             ▼
              ┌─────────────────────────────────┐
              │ Path D: phone exact + corrobora-│
              │ tor (date OR location match)    │  ── yes ──►  ELIGIBLE
              │ (off by default; opt-in via cfg)│
              └──────────────┬──────────────────┘
                             │ no
                             ▼
                        NOT ELIGIBLE → Data Dump
                        (reason carried through, e.g.
                         "record missing name parts: husseini")
```

**Why Path E exists** (the CDR/FISA/GEOINT fix): Call Detail Records, FISA
intercepts, and GEOINT pings are *phone-indexed by design* — `{caller, receiver,
IMEI, tower, timestamp}` — they never carry a name field. Path B would fail
every single one of them. Path E says: if the record's phone is the target's
phone (or any phone correlated to the target via SIM-swap detection in
`correlation_db`), the record IS about the target.

### 21.2 Arabic-prefix stripping — `_normalize_token()`

`"Al-Husseini"` == `"Alhusseini"` == `"Husseini"` == `"Al Husseini"` == `"El-Husseini"`.
Prefixes stripped: `al, el, ad, as, ar, ash, ath, az, abu, abdul, abdel, ibn, bin, bint`.
Guard: a prefix is only stripped if the remainder is ≥3 chars (so `"abu"` alone
stays `"abu"` and isn't reduced to `""`).

This combines with the existing `name_variation_generator` so that English
transliterations (`Mohammed` / `Mohammad` / `Muhammad` / `Mohamed`) also match.

### 21.3 Two-tier output: Confirmed Records vs Data Dump

When `living_dossier.write_agent_section()` writes an agent's section file
(`01_sigint.md`, `05_osint.md`, etc.), it now:

1. Splits the raw agent text into per-record blocks (`_split_raw_by_eligibility`)
2. Runs `is_assessment_eligible()` on each block
3. Formats the two sets separately:

```markdown
## SECTION 05: OPEN SOURCE INTELLIGENCE

### Summary
*(generated after collection)*

### Confirmed Records
... only records that passed strict-match ...

<!-- DATA_DUMP_BEGIN — excluded from assessments -->
### Data Dump — Loose Matches (Not Assessment-Eligible)
_These records matched the search query but did NOT satisfy the
identity-verification rule. They are retained for completeness and did
not contribute to any assessment in this report._

**Records filtered from this agent's response:**
... records that failed strict-match ...

**Reasons for exclusion:**
- record missing name parts: husseini
- record missing name parts: mohammed, ali

**Additional records from target store:**
| # | Source Record | Reason Excluded | Snippet |
| ...
<!-- DATA_DUMP_END -->
```

The `<!-- DATA_DUMP_BEGIN -->` / `<!-- DATA_DUMP_END -->` HTML-comment fences are
the key: `_strip_data_dump_blocks()` removes everything between them before the
text is handed to:
- the assessment scanner (`run_assessment`)
- the executive summary generator
- the narrative report generator

So the Data Dump is **visible to the analyst** in the section file and the
compiled dossier, but **never feeds the analysis**. No data loss; no
contamination.

### 21.4 Phone-owner resolution — `tools/shared/phone_owner_resolver.py`

When enrichment discovers a new phone, before it goes into the search queue,
`resolve_phone_owner(phone, ...)` tries to find the subscriber name:

1. SIGINT correlation table (`Config.CORRELATION_DB`)
2. TargetStore (search the `structured_data` column for the phone)
3. Voter registration DB
4. (hook) external reverse-phone-lookup API

Three outcomes in `_step_4_enrich`:

| Owner resolution | Queue action | Storage flag |
|------------------|--------------|--------------|
| Owner found, strict-matches target | queue normally | `assessment_eligible=1` (Path C) |
| Owner found, **mismatches** target | **skipped** — not queued | (no enrichment runs for this phone) |
| No owner found | queue (can't tell yet) | post-write strict-match decides |

This stops the failure mode where a co-traveler's or contact's phone gets
enriched and *their* records end up in the target's dossier.

### 21.5 Where each record's eligibility is persisted — `target_store.py`

The `records` table gained two columns (migration runs automatically on next
boot; existing rows default to `assessment_eligible=1`, `eligibility_reason=NULL`):

| Column | Meaning |
|--------|---------|
| `assessment_eligible` | `1` = drives assessments / narrative / exec summary. `0` = Data Dump only. |
| `eligibility_reason` | Human-readable reason recorded by `is_assessment_eligible()` at storage time. |

New query helpers:
- `get_confirmed_records(target_id, source_agent=None)` → `assessment_eligible=1` rows
- `get_data_dump_records(target_id, source_agent=None)` → `assessment_eligible=0` rows

### 21.6 Confidence tiers — `confidence_tier()`

Each record gets a tier (separate from eligible/not):

| Tier | When |
|------|------|
| HIGH | name match + at least one strong ID; OR name match + (phone OR country match) |
| MEDIUM | name match only (no strong ID); OR eligible via strong-ID alone |
| LOW | eligible via Path A/C/D/E but no full name match in the record itself |
| REJECT | not eligible (data dump) |

The strict tier overwrites the legacy `confidence_level` from `identity_scorer`.

### 21.7 How to tune strict-match

| Want to... | Where |
|-----------|-------|
| Turn the whole thing off (revert to legacy behavior) | `.env`: `STRICT_NAME_MATCH=false` |
| Change phone-match minimum digits | `.env`: `PHONE_MATCH_MIN_DIGITS=10` (lower it if your region uses shorter numbers) |
| Allow phone-only records WITHOUT a corroborator (Path D relaxed) | `.env`: `PHONE_ALLOW_WITHOUT_CORROBORATOR=true` |
| Stop stripping Al-/El- prefixes | `.env`: `NAME_STRIP_PREFIXES=false` (NOT recommended) |
| Stop enrichment from gating on owner-name | `.env`: `ENRICHMENT_REQUIRE_OWNER_NAME_MATCH=false` |
| Add a new strong-ID type | `strict_match.py` — add to `PASSPORT_KEYS` / `IMEI_KEYS` / etc., add a branch in `strong_id_match()` |
| Add a new name field the matcher should read | `strict_match.py` — add to `NAME_FIELD_KEYS` |
| Add a new Path | `strict_match.py` — add a branch in `is_assessment_eligible()` between the existing paths |

### 21.8 How to test it in isolation

```bash
cd V2/backend
python3 tests/test_strict_match.py     # 30 cases — decision tree, prefix-strip, tiers
python3 tests/test_e2e_noise.py        # 19 cases — inject partial-name records, verify they dump
python3 tests/test_bug_fixes.py        # 38 cases — R1-R9
python3 tests/test_round3_fixes.py     # 27 cases — G1-G4, §3.x
```

---

## 22. LE REPORT TEMPLATES (Issue 2 — the report-style fix)

> **What this is:** a JSON-template system that renders the final report from
> *fixed law-enforcement boilerplate* with `{{slots}}` filled by the LLM —
> instead of generating freeform LLM prose every time. Same structure every run.
> Negative findings stated explicitly. Citations to real record IDs only.
>
> **Why it exists:** the old `narrative_report.md` was pure LLM prose ("appears
> to…", "may suggest…", "we believe…"). LE reports follow rigid conventions —
> third-person passive, explicit negative findings, fixed section order, citations
> that trace to source records, classification banners. Two runs of the same
> target produced two stylistically different reports. Templates fix that.

### 22.1 How it works — the data flow

```
                    ┌──────────────────────────────────────────┐
                    │ DossierPipeline._step_7_final()           │
                    │                                          │
                    │  if Config.USE_TEMPLATE_REPORT:           │
                    │      _render_templated_report()  ◄── NEW │
                    │  else:                                    │
                    │      generate_narrative_report()          │
                    │      (legacy LLM-prose path — fallback)   │
                    └────────────┬─────────────────────────────┘
                                 │ (template path)
                                 ▼
                    ┌──────────────────────────────────────────┐
                    │ report_context_builder.build_context()   │
                    │  1. target_store.get_target() → target.*  │
                    │  2. parse assess_*.md → assessment.*      │
                    │  3. target_store record counts → stats.*  │
                    │  4. LLM (temp 0, ≤800 chars, hedge-scrub) │
                    │     → narrative summary_800 slots ONLY    │
                    │  → flat context dict                      │
                    └────────────┬─────────────────────────────┘
                                 ▼
                    ┌──────────────────────────────────────────┐
                    │ TemplateFiller.render(template, context) │
                    │  ZERO LLM. Pure substitution +           │
                    │  conditional evaluation.                 │
                    │  Writes sections/00_narrative_report.md  │
                    └──────────────────────────────────────────┘
```

The LLM does **only** fact-summarization for the `summary_800` slots, with a
hedge-scrubber that retries (and falls back to a bullet-list extract) if it
produces banned phrases. The prose wrapping those facts is fixed in the template.

### 22.2 The 4 templates — `report_templates/`

| File | Used for | Sections |
|------|----------|----------|
| `le_targeting_v1.json` | Full Targeting workflow | I. Subject Identification → X. Disposition + Appendices A/B/C |
| `le_trace_v1.json` | Trace workflow | Subject ID, Trace Summary, Name/Phone/Other identifier results, Appendices A/B |
| `le_leo_v1.json` | LEO workflow (selected data agents) | Subject ID, Data Summary, Communications/Travel/Digital/OSINT/Intel/GEOINT sections (conditional), Appendices A/B |
| `le_single_agent_v1.json` | Direct-search single-agent | Subject/Identifiers, Result Summary, Findings, Appendix |

The pipeline picks the template based on `workflow_type`:
`targeting` → `Config.REPORT_TEMPLATE_DEFAULT` (le_targeting_v1),
`trace` → `Config.REPORT_TEMPLATE_TRACE`,
`leo` → `Config.REPORT_TEMPLATE_LEO`,
`single_agent` → `Config.REPORT_TEMPLATE_SINGLE`.

### 22.3 The template JSON schema

```jsonc
{
  "template_id": "le_targeting_v1",       // must match the .json filename
  "version": "1.0",
  "display_name": "Law-Enforcement Targeting Report (Standard)",
  "description": "...",
  "classification": "LAW ENFORCEMENT SENSITIVE // FOR OFFICIAL USE ONLY",
  "header_verbiage": "...full boilerplate paragraph, can use {{slots}}...",
  "footer_verbiage": "...closing boilerplate...",
  "required_facts": ["target.full_name"],  // hard-fail render if any missing

  "sections": [
    {
      "id": "subject_identification",
      "title": "I. SUBJECT IDENTIFICATION",
      "verbiage": "The subject of this report is identified as {{target.full_name}} (native name: {{target.native_name_or_unknown}}), date of birth {{target.dob_or_unknown}}, ...",
      "slots": {
        "target.full_name":              {"source": "target.full_name", "required": true},
        "target.native_name_or_unknown": {"source": "target.native_name", "fallback": "not provided"},
        "target.dob_or_unknown":         {"source": "target.dob", "fallback": "unknown"}
      },
      "skip_if_empty": false
    },
    {
      "id": "information_access",
      "title": "III. INFORMATION ACCESS",
      "conditional": {
        "if": "assessment.access.findings_count > 0",
        "then": {
          "verbiage": "Investigation has developed information indicating that the subject {{access.verb}} access to {{access.targets}}. Specifically, {{access.summary_800}} ...",
          "slots": {
            "access.verb":       {"derive": "assessment.access.findings_count > 2 ? has demonstrated direct : appears to have"},
            "access.targets":    {"source": "assessment.access.targets", "join": ", ", "fallback": "undetermined subject matter"},
            "access.summary_800":{"source": "assessment.access.summary_800", "max_chars": 800, "fallback": ""}
          }
        },
        "else": {
          "verbiage": "No information of investigative significance was developed regarding the subject's information access at this time. Review of {{stats.records_confirmed}} verified records across {{stats.sources_count}} investigative data sources disclosed no indicators in this area.\n\nOverall access rating: INSUFFICIENT DATA.",
          "slots": {
            "stats.records_confirmed": {"source": "stats.records_confirmed", "fallback": "0"},
            "stats.sources_count":     {"source": "stats.sources_count", "fallback": "0"}
          }
        }
      },
      "skip_if_empty": false
    }
  ]
}
```

**Slot definition fields:**

| Field | Purpose |
|-------|---------|
| `source` | Dotted path into the context dict, e.g. `assessment.access.summary_800` |
| `fallback` | Text used when the source is empty/missing (this is what makes negative findings explicit) |
| `max_chars` | Truncate the value (backs up to the last sentence boundary — won't fuse mid-word with the next clause) |
| `max_items` | For list values — keep the first N, append "(and M more)" |
| `join` | Separator when the value is a list |
| `required` | If true and the value is empty → render aborts with an error |
| `derive` | One-line ternary: `<path> > <num> ? '<true text>' : '<false text>'` |

**Conditional operators in `if`:** `> < >= <= == != contains has_any has_none`
Examples: `assessment.access.findings_count > 0`, `target.aliases has_any`,
`stats.records_confirmed == 0`.

**`skip_if_empty`:** if `true`, the section is dropped entirely when every slot
resolves to its fallback. Default `false` — LE reports want explicit "no
information developed" sections.

### 22.4 Context dict — what slots are available

`report_context_builder.build_context()` populates these namespaces:

| Namespace | Examples | Source |
|-----------|----------|--------|
| `target.*` | `full_name`, `native_name`, `phone`, `passport`, `email`, `imei`, `dob`, `nationality`, `aliases` | target_store.get_target() — `native_name` is synthesized from name parts if not stored |
| `assessment.<type>.*` | `findings_count`, `summary_800`, `record_ids`, `rating`, `principal_category`, `predictability`, `officer_safety_note` | parsed from `assess_<type>.md` files |
| `stats.*` | `records_total`, `records_confirmed`, `records_loose`, `sources_count`, `sources_list` | target_store record counts |
| `report.*` | `id`, `date`, `prepared_by`, `classification` | computed at render time |
| `appendix.*` | `sources_table`, `identifiers_table`, `loose_matches_table` | built from target_store |
| `investigation.*` | `start_date`, `end_date` | from target's created_at |
| `workflow.*` (LEO only) | `name`, `selected_agents` | from the LEO config |
| `data.*` (LEO / single-agent) | `<agent>.summary_800`, `<agent>.record_ids`, `<agent>.findings_count` | parsed from section files |
| `disposition.*` (targeting) | `paragraph` | rule-based from assessment ratings (or user-overridable) |

### 22.5 The rating derivation (G1 fix)

Assessment ratings come from one of (in order):
1. An explicit `### Score: HIGH` line in the `assess_<type>.md` file
2. `**Score**: ELEVATED`
3. `Overall <X> rating: CRITICAL`
4. `Rating: MODERATE` / `**Rating**: HIGH`
5. **Derived** from `findings_count` + risk keywords in the body:
   - ≥8 findings, or ≥4 findings + risk keyword → HIGH
   - ≥4 findings, or ≥1 finding + risk keyword → ELEVATED
   - ≥1 finding → MODERATE
   - 0 findings → INSUFFICIENT DATA

Risk keywords: `high, elevated, critical, severe, imminent, threat, fraud,
illicit, kickback, laundering, smuggl, weapon, explos, terrorist, extremist,
IRGC, IRGC-QF, hostile`.

This is why the report no longer shows "INSUFFICIENT DATA" everywhere while the
detailed assessment below says HIGH — the parser now picks up the real value.

### 22.6 Citation discipline (G4 fix)

Two layers prevent the LLM from inventing record IDs:

1. **Prompt rule** — every assessment scan prompt gets `_CITATION_RULES`
   appended: "Cite ONLY real record IDs matching these patterns: `PNR_<digits>`,
   `CDR_<digits>`, `FISA_(ORDER|IP|INT)_<digits>`, `CNE_<ALPHA>_<digits>`,
   `IR-<YYYY>-<digits>`, `OSINT_<XX>_<digits>`, `GEOINT_<X>_<digits>`. If you
   cannot find a real ID, write '(no source record id available)'. NEVER cite
   bullet numbers or list indices."

2. **Post-validation** — `_strip_fabricated_record_ids()` scans the assessment
   markdown for "Record IDs: …" lists, keeps only IDs matching the real-pattern
   regex (`_VALID_RECORD_ID_RE`), and replaces invented citations
   (like `Record IDs: 28, 29, 30` — those are bullet numbers) with
   `(no source record id available)`.

### 22.7 Hedge scrubbing (R3 fix)

Both the template's `summary_800` slot extraction AND the per-section LLM
summaries (`summarize_section`) AND the chunked-summary REDUCE step now run
through a hedge scrubber:

- Banned phrases (15): `i think, i believe, we believe, in my opinion, perhaps,
  may suggest, appears to, seems to, could be, might be, possibly, potentially,
  it is likely, likely that, arguably`
- If the LLM output contains a banned phrase → retry once with a stricter prompt
  ("third-person passive only, no hedging words, state facts only")
- If still hedged → fall back to a bullet-list extract from the raw findings

### 22.8 HOW TO EDIT A TEMPLATE'S VERBIAGE (dumb-proof)

**Scenario: you want the Subject Identification section to read differently.**

1. Open `V2/backend/report_templates/le_targeting_v1.json`
2. Find the section with `"id": "subject_identification"`
3. Edit the `"verbiage"` string. Keep the `{{slot.name}}` placeholders — those
   get substituted. Change the prose around them.
4. If you add a NEW placeholder `{{target.something}}`, also add it to the
   section's `"slots"` object:
   ```json
   "slots": {
     "target.something": {"source": "target.something", "fallback": "not provided"}
   }
   ```
   The `source` path must exist in the context dict (see §22.4). If it's a
   target field that isn't there yet, add it in `report_context_builder.py`
   `_build_identifiers_dict()`.
5. No code change, no restart needed for the JSON edit alone — templates are
   loaded fresh per render. (You DO need a restart if you changed
   `report_context_builder.py`.)
6. Test the render without re-running the pipeline:
   ```bash
   curl -X POST "http://localhost:8000/targets/<target_id>/report/preview?template_id=le_targeting_v1"
   ```
   This returns the rendered markdown so you can iterate on the wording fast.

### 22.9 HOW TO ADD A NEW SECTION TO A TEMPLATE

1. Open the template JSON
2. Add a new object to the `"sections"` array, in the position you want it:
   ```json
   {
     "id": "financial_exposure",
     "title": "XI. FINANCIAL EXPOSURE",
     "conditional": {
       "if": "assessment.motivation.findings_count > 0",
       "then": {
         "verbiage": "Financial indicators developed: {{fin.summary}} (Reference records: {{fin.record_ids}}).",
         "slots": {
           "fin.summary":    {"source": "assessment.motivation.summary_800", "max_chars": 600},
           "fin.record_ids": {"source": "assessment.motivation.record_ids", "join": ", ", "max_items": 10, "fallback": "(no record IDs available)"}
         }
       },
       "else": {"verbiage": "No financial exposure indicators were developed at this time."}
     },
     "skip_if_empty": false
   }
   ```
3. That's it. The renderer iterates `sections` in order.

### 22.10 HOW TO ADD A WHOLE NEW TEMPLATE (e.g., for a new agency format)

1. Copy an existing template:
   ```bash
   cp V2/backend/report_templates/le_targeting_v1.json \
      V2/backend/report_templates/agency_x_v1.json
   ```
2. Edit `"template_id"` to match the filename: `"agency_x_v1"`.
3. Rewrite the `header_verbiage`, `footer_verbiage`, `classification`, and the
   section verbiage to match the target agency's conventions.
4. To make it the default for a workflow type, set the env var:
   ```bash
   export REPORT_TEMPLATE_DEFAULT=agency_x_v1   # for targeting workflows
   ```
   Or pass `report_template=agency_x_v1` if you wire that into the workflow
   creation endpoint.
5. Verify it loads:
   ```bash
   cd V2/backend
   python3 -c "
   import sys; sys.path.insert(0, '.')
   from tools.shared.template_filler import TemplateFiller
   from config import Config
   tf = TemplateFiller(Config.REPORT_TEMPLATES_DIR)
   print([t['template_id'] for t in tf.list_templates()])
   "
   # Should include 'agency_x_v1'
   ```

### 22.11 HOW TO TURN TEMPLATES ON / OFF

```bash
# Templates OFF (default — uses the legacy LLM-prose narrative)
unset USE_TEMPLATE_REPORT          # or set it to false

# Templates ON
export USE_TEMPLATE_REPORT=true

# Templates ON, but fall back to LLM-prose if a render fails (default behavior)
export USE_TEMPLATE_REPORT=true
export REPORT_TEMPLATE_FALLBACK_TO_LLM_PROSE=true

# Templates ON, hard-fail on render error (production: surface the error)
export USE_TEMPLATE_REPORT=true
export REPORT_TEMPLATE_FALLBACK_TO_LLM_PROSE=false
```

Restart the server after changing these.

### 22.12 The API endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /report/templates` | List all templates (id, version, display_name, description) |
| `GET /report/templates/{template_id}` | Return the raw JSON of one template (inspect slots/verbiage) |
| `POST /targets/{target_id}/report/preview?template_id=X&workflow_type=Y` | Dry-run render — iterate on verbiage without re-running the pipeline. Returns `{target_id, template_id, chars, markdown}`. |

### 22.13 How to test the template system

```bash
cd V2/backend
python3 tests/test_template_filler.py   # 22 cases — load, render, conditionals, determinism, missing-fact
```

---

## 23. CONFIGURATION — NEW FLAGS (added with Issues 1 & 2)

All env-overridable. Defaults shown. These are appended to the existing
`Config` class in `config.py` (see §12 for the original flags).

```python
# ── Issue 1: Strict name verification ──────────────────────────
STRICT_NAME_MATCH = True                       # master switch — false reverts to legacy
PHONE_MATCH_MIN_DIGITS = 10                     # was effectively 7; raise/lower per region
ENRICHMENT_REQUIRE_OWNER_NAME_MATCH = True      # gate enrichment phones on resolved owner
NAME_STRIP_PREFIXES = True                      # strip Al-/El-/Abu- before name compare
PHONE_ALLOW_WITHOUT_CORROBORATOR = False        # Path D off by default (strictest)

# ── Issue 2: Report templates ──────────────────────────────────
USE_TEMPLATE_REPORT = False                     # OPT-IN — false keeps legacy narrative
REPORT_TEMPLATES_DIR = "<BASE_DIR>/report_templates"
REPORT_TEMPLATE_DEFAULT = "le_targeting_v1"     # for workflow_type=targeting
REPORT_TEMPLATE_TRACE   = "le_trace_v1"         # for workflow_type=trace
REPORT_TEMPLATE_LEO     = "le_leo_v1"           # for workflow_type=leo
REPORT_TEMPLATE_SINGLE  = "le_single_agent_v1"  # for workflow_type=single_agent
REPORT_TEMPLATE_FALLBACK_TO_LLM_PROSE = True    # if a render fails, use the legacy path

# ── Narrative-slot extraction (constrained LLM) ────────────────
NARRATIVE_SLOT_MAX_CHARS = 800                  # per summary_800 slot (≈130 words)
NARRATIVE_SLOT_TEMPERATURE = 0.0
NARRATIVE_SLOT_BANNED_PHRASES = [               # 15 hedging phrases that trigger retry
    "i think", "i believe", "we believe", "in my opinion",
    "perhaps", "may suggest", "appears to", "seems to",
    "could be", "might be", "possibly", "potentially",
    "it is likely", "likely that", "arguably",
]
```

To set the summary length to ≈300 words per slot: `export NARRATIVE_SLOT_MAX_CHARS=1500`.
The LLM only fills facts — it won't pad to the limit when there's less to say.

New directory the platform now expects: `V2/backend/report_templates/` (holds the
4 template JSONs).

---

## 24. CHANGELOG — every fix, every detail

This section is the canonical record of what changed and why. Three rounds.

### Round 1 — Phase A: 4 quick patches (the leak fixes)

| Leak | File | What changed | Why |
|------|------|--------------|-----|
| **L1** | `identity_scorer.py` `_name_similarity()` | Removed the first-name-only `similarity = max(similarity, 0.5)` boost | Floored unrelated "Mohammed Ali" records at 0.5 → HIGH confidence. Name verification moved to strict_match.py. |
| **L2** | `identity_scorer.py` `score_record()` Phone block | `endswith(r_norm[-7:])` → require ≥10 matching digits | 7-digit suffix collisions are common in Iraqi/Saudi/Iranian dialing plans |
| **L3** | `identity_scorer.py` `score_record()` confidence block | Strong-ID auto-HIGH only for passport/email/IMEI/IMSI, OR phone+name | Phone alone (especially a loose tail match) isn't enough evidence for HIGH |
| **L4** | `contact_search.py` `_search_database()` | `name.lower() in record.lower()` → word-boundary regex; multi-token names require all tokens | "Ali" was matching inside "nationality", "Italian", "particularly" |

### Round 2 — Phases B, C, D: the strict-match system + LE templates

**Phase B — strict identity matching (NEW infrastructure):**
- **NEW** `tools/shared/strict_match.py` — `is_assessment_eligible()`, `confidence_tier()`, `target_name_fully_matched()`, `strong_id_match()`, `phone_exact_match()`, `phone_in_record_matches_known_phones()`, `extract_name_parts()`, `_normalize_token()` (with Arabic-prefix stripping)
- **NEW** `tools/shared/phone_owner_resolver.py` — `resolve_phone_owner()`
- `target_store.py` — migration adds `assessment_eligible` + `eligibility_reason` columns; `store_record()` persists them; new helpers `get_data_dump_records()`, `get_confirmed_records()`
- `dossier_pipeline.py` `_lightweight_store()` — calls `is_assessment_eligible()` on every record, stores the flag + tier
- `dossier_pipeline.py` `_step_4_enrich()` — phone-owner resolution at end of enrichment loop
- `living_dossier.py` — two-tier formatter split (`write_agent_section` runs `_split_raw_by_eligibility` before formatting), `_strip_data_dump_blocks()`, `_build_data_dump_md()`, `_get_data_sections_text(strip_data_dump=True)`

**Phase C — LE report templates (NEW infrastructure):**
- **NEW** `tools/shared/template_filler.py` — `TemplateFiller` class, zero-LLM renderer with conditionals/slots/derive/skip_if_empty
- **NEW** `report_templates/` directory + 4 templates: `le_targeting_v1.json`, `le_trace_v1.json`, `le_leo_v1.json`, `le_single_agent_v1.json`

**Phase D — context builder + pipeline wire-up + API:**
- **NEW** `tools/shared/report_context_builder.py` — `build_context()` + helpers (`_build_identifiers_dict`, `_build_assessment_dict`, `_build_stats_dict`, `_build_appendix_dict`, `_parse_record_ids`, `_llm_narrative_summary` with hedge scrubber, `_bullet_extract_fallback`)
- `dossier_pipeline.py` `_step_7_final()` — branches on `Config.USE_TEMPLATE_REPORT`; new `_render_templated_report()` helper
- `api_server.py` — 3 new endpoints: `/report/templates`, `/report/templates/{id}`, `/targets/{id}/report/preview`
- `config.py` — 17 new flags (see §23)

**Round-2 post-test polish (6 bugs found in the first real-data test run):**
| ID | File | Fix |
|----|------|-----|
| R1 | `report_context_builder.py` `_build_assessment_dict()` | Rating derived from `findings_count` + risk keywords (the regex-parse for `Overall X rating:` never matched the actual assessment output) |
| R2 | `report_context_builder.py` `_build_identifiers_dict()` | `native_name` synthesized from name parts when not stored (was showing "not provided") |
| R3 | `living_dossier.py` `_single_summary` / `_chunked_summary` | Hedge scrubber + retry + bullet-list fallback applied to per-section LLM summaries |
| R6 | `template_filler.py` `_resolve_slot()` | Truncation backs up to the last sentence terminator (was fusing mid-word with the next clause: "...complicate Reference records:") |
| R7 | `report_context_builder.py` `_parse_record_ids()` | Bare-integer record IDs ("Record IDs: 3, 4, 10") captured as `#3, #4, #10` |
| R9 | `living_dossier.py` `write_agent_section()` | **CRITICAL** — section files used to render ALL raw records under `### Confirmed Records` regardless of eligibility. Now `_split_raw_by_eligibility()` runs strict-match on the raw text BEFORE the formatter; Confirmed and Data Dump are formatted separately. |

### Round 3 — 7 fixes from the second deep-scan of the run log

| ID | Severity | File(s) | Fix |
|----|----------|---------|-----|
| **G1** | critical | `report_context_builder.py` `_build_assessment_dict()` | Rating parser now matches `### Score: HIGH`, `**Score**: ELEVATED`, `Overall X rating:`, `Rating:`, `**Rating**:` — against a whitelist of known LE rating values. Falls back to derivation (R1) if no explicit line. Fixes the "INSUFFICIENT DATA everywhere while the assessment says HIGH" contradiction. |
| **G2** | high | `strict_match.py` + `dossier_pipeline.py` + `living_dossier.py` | New **Path E**: record's phone matches one of target's known phones (primary + SIM-swap-correlated). `dossier_pipeline._build_known_phones()` walks the correlation table at Step 2 to harvest the correlated set. `living_dossier._extract_record_fields_for_strict_match()` now handles markdown-decorated keys (`- **name**: Foo`) and rejects YYYY-MM-DD dates as phones. Fixes CDR/FISA/GEOINT phone-only records (51 of 68 records) being wrongly dumped. |
| **G3** | high | `dossier_pipeline.py` `_step_4_enrich()` | Phone-owner resolution gate at **queue-time** — phones whose resolved owner doesn't strict-match the target are skipped from enrichment entirely. Fixes wrong-person CDRs (a co-traveler's calls) being written into the target's SIGINT section. |
| **G4** | medium | `living_dossier.py` | `_CITATION_RULES` appended to every assessment prompt + `_strip_fabricated_record_ids()` post-validation that keeps only real-pattern IDs and replaces invented citations ("Record IDs: 28, 29, 30" — those are bullet numbers) with "(no source record id available)". |
| **§3.1** | medium | (fixed by R9 + G2) | "Omar Al-Husseini" (different person, only family name shared) was in Confirmed Records. Now G2's markdown-name extraction picks up his `**name**: Omar Al-Husseini`; Path B fails (target's "mohammed", "ali" not present); Omar correctly goes to Data Dump. |
| **§3.3** | low | `living_dossier.py` `_resolve_contact_names()` | Expanded `skip_names` + new `BAD_TOKENS` vocabulary (a candidate is garbage if every token is a label word) + `name_prefix_patterns` get 3× weight so real `NAME: Foo` beats proximity-based "Source Record" / "Electronic Surveillance" / "Data Science" / "Network Analysis". |
| **§3.4/3.5** | low | `living_dossier.py` `append_to_section()` | Enrichment "Additional Data" blocks now run the same strict-match split. Confirmed → emitted inline as `#### Additional Data — Confirmed`. Loose-match → wrapped in `<!-- DATA_DUMP_BEGIN -->`…`<!-- DATA_DUMP_END -->` fences as `#### Additional Data — Data Dump` so the assessment scanner strips them. |

### Test coverage

| Test file | Cases | Covers |
|-----------|-------|--------|
| `tests/test_strict_match.py` | 30 | decision tree (paths A-E), prefix-stripping, confidence tiers, name extraction |
| `tests/test_template_filler.py` | 22 | template load, render, conditionals, determinism, required-fact validation |
| `tests/test_e2e_noise.py` | 19 | inject partial-name CDR/OSINT records, verify they go to Data Dump (3 layers: strict_match → target_store → section file) |
| `tests/test_bug_fixes.py` | 38 | R1, R2, R3, R6, R7, R9 |
| `tests/test_round3_fixes.py` | 27 | G1, G2, G4, §3.1, §3.3, §3.4/3.5 |
| **Total** | **136** | |

Run all: `cd V2/backend && for t in test_strict_match test_template_filler test_e2e_noise test_bug_fixes test_round3_fixes; do python3 tests/$t.py; done`

### Files changed (full inventory)

**NEW (12 files + 1 directory):**
- `tools/shared/strict_match.py`
- `tools/shared/phone_owner_resolver.py`
- `tools/shared/template_filler.py`
- `tools/shared/report_context_builder.py`
- `report_templates/` (directory) + `le_targeting_v1.json`, `le_trace_v1.json`, `le_leo_v1.json`, `le_single_agent_v1.json`
- `tests/__init__.py`, `tests/test_strict_match.py`, `tests/test_template_filler.py`, `tests/test_e2e_noise.py`, `tests/test_bug_fixes.py`, `tests/test_round3_fixes.py`

**MODIFIED (7 files):**
- `config.py` — 17 new flags
- `api_server.py` — 3 new endpoints
- `tools/shared/identity_scorer.py` — L1, L2, L3
- `tools/shared/contact_search.py` — L4
- `tools/shared/target_store.py` — migration + 2 helpers
- `tools/shared/dossier_pipeline.py` — strict-match integration, phone-owner resolution, `_build_known_phones`, enrichment gate, `_step_7_final` template branch
- `tools/shared/living_dossier.py` — two-tier formatter split, hedge scrubber, markdown-name extraction, citation rules + post-validation, contact-name garbage rejection, `append_to_section` eligibility split

### Outstanding (deferred — small)

- Frontend dropdown for "Report Style: LE Standard / Legacy LLM Prose" — backend is fully ready (`USE_TEMPLATE_REPORT` flag + `/report/templates` endpoint); the UI control is a ~30-min follow-up.

---

## 25. TROUBLESHOOTING & DIAGNOSTICS — "if you see X, look at Y"

> This is the symptom → file → knob lookup. When the report does something
> wrong, start here. Almost every diagnosis starts with one question:
> **what does `eligibility_reason` say on the record?** That string tells you
> which path in `strict_match.is_assessment_eligible()` made the decision —
> which tells you exactly which code to look at.

### 25.1 The first move — always check `eligibility_reason`

Every record stored gets an `eligibility_reason` string. It's the breadcrumb.

| `eligibility_reason` text | Which path decided | What it means |
|---------------------------|--------------------|---------------|
| `strong-ID match: passport` (or imei/imsi/email) | Path A | A unique identifier matched — record IS the target |
| `all target name parts matched (variant-aware)` | Path B | Every name part of the target appears in the record's name field |
| `phone owner matches target (...)` | Path C | Phone-owner resolution found a name that matched the target |
| `record phone <X> matches target's known phone <Y>` | Path E | Record's phone is the target's primary or a SIM-swap-correlated phone |
| `phone exact match + corroborating evidence` | Path D | Phone matched AND date/location also matched |
| `record missing name parts: husseini` (etc.) | REJECT (Path B failed) | The named parts in parentheses are NOT in the record → goes to Data Dump |
| `no name in record` | REJECT | The record had no name field at all AND no phone/strong-ID match |
| `strict_match error (defaulted eligible): ...` | FAIL-OPEN | strict_match itself threw — record was kept (no data loss) but check the error |

### 25.2 SQLite diagnostic queries — run these against `data/target_store.db`

```bash
# Get the latest target_id
TID=$(ls -t V2/backend/data/dossiers/ | head -1)
echo "Latest target: $TID"

# Confirmed vs Data-Dump counts per agent
sqlite3 V2/backend/data/target_store.db "
SELECT source_agent,
       SUM(CASE WHEN COALESCE(assessment_eligible,1)=1 THEN 1 ELSE 0 END) AS confirmed,
       SUM(CASE WHEN COALESCE(assessment_eligible,1)=0 THEN 1 ELSE 0 END) AS data_dump
FROM records WHERE target_id='$TID' GROUP BY source_agent;
"

# Top reasons for exclusion (data dump)
sqlite3 V2/backend/data/target_store.db "
SELECT eligibility_reason, COUNT(*) AS n
FROM records WHERE target_id='$TID' AND COALESCE(assessment_eligible,1)=0
GROUP BY eligibility_reason ORDER BY n DESC LIMIT 20;
"

# Reasons records WERE included (confirmed) — to spot a leak (a path letting in junk)
sqlite3 V2/backend/data/target_store.db "
SELECT eligibility_reason, COUNT(*) AS n
FROM records WHERE target_id='$TID' AND COALESCE(assessment_eligible,1)=1
GROUP BY eligibility_reason ORDER BY n DESC LIMIT 20;
"

# Inspect one specific record's structured data + reason
sqlite3 V2/backend/data/target_store.db "
SELECT source_agent, source_record_id, assessment_eligible, eligibility_reason,
       substr(structured_data, 1, 300)
FROM records WHERE target_id='$TID' AND source_record_id='CDR_00003';
"

# Did the migration run? (these columns must exist)
sqlite3 V2/backend/data/target_store.db "PRAGMA table_info(records);" | grep -E "assessment_eligible|eligibility_reason"
# If empty → migration didn't run. See §25.13.
```

### 25.3 SYMPTOM → FILE → KNOB master table

| Symptom | Look at this file / function | The knob / fix |
|---------|------------------------------|----------------|
| **Report wording is wrong** (the boilerplate prose, not the facts) | `report_templates/<template>.json` → the section's `verbiage` string | Edit the verbiage. Keep `{{slot}}` placeholders. See §22.8 |
| Report wording wrong AND you added a new `{{slot}}` | `report_templates/<template>.json` `slots` object + `report_context_builder.py` `_build_identifiers_dict` (if it's a `target.*` slot) | Add the slot definition; ensure `source` path exists in the context dict (§22.4) |
| **Still seeing FALSE POSITIVES** (wrong-person records in Confirmed / assessments) | First: the `eligibility_reason` (§25.1) tells you which path let it in. Then the relevant function in `strict_match.py` | See §25.4 step-by-step |
| **Seeing FALSE NEGATIVES** (the target's own records wrongly in Data Dump) | First: `eligibility_reason` says why it was rejected. Then `living_dossier.py` `_extract_record_fields_for_strict_match` (extraction) or `strict_match.py` (the path that should have caught it) | See §25.5 step-by-step |
| **Citation hallucination** (LLM cites `Record IDs: 28, 29, 30` — bullet numbers, not real) | `living_dossier.py` `_CITATION_RULES` (the prompt) + `_strip_fabricated_record_ids` + `_VALID_RECORD_ID_RE` (the regex of valid patterns) | See §25.6 |
| **Rating shows "INSUFFICIENT DATA"** while the assessment body clearly has findings | `report_context_builder.py` `_build_assessment_dict` — the rating-extraction loop + the findings_count derivation | See §25.7 |
| **Hedging in the report** ("appears to", "may suggest", "we believe") | `config.py` `NARRATIVE_SLOT_BANNED_PHRASES` + `living_dossier.py` `_HEDGE_PHRASES` + `report_context_builder.py` `_llm_narrative_summary` | See §25.8 |
| **Wrong-person data in a section's "Additional Data" block** | `dossier_pipeline.py` `_step_4_enrich` (the queue-time owner gate) + `living_dossier.py` `append_to_section` (the eligibility split) + `phone_owner_resolver.py` | See §25.9 |
| **Contact-network names are garbage** ("Source Record", "Electronic Surveillance") | `living_dossier.py` `_resolve_contact_names` — `BAD_TOKENS`, `skip_names`, `name_prefix_patterns` | See §25.10 |
| **Data Dump section is empty when it shouldn't be** | `living_dossier.py` `_build_data_dump_md` (reads `target_store.get_data_dump_records`) — OR the migration didn't run | See §25.11 + §25.13 |
| **Assessment still references records that should be in Data Dump** | `living_dossier.py` `_strip_data_dump_blocks` (the fence-stripping regex) + `_get_data_sections_text(strip_data_dump=True)` | See §25.11 |
| **Crash on boot: `no attribute 'get_data_dump_records'`** | `target_store.py` — the new helpers weren't applied | Re-apply the `target_store.py` changes (§24, B.5 in the patch) |
| **Crash: `ModuleNotFoundError: tools.shared.strict_match`** | The new module wasn't created | Create `tools/shared/strict_match.py` (bundle 03) |
| **Templates not loading / `/report/templates` returns empty** | `report_templates/` directory missing, or `Config.REPORT_TEMPLATES_DIR` wrong | `mkdir -p V2/backend/report_templates` and put the 4 JSONs there |
| **Render fails: `Cannot render template ... missing required facts: ['target.full_name']`** | The target has no name parts | Make sure the workflow was created with `first_name` etc., or remove `target.full_name` from the template's `required_facts` |
| **Pipeline is slow** | `config.py` chunk/worker settings | See §25.15 |
| **Want to disable everything, revert to old behavior** | `.env` | `STRICT_NAME_MATCH=false` and `USE_TEMPLATE_REPORT=false`, restart |

### 25.4 STILL SEEING FALSE POSITIVES — step-by-step

A record about the wrong person made it into Confirmed Records / the assessment.

1. **Find which path let it in.** Run the §25.2 "reasons records WERE included"
   query, or look up the specific record:
   ```sql
   SELECT source_record_id, eligibility_reason FROM records
   WHERE target_id='<TID>' AND source_record_id='<the bad record>';
   ```

2. **If reason = `all target name parts matched (variant-aware)`** → Path B.
   The record's name field genuinely contains all the target's name tokens.
   Possibilities:
   - The target's name parts are too generic (e.g., target = just "Mohammed").
     A 1-token target name is satisfied by ANY record containing "Mohammed".
     Fix: require more name parts at intake, OR add a minimum-parts check in
     `strict_match.py:target_name_fully_matched()` (e.g., require ≥2 parts).
   - Prefix-stripping made two different family names collide. E.g., target
     "Al-Karim" and record "Karim" — these are now treated as equal. If that's
     wrong for your data, set `.env: NAME_STRIP_PREFIXES=false` (you lose the
     Al-Husseini == Husseini matching, but you also lose the false collisions).
   - A variation in `name_variation_generator`'s dictionary is too loose
     (treats two distinct names as variants). Look at `ARABIC_NAME_VARIATIONS`.

3. **If reason = `record phone <X> matches target's known phone <Y>`** → Path E.
   The record contains a phone that's in `target.known_phones`. Possibilities:
   - `_build_known_phones()` in `dossier_pipeline.py` walked the correlation
     table too aggressively and picked up a phone that isn't really the
     target's (e.g., a shared-IMEI phone that belongs to a family member).
     Look at the correlation walk — the `repeat-until-stable` loop. You can
     tighten it (fewer passes) or only correlate via IMSI (more specific than
     IMEI).
   - The phone match used too few digits. Raise `.env: PHONE_MATCH_MIN_DIGITS`
     (e.g., to 12 if your numbers are longer).

4. **If reason = `phone exact match + corroborating evidence`** → Path D.
   Both phone AND date/location matched. This is rare and usually correct. If
   wrong, set `.env: PHONE_ALLOW_WITHOUT_CORROBORATOR=false` (it already is by
   default — Path D requires the corroborator) and check the `_date_matches_target`
   / `_location_matches_target` flags aren't being set incorrectly upstream.

5. **If reason = `strong-ID match: ...`** → Path A. A passport/IMEI/IMSI/email
   exact-matched. Almost always correct. If wrong, the target's stored identifier
   is wrong, or the record's identifier field was mis-parsed. Check
   `strict_match.py:strong_id_match()` and the field keys it reads.

6. **If reason = `strict_match error (defaulted eligible): ...`** → strict_match
   threw and the record was fail-open kept. Read the error, fix the bug, the
   record will be properly classified next run.

### 25.5 SEEING FALSE NEGATIVES (target's own records wrongly in Data Dump) — step-by-step

A record that IS the target's got dumped.

1. **Find the rejection reason:**
   ```sql
   SELECT source_record_id, eligibility_reason, substr(structured_data,1,400)
   FROM records WHERE target_id='<TID>' AND source_record_id='<the record>';
   ```

2. **If reason = `record missing name parts: husseini`** but the record DOES say
   "Al-Husseini":
   - The name field wasn't extracted. Check `living_dossier.py:_extract_record_fields_for_strict_match`
     — does the record use a field name not in `NAME_FIELD_KEYS`? Does it use a
     markdown format (`**name**:`, `- name:`) the regex doesn't catch? Add the
     field name / fix the regex.
   - The name field WAS extracted but prefix-stripping broke it. "Al-Husseini"
     should normalize to "husseini" and so should the target's "Al-Husseini".
     Test: `python3 -c "from tools.shared.strict_match import _normalize_token; print(_normalize_token('Al-Husseini'), _normalize_token('Husseini'))"` — both should print `husseini`. If not, the prefix list / guard is off.

3. **If reason = `no name in record`** but the record has the target's phone
   (CDR / FISA intercept / GEOINT ping):
   - Path E should have caught it. Two checks:
     a. Is `target.known_phones` populated? Check the boot/run log for
        `Known phones (incl. SIM-swap correlations): N`. If N is 0 or the
        primary isn't in there, `_build_known_phones()` in `dossier_pipeline.py`
        isn't reading the correlation table. Confirm `Config.CORRELATION_DB`
        points at a real file.
     b. Is the phone being extracted from the record? Check
        `_extract_record_fields_for_strict_match` — does it pick up `FROM:` /
        `TO:` / `SELECTOR:` / `MSISDN:` style phone fields? The generic phone
        regex requires ≥9 digits and rejects date-shaped strings. If the
        record's phone is formatted oddly, the regex may miss it. Add a
        targeted pattern for that field.

4. **If reason = `record missing name parts: mohammed, ali`** for a record that
   IS about the target but only shows the family name (e.g., a database that
   lists "Mr. Al-Husseini"):
   - This is correct strict behavior — partial name alone isn't enough. To
     admit it, the record needs a corroborating identifier (phone in
     `known_phones` → Path E, or passport → Path A). If the record genuinely
     has nothing but a partial name, it SHOULD be in Data Dump — that's the
     point. A reviewer can manually promote it if they're confident.

5. **Quick global relaxation** (use with care — re-introduces some false
   positives): `.env: PHONE_MATCH_MIN_DIGITS=9` and/or
   `PHONE_ALLOW_WITHOUT_CORROBORATOR=true`.

### 25.6 CITATION HALLUCINATION still happening — step-by-step

The assessment cites IDs that don't exist (bullet numbers, report numbers).

1. **Confirm it's reaching the report.** The post-validation runs in
   `living_dossier.py:run_assessment()` right before writing `assess_<type>.md`:
   `assessment_text = self._strip_fabricated_record_ids(assessment_text)`.
   If you don't see that line, the fix wasn't applied — re-apply `living_dossier.py`.

2. **Check the regex covers your ID format.** `_VALID_RECORD_ID_RE` in
   `living_dossier.py` lists the patterns it accepts:
   `PNR_\w+`, `CDR_\d+`, `FISA_(ORDER|IP|INT)_\d+`, `CNE_[A-Z]+_\w+`,
   `IR-\d{4}-\d{2,4}`, `OSINT_[A-Z]+_\w+`, `GEOINT_[A-Z0-9]+_\w+`, `GEO_\d+`,
   `REPORT_\d+`. If your real data uses a different ID shape (e.g., `WIRE-2024-001`),
   add it to that regex — otherwise it'll be stripped as "fabricated."

3. **The prompt rule is advisory; the post-validation is the enforcement.**
   `_CITATION_RULES` (appended to every assessment scan prompt) tells the LLM
   what valid IDs look like. The LLM may still ignore it — that's why
   `_strip_fabricated_record_ids` exists as the deterministic backstop. If you
   want to tighten the LLM's behavior further, edit `_CITATION_RULES` to be
   more emphatic, but don't rely on it alone.

4. **Test it:** `python3 tests/test_round3_fixes.py` includes
   `test_g4_fabricated_ids_stripped` with 4 cases. Add yours there.

### 25.7 RATING WRONG / "INSUFFICIENT DATA" everywhere — step-by-step

1. **Where does the rating come from?** `report_context_builder.py:_build_assessment_dict`
   tries, in order:
   - `### Score: HIGH` line in `assess_<type>.md`
   - `**Score**: ELEVATED`
   - `Overall <X> rating: CRITICAL`
   - `Rating: MODERATE` / `**Rating**: HIGH`
   - **Derived** from `findings_count` + risk keywords

2. **If your assessment scanner emits a rating in a format not in that list**
   (e.g., `CONFIDENCE: HIGH`), add a regex pattern to the loop. The accepted
   values are whitelisted: `CRITICAL, HIGH, ELEVATED, MODERATE, LOW, MINIMAL,
   INSUFFICIENT DATA, UNDETERMINED`. If your scanner uses different labels, map
   them.

3. **If the derivation is too conservative** (says MODERATE when you want
   ELEVATED), tune the thresholds in `_build_assessment_dict`:
   ```python
   if findings_count >= 8 or (findings_count >= 4 and has_risk): rating = "HIGH"
   elif findings_count >= 4 or (findings_count >= 1 and has_risk): rating = "ELEVATED"
   elif findings_count >= 1: rating = "MODERATE"
   else: rating = "INSUFFICIENT DATA"
   ```
   Lower the numbers, or add more terms to `risk_keywords`.

4. **The disposition (Section X) follows the ratings.** If you see "No specific
   actionable findings" while the body has findings, the ratings are still being
   read as INSUFFICIENT DATA — fix step 2/3 first; the disposition will follow.

### 25.8 HEDGING IN THE REPORT — step-by-step

1. **Where:** narrative `summary_800` slots AND per-section summaries AND the
   chunked-summary REDUCE step all run through a hedge scrubber.

2. **Add the offending phrase to the banned list.** Two places (keep them in
   sync):
   - `config.py` → `NARRATIVE_SLOT_BANNED_PHRASES` (template slot extraction)
   - `living_dossier.py` → `_HEDGE_PHRASES` (section summaries)
   Add the lowercase phrase. The scrubber does a case-insensitive substring check.

3. **The scrubber's flow:** detect → retry once with a stricter prompt → if
   still hedged, fall back to a bullet-list extract of the raw findings. If
   you're seeing hedging in the FINAL output, all three steps failed — usually
   means the phrase isn't in the banned list, OR the hedging is coming from the
   RAW record data (a data agent's text), which can't be scrubbed (it's source
   material, not LLM prose). Check whether the hedge is in a `### Confirmed
   Records` block (raw data — can't fix) or in a `### Summary` / narrative
   section (LLM prose — add to banned list).

### 25.9 WRONG-PERSON DATA IN "ADDITIONAL DATA" BLOCK — step-by-step

1. **Was the phone gated at queue time?** Check the run log for
   `[ENRICH-GATE] blocked N phone(s) whose resolved owner did not match the
   target`. If N=0 and you know a co-traveler's phone got enriched, the owner
   wasn't resolved. Check `phone_owner_resolver.resolve_phone_owner()` — is
   `Config.CORRELATION_DB` set? Does the correlation table actually name the
   owner? Does the voter DB have the phone?

2. **If the owner can't be resolved**, the phone is queued (we can't tell yet),
   BUT `append_to_section` runs the strict-match split — so the enriched
   records should land in `#### Additional Data — Data Dump` (inside the
   `<!-- DATA_DUMP_BEGIN -->` fence), not in `#### Additional Data — Confirmed`.
   If they're in Confirmed, check `living_dossier.py:append_to_section` — is it
   calling `self._split_raw_by_eligibility(...)`? Was the `living_dossier.py`
   change applied?

3. **Quick test:** `python3 tests/test_round3_fixes.py` →
   `test_s345_additional_data_wrapped` verifies this end-to-end.

### 25.10 CONTACT-NETWORK NAMES GARBAGE — step-by-step

1. **The resolver** is `living_dossier.py:_resolve_contact_names`. It picks the
   most-frequent capitalized-word-pair near each phone, with name-prefix-anchored
   matches getting 3× weight.

2. **If a garbage label slips through** ("Cargo Express", "Pattern Analysis"):
   - Add it to the `skip_names` set (exact match), OR
   - Add its tokens to `BAD_TOKENS` (a candidate is garbage if EVERY token is
     in `BAD_TOKENS`). E.g., adding "cargo" and "express" handles "Cargo Express",
     "Express Cargo", "Anatolian Express Cargo", etc.

3. **If a real name is being rejected**: it might be hitting `BAD_TOKENS` by
   coincidence (e.g., a person named "Network" — rare but possible). Remove the
   offending token from `BAD_TOKENS`, or add the real name to a whitelist check.

### 25.11 DATA DUMP EMPTY / ASSESSMENT STILL SEES DUMPED RECORDS — step-by-step

**Data Dump section is empty when records should be there:**
1. Check the migration ran (§25.13). No `assessment_eligible` column → every
   record reads as eligible (the `COALESCE(...,1)` default) → nothing is dumped.
2. Check `target_store.get_data_dump_records(target_id, agent_key)` returns rows
   for that target. If it returns `[]`, no records were flagged ineligible —
   which means strict-match wasn't run, or every record genuinely passed.
3. Check `living_dossier.py:_build_data_dump_md` is being called inside
   `write_agent_section` (it should be).

**Assessment still references records that are in Data Dump:**
1. `_get_data_sections_text(target_id, strip_data_dump=True)` must be the version
   called by `run_assessment`, `generate_executive_summary`,
   `generate_narrative_report`. The default is `True` — if someone passed
   `False`, the dump isn't stripped.
2. `_strip_data_dump_blocks()` regex must match the fence markers. The fences are
   exactly `<!-- DATA_DUMP_BEGIN` ... `<!-- DATA_DUMP_END -->`. If the section
   file's fences are malformed (e.g., a manual edit broke them), the strip won't
   work. Re-run the workflow to regenerate clean section files.

### 25.12 TEMPLATE WORDING — quick recap

See §22.8 for the full dumb-proof walkthrough. Short version:
- Edit `report_templates/<template>.json` → the section's `verbiage`
- New `{{slot}}` → add it to the section's `slots` object with a `source` path
- New `target.*` source path that doesn't exist yet → add it in
  `report_context_builder.py:_build_identifiers_dict`
- Iterate fast: `POST /targets/<id>/report/preview?template_id=<id>` returns the
  rendered markdown without re-running the pipeline
- JSON edits don't need a restart; `report_context_builder.py` edits do

### 25.13 RECOVERY — migration didn't run / corrupt state / reset

**Migration didn't add the new columns** (e.g., the DB file pre-existed and
something blocked the ALTER):
```bash
# Confirm the columns are missing
sqlite3 V2/backend/data/target_store.db "PRAGMA table_info(records);" | grep assessment_eligible
# Empty → missing.

# Option A — add them manually:
sqlite3 V2/backend/data/target_store.db "ALTER TABLE records ADD COLUMN assessment_eligible INTEGER DEFAULT 1;"
sqlite3 V2/backend/data/target_store.db "ALTER TABLE records ADD COLUMN eligibility_reason TEXT;"

# Option B — start fresh (loses prior dossiers, regenerable):
mv V2/backend/data/target_store.db V2/backend/data/target_store.db.old
# Next boot recreates it with the full schema.
```

**Reset everything to a clean slate** (keeps source code, wipes runtime state):
```bash
cd V2/backend
rm -f data/target_store.db data/results_store.db data/voter_registration.db
rm -rf data/dossiers/*
# Restart — fresh DBs created with current schema.
```

**Revert all the new behavior without uninstalling:**
```bash
# .env
STRICT_NAME_MATCH=false
USE_TEMPLATE_REPORT=false
# Restart. Pipeline behaves like before the patch (strict-match columns stay
# but every record reads as eligible; narrative uses the legacy LLM-prose path).
```

### 25.14 RUNNING ONE WORKFLOW FOR DEBUGGING

```bash
cd V2/backend
python3 main.py   # in one terminal

# In another terminal — kick off a targeting workflow via the API:
curl -X POST "http://localhost:8000/targeting/create" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "DEBUG",
    "target_identifiers": {
      "first_name": "Mohammed", "father_name": "Ali", "family_name": "Al-Husseini",
      "phone": "+964-770-555-1234", "passport": "A05551234"
    },
    "selected_agents": ["sigint", "osint"]
  }'
# Note the workflow_id in the response, then:
curl -X POST "http://localhost:8000/targeting/run/<workflow_id>"

# Watch the server stdout — every step prints. Then inspect:
TID=$(ls -t data/dossiers/ | head -1)
ls data/dossiers/$TID/sections/
cat data/dossiers/$TID/sections/00_narrative_report.md
cat data/dossiers/$TID/sections/05_osint.md   # look for ### Confirmed vs ### Data Dump
```

For a fast single-source debug, use `/single-agent` instead (one agent, no
enrichment, no cross-agent fan-out).

### 25.15 PERFORMANCE KNOBS

| Knob | File | Effect |
|------|------|--------|
| `DOSSIER_CHUNK_SIZE` | `config.py` (default 3000) | Smaller = more chunks, more LLM calls, finer assessment granularity |
| `DOSSIER_MAX_SUB_AGENT_WORKERS` | `config.py` (default 4) | More workers = faster but more concurrent LLM load |
| `SUB_AGENT_RECORDS_PER_WORKER` | `config.py` (default 2) | Larger = fewer workers, fewer LLM calls, coarser extraction |
| `TOOL_CALLING_MAX_ITERATIONS` | `config.py` (default 10) | Cap on the LLM tool loop per agent search |
| `ENRICHMENT_MAX_ITERATIONS` | `config.py` (default 3) | How many enrichment rounds before forced convergence |
| `max_workers` in `_step_3_collect()` | `dossier_pipeline.py` | Concurrent data agents (capped at 5) |
| `NARRATIVE_SLOT_MAX_CHARS` | `config.py` (default 800) | Per narrative-slot length cap (≈130 words). Set 1500 for ≈300 words. |

---

*End of documentation. Last updated: 2026-05-11.*
*Covers Living Dossier architecture + strict identity matching (Issue 1) + LE report templates (Issue 2). Old report_synthesizer/targeting_report_synthesizer/agent_consolidator are DELETED.*
*Section 18-19: plug-and-play migration guide for replacing demo tools with real APIs.*
*Section 20: full tool-by-tool reference.*
*Section 21: strict identity matching — the false-positive fix (Paths A-E, two-tier output, phone-owner resolution).*
*Section 22: LE report templates — schema, the 4 templates, how to edit/add them.*
*Section 23: new config flags (17, added with Issues 1 & 2).*
*Section 24: changelog — every fix (L1-L4, R1-R9, G1-G4, §3.x), test coverage, full file inventory.*
*Section 25: troubleshooting & diagnostics — symptom → file → knob; the eligibility_reason breadcrumb; SQLite diagnostic queries; false-positive/false-negative/citation/rating/hedging diagnosis flows; recovery procedures; performance knobs.*
