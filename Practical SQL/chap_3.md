# Beginning Data Exploration with SELECT — Explained Simply

This chapter is about **interviewing your data**. Think of it like asking questions to a job candidate — you want to find out if the data is clean, complete, and what story it tells.

---

## 🔍 The Big Picture: What SELECT Does

```mermaid
flowchart LR
    A[🗄️ Database] --> B[📊 Table]
    B --> C[SELECT query]
    C --> D[📋 Result Set: rows & columns]
```

- **SELECT** = The SQL keyword that retrieves data
- **Result Set** = The virtual table returned by your query
- You can filter, sort, and limit what you see — without changing the original table

---

## 1️⃣ Basic SELECT Syntax

```sql
SELECT * FROM teachers;
```

```mermaid
flowchart TD
    A[SELECT] --> B[* = all columns]
    B --> C[FROM]
    C --> D[teachers = table name]
    D --> E[; = end of statement]
```

| Part | Meaning |
|------|---------|
| `SELECT` | "I want to retrieve data" |
| `*` | Wildcard = **all columns** |
| `FROM teachers` | From the `teachers` table |
| `;` | End of statement |

### Three Ways to See All Rows
```mermaid
flowchart LR
    A[SELECT * FROM teachers] --> D[Same Result]
    B[TABLE teachers;] --> D
    C[pgAdmin: Right-click → View/Edit Data → All Rows] --> D
```

---

## 2️⃣ Querying a Subset of Columns

Instead of `*`, name the columns you want:

```sql
SELECT last_name, first_name, salary FROM teachers;
```

```mermaid
flowchart LR
    A[Full Table: 6 columns] --> B[Pick 3 columns]
    B --> C[Result: last_name, first_name, salary]
```

> 💡 **Column order in query ≠ column order in table.** You can rearrange them however you like.

---

## 3️⃣ Sorting Data with ORDER BY

```sql
SELECT first_name, last_name, salary
FROM teachers
ORDER BY salary DESC;
```

```mermaid
flowchart TD
    A[ORDER BY salary DESC] --> B[Highest salary first]
    B --> C[Lee Reynolds: 65000]
    C --> D[Samuel Cole: 43500]
    D --> E[Betty Diaz: 43500]
    E --> F[Kathleen Roush: 38500]
    F --> G[Janet Smith: 36200]
    G --> H[Samantha Bush: 36200]
```

### Sort Directions
| Keyword | Meaning |
|---------|---------|
| `ASC` | Ascending (A→Z, 1→9) — **default** |
| `DESC` | Descending (Z→A, 9→1) |

### Sort by Multiple Columns

```sql
SELECT last_name, school, hire_date
FROM teachers
ORDER BY school ASC, hire_date DESC;
```

```mermaid
flowchart TD
    A[ORDER BY school ASC, hire_date DESC] --> B[Group by school]
    B --> C[Within each school, newest hires first]
    C --> D[F.D. Roosevelt HS: Smith, Roush, Reynolds]
    C --> E[Myers Middle School: Bush, Diaz, Cole]
```

> 💡 You can also use column **numbers** (e.g., `ORDER BY 3 DESC`) instead of names.

---

## 4️⃣ Using DISTINCT to Find Unique Values

```sql
SELECT DISTINCT school FROM teachers ORDER BY school;
```

```mermaid
flowchart LR
    A[6 rows in table] --> B[DISTINCT school]
    B --> C[Only 2 unique schools]
    C --> D[F.D. Roosevelt HS]
    C --> E[Myers Middle School]
```

### DISTINCT on Multiple Columns

```sql
SELECT DISTINCT school, salary FROM teachers ORDER BY school, salary;
```

```mermaid
flowchart TD
    A[DISTINCT school, salary] --> B[Each unique pair]
    B --> C[F.D. Roosevelt HS: 36200]
    B --> D[F.D. Roosevelt HS: 38500]
    B --> E[F.D. Roosevelt HS: 65000]
    B --> F[Myers Middle School: 36200]
    B --> G[Myers Middle School: 43500]
```

> 💡 **Use case:** "For each X, what are all the Y values?" — e.g., for each school, what salaries exist?

---

## 5️⃣ Filtering Rows with WHERE

```sql
SELECT last_name, school, hire_date
FROM teachers
WHERE school = 'Myers Middle School';
```

```mermaid
flowchart TD
    A[All 6 rows] --> B[WHERE school = 'Myers Middle School']
    B --> C[Only 3 rows match]
    C --> D[Cole, Bush, Diaz]
```

### Comparison Operators (Table 3-1)

```mermaid
mindmap
  root((WHERE Operators))
    Comparison
      = Equal to
      <> or != Not equal
      > Greater than
      < Less than
      >= Greater or equal
      <= Less or equal
    Range
      BETWEEN inclusive range
    Set
      IN match any in list
    Pattern
      LIKE case-sensitive
      ILIKE case-insensitive
    Logic
      NOT negates condition
```

### Examples

| Query | What It Finds |
|-------|---------------|
| `WHERE first_name = 'Janet'` | Teachers named Janet |
| `WHERE school <> 'F.D. Roosevelt HS'` | Everyone except Roosevelt |
| `WHERE hire_date < '2000-01-01'` | Hired before 2000 |
| `WHERE salary >= 43500` | Earning $43,500 or more |
| `WHERE salary BETWEEN 40000 AND 65000` | Earning $40k–$65k (inclusive) |

