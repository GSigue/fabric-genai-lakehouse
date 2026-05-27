# DAX Measures — Gold Semantic Model

18 measures, organized by category. Each measure includes the formula and the business question it answers.

> **Tip for the report build:** create a single hidden measures table called `_Measures` and put all of these there. It keeps the field list clean and signals professional practice to anyone opening the .pbix.

---

## 1. Base counts & sums

### Total Trips
**Q: How many trips happened?**
```dax
Total Trips = COUNTROWS ( FactTrip )
```

### Total Passengers
**Q: How many passengers were carried?**
```dax
Total Passengers = SUM ( FactTrip[passenger_count] )
```

### Total Revenue
**Q: What was total revenue?**
```dax
Total Revenue = SUM ( FactTrip[total_amount] )
```

### Total Tips
**Q: What did drivers earn in tips?**
```dax
Total Tips = SUM ( FactTrip[tip_amount] )
```

---

## 2. Distinct counts

### Active Zones
**Q: How many pickup zones saw activity in the period?**
```dax
Active Zones =
CALCULATE (
    DISTINCTCOUNT ( FactTrip[pickup_location_key] ),
    FactTrip[pickup_location_key] <> BLANK ()
)
```

### Active Vendors
**Q: How many vendors operated in the period?**
```dax
Active Vendors = DISTINCTCOUNT ( FactTrip[vendor_key] )
```

---

## 3. Averages & ratios

### Avg Fare per Trip
**Q: What does a typical trip cost?**
```dax
Avg Fare per Trip =
DIVIDE ( [Total Revenue], [Total Trips] )
```

### Avg Trip Distance (mi)
**Q: How far is the average trip?**
```dax
Avg Trip Distance (mi) =
DIVIDE ( SUM ( FactTrip[trip_distance] ), [Total Trips] )
```

### Avg Trip Duration (min)
**Q: How long is the average trip?**
```dax
Avg Trip Duration (min) =
DIVIDE ( SUM ( FactTrip[trip_duration_seconds] ), [Total Trips] ) / 60
```

### Tip % of Fare
**Q: What share of fare comes back as tip?**
```dax
Tip % of Fare =
DIVIDE ( [Total Tips], [Total Revenue] - [Total Tips] )
```
*Denominator excludes tips so the ratio reflects tipping on pre-tip fare. Format as percentage.*

---

## 4. Iterators (the AVERAGEX flex)

### Avg Revenue per Active Zone
**Q: How much revenue does a typical zone generate per day?**
```dax
Avg Revenue per Active Zone =
AVERAGEX (
    VALUES ( DimLocation[location_key] ),
    CALCULATE ( [Total Revenue] )
)
```

### Peak Hour Trip Count
**Q: How busy is the single busiest hour of the period?**
```dax
Peak Hour Trip Count =
MAXX (
    VALUES ( DimDate[date_key] ),
    CALCULATE ( [Total Trips] )
)
```

---

## 5. Time intelligence

### Trips YoY
**Q: How does this period compare to the same period last year?**
```dax
Trips YoY =
VAR _current = [Total Trips]
VAR _prior =
    CALCULATE ( [Total Trips], SAMEPERIODLASTYEAR ( DimDate[full_date] ) )
RETURN
    DIVIDE ( _current - _prior, _prior )
```
*Format as percentage. Use a +/- icon on the visual.*

### Trips QoQ
**Q: How does this quarter compare to the previous quarter?**
```dax
Trips QoQ =
VAR _current = [Total Trips]
VAR _prior =
    CALCULATE (
        [Total Trips],
        DATEADD ( DimDate[full_date], -1, QUARTER )
    )
RETURN
    DIVIDE ( _current - _prior, _prior )
```

### Trips T12M
**Q: What's the trailing 12-month trip total?**
```dax
Trips T12M =
CALCULATE (
    [Total Trips],
    DATESINPERIOD (
        DimDate[full_date],
        MAX ( DimDate[full_date] ),
        -12,
        MONTH
    )
)
```

### Revenue T12M
**Q: What's trailing 12-month revenue?**
```dax
Revenue T12M =
CALCULATE (
    [Total Revenue],
    DATESINPERIOD (
        DimDate[full_date],
        MAX ( DimDate[full_date] ),
        -12,
        MONTH
    )
)
```

### Trips 7-Day Avg
**Q: Smoothed daily trip count for trend lines / Data Activator baseline.**
```dax
Trips 7-Day Avg =
AVERAGEX (
    DATESINPERIOD (
        DimDate[full_date],
        MAX ( DimDate[full_date] ),
        -7,
        DAY
    ),
    [Total Trips]
)
```

---

## 6. The Data Activator trigger measure

### Daily Trip Count Drop %
**Q: How much did today's volume fall vs the trailing 7-day average?**
```dax
Daily Trip Count Drop % =
VAR _today = [Total Trips]
VAR _baseline = [Trips 7-Day Avg]
RETURN
    DIVIDE ( _baseline - _today, _baseline )
```
*Data Activator rule: alert when this exceeds 0.30 (a 30% drop) for a single day at the all-up grain.*

---

## Notes on idiom and review

**On DIVIDE vs `/`.** Use `DIVIDE`. It handles divide-by-zero. The `/` operator gives you infinity/NaN errors that surface badly in visuals.

**On VAR/RETURN.** Use it for any measure with 2+ steps. Easier to read, easier to debug, and the engine evaluates each VAR once which can be measurably faster.

**On qualified column references.** `Table[Column]` always. Never bare `[Column]` for actual columns — that's reserved for measure references in your head, and DAX won't enforce the distinction. Using qualified names everywhere makes the difference visually obvious.

**On `CALCULATE` over `FILTER` when possible.** `CALCULATE ( [Measure], Table[Col] = "X" )` is faster than `CALCULATE ( [Measure], FILTER ( Table, Table[Col] = "X" ) )` for simple equality filters. Use FILTER only when you need the row-context.
