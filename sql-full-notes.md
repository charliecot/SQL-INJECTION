# SQL Injection (SQLi): Complete Beginner-to-Advanced Guide

**Author:** Nzegge Charles Ngong  
**Subject:** Web Application Security / Ethical Hacking  
**Languages:** Python, SQL  
**Databases:** SQLite, PostgreSQL, MySQL  
**HTTP Testing:** CURL, Burp Suite, OWASP ZAP  
**Level:** Beginner to Advanced

---

# PART 1 — UNDERSTANDING SQL INJECTION

## 1. What is SQL injection?

SQL Injection (SQLi) is a web security vulnerability that occurs when an application incorporates untrusted user input into an SQL query without properly separating data from SQL instructions.

An attacker may manipulate the intended SQL query to:

- View information they are not authorized to access.
- Bypass authentication.
- Retrieve information from other database tables.
- Modify database records.
- Delete data.
- In some circumstances, exploit additional database or server capabilities.

The main cause is unsafe construction of SQL statements.

### 1.1 How SQL injection happens

Consider a normal login query:

```sql
SELECT *
FROM users
WHERE username = 'alice'
AND password = 'secret1234';
```

The application expects:

- A username.
- A password.
- A matching database record.

Suppose the Python application constructs the query using an f-string:

```python
username = request.form.get("username")
password = request.form.get("password")

query = f"""
    SELECT *
    FROM users
    WHERE username = '{username}'
    AND password = '{password}'
"""
```

This is dangerous because the user input becomes part of the SQL statement.

The application should instead use parameterized queries.

### 1.2 SQL injection flow

```text
User
  |
  v
Browser / CURL
  |
  v
HTTP Request
  |
  v
Python Application
  |
  v
SQL Query Construction
  |
  v
Database
  |
  v
Query Result
  |
  v
HTTP Response
```

The vulnerability exists when untrusted input changes the intended SQL structure.

## 2. Impact of SQL injection

| Impact | Explanation | Severity |
|---|---|---|
| Data extraction | Unauthorized access to personal or business records | Critical |
| Authentication bypass | Access without valid credentials | Critical |
| Data manipulation | Unauthorized INSERT, UPDATE or DELETE | Critical |
| Privilege escalation | Access to higher-privilege database operations | High |
| Data destruction | Deleting or damaging database records | High |
| Denial of service | Making database operations unavailable | High |
| Server compromise | Possible in certain database configurations | Critical |

SQL injection does not automatically provide operating-system access. Additional database permissions, configuration, and exploitable functionality would be required.

---

# PART 2 — SQL FUNDAMENTALS

Before learning SQL injection, understand the SQL language itself.

## 3. Important SQL commands

### 3.1 SELECT

Retrieves information.

```sql
SELECT *
FROM products;
```

Select particular columns:

```sql
SELECT name, price
FROM products;
```

### 3.2 WHERE

Filters records.

```sql
SELECT *
FROM products
WHERE category = 'Electronics';
```

### 3.3 AND

Both conditions must be true.

```sql
SELECT *
FROM products
WHERE category = 'Electronics'
AND price > 100;
```

### 3.4 OR

At least one condition must be true.

```sql
SELECT *
FROM products
WHERE category = 'Electronics'
OR category = 'Accessories';
```

### 3.5 NOT

Reverses a condition.

```sql
SELECT *
FROM products
WHERE NOT category = 'Electronics';
```

### 3.6 ORDER BY

Sorts results.

```sql
SELECT *
FROM products
ORDER BY price DESC;
```

### 3.7 GROUP BY

Groups records.

```sql
SELECT category, COUNT(*)
FROM products
GROUP BY category;
```

### 3.8 UNION

Combines the results of SELECT statements.

```sql
SELECT name, category
FROM products

UNION

SELECT name, category
FROM archived_products;
```

Requirements:

- Both SELECT statements must return the same number of columns.
- Corresponding columns must have compatible data types.
- UNION removes duplicate rows.
- UNION ALL retains duplicates.

### 3.9 INSERT

Creates a record.

```sql
INSERT INTO products (name, category, price)
VALUES ('Laptop', 'Electronics', 850);
```

### 3.10 UPDATE

