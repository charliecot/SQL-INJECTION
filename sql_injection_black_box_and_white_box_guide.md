# SQL Injection Testing: Black-Box and White-Box Perspectives

## Purpose of this guide

This document brings together two complementary ways to assess SQL
injection risk:

-   **Black-box perspective:** test the application externally by
    sending requests and observing responses.
-   **White-box perspective:** inspect source code and trace
    user-controlled input to database queries.

Use both approaches only on systems you own or are explicitly authorized
to assess. Prefer local or staging environments with synthetic data. The
examples use a deliberately vulnerable local training application and
are not instructions to test arbitrary public systems.

## How the two perspectives fit together

  -----------------------------------------------------------------------
  Question                Black-box               White-box
  ----------------------- ----------------------- -----------------------
  What can I see?         Requests and responses  Source code, query
                                                  construction, logs

  What do I do?           Send controlled inputs  Trace input paths and
                          and compare behaviour   inspect database calls

  What can I establish?   Observable behaviour    Whether and where
                          that may indicate SQLi  unsafe query
                                                  construction occurs

  Main limitation         Internal cause may be   Uninspected paths can
                          unclear                 be missed

  Best evidence           Repeatable response     Code data flow plus
                          differences             controlled runtime
                                                  evidence
  -----------------------------------------------------------------------

A useful overall process is:

``` text
Map the application
       |
       v
Identify input vectors
       |
       v
Black-box tests and baseline comparisons
       |
       v
Inspect source code and database access
       |
       v
Trace input to query execution
       |
       v
Confirm safely in a local/staging environment
       |
       v
Fix, retest, and document
```

------------------------------------------------------------------------

# Part 1 --- Black-Box Perspective: SQL Injection Testing

## 1. What is black-box testing?

Black-box testing means testing an application from the outside, without
relying on access to its source code or internal database
implementation. You send requests, observe responses, and compare
behaviour.

For SQL injection testing, the main question is:

> Can input supplied through the application cause the database to
> interpret part of that input as SQL syntax instead of ordinary data?

Use only applications you own or are explicitly authorized to test. The
practical examples below are intended for a local training lab using
synthetic data.

## 2. Map the application

Before testing, identify the application's externally visible features
and entry points.

For an e-commerce application, examples include:

-   Product listing
-   Product search
-   Product details
-   Category and price filters
-   Login and registration
-   User profile
-   Shopping cart
-   Checkout
-   Order history
-   Admin search

Identify how data enters each feature:

  ----------------------------------------------------------------------------
  Input location          Example                      What to record
  ----------------------- ---------------------------- -----------------------
  Query parameter         `/products?category=books`   Parameter name and
                                                       endpoint

  URL path                `/products/25/`              Path parameter

  Form field              Search form                  Field name and HTTP
                                                       method

  JSON body               `{"category":"books"}`       JSON key and content
                                                       type

  Cookie                  Session or preference value  Cookie name and whether
                                                       app uses it

  HTTP header             Custom application header    Header name and whether
                                                       app uses it
  ----------------------------------------------------------------------------

Create an endpoint inventory. Include the HTTP method, input names,
authentication requirement, and normal expected response.

## 3. Establish a normal baseline

A baseline is the normal response before sending test input. It gives
you something to compare against.

For the local Flask training lab:

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category=Electronics"
```

Record:

-   HTTP status code
-   Response body
-   Number of returned records
-   Response time
-   Any visible error message

Do not immediately interpret an unusual response as SQL injection. First
understand the normal application behaviour.

## 4. Fuzz with SQL-related characters

Fuzzing here means sending carefully selected test characters and
observing whether the application's behaviour changes.

Examples of simple test values:

``` text
'
"
''
')
")
```

For the local lab:

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category='"
```

Observe:

-   Does the response status change?
-   Does the page show a database-related error?
-   Does the response body change unexpectedly?
-   Does the application return an empty result?
-   Does the server log show an exception?

A database error can be a clue, but it is not proof by itself. An HTTP
500 can also result from an unrelated application bug.

## 5. Compare Boolean conditions

Boolean-based testing compares an expression expected to be true with
one expected to be false. If user input is unsafely inserted into SQL,
the two requests may cause visibly different results.

