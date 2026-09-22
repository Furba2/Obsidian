# Importing and Exporting Data — Explained Simply

This chapter is about **moving data in bulk**. Instead of typing one row at a time with `INSERT`, you'll learn to load thousands (or millions) of rows from a file — and export them back out.

---

## 🎯 The Big Picture: Bulk Data Movement

```mermaid
flowchart LR
    A[📄 CSV File] -->|COPY FROM| B[🗄️ PostgreSQL Table]
    B -->|COPY TO| C[📄 CSV File]
```

- **`COPY ... FROM`** = Import data INTO a table
- **`COPY ... TO`** = Export data OUT of a table
- **Delimited text file** = The universal middle-ground format

---

## 📄 Delimited Text Files — The Universal Format

A **delimited text file** has:
- One row of data per line
- Each column separated (delimited) by a character (usually a comma)

```csv
FIRSTNAME,LASTNAME,STREET,CITY,STATE,PHONE
John,Doe,123 Main St.,Hyde Park,NY,845-555-1212
```

```mermaid
mindmap
  root((Delimited Files))
    CSV
      Comma-separated
      Most common
    TSV
      Tab-separated
    Pipe-delimited
      Uses | character
    Header Row
      Optional
      Lists column names
    Text Qualifier
      Usually double quotes
      Handles commas inside values
```

### Handling Header Rows

```mermaid
flowchart TD
    A[CSV File] --> B{Has header row?}
    B -->|Yes| C[Use HEADER option to skip it]
    B -->|No| D[Don't use HEADER]
    C --> E[Import starts at line 2]
```

### Quoting Columns with Commas

```csv
FIRSTNAME,LASTNAME,STREET,CITY,STATE,PHONE
John,Doe,"123 Main St., Apartment 200",Hyde Park,NY,845-555-1212
```

```mermaid
flowchart LR
    A["123 Main St., Apartment 200"] --> B[Double quotes wrap the value]
    B --> C[Comma inside is ignored]
    C --> D[Treated as ONE column]
```

> 💡 **Text qualifier** = the character (usually `"`) that tells the database to ignore delimiters inside.

---

## 📥 Importing Data with COPY

### The Three-Step Process

```mermaid
flowchart TD
    A[1. Obtain source CSV file] --> B[2. Create matching table]
    B --> C[3. Run COPY statement]
    C --> D[✅ Data imported]
```

### Basic COPY Syntax (Listing 5-1)

```sql
COPY table_name
FROM 'C:\YourDirectory\your_file.csv'
WITH (FORMAT CSV, HEADER);
```

```mermaid
flowchart LR
    A[COPY] --> B[table_name]
    B --> C[FROM 'file path']
    C --> D[WITH options]
```

### Common COPY Options

| Option | Purpose |
|--------|---------|
| `FORMAT CSV` | File is comma-separated |
| `HEADER` | Skip the header row (or include on export) |
| `DELIMITER '|'` | Use a different delimiter |
| `QUOTE '"'` | Specify a different text qualifier |

---

## 🏛️ Real Example: Census County Data

### The Dataset
- **3,142 rows** (every US county)
- **16 columns** (population estimates, geography, etc.)
- File: `us_counties_pop_est_2019.csv`

### Creating the Table (Listing 5-2)

```sql
CREATE TABLE us_counties_pop_est_2019 (
    state_fips text,
    county_fips text,
    region smallint,
    state_name text,
    county_name text,
    area_land bigint,
    area_water bigint,
    internal_point_lat numeric(10,7),
    internal_point_lon numeric(10,7),
    pop_est_2018 integer,
    pop_est_2019 integer,
    births_2019 integer,
    deaths_2019 integer,
    international_migr_2019 integer,
    domestic_migr_2019 integer,
    residual_2019 integer,
    CONSTRAINT counties_2019_key PRIMARY KEY (state_fips, county_fips)
);
```

### Data Type Choices Explained

```mermaid
flowchart TD
    A[Column Type Decisions] --> B[state_fips: text]
    A --> C[region: smallint]
    A --> D[area_land: bigint]
    A --> E[lat/lon: numeric 10,7]
    A --> F[pop_est: integer]
    B --> G[Leading zeros matter!]
    C --> H[Only 1-4 values]
    D --> I[Alaska is huge!]
    E --> J[7 decimal places]
    F --> K[Under 2.1 billion]
```

