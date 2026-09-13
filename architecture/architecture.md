# Niche Scout — Architecture

This document explains how Niche Scout is built at a high level. It contains no implementation code.

---

## 1. System overview

```mermaid
flowchart TB
    subgraph Client
        L["Public landing page"]
        D["Admin dashboard<br/>dashboard · products · niches · runs<br/>lookup · criteria · settings"]
    end

    subgraph App["Next.js 16 app"]
        PX["proxy<br/>session check + auth gate"]
        RSC["Server Components"]
        RH["Route Handlers<br/>scrape · continue · lookup · export<br/>niches · sources · settings · cron"]
    end

    subgraph Pipeline
        ORCH["Orchestrator<br/>resumable state machine"]
        REG["Adapter registry"]
        FETCH["Fetch layer<br/>direct · proxy API · Playwright"]
        ANA["Analysis<br/>currency · landed cost · availability · scoring"]
        AIL["AI layer<br/>enrich · explain · summarise · rate supplier"]
        XL["Excel builder"]
    end

    subgraph Supabase
        DB[("Postgres + RLS")]
        ST["Private storage<br/>images · workbooks"]
        AUTH["Auth"]
    end

    subgraph External
        SUP["Alibaba · 1688 · Made-in-China"]
        MKT["Daraz · PakWheels"]
        LLM["Claude / OpenAI"]
        GS["Google Programmable Search"]
    end

    L --> PX
    D --> PX --> RSC --> DB
    PX --> AUTH
    D --> RH --> ORCH
    ORCH --> REG --> FETCH
    FETCH --> SUP
    FETCH --> MKT
    ORCH --> ANA --> AIL --> LLM
    RH --> GS
    ORCH --> DB
    ORCH --> XL --> ST
```

---

## 2. Scoring pipeline (per product)

The stages run from cheapest to most expensive, so products that fail early never cost a market lookup or an AI call.

```mermaid
flowchart LR
    A["Supplier product"] --> X{"Excluded keyword<br/>or product?"}
    X -- yes --> Drop["Drop before saving"]
    X -- no --> E["Normalise title<br/>AI or rule-based fallback"]
    E --> C["Landed cost<br/>goods×FX + freight/kg<br/>+ duty & tax + clearing/MOQ"]
    C --> G1{"Over PKR<br/>ceiling?"}
    G1 -- yes --> Skip["skip"]
    G1 -- no --> AV["Local availability<br/>title-matched Daraz listings"]
    AV --> DM["Demand<br/>units sold · reviews · ratings"]
    DM --> SAT["Saturation<br/>listings · sellers · brand share<br/>· price convergence"]
    SAT --> M["Margin<br/>vs measured median<br/>or target multiple"]
    M --> S["Opportunity score 0–100<br/>+ list of passed/failed checks"]
    S --> V{"Verdict"}
    V --> Go["go"]
    V --> Watch["watch"]
    V --> Skip
    Go --> R["AI explanation<br/>(go/watch by default)"]
    Watch --> R
```

**Availability states:**

| State | Meaning |
|---|---|
| **available** | Sold locally; the price band is measured |
| **scarce** | Sold locally, but too few listings to trust the price band |
| **absent** | Not sold locally; any selling price is an assumption |
| **unknown** | The marketplace lookup failed, so nothing can be said about the market |

A product can reach **go** only with a measured local price. Promising products whose price is only estimated are capped at **watch**.

---

## 3. Resumable run state machine

```mermaid
stateDiagram-v2
    [*] --> Queued: admin starts a run
    Queued --> Running: claimed by function or worker
    Running --> Running: process product → save position
    Running --> Paused: time budget nearly used up
    Paused --> Running: self-continue call or cron sweep
    Running --> Completed: all niches and keywords done
    Running --> Failed: unrecoverable error
    Running --> Queued: stale run returned to queue after worker crash
    Completed --> [*]: Excel workbooks written, run summary generated
```

- **Saved position:** the run's progress is written to the database after every product, so any process can pick it up.
- **Snapshot:** thresholds and the AI provider are copied onto the run when it starts.

---

## 4. Deployment topologies

```mermaid
flowchart LR
    subgraph Serverless["Vercel"]
        W1["Next.js app"] -->|time-limited function| O1["Orchestrator"]
        O1 -->|pauses, calls itself again| O1
        CR["Cron sweep"] --> O1
    end

    subgraph Docker["Docker Compose"]
        W2["web container<br/>only adds runs to queue"]
        WK["worker container<br/>Playwright + Chromium<br/>no time limit"]
    end

    DB[("Supabase<br/>= job queue")]
    O1 --> DB
    W2 --> DB
    WK -->|claims queued runs| DB
    WK --> EXP["./exports on host"]
```

| | Vercel | Docker |
|---|---|---|
| Time limit | Per-function limit, handled by pausing and resuming | None, the worker runs a whole crawl |
| Browser | None, uses a scraping API for protected sites | Real Chromium |
| Dispatch | Runs inside the web app | Web adds to the queue, the worker runs it |

---

## 5. Data model

| Table | Purpose |
|---|---|
| **admin_users** | The allowlist of users who can access the dashboard |
| **settings** | Global thresholds, landed-cost model, AI provider |
| **niches** | Niche definitions: keywords, excluded keywords, category |
| **niche_keyword_exclusions** | Admin-confirmed keywords to stop searching |
| **dynamic_sources** | Admin-added sources with AI-suggested selector configs |
| **scrape_runs** | Run status, saved position, frozen settings, AI summary |
| **products** | Discovered supplier products, including landed cost and exclusion flag |
| **market_analyses** | Per-run availability, demand, saturation, score, verdict, reasoning |
| **exports** | Excel workbooks generated per run |

**Supporting objects:**
- **View:** a combined product-insights view.
- **Indexes:** a fuzzy-text (trigram) index on product titles, plus indexes on verdict, score, cost and run.
- **Access control:** row-level security policies and helper functions for the admin and editor roles.
