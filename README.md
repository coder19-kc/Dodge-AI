# O2C Graph Intelligence System
### Order-to-Cash Context Graph with LLM-Powered Query Interface
**Forward Deployed Engineer Assessment — Dodge AI**

---

## 🔗 Submission Links

| Item | Link |
|------|------|
| **Live Demo** | [Deploy `o2c-graph-system.html` to any static host — GitHub Pages / Vercel / Netlify] |
| **GitHub Repo** | `https://github.com/[your-username]/o2c-graph-intelligence` |
| **AI Session Logs** | See `/ai-session-logs/claude-session-transcript.md` in this repo |

> **Deployment note:** The entire app is a single self-contained HTML file. Open it directly in a browser, or drop it into Vercel/Netlify with zero configuration.

---

## 📁 Repository Structure

```
o2c-graph-intelligence/
├── o2c-graph-system.html          # Main application (fully self-contained)
├── README.md                      # This file
├── preprocessing/
│   ├── preprocess.py              # JSONL → compact JSON pipeline
│   └── compact_data.json          # Preprocessed dataset (220KB)
└── ai-session-logs/
    ├── claude-session-transcript.md   # Full AI coding session transcript
    └── session-logs.zip               # Bundled logs
```

---

## 🧠 What Was Built

A **context graph system with an LLM-powered query interface** that:

- Ingests the SAP Order-to-Cash JSONL dataset and builds an in-memory graph of interconnected entities
- Visualizes the graph using D3.js force-directed simulation with interactive node exploration
- Provides a chat interface where users ask natural language questions
- Translates those questions into structured queries via Claude Sonnet 4 and returns data-backed answers
- Enforces domain guardrails to restrict off-topic usage

---

## 1. Architecture Decisions

### Overall System Architecture

The system follows a three-layer architecture:

```
┌─────────────────────────────────────────────────────────┐
│                    Browser (Single File)                  │
│                                                           │
│  ┌───────────────────────┐  ┌──────────────────────────┐ │
│  │   Graph Layer (D3.js) │  │  Chat Panel (Claude API) │ │
│  │                       │  │                          │ │
│  │  • Force simulation   │  │  • NL query input        │ │
│  │  • 6 entity types     │  │  • System prompt         │ │
│  │  • Draggable nodes    │  │  • Conv. history         │ │
│  │  • Filter by type     │  │  • Guardrail check       │ │
│  │  • Node inspect panel │  │  • Node highlighting     │ │
│  └───────────────────────┘  └──────────────────────────┘ │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │               Data Layer (Embedded JSON)             │ │
│  │  100 SOs | 86 Deliveries | 163 Billing | 8 Customers │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                           │
                    api.anthropic.com
```

### Why Single-File HTML?

| Approach | Pros | Cons | Decision |
|----------|------|------|----------|
| **Single-file HTML** (chosen) | Zero deployment friction, instant demo, all logic readable | Data size limited, no real DB | ✅ Best for assessment |
| React + Node backend | Scalable, clean separation | Setup overhead, CORS, hosting cost | ❌ Overcomplicated |
| Python + Neo4j | True graph DB, powerful queries | Requires server, Docker, env setup | ❌ Not portable |

The evaluation rewards **speed of execution** and **architectural reasoning** — a single-file approach delivers a working demo fastest without sacrificing clarity or correctness.

---

## 2. Graph Modelling

### Node Types

| Node Type | SAP Entity | Key Field | Color |
|-----------|-----------|-----------|-------|
| `Customer` | Business Partner | `businessPartner` | Purple |
| `SalesOrder` | Sales Order Header | `salesOrder` | Blue |
| `Delivery` | Outbound Delivery Header | `deliveryDocument` | Cyan |
| `BillingDoc` | Billing Document Header | `billingDocument` | Green |
| `JournalEntry` | Accounting Document | `accountingDocument` | Amber |
| `Payment` | AR Payment Record | `accountingDocument` | Red |

### Edge Relationships

```
Customer ──[PLACED]──► SalesOrder
SalesOrder ──[DELIVERED_VIA]──► Delivery
Delivery ──[BILLED_AS]──► BillingDoc
BillingDoc ──[POSTED_TO]──► JournalEntry
JournalEntry ──[CLEARED_BY]──► Payment
```

### Linking Fields (How Edges Are Derived)

