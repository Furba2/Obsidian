# Basic Math and Stats with SQL — Explained Simply

This chapter is about **doing math on your data**. You'll learn everything from simple addition to finding averages, medians, and modes — the building blocks of data analysis.

---

## 🎯 The Big Picture: Math Operations in SQL

```mermaid
flowchart TD
    A[SQL Math] --> B[Basic Operators]
    A --> C[Math Functions]
    A --> D[Aggregate Functions]
    B --> E[+ - * / % ^ etc.]
    C --> F[sqrt, factorial, round]
    D --> G[sum, avg, median, mode]
```

---

## 1️⃣ Basic Math Operators

```mermaid
mindmap
  root((Math Operators))
    Standard ANSI
      Addition +
      Subtraction -
      Multiplication *
      Division /
    PostgreSQL-Specific
      Modulo %
      Exponentiation ^
      Square root |/
      Cube root ||/
      Factorial !
```

### Simple Examples (Listing 6-1)

```sql
SELECT 2 + 2;    -- Returns 4
SELECT 9 - 1;    -- Returns 8
SELECT 3 * 4;    -- Returns 12
```

> 💡 You can do math **without a table** — just `SELECT` the expression.

**Output column name is `?column?`** unless you use an alias:
```sql
SELECT 3 * 4 AS result;  -- Column named "result"
```

---

## 🔢 Division and Modulo — The Tricky Part

```mermaid
flowchart TD
    A[Division 11 / 6] --> B{Data types?}
    B -->|Both integers| C[Returns 1 — integer quotient]
    B -->|One is numeric| D[Returns 1.83333 — decimal]
```

### Four Examples (Listing 6-2)

```sql
SELECT 11 / 6;                          -- Returns 1 (integer division)
SELECT 11 % 6;                          -- Returns 5 (remainder)
SELECT 11.0 / 6;                        -- Returns 1.83333 (decimal)
SELECT CAST(11 AS numeric(3,1)) / 6;    -- Returns 1.83333 (forced decimal)
```

| Expression | Result | Why |
|------------|--------|-----|
| `11 / 6` | `1` | Integer ÷ integer = integer |
| `11 % 6` | `5` | Modulo returns remainder |
| `11.0 / 6` | `1.83333` | One decimal = decimal result |
| `CAST(11 AS numeric(3,1)) / 6` | `1.83333` | Forced conversion |

> 💡 **Modulo trick:** `number % 2 = 0` means the number is even.

---

## 🧮 Exponents, Roots, and Factorials

```sql
SELECT 3 ^ 4;           -- 81 (3 to the 4th power)
SELECT |/ 10;           -- Square root of 10
SELECT sqrt(10);        -- Same as above
SELECT ||/ 10;          -- Cube root of 10
SELECT factorial(4);    -- 24 (4 × 3 × 2 × 1)
SELECT 4 !;             -- Same as factorial(4), PostgreSQL ≤ 13 only
```

```mermaid
flowchart LR
    A[3 ^ 4] --> B[81]
    C[|/ 10] --> D[√10 ≈ 3.162]
    E[||/ 10] --> F[∛10 ≈ 2.154]
    G[factorial 4] --> H[24]
```

> ⚠️ **The `!` operator is removed in PostgreSQL 14+.** Use `factorial()` instead.

---

## 📐 Order of Operations (Precedence)

```mermaid
flowchart TD
    A[1. Exponents & Roots] --> B[2. Multiplication, Division, Modulo]
    B --> C[3. Addition & Subtraction]
```

### Examples

```sql
SELECT 7 + 8 * 9;       -- 79 (multiply first)
SELECT (7 + 8) * 9;     -- 135 (parentheses first)
SELECT 3 ^ 3 - 1;       -- 26 (exponent first)
SELECT 3 ^ (3 - 1);     -- 9 (parentheses first)
```

> ⚠️ **Always use parentheses** when you want a different order.

---

## 📊 Math Across Table Columns

### Selecting Census Data (Listing 6-4)

```sql
SELECT county_name AS county,
       state_name AS state,
       pop_est_2019 AS pop,
       births_2019 AS births,
       deaths_2019 AS deaths
FROM us_counties_pop_est_2019;
```

> 💡 **`AS` creates aliases** — shorter, more readable column names.

### Subtracting Columns (Listing 6-5)

```sql
SELECT county_name AS county,
       state_name AS state,
       births_2019 AS births,
       deaths_2019 AS deaths,
       births_2019 - deaths_2019 AS natural_increase
FROM us_counties_pop_est_2019
ORDER BY state_name, county_name;
```

```mermaid
flowchart LR
    A[births_2019] --> C[Subtract]
    B[deaths_2019] --> C
    C --> D[natural_increase]
```

