# Essential SQL Command Reference – Expanded Edition

A comprehensive yet succinct field manual for everyday relational‑database tasks. In addition to the core syntax, this edition supplies practical context, guidance on when *not* to apply a command, and illustrative mini‑scenarios. All examples assume a PostgreSQL‑style dialect unless otherwise noted.


## Table of Contents
1. [General Conventions](#general-conventions)
2. [Data Query Language (DQL)](#1-data-query-language-dql)
3. [Data Manipulation Language (DML)](#2-data-manipulation-language-dml)
4. [Data Definition Language (DDL)](#3-data-definition-language-ddl)
5. [Transaction Control Language (TCL)](#4-transaction-control-language-tcl)
6. [Data Control Language (DCL)](#5-data-control-language-dcl)
7. [Diagnostic & Maintenance Aids](#6-diagnostic--maintenance-aids)
8. [Practical Rules of Thumb](#practical-rules-of-thumb)
9. [SQL Command Categories](#-sql-command-categories-overview)
10. [Example SQL Snippets](#-create-table--syntax--constraints)
11. [Advanced SQL Essentials](#-advanced-sql-essentials--annotated-reference)
12. [Entity-Relationship Diagrams](#-entity-relationship-e-r-diagrams--explained-simply)
13. [Query Optimization](#-query-optimization--simple-explanation-with-examples)
14. [Indexing & Hashing](#-indexing--hashing--beginners-guide-with-clear-examples)
15. [Summary of Key Concepts](#-summary-of-key-concepts)

---

## General Conventions

* **Capitals vs lower‑case** – Keywords appear in capitals for clarity; most engines are case‑insensitive.
* **Schema prefixes** – Write `schema.table` when working across multiple schemas (e.g. `public.employees`).
* **Semicolons** – Terminate every statement with `;` to avoid accidental batch execution.
* **Parameterized queries** – Always bind variables (`$1`, `?`, `@p1`, etc.) in application code to prevent SQL injection.

> **When unsure**: Start with **`SELECT … LIMIT 0`** to validate syntax without touching data.

---

## 1 · Data Query Language (DQL)

Data Query Language statements read data without modifying it. Use these queries to retrieve, filter, and combine information from tables or views.

```sql
SELECT first_name, last_name FROM employees;
```

### 1.1 SELECT

**Purpose** Retrieve columns from one or more tables or views.

**When to use** Any time data must be read—dashboards, exports, sanity checks.

**Avoid** Embedding business logic in giant `SELECT` statements that become opaque.

```sql
-- Basic column projection
SELECT first_name, last_name
FROM   employees
WHERE  department_id = 3;

-- Derived columns and functions
SELECT order_id,
       total_price * 0.25 AS vat,
       total_price + (total_price * 0.25) AS grand_total
FROM   orders;

-- Window (analytic) function
SELECT order_id,
       customer_id,
       total_price,
       RANK() OVER (PARTITION BY customer_id ORDER BY total_price DESC) AS spend_rank
FROM   orders;
```

### 1.2 WHERE

Filter rows returned by `SELECT`, `UPDATE`, or `DELETE`.

**When to use** Any operation that must act on a subset of rows.

```sql
-- Date range using BETWEEN (inclusive)
SELECT *
FROM   orders
WHERE  order_date BETWEEN DATE '2025-01-01' AND CURRENT_DATE;
```

> **Tip** Compound predicates execute left‑to‑right; place the most selective conditions first to aid the planner.

### 1.3 DISTINCT

Eliminate duplicate rows.

**When to use** Producing unique lists (e.g. distinct countries) or feeding data into aggregations.

**Pitfall** `DISTINCT` may force a sort/hash that harms performance on large sets. Consider indexing the target column instead.

```sql
-- Unique ship‑to cities for logistics teams
SELECT DISTINCT ship_city
FROM   shipments;
```

### 1.4 GROUP BY

Aggregate rows sharing common values.

**When to use** Generating summaries like counts, sums, averages.

```sql
-- Monthly revenue
SELECT   DATE_TRUNC('month', order_date) AS month,
         SUM(total_price) AS revenue
FROM     orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

### 1.5 HAVING

Filter *after* aggregation.

```sql
-- Departments with more than ten staff
SELECT   department_id, COUNT(*) AS headcount
FROM     employees
GROUP BY department_id
HAVING   COUNT(*) > 10;
```

### 1.6 ORDER BY

Sort the final result set.

**Default order** Ascending (`ASC`). Use `DESC` for descending.

```sql
SELECT last_name, hire_date
FROM   employees
ORDER BY hire_date DESC, last_name ASC;
```

### 1.7 LIMIT / OFFSET

Restrict or paginate result sets.

**Alternatives** Use key‑set (“seek”) pagination—`WHERE id > $last_id ORDER BY id LIMIT n`—to avoid deep offsets.

```sql
SELECT *
FROM   products
ORDER BY product_id
LIMIT 10 OFFSET 20;  -- rows 21–30
```

### 1.8 JOIN

Combine rows from related tables.

**When to use** Enrich data from multiple domains (e.g. orders + customers).

```sql
SELECT e.first_name,
       d.department_name
FROM   employees AS e
INNER JOIN departments AS d
  ON   e.department_id = d.department_id;
```

| Variant         | Returned rows                  | Typical scenario            |
| --------------- | ------------------------------ | --------------------------- |
| **INNER**       | Matching rows in *both* tables | Standard look‑ups           |
| **LEFT OUTER**  | All left rows + matches        | Optional child records      |
| **RIGHT OUTER** | All right rows + matches       | Less common; legacy schemas |
| **FULL OUTER**  | Union of left and right        | Data reconciliation         |

> **Cross joins** (`CROSS JOIN` or comma syntax) generate cartesian products—reserve for exhaustive pairings only.

### 1.9 UNION / UNION ALL

Concatenate result sets from identical column lists.

```sql
-- One list of all supplier and customer cities
SELECT city FROM suppliers
UNION ALL                 -- keep duplicates to reflect true counts
SELECT city FROM customers;
```

### 1.10 CASE

Inline conditional logic.

```sql
SELECT   order_id,
         CASE
           WHEN total >= 1000 THEN 'Large'
           WHEN total >= 500  THEN 'Medium'
           ELSE 'Small'
         END AS order_size
FROM     orders;
```

### 1.11 IN · BETWEEN · LIKE · IS NULL

Helpful predicates.

```sql
SELECT *
FROM   books
WHERE  genre      IN ('Sci‑Fi', 'Fantasy')
  AND  page_count BETWEEN 200 AND 800
  AND  title      LIKE '%Galaxy%'
  AND  series_id IS NULL;  -- stand‑alone titles
```

### 1.12 Common Table Expressions (CTE)

Write temporary, named sub‑queries with `WITH` for clarity.

```sql
WITH recent_orders AS (
  SELECT *
  FROM   orders
  WHERE  order_date >= CURRENT_DATE - INTERVAL '7 days'
)
SELECT customer_id, COUNT(*) AS orders_last_week
FROM   recent_orders
GROUP BY customer_id;
```

> **Performance note** In PostgreSQL, a non‑recursive CTE is an optimisation fence until v12; materialisation may hurt. Consider in‑lining or adding `MATERIALIZED / NOT MATERIALIZED` hints.

---

## 2 · Data Manipulation Language (DML)
Data Manipulation Language commands change the contents of tables. Use them for inserting new rows, updating existing ones, or removing unwanted data.

```sql
INSERT INTO audit_log(message) VALUES ('system started');
```


### 2.1 INSERT

Add new records.

```sql
INSERT INTO customers (first_name, last_name, email)
VALUES ('Ada', 'Lovelace', 'ada@example.com');

-- Multi‑row insert
INSERT INTO products (name, price)
VALUES
  ('Keyboard', 29.99),
  ('Mouse',    19.99);
```

**Upsert (insert‑or‑update)** – dialect‑specific:

```sql
-- PostgreSQL
INSERT INTO inventory (sku, qty)
VALUES ('ABC123', 10)
ON CONFLICT (sku) DO UPDATE
  SET qty = inventory.qty + EXCLUDED.qty;
```

### 2.2 UPDATE

Modify existing rows.

```sql
UPDATE employees
SET    salary = salary * 1.05
WHERE  performance_rating = 'Excellent';
```

> **Best practice** Run a `SELECT` with the same `WHERE` clause first to confirm affected rows.

### 2.3 DELETE

Remove specific rows.

```sql
DELETE FROM sessions
WHERE  last_active < CURRENT_DATE - INTERVAL '30 days';
```

### 2.4 TRUNCATE TABLE

Fast wholesale removal of **all** rows.

*Implicit* commit in many systems; cannot be rolled back in MySQL.

```sql
TRUNCATE TABLE audit_log RESTART IDENTITY;
```

---

## 3 · Data Definition Language (DDL)
Data Definition Language statements create, modify, or remove database objects such as tables and indexes.

```sql
CREATE TABLE demo(id INT PRIMARY KEY);
```


### 3.1 CREATE / DROP DATABASE

```sql
CREATE DATABASE sales_analytics;
DROP   DATABASE test_lab;
```

### 3.2 USE (or \c in psql)

```sql
USE sales_analytics;  -- MySQL / SQL Server
\c  sales_analytics; -- PostgreSQL psql shell
```

### 3.3 CREATE TABLE

```sql
CREATE TABLE products (
  product_id   SERIAL PRIMARY KEY,
  name         VARCHAR(100) NOT NULL,
  price        DECIMAL(10,2) CHECK (price >= 0),
  created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.4 ALTER TABLE

```sql
ALTER TABLE products
ADD COLUMN discontinued BOOLEAN DEFAULT FALSE;

-- Rename a column (PostgreSQL)
ALTER TABLE products RENAME COLUMN name TO product_name;
```

### 3.5 DROP TABLE

```sql
DROP TABLE temp_import;
```

### 3.6 CREATE / DROP INDEX

```sql
CREATE INDEX idx_products_name
  ON products (name);

DROP INDEX idx_products_name;
```

> **When to use** Index columns appearing in `JOIN`, `WHERE`, and `ORDER BY` clauses that filter many rows. Avoid over‑indexing small or write‑heavy tables.

### 3.7 CREATE / DROP VIEW

```sql
CREATE VIEW active_customers AS
  SELECT * FROM customers WHERE active = TRUE;

DROP VIEW active_customers;
```

> **Tip** Use views to encapsulate business rules and provide backward compatibility during schema changes.

### 3.8 Constraints Cheat Sheet

| Constraint    | Purpose                        | Example                                  |
| ------------- | ------------------------------ | ---------------------------------------- |
| `PRIMARY KEY` | Uniquely identify each row     | `product_id SERIAL PRIMARY KEY`          |
| `UNIQUE`      | Prevent duplicate values       | `email VARCHAR(255) UNIQUE`              |
| `FOREIGN KEY` | Enforce parent‑child integrity | `dept_id INT REFERENCES departments(id)` |
| `CHECK`       | Custom rule                    | `CHECK (price >= 0)`                     |
| `NOT NULL`    | Require value                  | `name TEXT NOT NULL`                     |

---

## 4 · Transaction Control Language (TCL)
Transaction Control Language manages logical units of work. These commands allow you to commit or roll back groups of statements, ensuring atomicity.

```sql
BEGIN;
-- statements
COMMIT;
```


### 4.1 BEGIN TRANSACTION

Start an atomic unit of work (alias `START TRANSACTION`).

### 4.2 SAVEPOINT

Mark a logical rollback point.

### 4.3 COMMIT / ROLLBACK

Finalize or undo all changes since `BEGIN`.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  SAVEPOINT after_debit;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
  -- Suppose the second update fails …
ROLLBACK TO after_debit;  -- undo credit, keep debit
COMMIT;                   -- persist debit only
```

> **Isolation levels** `READ COMMITTED` is default in PostgreSQL; escalate to `SERIALIZABLE` for strict correctness at the cost of contention.

---

## 5 · Data Control Language (DCL)
Data Control Language governs permissions. Use it to grant or revoke user access to specific tables or entire databases.

```sql
GRANT SELECT ON employees TO hr_reader;
```


### 5.1 GRANT

```sql
GRANT SELECT, INSERT ON products TO reporting_user;
```

### 5.2 REVOKE

```sql
REVOKE INSERT ON products FROM reporting_user;
```

> **Security note** Prefer granting roles to users rather than direct object privileges for simpler audits.

---

## 6 · Diagnostic & Maintenance Aids
Diagnostic and maintenance commands help inspect the state of the database and keep it running efficiently.

```sql
EXPLAIN SELECT * FROM orders WHERE id = 1;
```


### 6.1 `EXPLAIN` / `EXPLAIN ANALYZE`

Display the query plan—and runtime metrics with `ANALYZE`.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;
```

### 6.2 `VACUUM` (PostgreSQL)

Reclaim space and update statistics.

```sql
VACUUM (VERBOSE, ANALYZE) orders;
```

### 6.3 `SHOW` / `SET`

Inspect or override session parameters.

```sql
SHOW search_path;
SET  statement_timeout = '5s';
```

---

## Practical Rules of Thumb
Quick reminders for day-to-day SQL work.


1. **Index sensibly** – Cover frequent predicates but weigh against slower writes.
2. **Guard production** – Run ad‑hoc `UPDATE`/`DELETE` in a transaction and confirm `SELECT COUNT(*)` first.
3. **Batch inserts** – Use multi‑row `INSERT` or engine bulk‑copy tools for large loads.
4. **Archive, don’t delete** – Soft‑delete with an `archived_at` timestamp to keep audit trails.
5. **Name constraints & indexes explicitly** – Easier troubleshooting (`CONSTRAINT ck_price_nonnegative`).
6. **Use CTEs for clarity, sub‑queries for speed** – Rewrite once logic stabilises.
7. **Prefer `TRUNCATE` over `DELETE`** when wiping whole tables and FK constraints permit.



## 🧾 SQL Command Categories (Overview)
Overview of how different SQL statements fit into broad categories.


```sql
-- SQL Command Categories
-- -----------------------
-- DDL – Schema definitions
CREATE DATABASE, DROP DATABASE, CREATE TABLE, ALTER TABLE, DROP TABLE

-- DML – Modify data
INSERT, UPDATE, DELETE

-- DQL – Query data
SELECT

-- DCL – Permissions
GRANT, REVOKE

-- TCL – Transactions
COMMIT, ROLLBACK, SAVEPOINT
```

---

## 📦 CREATE TABLE – Syntax & Constraints

```sql
CREATE TABLE Classroom (
  Building VARCHAR(15),
  Room     VARCHAR(7),
  Capacity DECIMAL(4,0),
  PRIMARY KEY (Building, Room)
);
```

```sql
CREATE TABLE Instructor (
  InstID    VARCHAR(5),
  InstName  VARCHAR(20),
  DeptName  VARCHAR(20),
  Salary    DECIMAL(8,2),
  PRIMARY KEY (InstID),
  FOREIGN KEY (DeptName) REFERENCES Department(DeptName)
);
```

> ✅ A `PRIMARY KEY` ensures:
> - Uniqueness of the column(s)
> - `NOT NULL` constraint on the key columns  
>
> ✅ A `FOREIGN KEY` ensures referential integrity between two tables.

---

## 🔁 FOREIGN KEY – Actions on DELETE

```sql
-- Set DeptName to NULL if the department is deleted
FOREIGN KEY (DeptName)
  REFERENCES Department(DeptName)
  ON DELETE SET NULL;

-- Delete instructors if their department is deleted
FOREIGN KEY (DeptName)
  REFERENCES Department(DeptName)
  ON DELETE CASCADE;
```

---

## 🔄 ALTER TABLE – Common Operations

```sql
-- Add a column with default NULLs
ALTER TABLE Student ADD COLUMN ShoeSize DECIMAL(2, 0);

-- Remove a column
ALTER TABLE Student DROP COLUMN ShoeSize;

-- Add keys
ALTER TABLE Student ADD PRIMARY KEY (StudID);
ALTER TABLE Student ADD FOREIGN KEY (DeptName) REFERENCES Department(DeptName);
```

---

## 📊 SELECT Examples

```sql
-- Basic projection
SELECT InstName FROM Instructor;

-- DISTINCT values
SELECT DISTINCT DeptName FROM Instructor;

-- Expressions and aliases
SELECT InstID, Salary / 12 AS Monthly FROM Instructor;

-- WHERE with compound conditions
SELECT InstName
FROM   Instructor
WHERE  DeptName = 'Comp. Sci.'
  AND  Salary > 70000;
```

---

## 🔗 JOIN Examples

```sql
-- Classic JOIN
SELECT InstName, CourseID
FROM   Instructor, Teaches
WHERE  Instructor.InstID = Teaches.InstID;

-- NATURAL JOIN variant
SELECT InstName, CourseID
FROM   Instructor NATURAL JOIN Teaches;
```

---

## 🛠 INSERT / UPDATE / DELETE

```sql
-- Insert a row
INSERT INTO Course (CourseID, Title, DeptName, Credits)
VALUES ('CS-437', 'Database Systems', 'Comp. Sci.', 4);

-- Update a value conditionally
UPDATE Instructor
SET    Salary = Salary * 1.10
WHERE  DeptName = 'Comp. Sci.';

-- Delete rows with safe mode off (MySQL)
SET SQL_SAFE_UPDATES = 0;
DELETE FROM Course WHERE Credits <= 3;
```

---

## 🎯 INSERT via SELECT

```sql
-- Promote students with >100 credits to instructors
INSERT INTO Instructor
SELECT StudID, StudName, DeptName, 10000
FROM   Student
WHERE  TotCredits > 100;
```

---

## 🧪 Example Exercise Schema (for practice)

```sql
-- Insurance DB Example
CREATE TABLE Person (
  DriverID    VARCHAR(10) PRIMARY KEY,
  DriverName  VARCHAR(100),
  Address     TEXT
);

CREATE TABLE Car (
  License     VARCHAR(10) PRIMARY KEY,
  Model       VARCHAR(50),
  ProdYear    INT
);

CREATE TABLE Accident (
  ReportNumber INT PRIMARY KEY,
  AccDate      DATE,
  Location     VARCHAR(100)
);

CREATE TABLE Owns (
  DriverID VARCHAR(10),
  License  VARCHAR(10),
  PRIMARY KEY (DriverID, License),
  FOREIGN KEY (DriverID) REFERENCES Person(DriverID),
  FOREIGN KEY (License)  REFERENCES Car(License)
);

CREATE TABLE Participants (
  ReportNumber   INT,
  License        VARCHAR(10),
  DriverID       VARCHAR(10),
  DamageAmount   DECIMAL(10,2),
  PRIMARY KEY (ReportNumber, License),
  FOREIGN KEY (ReportNumber) REFERENCES Accident(ReportNumber),
  FOREIGN KEY (License)      REFERENCES Car(License),
  FOREIGN KEY (DriverID)     REFERENCES Person(DriverID)
);
```


# 🧮 Advanced SQL Essentials — Annotated Reference

This document extends the foundational SQL reference with practical usage patterns, edge-case behaviour, and enhanced context for writing expressive and correct queries. Based on DTU’s “Introductory SQL Part 2” slides.

---

## 🔤 String Operations and Pattern Matching

### Basic String Handling
```sql
SELECT CONCAT('Course: ', Title) FROM Course;
```
- Strings are enclosed in `'...'`.
- Escaping characters like `'` and `\` uses backslashes.
- `CONCAT()` merges multiple string fields or constants.

### Pattern Matching with `LIKE`
```sql
SELECT InstName
FROM   Instructor
WHERE  InstName LIKE '%ri%';
```
- `%` = wildcard for zero or more characters
- `_` = wildcard for a single character
- `LIKE` is case-sensitive in standard SQL, but **not** in MySQL/MariaDB.

> **Use-case**: Partial name lookups, course titles with common prefixes, etc.

---

## ⚠️ NULL Handling and Three-Valued Logic

```sql
SELECT * FROM Instructor WHERE Salary IS NULL;
```

- `NULL` ≠ any value, even another `NULL`.
- `IS NULL` / `IS NOT NULL` are required — `= NULL` will not work.
- Arithmetic with `NULL` returns `NULL`.
- Conditions with `NULL` yield `UNKNOWN` → treated as **false** in `WHERE`.

### SELECT DISTINCT and NULLs
```sql
SELECT DISTINCT DeptName, Salary FROM Instructor;
```
- Two rows with `(DeptName, NULL)` will collapse into one, as NULLs are treated as equal by `DISTINCT`.

---

## 🧮 Aggregate Functions

```sql
SELECT AVG(Salary), COUNT(*), COUNT(Salary)
FROM   Instructor;
```

- `COUNT(*)` counts all rows, including those with NULLs.
- `COUNT(column)` ignores NULLs.
- Aggregates ignore `NULL` by default (`AVG`, `SUM`, `MAX`, `MIN`).
- Use `DISTINCT` inside aggregates to count unique values.

---

## 🧩 GROUP BY and HAVING

### Aggregation by group
```sql
SELECT DeptName, AVG(Salary)
FROM   Instructor
GROUP BY DeptName;
```

### Filter aggregated results
```sql
SELECT DeptName, AVG(Salary)
FROM   Instructor
GROUP BY DeptName
HAVING AVG(Salary) > 65000;
```

- `WHERE` filters **rows before grouping**.
- `HAVING` filters **grouped rows**.
- Never use `HAVING` when `WHERE` suffices.

---

## 🔎 Scalar Subqueries

```sql
SELECT InstName
FROM   Instructor
WHERE  Salary > (SELECT AVG(Salary) FROM Instructor);
```

- Returns instructors earning above average.
- Subquery must return exactly one value → 1x1 relation.

---

## 🔘 IN, NOT IN, EXISTS

### Membership
```sql
SELECT InstName
FROM   Instructor
WHERE  InstName NOT IN ('Mozart', 'Einstein');
```

### Subquery-based membership
```sql
SELECT CourseID
FROM   Section
WHERE  Semester = 'Fall' AND StudyYear = 2009
  AND  CourseID IN (
    SELECT CourseID
    FROM   Section
    WHERE  Semester = 'Spring' AND StudyYear = 2010
);
```

### EXISTS pattern
```sql
SELECT CourseID
FROM   Section S
WHERE  Semester='Fall' AND StudyYear=2009
  AND EXISTS (
    SELECT * FROM Section T
    WHERE Semester='Spring' AND StudyYear=2010
      AND S.CourseID = T.CourseID
  );
```

---

## 🎛 Comparison Using SOME and ALL

### SOME (≅ at least one match)
```sql
SELECT InstName
FROM   Instructor
WHERE  Salary > SOME (
  SELECT Salary FROM Instructor WHERE DeptName = 'Finance'
);
```

### ALL (≅ condition must hold for all)
```sql
SELECT DeptName
FROM   Department
WHERE  Budget >= ALL (
  SELECT Budget FROM Department
);
```

---

## ⚖️ Set Operations

### UNION
```sql
(SELECT CourseID FROM Section WHERE Semester='Fall' AND StudyYear=2009)
UNION
(SELECT CourseID FROM Section WHERE Semester='Spring' AND StudyYear=2010);
```

### INTERSECT (common)
```sql
... INTERSECT ...
```

### EXCEPT (difference)
```sql
... EXCEPT ...
```

> ⚠ MySQL doesn’t support `INTERSECT`/`EXCEPT` – use `IN` / `NOT IN` alternatives instead.

---

## 🎓 Practical Exercises (From Lecture)

1. Count enrollments:
   ```sql
   SELECT CourseID, SectionID, COUNT(StudID)
   FROM   Takes
   WHERE  Semester = 'Fall' AND StudyYear = 2009
   GROUP BY CourseID, SectionID;
   ```

2. Instructor credit summary:
   ```sql
   SELECT Title, SUM(Credits)
   FROM   Instructor
   NATURAL JOIN Teaches
   NATURAL JOIN Course
   WHERE  InstName = 'Brandt'
   GROUP BY Title;
   ```

3. Delete courses never taught:
   ```sql
   DELETE FROM Course
   WHERE  CourseID NOT IN (
     SELECT CourseID FROM Section
   );
   ```

---

*Prepared to deepen practical SQL knowledge and reinforce edge cases often overlooked in day-to-day querying.*



# 🔗 Intermediate SQL – Extended Reference

Based on Chapter 4: Joins, Constraints, Views, and Authorization (DTU, 2025). This guide adds annotated context, use-cases, caveats, and examples suitable for serious database engineering.

---

## 🔄 JOIN Types & Techniques

### 1. Cartesian Product / INNER JOIN
```sql
SELECT * FROM Courses, PreReqs;
SELECT * FROM Courses JOIN PreReqs;
```
- Joins every row in the first table with every row in the second.
- Avoid unless you have an explicit join condition, as this may lead to explosive result sets.

### 2. NATURAL JOIN
```sql
SELECT * FROM Courses NATURAL JOIN PreReqs;
```
- Joins on all columns with the same name.
- Can accidentally join on unintended columns—**prefer `USING` or `ON` for clarity**.

### 3. OUTER JOINs
- Use **LEFT**, **RIGHT**, or **FULL OUTER JOIN** to retain unmatched rows.
```sql
SELECT * FROM Courses NATURAL LEFT OUTER JOIN PreReqs;
```

**Important Note**: `FULL OUTER JOIN` is unsupported in MySQL/MariaDB; emulate using:
```sql
(SELECT ... LEFT OUTER JOIN ...) UNION (SELECT ... RIGHT OUTER JOIN ...);
```

### 4. JOIN USING vs ON
```sql
SELECT * FROM Courses JOIN PreReqs USING (CourseID);
SELECT * FROM Courses JOIN PreReqs ON Courses.CourseID = PreReqs.CourseID;
```
- `USING` joins on one or more columns **by name** and keeps one copy.
- `ON` gives **full control**, allowing joining on differently named columns.

> ✅ Use `ON` when schema evolution might cause divergence in attribute names.

---

## ✅ Constraints & Data Integrity

### NOT NULL and PRIMARY KEY
```sql
LastName VARCHAR(20) NOT NULL;
PRIMARY KEY(StudID);
```
- PK columns are implicitly `NOT NULL`.
- Prevents missing or ambiguous row identification.

### ENUM – Typed Literals
```sql
Semester ENUM('Fall', 'Winter', 'Spring', 'Summer');
```
- Enforces a finite set of allowed values.

### FOREIGN KEY
```sql
FOREIGN KEY(DeptName) REFERENCES Department(DeptName)
```
- Ensures referenced values exist in the parent table.

### ON DELETE / ON UPDATE Actions
```sql
FOREIGN KEY(DeptName) REFERENCES Department(DeptName)
  ON DELETE SET NULL
  ON UPDATE CASCADE;
```

#### Actions:
- `SET NULL`: orphaned children are retained but disconnected.
- `CASCADE`: child rows are deleted/updated in tandem.

---

## 👓 Views: Virtual Tables for Restriction & Abstraction

### Basic View Definition
```sql
CREATE VIEW SeniorStudents AS
SELECT StudID, StudName FROM Student WHERE TotCredits > 100;
```

### Views With Calculations
```sql
CREATE VIEW DepartmentSalary AS
SELECT DeptName, SUM(Salary) AS DeptSalary
FROM Instructor GROUP BY DeptName;
```

### View from Join
```sql
CREATE VIEW Studentview AS
SELECT StudID, StudName, TIMESTAMPDIFF(YEAR, Birth, CURRENT_DATE()) AS Age
FROM Student NATURAL LEFT OUTER JOIN Advisor;
```

> ⚠ Views are not materialized unless explicitly requested in systems like PostgreSQL or SQL Server.

### View Update Restrictions
- Must be based on a single base table.
- Cannot use aggregates or `GROUP BY`.
- All non-view columns must be nullable or have defaults.

---

## 🔐 Authorization: Managing Access

### CREATE and DROP Users
```sql
CREATE USER 'Karen'@'localhost' IDENTIFIED BY 'SetPassword';
DROP USER 'Karen'@'localhost';
```

### GRANT / REVOKE Privileges
```sql
GRANT SELECT ON University.* TO 'Karen'@'localhost';
REVOKE INSERT ON University.Student FROM 'Karen'@'localhost';
```

### Privilege Types
| Privilege | Grants Access To |
|----------|-------------------|
| SELECT   | Querying tables    |
| INSERT   | Adding rows        |
| UPDATE   | Modifying rows     |
| DELETE   | Removing rows      |
| INDEX    | Creating indexes   |
| CREATE   | Creating tables    |
| DROP     | Deleting tables    |
| GRANT OPTION | Delegating privileges |

### Roles (MariaDB only)
```sql
CREATE ROLE TeachingAssistant;
GRANT SELECT ON University.Takes TO TeachingAssistant;
GRANT TeachingAssistant TO 'Thomas'@'localhost';
```

---

## 🧪 Representative Exercises

### Section Count by Instructor (LEFT OUTER JOIN)
```sql
SELECT InstID, InstName, COUNT(SectionID) AS SectionsTaught
FROM Instructor NATURAL LEFT OUTER JOIN Teaches
GROUP BY InstID, InstName;
```

### Equivalent with Scalar Subquery
```sql
SELECT InstID, InstName,
  (SELECT COUNT(*) FROM Teaches WHERE Teaches.InstID = Instructor.InstID) AS SectionsTaught
FROM Instructor;
```

### Create a View with Aggregation
```sql
CREATE VIEW Creditview(StudyYear, SumCredits) AS
SELECT StudyYear, SUM(Credits)
FROM Takes NATURAL JOIN Course
GROUP BY StudyYear;
```

---

*Designed for developers and data engineers needing advanced relational constructs with data governance mechanisms.*



# 🧮 Advanced SQL Essentials — Annotated Reference

This document extends the foundational SQL reference with practical usage patterns, edge-case behaviour, and enhanced context for writing expressive and correct queries. Based on DTU’s “Introductory SQL Part 2” slides.

---

## 🔤 String Operations and Pattern Matching

### Basic String Handling
```sql
SELECT CONCAT('Course: ', Title) FROM Course;
```
- Strings are enclosed in `'...'`.
- Escaping characters like `'` and `\` uses backslashes.
- `CONCAT()` merges multiple string fields or constants.

### Pattern Matching with `LIKE`
```sql
SELECT InstName
FROM   Instructor
WHERE  InstName LIKE '%ri%';
```
- `%` = wildcard for zero or more characters
- `_` = wildcard for a single character
- `LIKE` is case-sensitive in standard SQL, but **not** in MySQL/MariaDB.

> **Use-case**: Partial name lookups, course titles with common prefixes, etc.

---

## ⚠️ NULL Handling and Three-Valued Logic

```sql
SELECT * FROM Instructor WHERE Salary IS NULL;
```

- `NULL` ≠ any value, even another `NULL`.
- `IS NULL` / `IS NOT NULL` are required — `= NULL` will not work.
- Arithmetic with `NULL` returns `NULL`.
- Conditions with `NULL` yield `UNKNOWN` → treated as **false** in `WHERE`.

### SELECT DISTINCT and NULLs
```sql
SELECT DISTINCT DeptName, Salary FROM Instructor;
```
- Two rows with `(DeptName, NULL)` will collapse into one, as NULLs are treated as equal by `DISTINCT`.

---

## 🧮 Aggregate Functions

```sql
SELECT AVG(Salary), COUNT(*), COUNT(Salary)
FROM   Instructor;
```

- `COUNT(*)` counts all rows, including those with NULLs.
- `COUNT(column)` ignores NULLs.
- Aggregates ignore `NULL` by default (`AVG`, `SUM`, `MAX`, `MIN`).
- Use `DISTINCT` inside aggregates to count unique values.

---

## 🧩 GROUP BY and HAVING

### Aggregation by group
```sql
SELECT DeptName, AVG(Salary)
FROM   Instructor
GROUP BY DeptName;
```

### Filter aggregated results
```sql
SELECT DeptName, AVG(Salary)
FROM   Instructor
GROUP BY DeptName
HAVING AVG(Salary) > 65000;
```

- `WHERE` filters **rows before grouping**.
- `HAVING` filters **grouped rows**.
- Never use `HAVING` when `WHERE` suffices.

---

## 🔎 Scalar Subqueries

```sql
SELECT InstName
FROM   Instructor
WHERE  Salary > (SELECT AVG(Salary) FROM Instructor);
```

- Returns instructors earning above average.
- Subquery must return exactly one value → 1x1 relation.

---

## 🔘 IN, NOT IN, EXISTS

### Membership
```sql
SELECT InstName
FROM   Instructor
WHERE  InstName NOT IN ('Mozart', 'Einstein');
```

### Subquery-based membership
```sql
SELECT CourseID
FROM   Section
WHERE  Semester = 'Fall' AND StudyYear = 2009
  AND  CourseID IN (
    SELECT CourseID
    FROM   Section
    WHERE  Semester = 'Spring' AND StudyYear = 2010
);
```

### EXISTS pattern
```sql
SELECT CourseID
FROM   Section S
WHERE  Semester='Fall' AND StudyYear=2009
  AND EXISTS (
    SELECT * FROM Section T
    WHERE Semester='Spring' AND StudyYear=2010
      AND S.CourseID = T.CourseID
  );
```

---

## 🎛 Comparison Using SOME and ALL

### SOME (≅ at least one match)
```sql
SELECT InstName
FROM   Instructor
WHERE  Salary > SOME (
  SELECT Salary FROM Instructor WHERE DeptName = 'Finance'
);
```

### ALL (≅ condition must hold for all)
```sql
SELECT DeptName
FROM   Department
WHERE  Budget >= ALL (
  SELECT Budget FROM Department
);
```

---

## ⚖️ Set Operations

### UNION
```sql
(SELECT CourseID FROM Section WHERE Semester='Fall' AND StudyYear=2009)
UNION
(SELECT CourseID FROM Section WHERE Semester='Spring' AND StudyYear=2010);
```

### INTERSECT (common)
```sql
... INTERSECT ...
```

### EXCEPT (difference)
```sql
... EXCEPT ...
```

> ⚠ MySQL doesn’t support `INTERSECT`/`EXCEPT` – use `IN` / `NOT IN` alternatives instead.

---

## 🎓 Practical Exercises (From Lecture)

1. Count enrollments:
   ```sql
   SELECT CourseID, SectionID, COUNT(StudID)
   FROM   Takes
   WHERE  Semester = 'Fall' AND StudyYear = 2009
   GROUP BY CourseID, SectionID;
   ```

2. Instructor credit summary:
   ```sql
   SELECT Title, SUM(Credits)
   FROM   Instructor
   NATURAL JOIN Teaches
   NATURAL JOIN Course
   WHERE  InstName = 'Brandt'
   GROUP BY Title;
   ```

3. Delete courses never taught:
   ```sql
   DELETE FROM Course
   WHERE  CourseID NOT IN (
     SELECT CourseID FROM Section
   );
   ```

---

*Prepared to deepen practical SQL knowledge and reinforce edge cases often overlooked in day-to-day querying.*


# 🧩 Entity-Relationship (E-R) Diagrams – Explained Simply

Entity-Relationship Diagrams are used to **model data** in a visual way before implementing it in a database. This helps ensure correctness, avoid duplication, and define how data elements relate.

---

## 🗺️ Phases of Database Design

1. **Conceptual Design**  
   → What data do we need?  
   → Represented using E-R diagrams. No technical database details yet.

2. **Logical Design**  
   → Convert the conceptual model into relational tables (e.g., for SQL databases).

3. **Physical Design**  
   → Decide how tables will be stored, indexed, optimized in an actual DBMS.

---

## 📦 Entities, Attributes, and Relationships

- **Entity**: A real-world object (e.g., a `Student`, a `Course`)
- **Entity Set**: A collection of similar entities (e.g., all students)
- **Attribute**: A property of an entity (e.g., `name`, `ID`, `salary`)
- **Relationship**: A connection between two or more entities (e.g., `Advisor` relates `Instructor` to `Student`)

```text
Student (ID, Name, TotalCredits)
Instructor (ID, Name, Salary)
```

---

## 🧷 Types of Attributes

| Type            | Description                                        | Example                   |
|-----------------|----------------------------------------------------|---------------------------|
| Simple (Atomic) | Cannot be divided                                  | `Name`, `Salary`          |
| Composite       | Consists of multiple parts                         | `Address` = street, city  |
| Multivalued     | Can have many values                               | `{PhoneNumber}`           |
| Derived         | Calculated from other attributes                   | `Age = CurrentDate - DOB` |

---

## 🔁 Relationship Properties

### 1. **Cardinality**  
Defines how many entities participate:

- One-to-One
- One-to-Many
- Many-to-Many

```text
Instructor ⟶⟶ Student
(One instructor advises many students)
```

### 2. **Participation**
- **Total**: Every entity **must** be part of a relationship (shown as double line)
- **Partial**: Some entities **may** not participate (shown as single line)

---

## 🔑 Keys

- **Super Key**: Any set of attributes that uniquely identifies an entity
- **Candidate Key**: A minimal super key (no unnecessary fields)
- **Primary Key**: Chosen candidate key used for actual identification

> Example:  
> For `Instructor(ID, Name, Salary)`  
> → `ID` is the Primary Key

---

## 🧬 Strong vs Weak Entities

- **Strong Entity**: Has its own primary key
- **Weak Entity**: Cannot be uniquely identified without another entity

> Example:  
> `Section` (depends on `Course`)  
> Primary Key = `CourseID + SectionID + Semester + Year`

---

## 🎯 Converting E-R to Tables

### Strong Entities
```sql
CREATE TABLE Course (
  CourseID VARCHAR(10) PRIMARY KEY,
  Title    VARCHAR(50),
  Credits  INT
);
```

### Weak Entities
Include key from identifying entity
```sql
CREATE TABLE Section (
  CourseID  VARCHAR(10),
  SecID     VARCHAR(5),
  Semester  VARCHAR(10),
  Year      INT,
  PRIMARY KEY (CourseID, SecID, Semester, Year),
  FOREIGN KEY (CourseID) REFERENCES Course
);
```

---

## 🔗 Converting Relationships to Tables

### Many-to-Many
```sql
CREATE TABLE Advisor (
  StudentID    VARCHAR(10),
  InstructorID VARCHAR(10),
  StartDate    DATE,
  PRIMARY KEY (StudentID, InstructorID),
  FOREIGN KEY (StudentID) REFERENCES Student,
  FOREIGN KEY (InstructorID) REFERENCES Instructor
);
```

### Many-to-One (optimized)
Add FK directly to many-side entity:
```sql
CREATE TABLE Instructor (
  ID         VARCHAR(10) PRIMARY KEY,
  Name       VARCHAR(50),
  DeptName   VARCHAR(50),
  Salary     DECIMAL,
  FOREIGN KEY (DeptName) REFERENCES Department
);
```

---

## 🧩 Diagram Elements Summary

- **Rectangle** = Entity
- **Ellipse** = Attribute
- **Diamond** = Relationship
- **Double rectangle** = Weak entity
- **Double diamond** = Identifying relationship
- **Underline** = Primary Key
- **Dashed underline** = Partial Key (for weak entities)

---

## 🧠 Tips for Understanding

1. **Start from nouns**: Entities = things; Attributes = properties; Relationships = verbs or associations.
2. **Draw participation & cardinality carefully**: They determine your foreign keys.
3. **Normalize names**: Be consistent in casing and naming attributes.
4. **Validate your model**: Can you answer common queries from your diagram?

---


# 🧼 Normalization – Explained for Beginners

Normalization helps keep your database clean, efficient, and consistent. It removes redundant data and prevents weird update issues. Here’s a simplified guide with examples to understand **1NF**, **2NF**, **3NF**, and **BCNF**.

---

## 🤔 Why Normalize?

Imagine this table of instructors:

| ID   | Name     | Dept     | Building | Budget |
|------|----------|----------|----------|--------|
| 1    | Alice    | Physics  | A        | 70000  |
| 2    | Bob      | Physics  | A        | 70000  |
| 3    | Clara    | Math     | B        | 90000  |

### ❌ Problems:
- **Redundancy**: Dept, Building, and Budget are repeated for each instructor.
- **Update anomaly**: Change the Physics department’s budget? You must change it in multiple rows.
- **Insert anomaly**: You can’t add a new department without adding an instructor.
- **Delete anomaly**: Remove the last Physics instructor, and you lose data about the department.

✅ **Normalization solves this**.

---

## 1️⃣ First Normal Form (1NF)

### Rule:
- Every column must have **atomic (single)** values.
- No lists or nested data.

### ❌ Bad Example:
| OrderNo | ItemNos       |
|---------|---------------|
| 1000    | 100, 102, 105 |

### ✅ Good Example (Normalized to 1NF):
| OrderNo | ItemNo |
|---------|--------|
| 1000    | 100    |
| 1000    | 102    |
| 1000    | 105    |

👉 **Explanation**: We split the multi-valued `ItemNos` field into individual rows.

---

## 2️⃣ Second Normal Form (2NF)

### Rule:
- Must be in 1NF.
- All non-key columns must depend on the **whole** primary key (not just part of it).

### ❌ Bad Example (1NF but not 2NF):

| OrderNo | ItemNo | ItemName        |
|---------|--------|-----------------|
| 1000    | 100    | Car Nitrothane  |
| 1000    | 102    | Airplane Spirit |

Here, the key is (OrderNo, ItemNo). But `ItemName` depends only on `ItemNo`.

### ✅ Solution (2NF):

**Orders**  
| OrderNo | ItemNo |
|---------|--------|
| 1000    | 100    |

**Items**  
| ItemNo | ItemName        |
|--------|-----------------|
| 100    | Car Nitrothane  |

---

## 3️⃣ Third Normal Form (3NF)

### Rule:
- Must be in 2NF.
- Non-key columns must not depend on **other non-key columns** (no transitive dependency).

### ❌ Bad Example (2NF but not 3NF):

| CustomerNo | PostNo | CityName  |
|------------|--------|-----------|
| 1          | 2500   | Valby     |
| 2          | 5000   | Glostrup  |

Here:
- `CustomerNo` → `PostNo`
- `PostNo` → `CityName`
- So `CustomerNo` → `CityName` (transitive)

### ✅ Solution:

**Customers3NF**  
| CustomerNo | PostNo |
|------------|--------|
| 1          | 2500   |

**Post**  
| PostNo | CityName |
|--------|----------|
| 2500   | Valby    |

---

## 🧠 Functional Dependencies

A → B means: “if I know A, I always know B”.

Example:

- In `Shipments(Vendor, Part, Qty)`
  - `{Vendor, Part} → Qty` (knowing both determines quantity)
  - `{Vendor} → City` (vendor uniquely determines their city)

---

## 🔑 Keys

- **Super Key**: Any combo of columns that uniquely identifies a row.
- **Candidate Key**: A *minimal* super key.
- **Primary Key**: A chosen candidate key.

Example:
- For Shipments(Vendor, Part, Qty), `{Vendor, Part}` is the primary key.

---

## 🧮 Boyce-Codd Normal Form (BCNF)

### Rule:
- Every non-trivial dependency must have a **candidate key** on the left side.

### ❌ BCNF Violation Example:
| Pizza | Topping     | ToppingType |
|-------|-------------|-------------|
| 1     | Mozarella   | Cheese      |
| 1     | Pepperoni   | Meat        |

FDs:
- `Pizza, ToppingType → Topping`
- `Topping → ToppingType`

But `Topping` is **not a key**, so `Topping → ToppingType` violates BCNF.

### ✅ Solution:

**R1 (Pizza, Topping)**  
**R2 (Topping, ToppingType)**

---

## 4️⃣ Fourth Normal Form (4NF)

Handles **multivalued dependencies**.

### ❌ Bad Example:

| Course   | Teacher       | Textbook              |
|----------|---------------|-----------------------|
| Physics  | Prof. Green   | Basic Mechanics       |
| Physics  | Prof. Green   | Principles of Optics  |

Here:
- One course can have many teachers.
- One course can have many books.
- These are unrelated — that’s a multivalued dependency.

### ✅ Solution (4NF):

**CT(Course, Teacher)**  
**CX(Course, Textbook)**

---

## 🧼 Summary Table

| Normal Form | Solves Problem of...                      |
|-------------|--------------------------------------------|
| 1NF         | Multi-valued cells                         |
| 2NF         | Partial dependency on composite key        |
| 3NF         | Transitive dependencies                    |
| BCNF        | Dependencies on non-candidate keys         |
| 4NF         | Unrelated multivalued dependencies         |


# 📐 Formal Query Languages – Beginner’s Guide with Examples

Formal query languages describe *what you want from the data* in a precise, mathematical way. These are not usually typed directly in applications but help you understand how databases “think.”

---

## 🧮 1. Relational Algebra (RA)

Relational Algebra is a **procedural** language — it shows **how** to compute the result.

### 🔧 Core Operators

| Operator       | Symbol         | SQL Equivalent               | Description & Example                             |
|----------------|----------------|-------------------------------|---------------------------------------------------|
| Selection      | σ (sigma)      | `WHERE`                      | Pick rows: `σ Dept='Physics'(Instructor)`        |
| Projection     | π (pi)         | `SELECT column1, column2...` | Choose columns: `π Name, ID(Instructor)`         |
| Union          | ∪              | `UNION`                      | Combine rows: `Instructor ∪ Student`             |
| Difference     | −              | `EXCEPT`                     | Subtract rows: `Instructor − TaughtPeople`       |
| Cartesian Prod | ×              | `JOIN` without condition     | All combinations: `Instructor × Student`         |
| Rename         | ρ (rho)        | `AS`                         | Rename a table or columns                        |

> **Example**:  
> Find instructors in 'Physics' with salary > 90000:  
> `π InstName, InstID (σ Dept='Physics' ∧ Salary > 90000 (Instructor))`

---

## 🌍 2. Domain Calculus (DC)

Domain calculus is **declarative** — it says **what** you want, not how to get it.

### ✍️ Syntax
```
{ <x1, x2, ... xn> | condition about x1 to xn }
```
Where `x1`, `x2`, … are **domain variables** (column values).

### 🧠 Example (same as above):
```text
{ <n, i> | ∃d,s ( <i, n, d, s> ∈ Instructor ∧ d = 'Physics' ∧ s > 90000 ) }
```

> Think of this as:  
> “Give me name and ID where there exist d, s such that (ID, name, dept, salary) is in Instructor and dept = 'Physics' and salary > 90000”

---

## 🧰 Translation Tips (RA ↔ SQL ↔ DC)

| Concept           | SQL                              | Relational Algebra                | Domain Calculus Example                      |
|------------------|-----------------------------------|-----------------------------------|----------------------------------------------|
| Projection       | `SELECT A,B`                      | `π A, B(R)`                       | `{<a,b> | ∃c ( <a,b,c> ∈ R )}`               |
| Selection        | `WHERE A > 3`                     | `σ A > 3 (R)`                     | `{<a,b,c> | <a,b,c> ∈ R ∧ a > 3}`            |
| Join             | `JOIN ON`                         | `R ⋈ S`                           | `{<a,c> | ∃b ( <a,b> ∈ R ∧ <b,c> ∈ S )}`     |
| Set Difference   | `EXCEPT`                          | `R − S`                           | `{<a,b,c> | <a,b,c> ∈ R ∧ <a,b,c> ∉ S}`      |
| Aggregation      | `GROUP BY`, `COUNT`, `SUM`        | `G COUNT(A)(σ cond(R))`          | Not expressible in basic DC                  |

---

## 🔂 Aggregation in Relational Algebra

```sql
SELECT COUNT(CourseID) FROM Course WHERE DeptName = 'Biology';
```

Relational Algebra:
```
G COUNT(CourseID)(σ DeptName = 'Biology' (Course))
```

---

## 🎯 Demo Problem (All 3 Ways)

**Query**: Get names and IDs of instructors in Physics with salary > 90000

**SQL**
```sql
SELECT InstName, InstID FROM Instructor
WHERE DeptName = 'Physics' AND Salary > 90000;
```

**Relational Algebra**
```
π InstName, InstID (σ DeptName = 'Physics' ∧ Salary > 90000 (Instructor))
```

**Domain Calculus**
```
{<n,i> | ∃d,s ( <i,n,d,s> ∈ Instructor ∧ d='Physics' ∧ s>90000 )}
```

---

## 🧩 Tips for Understanding Formal Queries

- **Relational Algebra** = a recipe (step-by-step)
- **Domain Calculus** = a rule (filtering by logic)
- Match **column positions** carefully in domain calculus.
- Use **π** for “columns you want,” and **σ** for “filters.”

---

If you'd like more practice problems or step-by-step breakdowns, I’m at your disposal, Mr. Elkjær.



# ⚡ Query Optimization – Simple Explanation with Examples

Query optimization is the process the **Database Management System (DBMS)** uses to **rearrange your query** into a faster version that returns the same result.

---

## 🧠 Why Optimize Queries?

Imagine this:
```sql
SELECT name FROM Instructor WHERE Dept = 'Physics';
```
The database could:
- Look at all instructors, then filter.
- Or filter first, then look at names.

Both give the same result — but one is faster.

---

## 🧩 What is an Execution Plan?

An **execution plan** is the DBMS’s internal plan of which steps to run, and in which order.

- Good plan = fast result
- Bad plan = same answer, but slow

The optimizer looks for **equivalent expressions** that do the same thing with fewer steps or less memory.

---

## 🌲 Example: Query Tree (page 5)

Find names of instructors in 'Comp. Sci.' along with the titles of the courses they teach:

**SQL:**
```sql
SELECT InstName, Title
FROM Instructor NATURAL JOIN Teaches NATURAL JOIN Course
WHERE DeptName = 'Comp. Sci.';
```

**Relational Algebra:**
```
π InstName, Title (σ DeptName='Comp. Sci.' (Instructor ⋈ Teaches ⋈ Course))
```

### Before Optimization Tree:
- Join 3 tables
- Then filter
- Then project columns

### After Optimization (page 16):
- **Push filters down** to Instructor (`Dept = 'Comp. Sci.'`)
- **Project early** to reduce data
- Then join the smaller results

---

## 🧱 Common Optimization Rules

### 1. Push `σ` (selections) down early
Instead of:
```
σ condition (R ⋈ S)
```
Do:
```
(σ condition(R)) ⋈ S
```
✔ Reduces the number of rows before the expensive join.

---

### 2. Push `π` (projections) early
Get rid of columns you don’t need ASAP.

Instead of:
```
π columns (R ⋈ S)
```
Do:
```
(π only_needed_from_R(R)) ⋈ (π only_needed_from_S(S))
```

---

### 3. Replace `×` + `σ` with a `⋈`
Instead of:
```
σ A = B (R × S)
```
Use:
```
R ⋈ S ON A = B
```

✔ Avoids massive temporary results from Cartesian Products.

---

### 4. Break compound filters into separate ones
```
σ A > 5 ∧ B = 3 (R)
```
Can be:
```
σ A > 5 (σ B = 3 (R))
```

✔ Each smaller filter can be pushed and optimized independently.

---

### 5. Apply filters before `UNION`, `INTERSECT`, or `DIFFERENCE`
Instead of:
```
σ condition (R ∪ S)
```
Use:
```
σ condition (R) ∪ σ condition (S)
```

✔ More efficient, especially if condition excludes many rows.

---

## 🧪 Summary Table of Tricks

| Optimization                      | What it does                                 |
|----------------------------------|----------------------------------------------|
| Push `σ` down before joins       | Reduces row count early                      |
| Push `π` down before joins       | Removes unused columns early                 |
| Replace `×` + filter with `⋈`    | Avoids big intermediate results              |
| Split compound selections        | Enables early filtering                      |
| Push filters before set ops      | Smaller unions, intersections, etc.          |

---

## 🧠 How Do You Use This?

You don’t write the optimized version. The DBMS does.

BUT: If you understand **how** it works, you can:
- Write **better SQL** (fewer joins, only needed columns)
- Read and debug **execution plans**
- Avoid mistakes that make queries slow

Would you like a live example in MySQL or SQL Server to see the actual execution plan, Sir?



# 🗂️ Indexing & Hashing – Beginner's Guide with Clear Examples

Indexes and hashing help databases **find data faster** — like an address book helps you avoid reading every page to find a name.

---

## 📁 What is a File in a Database?

- A **file** is a group of records.
- A **record** is a row (e.g. one instructor).
- Each record has **fields** (columns): `InstID`, `InstName`, `Salary`, etc.
- Files are stored in blocks on disk.

---

## 🧩 File Organization Types

### 1. Heap File
- Records stored anywhere there’s space.
- ✅ Easy for inserts; ❌ slow for searching.

### 2. Sequential File
- Sorted by a key (e.g. `InstID`).
- ✅ Fast for range queries, ❌ expensive to maintain order.

### 3. Hash File
- A hash function chooses which block to store a record in.
- ✅ Fast exact-match lookups.

---

## 🔎 Why Use Indexes?

Searching in a file without an index means scanning all blocks — very slow.

Instead, we use:

- **Index Files**: Store `(Search Key, Pointer)` pairs.
- Or **Hash Functions**: Compute which block holds the record.

---

## 📊 Index Types Explained

### 🥇 Primary vs Secondary
| Type       | Based On          | Determines Order? | How Many Per Table |
|------------|-------------------|-------------------|---------------------|
| Primary    | Primary Key       | Yes               | One                 |
| Secondary  | Any other column  | No                | Many allowed        |

### 📏 Dense vs Sparse
| Type   | Description                         |
|--------|-------------------------------------|
| Dense  | One index entry per data record     |
| Sparse| One entry per block (usually 1st row)|

---

## 🔍 Example: Find Instructor with ID 15151

**Dense Index (Page 14)**  
1. Use binary search in index (ordered).
2. Follow pointer to block with record.

**Sparse Index (Page 19)**  
1. Find the largest index key ≤ 15151.
2. Go to that block, then scan linearly.

---

## ✏️ SQL: Creating Indexes
```sql
CREATE INDEX idx_name ON Instructor(InstName);
CREATE INDEX idx_dept ON Instructor(DeptName) USING BTREE;
CREATE INDEX idx_salary ON Instructor(Salary) USING HASH;
```

---

## 🌲 Multilevel Indexes

Too big for memory? Use:
- **Inner index**: The full list.
- **Outer index**: Index of the inner index.

➡️ Works like a dictionary with tabs.

---

## 🌴 B+ Trees (Page 30–35)

- Used by databases for fast lookups.
- Keep keys **sorted**, with a **balanced** tree.
- ✅ Efficient for insert, delete, and range queries.

```text
Internal Node
  /   |   Leaf Nodes → (Dense index with record pointers)
```

Lookups are fast: e.g. log₅(n) depth if 5 children per node.

---

## 🪣 Hashing (Page 38–41)

Hashing stores data using a **math function** on the search key.

```sql
-- Hashing by department name
DeptName = 'Music'
Hash('Music') % 8 = 1 → go to bucket 1
```

- Each bucket holds records.
- Overflow handled by **overflow buckets**.

---

## 🧠 Choosing Between Index and Hash

| Use Case                       | Best Option     |
|--------------------------------|-----------------|
| Exact match (ID = 1234)        | Hashing         |
| Range queries (Salary > 50000) | B+ Tree Index   |
| Frequent sorting               | Ordered Index   |

---

## 🧪 Summary of Key Concepts

- **Index** = mini map pointing to data.
- **Primary index** follows table's sort order.
- **Secondary index** can be used for other lookups.
- **Dense** index = all records.
- **Sparse** index = one per block.
- **Hashing** = compute location directly.
- **B+ Trees** = always balanced and fast for ranges.

---

Shall I prepare this as a downloadable `.md` file for your personal archive, Mr. Elkjær?
