# Power BI Dashboard Specification — 4 Pages

> Group 03 · Digital Payments Settlement Latency & Merchant Churn Analytics.
> Audience: COO & Head of Merchant Acquiring. Build the model by importing the 7 CSVs from
> `dashboard/extracts/` (regenerate anytime via `dashboard/build_extracts.ipynb`).

## Model

| Extract | Grain | Rows | Used by |
|---|---|---|---|
| `monthly_overview.csv` | month | 18 | P1 trend, P4 header cards |
| `bank_performance.csv` | issuing bank | 12 | P1 league table + tier map |
| `error_pareto.csv` | error code | 5 | P1 Pareto |
| `settlement_performance.csv` | SLA cycle × batch month | 72 | P2 cycle bars + breach trend |
| `merchant_exposure.csv` | merchant (settled) | 2,517 | P3 churn visuals, all slicers |
| `routing_ab.csv` | route × tier × channel | 30 | P4 A/B lift |
| `channel_economics.csv` | tier × channel | 15 | P4 revenue matrix |

Relationships: single-table extracts (no joins required); `routed` renders as 0 = Standard, 1 = Dynamic SLA;
`breach_bucket` sort order: `0% → 0-10% → 10-25% → >25%`; months sorted ascending (`YYYY-MM`).

Suggested implicit measures: `SR = AVERAGE(success_rate_pct)` weighted by `txns` where present
(use `SUMX(routing, txns*success_rate_pct)/SUM(txns)` style weighted average); breach rate =
`SUM(breaches)/SUM(batches)`.

## Page 1 — Gateway Health (bank switch latency)

- **Cards:** overall success rate (83.7%), avg latency (≈1,431 ms), failed attempts (86.4K, 16.3%),
  infrastructure share of failures (48.8%).
- **Bar (horizontal):** avg latency by bank, colored by tier (PSU red / private blue) — shows the PSU
  block (BoB, Union, PNB worst) vs IndusInd best.
- **Line + clustered column:** monthly txns (columns) vs success rate (line); latency spikes by month
  (`latency_spikes`) as secondary columns.
- **Pareto combo:** failures by `error_code` (bars, colored by `fault_domain`) + `cumulative_pct` (line).
- **Slicers:** tier, bank, fault_domain.

## Page 2 — Settlement SLA Performance

- **Cards:** overall breach rate (~28.4%), batches settled on time (89,460), avg delay-if-breach (~1.5 d).
- **Clustered column:** breach % by `sla_cycle` (T+0/T+1 ~12% vs T+2/T+3 ~50% — the structural gap).
- **Line:** monthly breach % trend (flat 27.7–29.5% — no seasonality).
- **Matrix:** `sla_cycle` × `batch_month` breach heat (from `settlement_performance.csv`).
- **Slicers:** sla_cycle, year-month range.

## Page 3 — Merchant Churn & Exposure

- **Cards:** merchants covered (2,517), churn rate covered (16.09%), SMB churn (21.67% platform-wide),
  churn citing settlement delay (581 SMB merchants).
- **Clustered column:** churn % by `breach_bucket` split by `tier` — the 32.66% vs 10.12% SMB cliff.
- **Scatter:** merchant breach_rate (x) vs GPV (y, log), color by `is_churned`, size = `mdr_inr`.
- **Bar:** churned merchants by `churn_reason` for SMB (SETTLEMENT_DELAY leads).
- **Slicers:** tier, category, sla_cycle, breach_bucket.

## Page 4 — Smart-Routing & Revenue

- **Cards:** A/B success rates (79.40% standard vs 87.97% dynamic), lift (+8.57 pp), latency reduction
  (1,700 → 1,035 ms), realized MDR (₹2.83M / 18 mo).
- **Clustered column:** success rate by route (overall), then small multiples by tier and channel
  (from `routing_ab.csv`).
- **Matrix (heat):** `channel_economics.csv` — potential MDR, tier × channel; call out UPI = 0% row vs
  credit-card column (≈56% of potential MDR from ~20% of GMV).
- **Gauge/KPI:** protected volume — annualized GMV (₹51.6M at stake) and protected MDR (≈₹151K/yr,
  excess-churn scenario; assumptions per KBQ 8).
- **Slicers:** tier, channel, routed.

## Design conventions

- Currency in ₹ lakhs/crores or ₹M consistently (state units in every visual); percentages to 1 decimal.
- Consistent tier colors across pages: ENTERPRISE blue, MID_MARKET orange, SMB red.
- Every page carries a one-line takeaway subtitle (e.g., P2: "T+2/T+3 contracts breach at ~4× the
  T+0/T+1 rate — the churn cliff sits above 25% breach exposure").
- Dashboard is a deliverable supplement: all figures must reconcile to the notebook (same extracts
  pipeline, cross-checked SR/breach assertions in `build_extracts.ipynb`).
