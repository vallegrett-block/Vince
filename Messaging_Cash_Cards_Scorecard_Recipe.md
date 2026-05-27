# Messaging Cash Cards Scorecard

## Overview

**Recipe File:** `messaging-cash-cards-scorecard.yaml`  
**Schedule:** Every Monday at 2:00 PM CT (`0 14 * * 1`)  
**Timezone:** America/Chicago

This automated recipe refreshes the weekly Messaging Cash Cards scorecard by pulling KPI data from multiple sources and writing it to the [Cash Cards Messaging Weekly Google Sheet](https://docs.google.com/spreadsheets/d/1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s/edit?gid=2002305892#gid=2002305892).

---

## Data Sources

| Source | Description |
|--------|-------------|
| **Looker Explores** | `cash_support_pst` and `cash_support` models — CSAT, Resolution Rate, FCR, Complaints, QA, AHT, ATC, Adherence, ATA, TBTQ, Transfer Accuracy, SLA |
| **Reimbursement Dashboard** | `https://blockcell.sqprod.co/sites/reimbursement-dashboard-concentrix/data.json` — Refund amounts and high-dollar adjustments |
| **Void Transactions Sheet** | Google Sheet `1lfubQg0EURubK4ZL_DV2ifglGsS_1ZnAuB26tTjpCfU` (tab: "Unexpected Raw Data") — Void transaction amounts and outliers |

---

## Target Google Sheet

- **Spreadsheet ID:** `1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s`
- **Tab:** `Cash Cards Messaging Weekly`
- **Data starts:** Column G
- **Row 1:** Weekly date headers (MM/DD/YYYY format, Monday week-starts)

---

## Row Mapping

| Row | KPI | Source |
|-----|-----|--------|
| 3 | Advocate CSAT | Looker Explore |
| 4 | Res Rate | Looker Explore |
| 6 | FCR | Looker Explore |
| 7 | Complaints ID Rate | Looker Explore |
| 8 | TL Complaints Review (ML Exception) | Looker Explore |
| 10 | QA (Standard) | Looker Explore |
| 11 | Customer Critical | Looker Explore |
| 12 | Business Critical | Looker Explore |
| 13 | Compliance Critical | Looker Explore |
| 14 | QA (Complaints) | Looker Explore |
| 15 | QA Acknowledgement Rate | Looker Explore |
| 18 | AHT per Touch (min) | Looker Explore |
| 19 | Avg Time on Case - ATC (hrs) | Looker Explore |
| 20 | Adherence | Looker Explore |
| 21 | ATA (min) | Looker Explore |
| 23 | TBTQ>=45 | Looker Explore |
| 25 | Transfer Accuracy | Looker Explore |
| 31 | Refund Amount | Reimbursement Dashboard |
| 32 | # of High Amount Adjustments $100+ | Reimbursement Dashboard |
| 33 | Void Transaction Amount | Void Transactions Sheet |
| 34 | Void Transaction Outliers >5 per week | Void Transactions Sheet |
| 35 | SLA | Looker Explore |

**Rows NOT populated (skipped):** 2, 5, 9, 16, 17, 22, 24, 26, 27, 28, 29, 30

---

## Steps

### Step 1: Identify the Current Week Column

1. Read row 1 (G1 onward) to get date headers
2. Determine the most recent completed week's Monday date (if today is Monday, use the prior week)
3. Find which column corresponds to that week
4. If the week's date doesn't exist yet, add it as a new header in the next empty column
5. Only write data for the most recent completed week

---

### Step 1A: CSAT & Resolution Rate (Rows 3, 4)

- **Model:** `cash_support_pst`
- **View:** `cash_survey_results`
- **Fields:**
  - `cash_survey_results.survey_created_week`
  - `cash_survey_results.CSAT_greater_than_3_by_survey_hash_key`
  - `cash_survey_results.resolution_rate`
- **Filters:**
  - `cash_survey_results.survey_created_week`: 8 weeks
  - `cash_survey_results.house`: Messaging Cash Cards
  - `cash_survey_results.survey_channel`: Messaging
  - `cash_survey_results.survey_status`: SUBMITTED
  - `cash_survey_results.is_non_customer`: No
  - `cash_survey_results.source`: In-App Messaging
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
  - `cash_support_cases.is_overnight_worker_initial_touch`: No
  - `cash_support_cases.is_messaging_idle_timeout`: No
- **Output:**
  - Row 3 (CSAT): `CSAT_greater_than_3 × 100` → format as `"62.1%"`
  - Row 4 (Res Rate): `resolution_rate × 100` → format as `"77.2%"`

---

### Step 1B: FCR (Row 6)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Fields:**
  - `cash_support_cases.case_creation_week`
  - `dim_first_contact_resolution.fcr_7_day_percent`
- **Filters:**
  - `cash_support_cases.case_creation_week`: 8 weeks
  - `employee_cash_dim.house`: Messaging Cash Cards
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
  - `cash_support_cases.is_overnight_worker_initial_touch`: No
  - `cash_support_cases.is_messaging_idle_timeout`: No
  - `cash_survey_results.is_non_customer`: No
- **Output:**
  - Row 6 (FCR): `fcr_7_day_percent × 100` → format as `"89.6%"`

---

### Step 1C: Complaints (Rows 7, 8)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Filters:**
  - `cash_support_cases.case_creation_week`: 8 weeks
  - `cash_support_cases.house`: Messaging Cash Cards
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging

**Row 7 — Complaint Flag Rate:**
- Field: `cash_support_cases.complaint_flagging_rate`
- Output: `complaint_flagging_rate × 100` → format as `"20.3%"`

**Row 8 — ML Exception Rate:**
- Field: `cash_support_cases.complaint_exception_rate`
- Output: `complaint_exception_rate × 100` → format as `"3.5%"`

---

### Step 1D: QA Standard (Row 10)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Fields:**
  - `cash_support_cases.in_period_parameter__case_creation`
  - `qa_scores_by_eval_view.avg_qa_eval_score`
- **Filters:**
  - `cash_support_cases.pop_timeframes`: Current Period
  - `cash_support_cases.pop_date_pt`: 6 months
  - `cash_support_cases.date_granularity`: Week
  - `cash_support_cases.blueprint_house`: Messaging Cash Cards
  - `qa_scores_by_eval_view.qa_eval_channel`: Messaging,Chat
  - `qa_scores_by_question_view.qa_eval_cf1_name`: 2025 CS Messaging - Evaluation Form,2026 CS Messaging - Evaluation Form
  - `dim_report_period_pst.period_name`: 01 Day
  - `evaluated_advocate_employee_cash_dim.division`: Concentrix,Sutherland
  - `evaluated_advocate_employee_cash_dim.channel`: Messaging
  - `sdse_messaging_touches.is_overnight_worker`: No,Yes
  - `employee_cash_dim.carto_region`: Asia Pacific
- **Output:**
  - Row 10: `avg_qa_eval_score` → format as `"98.07%"`

---

### Step 1E: Customer / Business / Compliance Critical (Rows 11, 12, 13)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Fields:**
  - `cash_support_cases.in_period_parameter__case_creation`
  - `qa_scores_by_eval_view.avg_customer_section_score`
  - `qa_scores_by_eval_view.avg_business_section_score`
  - `qa_scores_by_eval_view.avg_compliance_section_score`
- **Filters:** Same as Step 1D (minus `qa_scores_by_eval_view.qa_eval_channel`)
- **Output:**
  - Row 11: `avg_customer_section_score` → format as `"98.88%"`
  - Row 12: `avg_business_section_score` → format as `"94.04%"`
  - Row 13: `avg_compliance_section_score` → format as `"98.95%"`

---

### Step 1F: QA Complaints (Row 14)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Fields:**
  - `cash_support_cases.in_period_parameter__case_creation`
  - `qa_scores_by_question_view.rubric_question_result`
  - `qa_scores_by_question_view.evaluation_count`
- **Filters:**
  - `cash_support_cases.pop_timeframes`: Current Period
  - `cash_support_cases.pop_date_pt`: 6 months
  - `cash_support_cases.date_granularity`: Week
  - `qa_scores_by_question_view.rubric_question_raw`: Complaints Adherence, Q10/Q11 variants
  - `qa_scores_by_question_view.rubric_question_result`: Yes,No,Pass,Fail
  - `dim_report_period_pst.period_name`: 01 Day
  - `evaluated_advocate_employee_cash_dim.channel`: Chat,Messaging
- **Calculation:** `(Yes + Pass) / (Yes + Pass + No + Fail) × 100`
- **Output:** Row 14 → format as `"90.66%"`

---

### Step 1G: QA Acknowledgement Rate (Row 15)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Fields:**
  - `cash_support_cases.case_creation_week`
  - `qa_scores_by_eval_view.evaluation_acknowledgement_status`
  - `qa_scores_by_eval_view.count`
- **Filters:**
  - `cash_support_cases.case_creation_week`: 8 weeks
  - `cash_support_cases.house`: Messaging Cash Cards
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
- **Calculation:** `ACKED / (ACKED + AWAITING_ACK + NOT_INITIATED) × 100`
- **Output:** Row 15 → format as `"53.4%"`

---

### Step 1H: AHT per Touch & ATA (Rows 18, 21)

- **Model:** `cash_support_pst`
- **View:** `sdse_messaging_touches`
- **Fields:**
  - `sdse_messaging_touches.touch_start_time_pst_week`
  - `sdse_messaging_touches.average_messaging_touch_lifetime`
  - `sdse_messaging_touches.average_messaging_assignment_to_action`
- **Filters:**
  - `sdse_messaging_touches.touch_start_time_pst_week`: 8 weeks
  - `sdse_messaging_touches.house`: Messaging Cash Cards
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
- **Output:**
  - Row 18 (AHT): `average_messaging_touch_lifetime` in minutes → format as `"9.51"`
  - Row 21 (ATA): `average_messaging_assignment_to_action` in minutes → format as `"0.94"`

---

### Step 1I: Average Time on Case / ATC (Row 19)

- **Model:** `cash_support_pst`
- **View:** `cash_support_cases`
- **Fields:**
  - `cash_support_cases.case_creation_week`
  - `cash_support_cases.average_active_case_lifetime_hours`
- **Filters:**
  - `cash_support_cases.case_creation_week`: 8 weeks
  - `employee_cash_dim.house`: Messaging Cash Cards
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
- **Output:**
  - Row 19 (ATC): `average_active_case_lifetime_hours` → format as `"6.20"`

---

### Step 1J: Schedule Adherence (Row 20)

- **Model:** `cash_support`
- **View:** `adherence_attr_detail`
- **Fields:**
  - `adherence_attr_detail.date_pt_parameter`
  - `adherence_attr_detail.total_in_adherence_seconds`
  - `adherence_attr_detail.total_scheduled_seconds`
- **Filters:**
  - `adherence_attr_detail.date_granularity`: Week
  - `adherence_attr_detail.pop_timeframes`: Current Period
  - `adherence_attr_detail.pop_date_pt`: 1 year
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
- **Calculation:** `(total_in_adherence_seconds / total_scheduled_seconds) × 100`
- **Output:** Row 20 → format as `"72.1%"`

---

### Step 1K: TBTQ>=45 (Row 23)

- **Model:** `cash_support_pst`
- **View:** `sdse_messaging_touches`
- **Fields:**
  - `sdse_messaging_touches.previous_touch_start_time_pst_week`
  - `previous_employee_cash_dim.mgmt_parameter`
  - `sdse_messaging_touches.total_touches_transfer_back_to_messaging_queue`
- **Filters:**
  - `sdse_messaging_touches.previous_touch_start_time_pst_date`: 6 months
  - `employee_cash_dim.channel`: Messaging
  - `previous_employee_cash_dim.mgmt_granularity`: Advocate
  - `sdse_messaging_touches.previous_touch_advocate_id`: -NULL
  - `sdse_messaging_touches.total_touches_transfer_back_to_messaging_queue`: >=45
  - `previous_employee_cash_dim.bpo_name`: Concentrix
  - `cash_support_cases.blueprint_house`: Messaging Cash Cards
- **Calculation:** Count distinct advocates (mgmt_parameter values) per week. Weeks with no rows = 0.
- **Output:** Row 23 → format as integer `"1"`

---

### Step 1L: Transfer Accuracy (Row 25)

- **Model:** `cash_support_pst`
- **View:** `chat_case_queue_transfer`
- **Fields:**
  - `chat_case_queue_transfer.date_pt_parameter`
  - `chat_case_queue_transfer.transfer_accuracy`
- **Filters:**
  - `chat_case_queue_transfer.date_granularity`: Week
  - `chat_case_queue_transfer.pop_timeframes`: Current Period
  - `chat_case_queue_transfer.pop_date_pt`: 6 months
  - `cash_support_cases.is_overnight_worker_initial_touch`: No
- **Note:** No house or BPO filter (intentionally blank on the dashboard)
- **Output:** Row 25: `transfer_accuracy × 100` → format as `"96.4%"`

---

### Step 1M: SLA (Row 35)

- **Model:** `cash_support_pst`
- **View:** `sdse_messaging_touches`
- **Fields:**
  - `sdse_messaging_touches.touch_start_time_pst_week`
  - `sdse_messaging_touches.percent_messaging_touch_in_sla`
- **Filters:**
  - `sdse_messaging_touches.touch_start_time_pst_week`: 8 weeks
  - `sdse_messaging_touches.house`: Messaging Cash Cards
  - `employee_cash_dim.bpo_name`: Concentrix
  - `employee_cash_dim.channel`: Messaging
- **Output:** Row 35: `percent_messaging_touch_in_sla × 100` → format as `"88.2%"`

---

### Step 2: Reimbursement Data (Rows 31, 32)

- **Source:** `https://blockcell.sqprod.co/sites/reimbursement-dashboard-concentrix/data.json`
- **JSON Structure:** `individualTransactions` array, each transaction is an array:
  - Index 4 = channel (0=Email, 1=Messaging, 2=Voice)
  - Index 7 = amount in USD
  - Index 8 = date (YYYY-MM-DD)
- **Filter:** Index 4 = 1 (Messaging) AND Index 8 within target week (Monday–Sunday)
- **Output:**
  - Row 31 (Refund Amount): Sum of amounts → format as `"$4,272.43"`
  - Row 32 (# High Adjustments $100+): Count where amount > 100 → format as integer `"3"`

---

### Step 3: Void Transaction Data (Rows 33, 34)

- **Source:** Google Sheet `1lfubQg0EURubK4ZL_DV2ifglGsS_1ZnAuB26tTjpCfU`, tab "Unexpected Raw Data"
- **Columns:**
  - B: Void Occurred Week (YYYY-MM-DD)
  - D: BPO Name
  - G: House
  - H: Advocate LDAP
  - Last column: Total Negative Dollars (with $ sign)
- **Filter:** week = target week AND house = "Messaging Cash Cards" AND bpo = "Concentrix"
- **Output:**
  - Row 33: Sum of Total Negative Dollars → format as `"$73.68"` (or `"$0.00"` if none)
  - Row 34: Count unique advocates with MORE THAN 5 voids that week → format as integer

---

### Step 4: Write Data

Write only the target week column to the Google Sheet. Do NOT overwrite previously written weeks.

- **Spreadsheet ID:** `1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s`
- **Sheet:** `Cash Cards Messaging Weekly`

---

### Step 5: Apply RAG (Red/Amber/Green) Formatting

Read Column D (D3:D35) dynamically to get current targets.

**Direction rules:**
- **Higher is better:** Rows 3, 4, 7, 10, 11, 12, 13, 14, 15, 20, 25, 35
- **Lower is better:** Rows 18, 19, 21, 23, 32, 34

**Color logic:**

| Color | RGB | Condition |
|-------|-----|-----------|
| 🟢 GREEN | `(0.714, 0.839, 0.659)` | Meets or exceeds target |
| 🟡 YELLOW | `(1.0, 0.851, 0.4)` | Within 3 points/units of target |
| 🔴 RED | `(0.918, 0.6, 0.6)` | More than 3 away from target |
| ⬜ WHITE | — | No data or no numeric target |

**Strict Zero Rule:** When target = 0 (Rows 23, 32, 34):
- Exact 0 = GREEN
- Any value > 0 = RED (no yellow)

Formatting is applied to ALL data columns for consistency.

---

### Step 6: Confirmation

The recipe prints a summary with:
- Timestamp
- Column written
- All values written
- RAG status per row
- Any errors or data unavailable
- Link to spreadsheet

---

## Extensions Required

| Extension | Purpose |
|-----------|---------|
| `computercontroller` | Web scraping, scripting, automation |
| `looker` | Querying Looker explores for KPI data |
| `googledrive` | Reading/writing Google Sheets |

---

## Recipe YAML

The recipe file is located at: `~/.config/goose/recipes/messaging-cash-cards-scorecard.yaml`
