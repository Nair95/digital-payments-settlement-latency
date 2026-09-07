# Project Blueprint: Digital Payments Settlement Latency & Merchant Churn Analytics

> **Purpose of this document:** Single source of truth for context management. Captures the technical goals, scope, constraints, requirements, deliverables, and analytical dimensions of the capstone project. Derived from `Capstone_Proeject_Details_Programming For Analytics.pdf` (universal protocol) and `Digital Payments Settlement Latency & Merchant Churn Analytics.pdf` (project brief). Schedule/deadline/administrative details are intentionally excluded.

---

## 1. Project Identity

| Attribute | Value |
|---|---|
| Course | Programming for Analytics — Comprehensive Group Capstone Project & Live Code Defense |
| Industry domain | FinTech / Digital Payments & Merchant Acquiring Infrastructure |
| Project title | Digital Payments Settlement Latency & Merchant Churn Analytics |
| Engagement framing | Analytics consulting engagement — team acts as a Senior Analytics Advisory Team reporting to the COO and Head of Merchant Acquiring of a digital payment aggregator |
| Client platform profile | Processes hundreds of thousands of daily transactions across UPI, Credit/Debit Cards, and NetBanking for 8,500+ merchants; revenue = Merchant Discount Rate (MDR) fees on processed volume |
| Database | `payments_analytics.db` (SQLite 3, strict FK enforcement) |
| Primary tooling | Python (sqlite3 / SQLAlchemy / pandas) + SQL against the relational DB; matplotlib (or equivalent) for visualization |

## 2. Business Problem (Technical Framing)

Two suspected operational breakdowns are driving elevated merchant churn, especially among high-velocity SMB merchants:

1. **Gateway/switch latency**: intermittent downtime across issuing-bank switches causing transaction drop-offs (timeouts, degraded authorization).
2. **Settlement delays**: batch payout breaches (T+2 / T+3 instead of contracted T+0 / T+1) creating merchant working-capital stress.

The commission: **diagnose gateway latency bottlenecks, correlate settlement SLA breaches with merchant attrition, and quantify the performance lift of the Dynamic SLA Smart-Routing engine.**

## 3. Technical Goals

1. Establish a validated, reproducible SQL→Python data pipeline over the relational SQLite database (no flattening into unrelated flat files).
2. Build the descriptive baseline: transaction volumes, success rates, latency distributions, settlement timeliness, revenue mix.
3. Perform diagnostic analytics to isolate drivers: which banks/tiers/time periods cause latency and failure; whether settlement SLA breaches causally track churn.
4. Quantify the Dynamic SLA Smart-Routing A/B lift and verify the benchmark (+8.50% absolute success-rate lift; target range 8.30%–8.70%; baseline ~79.5% → ~88.0%).
5. Quantify MDR revenue economics by merchant tier and payment method, and size protected volume (GMV + MDR revenue preserved by eliminating settlement delays for at-risk SMB merchants).
6. Convert findings into 5 concrete, prioritized, quantified business recommendations (top 3 highlighted for the executive deck).
7. Deliver a fully executed, error-free, restart-safe Jupyter notebook and a concise executive deck suitable for live code defense.

## 4. Scope

### In scope
- Analysis strictly on the provided `payments_analytics.db` (5 relational tables).
- Relational inspection, PK/FK validation, join integrity, data-quality treatment.
- Descriptive + diagnostic analytics per the 8 Key Business Questions (Section 6).
- Domain metric computation (success rate, SLA breach rate, ΔSR lift, net MDR revenue).
- Notebook, executive deck, recommendations with stated assumptions.

### Out of scope / prohibited
- Any external data or PII data merged into the analysis. All findings must derive from the provided project data only.
- Ignoring the relational structure (e.g., exporting tables to independent flat files and abandoning SQL).
- Purely descriptive work without a mandatory multi-factor/multivariate diagnostic.

## 5. Data Architecture

### 5.1 Entity overview (star-like schema: 3 dimensions → 2 facts)

