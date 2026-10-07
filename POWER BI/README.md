# Healthcare Performance & Patient Analytics Dashboard

**Power BI Final Project — IFEXA**

An interactive Power BI dashboard built for a healthcare organization operating across four Nigerian states, turning raw patient visit, revenue and operations data into insights management can act on.

---

## 1. Project Overview

This project analyzes one year (2025) of patient visit data across 4 states, 12 branches and 6 departments, covering 1,200 patients and 2,920 total visits. The goal was not just to build charts, but to answer a specific business question through data:

> **"How is our healthcare organization performing, what are our patients experiencing, and where are the major areas that require attention?"**

The final deliverable is a 5-page Power BI report:

| Page | Focus |
|---|---|
| Executive Overview | Top-level KPIs and organization-wide trends |
| Patient Analysis | Who the patients are and what they're treated for |
| Hospital Operations | How efficiently departments and branches run |
| Financial Performance | Revenue, cost, profit and target achievement |
| Patient Experience | Satisfaction and its relationship to other factors |

---

## 2. Business Problem

Management needed a single source of truth to answer five practical questions:

1. Are we growing, and where is our revenue coming from?
2. Who are our patients, and what are they being treated for?
3. Which departments and branches are operating efficiently — and which aren't?
4. Are we meeting our financial targets?
5. Are patients satisfied, and what actually drives that satisfaction?

Prior to this dashboard, this data existed only as flat spreadsheets with no way to slice it by state, branch, department or time, and no way to track performance against targets.

---

## 3. Dataset Description

Source: `IFEXA_HealthCare_BI_Project_Dataset.xlsx` — 3 sheets.

| Sheet | Rows | Description |
|---|---|---|
| **Patient_Visits** | 1,200 | Fact table. One row per patient: demographics, visit details, department/service, financials, waiting time, satisfaction, outcome. |
| **State_Targets** | 4 | Annual revenue target per state (₦45M–₦55M, ₦200M total). |
| **Date_Table** | 365 | Full calendar for 2025 with Month, Month Number, etc. |

**Key fields:** Patient_ID, Visit_Date, State, Branch, Department, Service, Age, Gender, Diagnosis, Payment_Method, Insurance_Type, Visit_Count, Revenue_NGN, Cost_NGN, Waiting_Time_Min, Satisfaction_Score, Outcome.

**Important structural note:** each `Patient_ID` appears exactly once — repeat visits are captured in the `Visit_Count` column (range 1–4, average 2.43), not by repeated rows. This means:
- **Total Patients** = `DISTINCTCOUNT(Patient_ID)` = 1,200
- **Total Visits** = `SUM(Visit_Count)` = 2,920
- **Returning patient** is defined in this project as `Visit_Count > 1` (no multi-year history exists to track patients returning across separate visits).

---

## 4. Tools Used

- **Power BI Desktop** — data modeling, DAX, report build
- **Power Query** — data loading and light transformation
- **DAX** — 20 measures across KPIs, time intelligence and targets
- **Python / pandas** — used during the analysis phase to validate data quality and pressure-test every insight before writing it into the dashboard

---

## 5. Data Cleaning Process

The dataset was inspected before any modeling began. Findings:

**Clean:**
- No missing values, no duplicate rows, no stray whitespace in any sheet
- No negative revenue, cost or waiting time values
- All satisfaction scores fall within the valid 1.4–5.0 range
- Cost never exceeds revenue on any row
- Every visit date exists in Date_Table; all states match State_Targets
- Category spellings (State, Branch, Department, Gender, Diagnosis, Outcome) are consistent throughout

**Data quality notes (kept, not deleted, but documented):**
- 94 of 189 Maternity visits are recorded against Male patients (~50%)
- 183 of 221 Pediatrics visits are for patients aged 18+
- 125 rows show Self Pay insurance paid via HMO; 46 show NHIA insurance paid in Cash

These rows were **not removed** — they appear to be artifacts of randomly generated sample data rather than data entry errors, and removing them would understate the dataset's actual volumes. They're disclosed here for transparency.

