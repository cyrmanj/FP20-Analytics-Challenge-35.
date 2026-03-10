# DAX Measures Reference — FP20 Challenge 35
> Enterprise Sales Pipeline & Deals Analytics

---

## 1. Core Pipeline & Revenue

```dax
Total Deal Value =
    SUM ( FactDeals[DealValueEUR] )
```

```dax
Open Pipeline Value =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Open"
    )
```

```dax
Won Revenue =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Won"
    )
```

```dax
Lost Value =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Lost"
    )
```

```dax
Total Deal Count =
    COUNTROWS ( FactDeals )
```

```dax
Open Deal Count =
    CALCULATE (
        COUNTROWS ( FactDeals ),
        FactDeals[Status] = "Open"
    )
```

```dax
Won Deal Count =
    CALCULATE (
        COUNTROWS ( FactDeals ),
        FactDeals[Status] = "Won"
    )
```

```dax
Lost Deal Count =
    CALCULATE (
        COUNTROWS ( FactDeals ),
        FactDeals[Status] = "Lost"
    )
```

```dax
Avg Deal Value =
    AVERAGE ( FactDeals[DealValueEUR] )
```

---

## 2. Win Rate & Conversion

```dax
Win Rate =
    VAR _closed =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] IN { "Won", "Lost" }
        )
    VAR _won = [Won Deal Count]
    RETURN
        DIVIDE ( _won, _closed, 0 )
```

```dax
Loss Rate =
    1 - [Win Rate]
```

```dax
Stage Conversion Rate =
    VAR _dealsInStage =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            ALLEXCEPT ( FactDeals, FactDeals[StageName] )
        )
    VAR _wonFromStage =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            ALLEXCEPT ( FactDeals, FactDeals[StageName] ),
            FactDeals[Status] = "Won"
        )
    RETURN
        DIVIDE ( _wonFromStage, _dealsInStage, 0 )
```

---

## 3. Weighted Pipeline & Forecast

```dax
Weighted Pipeline Value =
    CALCULATE (
        SUMX (
            FactDeals,
            FactDeals[DealValueEUR] * FactDeals[BaseWinProbability]
        ),
        FactDeals[Status] = "Open"
    )
```

```dax
Forecast Accuracy =
    VAR _prevWeighted =
        CALCULATE (
            [Weighted Pipeline Value],
            DATEADD ( DimDate[Date], -1, MONTH )
        )
    VAR _actualWon = [Won Revenue]
    RETURN
        IF (
            _prevWeighted = 0,
            BLANK (),
            1 - ABS ( DIVIDE ( _actualWon - _prevWeighted, _prevWeighted, 0 ) )
        )
```

```dax
Pipeline Coverage Ratio =
    DIVIDE ( [Open Pipeline Value], [Won Revenue], 0 )
```

---

## 4. Sales Cycle & Time

```dax
Avg Sales Cycle Days (Won) =
    VAR _wonDeals =
        FILTER ( FactDeals, FactDeals[Status] = "Won" )
    RETURN
        AVERAGEX (
            _wonDeals,
            VAR _created =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[CreatedDateKey]
                )
            VAR _closed =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[LastActivityDateKey]
                )
            RETURN
                DATEDIFF ( _created, _closed, DAY )
        )
```

```dax
Avg Sales Cycle Days (Lost) =
    VAR _lostDeals =
        FILTER ( FactDeals, FactDeals[Status] = "Lost" )
    RETURN
        AVERAGEX (
            _lostDeals,
            VAR _created =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[CreatedDateKey]
                )
            VAR _closed =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[LastActivityDateKey]
                )
            RETURN
                DATEDIFF ( _created, _closed, DAY )
        )
```

```dax
Avg Sales Cycle Days (SMB) =
    CALCULATE (
        [Avg Sales Cycle Days (Won)],
        DimCompany[CompanySize] = "SMB"
    )
```

