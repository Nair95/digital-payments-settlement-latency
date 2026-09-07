# Decision Log — Group 03 Capstone

> Every material analytical/methodological decision, with rationale. Decisions D1–D7 were taken during
> exploration profiling; D8+ during build. Corrections are logged at the bottom (append-only).

## Methodological decisions

- **D1 — Naming & metadata.** Deliverables carry Group 03 naming
  (`Group_03_Digital_Payments_Settlement_Latency[_SOLUTION].ipynb`). Member metadata uses placeholders
  (Members A–G, SAP ID / Roll No / Email / 100% contribution each) pending real names.
- **D2 — EDA lives inside the SOLUTION notebook.** No standalone analysis `.py` files in the deliverable
  tree; the notebook is the single analytical artifact (user directive). Builder scripts live in
  `exploration/` (gitignored agentic scratch).
- **D3 — Settlement-churn analysis scoped to covered merchants.** Only 2,517 of 8,500 merchants appear
  in `fact_merchant_settlements`. Overall covered vs uncovered churn is nearly identical
  (16.09% vs 16.26%), so partial coverage does not select on churn; exposure→churn comparisons are run
  within the covered population and the gap is disclosed wherever settlement-based figures appear.
- **D4 — "Timeout" counting discipline.** `status='TIMEOUT'` rows (21,611) and
  `error_code='ERR_BANK_TIMEOUT'` on FAILED rows (17,567) are different populations and are never
  summed or mixed. Additionally: `USER_DROPPED ⇔ ERR_AUTH_FAILED` is a strict 1:1 pairing (12,799);
  `ERR_AUTH_FAILED` also appears on 15,646 FAILED rows (total AUTH_FAILED 28,445) — pairing claims are
  made only about the USER_DROPPED population.
- **D5 — SMB breach-exposure buckets.** Merchant-level breach rate = share of their settlement batches
  with `sla_breach_flag=1`; buckets: `0%`, `(0–10]%`, `(10–25]%`, `>25%`. Buckets are computed once in
  a shared CTE (`MERCHANT_BREACH_CTE`) reused by KBQ 4 and KBQ 8 so both use one definition.
- **D6 — Annualization method (protected volume).** Trailing 18-month run-rate × 12/18. Justified by
  flat monthly breach (27.7–29.5%) and flat monthly realized MDR (₹0.152–0.165M). The 2025-01 stub
  month (₹0.0084M, late-settling T+2/T+3 batches) is excluded from trend/annualization windows; the
  underlying rows are kept (valid settlements).
- **D7 — Realized vs potential MDR accounting.** Realized = `net_mdr_deducted_inr` on settled batches
  (covers the 2,517 settled merchants; 18-mo ₹2.831M / ₹217.72M GPV). Potential = successful
  transaction volume × `dim_payment_methods.mdr_rate_pct` (all merchants). The two lenses are never
  mixed; the coverage gap explains realized < potential.
- **D8 — Dedicated uv environment (reproducibility).** `pyproject.toml` (pandas / sqlalchemy /
  matplotlib; dev: ipykernel / nbformat / nbconvert) + `uv.lock` + `.python-version` pinned to
  uv-managed CPython 3.13.14. `uv sync` rebuilds anywhere with uv; evaluators without uv use the
  fully pinned `requirements.txt` (runtime deps only, `uv export --no-dev`). Notebook kernelspec stays
  default `python3` — no custom kernel dependency. The delivered notebooks are **executed in this env**
  via `uv run jupyter nbconvert --execute` (not the Anaconda base or the MCP server's env), so the
  executed outputs are exactly what the declared environment produces.

## Analytical conventions

- **Success rate** = SUCCESS ÷ all attempts (per brief), implemented as one shared SQL expression.
- **p50/p95 latency** computed in SQL via `ROW_NUMBER()` rank selection (SQLite lacks PERCENTILE_CONT).
- **Aggregation pushdown:** all heavy grouping runs in SQLite; Python receives small frames (brief's
  efficiency recommendation). The only large pulls are single-column latency samples for histograms.
- **Multivariate diagnostics (≥1 required; 4 delivered):** MCC × cycle breach matrix (KBQ 3);
  breach-bucket × tier churn cross-tab (KBQ 4); route lift by tier and by channel (KBQ 5);
  route × bank latency comparison (KBQ 6).
- **Benchmark assertion:** the SOLUTION notebook asserts the routing lift lands in 8.30–8.70 pp and
  fails loudly otherwise. The TEMPLATE keeps this as a commented benchmark.
- **No hardcoded results in narrative cells:** all finding/exec-summary numbers are interpolated from
  computed variables (restart-safe — nothing depends on execution order across blocks; each block
  recomputes what it needs).

## Corrections (append-only)

- **C1 — Protected-volume arithmetic.** Draft estimate (₹13.0M GMV / ₹169K MDR) divided the excess
  churn share by the segment churn rate (22.54/32.66), inflating the protected share ~3×. Correct
  protected share = excess pp of the segment (894 × 22.54% ≈ 202 merchants) → protected ≈
  **₹11.6M GMV / ₹151K MDR per year**. Fixed in notebook and docs.
- **C2 — USER_DROPPED pairing precision.** Earlier note recorded "USER_DROPPED↔ERR_AUTH_FAILED 1:1
  (12,799)"; first notebook build quoted the total ERR_AUTH_FAILED row count (15,646 + 12,799 = 28,445)
  in the pairing sentence. Corrected to state both counts explicitly (D4).
- **C3 — Restart-safety fixes caught by execution.** Two cells referenced objects defined in later
  blocks (`parity`, `kpi`); fixed by moving the coverage check into the data-quality block and
  deriving the merchant total from the same query that uses it. Root cause of both: narrative cells
  citing aggregates computed for the story rather than the block's own inputs.
- **C4 — Environment note.** The uv env resolves pandas 3.0.5 (verified numbers were originally
  profiled under pandas 2.x/Anaconda). Full re-execution in the uv env reproduced every verified
  number exactly (see `PROGRESS.md` "Verified numbers"), so no version contingency remains.