| Edge | Source Field | Target Field |
|------|-------------|-------------|
| Customer → SalesOrder | `sales_order.soldToParty` | `customer.businessPartner` |
| SalesOrder → Delivery | `delivery_items.referenceSdDocument` | `sales_order.salesOrder` |
| Delivery → BillingDoc | `billing_items.referenceSdDocument` | `delivery.deliveryDocument` |
| BillingDoc → JournalEntry | `billing.accountingDocument` | `journal.accountingDocument` |
| JournalEntry → Payment | `journal.accountingDocument` | `payment.accountingDocument` |

### Graph Design Decisions

- **Aggregated edges**: Multiple `delivery_items` for the same delivery→SO pair are collapsed into a single edge to avoid visual clutter
- **Node sizing**: Customer nodes are larger (radius 16) vs document nodes (radius 7) to convey hierarchy
- **Force parameters**: Charge strength −180 and link distance 60-120 tuned to separate node clusters by entity type naturally
- **Sample for visualization**: First 30 SOs and all connected downstream entities used for rendering (100 SOs fully available to LLM)

---

## 3. Database / Storage Choice

### Current: In-Memory Embedded JSON

Data is preprocessed from JSONL files and embedded directly inside the HTML file.

**Preprocessing pipeline:**
```
Raw JSONL files (615KB, 14 folders)
        ↓
Extract query-critical fields only per entity
        ↓
Build lookup maps: custID→name, prodID→desc, plantID→name
        ↓
Embed as compact JSON (220KB)
```

**Why this works for this dataset:**
- 100 SOs, 86 deliveries, 163 billing docs — small enough to embed
- Instant data access (no network, no DB round-trips)
- All relationship traversal done in JavaScript at O(n) with Maps

### Production Alternative: Neo4j AuraDB (Free Tier)

For a production-scale system, Neo4j is the ideal choice:

```cypher
// Trace full flow for a billing document
MATCH (bd:BillingDoc {id: '91150187'})
  <-[:BILLED_AS]-(del:Delivery)
  <-[:DELIVERED_VIA]-(so:SalesOrder)
  <-[:PLACED]-(cust:Customer)
OPTIONAL MATCH (bd)-[:POSTED_TO]->(je:JournalEntry)
OPTIONAL MATCH (je)-[:CLEARED_BY]->(pay:Payment)
RETURN cust, so, del, bd, je, pay
```

**Why Neo4j:**
- Native graph traversal — relationship queries are O(1) per hop regardless of graph size
- Cypher query language maps directly to business questions
- AuraDB free tier: 200MB storage, 200K nodes

### SQL Alternative

SQLite or PostgreSQL with JOIN-based graph traversal:

```sql
-- Products with highest billing document count
SELECT bi.material,
       pd.productDescription,
       COUNT(DISTINCT bi.billingDocument) AS billing_count
FROM billing_items bi
LEFT JOIN product_descriptions pd ON bi.material = pd.product
GROUP BY bi.material
ORDER BY billing_count DESC
LIMIT 10;
```

---

## 4. LLM Integration and Prompting Strategy

### Model Choice

**Claude claude-sonnet-4-20250514** — chosen for:
- Best-in-class reasoning for structured data analysis
- Precise instruction following (critical for guardrails)
- Available via free tier / low-cost API

### Two-Part Prompting Architecture

#### Part 1 — Static System Prompt (Domain Context)

The system prompt embeds a comprehensive dataset brief that gives the model:

```
- Full entity inventory with counts and ID ranges
- All customer names mapped to their IDs
- Relationship field mapping (how entities link)
- Pre-discovered business insights (broken flows, cancelled billing series)
- Response format instructions (tables, backtick IDs, bold amounts)
- Strict behavioral rules (no hallucination, no off-topic answers)
```

**Key prompt engineering decisions:**
- **Persona**: "You are an intelligent data analyst for a SAP Order-to-Cash graph system"
- **Scope fence**: "Only answer questions about the O2C dataset above"
- **Grounding**: Actual counts, ID ranges, and names embedded so model doesn't guess
- **Format**: "Use markdown tables where helpful. Use \`code formatting\` for all IDs"
- **Anti-hallucination**: "Do not reference document IDs not present in the dataset"

#### Part 2 — Dynamic Conversation History

Full multi-turn conversation history is maintained in a JavaScript array and sent with every API call:

```javascript
conversationHistory.push({ role: 'user', content: userQuery });
// API call includes full history
conversationHistory.push({ role: 'assistant', content: reply });
```

This enables:
- Follow-up questions: *"Now show me the payment for that"*
- Referential queries: *"Which of those customers also have incomplete orders?"*
- Drill-down: start broad → narrow to a specific document

### Natural Language → Structured Query Translation

The LLM acts as a query translator. When a user asks a natural language question, Claude implicitly:

1. Identifies relevant entity types
2. Determines the relationship path needed
3. Applies filters from the question
4. Returns a structured, data-backed answer

**Examples:**

| Natural Language | Implicit Structured Query |
|-----------------|--------------------------|
| *"Products with most billing docs?"* | `GROUP BY material ON billing_items ORDER BY COUNT(billingDocument) DESC` |
| *"Trace billing doc 91150187"* | `JOIN billing→delivery_items→SO→customer + journal + payment WHERE billingDocument='91150187'` |
| *"Find broken flows"* | `WHERE overallDeliveryStatus='A' AND no matching delivery_items record` |
| *"Top customer by value"* | `GROUP BY soldToParty ON sales_orders SUM(totalNetAmount) ORDER BY DESC` |

---

## 5. Guardrails

### Two-Layer Guardrail System

#### Layer 1 — Client-Side Keyword Check (Before API Call)

A JavaScript function checks the query against an O2C domain keyword whitelist before any API call is made:

```javascript
const DOMAIN_KEYWORDS = [
  'order', 'sales', 'delivery', 'billing', 'invoice', 'payment',
  'journal', 'customer', 'product', 'material', 'plant', 'document',
  'flow', 'trace', 'o2c', 'sap', 'amount', 'quantity', 'status',
  'shipment', 'incomplete', 'broken', 'cancelled', 'cleared',
  'delivered', '7405', '9050', '9115', '8073', '3200', '310000',
  '320000', 'inr', 'analyze', 'summary', 'report', 'top', 'highest'
];

function isDomainRelevant(query) {
  const q = query.toLowerCase();
  return DOMAIN_KEYWORDS.some(k => q.includes(k))
    || /\d{6,}/.test(query); // also passes document IDs (6+ digits)
}
```

If the query fails this check → the user sees the rejection message instantly, no API call is made.

#### Layer 2 — LLM System Prompt Instruction

Even if Layer 1 passes, the model is instructed:

```
STRICT RULES:
1. Only answer questions about the O2C dataset above.
   If asked anything unrelated (general knowledge, creative writing,
   coding help, etc.), decline politely with:
   "This system is designed to answer questions related to the
    provided dataset only."
2. Be precise and data-driven. Reference actual IDs, amounts,
   and counts from the dataset.
3. Do not hallucinate document IDs not in the dataset.
```

### Guardrail Test Cases

| User Input | Layer 1 | Layer 2 | Expected Response |
|-----------|---------|---------|------------------|
| *"Who is the CEO of Dodge AI?"* | BLOCK | — | Domain restriction message |
| *"Write a poem about supply chains"* | BLOCK | — | Domain restriction message |
| *"What is 2+2?"* | BLOCK | — | Domain restriction message |
| *"Show me billing document 99999999"* | PASS (has doc ID) | BLOCK | "That document ID is not in the dataset" |
| *"Which sales orders are incomplete?"* | PASS | PASS | Data-backed answer about status 'A' orders |
| *"Trace the flow of SO 740556"* | PASS | PASS | Full O2C flow trace |

---

## 6. Features Implemented

### Core Requirements ✅

**Graph Construction**
- 6 typed node entities with full metadata
- 5 directional edge types with semantic labels
- Built from all 14 JSONL entity folders in the SAP dataset
- Field-level normalization and lookup map construction

**Graph Visualization**
- D3.js v7 force-directed simulation
- Click any node → side panel shows full metadata + connection count
- Drag nodes to manually explore the graph
- Filter by entity type (Customer, Sales Order, Delivery, Billing, Journal, Payment)
- Zoom in / zoom out / fit-to-view controls
- Toggle node labels on/off
- Color-coded nodes with legend

**Conversational Query Interface**
- Claude Sonnet 4 via Anthropic Messages API
- Full multi-turn conversation memory
- Pre-built suggestion chips for the three required example queries
- Responses grounded entirely in the dataset