```dax
Avg Sales Cycle Days (Enterprise) =
    CALCULATE (
        [Avg Sales Cycle Days (Won)],
        DimCompany[CompanySize] = "Enterprise"
    )
```

```dax
Days Since Last Activity (Avg Open) =
    VAR _today = TODAY ()
    VAR _openDeals =
        FILTER ( FactDeals, FactDeals[Status] = "Open" )
    RETURN
        AVERAGEX (
            _openDeals,
            VAR _lastAct =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[LastActivityDateKey]
                )
            RETURN
                DATEDIFF ( _lastAct, _today, DAY )
        )
```

---

## 5. At-Risk Deals

```dax
At-Risk Deal Count =
    VAR _today = TODAY ()
    VAR _threshold = 30
    RETURN
    CALCULATE (
        COUNTROWS ( FactDeals ),
        FactDeals[Status] = "Open",
        FILTER (
            FactDeals,
            VAR _lastAct =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[LastActivityDateKey]
                )
            RETURN
                DATEDIFF ( _lastAct, _today, DAY ) > _threshold
        )
    )
```

```dax
At-Risk Pipeline Value =
    VAR _today = TODAY ()
    VAR _threshold = 30
    RETURN
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Open",
        FILTER (
            FactDeals,
            VAR _lastAct =
                LOOKUPVALUE (
                    DimDate[Date],
                    DimDate[DateKey], FactDeals[LastActivityDateKey]
                )
            RETURN
                DATEDIFF ( _lastAct, _today, DAY ) > _threshold
        )
    )
```

```dax
At-Risk Deal % =
    DIVIDE ( [At-Risk Deal Count], [Open Deal Count], 0 )
```

---

## 6. Activity & Engagement

```dax
Total Activities =
    COUNTROWS ( FactActivities )
```

```dax
Avg Activities per Deal =
    DIVIDE ( [Total Activities], [Total Deal Count], 0 )
```

```dax
Avg Activities per Won Deal =
    VAR _wonDealIDs =
        CALCULATETABLE (
            VALUES ( FactDeals[DealID] ),
            FactDeals[Status] = "Won"
        )
    VAR _wonActivities =
        CALCULATE (
            COUNTROWS ( FactActivities ),
            TREATAS ( _wonDealIDs, FactActivities[DealID] )
        )
    RETURN
        DIVIDE ( _wonActivities, [Won Deal Count], 0 )
```

```dax
Avg Activities per Lost Deal =
    VAR _lostDealIDs =
        CALCULATETABLE (
            VALUES ( FactDeals[DealID] ),
            FactDeals[Status] = "Lost"
        )
    VAR _lostActivities =
        CALCULATE (
            COUNTROWS ( FactActivities ),
            TREATAS ( _lostDealIDs, FactActivities[DealID] )
        )
    RETURN
        DIVIDE ( _lostActivities, [Lost Deal Count], 0 )
```

```dax
Total Activity Duration (Hours) =
    DIVIDE ( SUM ( FactActivities[DurationMinutes] ), 60, 0 )
```

```dax
Meeting Count =
    CALCULATE (
        COUNTROWS ( FactActivities ),
        FactActivities[ActivityType] = "Meeting"
    )
```

```dax
Demo Count =
    CALCULATE (
        COUNTROWS ( FactActivities ),
        FactActivities[ActivityType] = "Demo"
    )
```

---

## 7. Sales Rep Performance

```dax
Rep Win Rate =
    VAR _won = [Won Deal Count]
    VAR _closed =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] IN { "Won", "Lost" }
        )
    RETURN
        DIVIDE ( _won, _closed, 0 )
```

```dax
Rep Pipeline Value =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Open"
    )
```

```dax
Rep Activity Count =
    COUNTROWS ( FactActivities )
```

```dax
Rep Won Revenue =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Won"
    )
```

```dax
Rep Avg Deal Size =
    AVERAGEX ( FactDeals, FactDeals[DealValueEUR] )
```

---

## 8. Period-over-Period & Trends