Changes an existing record.

```sql
UPDATE products
SET price = 900
WHERE id = 1;
```

### 3.11 DELETE

Deletes records.

```sql
DELETE FROM products
WHERE id = 1;
```

### 3.12 DROP

Removes a database object, such as a table.

```sql
DROP TABLE products;
```

**Warning:** DROP, DELETE and UPDATE can permanently change data. Study them using disposable databases, not production data.

---

# PART 3 — DATABASE DIFFERENCES

## 4. PostgreSQL versus MySQL

PostgreSQL and MySQL both support SQL, but they have different functions, data types, metadata structures, and some syntax rules.

### 4.1 Basic syntax comparison

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Retrieve all | `SELECT * FROM products;` | `SELECT * FROM products;` |
| Limit rows | `LIMIT 5` | `LIMIT 5` |
| Current date | `CURRENT_DATE` | `CURRENT_DATE` |
| Current time | `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` |
| Version | `version()` | `VERSION()` |
| String concatenation | `||` | `CONCAT()` |
| Case-insensitive matching | `ILIKE` | Commonly `LIKE` under case-insensitive collations |
| Comment | `--` | `--` |
| Block comment | `/* ... */` | `/* ... */` |
| MySQL hash comment | Not supported | `#` |

### 4.2 String quotation

PostgreSQL:

```sql
SELECT *
FROM users
WHERE username = 'alice';
```

MySQL:

```sql
SELECT *
FROM users
WHERE username = 'alice';
```

Both use single quotes for string literals.

Double quotes require special attention:

- PostgreSQL uses double quotes for identifiers.
- MySQL normally uses backticks for identifiers.
- MySQL can use double quotes for identifiers when `ANSI_QUOTES` is enabled.

Example PostgreSQL:

```sql
SELECT "username"
FROM "users";
```

Example MySQL:

```sql
SELECT `username`
FROM `users`;
```

### 4.3 Comments

PostgreSQL:

```sql
-- Single-line comment

/*
Multiple-line
comment
*/
```

MySQL:

```sql
-- Single-line comment

# MySQL single-line comment

/*
Multiple-line
comment
*/
```

MySQL requires whitespace or a control character after the second hyphen in its `--` comment syntax.

### 4.4 Database version

PostgreSQL:

```sql
SELECT version();
```

MySQL:

```sql
SELECT VERSION();
```

### 4.5 Type conversion

PostgreSQL:

```sql
SELECT CAST('123' AS INTEGER);
```

MySQL:

```sql
SELECT CAST('123' AS SIGNED);
```

### 4.6 String concatenation

PostgreSQL:

```sql
SELECT 'Hello' || ' ' || 'World';
```

MySQL:

```sql
SELECT CONCAT('Hello', ' ', 'World');
```

---

# PART 4 — UNDERSTANDING SQL INJECTION PAYLOADS

## 5. Basic detection payloads

A payload is input deliberately constructed to test how an application handles SQL syntax.

Your original payload collection is a useful starting point.

| Payload | Category | What it tests |
|---|---|---|
| `'` | Quote test | String delimiter handling |
| `''` | Escaping test | Single-quote escaping |
| `1'` | Context test | Numeric or string context |
| `1 AND 1=1` | Boolean test | True condition |
| `1 AND 1=2` | Boolean test | False condition |
| `1 OR 1=1` | Boolean test | OR condition |
| `--` | Comment test | Line comments |
| `#` | Comment test | MySQL line comments |
| `;` | Stacked query concept | Statement separation |

Important corrections:

- `''` means two single quotes, not two double quotes.
- `1'` is not universally a numeric-field test; it depends on how the application constructs SQL.
- `#` is a MySQL comment marker, not a PostgreSQL or SQLite comment marker.
- A semicolon does not mean that the database driver necessarily permits multiple statements.
- A syntax error alone does not prove SQL injection.

## 6. Boolean-based SQL injection

Boolean-based testing compares application behavior when a SQL condition is true and when it is false.

### True condition

```sql
SELECT *
FROM products
WHERE id = 1 AND 1=1;
```

`1=1` is true.

### False condition

```sql
SELECT *
FROM products
WHERE id = 1 AND 1=2;
```

