# Data Dictionary: Digital Payments Settlement & Churn Analytics

**Database File:** `data/raw/payments_analytics.db`  
**RDBMS Engine:** SQLite 3 (ACID compliant with Foreign Key enforcement enabled)  
**Entities:** 3 Dimension Tables, 2 Fact Tables  
**Total Scale:** 8,500+ Merchants, 12 Issuing Banks, 16 Payment Methods, 530,000+ Transactions, 125,000+ Settlements  

---

## Relational Schema Overview

| Table Name | Entity Type | Grain / Primary Key | Approximate Records | Purpose |
|------------|-------------|---------------------|---------------------|---------|
| `dim_merchants` | Dimension | `merchant_id` (PK) | 8,500+ | Merchant profile, category code (MCC), tier (Enterprise/SMB), SLA tier, and churn status. |
| `dim_issuing_banks` | Dimension | `bank_id` (PK) | 12 | 12 major Indian issuing banks across PSU & Private sectors with baseline network reliability. |
| `dim_payment_methods` | Dimension | `payment_method_id` (PK) | 16 | Payment instrument master (UPI, Credit Cards, Debit Cards, NetBanking, Wallets) and MDR rates. |
| `fact_payment_transactions` | Fact | `txn_id` (PK) | 530,000+ | Individual payment transaction audit logs with timestamps, status, error codes, and latency in ms. |
| `fact_merchant_settlements` | Fact | `settlement_id` (PK) | 125,000+ | Daily payout reconciliation ledger tracking gross volume, MDR deductions, expected vs actual payout dates. |

---

## Detailed Column Specifications

