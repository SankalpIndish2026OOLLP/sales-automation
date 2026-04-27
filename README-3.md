# Eastern Enterprise — Sales Automation Agentic System
**Version:** 3.0 — Complete System Architecture  
**Prepared for:** Technical Architect  
**Organisation:** Eastern Enterprise  
**Date:** April 2026  
**Status:** Planning — Pre-Development

---

## Executive Summary

Eastern Enterprise's sales data team loses an estimated 15–20 hours per week to 15 identified repetitive tasks spread across five tools: HubSpot CRM, Lusha, LinkedIn Sales Navigator, Ringover, and a custom Flask job scraper. These tasks fall into four failure categories — deduplication, standardisation, visibility, and reporting — and every one of them has the same root cause: **the tools do not talk to each other, and there is no shared intelligence layer**.

This document defines the architecture of an **AI-agentic orchestration system** that sits between all five tools and automates the data operations the team currently does by hand. The system is built on an existing on-premise CPU server, uses the Model Context Protocol (MCP) to connect Claude AI to HubSpot and other tools natively, and adds a similarity scoring engine that ranks new leads by conversion likelihood using 500+ historical outcomes already in HubSpot.

**The system resolves 13 of 15 problems fully and 2 partially (LinkedIn Sales Navigator — a LinkedIn constraint, not a build choice).**

The Flask scraper is one data source among several. Lusha enriches contacts from any source. Ringover provides call intelligence. Sales Navigator provides prospect discovery. HubSpot is the single system of record. Claude, via MCP, is the orchestration brain that coordinates all of them.

| Key metric | Value |
|---|---|
| Problems resolved | 13 fully · 2 partial · 0 impossible |
| Development time | 14 weeks · 1 developer |
| New monthly infrastructure cost | €40–120/month |
| Estimated monthly time recovered | 15–20 hours/week across the team |
| Server | On-premise CPU (existing) |
| Core tools | HubSpot Professional · Lusha · LinkedIn Sales Nav · Ringover |
| AI orchestration | Claude API via MCP |

---

## Table of Contents

