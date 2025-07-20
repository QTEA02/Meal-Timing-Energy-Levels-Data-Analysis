````markdown
# 1. Project Overview & Timeline

**Objective:**  
Discover how the clock-time of your meals affects your self-rated energy (1–10) afterward, and how skipping meals factors in.

**Duration:**  
14 days (two full weeks to capture both weekday and weekend patterns).

**Daily Routine:**  
- Log each eating event (or planned event you skipped) in Excel as it happens.  
- At a consistent time (~1 hr post-meal), record your **EnergyRating**.

---

# 2. Excel Logging

Create a simple workbook—one sheet, these columns:

| Column         | Example     | Notes                                                      |
| -------------- | ----------- | ---------------------------------------------------------- |
| **Date**       | 2025-07-21  | The calendar date of the meal.                             |
| **MealTime**   | 08:15:00    | Clock time you ate, in `HH:MM:SS` or `HH:MM` format.        |
| **EnergyRating** | 7         | How energized you feel 1 hr after eating (1–10).           |
| **SkipFlag**   | 0 or 1      | `1` if you skipped this scheduled meal; otherwise `0`.     |

> **Tip:** Pre-fill rows for your usual meals (e.g. breakfast at 8 AM, lunch at 12 PM, dinner at 6 PM).  
> For any you miss, mark `SkipFlag = 1` and leave `EnergyRating` blank or zero.

**Save As:**  
After Day 7 and Day 14, export this sheet as CSV for your SQL step.

---

# 3. SQL: Import, Clean & Transform

Use PostgreSQL, MySQL, or SQLite—whichever you prefer.

## a. Create Database & Table

```sql
-- In psql or your SQL client:
CREATE DATABASE meal_energy;
\c meal_energy

CREATE TABLE meals (
  id             SERIAL PRIMARY KEY,
  log_date       DATE,
  meal_time      TIME,
  energy_rating  INTEGER,
  skip_flag      BOOLEAN
);
````

## b. Load Your CSV

```sql
-- Adjust path as needed:
\copy meals(log_date, meal_time, energy_rating, skip_flag)
  FROM '/path/to/meal_log.csv' WITH CSV HEADER;
```

## c. Add Derived Columns

```sql
-- Extract hour-of-day for grouping:
ALTER TABLE meals ADD COLUMN meal_hour INTEGER;
UPDATE meals
  SET meal_hour = EXTRACT(HOUR FROM meal_time);

-- (Optional) Flag weekend vs. weekday:
ALTER TABLE meals ADD COLUMN day_type TEXT;
UPDATE meals
  SET day_type = CASE
    WHEN EXTRACT(DOW FROM log_date) IN (0,6) THEN 'Weekend'
    ELSE 'Weekday'
  END;
```

## d. Create a View for Tableau

```sql
CREATE VIEW v_meal_energy AS
SELECT
  log_date,
  meal_hour,
  AVG(energy_rating)          AS avg_energy,
  SUM(CASE WHEN skip_flag THEN 1 ELSE 0 END) AS skips,
  COUNT(*)                    AS total_records
FROM meals
GROUP BY log_date, meal_hour;
```

---

# 4. Tableau Dashboard

Connect Tableau Desktop to your `meal_energy` database (or use the view as a custom SQL source), then build three worksheets:

## 4.1 Energy vs. Time-of-Day Scatter

* **Columns:** `meal_time` (continuous)
* **Rows:** `energy_rating`
* **Detail/Color:** `skip_flag` (to hide or gray out skipped points)
* **Tooltip:** Show `log_date` & exact `energy_rating`

## 4.2 Meal-Time Distribution Histogram

* **Columns:** `meal_hour` (binned by hour)
* **Rows:** `COUNT(meal_time)`
* **Dual-Axis (optional):** Overlaid line of `SUM(skip_flag)` by the same bins

## 4.3 Avg Energy by Hour-of-Day Line

* **Columns:** `meal_hour`
* **Rows:** `AVG(energy_rating)`
* **Color (optional):** `day_type` to split Weekday vs. Weekend

---

# 5. Assemble & Polish

**Dashboard Layout:**

* Top Row: Scatter (wide)
* Bottom Left: Histogram
* Bottom Right: Line chart

**Global Filters / Controls:**

* **SkipFlag:** show/hide skipped points
* **DayType:** weekday vs. weekend
* **Time-of-Day Slider:** restrict to morning/afternoon/evening hours

**KPI Banner (optional):**

* Overall Avg Energy
* Total Skips

**Styling:**

* Consistent fonts and color palette (e.g. Tableau 10)
* Clear titles (e.g. “Post-Meal Energy by Clock Time”) and tooltips

---

# 6. Analysis & Next Steps

* **Interpret:** Look for your “energy peak” hour and any big dips when you skip meals.
* **Extend:** If results look promising, log for 30 days or add fields (e.g. `MealType`) to refine insights.
* **Share:** Publish to Tableau Public or export as PDF/PNG to review your own habits.