| Table | Type | Grain / PK | ~Rows | Focus |
|---|---|---|---|---|
| `dim_merchants` | Dimension | `merchant_id` | 8,500+ | Profile, `mcc_category`, `merchant_tier` (ENTERPRISE / MID_MARKET / SMB), `settlement_cycle_sla` (T_PLUS_0…T_PLUS_3), `is_active`, `churn_date`, `churn_reason` |
| `dim_issuing_banks` | Dimension | `bank_id` | 12 | Indian commercial banks; `tier` (TIER_1_PVT / TIER_1_PSU / TIER_2_PVT); `base_network_reliability_pct` (88.5–97.5) |
| `dim_payment_methods` | Dimension | `payment_method_id` | 16 | Channels (UPI, CREDIT_CARD, DEBIT_CARD, NET_BANKING, WALLET), `network_provider` (NPCI, VISA, MASTERCARD, RUPAY, DIRECT_NETBANKING), `mdr_rate_pct` (0.00%–2.40%) |
| `fact_payment_transactions` | Fact | `txn_id` | 530,000+ | `txn_timestamp`, `amount_inr`, `status`, `error_code`, `latency_ms` (250–12,000 ms), `is_routed_via_dynamic_sla` (0/1) |
| `fact_merchant_settlements` | Fact | `settlement_id` | 125,000+ | `gross_volume_inr`, `net_mdr_deducted_inr`, `net_settled_amount_inr`, `expected_settlement_date`, `actual_settlement_date`, `settlement_delay_days` (≥0), `sla_breach_flag` (0/1) |

Full column-level specs (types, constraints, value domains): see `data_dictionary.md`. ASCII ER diagram: `schema_diagram.txt`.

### 5.2 Cardinality & referential integrity (all ON DELETE RESTRICT)

- `dim_merchants` (1) → (*) `fact_payment_transactions`
- `dim_merchants` (1) → (*) `fact_merchant_settlements`
- `dim_issuing_banks` (1) → (*) `fact_payment_transactions`
- `dim_payment_methods` (1) → (*) `fact_payment_transactions`

Relationships to understand/verify: 1:1, 1:N as above; N:M analysis arises via the fact tables bridging dimensions (e.g., merchant × bank × payment method through transactions).

### 5.3 Key value domains (authoritative per data dictionary)

- `fact_payment_transactions.status`: `SUCCESS`, `FAILED`, `USER_DROPPED`, `TIMEOUT` (the brief's narrative lists SUCCESS/FAILED/TIMEOUT — treat the dictionary's 4-value domain as authoritative for the actual DB).
- `error_code`: `ERR_NONE`, `ERR_BANK_TIMEOUT`, `ERR_INSUFFICIENT_FUNDS`, `ERR_NPCI_DEGRADED`, `ERR_AUTH_FAILED`, `ERR_SWITCH_UNAVAILABLE`.
- `churn_reason`: `SETTLEMENT_DELAY`, `HIGH_FAILURE_RATE`, `HIGH_MDR_FEES`, `COMPETITOR_SWITCH` (NULL when active).
- `mcc_category`: E_COMMERCE, FOOD_DINING, TRAVEL_HOSPITALITY, UTILITIES_BILLS, GAMING_ENTERTAINMENT, HEALTHCARE_RETAIL, EDTECH.
- Temporal coverage: transactions 2023-07-01 → 2024-12-31; merchant onboarding 2023-01-01 → 2024-06-30.
- MDR economics: UPI = 0%; cards/NetBanking = 0.40%–2.40% (brief cites Credit Card 1.8%–2.4% for the revenue-mix narrative).

## 6. Key Business Questions (KBQs) & Analytical Tasks

### Part 1 — Gateway Reliability & Bank Switch Latency Diagnostics
1. **Issuing-bank latency & downtime profiling**: transaction volume, average latency, timeout rates across all 12 banks; identify institutions with severe switch degradation (>10,000 ms latency spikes); compare PSU vs Private sector banks.
2. **Error-code Pareto analysis**: distribution of failure reasons (ERR_BANK_TIMEOUT, ERR_SWITCH_UNAVAILABLE, ERR_NPCI_DEGRADED, ERR_INSUFFICIENT_FUNDS); % of failures from upstream banking infrastructure vs user-side issues.

### Part 2 — Settlement SLA Breaches & Merchant Churn Dynamics
3. **Settlement delay diagnostics**: SLA breach rate (`sla_breach_flag = 1`) and average delay days per contracted payout schedule (T_PLUS_0 … T_PLUS_3).
4. **Settlement friction vs attrition**: churn rates segmented by SLA-breach frequency × merchant tier (SMB vs ENTERPRISE); quantify how much more likely SMB merchants are to churn under recurrent settlement delays.