> ⚠️ **Beware of BETWEEN:** It's inclusive. `BETWEEN 10 AND 20` and `BETWEEN 20 AND 30` both include 20 → double-counting.

**Safer alternative:**
```sql
WHERE salary >= 40000 AND salary <= 65000
```

---

## 6️⃣ LIKE and ILIKE — Pattern Matching

| Symbol | Meaning |
|--------|---------|
| `%` | Matches **one or more** characters |
| `_` | Matches **exactly one** character |

### Pattern Examples for "baker"
```mermaid
flowchart LR
    A[baker] --> B[LIKE 'b%']
    A --> C[LIKE '%ak%']
    A --> D[LIKE '_aker']
    A --> E[LIKE 'ba_er']
```

### LIKE vs ILIKE

```sql
-- Case-sensitive (ANSI standard)
SELECT first_name FROM teachers WHERE first_name LIKE 'sam%';
-- Returns: 0 rows (because 'Samuel' has capital S)

-- Case-insensitive (PostgreSQL-only)
SELECT first_name FROM teachers WHERE first_name ILIKE 'sam%';
-- Returns: Samuel, Samantha
```

```mermaid
flowchart TD
    A[LIKE 'sam%'] --> B{Case sensitive?}
    B -->|Yes| C[❌ 0 results]
    A2[ILIKE 'sam%'] --> D{Case sensitive?}
    D -->|No| E[✅ Samuel, Samantha]
```

> 💡 **Tip:** Use `ILIKE` when vetting data — you don't know if someone capitalized names correctly.

---

## 7️⃣ Combining Operators with AND / OR

```sql
-- AND: both conditions must be true
SELECT * FROM teachers
WHERE school = 'Myers Middle School' AND salary < 40000;

-- OR: at least one condition must be true
SELECT * FROM teachers
WHERE last_name = 'Cole' OR last_name = 'Bush';

-- Parentheses: group conditions
SELECT * FROM teachers
WHERE school = 'F.D. Roosevelt HS'
AND (salary < 38000 OR salary > 40000);
```

```mermaid
flowchart TD
    A[AND] --> B[Both must be true]
    C[OR] --> D[At least one true]
    E[Parentheses] --> F[Evaluate group first]
```

> ⚠️ **Order of evaluation:** Without parentheses, SQL evaluates **AND before OR**. Use parentheses to control logic.

---

## 🧩 Putting It All Together — The Full SELECT Syntax

```mermaid
flowchart TD
    A[SELECT column_names] --> B[FROM table_name]
    B --> C[WHERE criteria]
    C --> D[ORDER BY column_names]
```

### Complete Example (Listing 3-10)

```sql
SELECT first_name, last_name, school, hire_date, salary
FROM teachers
WHERE school LIKE '%Roos%'
ORDER BY hire_date DESC;
```

```mermaid
flowchart LR
    A[SELECT columns] --> B[FROM teachers]
    B --> C[WHERE school LIKE '%Roos%']
    C --> D[ORDER BY hire_date DESC]
    D --> E[Result: Roosevelt teachers, newest first]
```

**Output:**
| first_name | last_name | school | hire_date | salary |
|------------|-----------|--------|-----------|--------|
| Janet | Smith | F.D. Roosevelt HS | 2011-10-30 | 36200 |
| Kathleen | Roush | F.D. Roosevelt HS | 2010-10-22 | 38500 |
| Lee | Reynolds | F.D. Roosevelt HS | 1993-05-22 | 65000 |

---

## 🎯 Complete Query Workflow

```mermaid
sequenceDiagram
    participant You
    participant PostgreSQL

    You->>PostgreSQL: SELECT * FROM teachers;
    PostgreSQL-->>You: All rows & columns

    You->>PostgreSQL: SELECT last_name, salary FROM teachers;
    PostgreSQL-->>You: Subset of columns

    You->>PostgreSQL: SELECT ... ORDER BY salary DESC;
    PostgreSQL-->>You: Sorted results

    You->>PostgreSQL: SELECT DISTINCT school FROM teachers;
    PostgreSQL-->>You: Unique values only

    You->>PostgreSQL: SELECT ... WHERE school = 'Myers';
    PostgreSQL-->>You: Filtered rows

    You->>PostgreSQL: SELECT ... WHERE ... AND ... ORDER BY ...;
    PostgreSQL-->>You: Fully refined result
```

---

## ✅ Chapter 3 Checklist

| Concept | SQL Keyword | Purpose |
|---------|-------------|---------|
| Select all | `SELECT *` | Get every column |
| Select specific | `SELECT col1, col2` | Get chosen columns |
| Sort | `ORDER BY` | Arrange results (ASC/DESC) |
| Unique values | `DISTINCT` | Remove duplicates |
| Filter rows | `WHERE` | Match criteria |
| Pattern match | `LIKE` / `ILIKE` | Search with wildcards |
| Combine filters | `AND` / `OR` | Multiple conditions |

---

## 🧠 Key Takeaways

1. **SELECT is your interview tool** — ask questions to understand data quality
2. **Start broad, then narrow** — `SELECT *` first, then add `WHERE`, `ORDER BY`
3. **DISTINCT reveals data quality** — spot spelling variations, inconsistent formats
4. **LIKE/ILIKE find patterns** — perfect for rooting out misspellings
5. **AND/OR with parentheses** — control exactly which rows come back
6. **SQL has a required order:** `SELECT → FROM → WHERE → ORDER BY`

> In **Chapter 4**, you'll go deeper into **data types** — understanding how numbers, text, and dates are stored and manipulated. 🚀