For the deliberately vulnerable local lab endpoint, compare:

**Expected true condition:**

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category=' OR '1'='1' -- "
```

**Expected false condition:**

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category=' AND '1'='2' -- "
```

Compare both with the normal baseline:

  Test              Expected logical result   What to compare
  ----------------- ------------------------- ----------------------------------------
  Normal category   Ordinary lookup           Normal matching products
  True condition    True                      Whether result set changes
  False condition   False                     Whether result set changes differently

The exact syntax depends on the database engine and the original query.
Do not assume a payload for one SQL dialect will work in another.

## 6. Understand time-delay testing

Some SQL injection flaws do not produce a visible difference in the
response body. A time-based test looks for a repeatable delay caused by
a database expression.

Common database-specific delay functions include:

  Database     Example delay expression
  ------------ -------------------------------
  MySQL        `SLEEP(5)`
  PostgreSQL   `pg_sleep(5)`
  SQL Server   `WAITFOR DELAY`
  Oracle       `DBMS_PIPE.RECEIVE_MESSAGE()`

For example, PostgreSQL supports this SQL expression:

``` sql
SELECT pg_sleep(5);
```

That statement pauses for approximately five seconds when executed by
PostgreSQL.

Important precautions:

-   Use only a disposable database and an authorized lab.
-   Keep delays short.
-   Compare multiple normal requests with test requests.
-   Account for network latency, server load, and application processing
    time.
-   Do not treat one slow response as proof.

## 7. Understand OAST testing

OAST means Out-of-band Application Security Testing. It is useful when
an application may cause an external DNS or HTTP interaction that is not
visible in the normal HTTP response.

At a high level:

1.  A tester uses a controlled interaction-monitoring service.
2.  A test is performed in an authorized environment.
3.  The tester checks whether the application or database caused an
    external DNS/HTTP interaction.
4.  The interaction is correlated with the test request and logs.

Tools commonly used for authorized OAST work include Burp Collaborator
and Interactsh.

An interaction is evidence that an external lookup or request occurred.
It does not automatically prove that sensitive data was extracted. Keep
tests non-destructive and do not use OAST to collect private data.

## 8. Black-box testing record

Keep a record for every test:

  Field             Example
  ----------------- ---------------------------------------
  Test ID           BB-SQLI-001
  Endpoint          `/vulnerable/products`
  Method            GET
  Input parameter   `category`
  Baseline input    `Electronics`
  Test input        Quote or controlled Boolean test
  Baseline result   Status and response summary
  Test result       Status and response summary
  Timing            Baseline versus test duration
  Evidence          Request, response, and relevant logs
  Conclusion        Confirmed, suspected, or not observed

Use cautious conclusions. Distinguish a confirmed vulnerability from a
suspicious behaviour that needs further investigation.

## 9. Black-box workflow summary

1.  Map the application and identify input points.
2.  Establish normal baseline responses.
3.  Send simple SQL-related characters.
4.  Compare controlled true and false conditions.
5.  Consider time-based behaviour in a safe lab when appropriate.
6.  Consider OAST only in an authorized, controlled environment.
7.  Correlate observations and logs.
8.  Document evidence and avoid destructive tests.

------------------------------------------------------------------------

# Part 2 --- White-Box Perspective: SQL Injection Testing and Code Review

## 1. What is white-box testing?

White-box testing examines an application from the inside. You can
inspect source code, configuration, database queries, and logs. The goal
is to trace user-controlled input from its entry point to the database
and determine whether it is handled as data or can alter SQL syntax.

Test only applications and databases you own or are explicitly
authorized to assess. Prefer a local or staging environment with
synthetic data.

## 2. Enable web server and application logging

Logging helps identify which endpoint received a request, which method
was used, what response was returned, and whether an application error
occurred.

### Apache on Ubuntu

Common log locations:

``` text
/var/log/apache2/access.log
/var/log/apache2/error.log
```

Watch the logs during a test:

``` bash
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/error.log
```

### Nginx

Common log locations:

``` text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

``` bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

Actual paths depend on the server configuration.

### Python application logging

Web server logs do not necessarily show which application function
handled a request. Add application-level logging:

