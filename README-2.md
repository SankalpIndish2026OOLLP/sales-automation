# Sales Automation — Agentic System Plan
**Organisation:** Eastern Enterprise  
**Prepared for:** Technical Architect  
**Date:** April 2026  
**Version:** 2.0 — Constrained to 4 Core Tools  
**Status:** Planning — Pre-Development

---

## Executive Summary

The Eastern Enterprise sales data team has 15 identified repetitive tasks destroying pipeline efficiency. This plan builds an **AI-agentic orchestration layer** on the existing on-premise CPU server that connects four core tools — **HubSpot CRM, Lusha, LinkedIn Sales Navigator, and Ringover** — into a single coordinated system. All other tools used are free-tier only.

The existing Flask job scraper (scrapes 3 Netherlands job boards) is the pipeline entry point. A **sales-maintained master file** gates what gets scraped. A **similarity scoring engine** (powered by 500+ already-labeled HubSpot companies) ranks new companies by conversion likelihood from Day 1. Ringover call history feeds directly into lead qualification scoring — a company that's been called 6 times with no response scores differently than one never contacted.

**Honest answer on all 15 problems:** 13 of 15 are fully resolved. 2 have partial resolution with known limitations (see Section 5).

**Core tool stack:** HubSpot Professional API · Lusha API · Ringover API · Anthropic Claude API (key required — not yet available) · Flask scraper (existing) · PostgreSQL + pgvector (on-premise)

**Total new monthly infrastructure cost:** €40–100/month (Claude API + NeverBounce free tier management). Everything else runs on-premise or is already paid.  
**Development time:** 12 weeks, 1 developer.  
**Similarity scoring accuracy:** Day 1 — 500+ labeled companies already in HubSpot.

---

## Table of Contents