### Part 3 — Dynamic SLA Smart-Routing Performance Lift
5. **A/B comparison**: success rate for `is_routed_via_dynamic_sla = 0` vs `= 1`. Benchmark to verify: **+8.50% absolute lift** (range 8.30%–8.70%), baseline ~79.5% → ~88.0%.
6. **Latency mitigation**: quantify average-latency (ms) reduction achieved by dynamically bypassing degraded bank switches.

### Part 4 — Merchant Tier Economics & Revenue Optimization
7. **MDR revenue contribution**: GPV and net MDR revenue by merchant tier × payment method (UPI 0% vs Credit Card 1.8%–2.4%).
8. **Protected volume sizing**: annualized GMV and MDR revenue preserved by eliminating settlement delays for at-risk SMB merchants.

## 7. Required Analytical Flow (mandatory pipeline shape)

```
Relational Data → Data Quality → Descriptive Analytics → Diagnostic Analytics
              → Business Recommendations → Quantified Impact
```

### 7.1 Data architecture, SQLite & cleaning (mandatory)
- Connect to the SQLite `.db` from Python (sqlite3, SQLAlchemy, pandas).
- Inspect the relational schema; validate PKs/FKs; understand 1:1 / 1:N / N:M relationships.
- Perform SQL/Python joins with verified correctness: **no unintended Cartesian multiplication; check for orphaned records** in relevant joins.
- Handle missing values, duplicates, datatypes, formatting issues, anomalies; **document every material data-quality decision**.
- Push preliminary transformation/filtering/joining down to SQL where appropriate (recommended to simplify Python and improve pipeline efficiency).

### 7.2 Descriptive analytics (baseline picture)
Summary statistics; distributions; transaction/operational volumes; status/category analysis; time trends; segment comparisons; relevant business KPIs. All visualizations must carry titles, axis labels, units, and readable formatting.

### 7.3 Diagnostic analytics (mandatory depth)
Move from "what happened" to "why": identify bottlenecks/drivers/contributors. **At least one multi-factor / multivariate diagnostic is mandatory** — e.g., cross-tabulation, cohort analysis, funnel/drop-off, grouped variance, correlation, segment×time, product×geography, customer×product. Conclusions must be evidence-backed from the data.

### 7.4 Business recommendations (5, prioritized)
Each recommendation must follow the chain: **Finding → Business Problem → Recommendation → Expected Impact**. Where applicable, estimate impact in revenue / cost / margin / conversion / retention / productivity / risk / operational efficiency. **Every impact estimate must state its assumptions.** Top 3 priority recommendations are highlighted for the deck.

## 8. Domain Formulas & KPI Definitions

| Metric | Formula / definition |
|---|---|
| Transaction Success Rate (SR %) | (`status = 'SUCCESS'` count ÷ total transaction attempts) × 100 |
| Settlement SLA turnaround (T+n) | T+0 same-day; T+1 next business day; T+2/T+3 multi-day batch |
| SLA Breach Rate | (settlement batches with `sla_breach_flag = 1` ÷ total batches) × 100 |
| Smart-routing lift (ΔSR) | SR(dynamic route = 1) − SR(standard route = 0) → expected ≈ +8.50% absolute |
| Net MDR platform revenue | Σ(gross settled volume × MDR rate %) |
| Merchant Discount Rate (MDR) | Processing fee charged to merchant, % of gross ticket value |
| Dynamic SLA Smart-Routing | Routing layer that monitors issuing-bank latency/switch availability and routes around degraded gateways |

## 9. Constraints

