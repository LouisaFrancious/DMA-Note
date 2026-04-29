# Data Management and Analysis 期末复习 Time Schedule

考试日期：2026-05-04  
根据 `finalnote.ipynb`：50% 是 T/F、选择题、填空题；50% 是 PyMongo/MongoDB 和 SQL 编码题。  
重点提醒：MongoDB 的 coding question 会通过 PyMongo 出题。Graph database 不考。

## Notion 风格 Time Schedule

用 checkbox 当状态；下面每个 topic 都是可以展开的复习页，里面有定义、代码模板、易错点、错题答疑。

| 日期 | 主目标 | Topics | 状态 |
|---|---|---|---|
| 2026-04-27 | SQL 基础 | Basic SQL, strings, arrays, dates, conditionals | [ ] |
| 2026-04-28 | 聚合专题 | SQL aggregation, `GROUP BY`, `HAVING`, aggregate 易错题 | [ ] |
| 2026-04-29 | 表关系专题 | Joins, database design, keys, constraints | [ ] |
| 2026-04-30 | 难点 SQL | Subqueries, window functions, aggregate functions as window functions | [ ] |
| 2026-05-01 | 性能专题 | `EXPLAIN ANALYZE`, adding indexes | [ ] |
| 2026-05-02 | Python + Postgres | `psycopg2`, SQLAlchemy patterns | [ ] |
| 2026-05-03 | MongoDB | `find`, aggregation pipeline, PyMongo, Mongo design | [ ] |
| 2026-05-04 | 考前总复习 | 一页 cheat sheet，重做错题，不开新 topic | [ ] |

## 可展开 Topic Pages

<details>
<summary>[ ] 1. Basic SQL</summary>

### 定义
Basic SQL means reading and filtering relational table data with `SELECT`, `FROM`, `WHERE`, `ORDER BY`, `LIMIT`, aliases, calculated fields, and `DISTINCT`.

Execution order to memorize:

```text
FROM -> JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT
```

### 代码模板

```sql
SELECT title,
       budget,
       gross,
       ROUND(((gross - budget) / budget)::numeric, 2) AS roi
FROM movie
WHERE budget > 0
ORDER BY roi DESC
LIMIT 10;
```

### 易错点
- `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation.
- `ORDER BY` can usually use an alias like `roi`, because it happens after `SELECT`.
- Integer division can surprise you. Cast to `numeric` when calculating ratios.
- `SELECT DISTINCT first` removes duplicate first names, not duplicate people.

### 错题答疑
Q: Why does `WHERE roi > 1` fail when `roi` is created in `SELECT`?  
A: `WHERE` runs before `SELECT`, so `roi` does not exist yet. Use the full expression, a subquery, or `HAVING` if it is aggregate-related.

</details>

<details>
<summary>[ ] 2. Aggregation</summary>

### 定义
Aggregation summarizes multiple rows into one value or one row per group. Common functions: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

### 代码模板

```sql
SELECT name,
       ROUND(AVG(score), 2) AS avg_score,
       COUNT(*) AS num_rows
FROM student
GROUP BY name
HAVING AVG(score) > 80
ORDER BY avg_score DESC;
```

### 易错点
- Every non-aggregated column in `SELECT` must appear in `GROUP BY`.
- `COUNT(*)` counts rows; `COUNT(column)` ignores `NULL`.
- You usually cannot nest aggregates directly, such as `AVG(COUNT(*))`, without a subquery.
- `WHERE AVG(score) > 80` is wrong; use `HAVING AVG(score) > 80`.

### 错题答疑
Q: Why does `SELECT name, MAX(score) FROM student;` fail?  
A: SQL does not know which `name` to show with the max score unless you group by `name` or use a subquery/window function to pick the matching row.

</details>

<details>
<summary>[ ] 3. Joins</summary>

### 定义
Joins combine rows from multiple tables. The join condition usually connects a foreign key to a primary key.

### 代码模板

```sql
SELECT m.title, g.name AS genre
FROM movie AS m
INNER JOIN genre AS g
    ON m.genre_id = g.genre_id;