`1=2` is false.

### Other Boolean expressions

| Expression | Result |
|---|---|
| `2 > 1` | TRUE |
| `2 < 1` | FALSE |
| `5 >= 5` | TRUE |
| `5 <= 4` | FALSE |
| `NOT 1=2` | TRUE |
| `NOT 1=1` | FALSE |
| `NULL IS NULL` | TRUE |
| `NULL IS NOT NULL` | FALSE |

### How to interpret the results

| Test | Expected behavior |
|---|---|
| Baseline | Normal result |
| True condition | Similar to baseline |
| False condition | Different result |
| Safe parameterized endpoint | Input treated as ordinary data |

A consistent difference between true and false conditions may indicate that input is affecting SQL logic.

## 7. Error-based SQL injection

Error-based testing investigates how database errors are generated and handled.

### Important SQL expressions

| Database | Expression | Purpose |
|---|---|---|
| PostgreSQL | `CAST('abc' AS INTEGER)` | Invalid conversion |
| MySQL | `CAST('abc' AS SIGNED)` | Type conversion |
| SQLite | `CAST('abc' AS INTEGER)` | Type conversion |
| SQL Server | `CONVERT(INT, 'abc')` | Type conversion |
| Oracle | `TO_NUMBER('abc')` | Numeric conversion |

### PostgreSQL example

```sql
SELECT CAST('123' AS INTEGER);
```

Result:

```text
123
```

Invalid conversion:

```sql
SELECT CAST('abc' AS INTEGER);
```

PostgreSQL raises an invalid-input-syntax error.

### MySQL example

```sql
SELECT CAST('123' AS SIGNED);
```

Result:

```text
123
```

Invalid conversions can produce warnings or errors depending on the operation and SQL mode.

### SQLite example

```sql
SELECT CAST('abc' AS INTEGER);
```

SQLite may return:

```text
0
```

This is an important difference: the same expression does not necessarily produce the same error on every database.

### Division by zero

PostgreSQL:

```sql
SELECT 10 / 0;
```

This raises a division-by-zero error.

MySQL:

```sql
SELECT 10 / 0;
```

Behavior can depend on SQL mode and execution context; it may return NULL with a warning or raise an error.

SQLite:

```sql
SELECT 10 / 0;
```

Typically returns NULL.

**What you learn:** Database errors and error behavior differ by database engine.

## 8. UNION-based SQL injection

UNION-based testing studies how SQL combines results from different SELECT statements.

Example:

```sql
SELECT name, category
FROM products

UNION

SELECT name, category
FROM archived_products;
```

The statements must return compatible result structures.

### Understanding column counts

A SELECT statement can return one, two, three, or more columns.

Example:

```sql
SELECT id
FROM products;
```

One column.

```sql
SELECT id, name
FROM products;
```

Two columns.

```sql
SELECT id, name, price
FROM products;
```

Three columns.

### Understanding NULL

NULL represents an unknown or missing value.

Example:

```sql
SELECT
    NULL,
    'Training record',
    NULL;
```

This returns three columns.

NULL is useful in SQL experiments because it can be compatible with different column types in many SQL contexts.

### Safe UNION exercise

Create a second table containing harmless training information:

```sql
CREATE TABLE training_notes (
    id INTEGER,
    title VARCHAR(100)
);
```

Insert sample data:

```sql
INSERT INTO training_notes (id, title)
VALUES
    (1, 'SQL fundamentals'),
    (2, 'Database security');
```

Now combine the two result sets:

```sql
SELECT id, name
FROM products

UNION

SELECT id, title
FROM training_notes;
```

This demonstrates UNION without accessing private information.

## 9. Blind SQL injection

Blind SQL injection occurs when the application does not directly display the SQL query result or database error.

You investigate the behavior indirectly.

There are two important categories.

### 9.1 Boolean-based blind SQLi

The application responds differently depending on whether a condition is true or false.

Example SQL:

```sql
SELECT *
FROM products
WHERE category = 'Electronics'
AND 1=1;
```

Compare with:

```sql
SELECT *
FROM products
WHERE category = 'Electronics'
AND 1=2;
```

The application may return different page content, row counts, or status behavior.

