# Cash Cards Voice Weekly Scorecard - Automation Recipe

## Purpose
This recipe automates the weekly refresh of the **Cash Cards Voice** scorecard on Google Sheets. It pulls data from Looker Dashboard 40389, a direct Looker Explore query (for CSAT), the Reimbursement Dashboard, and the Void Transactions Sheet, then writes values to the correct week column.

**Target Spreadsheet:** [Cash Cards Voice](https://docs.google.com/spreadsheets/d/1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s/edit?gid=1218305668#gid=1218305668)

---

## Step 0: Identify the Current Week Column

1. Read Row 1 (G1:Z1 and beyond) to get date headers (MM/DD/YYYY format, Monday week-starts)
2. Determine the most recent completed week's Monday date (if today is Monday, use the prior week)
3. Find which column corresponds to that week
4. If the week's date doesn't exist yet, add it as a new header in the next empty column
5. Only write data for the most recent completed week — never overwrite previously written weeks

---

## Row Mapping

| Row | KPI | Source |
|-----|-----|--------|
| 3 | CSAT | Looker Explore Query |
| 4 | Survey Count | Looker Explore Query |
| 5 | Res Rate | Looker Explore Query |
| 6 | FCR | Looker Dashboard 40389 |
| 7 | Complaints Flag Rate | Looker Dashboard 40389 |
| 8 | Complaints ML Exception Rate | Looker Dashboard 40389 |
| 10 | QA (Standard) | Looker Dashboard 40389 |
| 11 | Customer Critical | Looker Dashboard 40389 |
| 12 | Business Critical | Looker Dashboard 40389 |
| 13 | Compliance Critical | Looker Dashboard 40389 |
| 14 | QA (Complaints) | Looker Dashboard 40389 |
| 15 | QA + Complaints Acknowledgement Rate | Looker Dashboard 40389 |
| 17 | AHT | Looker Dashboard 40389 |
| 18 | Hold | Looker Dashboard 40389 |
| 19 | ACW | Looker Dashboard 40389 |
| 22 | Unresolved Cases Over 24 hours | Looker Dashboard 40389 |
| 25 | Refund Amount | Reimbursement Dashboard |
| 26 | # of High Amount Adjustments $100+ | Reimbursement Dashboard |
| 27 | Void Transaction Amount | Void Transactions Sheet |
| 28 | Void Transaction Outliers >5 per week | Void Transactions Sheet |
| 30 | SLA | Looker Dashboard 40389 |

**Rows NOT populated (skip):** 2, 9, 16, 20, 21, 23, 24, 29, 31

---

## Step 1A: CSAT, Survey Count & Resolution Rate (Rows 3, 4, 5)

### ⚠️ IMPORTANT
Do **NOT** use the CSAT tile from Dashboard 40389. That tile has hardcoded tile-level filter overrides using the WRONG house ("Voice General"), WRONG carto region ("Latin America" only), and WRONG survey channel filter. This produces incorrect/low CSAT numbers.

Instead, run a **direct explore query** using `run_explore_query`.

**Model:** cash_support_pst  
**View:** cash_survey_results

**Fields:**
- cash_survey_results.survey_created_week
- cash_survey_results.CSAT_greater_than_3_by_survey_hash_key
- cash_survey_results.CSAT_survey_count_by_survey_hash_key
- cash_survey_results.resolution_rate

**Filters:**
| Filter | Value |
|--------|-------|
| cash_survey_results.house | Voice Cash Card Only |
| cash_survey_results.survey_channel | IVR Phone,Voice,SIERRA,DECAGON |
| cash_survey_results.survey_status | SUBMITTED |
| cash_survey_results.survey_created_week | 6 weeks |
| cash_survey_results.is_non_customer | No |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.carto_region | Latin America,Asia Pacific |
| dim_report_period_pst.period_name | 01 Day |

**Sort:** cash_survey_results.survey_created_week

**Output:**
- **Row 3 (CSAT):** CSAT_greater_than_3 × 100 → format as "84.8%"
- **Row 4 (Survey Count):** CSAT_survey_count → format as integer "1456"
- **Row 5 (Res Rate):** resolution_rate × 100 → format as "89.2%"

---

## Step 1B: Remaining Metrics from Looker Dashboard 40389

Run Dashboard 40389 with these **dashboard-level filters:**

| Filter | Value |
|--------|-------|
| Period Timeframe Identifier | Current Period |
| Date Granularity | Week |
| Current Period Date Range | 1 year |
| House | Voice Cash Card Only |
| Bpo Name | Concentrix |
| Country | -United States of America |
| Carto Region | Latin America,Asia Pacific |
| Survey Channel | IVR Phone,Voice,SIERRA,DECAGON |
| City | *(empty)* |
| Initiation Method | *(empty)* |
| Ldap Today | *(empty)* |
| Manager | *(empty)* |
| Managers Manager | *(empty)* |
| New Hire Tenure Cohort Days | *(empty)* |
| Period | Weekly |

---

### FCR % Tile → Row 6

- Format as percentage with 1 decimal (e.g., "91.2%")
- ⚠️ This tile frequently times out. If it does, leave Row 6 blank.

---

### Complaint Flag Rate Tile → Row 7

- Format as percentage with 1 decimal (e.g., "7.4%")

---

### ML Exception Rate Tile → Row 8

- Use `complaint_exception_rate` field
- Format as percentage with 1 decimal (e.g., "19.2%")

---

### QA Section Score Tile → Rows 10, 11, 12, 13

- ⚠️ This tile returns **DAILY** data. Average all days within the target week (Mon–Sun).
- **Row 10 (QA Standard):** avg_qa_eval_score averaged for the week → format as "97.5%"
- **Row 11 (Customer Critical):** avg_customer_section_score averaged for the week → format as "98.2%"
- **Row 12 (Business Critical):** avg_business_section_score averaged for the week → format as "95.8%"
- **Row 13 (Compliance Critical):** avg_compliance_section_score averaged for the week → format as "99.1%"

---

### Complaints QA Tile → Row 14

- Calculate: Yes / (Yes + No) for the target week
- Format as "88.4%"

---

### QA Acknowledgement Tile → Row 15

- Calculate: ACKED / (ACKED + AWAITING_ACK + NOT_INITIATED) for the target week
- Format as "70.5%"

---

### Handle Time Tile → Rows 17, 19

- ⚠️ Returns **MONTHLY** data. Use the month containing the target week.
- **Row 17 (AHT):** avg_handle_time (in days) × 24 × 60 = minutes → format as "8.96"
- **Row 19 (ACW):** average_wrap_time_hours (in days) × 24 × 60 = minutes → format as "0.91"

---

### Holds Per Call Tile → Row 18

- ⚠️ Returns **MONTHLY** data. Use the month containing the target week.
- Use `hold_per_call` table calculation
- Format as "0.51"

---

### Unresolved Cases Tile → Row 22

- Snapshot count (point-in-time value)
- Write the current value as integer (e.g., "1893")

---

### SLA Tile → Row 30

- ⚠️ Returns **MONTHLY** data. Use the month containing the target week.
- service_level_percent × 100
- Format as "94.7%"

---

## Step 2: Reimbursement Data (Rows 25, 26)

**Source:** `https://blockcell.sqprod.co/sites/reimbursement-dashboard-concentrix/data.json`

### Logic:

1. Check if `txnHouses` array exists in the JSON response
2. **If YES (house-level transaction data available):**
   - Find the index for "Voice Cash Card Only" in `txnHouses`
   - Filter `individualTransactions` where index [11] equals that house index
   - Sum amounts for the target week
3. **If NO (txnHouses missing):**
   - Use the `byHouse` monthly aggregate
   - Find entry where house = "Voice Cash Card Only" for the target month
   - Pro-rate: `monthly_usd × (7 / days_in_month_with_data)`
   - ⚠️ Note this as approximate in summary

**Output:**
- **Row 25 (Refund Amount):** Sum of amounts for Voice Cash Card Only in target week → format as "$1,033.97"
- **Row 26 (# High Adjustments $100+):** Count where amount > 100 → format as integer "3"

---

## Step 3: Void Transaction Data (Rows 27, 28)

**Source:** Google Sheet `1lfubQg0EURubK4ZL_DV2ifglGsS_1ZnAuB26tTjpCfU`  
**Tab:** "Unexpected Raw Data"

**Columns:**
- B: Void Occurred Week (YYYY-MM-DD, Monday)
- G: House (filter for "Voice Cash Card Only")
- H: Advocate LDAP
- L: Total Negative Dollars

**Filter:** Column B = target week AND Column G = "Voice Cash Card Only"

**Output:**
- **Row 27 (Void Transaction Amount):** Sum of Column L → format as "$0.00" if none
- **Row 28 (Void Transaction Outliers):** Count unique advocates (Col H) with MORE THAN 5 voids that week → format as integer

---

## Step 4: Write Data

**Spreadsheet ID:** `1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s`  
**Sheet:** Cash Cards Voice

Write only the most recent completed week. Do NOT overwrite previously written weeks.

---

## Step 5: Confirm

Print summary with:
- Timestamp
- Column written
- All values written
- Any errors or limitations encountered
- Spreadsheet link

---

*Recipe documented: May 27, 2026*
