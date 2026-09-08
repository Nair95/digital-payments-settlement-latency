# Analysis & Insights — Digital Payments Settlement Latency & Merchant Churn Analytics

> Group 03 · Programming for Analytics capstone. Companion to the executed SOLUTION notebook
> (`Group_03_Digital_Payments_Settlement_Latency_SOLUTION.ipynb`). Every number below is computed live
> in that notebook from `payments_analytics.db` — this document narrates the same evidence for readers.

## Executive summary

A digital payments aggregator (530,000 transactions, 8,500 merchants, 18 months) suffers elevated SMB
merchant churn. Diagnosis across the two suspected failure modes:

1. **Gateway latency** is concentrated in PSU bank switches and in the 18:00–22:00 evening window, and
   ~49% of all failures are infrastructure-attributable. The Dynamic SLA Smart-Routing engine already
   fixes most of it: **+8.57 pp absolute success-rate lift (79.40% → 87.97%)** — inside the benchmark
   band 8.30–8.70 pp — and **−665 ms average latency (−39%)**.
2. **Settlement delays** breach SLA on ~28.4% of batches, with a structural split: T+0/T+1 contracts
   breach ~12% while T+2/T+3 contracts breach ~50%. Breach exposure is the controllable churn driver:
   SMB merchants with >25% of batches breaching churn at **32.66%** vs **10.12%** for low-breach peers
   (3.2×), and 49% of SMB churners cite settlement delay as the reason.
3. **The prize:** SMB merchants pay 65% of platform MDR (₹1.85M of ₹2.83M over 18 months). The at-risk
   SMB segment (894 merchants breaching >25% of the time) carries **₹51.6M annualized GMV and ₹0.669M
   annualized MDR**; eliminating delays protects ≈ **₹11.6M GMV / ₹151K MDR per year** even under the
   conservative excess-churn scenario.

## Part 1 — Gateway reliability & bank switch latency (KBQ 1–2)

### KBQ 1 · Issuing-bank profiling
- PSU banks (Tier-1 PSU) average **1,433 ms** vs 1,322 ms for Tier-1 private (+8%) and spike (≥10 s)
  on **1.92%** of attempts vs 1.47%.
- Worst banks are all PSU: Bank of Baroda (1,442.5 ms avg), Union Bank of India, Punjab National Bank;
  best performer: IndusInd (1,310.8 ms). p95 spans roughly 4.1 s (best private) to 7.0 s (worst PSU).
- Intraday: success rate is flat (83.3–84.1% across hours) — the stress shows in **latency**, not
  completion: evenings (18:00–22:00) average **1,541 ms vs 1,321 ms off-peak (+17%)** with spike counts
  of ~508/hr vs ~329/hr off-peak. An experience-level capacity ceiling that precedes hard failures.
- *Implication:* capacity/routing fixes should target PSU-bound evening traffic; the problem is
  addressable operationally (routing + SLOs), not a random nuisance.

### KBQ 2 · Error Pareto
- 86,437 failed attempts = **16.31%** of all attempts. Ranking: `ERR_BANK_TIMEOUT` 45.3%,
  `ERR_AUTH_FAILED` 32.9%, `ERR_INSUFFICIENT_FUNDS` 18.3%, `ERR_NPCI_DEGRADED` 1.75%,
  `ERR_SWITCH_UNAVAILABLE` 1.75%.
- **Upstream banking infrastructure = 48.83%** of failures vs 51.17% user-side.
- Pairing facts (kept precise): every `USER_DROPPED` attempt (12,799) carries `ERR_AUTH_FAILED` 1:1;
  the same code also marks 15,646 `FAILED` attempts. Accounting note: `status='TIMEOUT'` rows (21,611)
  and `ERR_BANK_TIMEOUT`-on-FAILED rows (17,567) are distinct counts and are never summed.
- *Implication:* roughly half the failure load is controllable by the platform's routing layer — the
  exact lever quantified in Part 3.

## Part 2 — Settlement SLA breaches & merchant churn (KBQ 3–4)

### KBQ 3 · Settlement delay diagnostics
- Breach rate by contracted cycle: **T+0 12.22% · T+1 11.96% · T+2 49.56% · T+3 51.74%** — a 4×
  structural gap, not noise. Breached batches arrive ~1.5 days late on average.
- Delay-day distribution: 89,460 batches settle on time; 20,062 +1d; 10,419 +2d; 4,116 +3d; 943 +4d
  (long right tail).
- Monthly breach rate is flat (27.7–29.5%) across 2023-07 → 2024-12 — no seasonality, which justifies
  run-rate annualization in Part 4.
- The MCC × cycle matrix shows elevated T+2/T+3 breach rates across **every** merchant category —
  platform-wide batch-pipeline failure, not a vertical-specific one.

### KBQ 4 · Settlement friction vs attrition (multivariate diagnostic)
- Churn by tier: SMB **21.67%**, MID_MARKET 7.41%, ENTERPRISE 3.16% (overall 16.21%).
- Churn by breach-exposure bucket × tier: SMB >25% bucket **32.66%** vs 0–10% bucket **10.12%**
  (3.2×). MID_MARKET shows the same direction (14.43% vs 4.14%); ENTERPRISE barely moves — small
  merchants absorb the working-capital shock.