### 9.2 Time-based blind SQLi

The application does not expose the result, but the response time changes because of database execution.

PostgreSQL:

```sql
SELECT pg_sleep(2);
```

MySQL:

```sql
SELECT SLEEP(2);
```

These functions delay execution by approximately two seconds.

Use them only in an isolated lab with controlled requests.

Do not conclude that a database is vulnerable from one slow response. Network delays, application load, and caching can also affect response times.

## 10. Out-of-band SQL injection (OAST)

Out-of-band testing investigates whether database execution can trigger an external network interaction.

For example, a vulnerable database configuration might allow an SQL operation to cause a DNS or HTTP interaction.

OAST is useful when:

- The application does not display query results.
- The application does not expose database errors.
- Boolean responses are unavailable.
- Time-based detection is unreliable.

It requires a controlled external interaction endpoint and database-specific functionality.

Use a controlled OAST service or local test infrastructure. Do not use third-party domains or infrastructure without authorization.

---

# PART 5 — PYTHON AND SQL INJECTION

## 11. Python database libraries

Python can communicate with different database engines using database drivers.

| Database | Python library |
|---|---|
| SQLite | `sqlite3` |
| PostgreSQL | `psycopg` |
| PostgreSQL | `psycopg2` |
| MySQL | `mysql-connector-python` |
| MySQL | `PyMySQL` |
| Multiple databases | SQLAlchemy |

### Installation

PostgreSQL:

```powershell
python -m pip install "psycopg[binary]"
```

MySQL:

```powershell
python -m pip install mysql-connector-python
```

SQLAlchemy:

```powershell
python -m pip install sqlalchemy
```

SQLite is included with standard Python installations.

## 12. Python and SQLite

Create a simple database:

```python
import sqlite3

connection = sqlite3.connect("training.db")

cursor = connection.cursor()

cursor.execute("""
    CREATE TABLE IF NOT EXISTS products (
        id INTEGER PRIMARY KEY,
        name TEXT,
        category TEXT,
        price REAL
    )
""")

cursor.execute("""
    INSERT INTO products (name, category, price)
    VALUES (?, ?, ?)
""", ("Laptop", "Electronics", 850))

connection.commit()

cursor.close()
connection.close()
```

The `?` characters are parameter placeholders.

The actual values are passed separately.

## 13. PostgreSQL with Python

Install:

```powershell
python -m pip install "psycopg[binary]"
```

Connect:

```python
import psycopg

connection = psycopg.connect(
    host="localhost",
    port=5432,
    dbname="training_db",
    user="postgres",
    password="YOUR_LOCAL_PASSWORD"
)

cursor = connection.cursor()

cursor.execute("""
    SELECT id, name, category, price
    FROM products
    WHERE category = %s
""", ("Electronics",))

products = cursor.fetchall()

for product in products:
    print(product)

cursor.close()
connection.close()
```

**Important:** Psycopg uses `%s` as a value placeholder. Do not put quotes around the placeholder.

Correct:

```python
cursor.execute(
    "SELECT * FROM products WHERE category = %s",
    ("Electronics",)
)
```

Incorrect:

```python
cursor.execute(
    "SELECT * FROM products WHERE category = '%s'",
    ("Electronics",)
)
```

## 14. MySQL with Python

Install:

```powershell
python -m pip install mysql-connector-python
```

Connect:

```python
import mysql.connector

connection = mysql.connector.connect(
    host="localhost",
    port=3306,
    database="training_db",
    user="root",
    password="YOUR_LOCAL_PASSWORD"
)

cursor = connection.cursor()

cursor.execute("""
    SELECT id, name, category, price
    FROM products
    WHERE category = %s
""", ("Electronics",))

products = cursor.fetchall()

for product in products:
    print(product)

cursor.close()
connection.close()
```

MySQL Connector/Python also uses `%s` for positional value parameters.

## 15. Unsafe versus safe Python SQL

### Unsafe string interpolation

```python
category = "Electronics"

query = f"""
    SELECT *
    FROM products
    WHERE category = '{category}'
"""

cursor.execute(query)
```

Problem: user input becomes part of the SQL statement.

### Unsafe concatenation

