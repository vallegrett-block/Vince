# Money Movements Messaging Weekly Scorecard - Automation Recipe

## Purpose
This recipe automates the weekly refresh of the **Money Movements Messaging Weekly** scorecard on Google Sheets. It pulls data from Looker Explores, the Reimbursement Dashboard, Snowflake, and the Void Transactions Sheet, then writes values and applies RAG (Red/Amber/Green) formatting.

**Target Spreadsheet:** [Money Movements Messaging Weekly](https://docs.google.com/spreadsheets/d/1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s/edit?gid=2002305892#gid=2002305892)

---

## Step 0: Identify the Current Week Column

1. Read Row 1 (G1:Z1 and beyond) to get date headers (MM/DD/YYYY format, Monday week-starts)
2. Determine the most recent completed week's Monday date (if today is Monday, use the prior week)
3. Find which column corresponds to that week
4. If the week's date doesn't exist yet, add it as a new header in the next empty column
5. Only write data for the most recent completed week ‚Äî never overwrite previously written weeks

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
| 31 | Refund Amount | Reimbursement Dashboard + Snowflake |
| 32 | # of High Amount Adjustments $100+ | Reimbursement Dashboard + Snowflake |
| 33 | Void Transaction Amount | Void Transactions Sheet |
| 34 | Void Transaction Outliers >5 per week | Void Transactions Sheet |
| 35 | SLA | Looker Explore |

**Rows NOT populated (skip):** 2, 5, 9, 16, 17, 22, 24, 26, 27, 28, 29, 30

---

## Step 1A: CSAT & Resolution Rate (Rows 3, 4)

**Model:** cash_support_pst  
**View:** cash_survey_results

**Fields:**
- cash_survey_results.in_period_parameter__survey_created
- cash_survey_results.CSAT_survey_count_by_survey_hash_key
- cash_survey_results.CSAT_greater_than_3_by_survey_hash_key
- cash_survey_results.resolution_rate

**Filters:**
| Filter | Value |
|--------|-------|
| cash_survey_results.date_granularity | Week |
| cash_survey_results.pop_date_pt | 1 year |
| cash_survey_results.pop_timeframes | Current Period |
| cash_survey_results.survey_channel | Messaging |
| cash_survey_results.survey_status | SUBMITTED |
| cash_survey_results.house | Messaging Money Movements |
| cash_survey_results.source | In-App Messaging |
| cash_survey_results.is_non_customer | No |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.carto_region | Asia Pacific |
| employee_cash_dim.channel | Messaging |
| dim_report_period_pst.period_name | 01 Day |
| cash_support_cases.is_overnight_worker_initial_touch | No |
| cash_support_cases.is_messaging_idle_timeout | No |

**Sort:** cash_survey_results.in_period_parameter__survey_created

**Output:**
- **Row 3 (CSAT):** CSAT_greater_than_3 √ó 100 ‚Üí format as "62.8%"
- **Row 4 (Res Rate):** resolution_rate √ó 100 ‚Üí format as "79.7%"

---

## Step 1B: FCR (Row 6)

**Model:** cash_support_pst  
**View:** cash_support_cases

**Fields:**
- cash_support_cases.case_creation_week
- dim_first_contact_resolution.fcr_7_day_percent

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.case_creation_week | 8 weeks |
| employee_cash_dim.house | Messaging Money Movements |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.channel | Messaging |
| cash_support_cases.is_overnight_worker_initial_touch | No |
| cash_support_cases.is_messaging_idle_timeout | No |
| cash_survey_results.is_non_customer | No |

**Sort:** cash_support_cases.case_creation_week

**Output:**
- **Row 6 (FCR):** fcr_7_day_percent √ó 100 ‚Üí format as "91.0%"

---

## Step 1C: Complaints (Rows 7, 8)

**Model:** cash_support_pst  
**View:** cash_support_cases

### Complaint Flag Rate (Row 7)

**Fields:** cash_support_cases.case_creation_week, cash_support_cases.complaint_flagging_rate

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.case_creation_week | 8 weeks |
| cash_support_cases.house | Messaging Money Movements |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.channel | Messaging |

**Output:** complaint_flagging_rate √ó 100 ‚Üí format as "17.3%"

### ML Exception Rate (Row 8)

**Fields:** cash_support_cases.case_creation_week, cash_support_cases.complaint_exception_rate

Same filters as Row 7.

**Output:** complaint_exception_rate √ó 100 ‚Üí format as "6.8%"

---

## Step 1D: QA Standard (Row 10)

**Model:** cash_support_pst  
**View:** cash_support_cases

**Fields:**
- cash_support_cases.in_period_parameter__case_creation
- qa_scores_by_eval_view.avg_qa_eval_score

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.pop_timeframes | Current Period |
| cash_support_cases.pop_date_pt | 6 months |
| cash_support_cases.date_granularity | Week |
| cash_support_cases.blueprint_house | Messaging Money Movements |
| qa_scores_by_eval_view.qa_eval_channel | Messaging,Chat |
| qa_scores_by_question_view.qa_eval_cf1_name | 2025 CS Messaging - Evaluation Form,2026 CS Messaging - Evaluation Form |
| dim_report_period_pst.period_name | 01 Day |
| evaluated_advocate_employee_cash_dim.division | Concentrix,Sutherland |
| evaluated_advocate_employee_cash_dim.channel | Messaging |
| sdse_messaging_touches.is_overnight_worker | No,Yes |
| employee_cash_dim.carto_region | Asia Pacific |

**Output:** avg_qa_eval_score ‚Üí format as "98.80%"

---

## Step 1E: Customer/Business/Compliance Critical (Rows 11, 12, 13)

**Model:** cash_support_pst  
**View:** cash_support_cases

**Fields:**
- cash_support_cases.in_period_parameter__case_creation
- qa_scores_by_eval_view.avg_customer_section_score
- qa_scores_by_eval_view.avg_business_section_score
- qa_scores_by_eval_view.avg_compliance_section_score

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.pop_timeframes | Current Period |
| cash_support_cases.pop_date_pt | 6 months |
| cash_support_cases.date_granularity | Week |
| cash_support_cases.blueprint_house | Messaging Money Movements |
| qa_scores_by_question_view.qa_eval_cf1_name | 2025 CS Messaging - Evaluation Form,2026 CS Messaging - Evaluation Form |
| dim_report_period_pst.period_name | 01 Day |
| evaluated_advocate_employee_cash_dim.division | Concentrix,Sutherland |
| evaluated_advocate_employee_cash_dim.channel | Messaging |
| sdse_messaging_touches.is_overnight_worker | No,Yes |
| employee_cash_dim.carto_region | Asia Pacific |

**Output:**
- **Row 11:** avg_customer_section_score ‚Üí format as "99.10%"
- **Row 12:** avg_business_section_score ‚Üí format as "96.65%"
- **Row 13:** avg_compliance_section_score ‚Üí format as "99.58%"

---

## Step 1F: QA Complaints (Row 14)

**Model:** cash_support_pst  
**View:** cash_support_cases

**Fields:**
- cash_support_cases.in_period_parameter__case_creation
- qa_scores_by_question_view.rubric_question_result
- qa_scores_by_question_view.evaluation_count

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.pop_timeframes | Current Period |
| cash_support_cases.pop_date_pt | 6 months |
| cash_support_cases.date_granularity | Week |
| cash_support_cases.blueprint_house | Messaging Money Movements |
| evaluated_advocate_employee_cash_dim.carto_region | Asia Pacific |
| sdse_messaging_touches.is_overnight_worker | No |
| qa_scores_by_question_view.rubric_question_raw | Complaints Adherence,Q10: If complaint language was present%2C was the complaint properly identified and filed?,Q10: If complaint language was present%2C was the complaint properly identified and logged?,Q11: If complaint language was present%2C was the complaint properly identified and filed? |
| qa_scores_by_question_view.rubric_question_result | Yes,No,Pass,Fail |
| dim_report_period_pst.period_name | 01 Day |

**Calculation:** (Yes + Pass) / (Yes + Pass + No + Fail) √ó 100

**Note:** May return 0 rows for Money Movements. If so, leave Row 14 blank.

---

## Step 1G: QA Acknowledgement Rate (Row 15)

**Model:** cash_support_pst  
**View:** cash_support_cases

**Fields:**
- cash_support_cases.case_creation_week
- qa_scores_by_eval_view.evaluation_acknowledgement_status
- qa_scores_by_eval_view.count

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.case_creation_week | 8 weeks |
| cash_support_cases.house | Messaging Money Movements |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.channel | Messaging |

**Calculation:** ACKED / (ACKED + AWAITING_ACK + NOT_INITIATED) √ó 100

**Output:** format as "56.4%"

---

## Step 1H: AHT per Touch & ATA (Rows 18, 21)

**Model:** cash_support_pst  
**View:** sdse_messaging_touches

**Fields:**
- sdse_messaging_touches.touch_start_time_pst_week
- sdse_messaging_touches.average_messaging_touch_lifetime
- sdse_messaging_touches.average_messaging_assignment_to_action

**Filters:**
| Filter | Value |
|--------|-------|
| sdse_messaging_touches.touch_start_time_pst_week | 8 weeks |
| sdse_messaging_touches.house | Messaging Money Movements |
| sdse_messaging_touches.is_overnight_worker | No |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.channel | Messaging |

**Output:**
- **Row 18 (AHT):** average_messaging_touch_lifetime (minutes) ‚Üí format as "9.47"
- **Row 21 (ATA):** average_messaging_assignment_to_action (minutes) ‚Üí format as "0.83"

---

## Step 1I: Average Time on Case / ATC (Row 19)

**Model:** cash_support_pst  
**View:** cash_support_cases

**Fields:**
- cash_support_cases.case_creation_week
- cash_support_cases.average_active_case_lifetime_hours

**Filters:**
| Filter | Value |
|--------|-------|
| cash_support_cases.case_creation_week | 8 weeks |
| employee_cash_dim.house | Messaging Money Movements |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.channel | Messaging |

**Output:** average_active_case_lifetime_hours ‚Üí format as "5.13"

---

## Step 1J: Schedule Adherence (Row 20)

**Model:** cash_support_pst  
**View:** assembled_adherence

**Fields:**
- assembled_adherence.in_period_parameter
- assembled_adherence.schedule_adherence

**Filters:**
| Filter | Value |
|--------|-------|
| assembled_adherence.date_granularity | Week |
| assembled_adherence.pop_date | 6 months |
| assembled_adherence.pop_timeframes | Current Period |
| employee_cash_dim.house | Messaging Money Movements |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.carto_region | Asia Pacific |
| employee_cash_dim.channel | Messaging |

**Output:** schedule_adherence √ó 100 ‚Üí format as "98.1%"

---

## Step 1K: TBTQ>=45 (Row 23)

**Model:** cash_support_pst  
**View:** sdse_messaging_touches

**Fields:**
- sdse_messaging_touches.previous_touch_start_time_pst_week
- previous_employee_cash_dim.mgmt_parameter
- sdse_messaging_touches.total_touches_transfer_back_to_messaging_queue

**Filters:**
| Filter | Value |
|--------|-------|
| sdse_messaging_touches.previous_touch_start_time_pst_date | 6 months |
| employee_cash_dim.channel | Messaging |
| previous_employee_cash_dim.mgmt_granularity | Advocate |
| sdse_messaging_touches.previous_touch_advocate_id | -NULL |
| sdse_messaging_touches.total_touches_transfer_back_to_messaging_queue | >=45 |
| previous_employee_cash_dim.bpo_name | Concentrix |
| cash_support_cases.blueprint_house | Messaging Money Movements |

**Calculation:** Count distinct advocates (mgmt_parameter values) per week. Weeks with no rows = 0.

**Output:** format as integer "1"

---

## Step 1L: Transfer Accuracy (Row 25)

**Model:** cash_support_pst  
**View:** chat_case_queue_transfer

**Fields:**
- chat_case_queue_transfer.date_pt_parameter
- chat_case_queue_transfer.transfer_accuracy

**Filters:**
| Filter | Value |
|--------|-------|
| chat_case_queue_transfer.date_granularity | Week |
| chat_case_queue_transfer.queue_start_time_pst_date | 6 months |
| chat_case_queue_transfer.house | Messaging Money Movements |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.carto_region | Asia Pacific |
| cash_support_cases.is_overnight_worker_initial_touch | No |

**‚ö†Ô∏è NOTE:** Do NOT include employee_cash_dim.channel filter (must be empty/absent).

**Output:** transfer_accuracy √ó 100 ‚Üí format as "98.5%"

---

## Step 1M: SLA (Row 35)

**Model:** cash_support_pst  
**View:** sdse_messaging_touches

**Fields:**
- sdse_messaging_touches.in_period_parameter__touch_start_time_pst_date
- sdse_messaging_touches.percent_messaging_touch_in_sla

**Filters:**
| Filter | Value |
|--------|-------|
| sdse_messaging_touches.date_granularity | Week |
| sdse_messaging_touches.pop_date_pt | 6 months |
| sdse_messaging_touches.pop_timeframes | Current Period |
| sdse_messaging_touches.house | Messaging Money Movements |
| sdse_messaging_touches.is_agent_work_requested_in_business_hours | Yes |
| sdse_messaging_touches.is_overnight_worker | No |
| employee_cash_dim.bpo_name | Concentrix |
| employee_cash_dim.carto_region | Asia Pacific |
| employee_cash_dim.channel | Messaging |

**Output:** percent_messaging_touch_in_sla √ó 100 ‚Üí format as "92.0%"

---

## Step 2: Reimbursement Data (Rows 31, 32)

**Source:** https://blockcell.sqprod.co/sites/reimbursement-dashboard-concentrix/data.json

1. Get individual transaction case IDs (index 10 in each transaction array) where channel = Messaging (index 4 = 1)
2. Query Snowflake to get house for each case:

```sql
SELECT c.ID as case_id, e.HOUSE
FROM SUPPORT_DE.CRM.CASH_SALESFORCE_CASE c
JOIN SUPPORT_DE.UNIFIED.EMPLOYEES e
  ON c.OWNER_ID = e.SALESFORCE_ID
  AND c.CREATED_DATE_UTC BETWEEN e.VALID_FROM_UTC AND COALESCE(e.VALID_TO_UTC, '2099-12-31')
WHERE c.ID IN (<case_ids>)
  AND e.HOUSE = 'Messaging Money Movements'
```

3. Filter transactions to only those with house = "Messaging Money Movements" AND within target week

**Output:**
- **Row 31 (Refund Amount):** Sum of amounts ‚Üí format as "$69.97"
- **Row 32 (# High Adjustments $100+):** Count where amount > 100 ‚Üí format as integer "0"

---

## Step 3: Void Transaction Data (Rows 33, 34)

**Source:** Google Sheet `1lfubQg0EURubK4ZL_DV2ifglGsS_1ZnAuB26tTjpCfU`  
**Tab:** "Unexpected Raw Data"

**Filter:** week = target week AND house = "Messaging Money Movements" AND bpo = "Concentrix"

**Output:**
- **Row 33:** Sum of Total Negative Dollars ‚Üí format as "$0.00"
- **Row 34:** Count unique advocates with MORE THAN 5 voids that week ‚Üí format as integer

---

## Step 4: Write Data

**Spreadsheet ID:** 1K1KjZcmHhUCF_Lgfd7sn8MiCijE0dXVICljI5b8Ug1s  
**Sheet:** Money Movements Messaging Weekly

Write only the most recent completed week. Do NOT overwrite previously written weeks.

---

## Step 5: Apply RAG (Red/Amber/Green) Formatting

Read Column D dynamically to get current targets.

### Direction Rules
- **Higher is better:** Rows 3, 4, 7, 10, 11, 12, 13, 14, 15, 20, 25, 35
- **Lower is better:** Rows 18, 19, 21, 23, 32, 34

### Color Logic
| Color | RGB Values | Condition (Higher is Better) | Condition (Lower is Better) |
|-------|-----------|------------------------------|----------------------------|
| GREEN | rgb(0.714, 0.839, 0.659) | value >= target | value <= target |
| YELLOW | rgb(1.0, 0.851, 0.4) | value >= (target - 3) AND value < target | value <= (target + 3) AND value > target |
| RED | rgb(0.918, 0.6, 0.6) | value < (target - 3) | value > (target + 3) |
| WHITE | ‚Äî | No data or no numeric target | No data or no numeric target |

### Strict Zero Rule
When target = 0 (e.g., Rows 23, 32, 34):
- **ONLY exact 0 = GREEN**
- **Any value > 0 = RED** (no yellow)

Apply to ALL data columns (not just the new week).

---

## Step 6: Confirm

Print summary with:
- Timestamp
- Column written
- All values
- RAG status
- Any errors
- Spreadsheet link

---

*Recipe documented: May 27, 2026*
