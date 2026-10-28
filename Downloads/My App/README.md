# Verisum: Privacy-Preserving Brand Validation Engine

> **Differentially Private Brand Governance with DPSparseVoteRAG & AI²L Human Authority**

Verisum is a production-grade prototype that validates marketing copy against sensitive enterprise brand guidelines while provably guaranteeing corpus confidentiality. Built for organizations in regulated and high-trust domains (health, wellness, finance), Verisum pairs **Sample-and-Aggregate Differential Privacy** with the **Sparse Vector Technique (SVT)** to deliver explainable, evidence-backed findings to human brand reviewers.

---

## 1. System Architecture

```
                       +---------------------------------------+
                       |          HUMAN REVIEWER (AI²L)        |
                       | (Approve, Request Edits, Reject, etc) |
                       +---------------------------------------+
                                          ^
                                          | Explainable Findings + Evidence
                       +---------------------------------------+
                       |           VERISUM ENGINE              |
                       |  +---------------------------------+  |
                       |  |    Sparse Vector Technique (SVT) |  |
                       |  |    + LimitedDomain Mechanism     |  |
                       |  +---------------------------------+  |
                       |                  ^                    |
                       |    m Parallel LLM Voters + Baseline   |
                       |                  |                    |
                       |     Uniform Disjoint Partitions       |
                       |                  |                    |
                       |        Top m*k BM25 Retrieval         |
                       +---------------------------------------+
                                          |
                       +---------------------------------------+
                       |      PRIVATE BRAND CORPUS (25 Docs)   |
                       | (Claims, Disclaimers, Memos, Secrets) |
                       +---------------------------------------+
```

### The Three Core Values
1. **Corpus Confidentiality:** Trade secrets, internal memos, and unreleased patents cannot be inferred or extracted by querying the system.
2. **Evidence Traceability:** When an issue is flagged, human reviewers see the exact guideline citation and consensus score without exposing private records.
3. **Human Final Authority ($\text{AI}^2\text{L}$):** The AI acts strictly as an augmented perception layer; human brand officers retain absolute final authority.

---

## 2. Changes from Original Theoretical Plan

In translating the theoretical formulation in `docs/solution_overview.md` to a modern web architecture, three deliberate adaptations were made:

| Component | Theoretical Plan | Production Implementation | Rationale |
| :--- | :--- | :--- | :--- |
| **Search Engine** | Apache Lucene / PyLucene | **Pure-TypeScript BM25 (`lib/retrieval/bm25.ts`)** | Enables instant, zero-dependency deployment on serverless Vercel runtimes without Java or native runtime bindings. |
| **Voting Granularity** | Autoregressive Token-Level Voting | **Decision-Level Voting across 4 Checks (`tone`, `terminology`, `unsupported_claim`, `policy`)** | Closed-source API LLMs (Google Gemini) do not support per-token prefix conditioning. Decision-level voting evaluates structured JSON verdicts per check, applying SVT and `LimitedDomain` histogram selection over discrete candidate verdicts. |
| **Application Stack** | Python backend + separate frontend | **Unified Next.js 14 App Router (TypeScript + Tailwind CSS)** | Eliminates microservice network latency, fits within Vercel free-tier limits, and provides serverless API routes with streaming UI. |

---

## 3. Algorithm Specification: $\text{DPSparseVoteRAG}$

1. **Top-$m \cdot k$ Retrieval:** Given marketing copy $x$, pure-TypeScript BM25 scores the 25-document private corpus and selects top $m \cdot k$ documents.
2. **Uniform Disjoint Partitioning:** The retrieved documents are shuffled and partitioned into $m$ disjoint subsets $D_x^1, \dots, D_x^m$, assigning $k$ documents per voter.
3. **Parallel Ensemble:**
   - 1 Non-RAG baseline voter reviews copy with zero retrieved context.
   - $m$ parallel subset voters review copy with strictly their assigned partition.
4. **Sparse Vector Technique (SVT) Budget Optimization:**
   - Threshold $\tau = m / 2$.
   - Noisy threshold: $\hat{\tau} = \tau + \text{Lap}(2 / \epsilon_{\text{Lap}})$.
   - Agreement count: $a_t = \sum_{i=1}^m \mathbf{1}[v_i == v_{\text{baseline}}]$.
   - Condition: $a_t + \text{Lap}(4 / \epsilon_{\text{Lap}}) \ge \hat{\tau}$.
   - **High-Consensus Case:** When consensus with the baseline holds, output baseline verdict with **ZERO privacy budget spent ($0\,\epsilon$)**.
   - **Disputed / Context-Dependent Case:** When consensus fails, invoke `LimitedDomain` bounded histogram selection with Laplace noise, consuming $(\epsilon_{\text{token}}, \delta_{\text{token}})$ from the `PrivacyAccountant`.

---

## 4. Getting Started & Setup

### Prerequisites
- Node.js 18+ (tested on Node v24)
- npm or pnpm

### Quickstart

1. **Clone repository & install dependencies:**
   ```bash
   git clone <repo-url>
   cd verisum
   npm install
   ```

2. **Configure Environment Variables:**
   ```bash
   cp .env.example .env.local
   ```
   *Edit `.env.local`:*
   ```env
   # Optional: Google Gemini API Key. If omitted, automatic deterministic mock voter is used.
   GEMINI_API_KEY=your_gemini_api_key

   # Set to true to force deterministic mock voters (ideal for CI/CD and offline tests)
   MOCK_LLM=true

   # Optional Postgres connection. If omitted, in-memory store with seeded demo reviews is used.
   # DATABASE_URL=postgres://...
   ```

3. **Run Seed Script:**
   ```bash
   npm run seed
   ```

4. **Start Development Server:**
   ```bash
   npm run dev
   ```
   Open [http://localhost:3000](http://localhost:3000) in your browser.

5. **Run Automated Tests:**
   ```bash
   npm test
   ```

6. **Production Build:**
   ```bash
   npm run build
   ```

---

## 5. Environment Variables Reference

| Variable | Required | Default | Description |
| :--- | :--- | :--- | :--- |
| `GEMINI_API_KEY` | Optional | `""` | Google Gemini API key. If unset or on HTTP 429 rate limit, falls back to deterministic rule voter. |
| `MOCK_LLM` | Optional | `false` | When set to `true`, forces mock rule voter execution for reproducible benchmarks. |
| `DATABASE_URL` | Optional | `""` | PostgreSQL connection string. If omitted, uses global in-memory persistence with pre-seeded demo submissions. |
| `PORT` | Optional | `3000` | Local development port. |

---

## 6. Page Navigation

- `/` : Landing page introducing the three core values and interactive algorithm flow.
- `/submit` : Copy submission form with 3 pre-loaded hackathon demo presets and configurable DP parameters ($m, k, \epsilon$).
- `/review/[id]` : Human-in-the-loop review interface with severity findings, guideline citations, privacy meter, and reviewer decision actions.
- `/audit` : Central audit log table with filterable decision history and CSV export.
- `/eval` : 40-sample benchmark evaluation dashboard calculating precision, recall, median latency, epsilon sweep curve, and Membership Inference Attack ($S^2\text{MIA}$) AUC comparison.
