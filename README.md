# Software Delivery & Quality Analytics

![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Measures-0078D4)
![ZoomCharts](https://img.shields.io/badge/ZoomCharts-Drill%20Down%20Visuals-6C5CE7)

<!-- Add your dashboard screenshots to /images and update the paths below -->
![Release Performance](hero.png)

---

## 1. Introduction

Shipping software fast is only useful if the releases are also stable. This project analyzes **643 software releases from 2024 and 2025** across **36 product modules** and **15 engineering teams**. The data covers planned vs. actual release dates, testing effort, defects found in production, incidents, downtime and the estimated cost of that downtime.

The report has two pages:

- **Release Performance:** delivery speed, punctuality and testing effectiveness.
- **Business Impact:** production defects, incidents, downtime, cost and a ranked priority list of products and modules.

[ZoomCharts Drill Down visuals](https://zoomcharts.com/) let managers move from a headline KPI to a team, product, module or day without leaving the page.

## 2. Problem Statement

Managers need clear answers to six questions:

- Are software updates delivered on time, and where are the biggest delays?
- Which products, teams or releases have the most software issues?
- Does more testing, or more test automation, reduce problems after release?
- Which releases cause the most incidents, downtime and business cost?
- Which teams and products perform best, and which need attention?
- Is delivery becoming faster and more reliable over time?

## 3. Skills Demonstrated

- **Data modeling:** star schema, role-playing date dimension, inactive relationships, field parameters.
- **DAX:** base measures, variables, time-period comparison, percentile ranking, dynamic text and color measures, user-defined functions.
- **Power Query:** typed imports, calculated bands and categories, sort-by columns.
- **Data validation:** key checks, orphan checks, reconciliation of totals, and independent recomputation of key measures.
- **Visual design:** KPI cards with dynamic reference labels, conditional formatting, a dark theme and ZoomCharts drill-down visuals.
- **Analytical thinking:** turning metrics into a business priority score and recommended actions.
- **Model auditing:** duplicate detection, dependency mapping and safe cleanup of measures.

## 4. Data Sourcing

The data comes from the challenge workbook `Software_Delivery_Quality_challenge_dataset.xlsx`, loaded into Power BI through Power Query in Import mode.

| Table | Content |
|---|---|
| `FactReleasePerformance` | One row per release (643): planned and actual dates, development and testing days, tests executed and failed, automation %, defects, incidents, downtime minutes, resolution hours |
| `ReleaseItems` | Items shipped in each release (6,226 rows) with item type and category |
| `DimProduct` | 36 product modules with product, module and product group |
| `DimTeam` | 15 delivery teams |
| `DimReleaseType` | Major, Minor and Patch |
| `DowntimeCost` | Downtime cost per minute for each product module, used to estimate business cost |
| `DimDate` | Calendar table built in Power Query (2023–2025), marked as the date table |
| `DownTime_Parameter` | Field parameter that switches downtime between hours and minutes |
| `_Measure` | Holds all DAX measures, organized in display folders |

Key facts at a glance:

- **Period:** releases from 2 January 2024 to 31 December 2025.
- **Releases:** 643, with 738 production defects (79 critical) and 218 production incidents.
- **Downtime:** 11,632 minutes (about 194 hours), estimated at about **$2.95M**.
- **Testing:** 373,153 tests executed.

## 5. Data Transformation

Power Query sets data types and builds the date table. Calculated columns add the report's categories:

- **Release delay:** `Release Delay Days` is the difference between planned and actual release dates.
- **Development effort bands:** Short-term (3–7 days), Standard Sprint (8–14), Extended Sprint (15–30), Long-term (over 30).
- **Downtime cost categories:** six cost bands from Minimal to Critical.
- **Date helpers:** Day, Day Number, and Weekend/Weekday.
- **Release Time of Day:** Morning, Afternoon, Evening or Night. The source has no time of day, so this column is **synthetic** and generated for demonstration. Do not use it for conclusions.

All business metrics are DAX measures in `_Measure`. Each measure reuses a base measure where it can, so definitions stay consistent.

### Core DAX measures

#### Delivery performance

```dax
Release Count = DISTINCTCOUNT(FactReleasePerformance[Release ID])

On-Time Release Count = CALCULATE([Release Count], FactReleasePerformance[Release Delay Days] <= 0)

On-Time Release Rate = DIVIDE([On-Time Release Count], [Release Count])

Delayed Release Count = CALCULATE([Release Count], FactReleasePerformance[Release Delay Days] > 0)

Average Release Delay Days =
CALCULATE(AVERAGE(FactReleasePerformance[Release Delay Days]), FactReleasePerformance[Release Delay Days] > 0)
```

#### Testing and quality

```dax
Test Failure Rate = DIVIDE([Tests Failed], [Tests Executed])

Test Automation Rate = AVERAGE(FactReleasePerformance[Automated Tests %])

Critical Defect Rate = DIVIDE([Critical Production Defects], [Production Defects])

Incident-Affected Release Rate = DIVIDE(COALESCE([Releases with Incidents], 0), [Release Count])
```

#### Incidents, downtime and cost

```dax
Total Downtime Hours = DIVIDE([Total Downtime Minutes], 60)

Estimated Downtime Cost =
SUMX(
    FactReleasePerformance,
    FactReleasePerformance[Downtime Minutes] * RELATED(DowntimeCost[Downtime Cost per Minute ($)])
)

Average Incident Resolution Hours =
DIVIDE(
    SUMX(
        FactReleasePerformance,
        FactReleasePerformance[Avg Incident Resolution Hours] * FactReleasePerformance[Production Incidents]
    ),
    [Production Incidents]
)
```

#### Period-over-period comparison (powers the KPI reference labels)

The comparison window follows the months the user selects. It compares them with the same number of months immediately before.

```dax
Previous Period On-Time Release Rate =
VAR _ps = [Previous Period Start]
VAR _pe = [Previous Period End]
RETURN
    IF(
        [Previous Period Available] = 1,
        CALCULATE([On-Time Release Rate], REMOVEFILTERS(DimDate), DATESBETWEEN(DimDate[Date], _ps, _pe))
    )

Change On-Time Release Rate (pp) =
VAR _c = [On-Time Release Rate]
VAR _p = [Previous Period On-Time Release Rate]
RETURN
    IF(NOT ISBLANK(_c) && NOT ISBLANK(_p), (_c - _p) * 100)
```

#### Business Impact Priority Score (0–100)

Each product module is ranked against its peers on four factors. Each rank becomes a percentile from 0 to 100. The score is the weighted average, with weights of 40% for cost, 25% for critical defect rate, 20% for incident-affected release rate and 15% for resolution time.

```dax
Priority Cost Percentile =
VAR _min = [Priority Min Releases]
VAR _peers =
    CALCULATETABLE(
        ADDCOLUMNS(
            VALUES(FactReleasePerformance[Product ID]),
            "@v", VAR _x = [Estimated Downtime Cost] RETURN IF(ISBLANK(_x), BLANK(), ROUND(_x, 6)),
            "@n", [Release Count]
        ),
        REMOVEFILTERS(DimProduct),
        REMOVEFILTERS(DowntimeCost)
    )
VAR _valid = FILTER(_peers, [@n] >= _min && NOT ISBLANK([@v]))
VAR _x0 = [Estimated Downtime Cost]
VAR _cur = IF(ISBLANK(_x0), BLANK(), ROUND(_x0, 6))
VAR _ok = DISTINCTCOUNT(FactReleasePerformance[Product ID]) = 1 && [Release Count] >= _min && NOT ISBLANK(_cur)
VAR _n = COUNTROWS(_valid)
VAR _l = COUNTROWS(FILTER(_valid, [@v] < _cur))
VAR _e = COUNTROWS(FILTER(_valid, [@v] = _cur))
RETURN
    IF(_ok && _n >= 2, 100 * DIVIDE(_l + 0.5 * (_e - 1), _n - 1))

Business Impact Priority Score =
VAR _pC = [Priority Cost Percentile]
VAR _pD = [Priority Critical Defect Percentile]
VAR _pI = [Priority Incident Rate Percentile]
VAR _pR = [Priority Resolution Percentile]
VAR _wC = IF(ISBLANK(_pC), 0, 0.40)
VAR _wD = IF(ISBLANK(_pD), 0, 0.25)
VAR _wI = IF(ISBLANK(_pI), 0, 0.20)
VAR _wR = IF(ISBLANK(_pR), 0, 0.15)
VAR _w = _wC + _wD + _wI + _wR
VAR _sum = COALESCE(_pC, 0) * _wC + COALESCE(_pD, 0) * _wD + COALESCE(_pI, 0) * _wI + COALESCE(_pR, 0) * _wR
RETURN
    IF(NOT ISBLANK(_pC) && _w > 0, MIN(100, MAX(0, DIVIDE(_sum, _w))))
```

The `Recommended Action` measure then reads the score and the four percentiles. It returns a short action such as *Prioritize Critical Defect Remediation*, *Investigate Release Reliability*, *Improve Incident Recovery* or *Review Downtime Cost Drivers*. It returns *Insufficient Data* when there is not enough data to rank a module.

Color measures return the hex values used for conditional formatting: `#2DD4BF` (favorable), `#FBBF24` (medium), `#FB7185` (high risk) and `#9AA6D1` (no data).

## 6. Modeling

The model is a star schema with `FactReleasePerformance` at the center. All relationships are single-direction, many-to-one.

```text
DimDate  ──(Release Date)──┐
DimDate  ┄(Planned Date)┄┐ │   (inactive, used via USERELATIONSHIP)
DimProduct ───────────────┼─┼──►  FactReleasePerformance  ◄── ReleaseItems
DimTeam ──────────────────┤ │
DimReleaseType ───────────┤ │
DowntimeCost ─────────────┘ │
```

Design choices:

- **Role-playing dates:** the active relationship uses the actual release date. The planned release date is an inactive relationship used only in measures such as planned release count.
- **Validation:** 643 release rows with 643 distinct Release IDs, 36 products, 15 teams and 3 release types, and no orphan rows on product, cost, team or date.
- **Cost reconciliation:** the model's cost of $2,950,024.31 matches an independent product-by-product recomputation.
- **Field parameter:** an Hours/Minutes switch changes the unit of the downtime visuals.
- **Peer-based scoring:** the priority score ignores the product and module selection when ranking, so a module's score does not change when it is clicked.
- **Model audit:** duplicate measures were removed, measures that follow the same pattern were aligned, and the remaining unused measures were documented.

## 7. Analysis & Visualization

### Page 1: Release Performance

**Visuals and purpose**

- **KPI cards with reference labels:** average release delay, on-time release rate, test failure rate and test automation rate, each showing the previous-period comparison.
- **Trend lines under each KPI:** show whether each metric is improving.
- **ZoomCharts Timeline (On-Time Release Trend):** tracks punctuality over time and drills from year to quarter, month and day.
- **ZoomCharts Facet Chart (Release Delay by Team):** compares delay across teams.
- **ZoomCharts Combo Bar (Tests Executed and Failed by Product):** relates testing effort to failures.
- **ZoomCharts Combo Bar (Downtime by Cost Category) with an Hours/Minutes switch:** shows where downtime concentrates.
- **ZoomCharts Pie (Incidents by Day & Item Type):** splits incidents by weekday/weekend and item type.
- **Slicers:** product group, release type and development days.

**Key insights**

- **Punctuality is only fair.** 65.5% of releases are on time. The 222 late releases are about 4.1 days late on average.
- **Delivery is improving.** On-time rate rose from 64.1% in 2024 to 66.9% in 2025. Average delay fell from 4.5 to 3.7 days. Q4 2025 slipped to 60.9%, the lowest quarter in the two years.
- **Teams differ widely.** Beacon (84.8%), Kestrel (79.2%) and Aurora (78.8%) are most punctual. Pulse (30.8%), Vertex (43.8%), Orion (46.2%) and Titan (46.9%) are the least punctual. Orion has the longest average delay at 6.8 days.
- **More automation goes with fewer problems.** Releases under 60% automation have 1.4 defects per release and a 5.9% test failure rate. Releases at 90% or above have 0.8 and 2.2%. This is an association, not proof of cause.
- **Test failure rate is stable.** It is 4.1% overall and barely changed between years (4.2% to 4.0%).

### Page 2: Business Impact

**Visuals and purpose**

- **KPI cards:** production defects, incidents, downtime hours and estimated downtime cost, with reference labels (for example, average cost per incident).
- **ZoomCharts Combo Bar (Production Defect & Critical by Product):** compares total and critical defects by product.
- **ZoomCharts Combo Bar (Releases with Downtime):** shows downtime by day, switchable between hours and minutes.
- **ZoomCharts Timeline (Downtime & Cost Trend):** tracks downtime and its estimated cost over time.
- **ZoomCharts Facet Chart (Releases with Incidents by Team):** compares incidents and cost by team.
- **Priority table:** product, module, critical defect rate, incident-affected release rate, resolution time, **Business Impact Priority Score** and **Recommended Action**, with conditional formatting.

**Key insights**

- **The business cost is concentrated.** Downtime of 194 hours costs about **$2.95M**, or about $13.5K per incident. The Platform and Commerce product groups hold **73%** of that cost. The 10 most expensive product modules carry **85%**.
- **Major releases carry the most risk.** Major releases are 15% of all releases but cause 38% of incidents (0.85 per release, against 0.32 for Minor and 0.10 for Patch). They account for 34% of cost.
- **Long projects are riskier.** Releases with over 30 days of development average 0.78 incidents and 2.1 defects each. Standard sprints (8–14 days) average 0.11 incidents.
- **Fast does not mean safe.** Beacon is the most punctual team, yet it has the second-highest cost ($608K), after Atlas ($826K). Atlas and Beacon together hold about 49% of the cost.
- **Cost fell, incidents did not.** Estimated downtime cost fell 14% (from $1.59M to $1.36M) and downtime hours fell 9%. However, incidents rose from 105 to 113.
- **Top priorities by score.** BillingPro Tax (92.3), Core Integrations (87.0), SearchAI Semantic Search (85.4) and BillingPro Invoicing (84.8). BillingPro Tax has 6 critical defects out of 16 defects. BillingPro Invoicing has the highest cost ($556K).
- **Incidents are not rare.** 22.7% of releases (146 of 643) had at least one production incident. Incidents are resolved in about 3.2 hours on average.

### Interaction design

- Drill down from year to quarter, month and day, or from product to module, using ZoomCharts.
- Dynamic reference labels show the previous period for any selected months.
- A consistent color convention is used: teal (favorable), amber (watch), pink-red (high risk), lavender-gray (no data).
- Conditional formatting is applied to the priority table, KPI labels and trend visuals.

## 8. Conclusion

The company is getting better at delivering on time, and the cost of downtime fell by 14% in 2025. Quality is not improving as fast: incidents rose, and the cost is concentrated in a few product modules, in Major releases and in long development efforts. Four teams deliver on time less than half the time.

### Recommendations

1. **Fix delivery in the four slowest teams.** Pulse, Vertex, Orion and Titan are on time less than 50% of the time. Review their planning and estimation against Beacon and Kestrel, and set an on-time target above the current 65.5%.
2. **Add safeguards to Major and long-running releases.** Use a readiness review, staged rollout and rollback plan for Major releases and for work longer than 30 days.
3. **Remediate the top-ranked modules first.** Start with BillingPro Tax, Core Integrations, SearchAI Semantic Search and BillingPro Invoicing, then the rest of Platform and Commerce, which hold 73% of the cost.
4. **Raise test automation.** More than half of all releases (339 of 643) run under 75% automation, and the less-automated releases show higher failure and defect rates.
5. **Track quality, not only speed.** Add incidents per release and Business Impact Priority Score to the regular delivery review. Investigate the Q4 2025 punctuality drop.

## Notes and limitations

- The **Release Time of Day** column is synthetic and for demonstration only.
- Incidents and downtime are recorded per release. The item-type pie chart therefore counts each release's incidents under every item type it contains. Read it as "incidents in releases that include this item type", not as incidents caused by that item type.
- Priority score thresholds (High 75, Medium 50) are initial analytical values, not service-level standards. They are stored in config measures and are easy to change.
- Relationships between automation or testing and quality are associations in observational data, not proof of cause.
- Product modules with fewer than 3 releases are not scored and show *Insufficient Data*.

## Project contents

- `Software Delivery & Quality Analytics.pbix`: Power BI project.
- `README.md`: this documentation.