```

Self join example from the notes:

```sql
SELECT a.movie_id, a.title, b.movie_id, b.title, a.year
FROM movie AS a
INNER JOIN movie AS b ON a.year = b.year
WHERE a.movie_id < b.movie_id
ORDER BY a.year, a.movie_id, b.movie_id;
```

### 易错点
- `CROSS JOIN` creates every possible pair. It can explode row count.
- `INNER JOIN` keeps only matching rows.
- `LEFT JOIN` keeps all rows from the left table, even if the right side is missing.
- In self joins, use aliases like `a` and `b`.
- Use `a.movie_id < b.movie_id` instead of `!=` when you want unique pairs without reversed duplicates.

### 错题答疑
Q: Why do I get each movie pair twice in a self join?  
A: `a.movie_id != b.movie_id` returns both A-B and B-A. Use `a.movie_id < b.movie_id`.

</details>

<details>
<summary>[ ] 4. Database Design</summary>

### 定义
Database design decides what tables exist, what columns they have, and how tables relate through primary keys, foreign keys, and constraints.

### 代码模板

```sql
CREATE TABLE web_user (
    web_user_id serial PRIMARY KEY,
    username varchar(8) NOT NULL UNIQUE,
    email text
);

CREATE TABLE address (
    address_id serial PRIMARY KEY,
    street text,
    city text,
    state varchar(2),
    zip text,
    web_user_id integer REFERENCES web_user(web_user_id)
);
```

### 易错点
- Primary key means unique and not null.
- Natural keys can change; surrogate keys like `serial` are often safer.
- Foreign keys protect references, so deleting a parent row may fail unless you handle the child rows or use `ON DELETE CASCADE`.
- Use constraints to prevent bad data, not just application logic.

### 错题答疑
Q: Why can dropping `web_user` fail when `address` exists?  
A: `address.web_user_id` depends on `web_user(web_user_id)`. Drop the foreign key/table in dependency order, or use the right cascade behavior.

</details>

<details>
<summary>[ ] 5. Normalization</summary>

### 定义
Normalization reduces duplication and update anomalies by separating data into related tables.

### 代码模板

Bad design:

```text
student_id | student_name | course1 | course2 | instructor1 | instructor2
```

Better design:

```sql
CREATE TABLE student (
    student_id serial PRIMARY KEY,
    student_name text NOT NULL
);

CREATE TABLE course (
    course_id serial PRIMARY KEY,
    course_name text NOT NULL
);

CREATE TABLE enrollment (
    student_id integer REFERENCES student(student_id),
    course_id integer REFERENCES course(course_id),
    PRIMARY KEY (student_id, course_id)
);
```

### 易错点
- Repeating columns like `phone1`, `phone2`, `phone3` usually violate good design.
- Many-to-many relationships need a bridge table.
- Do not store the same fact in many places.
- Normalization helps consistency, but too many joins can make reads more complex.

### 错题答疑
Q: How do I recognize a many-to-many relationship?  
A: If one student can take many courses and one course can have many students, use a bridge table like `enrollment`.

</details>

<details>
<summary>[ ] 6. Subqueries (*)</summary>

### 定义
A subquery is a query inside another query. It can appear in `WHERE`, `FROM`, or `SELECT`.

### 代码模板

```sql
SELECT title, gross
FROM movie
WHERE gross > (
    SELECT AVG(gross)
    FROM movie
);
```

Subquery in `FROM`:

```sql
SELECT genre_id, avg_gross
FROM (
    SELECT genre_id, AVG(gross) AS avg_gross
    FROM movie
    GROUP BY genre_id
) AS genre_summary
WHERE avg_gross > 100000000;
```

### 易错点
- A scalar subquery must return one row and one column.
- `IN` works with multiple values; `=` expects one value.
- Always alias a subquery used in `FROM`.
- Correlated subqueries run with reference to the outer query and can be slower.

### 错题答疑
Q: Why does `WHERE gross = (SELECT gross FROM movie)` fail?  
A: The subquery returns many rows. Use `IN`, aggregate it to one value, or add a condition.

</details>

<details>
<summary>[ ] 7. Strings and Arrays</summary>

### 定义
String topics include `text`, `varchar(n)`, concatenation, pattern matching, and text functions. Arrays store multiple values in one column, but use them carefully.

### 代码模板

```sql
SELECT first || ' ' || last AS full_name
FROM student
WHERE first ILIKE 'jo%';
```

Array example:

```sql
CREATE TABLE article (
    article_id serial PRIMARY KEY,
    title text,
    tags text[]
);

SELECT title
FROM article
WHERE 'sql' = ANY(tags);
```

### 易错点
- `LIKE` is case-sensitive; `ILIKE` is case-insensitive in PostgreSQL.
- `%` means any length; `_` means one character.
- `NULL || 'text'` becomes `NULL`; use `COALESCE`.
- Arrays are useful for simple tags, but related rows often deserve a separate table.

### 错题答疑
Q: Why does searching with `LIKE 'Jo%'` miss `john`?  
A: `LIKE` is case-sensitive. Use `ILIKE 'jo%'`.

</details>

<details>
<summary>[ ] 8. Dates and Times</summary>

### 定义
PostgreSQL date/time types include `date`, `time`, `timestamp`, and `timestamptz`. The notes use `timestamptz DEFAULT NOW()`.

### 代码模板

```sql
CREATE TABLE student (
    id serial PRIMARY KEY,
    name text,
    registered timestamptz DEFAULT NOW()
);