```python
query = (
    "SELECT * FROM products "
    "WHERE category = '" + category + "'"
)

cursor.execute(query)
```

This has the same fundamental problem.

### Safe parameterization: SQLite

```python
cursor.execute(
    "SELECT * FROM products WHERE category = ?",
    (category,)
)
```

### Safe parameterization: PostgreSQL

```python
cursor.execute(
    "SELECT * FROM products WHERE category = %s",
    (category,)
)
```

### Safe parameterization: MySQL

```python
cursor.execute(
    "SELECT * FROM products WHERE category = %s",
    (category,)
)
```

### Placeholder comparison

| Driver | Placeholder |
|---|---|
| SQLite | `?` |
| Psycopg | `%s` |
| MySQL Connector/Python | `%s` |
| SQLAlchemy named parameter | `:category` |

## 16. Dynamic SQL identifiers

A parameter placeholder is for a value, not an SQL identifier.

For example, this is not a valid way to parameterize a column name:

```python
cursor.execute(
    "SELECT * FROM products ORDER BY %s",
    ("price",)
)
```

The database treats the parameter as a value, not as a column identifier.

Use an allowlist:

```python
allowed_columns = {
    "name",
    "price",
    "category"
}

sort_column = request.args.get("sort", "name")

if sort_column not in allowed_columns:
    return {"error": "Invalid sort column"}, 400

query = f"""
    SELECT id, name, category, price
    FROM products
    ORDER BY {sort_column}
"""

cursor.execute(query)
```

The interpolation is acceptable here because `sort_column` is selected from a fixed allowlist rather than directly trusted from the request.

---

# PART 6 — CURL AND HTTP TESTING

## 17. What is CURL?

CURL is a command-line tool for sending HTTP requests.

It helps security testers interact with web applications without requiring a browser.

The request flow:

```text
CURL
  |
  v
HTTP GET / POST
  |
  v
Python Flask / Django
  |
  v
SQL Query
  |
  v
PostgreSQL / MySQL / SQLite
  |
  v
HTTP Response
```

For Windows PowerShell, use `curl.exe` rather than `curl` to avoid the PowerShell alias.

## 18. Important CURL options

| Option | Purpose |
|---|---|
| `-G` | Use GET and encode supplied data into the URL |
| `-X` | Specify HTTP method |
| `-d` | Send request data |
| `--data-urlencode` | URL-encode data |
| `-H` | Add HTTP headers |
| `-i` | Include response headers |
| `-I` | Request headers only |
| `-v` | Verbose connection and request information |
| `-L` | Follow redirects |
| `-b` | Send cookies |
| `-c` | Save cookies |
| `-s` | Silent mode |
| `-S` | Show errors in silent mode |
| `--max-time` | Set maximum request time |

## 19. GET request using CURL

Normal request:

```powershell
curl.exe -G "http://127.0.0.1:8000/products" `
    --data-urlencode "category=Electronics"
```

This sends:

```http
GET /products?category=Electronics HTTP/1.1
```

## 20. POST form request

```powershell
curl.exe -X POST "http://127.0.0.1:8000/products" `
    -H "Content-Type: application/x-www-form-urlencoded" `
    --data-urlencode "category=Electronics"
```

The application must have a matching POST endpoint.

## 21. POST JSON request

```powershell
curl.exe -X POST "http://127.0.0.1:8000/api/products" `
    -H "Content-Type: application/json" `
    -d '{"category":"Electronics"}'
```

Python Flask reads it with:

```python
data = request.get_json()

category = data.get("category")
```

JSON input is not automatically safe. It still needs parameterized SQL queries.

## 22. Sending cookies

```powershell
curl.exe -b "session=YOUR_TEST_SESSION" `
    "http://127.0.0.1:8000/products"
```

## 23. Sending custom headers

```powershell
curl.exe "http://127.0.0.1:8000/products" `
    -H "X-Category: Electronics"
```

An application may read this header:

```python
category = request.headers.get("X-Category")
```

If the value is subsequently incorporated into SQL unsafely, it can become an injection point.

---

# PART 7 — PRACTICAL LOCAL SQL INJECTION LAB

## 24. Create the project

Create this directory:

```text
SQL-INJECTION-LAB/
│
├── app.py
├── database.py
├── requirements.txt
│
├── templates/
│
└── database/
```

Create a virtual environment:

```powershell
mkdir SQL-INJECTION-LAB
cd SQL-INJECTION-LAB