**Transformations applied in Power Query:**
- Promoted headers, confirmed data types (dates as Date, scores as Decimal, IDs as Text)
- Trimmed text columns as a safeguard
- Loaded State_Targets and Date_Table as-is

**Calculated columns added:**
```dax
Age Group =
SWITCH (
    TRUE (),
    Patient_Visits[Age] < 18, "0-17",
    Patient_Visits[Age] < 35, "18-34",
    Patient_Visits[Age] < 50, "35-49",
    Patient_Visits[Age] < 65, "50-64",
    "65+"
)

Patient Type = IF ( Patient_Visits[Visit_Count] > 1, "Returning", "New" )

Satisfaction Band =
SWITCH (
    TRUE (),
    Patient_Visits[Satisfaction_Score] <= 2, "1 - 2 (Poor)",
    Patient_Visits[Satisfaction_Score] <= 3, "2 - 3 (Below Average)",
    Patient_Visits[Satisfaction_Score] <= 4, "3 - 4 (Good)",
    "4 - 5 (Excellent)"
)
```

---

## 6. Data Model

A star schema was built around the Patient_Visits fact table:

- **Fact:** Patient_Visits
- **Dimensions:** Date_Table (marked as the official date table), State (bridges Patient_Visits and State_Targets so targets respond to a State slicer), Branch (carries State as an attribute, since branch names repeat across states), Department/Service (1:1 relationship — every department maps to exactly one service)

**Relationships:** all dimension → fact, one-to-many, single direction, built on State, Branch, Department and Date.

**Known model limitation:** because State_Targets only exists at the State level, a Branch, Department or Gender slicer will reduce Total Revenue without reducing Revenue Target — this makes Achievement % collapse when sliced below State level. This is disclosed on the Financial Performance page rather than hidden.

---

## 7. DAX Measures

20 measures were built, organized in a dedicated Measures table.

**Core:**
```dax
Total Patients = DISTINCTCOUNT ( Patient_Visits[Patient_ID] )
Total Visits = SUM ( Patient_Visits[Visit_Count] )
Total Revenue = SUM ( Patient_Visits[Revenue_NGN] )
Total Cost = SUM ( Patient_Visits[Cost_NGN] )
Total Profit = [Total Revenue] - [Total Cost]
Profit Margin % = DIVIDE ( [Total Profit], [Total Revenue] )
Avg Revenue per Patient = DIVIDE ( [Total Revenue], [Total Patients] )
Avg Waiting Time = AVERAGE ( Patient_Visits[Waiting_Time_Min] )
Avg Satisfaction = AVERAGE ( Patient_Visits[Satisfaction_Score] )
```

**Time intelligence:**
```dax
Previous Month Revenue = CALCULATE ( [Total Revenue], DATEADD ( Date_Table[Date], -1, MONTH ) )
MoM Revenue Growth % = DIVIDE ( [Total Revenue] - [Previous Month Revenue], [Previous Month Revenue] )
Previous Year Revenue = CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( Date_Table[Date] ) )
YoY Revenue Growth % = DIVIDE ( [Total Revenue] - [Previous Year Revenue], [Previous Year Revenue] )
```
*Note: YoY Revenue Growth % returns blank throughout, as the dataset covers 2025 only and no 2024 data exists for comparison. The measure is correct; the blank result is a data limitation, not a bug.*

**Targets:**
```dax
Revenue Target = SUM ( State_Targets[Annual_Revenue_Target_NGN] ) * DISTINCTCOUNT ( Date_Table[Month_Number] ) / 12
Revenue Variance = [Total Revenue] - [Revenue Target]
Achievement % = DIVIDE ( [Total Revenue], [Revenue Target] )
```

**Patient behavior:**
```dax
Avg Visits per Patient = DIVIDE ( [Total Visits], [Total Patients] )
Returning Patients = CALCULATE ( [Total Patients], Patient_Visits[Visit_Count] > 1 )
New Patients = CALCULATE ( [Total Patients], Patient_Visits[Visit_Count] = 1 )
Returning % = DIVIDE ( [Returning Patients], [Total Patients] )
```