| Table Name | Column Name | Data Type | Constraint | Description |
|------------|-------------|-----------|------------|-------------|
| `dim_merchants` | `merchant_id` | `VARCHAR(16)` | `PRIMARY KEY` | Unique merchant account identifier (format `MCH_XXXXXX`). |
| `dim_merchants` | `business_name` | `VARCHAR(100)` | `NOT NULL` | Registered trading business or storefront name. |
| `dim_merchants` | `mcc_category` | `VARCHAR(30)` | `NOT NULL` | Merchant Category Code: `'E_COMMERCE'`, `'FOOD_DINING'`, `'TRAVEL_HOSPITALITY'`, `'UTILITIES_BILLS'`, `'GAMING_ENTERTAINMENT'`, `'HEALTHCARE_RETAIL'`, `'EDTECH'`. |
| `dim_merchants` | `onboarding_date` | `DATE` | `NOT NULL` | Date when the merchant completed KYC and went live (`2023-01-01` to `2024-06-30`). |
| `dim_merchants` | `merchant_tier` | `VARCHAR(20)` | `NOT NULL` | Volume tier: `'ENTERPRISE'`, `'MID_MARKET'`, `'SMB'`. |
| `dim_merchants` | `settlement_cycle_sla` | `VARCHAR(15)` | `NOT NULL` | Contracted payout turnaround: `'T_PLUS_0'`, `'T_PLUS_1'`, `'T_PLUS_2'`, `'T_PLUS_3'`. |
| `dim_merchants` | `is_active` | `INTEGER` | `NOT NULL, CHECK (0,1)` | Binary active status (1 = Actively processing, 0 = Churned/Deactivated). |
| `dim_merchants` | `churn_date` | `DATE` | `NULLABLE` | Date merchant stopped processing, or `NULL` if active. |
| `dim_merchants` | `churn_reason` | `VARCHAR(30)` | `NULLABLE` | Stated or modeled churn driver: `'SETTLEMENT_DELAY'`, `'HIGH_FAILURE_RATE'`, `'HIGH_MDR_FEES'`, `'COMPETITOR_SWITCH'`, or `NULL`. |
| `dim_issuing_banks` | `bank_id` | `VARCHAR(10)` | `PRIMARY KEY` | Unique bank code (e.g. `BNK_01` to `BNK_12`). |
| `dim_issuing_banks` | `bank_name` | `VARCHAR(50)` | `NOT NULL` | Commercial name (e.g. HDFC Bank, ICICI Bank, State Bank of India). |
| `dim_issuing_banks` | `bank_code` | `VARCHAR(10)` | `NOT NULL` | Standard IFSC routing prefix (`HDFC`, `ICIC`, `SBIN`, `UTIB`, `KKBK`, `PUNB`, `BARB`, `INDB`, `YESB`, `CNRB`, `UBIN`, `FDRL`). |
| `dim_issuing_banks` | `tier` | `VARCHAR(20)` | `NOT NULL` | Sector classification: `'TIER_1_PVT'`, `'TIER_1_PSU'`, `'TIER_2_PVT'`. |
| `dim_issuing_banks` | `base_network_reliability_pct` | `DECIMAL(5,2)` | `NOT NULL` | Baseline network uptime percentage (88.5% to 97.5%). |
| `dim_payment_methods` | `payment_method_id` | `VARCHAR(20)` | `PRIMARY KEY` | Unique payment method identifier (e.g. `PM_UPI_NPCI`, `PM_CC_VISA`). |
| `dim_payment_methods` | `payment_channel` | `VARCHAR(20)` | `NOT NULL` | Core channel: `'UPI'`, `'CREDIT_CARD'`, `'DEBIT_CARD'`, `'NET_BANKING'`, `'WALLET'`. |
| `dim_payment_methods` | `network_provider` | `VARCHAR(30)` | `NOT NULL` | Network switch: `'NPCI'`, `'VISA'`, `'MASTERCARD'`, `'RUPAY'`, `'DIRECT_NETBANKING'`. |
| `dim_payment_methods` | `mdr_rate_pct` | `DECIMAL(5,2)` | `NOT NULL` | Merchant Discount Rate in percentage (0.00% for UPI, 0.40%–2.40% for cards/netbanking). |
| `fact_payment_transactions` | `txn_id` | `VARCHAR(20)` | `PRIMARY KEY` | Unique transaction ID (format `TXN_XXXXXXXX`). |
| `fact_payment_transactions` | `merchant_id` | `VARCHAR(16)` | `FOREIGN KEY (dim_merchants)` | Transacting merchant ID. |
| `fact_payment_transactions` | `bank_id` | `VARCHAR(10)` | `FOREIGN KEY (dim_issuing_banks)` | Cardholder / Payer issuing bank. |
| `fact_payment_transactions` | `payment_method_id` | `VARCHAR(20)` | `FOREIGN KEY (dim_payment_methods)` | Instrument configuration used. |
| `fact_payment_transactions` | `txn_timestamp` | `DATETIME` | `NOT NULL` | Transaction initiation timestamp (`2023-07-01` to `2024-12-31`). |
| `fact_payment_transactions` | `amount_inr` | `DECIMAL(12,2)` | `NOT NULL` | Transaction ticket value in Indian Rupees (INR). |
| `fact_payment_transactions` | `status` | `VARCHAR(20)` | `NOT NULL` | Transaction status: `'SUCCESS'`, `'FAILED'`, `'USER_DROPPED'`, `'TIMEOUT'`. |
| `fact_payment_transactions` | `error_code` | `VARCHAR(30)` | `NOT NULL` | Error diagnostic: `'ERR_NONE'`, `'ERR_BANK_TIMEOUT'`, `'ERR_INSUFFICIENT_FUNDS'`, `'ERR_NPCI_DEGRADED'`, `'ERR_AUTH_FAILED'`, `'ERR_SWITCH_UNAVAILABLE'`. |
| `fact_payment_transactions` | `latency_ms` | `INTEGER` | `NOT NULL` | End-to-end switch response latency in milliseconds (250ms to 12,000ms). |
| `fact_payment_transactions` | `is_routed_via_dynamic_sla` | `INTEGER` | `NOT NULL, CHECK (0,1)` | Binary smart-routing flag (1 = Dynamic SLA smart route, 0 = Standard static route). |
| `fact_merchant_settlements` | `settlement_id` | `VARCHAR(20)` | `PRIMARY KEY` | Unique batch settlement ID (format `SETTLE_XXXXXXXX`). |
| `fact_merchant_settlements` | `merchant_id` | `VARCHAR(16)` | `FOREIGN KEY (dim_merchants)` | Merchant receiving funds. |
| `fact_merchant_settlements` | `settlement_batch_date` | `DATE` | `NOT NULL` | Date of captured daily transaction batch. |
| `fact_merchant_settlements` | `gross_volume_inr` | `DECIMAL(14,2)` | `NOT NULL` | Aggregated gross successful transaction amount for batch. |
| `fact_merchant_settlements` | `net_mdr_deducted_inr` | `DECIMAL(12,2)` | `NOT NULL` | Total MDR platform fee deducted from batch. |
| `fact_merchant_settlements` | `net_settled_amount_inr` | `DECIMAL(14,2)` | `NOT NULL` | Net funds transferred to merchant bank account (`gross - mdr`). |
| `fact_merchant_settlements` | `expected_settlement_date` | `DATE` | `NOT NULL` | Contractual payout SLA date (`batch_date + SLA days`). |
| `fact_merchant_settlements` | `actual_settlement_date` | `DATE` | `NOT NULL` | Date payout actually cleared in merchant account. |
| `fact_merchant_settlements` | `settlement_delay_days` | `INTEGER` | `NOT NULL` | Days of delay beyond expected SLA date (`>= 0`). |
| `fact_merchant_settlements` | `sla_breach_flag` | `INTEGER` | `NOT NULL, CHECK (0,1)` | Binary breach indicator (1 = Delay > 0 days, 0 = On time). |

---

## Payments Domain Glossary & Key Formulas
1. **Merchant Discount Rate (MDR):** Fee charged to a merchant for processing digital payment transactions (expressed as a percentage of gross ticket value).
2. **Settlement SLA ($T+n$):** Standard banking settlement turnaround time:
   - $T+0$: Same-day instant/evening settlement
   - $T+1$: Next business day settlement
   - $T+2 / T+3$: Multi-day batch settlement
3. **Transaction Success Rate (SR %):**
   $$\text{SR} = \frac{\sum \text{Transactions with status = 'SUCCESS'}}{\text{Total Transaction Attempts}} \times 100\%$$
4. **Dynamic SLA Smart-Routing:** An infrastructure optimization layer that dynamically monitors issuing bank latency and switch availability, automatically routing around degraded gateways to boost success rates by **+8.5%**.
