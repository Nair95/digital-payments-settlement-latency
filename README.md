# Digital Payments Settlement Latency & Merchant Churn Analytics

**Group 03 — Programming for Analytics, Comprehensive Group Capstone Project & Live Code Defense.**
Analytics-consulting engagement for a digital payments aggregator (530,000 transactions, 8,500
merchants, UPI / cards / NetBanking; revenue = Merchant Discount Rate on processed volume), advising
the **COO** and **Head of Merchant Acquiring** on two suspected operational breakdowns:

1. **Gateway/switch latency** — intermittent issuing-bank downtime causing transaction drop-offs.
2. **Settlement delays** — payout SLA breaches (T+2/T+3 instead of contracted T+0/T+1) stressing
   merchant working capital and driving churn.

## Headline findings

| Finding | Evidence |
|---|---|
| Smart-routing lift verified | **+8.57 pp** success rate (79.40% → 87.97%), inside the 8.30–8.70 benchmark band; latency −39% (1,700 → 1,035 ms) |
| Settlement breaches are structural | T+0/T+1 breach ~12% vs T+2/T+3 ~50%; flat ~28%/month (no seasonality) |
| Breach exposure drives churn | SMB merchants with >25% batches breaching churn **32.66%** vs **10.12%** low-breach (3.2×); 49% of SMB churners cite settlement delay |
| The prize | At-risk SMB pool = ₹51.6M annualized GMV / ₹0.67M MDR; fixing delays protects ≈ **₹11.6M GMV / ₹151K MDR per year** |

## Repository map

| Path | What it is |
|---|---|
| `Group_03_Digital_Payments_Settlement_Latency.ipynb` | **TEMPLATE notebook** — same structure with numbered beginner steps (≤ 3 lines of code per step); runs clean before solving |
| `Group_03_Digital_Payments_Settlement_Latency_SOLUTION.ipynb` | **SOLUTION notebook** — fully executed analysis (all 8 KBQs, 8 charts, 4 multivariate diagnostics, 5 quantified recommendations); passes Restart → Run All |
| `payments_analytics.db` | Provided SQLite database (5 relational tables; sole data source) — keep beside the notebooks |
| `requirements.txt` | Pinned runtime environment (numpy / pandas / matplotlib / seaborn) |
| `pyproject.toml` + `uv.lock` + `.python-version` | uv-managed environment definition (CPython 3.13.14) |
| `docs/analysis_and_insights.md` | Written analysis: findings per KBQ + recommendation table |
| `docs/decision_log.md` | Every material analytical decision + corrections (append-only) |
| `dashboard/build_extracts.ipynb` | Regenerates the 7 Power BI source extracts into `dashboard/extracts/` |
| `docs/dashboard_spec_powerbi.md` | 4-page Power BI dashboard specification |
| `slides/` | Executive deck (PPTX + PDF, 10 slides) |
| `PROJECT_BLUEPRINT.md` | Requirements contract distilled from the course brief |
| `docs/PROGRESS.md` | Session work log / resume state |

## Running the notebooks

The notebooks use **relative paths only** — keep them beside `payments_analytics.db` and run
*Restart Kernel → Run All Cells*. Python ≥ 3.13.

**Option A — uv (recommended, exact reproduction):**

```bash
uv sync                                  # creates .venv from uv.lock (CPython 3.13.14)
uv run jupyter lab                       # open either notebook
# batch validation:
uv run jupyter nbconvert --to notebook --execute --inplace Group_03_Digital_Payments_Settlement_Latency_SOLUTION.ipynb
```

**Option B — pip:**

```bash
python -m venv .venv && .venv\Scripts\activate        # Windows; source .venv/bin/activate on POSIX
pip install -r requirements.txt jupyter
jupyter lab
```

Delivered notebooks are already fully executed; re-running is for validation.

**Architecture:** SQL (Python's built-in `sqlite3`) **extracts** the five tables once into labelled
DataFrames; **pandas does all the work afterwards** — cleaning, integrity validation, EDA, aggregation,
diagnostics and feature engineering.
**Libraries:** `numpy`, `pandas`, `matplotlib`, `seaborn` (+ stdlib `sqlite3`) — nothing else.

## Data & method in one paragraph

Star-schema SQLite (3 dimensions → 2 facts: 530k transactions, 125k settlements). SQL extracts the
tables; pandas validates the relational contract before any analysis (duplicate-PK counts, left-merge
orphan tests with the `_merge` indicator, the settlement ledger identity), engineers calendar and
exposure features (`df_merchant_exposure`), and produces every aggregate via `groupby().agg()`,
`pivot_table()` and `groupby().quantile()` — documented as decisions Q1–Q6 in the notebook. Four
multivariate diagnostics (MCC × cycle breach matrix; breach-bucket × tier churn; routing lift by tier
and channel; route × bank latency) support the causal story: infrastructure-side failures and
long-cycle settlement breaches concentrate in the SMB segment that pays 65% of platform MDR — and both
are addressable with routing, liquidity and trigger-based retention.

## Submission package

`Group_03_Digital_Payments_Settlement_Latency.zip` = the two notebooks + renamed database
(`Group_03_Digital_Payments_Settlement_Latency.db`) + deck PDF + `requirements.txt`, all co-located at
the folder root (evaluator: extract and run — no path edits, no configuration).
