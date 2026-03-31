# How to Calculate Business Days Between Dates in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Business Day, SLA, Analytics

Description: ClickHouse has no built-in businessDay function. Learn how to count weekdays between dates using toDayOfWeek arithmetic and generate business day sequences for SLA analysis.

---

ClickHouse does not ship a dedicated `businessDays` function, but you can calculate the number of weekdays (Monday through Friday) between two dates using integer arithmetic on `toDayOfWeek()`, which returns 1 for Monday through 7 for Sunday. The approach involves counting full weeks (each containing exactly 5 weekdays) and then handling the partial week at each end. This technique is essential for SLA calculations, invoice due-date computation, and any business metric that should ignore weekends.

## Understanding toDayOfWeek

`toDayOfWeek(date)` returns an integer from 1 (Monday) to 7 (Sunday) by default. Knowing the day-of-week for the start and end dates is the foundation of all weekday counting logic.

```sql
-- See the day-of-week values for a sample range
SELECT
    toDate('2024-06-10') AS monday,    toDayOfWeek(toDate('2024-06-10')) AS dow_monday,
    toDate('2024-06-12') AS wednesday, toDayOfWeek(toDate('2024-06-12')) AS dow_wednesday,
    toDate('2024-06-15') AS saturday,  toDayOfWeek(toDate('2024-06-15')) AS dow_saturday,
    toDate('2024-06-16') AS sunday,    toDayOfWeek(toDate('2024-06-16')) AS dow_sunday;
```

```text
monday      dow_monday  wednesday   dow_wednesday  saturday    dow_saturday  sunday      dow_sunday
2024-06-10  1           2024-06-12  3              2024-06-15  6             2024-06-16  7
```

## Core Business Day Count Formula

The standard formula for counting weekdays between `start_date` and `end_date` (inclusive of start, exclusive of end) breaks the period into full weeks and a partial remainder.

```sql
-- Business days between two dates (start inclusive, end exclusive)
SELECT
    toDate('2024-06-10') AS start_date,  -- Monday
    toDate('2024-06-20') AS end_date,    -- Thursday
    dateDiff('day', start_date, end_date) AS total_days,
    -- Full weeks * 5 weekdays each
    intDiv(dateDiff('day', toMonday(start_date), toMonday(end_date)), 7) * 5
    -- Add weekdays in the partial week at the end
    + least(toDayOfWeek(end_date), 6) - 1
    -- Subtract weekdays in the partial week at the start
    - (least(toDayOfWeek(start_date), 6) - 1) AS business_days;
```

```text
start_date  end_date    total_days  business_days
2024-06-10  2024-06-20  10          8
```

June 10 (Mon) to June 20 (Thu) spans two full weeks minus two weekend days, giving 8 business days.

## Wrapping Into a Reusable Expression

For readability, define the formula as a named expression in a WITH clause.

```sql
-- Reusable business day calculation with WITH clause
WITH
    toDate('2024-06-03') AS start_date,
    toDate('2024-06-28') AS end_date,
    intDiv(dateDiff('day', toMonday(start_date), toMonday(end_date)), 7) * 5
        + least(toDayOfWeek(end_date),   6) - 1
        - (least(toDayOfWeek(start_date), 6) - 1) AS biz_days
SELECT
    start_date,
    end_date,
    biz_days;
```

## SLA Breach Detection

For a support ticket SLA of 3 business days, flag tickets where the business day count from creation to now exceeds the threshold.

```sql
-- Identify SLA breaches where more than 3 business days have elapsed
SELECT
    ticket_id,
    created_at,
    status,
    intDiv(dateDiff('day', toMonday(toDate(created_at)), toMonday(today())), 7) * 5
        + least(toDayOfWeek(today()), 6) - 1
        - (least(toDayOfWeek(toDate(created_at)), 6) - 1) AS business_days_open,
    business_days_open > 3 AS sla_breached
FROM support_tickets
WHERE status != 'closed'
ORDER BY business_days_open DESC;
```

## Computing a Business Day Due Date

To find the date that is N business days after a starting date, generate a sequence of future dates and filter weekends out.

```sql
-- Find the date that is 5 business days after each order date
SELECT
    order_id,
    order_date,
    -- Candidate dates: order_date + 1 through order_date + 14
    -- Take the 5th non-weekend date
    arrayElement(
        arrayFilter(
            d -> toDayOfWeek(d) NOT IN (6, 7),
            arrayMap(n -> order_date + toIntervalDay(n), range(1, 15))
        ),
        5
    ) AS due_date
FROM orders
WHERE order_date >= today() - 30;
```

## Counting Business Days Using a Numbers Table

For ranges where the above formula feels complex, an alternative is to enumerate all days and filter.

```sql
-- Count weekdays by enumerating days and filtering weekends
SELECT
    countIf(toDayOfWeek(toDate('2024-06-10') + toIntervalDay(number)) NOT IN (6, 7)) AS business_days
FROM numbers(dateDiff('day', toDate('2024-06-10'), toDate('2024-06-20')));
```

This approach is more intuitive but less efficient for very large ranges.

## Verifying Edge Cases

Test the formula against known dates to verify corner cases such as start on a weekend, end on a weekend, and multi-week spans.

```sql
-- Verify edge cases
SELECT
    'Mon to Fri (same week)' AS scenario,
    intDiv(dateDiff('day', toMonday(toDate('2024-06-10')), toMonday(toDate('2024-06-14'))), 7) * 5
        + least(toDayOfWeek(toDate('2024-06-14')), 6) - 1
        - (least(toDayOfWeek(toDate('2024-06-10')), 6) - 1) AS biz_days
UNION ALL SELECT
    'Fri to Mon (skip weekend)',
    intDiv(dateDiff('day', toMonday(toDate('2024-06-14')), toMonday(toDate('2024-06-17'))), 7) * 5
        + least(toDayOfWeek(toDate('2024-06-17')), 6) - 1
        - (least(toDayOfWeek(toDate('2024-06-14')), 6) - 1);
```

```text
scenario                    biz_days
Mon to Fri (same week)      4
Fri to Mon (skip weekend)   1
```

## Summary

ClickHouse requires manual weekday counting because there is no built-in business day function. The canonical approach uses `toDayOfWeek`, `toMonday`, and `dateDiff` to count complete weeks (5 weekdays each) and handle partial-week remainders. For SLA calculations, wrap the formula in a `WITH` clause or a subquery. For computing future business-day due dates, enumerate candidate dates with `range()` and filter out weekends. Always test edge cases involving start or end dates that fall on weekends.
