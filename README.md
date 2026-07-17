# apprenticeship-training-compliance-dashboard
An end-to-end Looker Studio dashboard built to monitor candidate lifecycle, trainer-led learning delivery, stipend billing, and client (employer) document compliance for a PAN-India apprenticeship and skilling organization. Designed to give leadership real-time visibility while giving regional and on-ground teams a tool to find and close their own compliance gaps.

> 👇 **Jump to the complete dashboard preview**  
> [Jump to Dashboard Preview ↓](#dashboard-preview)

![dashboard_hero.jpg](https://github.com/Harshvardan23/Compliance_Dashboard/blob/main/compliance%20Assets/dashboard_hero.jpg)

# 📌 Project Overview

This project presents an end-to-end Compliance & Performance Dashboard built using Looker Studio for a PAN-India organization that places candidates into employer sites for paid, hands-on apprenticeship training combined with structured learning.

The organization runs hundreds of employer (client) relationships, thousands of enrolled candidates, and a distributed trainer network — all generating data continuously through a Warehouse Management System (WMS). Because operational reality (what's happening on the ground) and system reality (what's formally recorded) rarely move at the same pace, the business needed a single source of truth that could:

. Show leadership where compliance is breaking down, region by region and client by client
. Give Regional Heads, supervisors, and MIS teams a way to drill into their own gaps and act on them
. Track candidates all the way from enrollment through document compliance, class attendance, and stipend billing

The solution transforms raw operational, candidate, and attendance data into actionable insights through structured KPIs, drill-downs, and calculated fields that encode real business rules — including several fields that had to reconcile data recorded at different grains (weekly ops data vs. per-client data vs. per-candidate data).

# 🧑‍💻 Technologies Used

. Warehouse Management System (WMS) – Source of operational, candidate, and attendance data
. Cloud Storage – Central, scalable raw data store
. PostgreSQL (pgAdmin) – Data storage, querying, and extraction
. Looker Studio – Dashboarding and visualization, connected directly to PostgreSQL (no intermediate validation layer)

# 🗂️ Data Schema

Unlike a single flattened table, this dashboard is built on **four core tables**, joined at different grains depending on the question being answered:

| Table | Grain | Key Fields |
|---|---|---|
| **Client table** | One row per client (employer) | Client ID, Client, Status, District, Revenue Model, Regional Head, Mode of Dealing, Client Onboarding Date, contract dates |
| **Candidate table** | One row per candidate | client_id, candidate_id, Product, Mode of Dealing, is_stipend_billed, Employee Code, Trainer, candidate_status, date_of_enrollment, date_of_joining |
| **Operations HC table (Weekly HC)** | One row per client, per week | client_id, Regional Head, Mode of Dealing, updated_hc, leave, target, openings, trainer_name, week_start_date |
| **Candidate Attendance table** | One row per candidate, per class, per week | client_id, candidate_name, mobile_number, trainer_name, product, week_start, attendance_status |

**Why four tables:** the business itself operates at different grains — a client is onboarded once, operations headcount is reported weekly, candidates are enrolled individually, and attendance is logged per class per week. Reconciling one-to-many, many-to-many, and many-to-one relationships across these grains was the central data-modeling challenge of this project (see Design Challenge below).

## Blend Structure

Four Looker Studio blends power the dashboard, each joining a different combination of these tables:

1. **Primary compliance blend** — Client + Weekly HC + Candidate, full outer joined on `Client ID`. Powers the Executive Summary and top-level Compliance Summary.
2. **Product/mode-level blend** — Candidate + Weekly HC, full outer joined on a **composite key**: `Client ID + Product + Mode of Dealing`. Needed because a single client can run multiple products under different modes of dealing, so a simple Client ID join would misattribute Ongoing HC against Active HC.
3. **Client-detail blend** — Client + Weekly + Candidate, full outer joined on `Client ID`. A leaner variant of the primary blend used for client-level document and detail views.
4. **Attendance blend** — Weekly Attendance + Candidate + Weekly HC, left outer joined on `client_id` (attendance table as the anchor). Powers TAL (Total Active Learners) reporting alongside HC context.

# 🔄 Data Flow

WMS → Cloud Storage → PostgreSQL (pgAdmin) → Looker Studio (direct connect)

## 🧠 How Data Moves

**WMS**
Candidate registration, operations headcount, and attendance data is captured continuously by on-ground and regional teams across PAN India.

**Cloud Storage**
Raw data is centrally stored to ensure scalability and availability.

**Database Layer (PostgreSQL / pgAdmin)**
Data is queried, validated, and structured into analysis-ready extracts across the four core tables.

**Visualization (Looker Studio)**
The four tables are blended at different grains and visualized using KPIs, filters, and calculated fields — connected directly to PostgreSQL with no intermediate validation layer.

# ❓ Business Questions Answered

### What are Active HC, Ongoing HC, Compliance %, and TAL for any Regional Head?

![executive_summary.png](assets/executive_summary.png)

## Design Challenge

The business tracks headcount two different ways, and they're never quite in sync. **Active HC** is what operations reports the moment a candidate is selected and starts work on-site. **Ongoing HC** is only counted once a candidate is fully registered in the WMS with complete documentation (Aadhar, PAN, bank details, etc.) — a process owned by the client-site supervisor or regional MIS team. Ground reality always moves faster than system registration, so the two numbers diverge — and that gap is exactly what this dashboard exists to surface and close.

## ⚙️ Metric Logic Approach

Active HC is calculated as a rolling weekly figure using a date-diff against the current date, rather than a static count:

```
Active HC (CW):
CASE
  WHEN DATETIME_DIFF(CURRENT_DATE(), week_start_date, WEEK) = 1
  THEN updated_hc + leave
  ELSE 0
END

Active HC (PW):
CASE
  WHEN DATETIME_DIFF(CURRENT_DATE(), week_start_date, WEEK) = 2
  THEN updated_hc + leave
  ELSE 0
END
```

Ongoing HC in its refined form accounts for candidates whose effective start date may come from any of three possible source fields, and recognizes more than one qualifying status:

```
DOJ_DOE_DOS (effective reference date, fallback chain):
IF(date_of_joining IS NOT NULL, date_of_joining,
  IF(date_of_enrollment IS NOT NULL, date_of_enrollment,
    IF(date_of_status IS NOT NULL, date_of_status, NULL)
  )
)

Week Start_DOJ_DOE_DOS (rounds the above to its week start):
DATETIME_SUB(DOJ_DOE_DOS, INTERVAL (EXTRACT(DAYOFWEEK FROM DOJ_DOE_DOS) - 2) DAY)

Ongoing HC:
CASE
  WHEN DATETIME_DIFF(CURRENT_DATE(), Week Start_DOJ_DOE_DOS, WEEK) >= 1
   AND candidate_status = 'Ongoing'
   OR candidate_status = 'Joined'
  THEN candidate_id
  ELSE NULL
END

Compliance %:
SUM(Ongoing HC) / SUM(Active HC (CW))
```

Because Compliance % is a ratio of two independently-sourced headcounts, it can legitimately exceed 100% when WMS registrations catch up on a backlog faster than new Active HC is added — this is expected and is itself a signal worth investigating, not a data error.

TAL (Total Active Learners) is calculated separately from the attendance table:

```
TAL (CW):
COUNT_DISTINCT(
  CASE
    WHEN DATETIME_DIFF(CURRENT_DATE(), week_start, WEEK) = 1
    THEN mobile_number
    ELSE NULL
  END
)
```

TAL is consistently the lowest number on the dashboard by design — candidates are blue-collar workers on live shifts, so leave, workload, shift timings, and client-side scheduling all reduce actual class attendance well below enrollment or headcount figures.

---

### What is the Revenue Model compliance for each Regional Head, and where is the Revenue Model undefined?

![revenue_model_compliance.png](assets/revenue_model_compliance.png)

## Design Challenge

Clients are grouped into three engagement types — **Inbound** (the organization owns salary, training, and everything else for the candidate), **Syndicate** (candidates are deployed and trained on machine use only, for a training fee), and **Vendor Managed** (the organization takes a service/training cut, without paying candidate salary). Each engagement type can independently run under a Pay & Collect or Collect & Pay revenue model — and a meaningful share of clients have no revenue model assigned at all (NULL), which this view exists to flag for resolution.

## ⚙️ Metric Logic Approach

Mode of Dealing and Revenue Model are tracked as two independent dimensions, cross-tabbed against Active Clients, Active HC, and Ongoing HC, so any Mode of Dealing × Revenue Model combination showing as NULL can be immediately routed back to the Regional Head for correction.

---

### For Inbound clients, has the candidate's stipend actually been billed through the system?

![stipend_billed_compliance.png](assets/stipend_billed_compliance.png)

## Design Challenge

Only **Inbound** clients require this check — since the organization pays the candidate's stipend directly under that engagement type, it must also be able to prove that stipend has been invoiced onward in the system. A candidate showing "stipend not billed" on an Inbound engagement is a direct financial and compliance risk.

## ⚙️ Metric Logic Approach

```
CASE
  WHEN is_stipend_billed = 1 THEN 'stipend billed through us'
  WHEN is_stipend_billed = 0 THEN 'stipend is not billed through us'
  ELSE 'N/A'
END
```

---

### Which Ongoing candidates are missing an Employee Code, and who are they?

![employee_code_compliance.png](assets/employee_code_compliance.png)

Ongoing HC is grouped by Employee Code, with NULL surfaced as its own bucket — currently the largest single bucket on the dashboard — so Regional Heads can immediately pull the underlying candidate list and close the gap.

---

### For a given Trainer and Client, what is Active HC, Ongoing HC, and TAL (CW/PW)?

![trainer_candidate_compliance.png](assets/trainer_candidate_compliance.png)

Every client is assigned a trainer who delivers scheduled classes on-site. This view lets a Regional Head or the training team see, per trainer per client, whether the candidates assigned are actually attending — directly connecting trainer accountability to attendance outcomes.

---

### What is the year-over-year monthly client onboarding trend?

![yoy_onboarding_trend.png](assets/yoy_onboarding_trend.png)

## ⚙️ Metric Logic Approach

```
CY 2026 YTD:
CASE
  WHEN YEAR(client_onboarding_date) = 2026
   AND client_onboarding_date IS NOT NULL
   AND status = 'Approved'
  THEN Client ID
  ELSE NULL
END
```
The same pattern is reused for each calendar year, with a companion `Calendar year` field (CY 2016–CY 2026) used to bucket and label the trend chart. Client names are trimmed for chart legibility using:
```
REGEXP_EXTRACT(client_name, '^([^ ]+ [^ ]+)')
```

---

### For any Regional Head, which client MOUs are expiring next month?

![contract_renewal_tracker.png](assets/contract_renewal_tracker.png)

## ⚙️ Metric Logic Approach

```
Days to expire:
CASE
  WHEN MONTH(DATE(contract_end_date)) =
    CASE WHEN MONTH(CURRENT_DATE()) = 12 THEN 1 ELSE MONTH(CURRENT_DATE()) + 1 END
   AND YEAR(DATE(contract_end_date)) =
    CASE WHEN MONTH(CURRENT_DATE()) = 12 THEN YEAR(CURRENT_DATE()) + 1 ELSE YEAR(CURRENT_DATE()) END
  THEN 'Expiring Next Month'
  ELSE 'Not Next Month'
END
```
The December → January year rollover is handled explicitly so contracts expiring across a calendar year boundary are still caught correctly.

---

### What is the document compliance status (MOU, WC, MC, GPA) for every active client?

![document_compliance.png](assets/document_compliance.png)

## Design Challenge

Every candidate deployment carries statutory and welfare obligations — Workmen's Compensation (WC) covers legal liability for workplace injury, Group Personal Accident (GPA) provides accident/death coverage, and MC and MOU track additional agreement and medical documentation. Missing or expired documents are a direct legal exposure if anything happens to a candidate on-site, which is why this is tracked at the same level of rigor as financial compliance.

## ⚙️ Metric Logic Approach

The same reusable pattern is applied across all four document types, swapped only on the relevant start/end date fields:

```
MOU:
CASE
  WHEN (contract_start_date IS NULL OR contract_end_date IS NULL) AND status = 'Approved'
  THEN 'NA'
  WHEN CURRENT_DATE() <= contract_end_date AND status = 'Approved'
  THEN 'Valid'
  WHEN CURRENT_DATE() > contract_end_date AND status = 'Approved'
  THEN 'Expired'
  ELSE status
END
```
The identical structure is reused for **WC** (`wc_start_date` / `wc_end_date`), **MC** (`mc_start_date` / `mc_end_date`), and **GPA** (`gpa_start_date` / `gpa_end_date`) — keeping the compliance logic consistent and auditable across every document type. A companion `Valid MOU` flag isolates just the Valid case for the agreement-wise client distribution chart.

---

### For every Ongoing candidate, has the admission letter and offer letter actually been generated?

![admission_offer_letter_compliance.png](assets/admission_offer_letter_compliance.png)

Every Ongoing candidate is supposed to have both an admission letter and an offer letter issued — in practice this frequently doesn't happen. This view surfaces exactly which candidates are missing one or both documents, so the gap can be pushed back to the responsible team and closed.

---

## Dashboard-Preview

![dashboard_full_preview.jpg](assets/dashboard_full_preview.jpg)

---

# 👨‍💻 Author
@https://github.com/Harshvardan23
