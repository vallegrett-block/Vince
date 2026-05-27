# Money Movements Voice Weekly Scorecard Recipe

## Overview
This recipe refreshes the weekly **Voice Money Movements** scorecard on the Google Sheet:
- **Spreadsheet ID:** `1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s`
- **Tab:** Money Movements Voice
- **Data starts:** Column G
- **Row 1:** Weekly date headers (MM/DD/YYYY, Monday week-starts)

---

## Step 0: Identify Target Column

1. Read row 1 (G1:Z1) to get date headers.
2. Today is Monday. The most recent COMPLETED week is **last week** (prior Monday).
3. Find the column for that week. If it does not exist, add it.

---

## Step 1: Run Looker Dashboard 40389

Run Looker Dashboard **40389** with `timeout_limit=60`.

After getting the response (saved to a temp file due to size), use a **python3 shell script** to parse ONLY the needed values. Do NOT read the full JSON into conversation context.

Write a python3 script that:
1. Opens the temp file path from the Looker response
2. Loads the JSON
3. Iterates through tiles by `tile_name`
4. Extracts only the specific values needed for the target week
5. Prints a compact summary of extracted values

### Key Tile Mappings

| Tile / Source | Extraction Logic |
|---|---|
| **CSAT** (rows 3/4/5) | ⚠️ Do NOT use the CSAT tile from the dashboard (wrong tile-level filter overrides — uses Voice General house and Latin America only). Instead, run a **direct explore query** (see below). |
| **FCR %** | Monthly dates. Find row for target month. Extract `fcr_7_day_percent * 100` |
| **Complaint Flag Rate** | Weekly. Find row matching target week. Extract `complaint_flagging_rate * 100` |
| **ML Exception Rate** | Weekly. Find row matching target week. Extract `complaint_exception_rate * 100` |
| **QA Section Score** | Daily. Filter rows for target week dates (Mon–Sun). Average `avg_qa_eval_score`, `avg_customer_section_score`, `avg_business_section_score`, `avg_compliance_section_score` |
| **Complaints QA** | Weekly. Find row matching target week. Extract `total_yes_answers[Voice] / (yes + no) * 100` |
| **QA Acknowledgement** | Weekly. Find row matching target week. Extract `ACKED / (ACKED + AWAITING_ACK + DISPUTE_REJECTED) * 100` |
| **Handle Time** | Monthly. Find row for target month. Use `aht_numerical` (already in minutes), `average_wrap_time_hours * 24 * 60` |
| **Holds Per Call** | Monthly. Find row for target month. Use `hold_per_call` table calculation |
| **SLA** | Monthly. Find row for target month. Extract `service_level_percent * 100` |
| **Unresolved Cases** | Snapshot. Extract `case_count` from first row |

### CSAT Direct Explore Query

Since the dashboard CSAT tile has incorrect filter overrides, run a direct explore query:

- **Model:** `cash_support_pst`
- **View:** `cash_survey_results`
- **Fields:**
  - `cash_survey_results.survey_created_week`
  - `cash_survey_results.CSAT_greater_than_3_by_survey_hash_key`
  - `cash_survey_results.CSAT_survey_count_by_survey_hash_key`
  - `cash_survey_results.resolution_rate`
- **Filters:**
  - `cash_survey_results.house` = `Voice Money Movements`
  - `cash_survey_results.survey_channel` = `IVR Phone,Voice,SIERRA,DECAGON`
  - `cash_survey_results.survey_status` = `SUBMITTED`
  - `cash_survey_results.survey_created_week` = `6 weeks`
  - `cash_survey_results.is_non_customer` = `No`
  - `employee_cash_dim.bpo_name` = `Concentrix`
  - `employee_cash_dim.carto_region` = `Latin America,Asia Pacific`
  - `dim_report_period_pst.period_name` = `01 Day`
- **From results:** Find row matching target week:
  - CSAT = value × 100
  - Survey Count = value
  - Res Rate = value × 100

---

## Step 2: Reimbursement Data

**Source:** `https://blockcell.sqprod.co/sites/reimbursement-dashboard-concentrix/data.json`

- The individual transactions lack house labels. Use **byHouse monthly aggregate** for `Voice Money Movements`.
- **Pro-rate:** `monthly_usd * (week_txn_count / total_month_txn_count)`
- **High amount adjustments:** If monthly avg per txn < $100, write `0`.

---

## Step 3: Void Transaction Data

- **Sheet:** `1lfubQg0EURubK4ZL_DV2ifglGsS_1ZnAuB26tTjpCfU`
- **Tab:** `Unexpected Raw Data`
- Search for `Voice Money Movements` in column G.
- Check column B for target week date.
- **Sum** column L amounts.
- **Count** advocates (col H) with more than 5 voids that week.

---

## Step 4: Write All Data

Use `batch_update_cells` to write to the target column in **one call**.

- **Spreadsheet:** `1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s`
- **Sheet:** `Money Movements Voice`

---

## Step 5: Print Summary

Print summary with:
- Timestamp
- Target column
- All values written
- Any errors encountered
- Link to the spreadsheet

---

## Row Mapping (Target Column)

| Row | Metric | Example |
|-----|--------|---------|
| 3 | CSAT | 77.5% |
| 4 | Survey Count | 1161 |
| 5 | Res Rate | 75.3% |
| 6 | FCR | 66.9% |
| 7 | Complaints Flag Rate | 8.1% |
| 8 | ML Exception Rate | 19.6% |
| 10 | QA Standard | 96.2% |
| 11 | Customer Critical | 97.2% |
| 12 | Business Critical | 93.6% |
| 13 | Compliance Critical | 97.3% |
| 14 | QA Complaints | 84.7% |
| 15 | QA Ack Rate | 77.2% |
| 17 | AHT (minutes) | 8.92 |
| 18 | Hold Ratio | 0.50 |
| 19 | ACW (minutes) | 0.90 |
| 22 | Unresolved Cases | 1716 |
| 25 | Refund Amount | $121.74 |
| 26 | High Adj Count | 0 |
| 27 | Void Txn Amount | $9.99 |
| 28 | Void Outliers | 0 |
| 30 | SLA | 96.5% |

---

## Notes
- Rows 9, 16, 20, 21, 23, 24, 29 are spacer/header rows — do not write to them.
- All percentage values are stored as numbers (e.g., 77.5 not 0.775).
- AHT and ACW are in minutes.
- Dollar amounts include the `$` prefix when displayed but are stored as numbers.
