# PostgreSQL — Teaching Lesson

> This lesson assumes **zero** prior database experience. Every term is defined
> before it's used. Read it in order — Section 8 assumes Section 4 already makes
> sense, and so on. Type every example yourself; reading SQL is not the same skill
> as writing it.

## Objective
Understand PostgreSQL deeply: how it stores data, enforces rules, connects to
applications, and stays fast as it grows.

---

## 1. What PostgreSQL is, and why it exists

**Definition:** PostgreSQL (often called "Postgres") is a **relational database
management system (RDBMS)** — a standalone program that stores data in **tables**
(rows and columns), enforces rules about what that data is allowed to look like, and
lets other programs (like a FastAPI backend) read and write that data safely, even
when many programs are doing so at the same time.

**Why not just use a file, or a Python list, or a dictionary?** Every method you've
used before this — a Python list, a JSON file — has the same fundamental limits:
- **No shared access.** Two programs writing to the same JSON file at the same time
  can corrupt it. A database is specifically designed to let many clients read and
  write safely, simultaneously.
- **No enforced rules.** Nothing stops a JSON file from ending up with a student
  missing a `score` field, or a score that's `"eighty"` instead of `80`. A database
  can refuse bad data at the storage layer itself, not just in your application code.
- **No efficient search.** Finding "every order over $100 placed last week" in a
  million-row JSON file means loading everything into memory and checking each one
  by hand, in your own code. A database can find exactly those rows without reading
  the rest, if it's set up correctly (Section 12).
- **No relationships without duplication.** Representing "this order belongs to this
  customer" in JSON usually means nesting or duplicating customer data inside every
  order. A relational database expresses this with a small reference (a **foreign
  key**, Section 6) instead, with no duplication.

**Why PostgreSQL specifically, and not SQLite?** SQLite (which you may have used for
practice) is a real database too, but it runs *inside* your application process, as
a single file on disk — perfect for learning, small apps, or a single-user tool. It
has real limits for anything bigger: limited concurrent write support, no built-in
network access from other machines, and a much smaller feature set (no real user
permissions, weaker support for some data types and constraints). PostgreSQL is a
**server** — a separate, standalone program your application *connects to*, over a
network — built from the ground up for many simultaneous users, strict data
integrity, and large datasets. It's one of the two or three most widely used
databases in professional backend work today, alongside MySQL.

---

## 2. Installing PostgreSQL and connecting with `psql`

### Installing

- **Mac:** `brew install postgresql@16` (using Homebrew), or download Postgres.app
  from postgresapp.com for a GUI-driven install.
- **Windows:** download the installer from postgresql.org/download/windows — it
  includes `psql` and pgAdmin (a GUI tool) together.
- **Linux (Debian/Ubuntu):** `sudo apt install postgresql postgresql-contrib`

**Or skip local installation entirely** and use a free cloud instance from
**Neon** (neon.tech) or **Supabase** (supabase.com) — both give you a real Postgres
database in under a minute, with a connection string, and nothing to install. This is
a completely legitimate way to work through this whole lesson; every example below
works identically either way.

### Connecting with `psql`

**Definition:** `psql` is PostgreSQL's official command-line client — the primary
tool you use to type SQL directly and see results immediately, and the tool this
entire lesson uses for every example.

```bash
psql -h localhost -U postgres -d postgres
```
**Reading each flag:** `-h` (host — where the server is; `localhost` for a local
install), `-U` (username), `-d` (which database to connect to). If you're using a
cloud provider like Neon, they give you a full connection string instead:
```bash
psql "postgresql://username:password@ep-xxxx.region.aws.neon.tech/dbname?sslmode=require"
```
Once connected, your prompt changes to something like `postgres=#` — you're now
typing SQL directly against a live database.