**Dynamic insight (bonus):**
```dax
Departmental Insight = 
VAR topDept =
    MAXX ( TOPN ( 1, ALL ( Patient_Visits[Department] ), [Total Patients] ), Patient_Visits[Department] )
VAR topWaitDept =
    MAXX ( TOPN ( 1, ALL ( Patient_Visits[Department] ), [Avg Waiting Time] ), Patient_Visits[Department] )
RETURN
IF (
    topDept = topWaitDept,
    topDept & " has both the highest patient volume and the longest average wait — a likely bottleneck.",
    topDept & " has the highest patient volume, while " & topWaitDept & " has the longest average wait."
)

Insight - Executive = 
VAR topState =
    MAXX ( TOPN ( 1, ALL ( Patient_Visits[State] ), [Total Revenue] ), Patient_Visits[State] )
VAR topRev = CALCULATE ( [Total Revenue], Patient_Visits[State] = topState )
VAR topAch = CALCULATE ( [Achievement %], Patient_Visits[State] = topState )
RETURN
topState & " leads in revenue at ₦" & FORMAT ( topRev / 1000000, "#,##0.0" ) &
"M, but is only at " & FORMAT ( topAch, "0.0%" ) & " of its annual target." &   [Performance Insight]

Insight - Experience = 
VAR BestBranchTable =
    TOPN (
        1,
        ALL ( Patient_Visits[Branch] ),
        [Avg Satisfaction Score], DESC,
        Patient_Visits[Branch], ASC
    )

VAR BestBranch =
    MAXX (
        BestBranchTable,
        Patient_Visits[Branch]
    )

VAR BestSat =
    MAXX (
        BestBranchTable,
        [Avg Satisfaction Score]
    )

VAR WorstBranchTable =
    TOPN (
        1,
        ALL ( Patient_Visits[Branch] ),
        [Avg Satisfaction Score], ASC,
        Patient_Visits[Branch], ASC
    )

VAR WorstBranch =
    MAXX (
        WorstBranchTable,
        Patient_Visits[Branch]
    )

VAR WorstSat =
    MAXX (
        WorstBranchTable,
        [Avg Satisfaction Score]
    )

VAR SatisfactionGap =
    BestSat - WorstSat

VAR GapAssessment =
    SWITCH (
        TRUE (),
        SatisfactionGap >= 1, "This is a substantial difference in patient experience.",
        SatisfactionGap >= 0.5, "This indicates a noticeable difference in patient experience.",
        SatisfactionGap > 0, "The difference in patient experience is relatively small.",
        "Patient satisfaction is consistent across the branches."
    )

RETURN
    BestBranch
        & " records the highest average patient satisfaction score of "
        & FORMAT ( BestSat, "0.00" )
        & ", while "
        & WorstBranch
        & " records the lowest score of "
        & FORMAT ( WorstSat, "0.00" )
        & ". The satisfaction gap between the two branches is "
        & FORMAT ( SatisfactionGap, "0.00" )
        & " points. "
        & GapAssessment
        & " Management should examine the service practices contributing to the strong performance at "
        & BestBranch
        & " and apply relevant improvements at "
        & WorstBranch
        & ", particularly in areas such as waiting time, staff responsiveness and quality of care."

Insight - Financial = 
VAR TopProfitDeptTable =
    TOPN (
        1,
        ALL ( Patient_Visits[Department] ),
        [Total Profit], DESC,
        Patient_Visits[Department], ASC
    )

VAR TopProfitDept =
    MAXX (
        TopProfitDeptTable,
        Patient_Visits[Department]
    )

VAR TopProfitAmount =
    MAXX (
        TopProfitDeptTable,
        [Total Profit]
    )

VAR LowestStateTable =
    TOPN (
        1,
        ALL ( Patient_Visits[State] ),
        [Achievement %], ASC,
        Patient_Visits[State], ASC
    )

VAR LowestState =
    MAXX (
        LowestStateTable,
        Patient_Visits[State]
    )

VAR LowestAchievement =
    MAXX (
        LowestStateTable,
        [Achievement %]
    )

VAR TargetShortfall =
    MAX ( 0, 1 - LowestAchievement )

VAR PerformanceAssessment =
    SWITCH (
        TRUE (),
        LowestAchievement >= 1,
            "The state has met or exceeded its financial target.",
        LowestAchievement >= 0.9,
            "The state is close to reaching its financial target.",
        LowestAchievement >= 0.75,
            "The state is performing below target and requires improvement.",
        "The state is significantly below target and requires urgent management attention."
    )

RETURN
    TopProfitDept
        & " is the most profitable department, generating "
        & FORMAT ( TopProfitAmount, "₦#,##0.00" )
        & " in total profit. In contrast, "
        & LowestState
        & " records the lowest target achievement rate at "
        & FORMAT ( LowestAchievement, "0.0%" )
        & ", representing a shortfall of "
        & FORMAT ( TargetShortfall, "0.0%" )
        & " from the target. "
        & PerformanceAssessment
        & " Management should review revenue performance, operating costs and resource allocation in "
        & LowestState
        & " while examining the practices driving profitability in the "
        & TopProfitDept
        & " department."
        
Insight - Operations = 
VAR TopBranchTable =
    TOPN (
        1,
        ALL ( Patient_Visits[Branch] ),
        [Total Visits], DESC,
        Patient_Visits[Branch], ASC
    )

VAR TopBranch =
    MAXX (
        TopBranchTable,
        Patient_Visits[Branch]
    )

VAR TopBranchVisits =
    MAXX (
        TopBranchTable,
        [Total Visits]
    )

VAR TopWaitBranchTable =
    TOPN (
        1,
        ALL ( Patient_Visits[Branch] ),
        [Avg Waiting Time], DESC,
        Patient_Visits[Branch], ASC
    )

VAR TopWaitBranch =
    MAXX (
        TopWaitBranchTable,
        Patient_Visits[Branch]
    )

VAR LongestWait =
    MAXX (
        TopWaitBranchTable,
        [Avg Waiting Time]
    )

RETURN
IF (
    TopBranch = TopWaitBranch,
    TopBranch
        & " records the highest patient volume with "
        & FORMAT ( TopBranchVisits, "#,##0" )
        & " visits and also has the longest average waiting time of "
        & FORMAT ( LongestWait, "0.0" )
        & " minutes. This suggests that high patient demand may be creating an operational bottleneck. Management should review staffing levels, appointment scheduling and patient-flow processes at this branch to reduce waiting time while maintaining service quality.",
  
    TopBranch
        & " records the highest patient volume with "
        & FORMAT ( TopBranchVisits, "#,##0" )
        & " visits, while "
        & TopWaitBranch
        & " has the longest average waiting time of "
        & FORMAT ( LongestWait, "0.0" )
        & " minutes. This indicates that patient volume alone may not be responsible for the delay at "
        & TopWaitBranch
        & ". Management should investigate staffing, service capacity and workflow efficiency at that branch."
)

Performance Insight = IF([Month Growth] > 0, "Revenue is increasing by" & FORMAT([Month Growth], "0.0%"), "Revenue is decreasing by" & FORMAT(ABS([Month Growth]), "0.0%"))

**Statistical:**
```dax
Satisfaction Std Dev = STDEV.P ( Patient_Visits[Satisfaction_Score] )
```

---

## 8. Dashboard Screenshots

*[Insert screenshots of each of the 5 pages here before publishing to GitHub — Executive Overview, Patient Analysis, Hospital Operations, Financial Performance, Patient Experience.]*

---

## 9. Key Insights

**1. Pediatrics is the standout performer, not Emergency.**
Pediatrics leads in patient volume (515 visits), revenue (₦9.39M) and profit margin (39.5%) simultaneously. Emergency, despite the name, is actually the lowest-volume department (464 visits).

**2. The real operational problem is at branch level, not department level.**
Department-level waiting time (42.7–47.0 min) and satisfaction (3.69–3.74) are nearly flat across all six departments. But branch-level figures vary meaningfully: Obio-Akpor (Rivers) has the highest visit volume of any branch (303), the longest average wait (49.0 min), and satisfaction tied for lowest (3.62). High volume and worse experience are coinciding at this specific location.

**3. Waiting time does not predict patient satisfaction.**
The correlation between waiting time and satisfaction is effectively zero overall (0.002) and stays near zero within every department (-0.09 to +0.06). Reducing wait times alone is unlikely to move satisfaction scores — whatever drives satisfaction in this data, it isn't how long patients waited.

**4. Most patients are satisfied; dissatisfaction is a small, scattered group, not a systemic pattern.**
81.6% of patients scored 3 or above. Only 24 patients (2%) gave scores of 2 or below, and this group is spread across 10 of 12 branches and all 6 departments — not concentrated anywhere, including not in Lekki or Obio-Akpor, the two lowest-*average* branches. The one mild signal: this group's average wait (51.2 min) runs about 6 minutes above the overall average, though with n=24 this is a weak signal, not a conclusion.

**5. No state is close to its revenue target, and the gap is uniform.**
Every state tracks at 21–28% of its annual target, with no state above 30%. Rivers and Abuja lead in absolute revenue, but Abuja has the best achievement rate because it carries the lowest target. A uniform shortfall across all four states — rather than one underperforming region — points more toward how targets were set than toward execution problems at any single branch.

**6. Diagnosis mix is consistent across age groups.**
A chi-square test of independence (p = 0.23) found no statistically significant relationship between Age Group and Diagnosis — Malaria and Typhoid are the top two diagnoses in every age group, in similar proportions. This rules out an age-driven diagnosis pattern, which is itself useful for ruling out false leads.

---

## 10. Recommendations

1. **Investigate Obio-Akpor branch specifically**, not Rivers state broadly or any department broadly — it is the one location where high volume, long wait and lower satisfaction coincide.
2. **Do not invest in wait-time reduction as a satisfaction strategy** on the strength of this data alone — there's no measurable link between the two. Investigate other drivers (staff interaction, facility condition, communication) instead.
3. **Review how the ₦200M annual revenue target was set.** A uniform 21–28% achievement rate across every state, every month, suggests the target itself may be the issue, not state-level execution. Confirm with whoever set it before using Achievement % as a performance scorecard.
4. **Study Pediatrics as an internal benchmark.** It leads on volume, revenue and margin at once — understanding what it's doing differently could inform the other five departments.
5. **Treat the 24 low-satisfaction patients as individual cases for follow-up**, not as evidence of a branch- or department-wide failure — the data doesn't support a systemic explanation for this group.

---

## 11. Conclusion

This dashboard moves the organization from scattered spreadsheet data to a structured, interactive view of performance across patients, operations, finance and experience. The most actionable finding is that **branch-level variation, not department-level variation, is where real operational and experience differences live** — department averages are consistently flat across this dataset, which could itself mislead a dashboard that only reported at department level. Equally important is what the data **doesn't** show: no link between wait time and satisfaction, no age-driven diagnosis pattern, and no branch-level concentration of individual dissatisfaction — each confirmed by testing the claim rather than assuming it. Combined with the revenue target gap, these findings give management a specific, evidence-backed starting point for investigation rather than a generic list of charts.

---

## Author's Note on Methodology

Every insight in this README was checked against the underlying data (via Python/pandas cross-tabulation and statistical testing) before being written up, including a chi-square test for the age/diagnosis relationship and a scattered-vs-concentrated check on low-satisfaction patients. Two early hypotheses (an age-diagnosis link, and low scores clustering at specific branches) were tested and **not** supported by the data, and are reported as such rather than omitted — in keeping with the project requirement that insights come from analysis, not assumption.