> 💡 **Key insight:** Codes (like FIPS) are **text**, not numbers. Alaska's `02` would become `2` if stored as integer.

### Performing the Import (Listing 5-3)

```sql
COPY us_counties_pop_est_2019
FROM 'C:\YourDirectory\us_counties_pop_est_2019.csv'
WITH (FORMAT CSV, HEADER);
```

**Success message:**
```
COPY 3142
Query returned successfully in 75 msec.
```

### Inspecting the Import

```sql
-- Check largest land areas
SELECT county_name, state_name, area_land
FROM us_counties_pop_est_2019
ORDER BY area_land DESC
LIMIT 3;
```

**Output:**
| county_name | state_name | area_land |
|-------------|------------|-----------|
| Yukon-Koyukuk Census Area | Alaska | 377038836685 |
| North Slope Borough | Alaska | 230054247231 |
| Bethel Census Area | Alaska | 105232821617 |

> 🎉 Alaska dominates because it's massive!

### The Antimeridian Surprise

```sql
SELECT county_name, state_name, internal_point_lon
FROM us_counties_pop_est_2019
ORDER BY internal_point_lon DESC
LIMIT 5;
```

**Output:**
| county_name | state_name | internal_point_lon |
|-------------|------------|---------------------|
| Aleutians West Census Area | Alaska | 179.6211882 |
| Washington County | Maine | -67.6093542 |

```mermaid
flowchart LR
    A[Longitude normally] --> B[Negative = West]
    A --> C[Positive = East]
    D[Aleutian Islands] --> E[Cross the antimeridian]
    E --> F[Longitude flips to positive]
```

> 💡 Not a data error — just geography!

---

## 🔧 Advanced Import Techniques

### 1️⃣ Importing a Subset of Columns (Listing 5-5)

**Scenario:** CSV only has 3 columns, but table has 7.

```sql
COPY supervisor_salaries (town, supervisor, salary)
FROM 'C:\YourDirectory\supervisor_salaries.csv'
WITH (FORMAT CSV, HEADER);
```

```mermaid
flowchart LR
    A[CSV: town, supervisor, salary] --> B[Map to specific columns]
    B --> C[Other columns: NULL or auto]
    C --> D[id auto-filled by IDENTITY]
```

> ⚠️ **Error if you don't specify columns:** PostgreSQL tries to fill `id` with `"Anytown"` → fails.

### 2️⃣ Importing a Subset of Rows (Listing 5-6)

```sql
COPY supervisor_salaries (town, supervisor, salary)
FROM 'C:\YourDirectory\supervisor_salaries.csv'
WITH (FORMAT CSV, HEADER)
WHERE town = 'New Brillig';
```

```mermaid
flowchart TD
    A[CSV File] --> B[WHERE town = 'New Brillig']
    B --> C[Only matching rows imported]
    C --> D[Other rows skipped]
```

> 💡 **PostgreSQL 12+** supports `WHERE` in `COPY`.

### 3️⃣ Adding a Value During Import (Listing 5-7)

**Scenario:** CSV lacks `county`, but you know it should be `'Mills'`.

```sql
-- Step 1: Create temp table
CREATE TEMPORARY TABLE supervisor_salaries_temp
(LIKE supervisor_salaries INCLUDING ALL);

-- Step 2: Import CSV into temp table
COPY supervisor_salaries_temp (town, supervisor, salary)
FROM 'C:\YourDirectory\supervisor_salaries.csv'
WITH (FORMAT CSV, HEADER);

-- Step 3: Insert with hardcoded county value
INSERT INTO supervisor_salaries (town, county, supervisor, salary)
SELECT town, 'Mills', supervisor, salary
FROM supervisor_salaries_temp;

-- Step 4: Clean up
DROP TABLE supervisor_salaries_temp;
```

```mermaid
flowchart TD
    A[CSV] --> B[Temp Table]
    B --> C[SELECT with 'Mills' as county]
    C --> D[Final Table]
    D --> E[Drop temp table]
```

> 💡 **Temporary tables** vanish when you disconnect. Perfect for staging data.

---

## 📤 Exporting Data with COPY

### The Three Export Types