> ⚠️ Requires DimDate marked as a Date Table in Power BI.

```dax
Won Revenue MoM % =
    VAR _current = [Won Revenue]
    VAR _prior =
        CALCULATE (
            [Won Revenue],
            DATEADD ( DimDate[Date], -1, MONTH )
        )
    RETURN
        DIVIDE ( _current - _prior, _prior, 0 )
```

```dax
Won Revenue YoY % =
    VAR _current = [Won Revenue]
    VAR _prior =
        CALCULATE (
            [Won Revenue],
            SAMEPERIODLASTYEAR ( DimDate[Date] )
        )
    RETURN
        DIVIDE ( _current - _prior, _prior, 0 )
```

```dax
Open Pipeline MoM % =
    VAR _current = [Open Pipeline Value]
    VAR _prior =
        CALCULATE (
            [Open Pipeline Value],
            DATEADD ( DimDate[Date], -1, MONTH )
        )
    RETURN
        DIVIDE ( _current - _prior, _prior, 0 )
```

```dax
Deals Created MTD =
    CALCULATE (
        [Total Deal Count],
        DATESMTD ( DimDate[Date] )
    )
```

```dax
Deals Created QTD =
    CALCULATE (
        [Total Deal Count],
        DATESQTD ( DimDate[Date] )
    )
```

---

## 9. Funnel Stage Analysis

```dax
Deals in Stage =
    CALCULATE (
        COUNTROWS ( FactDeals ),
        FactDeals[Status] = "Open"
    )
```

```dax
Pipeline Value in Stage =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Open"
    )
```

```dax
Stage Drop-Off Rate =
    VAR _total = COUNTROWS ( FactDeals )
    VAR _lost =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] = "Lost"
        )
    RETURN
        DIVIDE ( _lost, _total, 0 )
```

```dax
Avg Deal Value in Stage =
    CALCULATE (
        AVERAGE ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Open"
    )
```

---

## 10. Product Category

```dax
Win Rate by Product =
    VAR _won =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] = "Won"
        )
    VAR _closed =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] IN { "Won", "Lost" }
        )
    RETURN
        DIVIDE ( _won, _closed, 0 )
```

```dax
Won Revenue by Product =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Won"
    )
```

```dax
Product Pipeline Share % =
    DIVIDE (
        [Open Pipeline Value],
        CALCULATE (
            [Open Pipeline Value],
            ALL ( FactDeals[ProductCategory] )
        ),
        0
    )
```

---

## 11. Region & Industry

```dax
Region Win Rate =
    VAR _won =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] = "Won"
        )
    VAR _closed =
        CALCULATE (
            COUNTROWS ( FactDeals ),
            FactDeals[Status] IN { "Won", "Lost" }
        )
    RETURN
        DIVIDE ( _won, _closed, 0 )
```

```dax
Region Pipeline Value =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Open"
    )
```

```dax
Industry Won Revenue =
    CALCULATE (
        SUM ( FactDeals[DealValueEUR] ),
        FactDeals[Status] = "Won"
    )
```

```dax
Industry Pipeline Share % =
    DIVIDE (
        [Open Pipeline Value],
        CALCULATE (
            [Open Pipeline Value],
            ALL ( DimCompany[Industry] )
        ),
        0
    )
```

---

## Model Relationships

| From | To | Key | Status |
|---|---|---|---|
| FactDeals | DimDate | CreatedDateKey | ✅ Active |
| FactDeals | DimDate | LastActivityDateKey | ⚠️ Inactive |
| FactDeals | DimCompany | CompanyID | ✅ Active |
| FactDeals | DimSalesRep | RepID | ✅ Active |
| FactActivities | FactDeals | DealID | ✅ Active |
| FactActivities | DimDate | ActivityDateKey | ✅ Active |
| FactActivities | DimSalesRep | RepID | ⚠️ Inactive |

> To use `LastActivityDateKey` in time intelligence measures, activate it with `USERELATIONSHIP()` inside a `CALCULATE`.