``` python
import logging

logger = logging.getLogger(__name__)

def search_products(request):
    category = request.GET.get("category", "")
    logger.info(
        "Product search: method=%s path=%s",
        request.method,
        request.path,
    )
    # Continue with the normal search.
```

Log useful metadata, but do not log passwords, access tokens, session
cookies, payment details, or unnecessary personal data. Treat raw user
input carefully because it may contain newlines or control characters.
Prefer request IDs and bounded, sanitized values.

Useful fields to correlate:

  ---------------------------------------------------------------------
  Field                              Purpose
  ---------------------------------- ----------------------------------
  Timestamp                          Match events across logs

  Request/correlation ID             Connect web, application, and
                                     database events

  Method and route                   Identify the endpoint

  Input field name                   Identify the input vector

  Response status                    Record the external result

  Exception type                     Locate an internal failure

  Query operation/location           Identify the database code path
  ---------------------------------------------------------------------

A HTTP 500 response alone does not prove SQL injection.

## 3. Enable database logging

Database logs can show which SQL statements reached the database. Enable
detailed logging only in a controlled environment and only as long as
necessary: statements can contain sensitive values and verbose logs can
become large.

### PostgreSQL

Review `postgresql.conf`. Relevant settings include:

``` conf
logging_collector = on
log_statement = 'all'
log_min_error_statement = error
log_line_prefix = '%m [%p] user=%u db=%d app=%a '
```

Inspect current settings:

``` sql
SHOW log_statement;
SHOW log_min_error_statement;
SHOW log_directory;
SHOW logging_collector;
```

`log_statement = 'all'` can generate substantial logs and expose
sensitive query values. Check PostgreSQL documentation for whether a
setting needs a reload or restart.

### MySQL

Inspect logging settings:

``` sql
SHOW VARIABLES LIKE 'general_log';
SHOW VARIABLES LIKE 'general_log_file';
SHOW VARIABLES LIKE 'log_error';
```

An administrator can enable the general query log temporarily in a
controlled test:

``` sql
SET GLOBAL general_log = 'ON';
```

Disable it afterward:

``` sql
SET GLOBAL general_log = 'OFF';
```

### SQLite

SQLite is embedded and normally has no separate database server log.
Python's SQLite connection can trace statements in a local test:

``` python
import sqlite3

connection = sqlite3.connect("database.db")
connection.set_trace_callback(print)

# Execute controlled test queries through this connection.
```

Use this only in development; traced statements may contain sensitive
values.

## 4. Map the application

First list the functionality users can access. For an e-commerce
application, examples include:

-   Product listing and detail pages
-   Product search
-   Category and price filters
-   Login and registration
-   User profiles
-   Shopping cart and checkout
-   Order history
-   Admin search and reports

### Identify input vectors

An input vector is a place where data enters the application and may
later influence a database query.

Inspect:

-   URL query parameters, e.g. `/products?category=books`
-   URL path parameters, e.g. `/products/25/`
-   HTML form fields
-   JSON request bodies
-   Cookies
-   HTTP headers, if the application uses them
-   Search boxes and filters
-   Sort and pagination parameters
-   Admin-only forms and internal APIs
-   Imported data or file metadata if later used in queries

Build an inventory:

  Feature           Input          Entry point       Possible database operation
  ----------------- -------------- ----------------- -----------------------------
  Product search    `q`            Query parameter   Search products
  Category filter   `category`     Query parameter   Filter products
  Product detail    `product_id`   URL path          Retrieve one product
  Login             `username`     Form/JSON body    Find user
  Order history     `status`       Query parameter   Filter orders
  Product sorting   `sort`         Query parameter   Order results

Confirm each item against the actual application. Do not assume every
visible input reaches the database.

## 5. Search the code for database access

Search the project for places that execute queries. No single regular
expression finds every database operation, so use several searches and
inspect their results.

### ripgrep (`rg`)

``` bash
rg -n -i "execute\\(|executemany\\(|cursor\\(|raw\\(|extra\\(|\\.query|text\\(" .
rg -n -i "sqlite3|psycopg|psycopg2|mysql|pymysql|sqlalchemy|django\\.db" .
rg -n -i "SELECT|INSERT|UPDATE|DELETE|FROM|WHERE|ORDER BY|JOIN" .
```

Exclude virtual environments and generated folders where possible:

``` bash
rg -n "execute\\(" . -g "*.py" -g "!venv/**" -g "!APIenv/**" -g "!**/__pycache__/**"
```

### PowerShell alternative

``` powershell
Get-ChildItem -Recurse -File -Include *.py |
    Select-String -Pattern 'execute\(|executemany\(|cursor\(|raw\(|extra\('
```

``` powershell
Get-ChildItem -Recurse -File -Include *.py |
    Select-String -Pattern 'SELECT|INSERT|UPDATE|DELETE|FROM|WHERE|ORDER BY'
```

### Interpret search results

For every match, ask:

1.  Is this code executed, or is it only a comment/example?
2.  Which view, route, or function calls it?
3.  Where does each query value originate?
4.  Are values passed as bound parameters?
5.  Is SQL assembled with concatenation, f-strings, `%` formatting, or
    `.format()`?
6.  Are dynamic table/column names or sort directions restricted to an
    allowlist?
7.  What validation or transformation occurs before the query?

A search match is a lead, not proof. Follow callers and inspect
surrounding code.

## 6. Code review: follow the complete input path

Trace each input from the HTTP request to the database:

``` text
HTTP request
    |
    v
URL parameter / form / JSON body
    |
    v
View or route handler
    |
    v
Validation and serializer
    |
    v
Service or repository function
    |
    v
SQL query construction
    |
    v
Database driver / ORM
    |
    v
Database
```

At each stage, determine whether the value is validated, transformed,
rejected, or passed through.

### Unsafe Python example

``` python
def get_products(connection, category):
    sql = f"SELECT id, name FROM products WHERE category = '{category}'"
    return connection.execute(sql).fetchall()
```

The f-string combines SQL syntax and user input into one SQL string. If
the input contains SQL syntax, it may change the query's meaning.

### Safer SQLite example

``` python
def get_products(connection, category):
    sql = """
        SELECT id, name
        FROM products
        WHERE category = ?
    """
    return connection.execute(sql, (category,)).fetchall()
```

The `?` is a placeholder. `(category,)` is a one-element tuple
containing the value. The driver sends the value separately from the SQL
statement.

### PostgreSQL with Psycopg

``` python
cursor.execute(
    """
    SELECT id, name
    FROM products
    WHERE category = %s
    """,
    (category,),
)
```

### MySQL Connector/Python

``` python
cursor.execute(
    """
    SELECT id, name
    FROM products
    WHERE category = %s
    """,
    (category,),
)
```

Placeholder syntax depends on the driver. SQLite commonly uses `?`;
Psycopg and MySQL Connector/Python commonly use `%s` for positional
parameters. Do not mix placeholder styles.

### Django ORM and raw SQL

Ordinary ORM filtering:

``` python
Product.objects.filter(category=category)
```

Parameterized raw SQL:

``` python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute(
        "SELECT id, name FROM shop_product WHERE category = %s",
        [category],
    )
    rows = cursor.fetchall()
```

Unsafe raw SQL construction:

``` python
sql = f"SELECT id, name FROM shop_product WHERE category = '{category}'"
with connection.cursor() as cursor:
    cursor.execute(sql)
```

The ORM is generally safe when used normally, but review raw SQL, custom
expressions, and dynamically assembled query fragments.

## 7. Review dynamic SQL identifiers

Bound parameters are for values. They generally cannot replace SQL
identifiers such as table names, column names, or keywords.

Do not expect this to select a dynamic column:

``` python
cursor.execute(
    "SELECT id, name FROM products ORDER BY %s",
    (sort_column,),
)
```

Map user choices to fixed, trusted identifiers:

``` python
ALLOWED_SORT_COLUMNS = {
    "name": "name",
    "price": "price",
    "created": "created_at",
}

sort_choice = request.args.get("sort", "name")
sort_column = ALLOWED_SORT_COLUMNS.get(sort_choice, "name")

sql = f"SELECT id, name, price FROM products ORDER BY {sort_column}"
```

The interpolated identifier comes from a fixed allowlist, not directly
from user input. Continue to bind ordinary values separately.

## 8. Test potential SQL injection findings

After code review identifies a suspicious input path, verify it in an
authorized local or staging environment.

### Establish a baseline