SELECT name, registered
FROM student
WHERE registered >= NOW() - INTERVAL '7 days';
```

### 易错点
- `date` stores only date; `time` stores only time; `timestamp` stores date and time.
- `timestamptz` handles time zones better for real events.
- Use intervals for date math.
- Do not compare date strings when you can compare actual date/time values.

### 错题答疑
Q: Why is `registered = NOW()` usually wrong?  
A: Exact timestamps almost never match. Use a range like `registered >= CURRENT_DATE`.

</details>

<details>
<summary>[ ] 9. Conditionals</summary>

### 定义
Conditionals let SQL choose values based on rules. The main tool is `CASE`.

### 代码模板

```sql
SELECT name,
       score,
       CASE
           WHEN score >= 90 THEN 'A'
           WHEN score >= 80 THEN 'B'
           WHEN score >= 70 THEN 'C'
           ELSE 'Needs review'
       END AS grade_band
FROM student;
```

### 易错点
- `CASE` returns a value, not rows by itself.
- Conditions are checked top to bottom.
- Use `COALESCE(value, fallback)` to replace `NULL`.
- Use `NULLIF(a, b)` to avoid division by zero.

### 错题答疑
Q: Why should `WHEN score >= 90` come before `WHEN score >= 80`?  
A: If `>= 80` comes first, a 95 matches that branch and never reaches the A condition.

</details>

<details>
<summary>[ ] 10. Window Functions</summary>

### 定义
Window functions calculate across related rows without collapsing the result into one row per group.

### 代码模板

```sql
SELECT title,
       genre_id,
       gross,
       RANK() OVER (
           PARTITION BY genre_id
           ORDER BY gross DESC
       ) AS genre_rank
FROM movie;
```

### 易错点
- `GROUP BY` reduces rows; window functions keep rows.
- `PARTITION BY` is like grouping for the window, but it does not collapse rows.
- `ORDER BY` inside `OVER` controls ranking/order inside the window.
- Window functions are usually computed after `WHERE`, so filter carefully.

### 错题答疑
Q: Why use `RANK()` instead of `GROUP BY` for top movies per genre?  
A: You need individual movie rows, not one summary row per genre.

</details>

<details>
<summary>[ ] 11. Aggregate Functions as Window Functions</summary>

### 定义
Aggregate functions like `AVG`, `SUM`, and `COUNT` can be used with `OVER()` to show group-level summary values beside each row.

### 代码模板

```sql
SELECT title,
       genre_id,
       gross,
       AVG(gross) OVER (PARTITION BY genre_id) AS avg_gross_in_genre,
       gross - AVG(gross) OVER (PARTITION BY genre_id) AS above_genre_avg
FROM movie;
```

### 易错点
- `AVG(gross) OVER (...)` keeps each movie row.
- `AVG(gross)` with `GROUP BY genre_id` returns one row per genre.
- If you use `ORDER BY` inside the window for `SUM`, it may become a running total.

### 错题答疑
Q: Why did my `SUM()` become cumulative?  
A: You probably wrote `SUM(x) OVER (PARTITION BY group ORDER BY date)`. Remove `ORDER BY` if you want the whole partition total.

</details>

<details>
<summary>[ ] 12. EXPLAIN and ANALYZE</summary>

### 定义
`EXPLAIN` shows the query plan PostgreSQL expects to use. `EXPLAIN ANALYZE` actually runs the query and reports real timing and row counts.

### 代码模板

```sql
EXPLAIN ANALYZE
SELECT *
FROM movie
WHERE genre_id = 3
ORDER BY gross DESC;
```

### 易错点
- `EXPLAIN ANALYZE` executes the query, so be careful with `UPDATE`, `DELETE`, and `INSERT`.
- Estimated rows and actual rows being very different can indicate stale statistics or poor planning.
- Sequential scan is not always bad for small tables.
- Index scan is not automatically faster if many rows are returned.

### 错题答疑
Q: Does `EXPLAIN` change data?  
A: Plain `EXPLAIN` does not run the query. `EXPLAIN ANALYZE` runs it, so data-changing queries can change data.

</details>

<details>
<summary>[ ] 13. Adding Indexes</summary>

### 定义
Indexes speed up lookups, joins, sorting, and filtering, but they cost storage and slow down writes.

### 代码模板

```sql
CREATE INDEX idx_movie_genre_id
ON movie (genre_id);

