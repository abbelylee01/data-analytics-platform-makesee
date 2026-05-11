# Makesee — Product Plan

**Version:** 0.1  
**Status:** Draft 1 — In Progress  
**Pricing:** TBD (next iteration)

> This document consolidates initial findings on the Makesee platform — product vision, user personas, core modules, AI architecture, and open questions. Pricing will be addressed in a later iteration.

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [User Personas](#2-user-personas)
3. [Core Modules](#3-core-modules)
4. [AI Context Architecture](#4-ai-context-architecture)
5. [System Architecture Overview](#5-system-architecture-overview)
6. [Open Questions & Next Steps](#6-open-questions--next-steps)

---

## 1. Product Overview

Makesee is a retail analytics platform that turns raw daily transaction CSV files into an actionable intelligence layer. It combines a visual dashboard with an AI advisor that speaks in plain English about what's selling, what isn't, and what to do about it.

The product is designed for retail shop owners and managers who currently rely on spreadsheets or basic POS reports. Makesee removes the need for a data analyst on staff by surfacing insights automatically and letting users ask questions in natural language.

> **Core value proposition:** Upload your daily transaction file. Makesee tells you what's selling, flags what needs attention, and advises on what to do — without you having to ask.

---

## 2. User Personas

Makesee serves two distinct user types. The core product experience is the same for both; what differs is the lens through which data is viewed and the level of cross-location comparison available.

### Persona A — The Solo Retailer

| Attribute | Detail |
|---|---|
| Who they are | A shop owner — clothing, gifts, food, hardware — who manages everything themselves. No analyst on staff. |
| Core question | "What's selling and what should I order more of?" |
| Upload behaviour | One CSV per day or week, exported manually from their till system. |
| Dashboard needs | Simple KPIs, top 5 sellers, low stock alerts. No complexity. |
| AI needs | Plain English advice with specific product names and numbers. No jargon. |
| Plan tier | Starter (free or low-cost) |

### Persona B — The Multi-Branch Manager

| Attribute | Detail |
|---|---|
| Who they are | An owner or ops manager with 2–10 branches. Spends time comparing performance across locations. |
| Core question | "Why is Branch 2 underperforming compared to Branch 1?" |
| Upload behaviour | Separate CSV per branch (with `store_id` column), or a combined file. |
| Dashboard needs | Branch filter, side-by-side comparisons, product performance by location. |
| AI needs | "Branch 3 sells 3× more umbrellas — should I redistribute stock?" |
| Plan tier | Pro (paid upgrade) |

> **One product, two modes:** A setup question during onboarding — "Do you operate more than one location?" — unlocks branch features for multi-location users. Solo retailers see a clean, uncluttered experience by default.

---

## 3. Core Modules

### 3.1 Data Ingestion

Users upload CSV files containing daily transaction records. The system validates each file against an expected schema, flags errors, and normalises the data before storing it.

#### Expected CSV Schema

| Field | Type | Example |
|---|---|---|
| `date` | YYYY-MM-DD | 2025-11-14 |
| `product_id` | string | SKU-4821 |
| `product_name` | string | Wool scarf |
| `category` | string | Accessories |
| `quantity_sold` | integer | 12 |
| `unit_price` | decimal | 24.99 |
| `total_revenue` | decimal | 299.88 |
| `store_id` | string | STORE-01 *(multi-branch only)* |

### 3.2 Analytics Dashboard

The central view. Once data is loaded, the dashboard presents KPI cards, product performance charts, and sales trends over time. Filters by date range, product, or store location make this useful for any business size.

**Key dashboard components:**

- KPI cards: total revenue, units sold, average order value, revenue vs prior period
- Top 10 and bottom 10 performing products
- Sales trend chart over a selectable date range
- Category breakdown (e.g. by product type or department)
- Low stock alert list — products trending toward zero
- Branch comparison panel *(Pro tier only)*

### 3.3 AI Advisor

The AI advisor is the core differentiator. It surfaces recommendations automatically when a user opens the dashboard and responds to natural language questions in the chat interface.

#### Proactive mode (no question required)

Every time data is uploaded, the AI reads the computed summary and generates 3–5 recommendation cards on the dashboard. Examples:

- "Wool scarves are your fastest-moving item — you've sold 47 this week vs 12 last week. Consider restocking."
- "Gift sets have had zero sales for 14 days. You may want to discount or reposition."
- "Revenue is down 18% vs last week. The biggest drop is in the Accessories category."

#### Chat mode (user-initiated)

Users can ask natural language questions at any time. The system retrieves relevant data and the AI interprets it. Example questions:

- "What sells most on weekends vs weekdays?"
- "Which products have sold out in the last month?"
- "Should I stock more of product X heading into December?"
- "Why did revenue dip in week 3?"
- "Branch 2 vs Branch 1 — what's the main difference?"

### 3.4 Pattern & Seasonal Alerts

A background layer that watches for recurring signals across the transaction history and surfaces them as proactive notifications. Examples include products that spike at the same time each year, items that consistently run to zero stock before reorder, or unusual drops suggesting a supply or pricing issue.

---

## 4. AI Context Architecture

The AI needs access to transaction data to give useful answers. Makesee uses a two-layer architecture that balances speed, cost, and depth of response.

### Layer 1 — Pre-computed Summaries (Proactive Layer)

After every CSV upload, the system computes a compact structured summary: top sellers, bottom sellers, revenue vs prior period, sold-out products, zero-movement products. This is stored as a small JSON object (typically 2–3 KB per upload).

When the dashboard loads, this summary is automatically passed to the AI, which generates recommendation cards without the user asking anything. This approach is fast, cheap, and always on.

**Example AI prompt (proactive mode):**
```
You are a retail advisor for [store name]. Here is their sales summary for the past 7 days: [JSON summary]. Identify 3–5 clear, actionable insights. Be specific about product names and numbers. Keep each insight to 1–2 sentences.
```

### Layer 2 — Live Query (Chat Mode)

When a user types a question in the chat, the system runs a targeted query against the transaction database to fetch the specific data needed, then passes it to the AI along with the question.

The AI does not write SQL directly. Instead, the system uses a library of pre-templated queries mapped to common question types. The AI selects the right template based on the intent of the question, fills in the parameters, and the query result is passed back for interpretation.

**Example AI prompt (chat mode):**
```
You are a retail advisor. The user has asked: "[question]". Here is the relevant data retrieved from their transaction history: [query result table]. Interpret this data and give a plain English answer with a specific recommendation.
```

### Approach Comparison

| Approach | Speed | Cost | Best for |
|---|---|---|---|
| Pre-computed summaries | Very fast | Very low | Daily recommendations |
| Live query layer | Moderate | Moderate | Deep ad-hoc chat questions |
| Full RAG retrieval | Moderate | Higher | Product Q&A at large scale |

> Makesee uses Layer 1 and Layer 2 in combination. RAG retrieval is noted as a future option for larger merchants with months of transaction history.

---

## 5. System Architecture Overview

| Layer | Components |
|---|---|
| **Input** | CSV upload → schema validation → error flagging → normalisation. Future: scheduled auto-import via API hook to POS system. |
| **Storage** | Centralised transaction database storing product, date, quantity, revenue, and store ID per record. |
| **Intelligence** | Summary engine (runs on upload), templated query library, AI advisor (proactive + chat), seasonal pattern detection. |
| **Output** | Analytics dashboard with KPI cards and charts, AI chat interface, reports and export (PDF / CSV / email). |

---

## 6. Open Questions & Next Steps

| Area | Question |
|---|---|
| **Pricing** | Pricing model and tier structure to be defined in the next iteration. |
| **Onboarding** | What does the first-upload experience look like? Should sample data be available? |
| **Query templates** | Which question intents should be pre-templated at launch? Requires user research. |
| **AI provider** | Which LLM provider should power the advisor? Cost, latency, and data privacy all factor in. |
| **Data retention** | How long is transaction history kept? Are users alerted before data expires? |
| **POS integrations** | Which till systems should be prioritised for auto-import in v2? |
| **Mobile support** | Is a mobile-responsive web app sufficient, or is a native app needed? |

---

*Draft 1 — All findings subject to revision. Pricing, design wireframes, and technical specifications will be added in subsequent iterations.*