```mermaid
flowchart TD
    A[COPY TO] --> B[Entire Table]
    A --> C[Selected Columns]
    A --> D[Query Results]
```

### 1️⃣ Export Entire Table (Listing 5-8)

```sql
COPY us_counties_pop_est_2019
TO 'C:\YourDirectory\us_counties_export.txt'
WITH (FORMAT CSV, HEADER, DELIMITER '|');
```

**Output file:**
```
state_fips|county_fips|region|state_name|county_name|...
01|001|3|Alabama|Autauga County|...
```

### 2️⃣ Export Selected Columns (Listing 5-9)

```sql
COPY us_counties_pop_est_2019
(county_name, internal_point_lat, internal_point_lon)
TO 'C:\YourDirectory\us_counties_latlon_export.txt'
WITH (FORMAT CSV, HEADER, DELIMITER '|');
```

```mermaid
flowchart LR
    A[Full Table] --> B[Pick 3 columns]
    B --> C[Export only those]
```

### 3️⃣ Export Query Results (Listing 5-10)

```sql
COPY (
    SELECT county_name, state_name
    FROM us_counties_pop_est_2019
    WHERE county_name ILIKE '%mill%'
)
TO 'C:\YourDirectory\us_counties_mill_export.csv'
WITH (FORMAT CSV, HEADER);
```

**Output:**
```
county_name,state_name
Miller County,Arkansas
Miller County,Georgia
Vermillion County,Indiana
...
```

> 💡 **Powerful:** Any `SELECT` query can be exported!

---

## 🖥️ pgAdmin Import/Export Wizard

**When to use it:** When PostgreSQL is on a **remote server** (cloud) and `COPY` can't see your local files.

```mermaid
flowchart TD
    A[pgAdmin Object Browser] --> B[Right-click table]
    B --> C[Import/Export]
    C --> D{Import or Export?}
    D -->|Import| E[Pick CSV file]
    D -->|Export| F[Pick output path]
    E --> G[Set format options]
    F --> G
    G --> H[Click OK]
```

> 💡 pgAdmin's wizard uses `\copy` behind the scenes — a friendlier face for the same functionality.

---

## 🧩 Complete Import/Export Decision Flow

```mermaid
flowchart TD
    A[Need to move data?] --> B{Direction?}
    B -->|Into PostgreSQL| C[COPY FROM]
    B -->|Out of PostgreSQL| D[COPY TO]
    C --> E{All columns present?}
    E -->|Yes| F[Basic COPY]
    E -->|No| G[Specify columns in parentheses]
    F --> H{All rows wanted?}
    H -->|Yes| I[Done]
    H -->|No| J[Add WHERE clause]
    G --> I
    D --> K{Whole table?}
    K -->|Yes| L[COPY table TO]
    K -->|No| M{Specific columns?}
    M -->|Yes| N[List columns]
    M -->|No| O[Wrap SELECT query]
```

---

## ✅ Chapter 5 Checklist

| Task | Command |
|------|---------|
| Import all columns | `COPY table FROM 'file.csv' WITH (FORMAT CSV, HEADER);` |
| Import subset of columns | `COPY table (col1, col2) FROM 'file.csv' ...` |
| Import subset of rows | `COPY table FROM 'file.csv' ... WHERE condition;` |
| Add value during import | Use temp table + INSERT SELECT |
| Export entire table | `COPY table TO 'file.csv' WITH (FORMAT CSV, HEADER);` |
| Export selected columns | `COPY table (col1, col2) TO 'file.csv' ...` |
| Export query results | `COPY (SELECT ...) TO 'file.csv' ...` |
| Remote server | Use pgAdmin Import/Export wizard |

---

## 🎯 Key Takeaways

1. **CSV is the universal format** — every tool can read/write it
2. **`COPY` is fast** — built for bulk operations
3. **Codes are text, not numbers** — FIPS codes need leading zeros
4. **`bigint` for big numbers** — Alaska's land area exceeds `integer` limits
5. **Temp tables are staging areas** — transform data before final insert
6. **`COPY` can export queries** — wrap any `SELECT` in parentheses
7. **Remote servers need pgAdmin** — `COPY` can only see local files
8. **Always inspect after import** — check row counts and sample data

> In **Chapter 6**, you'll learn **math functions** to analyze your newly imported data — counting, summing, averaging, and more. 🚀