**Useful `psql` meta-commands** (these start with `\`, not `SELECT` — they're
`psql`-specific shortcuts, not SQL itself):
```
\l          list all databases
\c dbname   connect to a different database
\dt         list tables in the current database
\d table    describe a table's columns and constraints
\q          quit psql
```

**Checkpoint:** you can run `psql`, see a `dbname=#` prompt, and run `\dt` (even if it
shows no tables yet — that's expected on a fresh database).

---

## 3. Databases, schemas, and tables — the three levels of organisation

**Definition:** A PostgreSQL **server** (one running installation) can hold many
**databases** — separate, isolated collections of data, normally one per
application. Inside one database, tables are further organised into **schemas**
(think: folders for tables) — most projects use the default schema, called
`public`, and never need more than that, but it's worth knowing the word exists.

```sql
CREATE DATABASE school;
```
```bash
\c school
```
Everything from here on happens *inside* one connected database — a table you
create only exists within that specific database, not shared across all of them.

---

## 4. Creating your first table, and PostgreSQL's data types

**Definition:** `CREATE TABLE` defines a table's shape: its columns, and each
column's **data type** — what kind of value that column is allowed to hold.

```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    score INTEGER,
    gpa NUMERIC(3, 2),
    is_active BOOLEAN DEFAULT TRUE,
    enrolled_at TIMESTAMP DEFAULT NOW()
);
```

### The core data types, one at a time

- **`SERIAL`** — an auto-incrementing integer, commonly used for a `PRIMARY KEY`
  (Section 5). Postgres assigns `1`, `2`, `3`, ... automatically; you never set it
  yourself. (Modern Postgres also offers `GENERATED ALWAYS AS IDENTITY` as a more
  standards-compliant alternative — functionally very similar; `SERIAL` remains
  extremely common and is what you'll see in most existing code.)
- **`VARCHAR(n)`** — variable-length text, capped at `n` characters. Use this when
  there's a genuine, sensible maximum length (a name, an email).
- **`TEXT`** — variable-length text with **no** length limit. Use this for anything
  that could be long and unpredictable (a blog post body, a comment).
- **`INTEGER`** — a whole number. (`SMALLINT` and `BIGINT` exist for smaller/larger
  ranges when you specifically need them.)
- **`NUMERIC(precision, scale)`** — an exact decimal number — `NUMERIC(3, 2)` means
  up to 3 total digits, 2 of them after the decimal point (so, `0.00` to `9.99`).
  **Always use `NUMERIC` for money or anything requiring exact decimal precision** —
  never `FLOAT`/`REAL`, which store approximate binary representations that can
  introduce small rounding errors (`0.1 + 0.2` famously doesn't equal exactly `0.3`
  in floating-point math).
- **`BOOLEAN`** — `TRUE`, `FALSE`, or `NULL` (unknown/not set).
- **`TIMESTAMP`** — a date and time, with no timezone information attached.
  **`TIMESTAMPTZ`** (timestamp with time zone) is usually the better default for
  real applications — it stores the instant unambiguously, correctly interpreted
  when displayed in any timezone.
- **`DATE`** — just a calendar date, no time component.
- **`JSONB`** — stores a JSON document directly in a column, *and* lets you query
  inside it with SQL. Useful for genuinely flexible, semi-structured data that
  doesn't fit neatly into columns — but reach for it deliberately, not as a way to
  avoid designing proper columns and tables.

### `NULL` — a value's third possible state

**Definition:** Every column, unless restricted by `NOT NULL`, can hold `NULL` — a
special marker meaning "no value / unknown," distinct from `0`, `""` (empty
string), or `FALSE`. This trips up every beginner at least once: `NULL` is not
"equal" to anything, including itself.
```sql
SELECT NULL = NULL;      -- returns NULL, not TRUE!
SELECT score = NULL;      -- always NULL, never TRUE or FALSE
SELECT score IS NULL;     -- the correct way to check for NULL
SELECT score IS NOT NULL; -- and its opposite
```
**Always use `IS NULL` / `IS NOT NULL`, never `= NULL`.**

**Checkpoint:** run the `CREATE TABLE students` statement above, then `\d students`
to see PostgreSQL's own description of the table you just created, including which
type and constraints it recorded for each column.

---

## 5. Constraints — the database enforcing your rules

**Definition:** A constraint is a rule PostgreSQL enforces automatically, at the
storage layer — the database itself rejects any `INSERT` or `UPDATE` that would
violate one, regardless of what your application code does or forgets to check.

- **`PRIMARY KEY`** — a column (or combination of columns) that uniquely identifies
  each row. Automatically implies both `NOT NULL` and `UNIQUE`. Every table should
  have one.
- **`NOT NULL`** — this column can never be left empty.
- **`UNIQUE`** — no two rows may share the same value in this column (`NULL` values
  are an exception — multiple `NULL`s are allowed unless you also add `NOT NULL`).
- **`CHECK`** — a custom condition every row must satisfy:
  ```sql
  score INTEGER CHECK (score >= 0 AND score <= 100)
  ```
  Try to insert a `score` of `150`, and Postgres rejects it outright — this is the
  same rule you might enforce in Pydantic (if you've done any FastAPI work), but
  enforced here even if some *other*, careless piece of code tries to write directly
  to the database, bypassing your API entirely.
- **`DEFAULT`** — the value used automatically when none is provided:
  ```sql
  is_active BOOLEAN DEFAULT TRUE
  ```
- **`FOREIGN KEY`** — covered fully in Section 6, since it needs a second table to
  make sense of.

**What happens when a constraint is violated:**
```sql
INSERT INTO students (name, score) VALUES (NULL, 91);
-- ERROR: null value in column "name" violates not-null constraint
```
Postgres refuses the entire statement — nothing gets partially written. This is a
deliberate safety guarantee, not an accident (see **Section 13, Transactions**, for
more on this guarantee applied to multi-statement operations).

---

## 6. Foreign keys and relationships between tables

**Definition:** A foreign key is a column in one table that stores the primary key
of a row in another table, expressing "this row relates to that row" — without
duplicating that row's data.

```sql
CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL
);

CREATE TABLE enrollments (
    id SERIAL PRIMARY KEY,
    student_id INTEGER NOT NULL REFERENCES students(id),
    course_id INTEGER NOT NULL REFERENCES courses(id),
    enrolled_at TIMESTAMP DEFAULT NOW()
);
```
**`REFERENCES students(id)`** is the foreign key declaration — it tells Postgres
`student_id` must always match a real `id` that exists in `students`. Try inserting
an `enrollments` row with a `student_id` of `9999` when no such student exists:
```sql
INSERT INTO enrollments (student_id, course_id) VALUES (9999, 1);
-- ERROR: insert or update on table "enrollments" violates foreign key constraint
```
Rejected automatically — the database itself guarantees your data can never point to
something that doesn't exist, a guarantee no amount of careful application code can
fully replace on its own.

**`ON DELETE` behaviour** — what should happen to an `enrollment` if the `student`
it references gets deleted?
```sql
student_id INTEGER NOT NULL REFERENCES students(id) ON DELETE CASCADE
```
- **`ON DELETE CASCADE`** — automatically delete dependent rows too (deleting a
  student deletes their enrollments).
- **`ON DELETE RESTRICT`** (the default) — refuse to delete the student at all while
  enrollments referencing them still exist.
- **`ON DELETE SET NULL`** — set `student_id` to `NULL` on the dependent row instead
  (only valid if that column allows `NULL`).

**Choose deliberately** — `CASCADE` is convenient but can silently delete far more
data than you intended if you're not careful about what depends on what.

This particular shape — `enrollments` linking `students` and `courses` — is called a
**join table** (or **junction table**), and it's how you represent a **many-to-many**
relationship: one student can enroll in many courses, and one course can have many
students. Compare this to a simpler **one-to-many** relationship (like one course
having many students directly, if a student could only ever take one course) —
that would just need a single `course_id` foreign key directly on `students`, no
separate join table required.

---

## 7. `SELECT`, `WHERE`, `ORDER BY`, `LIMIT` — reading data, in depth

```sql
INSERT INTO students (name, email, score, gpa) VALUES
    ('Ada', 'ada@example.com', 91, 3.8),
    ('Kofi', 'kofi@example.com', 68, 2.4),
    ('Zara', 'zara@example.com', 84, 3.5);
```

**`SELECT`** — choose which columns come back:
```sql
SELECT * FROM students;                -- every column
SELECT name, score FROM students;      -- only these two
SELECT name, score * 1.1 AS boosted FROM students;   -- computed column, aliased
```

**`WHERE`** — filter which rows come back, evaluated per row, before anything else:
```sql
SELECT * FROM students WHERE score >= 80;
SELECT * FROM students WHERE score >= 80 AND is_active = TRUE;
SELECT * FROM students WHERE name IN ('Ada', 'Kofi');
SELECT * FROM students WHERE score BETWEEN 70 AND 90;
SELECT * FROM students WHERE name LIKE 'A%';        -- starts with "A"
SELECT * FROM students WHERE email ILIKE '%EXAMPLE%'; -- case-insensitive match
```
**`LIKE` vs. `ILIKE`:** `LIKE` is case-sensitive; `ILIKE` (Postgres-specific) isn't.
`%` matches any sequence of characters; `_` matches exactly one character.

**`ORDER BY`** — sort the results:
```sql
SELECT * FROM students ORDER BY score DESC;              -- highest first
SELECT * FROM students ORDER BY score DESC, name ASC;    -- tie-break by name
```

**`LIMIT` / `OFFSET`** — cap how many rows come back, and skip some — the
foundation of pagination in any real application:
```sql
SELECT * FROM students ORDER BY id LIMIT 10;             -- first 10
SELECT * FROM students ORDER BY id LIMIT 10 OFFSET 10;   -- the next 10 ("page 2")
```

**`DISTINCT`** — remove duplicate rows from the result:
```sql
SELECT DISTINCT is_active FROM students;
```

---

## 8. `UPDATE` and `DELETE` — changing data, safely

```sql
UPDATE students SET score = 95 WHERE id = 1;
DELETE FROM students WHERE id = 2;
```
**The single most important habit in this entire lesson: never run `UPDATE` or
`DELETE` without a `WHERE` clause, and always check the `WHERE` clause carefully
before running either.** Without one, both statements apply to **every row in the
table**, no confirmation asked:
```sql
UPDATE students SET score = 0;   -- resets EVERY student's score. Every single one.
```
**A safe habit:** write and run the equivalent `SELECT` first, to see exactly which
rows would be affected, before switching it to `UPDATE`/`DELETE`:
```sql
SELECT * FROM students WHERE score < 50;    -- check first...
UPDATE students SET score = 50 WHERE score < 50;   -- ...then act
```

---

## 9. `JOIN`s — combining data from multiple tables

**Definition:** A `JOIN` combines rows from two (or more) tables based on a
matching condition, usually a foreign key relationship — this is the mechanism that
makes splitting data across multiple tables (Section 6) actually useful for reading,
not just for storage.

Set up some sample data first:
```sql
INSERT INTO courses (title) VALUES ('Mathematics'), ('Physics');
INSERT INTO enrollments (student_id, course_id) VALUES (1, 1), (1, 2), (2, 1);
```

### `INNER JOIN` (often just written `JOIN`)
Returns only rows that have a match in **both** tables.
```sql
SELECT students.name, courses.title
FROM enrollments
JOIN students ON enrollments.student_id = students.id
JOIN courses ON enrollments.course_id = courses.id;
```
Ada appears twice (Mathematics and Physics); Zara doesn't appear at all, since she
has no enrollments — that row simply has no match, so an inner join excludes it.

### `LEFT JOIN`
Returns **every** row from the left (first-named) table, with matching data from the
right table where it exists, and `NULL`s where it doesn't.
```sql
SELECT students.name, courses.title
FROM students
LEFT JOIN enrollments ON students.id = enrollments.student_id
LEFT JOIN courses ON enrollments.course_id = courses.id;
```
Now Zara appears too, with `NULL` for `title` — this is the query shape you reach
for whenever you want "everything from table A, whether or not it has a match in
table B" (a very common real need — "every course, even ones with zero students,"
for example).

### `RIGHT JOIN` and `FULL JOIN`
`RIGHT JOIN` is the mirror image of `LEFT JOIN` (every row from the right table
instead). `FULL JOIN` (or `FULL OUTER JOIN`) keeps every row from **both** tables,
matched where possible, `NULL` where not. In practice, most developers write
everything as a `LEFT JOIN` with tables ordered deliberately, and rarely reach for
`RIGHT JOIN` — but recognise the term when you see it.

**A mental model that helps:** picture two overlapping circles (a Venn diagram).
`INNER JOIN` is just the overlap. `LEFT JOIN` is the entire left circle (overlap
included). `FULL JOIN` is both circles entirely.

---

## 10. `GROUP BY`, `HAVING`, and aggregate functions

**Definition:** `GROUP BY` collapses rows sharing the same value in a column into
one row per distinct value, so aggregate functions can compute a result **per
group**.

```sql
SELECT course_id, COUNT(*) AS enrolled_count
FROM enrollments
GROUP BY course_id;
```

**The core aggregate functions:** `COUNT(*)`, `SUM(column)`, `AVG(column)`,
`MIN(column)`, `MAX(column)`.
```sql
SELECT AVG(score), MIN(score), MAX(score) FROM students;
```

**The rule that catches everyone once:** every column in your `SELECT` list must
either appear in `GROUP BY`, or be wrapped in an aggregate function. Postgres
rejects a query that tries to select an ungrouped, unaggregated column alongside a
`GROUP BY`.

**`HAVING`** — filters *groups*, after aggregation (unlike `WHERE`, which filters
individual rows *before* grouping):
```sql
SELECT course_id, COUNT(*) AS enrolled_count
FROM enrollments
GROUP BY course_id
HAVING COUNT(*) >= 2;
```

**Combining a `JOIN` with `GROUP BY`** — a genuinely common, useful real-world
query:
```sql
SELECT courses.title, COUNT(enrollments.id) AS enrolled_count
FROM courses
LEFT JOIN enrollments ON courses.id = enrollments.course_id
GROUP BY courses.id, courses.title
ORDER BY enrolled_count DESC;
```
Notice the `LEFT JOIN` here — this ensures a course with **zero** enrollments still
shows up, with a count of `0`, rather than silently disappearing (which an `INNER
JOIN` would do, since a course with no enrollments has nothing to match against in
`enrollments`).

---

## 11. Subqueries and CTEs — queries built from other queries

**Definition:** A subquery is a `SELECT` nested inside another SQL statement, used
where its result — a value, a list of values, or a whole table-shaped result — is
needed to complete the outer query.

```sql
-- Students with an above-average score
SELECT name, score FROM students
WHERE score > (SELECT AVG(score) FROM students);
```
```sql
-- Students enrolled in "Mathematics"
SELECT name FROM students
WHERE id IN (
    SELECT student_id FROM enrollments
    JOIN courses ON enrollments.course_id = courses.id
    WHERE courses.title = 'Mathematics'
);
```

**CTEs (Common Table Expressions)** — a named, temporary result set, defined with
`WITH`, that you can reference like a table within the rest of the query. They exist
mainly for **readability**: breaking a complex query into clearly-named, sequential
steps, rather than deeply nesting subqueries inside each other.
```sql
WITH course_counts AS (
    SELECT course_id, COUNT(*) AS enrolled_count
    FROM enrollments
    GROUP BY course_id
)
SELECT courses.title, course_counts.enrolled_count
FROM courses
JOIN course_counts ON courses.id = course_counts.course_id
WHERE course_counts.enrolled_count >= 2;
```
This computes exactly the same thing a nested subquery could, but reads top-to-bottom
as "first compute this, then use it here" — genuinely easier to follow once queries
get more than one or two steps deep.

---

## 12. Indexes and `EXPLAIN` — understanding performance

**Definition:** An index is a separate, sorted data structure PostgreSQL maintains
alongside a table, letting it locate matching rows directly instead of checking
every row in the table one by one — the same idea as a book's index letting you jump
to a page instead of reading the whole book to find one topic.

```sql
CREATE INDEX idx_students_email ON students(email);
```
A `PRIMARY KEY` and a `UNIQUE` constraint each create an index automatically — you've
had indexes since Section 4 without necessarily noticing. Today's is a deliberate,
extra one, on a column you expect to filter or join on often but that isn't already
covered.

**`EXPLAIN`** — see how Postgres actually plans to run a query, instead of running
it:
```sql
EXPLAIN SELECT * FROM students WHERE email = 'ada@example.com';
```
Before the index existed, look for **`Seq Scan`** in the output — a full sequential
scan, checking every row. After creating the index, re-run the same `EXPLAIN` and
look for **`Index Scan`** instead — confirming Postgres is now using it.

**`EXPLAIN ANALYZE`** goes further — it actually *runs* the query and reports real
timing alongside the plan, useful once you're comparing genuine performance, not just
the intended strategy:
```sql
EXPLAIN ANALYZE SELECT * FROM students WHERE email = 'ada@example.com';
```

**When to add an index, and when not to:** add one on columns you frequently
`WHERE`, `JOIN`, or `ORDER BY` on, especially as a table grows past a few thousand
rows. **Don't** index every column reflexively — every index speeds up matching
reads but slightly slows down every `INSERT`/`UPDATE`/`DELETE` (since the index
itself has to be kept up to date), and uses additional disk space. Index
deliberately, based on your actual query patterns, not by default.

---

## 13. Transactions — all-or-nothing, and `ACID`

**Definition:** A transaction is a group of one or more SQL statements executed as a
single, indivisible unit — either **all** of them succeed and are saved together, or
**none** of them are, even if the failure happens partway through.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```
**Why this matters concretely:** imagine transferring money between two accounts —
subtract from one, add to the other. If the server crashed *between* those two
statements without a transaction wrapping them, money would simply vanish — deducted
from one account, never credited to the other. `BEGIN`/`COMMIT` guarantees that can
never happen: either both statements take effect, or neither does.

**`ROLLBACK`** undoes everything since `BEGIN`, instead of committing it:
```sql
BEGIN;
DELETE FROM students WHERE score < 50;
-- wait, that's not what I meant to do!
ROLLBACK;
```
Your `students` table is completely unaffected — as if the `DELETE` never happened.
This is an extremely useful safety net while you're experimenting directly in
`psql`: wrap anything you're unsure about in `BEGIN`, check the result with a
`SELECT`, and `ROLLBACK` if it's wrong, `COMMIT` if it's right.

**`ACID`** is the acronym describing the guarantees a transaction gives you:
- **Atomicity** — all-or-nothing, exactly as shown above.
- **Consistency** — a transaction can never leave the database violating a
  constraint (Section 5) — Postgres checks constraints before allowing a `COMMIT`.
- **Isolation** — transactions running at the same time, from different connections,
  don't see each other's uncommitted changes.
- **Durability** — once a transaction is committed, it survives — even a server
  crash immediately afterward won't lose it.

You've actually been relying on all four properties, invisibly, in every single
statement you've run so far in this lesson — PostgreSQL wraps every individual
statement in an implicit transaction automatically, even when you don't write
`BEGIN`/`COMMIT` yourself. `BEGIN`/`COMMIT`/`ROLLBACK` matter specifically when you
need **multiple** statements to succeed or fail *together*.

---

## 14. Users, roles, and permissions

**Definition:** A role in PostgreSQL represents a user (or a group of users) that
can own database objects and hold permissions — what they're allowed to do, and to
what.

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'a-strong-password';
GRANT CONNECT ON DATABASE school TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON students TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON courses TO app_user;
```
**Why this matters for a real application:** your backend application should
usually connect using a role with exactly the permissions it needs — not the
database's all-powerful superuser account. This limits the damage a bug, a leaked
credential, or a compromised server can do — an application role with only
`SELECT`/`INSERT`/`UPDATE`/`DELETE` on specific tables can't, for instance,
accidentally (or maliciously) drop a table it was never granted permission to touch.

```sql
REVOKE DELETE ON students FROM app_user;   -- take a permission back
\du                                         -- list all roles (psql meta-command)
```
This is the same underlying principle you may already know from FastAPI-level
application code (a "role" field, checked in your own logic) — here it's enforced
one layer deeper, by the database itself, independent of anything your application
code does or forgets to check.

---

## 15. Connecting from Python

### With `psycopg2` directly

```bash
pip install psycopg2-binary
```
```python
import psycopg2

conn = psycopg2.connect(
    host="localhost",
    dbname="school",
    user="app_user",
    password="a-strong-password",
)
cursor = conn.cursor()

cursor.execute("SELECT name, score FROM students WHERE score >= %s", (80,))
rows = cursor.fetchall()
for row in rows:
    print(row)

conn.commit()
cursor.close()
conn.close()
```
**Notice `%s` placeholders, not an f-string** — this is `psycopg2`'s version of the
same rule from earlier SQL lessons: never build a query string by interpolating
values directly; always let the driver substitute them safely. Building SQL with an
f-string (`f"WHERE score >= {value}"`) opens the door to **SQL injection** — a
malicious or malformed value could alter the query's actual structure, not just its
data.

### With SQLAlchemy (the higher-level, more common approach)

```python
from sqlalchemy import create_engine, text

engine = create_engine("postgresql://app_user:a-strong-password@localhost/school")

with engine.connect() as connection:
    result = connection.execute(
        text("SELECT name, score FROM students WHERE score >= :min_score"),
        {"min_score": 80},
    )
    for row in result:
        print(row)
```
SQLAlchemy's ORM layer (models, sessions, `relationship()`) sits on top of exactly
this kind of connection, generating SQL like everything in this lesson behind the
scenes — understanding the raw SQL first, as you've just done, is what makes the ORM
layer's behavior predictable instead of a black box.

---

## 16. Backing up and restoring with `pg_dump`

**Definition:** `pg_dump` exports a database's structure and data into a file;
`psql` (or `pg_restore` for a different output format) can load that file back into
a fresh database — this is how you back up real data, and how you move a database
from one environment to another.

```bash
pg_dump -h localhost -U postgres school > school_backup.sql
```
Restoring into a fresh, empty database:
```bash
createdb school_restored
psql -h localhost -U postgres -d school_restored -f school_backup.sql
```
Real production systems automate this on a schedule — cloud providers like Neon
handle continuous backups for you automatically, but it's worth knowing exactly what
`pg_dump` does under the hood, since you'll eventually need it for a manual export,
a migration between providers, or restoring from a mistake.

---

## Common mistakes to watch for (a consolidated reference)
- **`UPDATE`/`DELETE` without a `WHERE` clause** — Section 8. Always check with
  `SELECT` first.
- **Comparing with `= NULL` instead of `IS NULL`** — Section 4. `NULL` is never
  "equal" to anything.
- **Building SQL by string-interpolating values, instead of using placeholders** —
  Section 15. This is the root cause of SQL injection.
- **Selecting an ungrouped, unaggregated column alongside `GROUP BY`** — Section 10.
  Postgres will reject the query outright; every selected column must be grouped or
  aggregated.
- **Using `FLOAT`/`REAL` for money or exact decimals** — Section 4. Use `NUMERIC`.
- **An `INNER JOIN` silently dropping rows with no match** when you actually wanted
  everything included — Section 9. Reach for `LEFT JOIN` when "everything from
  table A" is the actual requirement.
- **Indexing every column "just in case"** — Section 12. Indexes cost write
  performance and storage; add them deliberately, based on real query patterns.
- **Running your application as the database's superuser** — Section 14. Create a
  scoped role with only the permissions the application actually needs.