1. **Data privacy**: provided datasets exclusively; no external or PII data introduced or merged.
2. **Relational fidelity**: SQL (via Python) against the relational DB where appropriate; do not reduce to unrelated flat files.
3. **Relative paths only** in the notebook; evaluator must be able to run it with zero file-path edits, file moves, or manual configuration.
4. **Execution safety**: submitted notebook must be fully executed, fully populated, and error-free; must survive **Restart Kernel → Run All Cells** end-to-end (failures from broken paths, missing DB, syntax/runtime errors, missing dependencies, or manual intervention cause deductions).
5. **Environment reproducibility**: include `requirements.txt` listing essential libraries (e.g., pandas, sqlalchemy, matplotlib).
6. **Deliverable packaging**: one ZIP; `.ipynb`, `.db`, and deck PDF co-located in the same project folder; ZIP/folder named `Group_XX_Project_Name` (e.g., `Group_01_Customer_Retention`).
7. **Deck budget**: maximum 10 slides (universal protocol; note the brief's milestone text says 10–12 — treat ≤10 as the safe constraint); business story/evidence/diagnosis/recommendations, not a notebook reproduction.
8. **Notebook header (Markdown cell at very top)**: Group Name, Project Name, and per-member Team Member Name, SAP ID, Roll Number, Email ID, % Contribution (100% each if equal).
9. **Language/runtime**: Python ≥ 3.13 (per `pyproject.toml`); SQLite 3 storage engine.
10. **Defense readiness**: every group member must be able to explain all submitted code — SQL queries, joins, PK/FK relationships, cleaning, Python transformations, analytical calculations, visualizations, diagnostic logic, and business-impact assumptions.

## 10. Deliverables

| # | Deliverable | Format / location | Key acceptance criteria |
|---|---|---|---|
| D1 | Analysis notebook | `Group_XX_Project_Name.ipynb` (folder root, beside `.db`) | Top Markdown header cell; relative paths; fully executed & populated (code, SQL, tables/DataFrames, stats, charts, findings, recommendations); passes Restart→Run All |
| D2 | Database | `Group_XX_Project_Name.db` (the provided SQLite DB, renamed to match) | Unmodified relational structure; co-located with notebook |
| D3 | Executive deck | `Group_XX_Executive_Deck.pdf` | ≤10 slides; business story, evidence, diagnosis; top-3 recommendations with quantified impact |
| D4 | Environment spec | `requirements.txt` (recommended) | Essential libraries listed for replication |
| D5 | Submission package | `Group_XX_Project_Name.zip` containing the project folder | Extract-and-run: evaluator opens folder and runs notebook immediately |

### Milestone roadmap (technical deliverables per stage)
1. **Schema & Data Ingestion** — load SQLite DB; verify entity relationships, PK/FK integrity, table volume counts; signed-off Project Charter and Data Architecture documentation.
2. **EDA & Gateway Profiling** — EDA on transaction and settlement logs; hourly transaction success rates; bank-wise latency distributions; settlement-delay-day distribution across MCC categories.
3. **Churn Correlation & Smart-Routing Lift** — cross-tabs of SLA breach rates vs churn; compute the +8.5% success-rate lift; quantify upstream switch timeout bottlenecks (PSU vs Private).
4. **Executive Deck & Final Defense** — final deck for COO/Head of Merchant Acquiring; gateway-routing recommendations, payout SLA restructuring, revenue impact.

## 11. Technical Acceptance Checklist (final self-check)

- [ ] ZIP named `Group_XX_Project_Name.zip` containing the project folder with `.ipynb`, `.db`, deck PDF (+ `requirements.txt`)
- [ ] Notebook uses relative paths; runs end-to-end via Restart Kernel → Run All; fully executed with all outputs/charts visible
- [ ] PK/FK relationships and joins validated; no Cartesian multiplication; orphan checks performed
- [ ] Data-quality issues identified, handled, and documented
- [ ] Descriptive analysis complete (KPIs, distributions, trends, segments)
- [ ] Diagnostic analysis identifies drivers/bottlenecks; ≥1 multi-factor diagnostic included
- [ ] All 8 KBQs addressed
- [ ] +8.5% smart-routing lift verified against the 8.30%–8.70% target band
- [ ] 5 recommendations in notebook (Finding → Problem → Recommendation → Impact); 3 priority ones in deck
- [ ] Business impact quantified with stated assumptions
- [ ] Notebook header cell with full team metadata

## 12. Current Workspace State (context for continuation)

| File | Role |
|---|---|
| `payments_analytics.db` | The production SQLite database (~154 MB) — the sole data source |
| `data_dictionary.md` | Column-level data dictionary (authoritative value domains) |
| `schema_diagram.txt` | ASCII ER diagram + cardinality/FK actions |
| `pyproject.toml` | Python ≥ 3.13 project config (dependencies not yet declared) |
| `main.py` | Placeholder entry point |
| `README.md` | Empty — candidate pointer to this blueprint |
| `_extracted_raw.txt` / `_extracted_clean.txt` | Text extractions of the two source PDFs (clean = de-noised) |
| `PROJECT_BLUEPRINT.md` | This document |