python -m venv venv

venv\Scripts\activate
```

Install Flask:

```powershell
python -m pip install flask
```

Create `requirements.txt`:

```text
Flask
```

## 25. Understand the lab endpoints

The application should expose two endpoints:

```text
/vulnerable/products
/safe/products
```

The first deliberately constructs SQL unsafely for controlled testing.

The second uses parameterized SQL.

Keep the application bound to `127.0.0.1`, and use synthetic training records only.

### Vulnerable query concept

```python
query = f"""
    SELECT id, name, category, price
    FROM products
    WHERE category = '{category}'
"""
```

### Safe query concept

```python
query = """
    SELECT id, name, category, price
    FROM products
    WHERE category = ?
"""

cursor.execute(query, (category,))
```

## 26. Test the endpoints

Normal request:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=Electronics"
```

Compare the response with the safe endpoint:

```powershell
curl.exe -G "http://127.0.0.1:8000/safe/products" `
    --data-urlencode "category=Electronics"
```

For the Boolean comparison, use the controlled local lab's documented true-condition and false-condition test inputs.

Compare:

- HTTP status code.
- Response body.
- Number of returned rows.
- Server-side SQL log.
- Database errors.

Do not assume that every unusual response proves SQL injection.

---

# PART 8 — SQL INJECTION TOOLS

## 27. Burp Suite Community Edition

<https://portswigger.net/burp/communitydownload>

<details>
<summary>What is Burp Suite?</summary>

Burp Suite is a web application security testing platform.

Important features:

- Proxy: Intercepts HTTP requests and responses.
- HTTP history: Shows previous requests.
- Repeater: Allows manual request modification and resending.
- Decoder: Encodes and decodes data.
- Intruder: Automates controlled request variations, with limits in Community Edition.

</details>

Recommended workflow:

1. Open your local lab.
2. Configure the browser to use Burp Proxy.
3. Visit the application.
4. Capture a request.
5. Send it to Repeater.
6. Modify one input at a time.
7. Compare responses.

## 28. OWASP ZAP

<https://www.zaproxy.org/>

OWASP ZAP is an open-source web application security testing tool.

Features:

- Automated scanning.
- Passive scanning.
- Active scanning.
- Request interception.
- HTTP history.
- API testing.

Only run active scans against authorized targets.

## 29. SQLmap

<https://sqlmap.org/>

SQLmap is an automated SQL injection detection and testing tool.

It supports different injection categories:

- Boolean-based blind.
- Error-based.
- UNION-based.
- Time-based blind.
- Stacked queries.
- Out-of-band techniques where supported.

Example: perform a limited detection test against your own local lab:

```powershell
sqlmap -u "http://127.0.0.1:8000/vulnerable/products?category=Electronics" `
    -p category `
    --batch `
    --risk=1 `
    --level=1
```

What the options mean:

| Option | Meaning |
|---|---|
| `-u` | Target URL |
| `-p` | Specify the parameter to test |
| `--batch` | Use default answers to prompts |
| `--risk=1` | Use the lowest risk level |
| `--level=1` | Use the lowest testing level |

This is a starting detection exercise, not proof that the endpoint is vulnerable. SQLmap's findings must be verified.

Do not use SQLmap against systems you do not own or have permission to assess.

## 30. Database management tools

| Tool | Database |
|---|---|
| pgAdmin | PostgreSQL |
| DBeaver | Multiple databases |
| MySQL Workbench | MySQL |
| DB Browser for SQLite | SQLite |
| TablePlus | Multiple databases |

These tools help you inspect database tables, SQL queries, schemas, and test records.

## 31. Browser developer tools

Press:

```text
F12
```

Useful browser tabs:

| Tab | Purpose |
|---|---|
| Network | Inspect HTTP requests |
| Console | Inspect JavaScript errors |
| Application | Inspect cookies and storage |
| Sources | Inspect frontend source files |

The Network tab is particularly useful for identifying GET parameters, POST bodies, JSON, cookies, and headers.

---

# PART 9 — ADVANCED SQL INJECTION CONCEPTS

## 32. Second-order SQL injection

Second-order SQL injection occurs when an application initially stores input as data, but later uses that stored value unsafely in another SQL statement.

Example workflow:

```text
User submits input
       |
       v