CREATE INDEX idx_movie_title_lower
ON movie (LOWER(title));
```

### 易错点
- Index columns used in `WHERE`, `JOIN ON`, and sometimes `ORDER BY`.
- Do not index every column.
- An index on `LOWER(title)` helps `WHERE LOWER(title) = 'jaws'`, not necessarily `WHERE title = 'Jaws'`.
- Foreign keys often benefit from indexes on the child table column.

### 错题答疑
Q: Why did adding an index not change the plan?  
A: The table may be small, the filter may return too many rows, or the query expression may not match the index.

</details>

<details>
<summary>[ ] 14. Python + Postgres: psycopg2</summary>

### 定义
`psycopg2` is a Python driver for connecting to PostgreSQL, executing SQL, passing parameters, and reading results.

### 代码模板

```python
import psycopg2

conn = psycopg2.connect(
    dbname="postgres",
    user="postgres",
    password="password",
    host="localhost",
)

with conn:
    with conn.cursor() as cur:
        cur.execute(
            "SELECT title FROM movie WHERE year = %s LIMIT %s",
            (1994, 5),
        )
        rows = cur.fetchall()
        for row in rows:
            print(row)

conn.close()
```

### 易错点
- Do not build SQL with f-strings for user values.
- Use `%s` placeholders, even for numbers.
- Remember to commit data-changing statements, or use `with conn:`.
- Close cursors/connections when done.

### 错题答疑
Q: Why is `f"WHERE name = '{name}'"` dangerous?  
A: It allows SQL injection and can break on quotes. Use parameterized queries.

</details>

<details>
<summary>[ ] 15. Python + Postgres: SQLAlchemy</summary>

### 定义
SQLAlchemy provides a higher-level way to connect to databases. For exam review, focus on engine creation and running SQL safely.

### 代码模板

```python
from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg2://postgres:password@localhost/postgres")

with engine.connect() as conn:
    result = conn.execute(
        text("SELECT title FROM movie WHERE year = :year LIMIT :limit"),
        {"year": 1994, "limit": 5},
    )
    for row in result:
        print(row.title)
```

### 易错点
- SQLAlchemy named parameters use `:name`, not `%s`.
- Wrap raw SQL in `text()`.
- Use connection context managers.
- Know the difference between SQLAlchemy engine and raw `psycopg2` connection.

### 错题答疑
Q: Why does `%s` not work in SQLAlchemy `text()`?  
A: SQLAlchemy `text()` uses named parameters like `:year`.

</details>

<details>
<summary>[ ] 16. MongoDB Find (*)</summary>

### 定义
MongoDB stores JSON-like documents. `find(query, projection)` returns documents matching a filter.

### 代码模板

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost")
db = client.database_name

res = db.collection_name.find(
    {"salary": {"$gte": 50000, "$lte": 90000}},
    {"_id": 0, "name": 1, "salary": 1},
).sort("salary", -1).limit(3)

for doc in res:
    print(doc)
```

### 易错点
- PyMongo uses Python dictionaries, so keys and string values need quotes.
- Mongo operators start with `$`, such as `$gt`, `$gte`, `$lt`, `$lte`, `$in`.
- Projection: `1` includes a field, `0` excludes a field. Do not mix include and exclude except `_id`.
- `.find()` returns a cursor; loop over it or convert with `list()`.

### 错题答疑
Q: Why does `{salary: {$gt: 50000}}` fail in Python?  
A: That is JavaScript shell style. In PyMongo, write `{"salary": {"$gt": 50000}}`.

</details>

<details>
<summary>[ ] 17. MongoDB Aggregation</summary>

### 定义
MongoDB aggregation uses a pipeline: documents flow through stages such as `$match`, `$project`, `$group`, `$sort`, and `$limit`.

### 代码模板

```python
pipeline = [
    {"$match": {"salary": {"$gte": 50000}}},
    {"$group": {
        "_id": "$department",
        "avg_salary": {"$avg": "$salary"},
        "count": {"$sum": 1},
    }},
    {"$match": {"avg_salary": {"$gt": 70000}}},
    {"$sort": {"avg_salary": -1}},
    {"$limit": 5},
]

for doc in db.employees.aggregate(pipeline):
    print(doc)
```

### 易错点
- `$match` before `$group` acts like SQL `WHERE`.
- `$match` after `$group` acts like SQL `HAVING`.
- In `$group`, `"_id"` is the grouping key.
- Field references need `$`, such as `"$salary"`.
- Counting rows often uses `{"$sum": 1}`.