1. [The 15 Problems — Honest Resolution Status](#1-the-15-problems--honest-resolution-status)
2. [Why These 4 Tools Are Sufficient](#2-why-these-4-tools-are-sufficient)
3. [System Architecture](#3-system-architecture)
4. [Core Agent Components](#4-core-agent-components)
5. [Feasibility Assessment](#5-feasibility-assessment)
6. [Phase-by-Phase Development Plan](#6-phase-by-phase-development-plan)
7. [Tech Stack](#7-tech-stack)
8. [API Access Requirements](#8-api-access-requirements)
9. [Cost Breakdown — Frank and Open](#9-cost-breakdown--frank-and-open)
10. [Data Architecture](#10-data-architecture)
11. [Risks and Mitigations](#11-risks-and-mitigations)
12. [Pre-Development Checklist](#12-pre-development-checklist)
13. [Out of Scope](#13-out-of-scope)

---

## 1. The 15 Problems — Honest Resolution Status

| # | Problem | Resolution | How |
|---|---|---|---|
| 1 | Lead sourcing duplicates | ✅ **Fully resolved** | Dedup registry checks LinkedIn URL before any scrape or CRM write |
| 2 | Lead enrichment duplicates | ✅ **Fully resolved** | 60-day enrichment cache — Lusha API only called if no cached record |
| 3 | CRM upload duplicates | ✅ **Fully resolved** | Agent gate checks HubSpot before any write; fuzzy name matching for variants |
| 4 | Data re-cleaning | ✅ **Fully resolved** | Standardisation agent enforces rules at point of entry; one-time retroactive batch clean |
| 5 | Lead re-qualification | ✅ **Fully resolved** | Ringover call log + HubSpot status written back automatically; clear `qualification_stage` field with timestamp |
| 6 | Campaign list rebuilding | ✅ **Fully resolved** | HubSpot saved lists created and managed by agent; reuse framework enforced |
| 7 | Data re-segmentation | ✅ **Fully resolved** | Saved filters + segment library maintained in HubSpot by agent |
| 9 | Email re-validation | ✅ **Fully resolved** | 90-day validation cache — NeverBounce free tier (1,000/month) with aggressive caching makes this viable |
| 10 | Report data re-prep | ✅ **Fully resolved** | Automated weekly report from HubSpot API + Ringover API; no Excel |
| 11 | Campaign re-tagging | ✅ **Fully resolved** | Approved tag taxonomy enforced at entry; no unapproved tags can be written |
| 13 | Repeated tool extraction | ⚠️ **Partially resolved** | Sales Nav: CSV export only (no API — see note). All other tools fully automated. |
| 14 | Tech stack re-research | ✅ **Fully resolved** | Flask scraper already extracts tech stack from job postings. Cached in dedup registry. Wappalyzer free tier (50/day) fills gaps. |
| 15 | Data re-formatting | ✅ **Fully resolved** | `standards.yaml` config enforces all field formats at write time |
| — | *New: Company similarity scoring* | ✅ **Added** | Similarity engine ranks new companies by match to historical successes |
| — | *New: Ringover qualification signals* | ✅ **Added** | Call frequency, answer rate, and call outcomes feed qualification scoring |

**Problem 8** was not in the original list (numbering gap in source document).

### The 2 Partial Resolutions — Being Candid

**Problem 13 (Tool extraction — Sales Navigator):**  
LinkedIn does not provide a viable programmatic API for the company/contact data the sales team needs. Any attempt to automate browser sessions on LinkedIn risks account suspension. The only safe approach is: manual CSV export from Sales Nav → drop into a shared folder → ingestion agent picks it up and processes it automatically. The manual export step cannot be removed. This is a LinkedIn constraint, not a build choice.

**NeverBounce free tier (Problem 9):**  
The free tier gives 1,000 verifications/month. With the 90-day validation cache, this is sufficient for most operating volumes. If the team scales to validating more than 1,000 new email addresses per month, a paid plan (~€8/month) will be required. Flag this as a growth trigger, not a current blocker.

---

## 2. Why These 4 Tools Are Sufficient

| Tool | Role in system | What it replaces |
|---|---|---|
| **HubSpot Professional** | CRM of record, list management, campaign tracking, reporting data | BuiltWith (company fields), ZeroBounce (dedup via CRM search), manual Excel dashboards |
| **Lusha** | Contact enrichment — phone, email, LinkedIn data | Apollo, ZoomInfo, manual LinkedIn research |
| **LinkedIn Sales Navigator** | Company research and lead identification (manual export only) | No automation possible; CSV export is the only safe integration |
| **Ringover** | Call logs, call outcomes, engagement history | Manual call tracking, CRM activity notes |
| **Flask scraper (existing)** | Pipeline entry point — scrapes NL job boards, extracts tech stack | BuiltWith (tech stack), manual sourcing |
| **Claude API (Haiku/Sonnet)** | Agent reasoning, fuzzy matching, report generation, scoring | Manual data operations |
| **Wappalyzer free tier** | Tech stack supplement where scraper gaps exist | BuiltWith paid subscription |
| **NeverBounce free tier** | Email validation with caching | ZeroBounce paid, NeverBounce paid |

**Tools explicitly not added:** BuiltWith paid, ZeroBounce paid, Apollo, Clearbit, Zapier, n8n, Make. None are needed given the above stack.

---

## 3. System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         ON-PREMISE SERVER                                │
│                                                                          │
│  ┌─────────────────┐    ┌───────────────────────────────────────────┐   │
│  │   Master File   │───▶│           Flask Job Scraper               │   │
│  │  (sales-owned,  │    │  3 NL job boards · tech stack extraction  │   │
│  │  weekly update) │    │  block list · master file gate            │   │
│  └─────────────────┘    └────────────────────┬──────────────────────┘   │
│                                              │                           │
│  ┌─────────────────┐                         │                           │
│  │ Sales Nav CSV   │─────────────────────────┤                           │
│  │ (manual export  │                         │                           │
│  │  → shared folder│                         │                           │
│  └─────────────────┘                         │                           │
│                                              ▼                           │
│                           ┌──────────────────────────────┐              │
│                           │         AGENT GATE           │              │
│                           │  · Block list check          │              │
│                           │  · Dedup registry check      │              │
│                           │  · HubSpot presence check    │              │
│                           │  · Ringover: prior contact?  │              │
│                           └──────────────┬───────────────┘              │
│                                          │                               │
│             ┌────────────────────────────┼───────────────────┐          │
│             │                            │                   │          │
│             ▼                            ▼                   ▼          │
│  ┌──────────────────┐   ┌───────────────────────┐  ┌──────────────────┐ │
│  │ Enrichment Agent │   │  Similarity Scoring   │  │ Standardisation  │ │
│  │                  │   │       Agent           │  │     Agent        │ │
│  │  Lusha API       │   │  pgvector · 500+      │  │  standards.yaml  │ │
│  │  (cache-first)   │   │  labeled companies    │  │  tag enforcement │ │
│  │  NeverBounce     │   │  Ringover signals     │  │  format rules    │ │
│  │  (90-day cache)  │   └───────────────────────┘  └──────────────────┘ │
│  └──────────────────┘                                                    │
│             │                            │                   │          │
│             └────────────────────────────┼───────────────────┘          │
│                                          ▼                               │
│                     ┌────────────────────────────────────┐              │
│                     │     Sales Dashboard (Flask UI)     │              │
│                     │  Companies ranked by match score   │              │
│                     │  "Similar to X (long-term client)" │              │
│                     │  Accept · Reject · Add to blocklist│              │
│                     └──────────────┬─────────────────────┘              │
│                                    │                                     │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │
             ┌───────────────────────┼────────────────────────┐
             │                       │                        │
             ▼                       ▼                        ▼
      ┌────────────┐      ┌──────────────────┐    ┌───────────────────┐
      │  HubSpot   │      │  Similarity DB   │    │  Reporting Agent  │
      │    CRM     │      │  updated with    │    │  Monday 08:00     │
      │  (clean,   │      │  team decisions  │    │  HubSpot + Ring-  │
      │  tagged,   │      │  (feedback loop) │    │  over combined    │
      │  enriched) │      └──────────────────┘    └───────────────────┘
      └────────────┘
```

### Ringover Integration — What It Adds

Ringover sits in two places in this system:

**1. Agent Gate** — Before a company is processed, the system checks Ringover: has this company been called before? How many times? What was the outcome? A company with 5 unanswered calls and a "not interested" note from Ringover should be routed to the block list, not enriched again.

**2. Similarity Scoring** — Call engagement is a conversion signal. Companies that became long-term clients typically had a different call pattern than companies that rejected proposals. This data enriches the feature vector used for similarity scoring.

**3. Reporting** — The weekly report includes call activity summaries: calls made, answer rates, conversion from call to meeting, by campaign and by rep.

---

## 4. Core Agent Components

### 4.1 Master File (Sales-Maintained)
A structured spreadsheet (Excel/Google Sheets, synced to server) maintained weekly by **one designated owner**.

**Required fields:**
- `linkedin_url` — primary unique identifier (never company name alone)
- `company_name` — display name
- `hq_location` — country/city
- `employee_size_band` — e.g. 50–200, 200–500, 500–1000
- `revenue_band_EUR_M` — e.g. <10, 10–50, 50–200
- `india_presence` — boolean
- `industry_vertical` — from approved taxonomy
- `active_for_scraping` — boolean toggle
- `last_updated` — date

**Hard rule:** The scraper will not run on master file data older than 10 days. It will alert and pause.

### 4.2 Agent Gate
Synchronous checks before any company is processed downstream.

| Check | Source | Action on fail |
|---|---|---|
| Block list | Local DB | Discard silently, log |
| Dedup — 60-day enrichment | Local dedup registry | Skip enrichment, reuse cached data |
| HubSpot presence — terminal status | HubSpot API | Discard if status = not interested / rejected |
| Ringover — prior contact | Ringover API | Add call history to company record; flag if repeatedly contacted with no outcome |

### 4.3 Enrichment Agent
Cache-first enrichment using Lusha and NeverBounce.

```
IF company in dedup_registry AND last_enriched < 60 days:
    reuse cached data — no API call
ELSE:
    call Lusha API → cache result with timestamp
    call Wappalyzer free tier if tech stack missing → cache

FOR each contact email:
    IF email in validation_cache AND validated < 90 days:
        reuse result
    ELSE:
        call NeverBounce → cache result
```

This logic alone reduces Lusha credit consumption by an estimated 60–80% compared to current practice.

### 4.4 Similarity Scoring Agent
Ranks new companies against 500+ labeled historical companies from HubSpot.

**Feature vector inputs:**
- Industry vertical (encoded)
- Employee size band (encoded)
- HQ country (encoded)
- Revenue band (encoded)
- India presence (boolean)
- Tech stack flags (binary, top 10 technologies)
- Job function of roles being hired (from scraper)
- Ringover engagement signal (contacted/not contacted, answer rate)

**Output per company:**
- Match score: 0–100
- Top 3 "similar to" companies with their HubSpot status
- One-line explanation: *"Similar to TechFirm NL (long-term client) and DataCo Amsterdam (short-term client)"*

**Model:** `sentence-transformers/all-MiniLM-L6-v2` — open-source, runs on CPU, no GPU needed.  
**Storage:** pgvector Postgres extension — same server, no external vector DB.

### 4.5 Standardisation Agent
Config-driven. All rules in `standards.yaml`. Sales team can edit config without touching code.

**Enforces:**
- Company name normalisation (title case, suffix handling for matching vs display)
- Industry vertical → approved taxonomy
- Country names → ISO codes
- Employee size → standard bands
- Tech stack → canonical names (React.js → React, NodeJS → Node.js)
- Campaign tags → approved list only; unapproved tags rejected with an alert

### 4.6 Reporting Agent
Scheduled: every Monday 08:00. Queries HubSpot + Ringover APIs, generates report, delivers via configured channel (Slack webhook or email).

**Report sections:**
- New companies added (by status, by score band)
- Pipeline movement (status changes this week)
- Top 5 highest-scoring new companies
- Lusha credits used vs saved by cache
- Ringover: calls made, answer rate, conversion to meeting
- Scraper run summary: scraped / passed gate / blocked / reason breakdown

No Excel. No manual data preparation. No human intervention.

---

## 5. Feasibility Assessment

| # | Task | Status | Note |
|---|---|---|---|
| 1 | Lead sourcing dedup | ✅ 100% | Dedup registry + LinkedIn URL as key |
| 2 | Enrichment gating | ✅ 100% | Requires Lusha API access on current plan — confirm before build |
| 3 | CRM dedup on upload | ✅ 100% | HubSpot duplicate rules must be pre-configured in HubSpot settings |
| 4 | Data cleaning | ✅ 100% | One-time retroactive batch + standards.yaml going forward |
| 5 | Lead qualification | ✅ 100% | Requires team to define qualification criteria in writing before Phase 2 |
| 6 | Campaign lists | ✅ 100% | HubSpot Professional API supports list management |
| 7 | Segmentation | ✅ 100% | HubSpot saved filters + agent-managed segment library |
| 9 | Email validation | ✅ 100% | NeverBounce free (1,000/mo) viable with 90-day cache. Upgrade trigger: >1,000 new emails/month |
| 10 | Reporting | ✅ 100% | HubSpot + Ringover APIs + Claude Haiku |
| 11 | Campaign tagging | ✅ 100% | Taxonomy defined once; enforced by standardisation agent |
| 13 | Tool extraction | ⚠️ Partial | Sales Nav: manual CSV export only. All other tools fully automated. |
| 14 | Tech stack research | ✅ 100% | Scraper extracts from job postings + Wappalyzer free tier supplement |
| 15 | Data formatting | ✅ 100% | standards.yaml covers all field formats |
| — | Similarity scoring | ✅ 100% | 500+ labeled records → Day 1 accuracy |
| — | Ringover integration | ✅ 100% | Ringover REST API is well-documented and accessible |
| — | LinkedIn Sales Navigator | ❌ No automation | LinkedIn bans automated access. CSV export is the only compliant path. |

---

## 6. Phase-by-Phase Development Plan

> 1 developer (Flask scraper builder). Conservative estimates with normal interruptions.

---

### Phase 1 — Foundation (Weeks 1–3)
**Goal:** Stop duplicates. Gate the pipeline. Cache enrichment.

| Task | Effort |
|---|---|
| On-premise environment: Python 3.11+, PostgreSQL 15+, pgvector, virtualenv | 2 days |
| Database schema: `companies`, `enrichment_log`, `validation_cache`, `block_list`, `scrape_runs` | 1 day |
| Master file schema definition + sync script (CSV/Sheets → DB, daily schedule) | 2 days |
| Agent gate: block list check | 1 day |
| Agent gate: dedup registry check (60-day window) | 1 day |
| Agent gate: HubSpot presence check via API | 2 days |
| Agent gate: Ringover prior-contact check | 1 day |
| Enrichment agent: Lusha API wrapper with cache-first logic | 2 days |
| Email validation: NeverBounce wrapper with 90-day cache | 1 day |
| Flask scraper extension: `/scrape` POST endpoint + JSON output + master file gate | 2 days |
| Integration test: scraper → gate → enrichment → dedup registry | 2 days |

**Phase 1 total: 3 weeks**  
**Deliverable:** Gated, dedup-safe pipeline. Lusha waste stops. CRM duplicates stop.

---

### Phase 2 — Similarity Scoring (Weeks 4–6)
**Goal:** Ranked, explained company view in the dashboard. Ringover signals live.

| Task | Effort |
|---|---|
| HubSpot historical export + normalisation (500+ companies) | 1 day |
| Ringover API integration: pull call history per company | 2 days |
| Feature vector design + encoding | 1 day |
| `sentence-transformers` setup on on-premise server (CPU mode) | 1 day |
| Embedding generation for all 500+ HubSpot companies → pgvector | 2 days |
| Similarity scoring agent: top-5 neighbours, weighted score, explanation string | 3 days |
| Dashboard integration: score column, "similar to" display, sort by score | 3 days |
| Feedback loop: reject → write negative weight to pgvector + block list option | 2 days |
| Score calibration with sales team (validate 10 known good / bad companies) | 1 day |

**Phase 2 total: 3 weeks**  
**Deliverable:** Dashboard ranks companies. "Similar to X" explanation visible. Feedback loop live.

---

### Phase 3 — Standardisation (Weeks 7–9)
**Goal:** Every record entering HubSpot is clean. Existing CRM data cleaned retroactively.

| Task | Effort |
|---|---|
| Workshop with sales team: define industry taxonomy, approved tag list, country codes | 2 days (collaborative) |
| Build `standards.yaml` config file from workshop output | 1 day |
| Standardisation agent: config-driven formatter | 3 days |
| Campaign tag enforcement via HubSpot property validation | 2 days |
| Tech stack normalisation: canonical name mapping | 1 day |
| Wappalyzer free tier integration | 1 day |
| Retroactive CRM cleanup: batch-run standardisation on all existing HubSpot records | 2 days |
| QA with sales team on cleaned records | 1 day |

**Phase 3 total: 2.5 weeks**  
**Deliverable:** Clean HubSpot from this point forward. Retroactive cleanup complete.

---

### Phase 4 — Reporting Automation (Weeks 10–11)
**Goal:** Zero manual report prep. Automated weekly summary including Ringover call data.

| Task | Effort |
|---|---|
| HubSpot API query set: pipeline status, movement, new additions | 2 days |
| Ringover API query set: calls, answer rates, outcomes, rep breakdown | 2 days |
| Report assembly: Claude Haiku generates natural-language summary | 1 day |
| Enrichment usage report (credits used vs cached) | 0.5 days |
| Delivery: Slack webhook or email, configurable | 1 day |
| Cron schedule: Monday 08:00, configurable | 0.5 days |
| Test: 2 consecutive reports, validate accuracy | 1 day |

**Phase 4 total: 1.5 weeks**  
**Deliverable:** Weekly report delivered automatically. Includes HubSpot + Ringover data combined.

---

### Phase 5 — Hardening & Documentation (Week 12)

- Regression testing across all phases
- Log-based alerting for agent failures (email alert if any agent errors out)
- Developer documentation
- Sales team usage guide (1-page: how to maintain the master file, how scores work)
- Deployment checklist
- Handover

---

### Timeline Summary

```
Week 1–3:   Phase 1 — Foundation
Week 4–6:   Phase 2 — Similarity scoring + Ringover signals
Week 7–9:   Phase 3 — Standardisation + retroactive CRM cleanup
Week 10–11: Phase 4 — Reporting automation
Week 12:    Hardening, documentation, handover

Total: 12 weeks · 1 developer
```

---

## 7. Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Language | Python 3.11+ | Existing scraper stack — no context switch |
| Web framework | Flask | Existing — dashboard already built here |
| Database | PostgreSQL 15+ | Reliable, on-premise, supports pgvector |
| Vector search | pgvector extension | No external vector DB needed; same server |
| Embedding model | `sentence-transformers/all-MiniLM-L6-v2` | Open-source, CPU-only, 384-dim vectors |
| AI reasoning | Claude API — Haiku (bulk), Sonnet (complex) | Haiku at ~€0.23/MTok for 90% of tasks |
| Scheduler | Python `APScheduler` or system `cron` | No extra infrastructure |
| HubSpot | REST API v3 | Official, stable, included in Professional |
| Lusha | REST API | Existing subscription |
| Ringover | REST API | Available on all Ringover plans |
| Sales Navigator | CSV export → file watcher | No API automation possible |
| Email validation | NeverBounce REST API | Free tier: 1,000/month with caching |
| Tech stack | Wappalyzer REST API | Free tier: 50/day supplement to scraper |
| Standards | YAML config file | Editable by sales team without code changes |

**Deliberately excluded:** n8n, Zapier, Make (per-task cost, no benefit for custom logic), Docker (add later if needed), any paid BuiltWith/Clearbit subscription.

---

## 8. API Access Requirements

Secure these **before development starts**.

| Tool | What to request | Lead time | Notes |
|---|---|---|---|
| **HubSpot** | Private App API key | Instant | Scopes: `crm.objects.contacts.read/write`, `crm.objects.companies.read/write`, `crm.lists.read/write`, `crm.schemas.read` |
| **Lusha** | API access confirmation | 1–3 days | **Confirm with account rep that API is included in current plan tier.** Some plans lock the API behind a higher tier. Get confirmation in writing. |
| **Ringover** | API key | Instant | Available in Ringover dashboard under Developer settings. All plans include API access. |
| **Anthropic (Claude)** | API key | Instant | Currently using Claude Pro Desktop — this does NOT include API access. API key must be created separately at console.anthropic.com. This is a hard prerequisite for the agent system. |
| **NeverBounce** | API key (free tier) | Instant | 1,000 verifications/month free. No credit card required. |
| **Wappalyzer** | API key (free tier) | Instant | 50 lookups/day free. Sign up at wappalyzer.com. |
| **On-premise server** | Outbound HTTPS | Verify now | Confirm outbound access to: `api.hubapi.com`, `api.lusha.com`, `api.ringover.com`, `api.anthropic.com`, `api.neverbounce.com`, `api.wappalyzer.com`. **This is the single most important pre-build check.** Corporate firewalls frequently block these. |
| **LinkedIn Sales Nav** | CSV export only | N/A | No API automation. Process: manual export → drop CSV in shared folder → file watcher picks up and ingests automatically. |

### Critical Note on Claude API Key
Eastern Enterprise currently has Claude Pro Desktop (claude.ai). This is a consumer product — it provides chat access but **no programmatic API access**. The agentic system requires an API key from console.anthropic.com, which is a separate billing relationship. Budget: pay-as-you-go. First month cost will be €15–40 depending on volume during testing.

---

## 9. Cost Breakdown — Frank and Open

### 9.1 What You Already Pay (No Change)
- HubSpot Professional subscription
- Lusha subscription (assuming API included)
- Ringover subscription
- LinkedIn Sales Navigator subscription
- On-premise server (hardware already owned)

### 9.2 New Monthly Costs (Post-Build)

| Item | Cost/Month | Notes |
|---|---|---|
| **Anthropic Claude API** | €15–50 | Haiku for 90% of tasks (~€0.23/MTok). Sonnet for fuzzy matching/scoring (~€2.80/MTok). Actual spend depends on volume. First 3 months will vary as caching matures. |
| **NeverBounce** | €0 | Free tier (1,000/month) sufficient with 90-day cache. If volume exceeds 1,000 new emails/month: upgrade to ~€8/month. |
| **Wappalyzer** | €0 | Free tier (50/day) sufficient for tech stack gap-filling. Flask scraper handles primary tech detection. |
| **PostgreSQL hosting** | €0 | Runs on on-premise server |
| **pgvector** | €0 | Open-source Postgres extension |
| **sentence-transformers** | €0 | Open-source, runs on-premise CPU |
| **Total new monthly cost** | **€15–50/month** | Conservative to moderate volume |

### 9.3 One-Time Development Cost

| Scenario | Estimate |
|---|---|
| Internal developer (existing Flask dev) | 12 weeks of existing salary allocation |
| External contract developer (mid-level) | €5,000–10,000 depending on rate |

### 9.4 What You Save Monthly (Estimated)

| Saving | Basis |
|---|---|
| Lusha credits | 60–80% fewer enrichment API calls via 60-day cache |
| NeverBounce | ~80% fewer validation calls via 90-day cache |
| Sales team data time | 8–15 hours/week recovered (dedup, formatting, list building, report prep) |
| Reduced tool overlap | Wappalyzer free replaces any paid BuiltWith spend |

At even €30/hour loaded cost for sales team time, recovering 10 hours/week = **€1,200+/month in recovered productivity** against a €15–50/month infrastructure cost. The ROI case is clear.

---

## 10. Data Architecture

### 10.1 Core Tables (PostgreSQL on-premise)

**`companies`**
```sql
id                  UUID PRIMARY KEY
linkedin_url        TEXT UNIQUE NOT NULL       -- primary key for dedup
company_name        TEXT
normalised_name     TEXT                       -- for fuzzy matching
hq_country          TEXT
employee_size_band  TEXT
revenue_band        TEXT
india_presence      BOOLEAN
industry_vertical   TEXT
tech_stack          JSONB
hubspot_id          TEXT
hubspot_status      TEXT
ringover_contacted  BOOLEAN
ringover_call_count INTEGER
ringover_last_outcome TEXT
first_seen          TIMESTAMP
last_updated        TIMESTAMP
```

**`enrichment_log`**
```sql
id                  UUID PRIMARY KEY
company_id          UUID REFERENCES companies
source              TEXT                       -- lusha, wappalyzer, manual
enriched_at         TIMESTAMP
data_snapshot       JSONB
credits_used        INTEGER
cache_hit           BOOLEAN
```

**`validation_cache`**
```sql
email               TEXT PRIMARY KEY
valid               BOOLEAN
validated_at        TIMESTAMP
source              TEXT
```

**`block_list`**
```sql
id                  UUID PRIMARY KEY
linkedin_url        TEXT
company_name        TEXT
reason              TEXT
added_by            TEXT
added_at            TIMESTAMP
```

**`similarity_vectors`** (managed by pgvector)
```sql
id                  UUID PRIMARY KEY
company_id          UUID REFERENCES companies
embedding           VECTOR(384)
hubspot_status      TEXT
weight              FLOAT        -- 1.0=long-term, 0.8=short-term, -0.5=rejected
updated_at          TIMESTAMP
```

**`scrape_runs`**
```sql
run_id              UUID PRIMARY KEY
triggered_at        TIMESTAMP
source              TEXT         -- flask_scraper, sales_nav_csv
companies_scraped   INTEGER
passed_gate         INTEGER
blocked_blocklist   INTEGER
blocked_dedup       INTEGER
blocked_hubspot     INTEGER
errors              JSONB
```

### 10.2 Similarity Score Computation

```python
# Pseudocode
def score_company(new_company, ringover_data):
    features = build_feature_vector(new_company, ringover_data)
    vector = embed(features)  # sentence-transformers
    
    neighbours = pgvector.query(vector, top_k=5)
    
    score = sum(n.similarity * n.weight for n in neighbours)
    normalised = min(100, int(score * 100))
    
    explanation = f"Similar to: {', '.join(
        f'{n.company_name} ({n.hubspot_status})' 
        for n in neighbours[:3]
    )}"
    
    return normalised, explanation
```

---

## 11. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Server outbound firewall blocks APIs** | Medium | Critical | Verify outbound HTTPS to all 6 endpoints before writing a line of code. This blocks everything if not resolved. |
| **Lusha API not in current plan** | Medium | High — delays Phase 1 | Confirm with Lusha account rep before build starts. Get it in writing. |
| **Claude API key not yet available** | Current state | High — no agents work without it | Register at console.anthropic.com. Separate from Claude Pro Desktop. Needs billing setup. |
| **Master file not maintained weekly** | Medium | High — degrades pipeline | One named owner. Scraper pauses and alerts if file is >10 days stale. |
| **Standards config not defined before Phase 3** | Medium | Medium — Phase 3 blocked | Sales team workshop in week 6 (before Phase 3 starts). |
| **Qualification criteria undefined** | Medium | Medium — Phase 2 scoring is rough | Sales team defines criteria in writing before Phase 2. 2-hour meeting. |
| **Ringover API rate limits** | Low | Low | Ringover allows up to 100 req/min. More than sufficient. |
| **NeverBounce free tier exceeded** | Low-medium (growth trigger) | Low | Monitor monthly. Upgrade to paid (~€8/mo) if >1,000 new emails/month. |
| **Scope creep** | Medium | Medium | This document defines the scope. New tasks go through change request. |
| **LinkedIn automation temptation** | Low (if documented clearly) | High if attempted | Explicitly out of scope. LinkedIn bans automation. CSV export is the compliant path. |

---

## 12. Pre-Development Checklist

These must be completed before development starts. Incomplete items block specific phases.

### Technical (blocks build)
- [ ] Verify server outbound HTTPS access to: `api.hubapi.com`, `api.lusha.com`, `api.ringover.com`, `api.anthropic.com`, `api.neverbounce.com`, `api.wappalyzer.com`
- [ ] Confirm PostgreSQL 15+ and pgvector can be installed on on-premise server (check OS, permissions, disk space)
- [ ] Create HubSpot Private App with required scopes
- [ ] Confirm Lusha API access included in current plan (written confirmation from Lusha)
- [ ] Obtain Ringover API key from Ringover developer settings
- [ ] Register at console.anthropic.com and set up API billing — **separate from Claude Pro Desktop**
- [ ] Obtain NeverBounce free tier API key
- [ ] Obtain Wappalyzer free tier API key
- [ ] Export full HubSpot company dataset with status labels (500+ companies) — ready for Phase 2 bootstrap

### Business (blocks phases if missing)
- [ ] **Designate master file owner** — one named person, not a committee
- [ ] Define master file schema and create initial version with current target companies
- [ ] Define lead qualification criteria in writing (needed before Phase 2)
- [ ] Define industry vertical taxonomy (needed before Phase 3)
- [ ] Define approved campaign tag list (needed before Phase 3)
- [ ] Book sales team workshop in week 6 for standards config session

---

## 13. Out of Scope

Explicitly excluded. Do not add mid-build without a formal change request.

- **LinkedIn Sales Navigator automation** — LinkedIn bans this. Not negotiable.
- **Automated email outreach/sequencing** — this system handles data, not sending
- **CRM migration** — HubSpot remains the CRM of record
- **Mobile application** — dashboard is a web interface
- **Custom ML model training** — pre-trained sentence-transformers are sufficient
- **Paid BuiltWith, Clearbit, Apollo, ZoomInfo** — not required given current stack
- **Multi-country scraping** — Netherlands only in this phase
- **GDPR compliance audit** — handled separately by legal/compliance

---

*Document version 2.0 — April 2026 — Constrained to 4 core tools: HubSpot, Lusha, Sales Navigator, Ringover*  
*Next review: after Phase 1 completion or when Claude API key is obtained*