Application stores it
       |
       v
Later, another feature retrieves it
       |
       v
Unsafe SQL construction
       |
       v
Potential SQL injection
```

The input may appear harmless when first stored.

The vulnerability occurs during a later SQL operation.

Prevention: use parameterized queries both when storing data and when using retrieved data in future queries.

## 33. Stored procedure considerations

Stored procedures are database-side functions that execute SQL operations.

PostgreSQL:

```sql
CREATE FUNCTION get_product(product_id INTEGER)
RETURNS TABLE (
    id INTEGER,
    name TEXT
)
LANGUAGE SQL
AS $$
    SELECT id, name
    FROM products
    WHERE id = product_id;
$$;
```

MySQL:

```sql
DELIMITER //

CREATE PROCEDURE get_product(IN product_id INT)
BEGIN
    SELECT id, name
    FROM products
    WHERE id = product_id;
END //

DELIMITER ;
```

Stored procedures are not automatically immune to SQL injection.

Unsafe dynamic SQL inside a stored procedure can still introduce vulnerabilities.

## 34. Database metadata

Databases expose information about their schemas.

PostgreSQL:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public';
```

MySQL:

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = DATABASE();
```

SQLite:

```sql
SELECT name
FROM sqlite_master
WHERE type = 'table';
```

These are useful for understanding database structure in your own training database.

## 35. Time-based detection comparison

| Database | Delay function |
|---|---|
| PostgreSQL | `pg_sleep(2)` |
| MySQL | `SLEEP(2)` |
| SQL Server | `WAITFOR DELAY` |
| Oracle | `DBMS_PIPE.RECEIVE_MESSAGE()` |

PostgreSQL:

```sql
SELECT pg_sleep(2);
```

MySQL:

```sql
SELECT SLEEP(2);
```

These are database-specific functions. Use only in a local test environment and compare repeated baseline requests to avoid confusing ordinary network latency with SQL execution delays.

## 36. SQL injection through sorting and filtering

Consider:

```sql
SELECT *
FROM products
ORDER BY price DESC;
```

A common development mistake is directly inserting a request's sorting field:

```python
sort = request.args.get("sort")

query = f"SELECT * FROM products ORDER BY {sort}"
```

A value parameter cannot safely replace a column identifier.

Use an allowlist:

```python
allowed_sort = {
    "name": "name",
    "price": "price",
    "category": "category"
}

sort = request.args.get("sort", "name")

if sort not in allowed_sort:
    return {"error": "Invalid sorting field"}, 400

query = f"""
    SELECT *
    FROM products
    ORDER BY {allowed_sort[sort]}
"""

cursor.execute(query)
```

This applies to:

- Sorting.
- Dynamic table names.
- Dynamic column names.
- Dynamic SQL directions.
- Filtering expressions.

---

# PART 10 — SECURING PYTHON APPLICATIONS

## 37. Parameterized queries

The most important defense for SQL values is parameterization.

SQLite:

```python
cursor.execute(
    "SELECT * FROM products WHERE id = ?",
    (product_id,)
)
```

PostgreSQL:

```python
cursor.execute(
    "SELECT * FROM products WHERE id = %s",
    (product_id,)
)
```

MySQL:

```python
cursor.execute(
    "SELECT * FROM products WHERE id = %s",
    (product_id,)
)
```

## 38. Validate user input

Example:

```python
product_id = request.args.get("id")

try:
    product_id = int(product_id)
except (TypeError, ValueError):
    return {"error": "Invalid product ID"}, 400
```

Input validation is useful, but it does not replace parameterized SQL.

## 39. Least-privilege database accounts

Do not connect your public web application using a database administrator account.

Create a database user with only the permissions it needs.

For example, a product-listing application may need SELECT permission on its product table but not permission to drop databases or create administrators.

## 40. Secure error handling

Avoid returning database error details to users.

Unsafe:

```python
except Exception as error:
    return {"error": str(error)}, 500