1. [The Complete Problem Map](#1-the-complete-problem-map)
2. [Tool Landscape and Data Flows](#2-tool-landscape-and-data-flows)
3. [Full System Architecture](#3-full-system-architecture)
4. [MCP Integration Strategy](#4-mcp-integration-strategy)
5. [The Five Agents — Detailed Design](#5-the-five-agents--detailed-design)
6. [Data Flow: End-to-End Lifecycle](#6-data-flow-end-to-end-lifecycle)
7. [How Each of the 15 Problems Gets Resolved](#7-how-each-of-the-15-problems-gets-resolved)
8. [Phase-by-Phase Development Plan](#8-phase-by-phase-development-plan)
9. [Tech Stack](#9-tech-stack)
10. [API Access Requirements](#10-api-access-requirements)
11. [Cost Breakdown](#11-cost-breakdown)
12. [Data Architecture](#12-data-architecture)
13. [Risks and Mitigations](#13-risks-and-mitigations)
14. [Pre-Development Checklist](#14-pre-development-checklist)
15. [Out of Scope](#15-out-of-scope)

---

## 1. The Complete Problem Map

The 15 problems occur across the full sales data lifecycle — not just at the point of company discovery. Each tool contributes to the mess:

| # | Problem | Where it happens | Tool involved | Root cause |
|---|---|---|---|---|
| 1 | Lead sourcing duplicates | Prospecting stage | HubSpot + Sales Nav + Scraper | No shared registry across sources |
| 2 | Lead enrichment duplicates | Enrichment stage | Lusha | No enrichment log or cache |
| 3 | CRM upload duplicates | CRM write stage | HubSpot | No pre-write dedup check |
| 4 | Data re-cleaning | Post-upload | HubSpot | No entry-time standards enforcement |
| 5 | Lead re-qualification | Pipeline management | HubSpot + Ringover | No clear status field; call data not connected to CRM qualification |
| 6 | Campaign list rebuilding | Campaign setup | HubSpot | No reusable list library |
| 7 | Data re-segmentation | Campaign setup | HubSpot | No saved segments/filters |
| 9 | Email re-validation | Enrichment stage | NeverBounce / Lusha | No validation cache |
| 10 | Report data re-prep | Reporting | HubSpot + Excel | No automated reporting pipeline |
| 11 | Campaign re-tagging | Campaign management | HubSpot | No tag governance or enforcement |
| 13 | Tool-based extraction | Research stage | Sales Nav + Lusha + others | No unified extraction layer; each tool used independently |
| 14 | Tech stack re-research | Research stage | Scraper + Wappalyzer | No cached tech stack per company |
| 15 | Data re-formatting | Any data entry point | HubSpot | No formatting rules enforced at write |
| — | *(New)* No similarity scoring | Decision stage | None currently | No intelligence layer to rank leads |
| — | *(New)* Ringover data siloed | Qualification | Ringover | Call outcomes not feeding back to CRM qualification |

**Problem 8** was absent from the original numbering and is not included.

---

## 2. Tool Landscape and Data Flows

### 2.1 Current State — What Each Tool Does Today (in isolation)

**HubSpot Professional** — System of record for all company and contact data. Stores pipeline status, campaign associations, and tags. Currently suffers from duplicates, inconsistent field values, and manual data entry.

**Lusha** — Contact enrichment. Used manually per lead to find emails and phone numbers. No tracking of what has already been enriched. Credits are being wasted on re-enriching the same contacts.

**LinkedIn Sales Navigator** — Prospect discovery and research. Used manually to identify target companies and contacts. No automated way to extract data due to LinkedIn's API restrictions. Output is manual — either copy-paste or CSV export.

**Ringover** — Outbound calling and call tracking. Stores call logs, outcomes, and recording notes. Currently completely siloed from HubSpot. The sales team manually notes call outcomes and there is no automated feedback loop into lead qualification.

**Flask Job Scraper (existing)** — Custom-built tool that scrapes 3 Netherlands-based job boards. Identifies companies actively hiring in relevant roles. Also extracts tech stack from job descriptions. Has a block list and a HubSpot status upload feature. This is one data source, not the whole pipeline.

### 2.2 Target State — What Each Tool Does in the Agentic System

| Tool | Role in agentic system | Direction |
|---|---|---|
| HubSpot | System of record; all agents read/write here via MCP | Bidirectional |
| Lusha | Enrichment on-demand, cache-first; triggered by agent, never manually | Inbound data |
| Sales Navigator | Prospect discovery via manual CSV export; auto-ingested by agent | Inbound data |
| Ringover | Call outcomes and engagement signals fed to qualification agent automatically | Inbound signals |
| Flask scraper | Job-based company discovery; one of three inbound channels | Inbound data |
| Claude via MCP | Orchestration brain; reads all tools, makes decisions, writes to HubSpot | Orchestration |

### 2.3 The Three Inbound Channels

The agentic system has **three distinct ways companies enter the pipeline**. All three converge at the Agent Gate:

1. **Job scraper** — Automated. Companies hiring in NL for relevant roles. Triggered weekly or on-demand. Output: company name, LinkedIn URL, tech stack, role type.

2. **Sales Navigator CSV export** — Semi-automated. Sales rep exports a list from Sales Nav. Drops the CSV in a shared folder. A file-watcher agent ingests it automatically within minutes.

3. **Manual entry / direct HubSpot import** — Human-initiated. Any company the sales team adds directly to HubSpot or imports from an external list. The standardisation and dedup checks still run on these records automatically.

All three channels feed the same Agent Gate and downstream pipeline.

---

## 3. Full System Architecture

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    EASTERN ENTERPRISE — AGENTIC SYSTEM                     ║
║                         (On-Premise Server)                                ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  ┌─────────────────────────────────────────────────────────────────────┐   ║
║  │                    INBOUND CHANNELS (3)                             │   ║
║  │                                                                     │   ║
║  │  ┌──────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │   ║
║  │  │  Flask Scraper   │  │  Sales Nav CSV  │  │  Manual / Import │  │   ║
║  │  │  (automated,     │  │  (manual export │  │  (direct HubSpot │  │   ║
║  │  │   weekly)        │  │   → file watch) │  │   or bulk CSV)   │  │   ║
║  │  └────────┬─────────┘  └────────┬────────┘  └────────┬─────────┘  │   ║
║  └───────────┼──────────────────────┼────────────────────┼────────────┘   ║
║              └──────────────────────┼────────────────────┘                ║
║                                     ▼                                      ║
║  ┌──────────────────────────────────────────────────────────────────────┐  ║
║  │                         AGENT GATE                                   │  ║
║  │  ① Block list check   ② 60-day dedup check   ③ HubSpot presence    │  ║
║  │  ④ Ringover: was this company previously contacted?                 │  ║
║  │  ⑤ Master file: is this company approved for enrichment?           │  ║
║  └──────────────────────────────┬───────────────────────────────────────┘  ║
║                                  │                                          ║
║         ┌────────────────────────┼──────────────────────┐                  ║
║         ▼                        ▼                      ▼                  ║
║  ┌─────────────────┐  ┌──────────────────────┐  ┌────────────────────┐   ║
║  │ ENRICHMENT      │  │  SIMILARITY SCORING  │  │  STANDARDISATION   │   ║
║  │ AGENT           │  │  AGENT               │  │  AGENT             │   ║
║  │                 │  │                      │  │                    │   ║
║  │ · Lusha API     │  │ · pgvector DB        │  │ · standards.yaml   │   ║
║  │   (cache-first) │  │ · 500+ labeled cos.  │  │ · Tag enforcement  │   ║
║  │ · NeverBounce   │  │ · Ringover signals   │  │ · Format rules     │   ║
║  │   (90-day cache)│  │ · Score 0–100        │  │ · Industry taxonomy│   ║
║  │ · Wappalyzer    │  │ · "Similar to X"     │  │ · Tech stack norm  │   ║
║  └────────┬────────┘  └──────────┬───────────┘  └─────────┬──────────┘   ║
║           └───────────────────────┼──────────────────────────┘             ║
║                                   ▼                                         ║
║  ┌──────────────────────────────────────────────────────────────────────┐  ║
║  │              QUALIFICATION AGENT                                     │  ║
║  │                                                                      │  ║
║  │  · Reads Ringover call history (frequency, outcomes, last contact)  │  ║
║  │  · Reads HubSpot pipeline history and existing status               │  ║
║  │  · Applies team-defined qualification criteria                      │  ║
║  │  · Writes qualification_stage + timestamp back to HubSpot via MCP  │  ║
║  │  · Flags companies needing human decision vs auto-qualified         │  ║
║  └──────────────────────────────────┬───────────────────────────────────┘  ║
║                                     ▼                                       ║
║  ┌──────────────────────────────────────────────────────────────────────┐  ║
║  │              SALES DASHBOARD (Flask UI — existing, extended)         │  ║
║  │                                                                      │  ║
║  │  · Companies sorted by similarity score (highest first)             │  ║
║  │  · Match score + "Similar to [X] (long-term client)" explanation    │  ║
║  │  · Qualification stage visible                                      │  ║
║  │  · Ringover call history visible per company                        │  ║
║  │  · Accept → writes to HubSpot   Reject → optionally adds to block  │  ║
║  └──────────────────────────────────┬───────────────────────────────────┘  ║
║                                     │                                       ║
║        ┌────────────────────────────┼────────────────────────┐             ║
║        ▼                            ▼                        ▼             ║
║  ┌───────────────┐    ┌──────────────────────┐   ┌─────────────────────┐  ║
║  │  HubSpot CRM  │    │  Similarity DB       │   │  REPORTING AGENT    │  ║
║  │  (via MCP)    │    │  (pgvector, updated  │   │                     │  ║
║  │               │    │   from decisions)    │   │  · Monday 08:00     │  ║
║  │  Clean, tagged│    │                      │   │  · HubSpot queries  │  ║
║  │  enriched,    │    │  Feedback loop:      │   │  · Ringover queries │  ║
║  │  qualified    │    │  accept/reject       │   │  · Lusha usage      │  ║
║  │               │    │  improves scores     │   │  · Slack or email   │  ║
║  └───────────────┘    └──────────────────────┘   └─────────────────────┘  ║
║                                                                              ║
║  ┌──────────────────────────────────────────────────────────────────────┐  ║
║  │              SHARED STATE LAYER (PostgreSQL + pgvector)              │  ║
║  │  Dedup registry · Enrichment log · Validation cache · Block list    │  ║
║  │  Segment library · Tag taxonomy · Similarity vectors · Scrape log   │  ║
║  └──────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 4. MCP Integration Strategy

MCP (Model Context Protocol) is the mechanism that allows Claude to directly read and write to HubSpot — and potentially Ringover and Lusha — as native tools, rather than requiring custom API wrappers for every operation.

### 4.1 Why MCP for This Project

Without MCP, every HubSpot operation requires a developer to write and maintain a custom API integration. With MCP, Claude has access to HubSpot's full data model as first-class tools — search contacts, create companies, update properties, manage lists — using the same interface for every agent. This dramatically reduces the code needed to build each agent and makes the system easier to extend.

### 4.2 MCP Servers to Deploy

| Tool | MCP Server | Status | What it enables |
|---|---|---|---|
| HubSpot | Official HubSpot MCP server | Available | Full CRM read/write: contacts, companies, deals, lists, properties, campaigns |
| Ringover | Custom MCP wrapper (build required) | Build required | Call log queries, outcome retrieval, rep activity |
| Lusha | Custom MCP wrapper (build required) | Build required | Contact enrichment with cache check, credit tracking |
| NeverBounce | REST API (no MCP needed) | Direct integration | Email validation with cache |
| Wappalyzer | REST API (no MCP needed) | Direct integration | Tech stack lookup |

### 4.3 How Agents Use MCP

Each agent is a Claude API call with a system prompt that defines its role and the MCP tools it has access to. Example:

```
Enrichment Agent system prompt:
"You are an enrichment agent. Before calling Lusha, check the enrichment_log 
table. If the company was enriched within 60 days, return cached data. 
Otherwise, call Lusha MCP tool, write the result to enrichment_log with 
today's date and credits_used, then return the enriched record."
```

The agent reasons about what to do, calls the appropriate MCP tools, and returns a structured result. This is the "agentic" part — Claude is not just executing a fixed script, it is reasoning about state, making decisions, and taking multi-step actions.

### 4.4 MCP vs Direct API — Decision Rule

- **Use MCP** when the operation involves reading or writing to HubSpot, or when the agent needs to reason about what action to take.
- **Use direct REST API** for single-purpose operations with no reasoning required: NeverBounce validation, Wappalyzer tech stack lookup, Ringover call log fetch (until Ringover MCP wrapper is built).

---

## 5. The Five Agents — Detailed Design

### Agent 1: Agent Gate

**Purpose:** First checkpoint. Every company from every inbound channel passes through this gate. Nothing reaches enrichment or HubSpot without passing all checks.

**Trigger:** Company record received from scraper webhook, CSV file-watcher, or manual import event.

**Checks (in order):**
1. Block list lookup — LinkedIn URL match against `block_list` table
2. Dedup registry — Has this company been processed in the last 60 days? If yes, reuse cached data, skip enrichment
3. HubSpot presence — Does this company already exist in HubSpot with a terminal status (not interested, rejected)? If yes, discard
4. Ringover prior contact — Has this company been called more than 5 times with no positive outcome? Auto-flag for block list review
5. Master file check — Is this company in the approved master file for enrichment?

**Output:** PASS (proceed to enrichment) | SKIP (reuse cache) | DISCARD (log reason) | FLAG (needs human review)

---

### Agent 2: Enrichment Agent

**Purpose:** Enrich every new company with contact data, email validation, and tech stack. Always cache-first — never call an external API if valid cached data exists.

**Data sources used:**
- **Lusha** (via MCP wrapper): phone, email, LinkedIn contacts, company size
- **NeverBounce** (direct API): email validation
- **Wappalyzer** (direct API): tech stack supplement (Flask scraper handles primary tech stack; Wappalyzer fills gaps for Sales Nav and manual imports)

**Cache rules:**
- Lusha: skip if enriched within 60 days
- NeverBounce: skip if validated within 90 days
- Wappalyzer: skip if tech stack populated

**Writes to:** `enrichment_log` table, company record in PostgreSQL

**HubSpot write:** Enriched fields pushed to HubSpot company record via MCP after standardisation agent runs

---

### Agent 3: Similarity Scoring Agent

**Purpose:** Score every new company against the historical pool of 500+ labeled HubSpot companies. Surface the score and a plain-English explanation in the dashboard.

**How it works:**
1. Build a feature vector for the new company from: industry vertical, employee size band, HQ country, revenue band, India presence, tech stack (binary flags for top 15 technologies), job roles being hired, Ringover engagement signal
2. Query pgvector for top 5 nearest neighbours from the 500+ historical company embeddings
3. Compute weighted score: long-term client = weight 1.0, short-term = 0.8, not connected = 0.3, not interested = -0.3, rejected = -0.5
4. Generate explanation string: *"Similar to TechFirm NL (long-term client, 92% match) and DataCo Amsterdam (short-term client, 78% match)"*
5. Write score and explanation to company record

**Feedback loop:** When sales rep accepts or rejects a company in the dashboard, the decision is written back to pgvector with its weight, improving future scores passively.

**Model:** `sentence-transformers/all-MiniLM-L6-v2` — open-source, CPU-only, runs on-premise

---

### Agent 4: Qualification Agent

**Purpose:** Determine and maintain lead qualification status across the entire pipeline. Eliminates re-checking and status ambiguity by maintaining a clear, timestamped qualification field in HubSpot.

**Data sources:**
- HubSpot pipeline history (via MCP)
- Ringover call log: number of calls, last outcome, last contact date, average answer rate
- Similarity score from Agent 3
- Sales team-defined qualification criteria (stored in config)

**Qualification logic:**
```
IF ringover.calls > 0 AND ringover.last_outcome == "meeting_booked":
    → qualification_stage = "sales_qualified"
IF ringover.calls >= 3 AND ringover.answer_rate < 0.2:
    → qualification_stage = "low_engagement" (flag for review)
IF similarity_score >= 70 AND hubspot.status == "new":
    → qualification_stage = "high_priority" (surface to top of dashboard)
IF ringover.calls >= 5 AND ringover.last_outcome == "not_interested":
    → qualification_stage = "disqualified" + prompt to block list
```

**Writes to:** HubSpot `qualification_stage` property + timestamp, via MCP

**Critical:** The qualification criteria config must be defined by the sales team before this agent is built. The agent applies rules — it does not invent them.

---

### Agent 5: Reporting Agent

**Purpose:** Automated weekly pipeline report. Zero manual data prep. Combines HubSpot and Ringover data into a structured summary.

**Schedule:** Monday 08:00 (cron). Configurable.

**Data queries:**
- HubSpot (via MCP): new companies this week by status, pipeline movement (status changes), campaign list activity, tag usage, enrichment coverage
- Ringover (direct API): calls made, answer rate, meetings booked, conversion rate, per-rep breakdown
- Enrichment log: Lusha credits used vs saved by cache, NeverBounce calls vs cache hits
- Scraper log: companies scraped, passed gate, blocked (reason breakdown)

**Output format:** Structured markdown report delivered via:
- Slack webhook (configurable channel)
- OR email (configurable recipient)
- OR writes to a shared network folder as a dated file

**No Excel. No manual queries. No human intervention.**

---

### Agent 6: Standardisation Agent (runs inline, not standalone)

**Purpose:** Not a separate agent invocation — runs as a post-processing step within every write operation before any data reaches HubSpot.

**Enforced by `standards.yaml` config (edited by sales team, no code changes):**
- Company name: title case, strip legal suffixes for matching (BV, GmbH, Ltd, Inc) while preserving for display
- Industry vertical: maps input to approved taxonomy. Unknown values → flagged, not silently written
- Country: normalises all variants to ISO codes (Netherlands / NL / The Netherlands → NL)
- Employee size: bucketed to standard bands (1–10, 11–50, 51–200, 201–500, 501–1000, 1000+)
- Revenue: bucketed to bands in EUR millions
- Tech stack: canonical name mapping (ReactJS → React, NodeJS → Node.js, Postgres → PostgreSQL)
- Campaign tags: only tags from approved list permitted. Unapproved tags → rejected with alert to rep

---

## 6. Data Flow: End-to-End Lifecycle

### 6.1 New Company — Full Journey

```
Day 0:
Company identified (scraper / Sales Nav CSV / manual)
    ↓
Agent Gate (block? dedup? already in HubSpot? prior Ringover contact?)
    ↓ PASS
Enrichment Agent (Lusha → contacts, NeverBounce → validate, Wappalyzer → tech)
    ↓
Standardisation (inline) — format all fields per standards.yaml
    ↓
Similarity Scoring — score 0–100, generate "similar to" explanation
    ↓
Qualification Agent — check Ringover history, apply criteria, set stage
    ↓
Write to PostgreSQL (shared state) + write to HubSpot via MCP
    ↓
Appear in Sales Dashboard — sorted by score, with explanation and qualification stage
    ↓
Sales rep: Accept or Reject
    ↓ Accept                              ↓ Reject
HubSpot status → active pipeline    Similarity DB updated (negative weight)
                                    Option: Add to block list
```

### 6.2 Existing Company — Ongoing Maintenance

```
Weekly:
Ringover call log sync
    ↓
Qualification Agent re-evaluates any company with new call activity
    ↓
HubSpot qualification_stage updated via MCP

Campaign tagging:
Any HubSpot property update triggers standardisation check
    ↓
Unapproved tags → rejected, rep alerted

Reporting:
Monday 08:00 — Reporting Agent queries HubSpot + Ringover
    ↓
Report assembled and delivered (Slack / email)
```

### 6.3 Sales Navigator Import

```
Sales rep manually exports CSV from Sales Nav
    ↓
Drops CSV in /shared/sales_nav_imports/ folder
    ↓
File-watcher daemon detects new file (within 60 seconds)
    ↓
Parse CSV → extract: company name, LinkedIn URL, size, industry, country
    ↓
Agent Gate (same checks as all other channels)
    ↓
Full pipeline: Enrichment → Scoring → Qualification → Dashboard
```

---

## 7. How Each of the 15 Problems Gets Resolved

| # | Problem | Resolution mechanism | Agent(s) |
|---|---|---|---|
| 1 | Lead sourcing duplicates | Dedup registry checks LinkedIn URL across all 3 inbound channels before any processing. Same company from scraper and Sales Nav = one record | Agent Gate |
| 2 | Lead enrichment duplicates | 60-day Lusha cache. Agent Gate checks `enrichment_log` before any Lusha call. Credits saved immediately | Enrichment Agent |
| 3 | CRM upload duplicates | Agent Gate checks HubSpot via MCP before any write. Fuzzy name matching catches "TCS" vs "Tata Consultancy Services" | Agent Gate + MCP |
| 4 | Data re-cleaning | Standardisation runs inline on every write. `standards.yaml` config is the single source of truth. Retroactive batch cleanup in Phase 3 | Standardisation |
| 5 | Lead re-qualification | Ringover call outcomes + HubSpot history feed Qualification Agent automatically. `qualification_stage` field written with timestamp. No manual re-checking | Qualification Agent |
| 6 | Campaign list rebuilding | HubSpot list library maintained by agent. Saved lists created once, reused. New companies automatically added to relevant lists based on properties | Reporting Agent + MCP |
| 7 | Data re-segmentation | Saved HubSpot filters + segment library. Agent creates and names segments that persist. Standards enforcement means segments stay clean | MCP + Standardisation |
| 9 | Email re-validation | 90-day NeverBounce cache. Email validated once and result stored. Free tier (1,000/mo) is sufficient with caching | Enrichment Agent |
| 10 | Report data re-prep | Reporting Agent queries HubSpot + Ringover every Monday. No Excel. No manual queries | Reporting Agent |
| 11 | Campaign re-tagging | `standards.yaml` tag taxonomy enforced at write. Unapproved tag → rejected, not silently written. One-time retroactive tag cleanup in Phase 3 | Standardisation |
| 13 | Tool extraction (Sales Nav) | **Partial.** LinkedIn prohibits API automation. Safe path: CSV export → file-watcher → auto-ingested within 60 seconds. The manual step is export only. All downstream processing is automatic | File-watcher + Agent Gate |
| 14 | Tech stack re-research | Flask scraper extracts tech stack from job postings. Wappalyzer supplements for Sales Nav and manual imports. Both cached in `company.tech_stack` JSONB. Never re-fetched if populated | Enrichment Agent |
| 15 | Data re-formatting | `standards.yaml` enforces all field formats at every write. No manual reformatting | Standardisation |
| — | No similarity scoring | Similarity Scoring Agent ranks all new companies. 500+ labeled HubSpot companies provide Day 1 accuracy | Scoring Agent |
| — | Ringover data siloed | Qualification Agent reads Ringover automatically. Call outcomes feed lead scoring and disqualification | Qualification Agent |

---

## 8. Phase-by-Phase Development Plan

> **Developer:** 1 (the existing Flask scraper builder)  
> **Server:** Existing on-premise CPU  
> **HubSpot plan:** Professional (API fully included)  
> **Labeled HubSpot companies:** 500+ (similarity scoring accurate from Day 1)

---

### Phase 1 — Foundation & Infrastructure (Weeks 1–3)

**Goal:** Build the shared state layer, configure the HubSpot MCP server, and get the Agent Gate working. This phase has no AI agents yet — it is pure infrastructure. Everything else depends on it.

| Task | Detail | Effort |
|---|---|---|
| On-premise environment setup | Python 3.11+, PostgreSQL 15+, pgvector extension, virtualenv, cron | 2 days |
| Database schema | `companies`, `enrichment_log`, `validation_cache`, `block_list`, `similarity_vectors`, `scrape_runs`, `segment_library` | 2 days |
| HubSpot MCP server setup | Deploy official HubSpot MCP server. Configure private app API key with all required scopes. Test all read/write operations | 2 days |
| Agent Gate — block list check | Lookup against `block_list` table by LinkedIn URL | 1 day |
| Agent Gate — dedup registry | 60-day enrichment cache check | 1 day |
| Agent Gate — HubSpot presence | Query HubSpot via MCP for existing company by LinkedIn URL or normalised name | 2 days |
| Agent Gate — Ringover check | Direct API call to Ringover: prior calls, last outcome | 1 day |
| File-watcher daemon | Watches `/shared/sales_nav_imports/` for new CSVs. Parses and routes to Agent Gate | 2 days |
| Scraper webhook | Extend Flask scraper to POST to Agent Gate endpoint on company discovery | 1 day |
| Manual import handler | Handle bulk HubSpot CSV imports via webhook or scheduled sync | 1 day |
| Integration test — all 3 inbound channels | End-to-end test: scraper → gate, CSV → gate, manual → gate | 2 days |

**Phase 1 deliverable:** All inbound channels flow through a working Agent Gate. Nothing bad enters the pipeline. Shared state layer is live.

---

### Phase 2 — Enrichment Agent (Weeks 4–5)

**Goal:** Automate all enrichment — Lusha, NeverBounce, Wappalyzer — with cache-first logic. Stop credit waste immediately.

| Task | Detail | Effort |
|---|---|---|
| Lusha MCP wrapper | Build custom MCP server wrapping Lusha REST API. Expose as Claude tool: `enrich_company`, `enrich_contact` | 3 days |
| Lusha cache-first logic | Before any Lusha call, check `enrichment_log`. Cache hit = return stored data | 1 day |
| NeverBounce integration | Direct API wrapper. 90-day validation cache. Batch validation support | 2 days |
| Wappalyzer integration | Direct API wrapper. Cache result in `company.tech_stack` | 1 day |
| Enrichment agent prompt | Claude system prompt that orchestrates: check cache → enrich if needed → write to log | 1 day |
| Credit usage tracking | `enrichment_log.credits_used` field. Reporting query calculates monthly savings | 1 day |
| Integration test | Full enrichment pipeline for 20 test companies | 1 day |

**Phase 2 deliverable:** Lusha credits stop being wasted. Every enrichment is cached. 60–80% reduction in API calls from day 1.

---

### Phase 3 — Similarity Scoring (Weeks 6–7)

**Goal:** The sales dashboard shows ranked, explained companies. Sales reps see what matters first.

| Task | Detail | Effort |
|---|---|---|
| HubSpot historical export | Pull all 500+ labeled companies from HubSpot via MCP. Normalise fields | 1 day |
| Ringover historical sync | Pull call history for all existing companies. Match to HubSpot records | 1 day |
| Feature vector design | Define and encode all vector fields. Validate with sales team | 1 day |
| sentence-transformers setup | Install `all-MiniLM-L6-v2` on on-premise server. Test CPU inference speed | 0.5 days |
| Embed all 500+ companies | Generate embeddings. Store in pgvector `similarity_vectors` table | 1 day |
| Scoring agent | Claude prompt: compute similarity, weight by status, generate explanation string | 2 days |
| Dashboard integration | Add score column, sort by score, "similar to" explanation, colour-coded score band | 3 days |
| Feedback loop | Accept/reject in dashboard writes weight to pgvector. Block list prompt on reject | 1 day |
| Calibration | Test against 20 known good/bad companies. Adjust weights | 1 day |

**Phase 3 deliverable:** Dashboard ranks companies by conversion likelihood. Explanation tells reps exactly why.

---

### Phase 4 — Qualification Agent (Weeks 8–9)

**Goal:** Lead qualification status is always current, always visible, and never requires a human to re-check.

| Task | Detail | Effort |
|---|---|---|
| Ringover MCP wrapper | Build custom MCP server wrapping Ringover REST API. Expose: `get_call_history`, `get_outcomes` | 3 days |
| Qualification criteria config | Work with sales team to define criteria. Store in `qualification_config.yaml` | 2 days (collaborative) |
| Qualification agent | Claude prompt that reads Ringover history + HubSpot status + similarity score → applies criteria → writes `qualification_stage` | 3 days |
| Scheduled re-qualification | Run qualification agent on any company with Ringover activity in last 7 days | 1 day |
| Dashboard qualification column | Show `qualification_stage` badge on each company card | 1 day |

**Phase 4 deliverable:** No lead is ever re-qualified manually. Ringover outcomes feed HubSpot automatically.

---

### Phase 5 — Standardisation (Weeks 10–11)

**Goal:** Every record entering HubSpot is clean and formatted. Retroactive cleanup of existing data.

| Task | Detail | Effort |
|---|---|---|
| Sales team workshop | Define: industry taxonomy, tag list, country codes, size bands, tech stack canonical names | 2 days (collaborative) |
| `standards.yaml` | Build complete config file from workshop output | 1 day |
| Standardisation inline module | Python module called within every write operation. Input: raw record. Output: standardised record or rejection with reason | 2 days |
| HubSpot write enforcement | All MCP writes pass through standardisation. No direct HubSpot writes bypass this | 1 day |
| Campaign tag enforcement | Validate against approved tag list before write | 1 day |
| Retroactive CRM batch clean | Run standardisation module against all existing HubSpot records. Log changes | 2 days |
| QA with sales team | Review 50 cleaned records for accuracy | 1 day |

**Phase 5 deliverable:** Clean HubSpot from this point forward. All existing records normalised.

---

### Phase 6 — Reporting Agent (Weeks 12–13)

**Goal:** Zero manual reporting. Weekly summary delivered Monday morning.

| Task | Detail | Effort |
|---|---|---|
| HubSpot reporting queries | Via MCP: pipeline status, movement, new companies, list activity, tag usage, enrichment coverage | 2 days |
| Ringover reporting queries | Direct API: calls, answer rate, meetings booked, rep breakdown | 1 day |
| Report assembly | Claude Haiku generates structured markdown summary from raw query data | 1 day |
| Enrichment usage section | Lusha credits used vs saved, NeverBounce cache hit rate | 0.5 days |
| Scraper + gate summary | Weekly scrape stats, gate rejection breakdown | 0.5 days |
| Delivery integration | Slack webhook OR email, configurable per organisation | 1 day |
| Scheduling | Cron job: Monday 08:00. Configurable | 0.5 days |
| Test | Run 2 consecutive weekly reports, validate all numbers against manual spot-check | 1 day |

**Phase 6 deliverable:** No Excel. No manual queries. Weekly report arrives automatically.

---

### Phase 7 — Hardening & Handover (Week 14)

- Full regression test across all phases and all 3 inbound channels
- Log-based alerting: email alert if any agent fails or errors
- Monitoring dashboard: simple Flask page showing agent health, last run times, cache hit rates
- Developer documentation (architecture, each agent, config files)
- Sales team handover guide (1-page: master file, how scores work, how to read the report)
- Deployment checklist

---

### Timeline Summary

```
Week 1–3:   Phase 1 — Foundation & Infrastructure
Week 4–5:   Phase 2 — Enrichment Agent
Week 6–7:   Phase 3 — Similarity Scoring
Week 8–9:   Phase 4 — Qualification Agent
Week 10–11: Phase 5 — Standardisation
Week 12–13: Phase 6 — Reporting Agent
Week 14:    Phase 7 — Hardening & Handover

Total: 14 weeks · 1 developer
```

---

## 9. Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Language | Python 3.11+ | Existing scraper is Flask/Python |
| Web framework | Flask | Existing dashboard; no migration |
| Database | PostgreSQL 15+ | Reliable, runs on-premise, supports pgvector |
| Vector search | pgvector extension | No external vector DB; same server |
| Embedding model | `sentence-transformers/all-MiniLM-L6-v2` | Open-source, CPU-only, 384-dim |
| AI orchestration | Claude API — Haiku (bulk operations), Sonnet (complex reasoning) | Haiku ~€0.23/MTok for 90% of tasks |
| MCP: HubSpot | Official HubSpot MCP server | Maintained by HubSpot, full CRM coverage |
| MCP: Lusha | Custom wrapper (build in Phase 2) | No official Lusha MCP server exists |
| MCP: Ringover | Custom wrapper (build in Phase 4) | No official Ringover MCP server exists |
| Scheduler | Python `APScheduler` or `cron` | Simple, no extra infrastructure |
| File-watcher | Python `watchdog` library | Monitors Sales Nav CSV drop folder |
| Enrichment | Lusha REST API (via MCP wrapper) | Existing subscription |
| Call data | Ringover REST API (via MCP wrapper) | Existing subscription |
| Email validation | NeverBounce REST API (direct) | Free tier + cache |
| Tech stack | Wappalyzer REST API (direct) | Free tier: 50/day |
| Standards config | YAML | Editable by sales team without code |
| Logging | Python logging + file rotation | On-premise, no external service |

---

## 10. API Access Requirements

Confirm all of the following **before development starts**. Some have delays.

| Tool | What to request | Lead time | Notes |
|---|---|---|---|
| **HubSpot** | Private App API key | Instant | Scopes required: `crm.objects.contacts.read/write`, `crm.objects.companies.read/write`, `crm.lists.read/write`, `crm.schemas.read`, `crm.objects.deals.read`, `sales-email-read` |
| **Lusha** | API access confirmation | 1–3 days | **Confirm with Lusha account rep that API is included in current plan.** Some SMB tiers lock the API. Get written confirmation before Phase 2 starts. |
| **Ringover** | API key from developer settings | Instant | Available in Ringover dashboard. All plans include API. |
| **Anthropic (Claude)** | API key | Instant | **Claude Pro Desktop ≠ API access.** Register at console.anthropic.com separately. Set up pay-as-you-go billing. This is a hard prerequisite for all agents. |
| **NeverBounce** | API key (free tier) | Instant | 1,000 verifications/month free. No credit card required. |
| **Wappalyzer** | API key (free tier) | Instant | 50 lookups/day free. wappalyzer.com |
| **On-premise server** | Outbound HTTPS to 6 endpoints | Verify now | `api.hubapi.com`, `api.lusha.com`, `api.ringover.com`, `api.anthropic.com`, `api.neverbounce.com`, `api.wappalyzer.com` — Corporate firewalls frequently block outbound HTTPS. **Verify this before writing a line of code.** |
| **LinkedIn Sales Navigator** | CSV export only | N/A | LinkedIn prohibits API automation. Manual export → `/shared/sales_nav_imports/` → file-watcher picks up automatically. |

---

## 11. Cost Breakdown

### 11.1 What You Already Pay (No Change Required)

- HubSpot Professional subscription
- Lusha subscription
- Ringover subscription
- LinkedIn Sales Navigator subscription
- On-premise server hardware

### 11.2 New Monthly Infrastructure Costs

| Item | Monthly cost | Notes |
|---|---|---|
| Claude API — Haiku | €15–40 | Bulk operations: enrichment cache checks, standardisation, report generation, gate decisions |
| Claude API — Sonnet | €5–20 | Complex reasoning: fuzzy name matching, ambiguous qualification scoring, explanation generation |
| NeverBounce | €0 | Free tier (1,000/mo) + 90-day cache. Upgrade trigger: >1,000 new emails/month → ~€8/mo |
| Wappalyzer | €0 | Free tier (50/day) for tech stack gaps |
| PostgreSQL + pgvector | €0 | On-premise |
| sentence-transformers | €0 | On-premise CPU inference |
| Server hosting | €0 | Existing hardware |
| **Total new monthly cost** | **€20–60/month** | |

### 11.3 One-Time Development Cost

| Scenario | Estimate |
|---|---|
| Internal developer (Flask scraper builder, existing salary) | 14 weeks of existing allocation |
| External contract developer (mid-level) | €6,000–14,000 depending on rate |

### 11.4 Estimated Monthly Savings

| Saving | Estimated value |
|---|---|
| Lusha credits (60–80% reduction via cache) | €50–120/month |
| NeverBounce calls (80% reduction via cache) | €10–30/month |
| Sales team time: 15 hours/week recovered at €30/hour | €1,800/month |
| Report preparation: 5 hours/week recovered | €600/month |
| **Total estimated monthly recovery** | **€2,460–2,550/month** |

**ROI:** €20–60 monthly cost → €2,400+ monthly savings. Payback on development cost in under 1 month of operation.

---

## 12. Data Architecture

### 12.1 Core Tables

**`companies`** — canonical company record

```sql
id                    UUID PRIMARY KEY
linkedin_url          TEXT UNIQUE NOT NULL
company_name          TEXT NOT NULL
normalised_name       TEXT
hq_country            TEXT
employee_size_band    TEXT
revenue_band_eur      TEXT
india_presence        BOOLEAN
industry_vertical     TEXT
tech_stack            JSONB
hubspot_id            TEXT
hubspot_status        TEXT
qualification_stage   TEXT
similarity_score      FLOAT
similarity_explanation TEXT
ringover_contacted    BOOLEAN
ringover_call_count   INTEGER
ringover_last_outcome TEXT
ringover_last_contact TIMESTAMP
source                TEXT  -- scraper | sales_nav | manual
first_seen            TIMESTAMP
last_updated          TIMESTAMP
```

**`enrichment_log`**
```sql
id              UUID PRIMARY KEY
company_id      UUID REFERENCES companies
source          TEXT  -- lusha | wappalyzer | neverbounce
enriched_at     TIMESTAMP
data_snapshot   JSONB
credits_used    INTEGER
cache_hit       BOOLEAN
```

**`validation_cache`**
```sql
email           TEXT PRIMARY KEY
valid           BOOLEAN
reason          TEXT
validated_at    TIMESTAMP
```

**`block_list`**
```sql
id              UUID PRIMARY KEY
linkedin_url    TEXT UNIQUE
company_name    TEXT
reason          TEXT
added_by        TEXT  -- agent | rep_name
added_at        TIMESTAMP
```

**`similarity_vectors`** (pgvector)
```sql
id              UUID PRIMARY KEY
company_id      UUID REFERENCES companies
embedding       VECTOR(384)
hubspot_status  TEXT
weight          FLOAT
updated_at      TIMESTAMP
```

**`scrape_runs`**
```sql
run_id            UUID PRIMARY KEY
triggered_at      TIMESTAMP
source            TEXT
companies_found   INTEGER
passed_gate       INTEGER
blocked_blocklist INTEGER
blocked_dedup     INTEGER
blocked_hubspot   INTEGER
errors            JSONB
```

**`segment_library`**
```sql
id              UUID PRIMARY KEY
segment_name    TEXT
hubspot_list_id TEXT
criteria_json   JSONB
created_at      TIMESTAMP
last_used       TIMESTAMP
```

### 12.2 `standards.yaml` Structure

```yaml
company_name:
  case: title
  strip_suffixes_for_matching: [BV, GmbH, Ltd, Inc, NV, SL]
  preserve_suffix_for_display: true

industry_verticals:
  approved:
    - Software & SaaS
    - FinTech
    - E-commerce
    - Logistics & Supply Chain
    - Healthcare IT
    - Manufacturing Tech
    - HR Tech
    - EdTech
  on_unknown: flag  # options: flag | reject | map_to_other

countries:
  NL: [Netherlands, The Netherlands, Holland, nederland]
  DE: [Germany, Deutschland]
  BE: [Belgium, Belgique]
  # ... extend as needed

employee_bands:
  - "1-10"
  - "11-50"
  - "51-200"
  - "201-500"
  - "501-1000"
  - "1000+"

tech_stack_canonical:
  React: [ReactJS, React.js, react]
  Node.js: [NodeJS, Node, node.js]
  PostgreSQL: [Postgres, postgres, PG]
  # ... extend as needed

campaign_tags:
  approved:
    - outbound-q1-2026
    - netherlands-saas
    - india-presence
    - tech-hiring
    # ... extend as needed
  on_unapproved: reject_with_alert
```

---

## 13. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Server outbound firewall blocks APIs | Medium | **Critical** | Verify before any code is written. This blocks every single agent if not resolved. |
| Claude API key not available | Current state | **Critical** | Register at console.anthropic.com now. Separate from Claude Pro Desktop. |
| Lusha API not in current plan | Medium | High | Written confirmation from Lusha account rep before Phase 2 starts. |
| Sales team doesn't maintain master file | Medium | High | One named owner. Automated stale-file alert if >10 days without update. Scraper pauses. |
| Qualification criteria not defined | Medium | High (blocks Phase 4) | 2-hour workshop with sales team in Week 7, before Phase 4 starts. |
| Standards config not defined | Low | Medium (blocks Phase 5) | Workshop in Week 9. Standard drafts can be prepared by architect in advance. |
| Ringover API rate limits | Low | Low | Ringover allows 100 req/min. More than sufficient. |
| NeverBounce free tier exceeded | Low (growth trigger) | Low | Monitor monthly. Upgrade (~€8/mo) if >1,000 new emails/month. |
| pgvector CPU inference too slow | Low | Low | `all-MiniLM-L6-v2` on CPU generates one embedding in ~15ms. 500 companies = ~7.5 seconds. Acceptable. |
| LinkedIn CSV export format changes | Low | Medium | File-watcher parser is configurable. Column mapping in config file, not hardcoded. |
| Scope creep | Medium | Medium | This document is the scope. Changes require formal review. |

---

## 14. Pre-Development Checklist

### Technical Prerequisites (must be done before Week 1)
- [ ] Verify server outbound HTTPS to: `api.hubapi.com`, `api.lusha.com`, `api.ringover.com`, `api.anthropic.com`, `api.neverbounce.com`, `api.wappalyzer.com`
- [ ] Confirm PostgreSQL 15+ and pgvector can be installed (check OS version and permissions)
- [ ] Create HubSpot Private App with all required API scopes
- [ ] Get Lusha API access confirmation **in writing** from account rep
- [ ] Obtain Ringover API key from developer settings
- [ ] Register at console.anthropic.com — set up billing — obtain API key *(separate from Claude Pro Desktop)*
- [ ] Sign up for NeverBounce free tier — obtain API key
- [ ] Sign up for Wappalyzer free tier — obtain API key
- [ ] Export full HubSpot dataset (500+ labeled companies) for Phase 3 bootstrap
- [ ] Define shared folder path for Sales Nav CSV drops: `/shared/sales_nav_imports/`

### Business Prerequisites (staggered — each needed before its phase)
- [ ] **Before Phase 2:** Designate master file owner (one person)
- [ ] **Before Phase 2:** Create initial master file with current target companies
- [ ] **Before Phase 4:** Define lead qualification criteria in writing (signed off by sales lead)
- [ ] **Before Phase 5:** Complete standards workshop (industry taxonomy, tag list, country codes, size bands, tech canonicals)
- [ ] **Before Phase 5:** Export and review existing HubSpot records for retroactive cleanup scope

---

## 15. Out of Scope

Explicitly excluded. Do not add during build without a formal change request and updated timeline estimate.

- **LinkedIn Sales Navigator API automation** — LinkedIn prohibits and bans this. CSV export is the only compliant path.
- **Automated email outreach or sequencing** — This system handles data operations, not sending
- **CRM migration** — HubSpot stays as the system of record
- **Mobile application** — Dashboard is web-only
- **Custom ML model training** — Pre-trained sentence-transformers are sufficient
- **Paid BuiltWith, Clearbit, Apollo, or ZoomInfo** — Not required given current tool stack
- **Multi-country scraping** — Netherlands only in this phase
- **GDPR compliance audit** — Separate legal workstream, must be completed before go-live
- **Docker / containerisation** — Add in a future hardening phase if needed
- **Multi-tenant or multi-user permissions** — Single-organisation deployment

---

*Document version 3.0 — April 2026*  
*Scope: Full system — all 15 problems, all 5 tools, all 3 inbound channels*  
*Supersedes versions 1.0 and 2.0*
