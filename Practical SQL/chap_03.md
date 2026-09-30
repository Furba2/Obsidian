```sql
SELECT * FROM teachers;
```

| `SELECT` | "select data" |
|------|---------|
| `*` | **all columns** |
| `FROM teachers` | From `teachers` table |
| `;` | End statement |

### Three Ways to See All Rows
```mermaid
flowchart LR
    A[SELECT * FROM teachers] --> D[Same Result]
    B[TABLE teachers;] --> D
```

---

```sql
SELECT last_name, first_name, salary FROM teachers;
```

---

```sql
SELECT first_name, last_name, salary
FROM teachers
ORDER BY salary DESC;
```

---
### Sort  Multiple Columns

```sql
SELECT last_name, school, hire_date
FROM teachers
ORDER BY school ASC, hire_date DESC;
```

> can also use column **numbers** (`ORDER BY 3 DESC`).

---

## Using DISTINCT to Find Unique Values

```sql
SELECT DISTINCT school FROM teachers ORDER BY school;
```

```mermaid
flowchart TD
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
flowchart LR
    A[DISTINCT school, salary] --> B[Each unique pair]
    B --> C[F.D. Roosevelt HS: 36200]
    B --> D[F.D. Roosevelt HS: 38500]
    B --> E[F.D. Roosevelt HS: 65000]
    B --> F[Myers Middle School: 36200]
    B --> G[Myers Middle School: 43500]
```

---

## Filter Rows with WHERE

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

| Symbol | Meaning |
|--------|---------|
| `%` | Match **one or more** characters |
| `_` | Match **exactly one** character |

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

---
## Combining Operators with AND / OR

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
flowchart LR
    A[AND] --> B[Both must be true]
    C[OR] --> D[At least one true]
    E[Parentheses] --> F[Evaluate group first]
```

> ⚠️ **Order of evaluation:** Without parentheses, SQL evaluates **AND before OR**. Use parentheses to control logic.

---

```sql
SELECT first_name, last_name, school, hire_date, salary
FROM teachers
WHERE school LIKE '%Roos%'
ORDER BY hire_date DESC;
```

```mermaid
flowchart TD
    A[SELECT columns] --> B[FROM teachers]
    B --> C[WHERE school LIKE '%Roos%']
    C --> D[ORDER BY hire_date DESC]
    D --> E[Result: Roosevelt teachers, newest first]
```

**Output:**

| first_name | last_name | school |hire_date | salary |
|------------|-----------|--------|-----------|--------|
| Janet | Smith | F.D. Roosevelt HS | 2011-10-30 | 36200 |
| Kathleen | Roush | F.D. Roosevelt HS | 2010-10-22 | 38500 |
| Lee | Reynolds | F.D. Roosevelt HS | 1993-05-22 | 65000 |

---
