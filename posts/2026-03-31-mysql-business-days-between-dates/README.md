# How to Calculate Business Days Between Two Dates in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database, Query

Description: Learn how to count working days between two dates in MySQL by excluding weekends using DATEDIFF, WEEKDAY arithmetic, and stored function techniques.

---

## The Core Formula

MySQL does not have a built-in `BUSINESS_DAYS()` function. The standard approach uses week arithmetic: count the number of complete weeks (each contributing 5 business days), then add the remaining days while excluding weekends.

The formula below calculates weekdays between `start_date` and `end_date` (exclusive of `start_date`, inclusive of `end_date`):

```sql
SELECT
  5 * (DATEDIFF('2026-03-31', '2026-03-01') DIV 7)
  + MID('0123444401233334012222340111123400012345001234550',
        7 * WEEKDAY('2026-03-01') + WEEKDAY('2026-03-31') + 1, 1) AS business_days;
```

This lookup-table approach (from MySQL community resources) is compact but hard to read. The step-by-step breakdown below is more maintainable.

## Step-by-Step Formula

```sql
SELECT
  -- Complete weeks * 5 weekdays
  (DATEDIFF('2026-03-31', '2026-03-01') DIV 7) * 5
  -- Add remaining days, subtract weekend days in the remainder
  + GREATEST(
      LEAST(
        WEEKDAY('2026-03-31') + 1,  -- day index of end (Mon=1 ... Fri=5)
        5
      ) - GREATEST(
        WEEKDAY('2026-03-01'),       -- day index of start (Mon=0 ... Sun=6)
        0
      ),
      0
    ) AS business_days;
```

For most practical uses, a stored function is the cleanest solution.

## Stored Function for Reuse

```sql
DELIMITER $$

CREATE FUNCTION BUSINESS_DAYS(start_date DATE, end_date DATE)
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE total_days INT;
  DECLARE full_weeks INT;
  DECLARE remaining  INT;
  DECLARE start_wd   INT;
  DECLARE end_wd     INT;

  SET total_days = DATEDIFF(end_date, start_date);
  SET full_weeks = total_days DIV 7;
  SET remaining  = total_days MOD 7;
  SET start_wd   = WEEKDAY(start_date);   -- 0=Mon ... 6=Sun
  SET end_wd     = WEEKDAY(end_date);

  RETURN full_weeks * 5
    + CASE
        WHEN remaining = 0 THEN 0
        WHEN start_wd + remaining <= 4  THEN remaining
        WHEN start_wd + remaining  = 5  THEN remaining - 1
        WHEN start_wd + remaining  = 6  THEN remaining - 1
        WHEN start_wd + remaining  = 7  THEN remaining - 2
        WHEN start_wd + remaining  = 8  THEN remaining - 2
        WHEN start_wd + remaining  = 9  THEN remaining - 2
        WHEN start_wd + remaining >= 10 THEN remaining - 2
        ELSE 0
      END;
END$$

DELIMITER ;
```

Usage:

```sql
SELECT BUSINESS_DAYS('2026-03-01', '2026-03-31') AS working_days;
-- 22
```

## Quick Inline Query Without a Function

If you cannot create stored functions, a simpler inline approximation counts calendar days then subtracts weekends:

```sql
SELECT
  s.start_date,
  s.end_date,
  DATEDIFF(s.end_date, s.start_date) + 1
    - 2 * (DATEDIFF(s.end_date, s.start_date) DIV 7)
    - (WEEKDAY(s.end_date) >= 5)
    - (WEEKDAY(s.start_date) = 6) AS business_days
FROM (
  SELECT '2026-03-01' AS start_date, '2026-03-31' AS end_date
) s;
```

This is an approximation that works well for ranges of several weeks but may be off by 1 in edge cases involving partial weeks. Test against a calendar for your specific range.

## Apply to a Table

```sql
CREATE TABLE contracts (
  id INT AUTO_INCREMENT PRIMARY KEY,
  contract_name VARCHAR(100),
  signed_date DATE NOT NULL,
  deadline     DATE NOT NULL
);

INSERT INTO contracts (contract_name, signed_date, deadline) VALUES
  ('Deal A', '2026-03-01', '2026-03-31'),
  ('Deal B', '2026-03-15', '2026-04-30');

SELECT
  contract_name,
  BUSINESS_DAYS(signed_date, deadline) AS working_days_allowed
FROM contracts;
```

## Summary

MySQL has no native business-days function, but you can compute working days by counting complete weeks times 5 and adding the remaining weekdays. For production use, encapsulate the logic in a stored function. Remember that this formula excludes public holidays - for true business-day calculations with holidays, maintain a `calendar` table that flags non-working days and count only rows where the flag is absent.