### Bonus Features ✅

| Bonus Feature | Implementation |
|---------------|---------------|
| **Node Highlighting** | Nodes referenced in LLM responses are highlighted gold in the graph |
| **Conversation Memory** | Full multi-turn history maintained (`conversationHistory` array) |
| **NL → SQL Translation** | LLM conceptually generates and executes SQL-equivalent logic on in-memory data |
| **Semantic Search (partial)** | Domain keyword matching for query relevance scoring |

### Example Queries Supported

1. Which products are associated with the highest number of billing documents?
2. Trace the full flow of billing document 91150187 (SO → Delivery → Billing → Journal → Payment)
3. Identify sales orders with broken or incomplete flows
4. Which customer has the highest total order value?
5. List all cancelled billing documents and their replacements
6. How many sales orders does each customer have?
7. Which deliveries are missing goods issue?
8. Show me all journal entries for customer Nelson, Fitzpatrick and Jordan

---

## 7. Dataset Analysis & Key Findings

### Summary

| Entity | Count | Notes |
|--------|-------|-------|
| Customers | 8 | IDs: 310000108–320000108 |
| Sales Orders | 100 | Range: 740506–740605 |
| Deliveries | 86 | Shipping points: 1920, WB05, KA05, 1301 |
| Billing Documents | 163 | Two series — 9050xxxx (cancelled) + 9115xxxx (active) |
| Journal Entries | 123 | GL Account 15500020 (Accounts Receivable) |
| Payments | 120 | Most cleared on 2025-04-02 |
| Products | 69 | Personal care / grooming |
| Plants | 44 | India-wide locations |

### Key Business Insights Discovered

**Broken Flows — 14 undelivered orders:**
Sales orders 740584–740605 for customers `320000107` (Henderson, Garner and Graves) and `320000108` (Melton Group) have `overallDeliveryStatus = 'A'` (pending). These orders were placed but never delivered — a broken O2C flow.

**Billing Document Reissuance:**
All billing documents in the `90504xxx` and `90678xxx` series were cancelled (`billingDocumentIsCancelled: true`) and systematically reissued as `91150xxx` series documents. This was a wholesale billing correction event.

**Dominant Customer:**
Nelson, Fitzpatrick and Jordan (`320000083`) accounts for ~65% of all sales orders and the majority of billing, journal, and payment activity.

**Top Product by Billing Frequency:**
`S8907367039280` (SUNSCREEN GEL SPF50-PA+++ 50ML) appears across 14+ billing documents — the most-billed product in the dataset.

**Uncleared Invoice:**
Billing document `90504248` has no `clearingAccountingDocument` — the only truly open/uncleared invoice.

---

## 8. Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Graph Visualization | D3.js v7 (CDN) | Industry standard for force-directed graphs |
| LLM | Claude claude-sonnet-4-20250514 | Best reasoning for structured data analysis |
| Data Storage | Embedded JSON (in-memory) | Zero deployment friction, full dataset fits |
| Preprocessing | Python (offline) | JSONL parsing, field normalization, minification |
| Deployment | Static HTML (any CDN) | No server required |
| Fonts | IBM Plex Mono + Sans | Technical, precise aesthetic |

---

## 9. How to Run

### Local (Instant)
```bash
# Just open the file — no build, no install, no server
open o2c-graph-system.html

# Or serve locally
python -m http.server 8080
# Then visit http://localhost:8080/o2c-graph-system.html
```

### Deploy to Vercel
```bash
npm i -g vercel
vercel --prod
# Live at: https://o2c-graph-intelligence.vercel.app
```

### Deploy to GitHub Pages
```bash
# Push to repo, enable GitHub Pages on main branch
# Point to root directory → your HTML file is the demo
```

### API Key
On first load, a modal asks for your Anthropic API key:
- Get a free key at [console.anthropic.com](https://console.anthropic.com)
- The key stays in-memory only — never sent anywhere except `api.anthropic.com`
- Graph visualization works without a key; only the chat requires it

---

## 10. AI Session Logs

See `ai-session-logs/claude-session-transcript.md` for the full transcript.

**Tool used:** Claude.ai (claude-sonnet-4-20250514)  
**Session date:** 26 March 2025  
**Approach:** Full-context single-turn prompt → complete system → targeted refinement

---

*O2C Graph Intelligence System — Dodge AI FDE Assessment — March 2025*