**Output:**
| county | state | births | deaths | natural_increase |
|--------|-------|--------|--------|------------------|
| Autauga County | Alabama | 624 | 541 | 83 |
| Baldwin County | Alabama | 2304 | 2326 | -22 |
| Barbour County | Alabama | 256 | 312 | -56 |

### Validating Data with Column Math (Listing 6-6)

```sql
SELECT county_name AS county,
       state_name AS state,
       pop_est_2019 AS pop,
       pop_est_2018 + births_2019 - deaths_2019 +
       international_migr_2019 + domestic_migr_2019 +
       residual_2019 AS components_total,
       pop_est_2019 - (pop_est_2018 + births_2019 - deaths_2019 +
       international_migr_2019 + domestic_migr_2019 +
       residual_2019) AS difference
FROM us_counties_pop_est_2019
ORDER BY difference DESC;
```

```mermaid
flowchart TD
    A[pop_est_2019] --> C{Should equal?}
    B[pop_est_2018 + births - deaths + migration] --> C
    C -->|Yes| D[difference = 0 ✅]
    C -->|No| E[difference ≠ 0 ❌]
```

> 💡 **Great data quality check:** If `difference` is always 0, your import is clean.

---

## 📈 Percentages and Percent Change

### Percent of the Whole (Listing 6-7)

```sql
SELECT county_name AS county,
       state_name AS state,
       area_water::numeric / (area_land + area_water) * 100 AS pct_water
FROM us_counties_pop_est_2019
ORDER BY pct_water DESC;
```

```mermaid
flowchart LR
    A[area_water] --> C[Divide by total area]
    B[area_land + area_water] --> C
    C --> D[Multiply by 100]
    D --> E[Percentage of water]
```

**Output (top 5):**
| county | state | pct_water |
|--------|-------|-----------|
| Keweenaw County | Michigan | 90.95% |
| Leelanau County | Michigan | 86.29% |
| Nantucket County | Massachusetts | 84.80% |
| St. Bernard Parish | Louisiana | 82.48% |
| Alger County | Michigan | 81.87% |

> 💡 **Casting to `numeric`** is required — otherwise integer division returns 0.

### Percent Change (Listing 6-8)

**Formula:**
```
(new number – old number) / old number
```

```sql
CREATE TABLE percent_change (
    department text,
    spend_2019 numeric(10,2),
    spend_2022 numeric(10,2)
);
INSERT INTO percent_change
VALUES
    ('Assessor', 178556, 179500),
    ('Building', 250000, 289000),
    ('Clerk', 451980, 650000),
    ('Library', 87777, 90001),
    ('Parks', 250000, 223000),
    ('Water', 199000, 195000);

SELECT department,
       spend_2019,
       spend_2022,
       round((spend_2022 - spend_2019) / spend_2019 * 100, 1) AS pct_change
FROM percent_change;
```

```mermaid
flowchart LR
    A[spend_2022] --> C[Subtract]
    B[spend_2019] --> C
    C --> D[Divide by spend_2019]
    D --> E[Multiply by 100]
    E --> F[Round to 1 decimal]
```

**Output:**
| department | spend_2019 | spend_2022 | pct_change |
|------------|------------|------------|------------|
| Assessor | 178556.00 | 179500.00 | 0.5 |
| Building | 250000.00 | 289000.00 | 15.6 |
| Clerk | 451980.00 | 650000.00 | 43.8 |
| Library | 87777.00 | 90001.00 | 2.5 |
| Parks | 250000.00 | 223000.00 | -10.8 |
| Water | 199000.00 | 195000.00 | -2.0 |

---

## 📊 Aggregate Functions

```mermaid
mindmap
  root((Aggregate Functions))
    Sum
      sum column
    Average
      avg column
    Median
      percentile_cont .5
    Mode
      mode
    Other Quantiles
      percentile_cont array
```

### Sum and Average (Listing 6-9)

```sql
SELECT sum(pop_est_2019) AS county_sum,
       round(avg(pop_est_2019), 0) AS county_average
FROM us_counties_pop_est_2019;
```

**Output:**
| county_sum | county_average |
|------------|----------------|
| 328239523 | 104468 |

### Median vs. Average — Why It Matters

```mermaid
flowchart TD
    A[Average] --> B[Sensitive to outliers]
    C[Median] --> D[Resistant to outliers]
    B --> E[Can mislead]
    D --> F[Better for skewed data]
```

**Example:** Ages 10, 11, 10, 9, 13, 12, 46
- **Average:** 15.9 (skewed by 46)
- **Median:** 11 (better representation)

### Finding the Median (Listing 6-10)

```sql
CREATE TABLE percentile_test (
    numbers integer
);
INSERT INTO percentile_test (numbers) VALUES
(1), (2), (3), (4), (5), (6);

SELECT
    percentile_cont(.5) WITHIN GROUP (ORDER BY numbers),
    percentile_disc(.5) WITHIN GROUP (ORDER BY numbers)
FROM percentile_test;
```