- Coverage validity: only 2,517 of 8,500 merchants appear in the settlement fact, but covered vs
  not-covered churn is nearly identical (16.09% vs 16.26%) — exposure analysis within the covered
  population is unbiased.
- Churn reasons (SMB): SETTLEMENT_DELAY 581 (49% of SMB churn), HIGH_FAILURE_RATE 281,
  COMPETITOR_SWITCH 227, HIGH_MDR_FEES 93. Churned merchants leave after ~148 days regardless of tier.
- *Implication:* breach exposure is the controllable churn driver; the churn cliff sits above the 25%
  breach-share mark — a natural trigger threshold for a save-desk.

## Part 3 — Dynamic SLA Smart-Routing performance (KBQ 5–6)

### KBQ 5 · A/B success-rate lift
- Standard route: **79.40%** (n=264,736) vs dynamic route: **87.97%** (n=265,264) →
  **+8.57 pp absolute lift** — inside the benchmark band (target +8.50 pp; accepted 8.30–8.70 pp).
- Lift is stable across segments: by tier 8.21–8.74 pp; by channel 8.04–9.18 pp — a platform-wide
  engine effect, not a segment artifact.

### KBQ 6 · Latency mitigation
- Average latency 1,700 → 1,035 ms (**−665 ms, −39%**); p95 7,335 → 3,793 ms.
- Route × bank: on every degraded PSU switch the dynamic route claws back ~666 ms on average —
  the engine effectively neutralizes the worst-switch penalty.
- *Implication:* extending dynamic routing to 100% of traffic is the single highest-leverage
  operational action; ~264,736 attempts were still standard-routed in the window.

## Part 4 — Merchant tier economics & revenue optimization (KBQ 7–8)

### KBQ 7 · MDR revenue contribution
- Realized MDR (settled batches, 18 mo): **₹2.831M on ₹217.72M settled GPV** (blended 1.30%).
  By tier: SMB ₹1.847M / ₹142.27M GPV; MID_MARKET ₹0.761M / ₹58.38M; ENTERPRISE ₹0.223M / ₹17.07M.
  **SMB = 65% of platform MDR.**
- Potential MDR (successful transactions × contracted rates): ≈ ₹4.65M. **UPI = 55% of successful GMV
  (₹400.8M) at 0% MDR**; credit cards = ~20% of GMV but ~56% of potential MDR (rates 1.2–2.4%);
  NetBanking/debit/wallet share the rest. Monetization is card-concentrated; volume is UPI-concentrated.
- Monthly realized MDR is flat (₹0.152–0.165M); a 2025-01 stub (₹0.0084M) exists from late-settling
  T+2/T+3 batches and is excluded from trend/annualization windows.

### KBQ 8 · Protected volume sizing
- At-risk SMB (breach >25% of batches): **n = 894**, observed churn **32.66%** (292 merchants);
  18-month GPV ₹77.43M, MDR ₹1.0028M → annualized (×12/18) **₹51.62M GMV / ₹0.669M MDR at stake**.
- Excess-churn scenario: excess churn = 32.66% − 10.12% = **22.54 pp** → ~202 saveable merchants →
  protected ≈ **₹11.6M GMV / ₹151K MDR per year**.
- Assumptions: (1) trailing 18-month run-rate annualization (flat breach/MDR trend verified);
  (2) post-fix churn reverts to the low-breach SMB baseline; (3) retained merchants sustain
  segment-average GPV/MDR economics; (4) no credit taken for replacement acquisition (conservative).

## Recommendations (Finding → Problem → Recommendation → Impact)

| # | ★ | Recommendation | Quantified impact (annual, stated assumptions) |
|---|---|----------------|------------------------------------------------|
| 1 | ★ | T+1 settlement guarantee / liquidity buffer for breach-exposed SMB; stop new SMB onboarding on T+2/T+3 | Protects ≈ ₹11.6M GMV / ₹151K MDR per year (excess-churn scenario; KBQ 8 assumptions) |
| 2 | ★ | Route 100% of traffic via Dynamic SLA engine; evening throttling toward healthy rails; bank-switch SLA scorecard with worst-3 renegotiation | ≈ 22.7k recovered successful txns / 18 mo ≈ ₹36M GMV (platform-average ticket; routing lift 8.57 pp on still-standard-routed volume) |
| 3 | ★ | Save-desk trigger when a merchant's rolling 30-day breach share crosses 20% (fee holiday / priority settlement) | ¼ of the ~202 excess churners ≈ ₹2.9M GMV / ₹38K MDR per year on top of Rec 1 (uptake ≥ 50%) |
| 4 |   | Monetize UPI base via value-added rails (instant-settlement SaaS, working capital), not per-txn fees | 10% attach on 5,455 SMB merchants × ₹1,000/mo ≈ ₹6.5M/yr (no transaction-fee change) |
| 5 |   | Per-switch evening SLOs (p95 < 3 s), pre-scale 17–23 h, automated shedding | ~1% abandonment improvement on ~21% of daily volume ≈ ₹0.09–0.1M GMV/month protected |

## Reproducibility

- Notebook executes end-to-end via *Restart Kernel → Run All*; relative paths only; environment pinned
  in `pyproject.toml` + `uv.lock` (uv) and `requirements.txt` (pip). See `decision_log.md` for every
  material analytical decision.