### 错题答疑
Q: Why is `_id: "department"` wrong in `$group`?  
A: It groups by the literal string `"department"`. Use `"_id": "$department"` to group by the field value.

</details>

<details>
<summary>[ ] 18. MongoDB with Python / PyMongo</summary>

### 定义
The exam note says MongoDB coding questions will be through PyMongo. Be ready to connect, select database/collection, find documents, and run aggregation pipelines.

### 代码模板

```python
from pymongo import MongoClient

dsn = "mongodb://localhost"
client = MongoClient(dsn)
db = client["database_name"]
collection = db["collection_name"]

collection.insert_one({"name": "Ana", "salary": 72000})

doc = collection.find_one({"name": "Ana"}, {"_id": 0})
print(doc)

collection.update_one(
    {"name": "Ana"},
    {"$set": {"salary": 76000}},
)

collection.delete_one({"name": "Ana"})
```

### 易错点
- `db.collection_name` works, but `db["collection_name"]` is safer for dynamic names.
- Updates need update operators like `$set`; otherwise you may replace the whole document.
- Use `insert_one`, `insert_many`, `update_one`, `delete_one`, `delete_many` correctly.
- Be careful with `{}` in `delete_many({})`: it deletes everything in the collection.

### 错题答疑
Q: What is wrong with `update_one({"name": "Ana"}, {"salary": 76000})`?  
A: It lacks `$set` and can behave like replacement-style update. Use `{"$set": {"salary": 76000}}`.

</details>

<details>
<summary>[ ] 19. Database Design with MongoDB</summary>

### 定义
MongoDB design decides when to embed related data inside one document and when to reference another collection.

### 代码模板

Embedded design:

```python
order = {
    "order_id": 1,
    "customer": {"name": "Ana", "email": "ana@example.com"},
    "items": [
        {"sku": "A1", "qty": 2, "price": 9.99},
        {"sku": "B2", "qty": 1, "price": 14.99},
    ],
}
```

Referenced design:

```python
customer = {"_id": 101, "name": "Ana"}
order = {"order_id": 1, "customer_id": 101, "items": ["A1", "B2"]}
```

### 易错点
- Embed when data is usually read together and bounded in size.
- Reference when data is large, shared, frequently updated independently, or many-to-many.
- MongoDB does not enforce joins like relational foreign keys by default.
- Avoid unbounded arrays that grow forever.

### 错题答疑
Q: Should every relationship be embedded in MongoDB?  
A: No. Embed for read-together, stable, bounded data. Reference for large, shared, or independently changing data.

</details>

## 考前 24 小时 Checklist

- [ ] Redo 10 SQL coding questions from `pgexercises`.
- [ ] Redo one SQL mystery-style problem to practice joins and filtering.
- [ ] Redo MongoDB homework 09 questions, especially PyMongo `find` and aggregation.
- [ ] Make a one-page cheat sheet with syntax only, not long explanations.
- [ ] Memorize execution order: `FROM -> WHERE -> GROUP BY/HAVING -> SELECT -> ORDER BY -> LIMIT`.
- [ ] Memorize Mongo pipeline order: documents -> stage 1 -> stage 2 -> output.
- [ ] Skip graph database.

## 一页 Cheat Sheet 草稿

```sql
-- Basic select
SELECT col, expression AS alias
FROM table
WHERE condition
ORDER BY col DESC
LIMIT n;

-- Aggregation
SELECT group_col, COUNT(*), AVG(x)
FROM table
WHERE row_condition
GROUP BY group_col
HAVING AVG(x) > value;

-- Join
SELECT a.col, b.col
FROM table_a AS a
INNER JOIN table_b AS b ON a.id = b.a_id;

-- Window
SELECT col,
       AVG(x) OVER (PARTITION BY group_col) AS group_avg,
       RANK() OVER (PARTITION BY group_col ORDER BY x DESC) AS rank_in_group
FROM table;

-- Performance
EXPLAIN ANALYZE SELECT * FROM table WHERE col = value;
CREATE INDEX idx_table_col ON table(col);
```

```python
# PyMongo find
client = MongoClient("mongodb://localhost")
db = client["database_name"]
docs = db["collection"].find({"x": {"$gt": 10}}, {"_id": 0, "x": 1}).limit(5)

# PyMongo aggregation
pipeline = [
    {"$match": {"x": {"$gt": 10}}},
    {"$group": {"_id": "$category", "avg_x": {"$avg": "$x"}, "count": {"$sum": 1}}},
    {"$sort": {"avg_x": -1}},
]
list(db["collection"].aggregate(pipeline))
```