| Function | Result | Meaning |
|----------|--------|---------|
| `percentile_cont(.5)` | 3.5 | Continuous — averages middle two |
| `percentile_disc(.5)` | 3 | Discrete — picks one value |

> 💡 **Use `percentile_cont(.5)` for median** — it follows the standard method.

### Median with Census Data (Listing 6-11)

```sql
SELECT sum(pop_est_2019) AS county_sum,
       round(avg(pop_est_2019), 0) AS county_average,
       percentile_cont(.5) WITHIN GROUP (ORDER BY pop_est_2019) AS county_median
FROM us_counties_pop_est_2019;
```

**Output:**
| county_sum | county_avg | county_median |
|------------|------------|---------------|
| 328239523 | 104468 | 25726 |

> ⚠️ **Huge gap!** Average (104,468) vs Median (25,726). A few huge counties (like LA) skew the average.

### Finding Quartiles with Arrays (Listing 6-12)

```sql
SELECT percentile_cont(ARRAY[.25,.5,.75])
       WITHIN GROUP (ORDER BY pop_est_2019) AS quartiles
FROM us_counties_pop_est_2019;
```

**Output:**
```
{10902.5,25726,68072.75}
```

```mermaid
flowchart LR
    A[Q1: 10,902.5] --> B[Q2: 25,726]
    B --> C[Q3: 68,072.75]
    A -->|25% below| D[Smallest counties]
    C -->|25% above| E[Largest counties]
```

### Using `unnest()` to Turn Array into Rows (Listing 6-13)

```sql
SELECT unnest(
    percentile_cont(ARRAY[.25,.5,.75])
    WITHIN GROUP (ORDER BY pop_est_2019)
) AS quartiles
FROM us_counties_pop_est_2019;
```

**Output:**
| quartiles |
|-----------|
| 10902.5 |
| 25726 |
| 68072.75 |

> 💡 **`unnest()` makes arrays readable** as rows.

### Finding the Mode (Listing 6-14)

```sql
SELECT mode() WITHIN GROUP (ORDER BY births_2019)
FROM us_counties_pop_est_2019;
```

**Output:** `86` — 16 counties had exactly 86 births.

```mermaid
flowchart TD
    A[Mode] --> B[Most frequent value]
    B --> C[86 births]
    C --> D[Shared by 16 counties]
```

---

## 🧩 Complete Stats Workflow

```mermaid
sequenceDiagram
    participant You
    participant PostgreSQL

    You->>PostgreSQL: SELECT sum(col) FROM table;
    PostgreSQL-->>You: Total sum

    You->>PostgreSQL: SELECT avg(col) FROM table;
    PostgreSQL-->>You: Average

    You->>PostgreSQL: SELECT percentile_cont(.5) WITHIN GROUP (ORDER BY col);
    PostgreSQL-->>You: Median

    You->>PostgreSQL: SELECT mode() WITHIN GROUP (ORDER BY col);
    PostgreSQL-->>You: Mode

    You->>PostgreSQL: SELECT percentile_cont(ARRAY[.25,.5,.75]) ...;
    PostgreSQL-->>You: Quartiles array
```

---

## ✅ Chapter 6 Checklist

| Task | SQL |
|------|-----|
| Add/subtract/multiply | `SELECT a + b;` |
| Divide (decimal) | `SELECT a::numeric / b;` |
| Remainder | `SELECT a % b;` |
| Exponent | `SELECT a ^ b;` |
| Square root | `SELECT |/ a;` |
| Factorial | `SELECT factorial(a);` |
| Column math | `SELECT col1 - col2 FROM table;` |
| Percent | `SELECT (part / total) * 100 FROM table;` |
| Percent change | `SELECT (new - old) / old * 100 FROM table;` |
| Sum | `SELECT sum(col) FROM table;` |
| Average | `SELECT avg(col) FROM table;` |
| Median | `SELECT percentile_cont(.5) WITHIN GROUP (ORDER BY col) FROM table;` |
| Mode | `SELECT mode() WITHIN GROUP (ORDER BY col) FROM table;` |
| Quartiles | `SELECT percentile_cont(ARRAY[.25,.5,.75]) ...;` |

---

## 🎯 Key Takeaways

1. **Integer division truncates** — cast to `numeric` for decimals
2. **Modulo (`%`) returns remainder** — useful for even/odd checks
3. **SQL follows math order of operations** — use parentheses to control
4. **Aliases (`AS`) make results readable**
5. **Column math validates data** — if `difference` is 0, import is clean
6. **Average vs. Median** — median resists outliers
7. **`percentile_cont(.5)` = median** — no built-in `median()` function
8. **Arrays let you get multiple percentiles at once**
9. **`unnest()` turns arrays into rows** for readability
10. **`mode()` finds the most frequent value**

> In **Chapter 7**, you'll learn about **joins** — combining data from multiple tables to answer richer questions. 🚀