```

Better:

```python
except Exception:
    app.logger.exception("Database operation failed")

    return {
        "error": "Unable to process your request"
    }, 500
```

Detailed errors should be recorded securely in server logs.

## 41. Authentication best practices

Do not build authentication using raw SQL string interpolation.

Also:

- Hash passwords using a suitable password-hashing algorithm.
- Never store plaintext passwords.
- Use established authentication libraries.
- Use parameterized SQL for username lookup.
- Apply rate limiting to login endpoints.
- Use proper authorization checks.

---

# PART 11 — PRACTICAL STUDY PLAN

## 42. Beginner level

Learn:

1. What SQL is.
2. SELECT, INSERT, UPDATE and DELETE.
3. WHERE, AND, OR and NOT.
4. Single quotes and double quotes.
5. SQL comments.
6. Python database connections.
7. CURL GET and POST.
8. Parameterized queries.

**Project:** Build a product-search application using Flask and SQLite.

## 43. Intermediate level

Learn:

1. Boolean-based SQL injection.
2. Error-based SQL injection.
3. UNION and UNION ALL.
4. Blind SQL injection.
5. SQL metadata.
6. GET, POST, JSON and cookie input.
7. Burp Suite Repeater.
8. OWASP ZAP.

**Project:** Build a controlled SQL injection detection lab with vulnerable and safe endpoints.

## 44. Advanced level

Learn:

1. Time-based blind SQL injection.
2. Second-order SQL injection.
3. Stored procedure security.
4. Dynamic SQL.
5. Database-specific SQL behavior.
6. Out-of-band testing concepts.
7. SQLmap.
8. Automated testing.
9. Secure code reviews.
10. SQL injection prevention and verification.

**Project:** Build a Python SQL injection testing and prevention demonstration tool for your own lab.

---

# PART 12 — FINAL REFERENCE

## 45. Database and Python quick reference

| Feature | SQLite | PostgreSQL | MySQL |
|---|---|---|---|
| Python library | `sqlite3` | `psycopg` | `mysql.connector` |
| Value placeholder | `?` | `%s` | `%s` |
| Version function | `sqlite_version()` | `version()` | `VERSION()` |
| Line comment | `--` | `--` | `--` and `#` |
| Block comment | `/* */` | `/* */` | `/* */` |
| Delay function | No direct equivalent in standard SQLite SQL | `pg_sleep()` | `SLEEP()` |
| String concatenation | `||` | `||` | `CONCAT()` |

## 46. Useful resources

| Resource | Purpose | Website |
|---|---|---|
| PortSwigger Web Security Academy | SQL injection lessons and labs | https://portswigger.net/web-security/sql-injection |
| OWASP SQL Injection Prevention | Prevention guidance | https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html |
| OWASP Juice Shop | Vulnerable web application lab | https://owasp.org/www-project-juice-shop/ |
| OWASP WebGoat | Security training application | https://owasp.org/www-project-webgoat/ |
| DVWA | Deliberately vulnerable web application | https://github.com/digininja/DVWA |
| SQLmap | Automated SQL injection testing | https://sqlmap.org/ |
| Burp Suite | Web security testing | https://portswigger.net/burp |
| OWASP ZAP | Web application security testing | https://www.zaproxy.org/ |
| PostgreSQL Documentation | PostgreSQL SQL reference | https://www.postgresql.org/docs/ |
| MySQL Documentation | MySQL SQL reference | https://dev.mysql.com/doc/ |
| Python Documentation | Python database programming | https://docs.python.org/3/library/sqlite3.html |

## 47. Final advice

Do not approach SQL injection as a collection of strings to memorize.

Understand these four things:

1. **SQL:** What query is the application trying to execute?
2. **Python:** How is the query constructed and executed?
3. **HTTP:** How does user input reach the Python application?
4. **Database:** How does PostgreSQL, MySQL, or SQLite interpret that query?

When you understand these four layers, you can identify SQL injection more accurately, explain why it happens, reproduce it in an authorized lab, and implement the correct prevention.

**The goal is not simply to learn how to exploit SQL injection. It is to understand how to discover, demonstrate, diagnose, and prevent it.**