For the deliberately vulnerable local training endpoint:

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category=Electronics"
```

Record the status, response body, number of records, response time, and
relevant log entries.

### Compare controlled inputs

These examples are for the local training lab endpoint, not arbitrary
websites:

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category='"
```

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category=' OR '1'='1' -- "
```

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/vulnerable/products" --data-urlencode "category=' AND '1'='2' -- "
```

Compare the normal request with the quote-containing, true-condition,
and false-condition requests. Correlate the HTTP result with application
and database logs. Payload behaviour depends on the database engine and
exact query structure.

### Verify the safe endpoint

``` powershell
curl.exe -i -G "http://127.0.0.1:8000/safe/products" --data-urlencode "category=' OR '1'='1' -- "
```

A correctly parameterized query treats the entire value as data. It
should not turn the condition into executable SQL.

A changed response alone is not conclusive. Strong verification combines
source-code data flow, query construction, and controlled runtime
evidence.

## 9. Common code-review warning signs

Review these patterns carefully when values may originate from users:

``` python
sql = "SELECT * FROM users WHERE name = '" + name + "'"
```

``` python
sql = "SELECT * FROM users WHERE name = '{}'".format(name)
```

``` python
sql = f"SELECT * FROM users WHERE name = '{name}'"
```

Also inspect raw SQL APIs, dynamic `ORDER BY` clauses, dynamic
table/column names, stored procedures that construct dynamic SQL, and
search filters assembled from request parameters.

These patterns are warning signs, not automatic proof. Confirm whether
untrusted data reaches the executed query and whether it is safely
handled.

## 10. Practical white-box workflow

1.  Prepare an owned or authorized local/staging environment with
    synthetic data.
2.  Enable useful web server, application, and database logging.
3.  Map the application's visible functionality.
4.  Inventory relevant input vectors.
5.  Search for database access using `rg` or PowerShell.
6.  Trace each suspicious input from the request to the database.
7.  Review parameter binding and dynamic SQL identifiers.
8.  Test suspected paths with controlled, non-destructive inputs.
9.  Correlate responses with application and database logs.
10. Fix unsafe construction and retest.
11. Report the source location, data flow, evidence, impact, and
    remediation.

## 11. Black-box vs white-box

  -----------------------------------------------------------------------
  Area                    Black-box               White-box
  ----------------------- ----------------------- -----------------------
  Source code             Usually unavailable     Available

  Viewpoint               Outside the application Inside the application

  Main activity           Send requests and       Trace inputs and
                          compare responses       inspect query
                                                  construction

  Logs                    May be unavailable      Web, application, and
                                                  database logs can be
                                                  correlated

  Evidence                Observable behaviour    Code path plus
                                                  controlled runtime
                                                  evidence

  Limitation              Internal cause may be   Uninspected or
                          unclear                 unexecuted paths can be
                                                  missed
  -----------------------------------------------------------------------

The methods complement each other. Black-box testing discovers
externally observable behaviour; white-box testing helps explain the
internal cause and locate vulnerable code.

## 12. Final checklist

-   [ ] Testing is limited to an owned or explicitly authorized
    environment.
-   [ ] A resettable database with synthetic data is available.
-   [ ] Web server access and error logs are available.
-   [ ] Application logs identify the request and code path.
-   [ ] Database logging/tracing is enabled only as needed.
-   [ ] Visible functionality has been mapped.
-   [ ] Input vectors have been inventoried.
-   [ ] Database access points have been searched.
-   [ ] Suspicious inputs have been traced to query execution.
-   [ ] Values use the correct driver-specific parameter placeholders.
-   [ ] Dynamic identifiers are selected from strict allowlists.
-   [ ] Tests are compared with a normal baseline.
-   [ ] Logs and responses are correlated before conclusions are drawn.
-   [ ] Findings include source locations and safe evidence.
-   [ ] Fixes have been retested.

## Key principle

White-box SQL injection testing is not merely searching for the word
`SELECT`. It is tracing untrusted data from its entry point to the
database, understanding how the query is constructed, and verifying in a
controlled environment whether the database interprets that data as SQL
syntax.

The strongest finding combines a clear source-code data flow, an unsafe
query construction pattern, and safe runtime evidence.
