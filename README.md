# Sales Automation — Agentic System Plan
**Prepared for:** Technical Architect  
**Prepared by:** Product / Strategy  
**Date:** April 2026  
**Status:** Planning — Pre-Development

---

## Executive Summary

The sales data team currently loses significant time to 14 identified repetitive tasks: duplicate lead sourcing, redundant enrichment calls, unclean CRM data, inconsistent tagging, manual report prep, and more. Every one of these traces back to four root causes — no deduplication registry, no data standards, no CRM visibility, and no automated pipelines.

The plan is to build an **AI-agentic orchestration layer** on the organisation's existing on-premise server that sits between the company's existing tools (HubSpot CRM, Lusha, NeverBounce, Flask job scraper) and the sales team's workflow. This layer automates repetitive data operations, enforces standards, and — critically — adds a **similarity scoring engine** that ranks newly scraped companies by their likelihood of converting, based on historical CRM outcomes.

The existing Flask-based job scraper (already built, scrapes 3 Netherlands job boards) is the pipeline entry point and will be integrated as Phase 1 priority. A **sales-maintained master file** of pre-approved companies will gate what gets scraped, eliminating wasted downstream effort entirely.

**Total estimated infrastructure cost post-build:** €50–170/month (Claude API + NeverBounce; everything else runs on-premise at zero marginal cost).  
**Estimated development time:** 12 weeks for 1 developer across 4 phases.  
**HubSpot plan:** Professional — API fully included, no upgrades needed.  
**Similarity scoring:** Ready for accuracy from Day 1 — 500+ labeled companies already in HubSpot.  
**Risk level:** Low-to-medium. All tasks are technically feasible except LinkedIn Sales Navigator automation, which must not be attempted.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [System Overview](#2-system-overview)
3. [Core Components](#3-core-components)
4. [Phase-by-Phase Development Plan](#4-phase-by-phase-development-plan)
5. [Feasibility Assessment](#5-feasibility-assessment)
6. [Tech Stack](#6-tech-stack)
7. [API Access Requirements](#7-api-access-requirements)
8. [Cost Breakdown](#8-cost-breakdown)
9. [Data Architecture](#9-data-architecture)
10. [Risks and Mitigations](#10-risks-and-mitigations)
11. [Pre-Development Checklist](#11-pre-development-checklist)
12. [Out of Scope](#12-out-of-scope)

---

## 1. Problem Statement

The sales data team has identified 14 recurring inefficiencies. They are grouped below by root cause:

### 1.1 Deduplication failures
| # | Task | Symptom | Source |
|---|---|---|---|
| 1 | Lead sourcing | Same accounts sourced across campaigns | HubSpot, campaign planning |
| 2 | Lead enrichment | Same lead enriched multiple times | Lusha, no enrichment tracking |
| 3 | CRM upload | Duplicate contacts/accounts | HubSpot import, no dedup rules |
| 9 | Data validation | Emails re-validated repeatedly | NeverBounce/ZeroBounce overuse |

### 1.2 Standardisation failures
| # | Task | Symptom | Source |
|---|---|---|---|
| 4 | Data cleaning | Re-cleaning already standardised data | No versioned clean dataset |
| 15 | Data formatting | Re-formatting names, industries, fields | No formatting ruleset |
| 11 | Campaign tagging | Tags corrected repeatedly | No tagging governance |
| 14 | Tech stack research | Re-checking company tech data | BuiltWith/Wappalyzer overlap |

### 1.3 Visibility failures
| # | Task | Symptom | Source |
|---|---|---|---|
| 5 | Lead qualification | Re-checking qualified leads | CRM visibility gaps |
| 6 | Campaign list building | Lists built from scratch repeatedly | No list library/reuse framework |
| 7 | Data segmentation | Re-segmenting same dataset | No saved filters in CRM |
| 13 | Tool extraction | Pulling same data repeatedly | Sales Nav, overlapping tools |

### 1.4 Reporting failures
| # | Task | Symptom | Source |
|---|---|---|---|
| 10 | Report data prep | Raw data re-prepared for every report | No dashboards, Excel dependency |

---

## 2. System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ON-PREMISE SERVER                                │
│                                                                     │
│  ┌──────────────┐        ┌───────────────────────────────────────┐  │
│  │ Master File  │──────▶ │         Flask Job Scraper             │  │
│  │ (weekly, by  │        │  3 NL job boards · tech stack · block │  │
│  │  sales team) │        │  list · master file gate              │  │
│  └──────────────┘        └──────────────┬────────────────────────┘  │
│                                         │                           │
│                                         ▼                           │
│                          ┌──────────────────────────┐              │
│                          │      Agent Gate           │              │
│                          │  · Block list check       │              │
│                          │  · Dedup registry check   │              │
│                          │  · Already in HubSpot?    │              │
│                          └──────────────┬────────────┘              │
│                                         │                           │
│                    ┌────────────────────┼────────────────────┐      │
│                    │                    │                    │      │
│                    ▼                    ▼                    ▼      │
│         ┌──────────────┐   ┌──────────────────┐  ┌───────────────┐ │
│         │  Enrichment  │   │   Similarity     │  │ Standardisa-  │ │
│         │    Agent     │   │  Scoring Agent   │  │ tion Agent    │ │
│         │ Lusha · NB   │   │  pgvector DB     │  │ Format rules  │ │
│         └──────┬───────┘   └────────┬─────────┘  └──────┬────────┘ │
│                │                    │                    │          │
│                └────────────────────┼────────────────────┘          │
│                                     │                               │
│                                     ▼                               │
│                     ┌───────────────────────────┐                  │
│                     │    Sales Dashboard (Flask) │                  │
│                     │  Ranked by match score     │                  │
│                     │  "Similar to X company"    │                  │
│                     │  Accept / Reject / Block   │                  │
│                     └──────────┬────────────────┘                  │
│                                │                                    │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
       ┌────────────┐   ┌──────────────┐   ┌─────────────────┐
       │  HubSpot   │   │ Similarity   │   │  Reporting      │
       │    CRM     │   │  DB updated  │   │  Agent (auto)   │
       └────────────┘   └──────────────┘   └─────────────────┘
```

### Key design principles

- **On-premise first.** The orchestrator, database, pgvector, and all agents run on the organisation's existing CPU server. No cloud hosting costs.
- **Master file as the gate.** The sales team maintains a weekly-updated list of pre-approved companies. The scraper only operates on this list. This inverts the old logic (scrape everything, filter later) to the correct logic (filter first, scrape only what matters).
- **Dedup before action.** No enrichment call, validation call, or CRM write happens without first checking the dedup registry. This is the single biggest cost saver.
- **Similarity scoring is additive, not a blocker.** The scoring system ranks companies in the dashboard but does not auto-reject. The sales team retains full accept/reject authority. Over time, their decisions feed back into the model and improve scoring accuracy.
- **Human-in-the-loop is preserved.** No company reaches HubSpot without a human accept decision. Automation handles data operations; humans handle judgment calls.

---

## 3. Core Components

### 3.1 Master File (Sales-Maintained)
A structured spreadsheet (Google Sheets or Excel, synced to server) maintained weekly by one designated sales team member.

**Required fields:**
- `company_name` — display name
- `linkedin_url` — primary unique identifier (not company name — names have variants)
- `hq_location` — country/city
- `employee_size_band` — e.g. 50–200, 200–500, 500–1000
- `revenue_band_EUR_M` — e.g. <10, 10–50, 50–200
- `india_presence` — boolean (yes/no)
- `industry_vertical` — standardised taxonomy (defined once, enforced)
- `active_for_scraping` — boolean toggle (pause without deleting)
- `last_updated` — date

**Governance rule:** One named owner. Updated every Monday before the weekly scrape run. The scraper will not run on stale data older than 10 days — it will alert and wait.

### 3.2 Flask Job Scraper (Existing — Extended)
Already built. Scrapes 3 Netherlands-based job boards based on custom inputs.  
Existing features retained: block list, tech stack auto-fill, HubSpot status upload.

**Extensions required:**
- Add a `/scrape` POST endpoint so the orchestrator can trigger it programmatically
- Before scraping, validate company against master file (active_for_scraping = true)
- After scraping, emit a structured JSON payload per company to the agent gate
- Accept a `run_id` parameter for traceability

### 3.3 Agent Gate
First stop for every company output from the scraper. Runs three checks synchronously before passing the company downstream.

**Check 1 — Block list:** Is this company on the forbidden list? If yes, discard silently and log.  
**Check 2 — Dedup registry:** Has this company been processed in the last N days? (configurable, default 60 days.) If yes, skip enrichment and validation — reuse cached data.  
**Check 3 — HubSpot presence:** Does this company already exist in HubSpot with a terminal status (not interested / rejected)? If yes, discard and log.

Only companies that pass all three checks proceed to enrichment and scoring.

### 3.4 Enrichment Agent
Calls Lusha API and/or Wappalyzer API to fill in missing fields. Writes results to the dedup registry with a timestamp.

**Logic:**
```
IF company in dedup_registry AND last_enriched < 60 days:
    use cached data, skip API call
ELSE:
    call Lusha for contact data
    call Wappalyzer for tech stack
    write to dedup_registry with timestamp
```

This single rule eliminates the majority of redundant Lusha credit spend.

**Email validation (NeverBounce):**  
Only called once per email address. Results cached in dedup registry. Never re-validated unless the record is older than 90 days.

### 3.5 Similarity Scoring Agent
Compares each new company against the historical pool of accepted/successful companies from HubSpot using vector similarity search.

**How it works:**
1. At setup, generate feature vectors for all HubSpot companies that have a status (short-term, long-term, not interested, rejected). Store in pgvector.
2. For each new company, generate its feature vector using the same fields.
3. Query pgvector for the top 5 nearest neighbours.
4. Compute a match score (0–100). Weight positive statuses (long-term = 1.0, short-term = 0.8) and penalise negative ones (rejected = -0.5).
5. Return: score, top 3 "similar to" companies with their names and statuses, and a one-line explanation.

**Feature vector fields (inputs to embedding):**
- Industry vertical (encoded)
- Employee size band (encoded)
- HQ country (encoded)
- Revenue band (encoded)
- India presence (boolean)
- Tech stack flags (top 10 technologies as binary features)
- Job function of roles being hired (from scraper output)

**Important caveat on accuracy:** With fewer than 150 labeled companies in HubSpot, scores will be rough. The system becomes reliably useful at 150–200+ labeled records. If current HubSpot data is below this threshold, the team should retroactively label all historical companies before go-live. This is a 2–3 hour one-time task — it directly determines Day 1 scoring quality.

**Model choice:** No external ML service needed. Use `sentence-transformers` (open-source, runs on CPU) for embedding generation. pgvector handles similarity search. Total compute cost: near zero on the on-premise server.

### 3.6 Standardisation Agent
Runs on every company record before it is written to HubSpot. Applies a fixed ruleset defined in a config file.

**What it standardises:**
- Company name normalisation (title case, remove Ltd/BV/GmbH suffixes for matching, preserve for display)
- Industry vertical mapping to a fixed taxonomy (defined once by sales team before build)
- Employee size band bucketing
- Country name standardisation (Netherlands → NL, The Netherlands → NL, etc.)
- Campaign tag enforcement — only tags from an approved list are permitted
- Tech stack field normalisation (React.js → React, NodeJS → Node.js, etc.)

**Config approach:** All rules live in a single `standards.yaml` file. The sales team can update rules without touching code. The developer never hardcodes field values.

### 3.7 Reporting Agent
Runs on a schedule (configurable, default: every Monday 08:00). Queries HubSpot via API, assembles a structured report, and delivers it via the configured channel (Slack webhook, email, or writes to a shared folder).

**Report contents:**
- New companies added this week (by status)
- Pipeline movement (companies that changed status)
- Enrichment call usage (Lusha credits consumed vs saved by cache)
- Scraper run summary (total scraped, passed gate, rejected at gate, reason breakdown)
- Top 5 highest-scoring new companies from the week

**No Excel. No manual prep. No human intervention required.**

---

## 4. Phase-by-Phase Development Plan

> **Assumption:** 1 developer (the existing Flask scraper builder). Timelines are working-week estimates with normal interruptions factored in.  
> **HubSpot plan:** Professional — API access is fully included, no upgrades needed.  
> **Labeled companies in HubSpot:** 500+ — similarity scoring will be accurate from Day 1. No retroactive labeling session required before Phase 2.

---

### Phase 1 — Foundation (Weeks 1–3)
**Goal:** Stop the bleeding. Eliminate duplicate enrichment, duplicate CRM uploads, and invalid scraping targets.

| Task | Detail | Effort |
|---|---|---|
| On-premise environment setup | Install Python 3.11+, PostgreSQL + pgvector extension, set up virtualenv, configure outbound firewall for required API endpoints | 2 days |
| Dedup registry schema | Design and create PostgreSQL tables: `companies`, `enrichment_log`, `validation_cache`, `block_list` | 1 day |
| Master file schema + sync | Define fields, create sync script (CSV/Google Sheets → DB, runs on schedule) | 2 days |
| Agent gate — block list check | Implement check against block_list table | 1 day |
| Agent gate — dedup check | Implement 60-day enrichment cache check | 1 day |
| Agent gate — HubSpot presence check | HubSpot API integration: search companies by LinkedIn URL | 2 days |
| Enrichment agent — Lusha integration | API wrapper with cache-first logic | 2 days |
| Email validation — NeverBounce | API wrapper with 90-day cache | 1 day |
| Scraper extension | Add `/scrape` POST endpoint, master file gate, structured JSON output | 2 days |
| Integration testing | End-to-end test: scraper → gate → enrichment → dedup registry | 2 days |

**Phase 1 total: ~3 weeks**  
**Deliverable:** A gated, dedup-safe enrichment pipeline. Lusha credit waste stops. CRM duplicate uploads stop.

---

### Phase 2 — Similarity Scoring (Weeks 4–6)
**Goal:** Give the sales team a ranked, explained view of new companies in the dashboard.

| Task | Detail | Effort |
|---|---|---|
| HubSpot historical data import | Pull all companies with status labels from HubSpot. Clean and normalise. | 2 days |
| Feature vector design | Define and encode all feature fields. Validate with sales team that fields are meaningful. | 1 day |
| Embedding generation | Run `sentence-transformers` on existing HubSpot companies. Store in pgvector. | 2 days |
| Similarity scoring agent | Query pgvector for top 5 neighbours, compute weighted score, generate explanation string | 3 days |
| Dashboard integration | Add score column, "similar to" display, and sort-by-score to existing Flask dashboard | 3 days |
| Feedback loop | On reject in dashboard, write rejection to pgvector with negative weight; option to add to block list | 2 days |
| Testing + calibration | Validate scores against known good/bad companies. Adjust weights. | 2 days |

**Phase 2 total: ~3 weeks**  
**Deliverable:** Sales dashboard shows ranked companies with match scores and explanations. Feedback loop is live.

---

### Phase 3 — Standardisation (Weeks 7–9)
**Goal:** Enforce data standards at the point of entry. Clean CRM from this point forward.

| Task | Detail | Effort |
|---|---|---|
| Define standards config | Work with sales team to define `standards.yaml`: industry taxonomy, tag list, country codes, tech stack normalisations | 2 days (collaborative) |
| Standardisation agent | Build config-driven formatting pipeline. Runs on every record before HubSpot write. | 3 days |
| Campaign tagging enforcement | HubSpot property validation — only approved tags permitted | 2 days |
| Tech stack normalisation | Map raw tech strings to canonical names | 1 day |
| Wappalyzer integration | Replace or supplement BuiltWith with Wappalyzer API (free tier: 50 req/day) | 1 day |
| Retroactive CRM cleanup | Run standardisation agent against existing HubSpot records (one-time batch job) | 2 days |
| QA with sales team | Review sample of cleaned records for accuracy | 1 day |

**Phase 3 total: ~2.5 weeks**  
**Deliverable:** Every record entering HubSpot is clean, tagged, and standardised. Retroactive cleanup of existing data.

---

### Phase 4 — Reporting Automation (Weeks 10–11)
**Goal:** Zero manual report prep. Automated weekly pipeline summary.

| Task | Detail | Effort |
|---|---|---|
| HubSpot reporting queries | Build API queries for pipeline status, movement, new additions | 2 days |
| Report assembly agent | Claude API (Haiku) generates natural-language summary from raw query data | 1 day |
| Enrichment usage report | Query dedup registry for credits used vs cached | 1 day |
| Delivery integration | Slack webhook or email delivery. Configurable. | 1 day |
| Scheduling | Cron job on server. Monday 08:00 default. Configurable. | 0.5 days |
| Testing | Run 3 consecutive weekly reports, validate accuracy | 1 day |

**Phase 4 total: ~1.5 weeks**  
**Deliverable:** Automated weekly report delivered every Monday. No Excel. No manual prep.

---

### Buffer + Hardening (Week 12)
Regression testing across all phases, documentation, deployment checklist, monitoring setup (simple log-based alerting for agent failures), handover.

---

### Summary Timeline

```
Week 1–3:   Phase 1 — Foundation (dedup, gate, enrichment cache)
Week 4–6:   Phase 2 — Similarity scoring + dashboard ranking
Week 7–9:   Phase 3 — Standardisation + retroactive CRM cleanup
Week 10–11: Phase 4 — Reporting automation
Week 12:    Hardening, documentation, handover
```

**Total: 12 weeks for 1 developer**

---

## 5. Feasibility Assessment

| Task | Feasibility | Condition / Note |
|---|---|---|
| Dedup registry | 100% | No external dependencies |
| Enrichment gating (Lusha cache) | 100% | Requires Lusha API access on current plan |
| CRM dedup on upload | 100% | HubSpot duplicate rules must be configured in HubSpot settings first |
| Scraper integration | 100% | Small extension to existing Flask app |
| Block list / forbidden companies | 100% | Already partially exists in scraper |
| Data formatting / standardisation | 100% | Standards config must be defined by sales team before development starts |
| Campaign tagging enforcement | 100% | Approved tag list must be defined before development starts |
| Tech stack normalisation | 100% | Canonical name mapping defined once in config |
| Email validation caching | 100% | NeverBounce starter plan required |
| Lead qualification scoring | 100% | Requires qualification criteria defined by sales team upfront |
| Similarity scoring engine | 100% | Accuracy improves with more labeled data — see note below |
| Reporting automation | 100% | HubSpot API + Claude Haiku |
| Sales Navigator automation | **DO NOT ATTEMPT** | LinkedIn actively detects and bans automated access. Use CSV export + manual ingestion only. |

**Note on similarity scoring accuracy:** With 500+ labeled companies already in HubSpot, the scoring engine will be meaningfully accurate from Day 1. No retroactive labeling session is required. The model will continue improving passively as the sales team makes more accept/reject decisions through the dashboard.

---

## 6. Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Orchestrator language | Python 3.11+ | Existing scraper is Flask/Python — same ecosystem |
| Web framework (dashboard + API) | Flask | Existing — no migration needed |
| Database | PostgreSQL 15+ | Reliable, runs on-premise, supports pgvector |
| Vector search | pgvector (Postgres extension) | No separate vector DB needed; runs on same server |
| Embedding model | `sentence-transformers` (`all-MiniLM-L6-v2`) | Open-source, runs on CPU, no GPU required |
| AI reasoning (agent tasks) | Claude API — `claude-haiku-4` for bulk, `claude-sonnet-4` for complex reasoning | Haiku at ~$0.25/MTok for 90% of tasks |
| Job scheduler | Python `APScheduler` or system `cron` | Simple, no additional infrastructure |
| HubSpot integration | HubSpot REST API v3 | Official API, stable |
| Enrichment | Lusha REST API | Existing subscription |
| Email validation | NeverBounce API | Existing or starter plan |
| Tech stack lookup | Wappalyzer API (free tier) | Replaces BuiltWith — 50 lookups/day free |
| Standards config | YAML | Human-readable, editable without code changes |
| Logging | Python `logging` + file rotation | Simple, on-premise, no external service |

**What is deliberately not used:**
- No n8n, Zapier, or Make — these charge per-task and add an abstraction layer with no benefit for custom logic
- No cloud vector database (Pinecone, Weaviate) — pgvector on-premise is sufficient
- No separate ML training infrastructure — `sentence-transformers` CPU inference is fast enough at this data volume
- No Docker initially — add containerisation in a later hardening pass if needed

---

## 7. API Access Requirements

These must be secured **before development starts**. Some have approval delays.

| Tool | What to request | Lead time | Action required |
|---|---|---|---|
| HubSpot | Private App API key | Instant | Create in HubSpot > Settings > Integrations > Private Apps. Request scopes: `crm.objects.contacts.read/write`, `crm.objects.companies.read/write`, `crm.lists.read/write`, `crm.schemas.read` |
| Lusha | API access confirmation | 1–3 days | Confirm with Lusha account rep that your current plan tier includes API access. Some SMB plans have API locked behind a higher tier. Get this in writing. |
| NeverBounce | API key | Instant | Requires paid starter plan. Free tier rate limits are too low for production use. |
| Wappalyzer | API key | Instant | Free tier: 50 lookups/day. Sign up at wappalyzer.com. Upgrade only if volume requires. |
| Anthropic (Claude) | API key | Instant | Sign up at console.anthropic.com. Start on pay-as-you-go. |
| On-premise server | Outbound HTTPS access | Verify now | Confirm the server can make outbound HTTPS calls to: `api.hubapi.com`, `api.lusha.com`, `api.neverbounce.com`, `api.wappalyzer.com`, `api.anthropic.com`. Corporate firewalls often block these. This must be verified before a single line of code is written. |
| LinkedIn Sales Navigator | CSV export only | N/A | Do not request API access. LinkedIn has no viable official API for the data needed. Process: manual export from Sales Nav → drop CSV into a shared folder → ingestion agent picks it up on a schedule. |

---

## 8. Cost Breakdown

### 8.1 Infrastructure — Monthly (Post-Build)

| Item | Cost/Month | Notes |
|---|---|---|
| On-premise server (existing) | €0 | All agents, DB, pgvector, orchestrator run here |
| Claude API — Haiku (bulk tasks) | ~€15–40 | Dedup checks, formatting, report generation. Haiku at ~€0.23/MTok input. |
| Claude API — Sonnet (reasoning) | ~€10–30 | Fuzzy matching, qualification scoring. Sonnet selectively. |
| NeverBounce | ~€10–40 | With 90-day cache, validation volume drops ~80% vs current |
| Wappalyzer | €0 | Free tier sufficient at current scrape volumes |
| **Total new infrastructure** | **~€35–110/month** | Does not include existing Lusha, HubSpot subscriptions |

### 8.2 Development Cost (One-Time)

| Scenario | Estimate |
|---|---|
| Internal developer (existing Flask developer) | 12 weeks of existing salary allocation |
| External developer (mid-level, contract) | €6,000–12,000 depending on rate and scope creep |

### 8.3 Cost Savings (Estimated Monthly)

| Saving | Basis |
|---|---|
| Lusha credits (dedup cache) | 60–80% reduction in redundant enrichment calls |
| NeverBounce calls (validation cache) | ~80% reduction in redundant validation calls |
| Sales team time (manual data tasks) | Estimated 8–15 hours/week recovered across the team |
| Report preparation time | ~3–5 hours/week recovered |

---

## 9. Data Architecture

### 9.1 Core Tables (PostgreSQL)

**`companies`** — canonical record per company
```
id                  UUID PRIMARY KEY
linkedin_url        TEXT UNIQUE NOT NULL       -- primary identifier
company_name        TEXT
normalised_name     TEXT                       -- for dedup matching
hq_country          TEXT
employee_size_band  TEXT
revenue_band        TEXT
india_presence      BOOLEAN
industry_vertical   TEXT
tech_stack          JSONB
hubspot_id          TEXT
hubspot_status      TEXT                       -- short-term, long-term, rejected, etc.
first_seen          TIMESTAMP
last_updated        TIMESTAMP
```

**`enrichment_log`** — tracks every enrichment event
```
id                  UUID PRIMARY KEY
company_id          UUID REFERENCES companies
source              TEXT                       -- lusha, wappalyzer, manual
enriched_at         TIMESTAMP
data_snapshot       JSONB
credits_used        INTEGER
cache_hit           BOOLEAN
```

**`validation_cache`** — email validation results
```
email               TEXT PRIMARY KEY
valid               BOOLEAN
validated_at        TIMESTAMP
source              TEXT                       -- neverbounce
```

**`block_list`** — forbidden companies
```
id                  UUID PRIMARY KEY
linkedin_url        TEXT
company_name        TEXT
reason              TEXT
added_by            TEXT
added_at            TIMESTAMP
```

**`similarity_vectors`** — managed by pgvector
```
id                  UUID PRIMARY KEY
company_id          UUID REFERENCES companies
embedding           VECTOR(384)               -- all-MiniLM-L6-v2 dimensions
hubspot_status      TEXT
weight              FLOAT                     -- 1.0 long-term, 0.8 short-term, -0.5 rejected
updated_at          TIMESTAMP
```

**`scrape_runs`** — audit trail
```
run_id              UUID PRIMARY KEY
triggered_at        TIMESTAMP
companies_scraped   INTEGER
passed_gate         INTEGER
blocked             INTEGER
dedup_skipped       INTEGER
enriched            INTEGER
errors              JSONB
```

### 9.2 Master File → Database Sync

The master file (Google Sheets or Excel) is the source of truth for scraping targets. A sync script runs on schedule (daily, or on-demand before a scrape run). It does not overwrite HubSpot-sourced data — it only manages the `active_for_scraping` flag and the fields that sales maintain manually.

### 9.3 Similarity Score Computation

```python
# Pseudocode — not final implementation
def score_company(new_company):
    vector = generate_embedding(new_company)
    neighbours = pgvector.query(vector, top_k=5)
    
    score = 0
    explanations = []
    for neighbour in neighbours:
        score += neighbour.similarity * neighbour.weight
        explanations.append(f"{neighbour.company_name} ({neighbour.hubspot_status})")
    
    normalised_score = min(100, int(score * 100))
    explanation = f"Similar to: {', '.join(explanations[:3])}"
    
    return normalised_score, explanation
```

---

## 10. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Outbound firewall blocks API endpoints | Medium | High — blocks entire system | Verify outbound HTTPS to all required domains before development starts. This is the single most important pre-build check. |
| Lusha API not included in current plan | Medium | Medium — Phase 1 delay | Confirm with Lusha account rep in week 1 before building the enrichment agent |
| Master file not maintained consistently | Medium | High — degrades scraping quality over time | Assign a single named owner. Build an alert if the file is older than 10 days when a scrape is triggered. |
| Similarity scoring is low quality at launch | **Eliminated** | 500+ labeled companies in HubSpot — scoring will be accurate from Day 1 |
| Sales team rejects the tool | Low | High | Involve sales team in defining standards (Phase 3 config), qualification criteria, and the master file schema. Ownership drives adoption. |
| LinkedIn-related issues | Low (if scoped correctly) | High | Do not automate Sales Navigator. Scope is Netherlands job board scraping only, which is the existing tool. |
| Scraper detected and blocked by job boards | Low-Medium | Medium | Add rate limiting, randomised delays, and user-agent rotation to the existing scraper as part of Phase 1 extension work |
| Scope creep | Medium | Medium | This document is the scope. Any new task goes through a formal change request before being added to a phase. |

---

## 11. Pre-Development Checklist

The following must be completed before the first line of code is written. Several items require collaboration between the technical architect, the sales team lead, and tool vendors.

- [ ] **Verify server outbound internet access** to: `api.hubapi.com`, `api.lusha.com`, `api.neverbounce.com`, `api.wappalyzer.com`, `api.anthropic.com`
- [ ] **Confirm Lusha API access** is included in current plan (contact Lusha account rep)
- [ ] **Create HubSpot Private App** with required scopes — Professional plan fully supports this (see Section 7)
- [ ] **Sign up for NeverBounce** starter paid plan and obtain API key
- [ ] **Sign up for Wappalyzer** and obtain free API key
- [ ] **Obtain Anthropic API key** from console.anthropic.com
- [ ] **Export full HubSpot company dataset** with status labels for similarity scoring bootstrap — 500+ records already labeled, ready to use
- [ ] **Sales team: define industry vertical taxonomy** (the fixed list that standardisation will enforce)
- [ ] **Sales team: define approved campaign tag list**
- [ ] **Sales team: define lead qualification criteria** (written, agreed, signed off)
- [ ] **Designate master file owner** — one named person responsible for weekly updates
- [ ] **Define master file schema** and create initial version with current target companies
- [ ] **Verify PostgreSQL can be installed** on on-premise server (check OS version, permissions)
- [ ] **Install pgvector extension** — confirm it is compatible with the server's PostgreSQL version

---

## 12. Out of Scope

The following are explicitly excluded from this plan. They should not be added mid-build without a formal scope change.

- **LinkedIn Sales Navigator automation** — prohibited due to LinkedIn's terms of service and ban risk
- **Automated email outreach or sequencing** — this system handles data, not outreach
- **CRM migrations** — HubSpot remains the CRM of record; no migration planned
- **Mobile application** — the dashboard is a web interface
- **Custom ML model training** — `sentence-transformers` pre-trained models are sufficient; no custom training pipeline
- **Third-party sales tools** (Apollo, ZoomInfo, Clearbit) — not in current stack; add only if Lusha proves insufficient
- **Multi-country scraping** — current scope is Netherlands only; expand in a future phase
- **GDPR compliance audit** — flagged as important but handled separately by legal/compliance, not in this build

---

*Document version 1.0 — April 2026*  
*Next review: after Phase 1 completion*
