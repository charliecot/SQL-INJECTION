# Database Shell and Docker Reference

## PostgreSQL, MySQL, MongoDB, Redis and MariaDB

This guide explains how to run each database in two ways: 1. **Docker
image/container** --- the database server runs inside a container. 2.
**Direct installation** --- the database server is installed on Windows
or Linux/Ubuntu.

Examples use Windows 11, PowerShell, Ubuntu/WSL and Docker Desktop. Use
them on your own machine or an authorized lab.

------------------------------------------------------------------------

# 1. Understand the pieces

-   **Database server:** the process that stores data and answers
    queries.
-   **Database client:** a shell program that connects to the server
    (`psql`, `mysql`, `mongosh`, `redis-cli`, `mariadb`).
-   **Image:** a packaged template from which Docker creates a
    container.
-   **Container:** an isolated running or stopped instance created from
    an image.
-   **Port:** the network endpoint used by a client to reach a server.
-   **Volume:** persistent storage that keeps database files when a
    container is replaced.

An image is not a running database. You create a container from the
image.

## How connections work

If the database runs in Docker and your application runs directly on
Windows/Ubuntu, connect through the published host port:

``` text
Host: localhost
Port: the left-hand side of Docker's -p HOST_PORT:CONTAINER_PORT
```

If both your application and database are containers on the same Docker
Compose network, use the service name as the host, for example
`postgres:5432`.

Inside a container, `localhost` means that same container. It does not
mean another container or your Windows host.

------------------------------------------------------------------------

# 2. Prepare Docker

Install Docker Desktop for Windows:
https://www.docker.com/products/docker-desktop/

Start Docker Desktop, enable its WSL 2 backend if prompted, and enable
WSL integration for your Ubuntu distribution.

Check from PowerShell or an integrated WSL terminal:

``` bash
docker --version
docker compose version
docker info
```

If `docker info` cannot connect to the daemon, start Docker Desktop.

## Common Docker commands

``` bash
docker pull IMAGE:TAG             # Download an image
docker image ls                   # List downloaded images
docker ps                         # Running containers
docker ps -a                      # All containers, including stopped
docker logs CONTAINER              # Show logs
docker logs -f CONTAINER           # Follow logs
docker stop CONTAINER              # Stop
docker start CONTAINER             # Start an existing container
docker restart CONTAINER           # Restart
docker exec -it CONTAINER COMMAND  # Run a command inside it
docker inspect CONTAINER           # Inspect configuration
docker rm CONTAINER               # Remove a stopped container
docker rm -f CONTAINER             # Force-remove a container
docker volume ls                  # List persistent volumes
```

Do not remove a database volume just to restart a container.

------------------------------------------------------------------------

# 3. PostgreSQL

PostgreSQL is a relational SQL database.

-   Default port: `5432`
-   Shell client: `psql`
-   Docker image: `postgres`
-   Python driver: `psycopg`

## 3.1 Docker installation

Download:

``` bash
docker pull postgres:18
```

Run:

``` bash
docker run -d \
  --name my-postgres \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD='ChangeThisPassword123!' \
  -e POSTGRES_DB=shop_db \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql \
  postgres:18
```

Meaning:

  Option                                   Meaning
  ---------------------------------------- ------------------------------------
  `-d`                                     Run in background
  `--name`                                 Container name
  `POSTGRES_USER`                          Initial database superuser
  `POSTGRES_PASSWORD`                      Password for that user
  `POSTGRES_DB`                            Initial database
  `-p 5432:5432`                           Host port mapped to container port
  `-v postgres_data:/var/lib/postgresql`   Persistent named volume
  `postgres:18`                            Image and major version

**PostgreSQL 18 note:** the official image uses `/var/lib/postgresql` as
its volume target for version 18. Older major versions use a different
data-directory layout; follow the matching image documentation rather
than reusing paths blindly.

Check:

``` bash
docker ps
docker logs my-postgres
```

Enter `psql` inside the container:

``` bash
docker exec -it my-postgres psql -U admin -d shop_db
```

Connect from a client installed on the host:

``` bash
psql -h localhost -p 5432 -U admin -d shop_db
```

`-h` host, `-p` port, `-U` database user, `-d` database.

## 3.2 Useful `psql` commands

These begin with a backslash and are `psql` client commands, not SQL
statements.

  Command             Meaning
  ------------------- --------------------------------------
  `\?`                List client commands
  `\h`                SQL help
  `\h SELECT`         Help for SELECT
  `\l`                List databases
  `\c shop_db`        Connect to a database
  `\conninfo`         Current connection details
  `\du`               List roles/users
  `\dt`               List tables
  `\dt *.*`           List tables across schemas
  `\d students`       Describe a table
  `\dn`               List schemas
  `\dx`               List extensions
  `\password admin`   Change a role password interactively
  `\q`                Quit

## 3.3 PostgreSQL SQL practice

``` sql
CREATE DATABASE school_db;

-- In psql, connect to it:
-- \c school_db

CREATE TABLE students (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    age INTEGER CHECK (age >= 0),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO students (full_name, age)
VALUES ('Charles', 24), ('Alice', 22);

SELECT * FROM students;

SELECT full_name, age
FROM students
WHERE age >= 23
ORDER BY full_name;

UPDATE students SET age = 25 WHERE full_name = 'Charles';
DELETE FROM students WHERE full_name = 'Alice';
```

## 3.4 Direct installation on Ubuntu/WSL

``` bash
sudo apt update
sudo apt install postgresql postgresql-client
```

Check and manage the service:

``` bash
sudo systemctl status postgresql
sudo systemctl start postgresql
sudo systemctl stop postgresql
sudo systemctl restart postgresql
```

If systemd is not enabled in WSL, try:

``` bash
sudo service postgresql status
sudo service postgresql start
```

Open the local PostgreSQL administrative account:

``` bash
sudo -u postgres psql
```

Create an application role and database:

``` sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'UseA_Strong_Password';
CREATE DATABASE app_db OWNER app_user;
```

Connect:

``` bash
psql -h 127.0.0.1 -p 5432 -U app_user -d app_db
```

Local PostgreSQL may use peer authentication for Unix-socket
connections. TCP connections such as `-h localhost` may use password
authentication according to `pg_hba.conf`.

## 3.5 PostgreSQL connection string and Python

Connection URI:

``` text
postgresql://USERNAME:PASSWORD@HOST:PORT/DATABASE
```

Example:

``` text
postgresql://app_user:YourPassword@localhost:5432/app_db
```

Django settings:

``` python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "app_db",
        "USER": "app_user",
        "PASSWORD": "YourPassword",
        "HOST": "127.0.0.1",
        "PORT": "5432",
    }
}
```

Install the driver in your virtual environment:

``` bash
python -m pip install "psycopg[binary]"
```

------------------------------------------------------------------------

# 4. MySQL

MySQL is a relational SQL database.

-   Default port: `3306`
-   Shell client: `mysql`
-   Docker image: `mysql`
-   Python drivers: `mysql-connector-python` or `PyMySQL`

## 4.1 Docker installation

``` bash
docker pull mysql:8.4
```

Run:

``` bash
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD='RootPassword123!' \
  -e MYSQL_DATABASE=shop_db \
  -e MYSQL_USER=app_user \
  -e MYSQL_PASSWORD='AppPassword123!' \
  -p 3306:3306 \
  -v mysql_data:/var/lib/mysql \
  mysql:8.4
```

The first initialization may take a little time.

``` bash
docker ps
docker logs my-mysql
```

Enter the shell inside the container:

``` bash
docker exec -it my-mysql mysql -u app_user -p shop_db
```

Or as root:

``` bash
docker exec -it my-mysql mysql -u root -p
```

Connect from a host-installed client:

``` bash
mysql -h 127.0.0.1 -P 3306 -u app_user -p shop_db
```

Notice uppercase `-P` means port; lowercase `-p` prompts for a password.

## 4.2 MySQL shell commands

  Command                          Meaning
  -------------------------------- -------------------------------------
  `help`                           Client help
  `status`                         Connection/server status
  `SHOW DATABASES;`                List databases
  `USE shop_db;`                   Select a database
  `SHOW TABLES;`                   List tables
  `DESCRIBE students;`             Show table columns
  `SHOW CREATE TABLE students\G`   Display table definition vertically
  `SELECT USER();`                 Connected account
  `SELECT VERSION();`              Server version
  `EXIT;`                          Quit

## 4.3 MySQL SQL practice

``` sql
CREATE DATABASE school_db;
USE school_db;

CREATE TABLE students (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    age INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO students (full_name, age)
VALUES ('Charles', 24), ('Alice', 22);

SELECT * FROM students;

SELECT full_name, age
FROM students
WHERE age >= 23;

UPDATE students SET age = 25 WHERE full_name = 'Charles';
DELETE FROM students WHERE full_name = 'Alice';
```

## 4.4 Direct installation on Ubuntu/WSL

``` bash
sudo apt update
sudo apt install mysql-server mysql-client
```

Manage the service:

``` bash
sudo systemctl status mysql
sudo systemctl start mysql
sudo systemctl enable mysql
```

If systemd is unavailable:

``` bash
sudo service mysql status
sudo service mysql start
```

Open the local administrative client:

``` bash
sudo mysql
```

On many Ubuntu installations, the local MySQL root account uses socket
authentication, which is why `sudo mysql` may work without a root
database password.

Create a database and dedicated user:

``` sql
CREATE DATABASE app_db;

CREATE USER 'app_user'@'localhost'
IDENTIFIED BY 'UseA_Strong_Password';

GRANT ALL PRIVILEGES ON app_db.*
TO 'app_user'@'localhost';

FLUSH PRIVILEGES;
```

Connect:

``` bash
mysql -u app_user -p app_db
```

Or explicitly use TCP:

``` bash
mysql -h 127.0.0.1 -P 3306 -u app_user -p app_db
```

## 4.5 MySQL Python connection

``` bash
python -m pip install mysql-connector-python
```

``` python
import mysql.connector

connection = mysql.connector.connect(
    host="127.0.0.1",
    port=3306,
    user="app_user",
    password="YourPassword",
    database="app_db",
)

try:
    cursor = connection.cursor()
    cursor.execute("SELECT VERSION()")
    print(cursor.fetchone())
finally:
    cursor.close()
    connection.close()
```

------------------------------------------------------------------------

# 5. MongoDB

MongoDB is a document-oriented NoSQL database. It stores BSON documents
in collections rather than rows in relational tables.

-   Default port: `27017`
-   Shell: `mongosh`
-   Docker image: `mongo`
-   Python driver: `pymongo`

## 5.1 Docker installation

``` bash
docker pull mongo:8
```

Run with authentication:

``` bash
docker run -d \
  --name my-mongo \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD='MongoPassword123!' \
  -p 27017:27017 \
  -v mongo_data:/data/db \
  mongo:8
```

Check:

``` bash
docker ps
docker logs my-mongo
```

Enter `mongosh` inside the container:

``` bash
docker exec -it my-mongo mongosh \
  -u admin \
  -p \
  --authenticationDatabase admin
```

The shell prompts for the password.

Connect using a host-installed `mongosh`:

``` bash
mongosh "mongodb://admin:YOUR_PASSWORD@localhost:27017/?authSource=admin"
```

If a password contains URL-special characters, URL-encode them in the
URI.

## 5.2 Useful `mongosh` commands

  Command                             Meaning
  ----------------------------------- ----------------------------------------------------
  `show dbs`                          List databases containing data
  `use shop_db`                       Switch database; it is created when data is stored
  `db`                                Show current database
  `show collections`                  List collections
  `db.createCollection("students")`   Create collection
  `db.students.find()`                Read documents
  `db.students.countDocuments()`      Count documents
  `db.stats()`                        Database statistics
  `db.version()`                      Server version
  `exit`                              Quit

## 5.3 MongoDB practice

``` javascript
use school_db

db.students.insertOne({
    name: "Charles",
    age: 24,
    course: "Computer Science"
})

db.students.insertMany([
    { name: "Alice", age: 22, course: "Networking" },
    { name: "John", age: 25, course: "Cybersecurity" }
])

db.students.find()

db.students.find({ age: { $gte: 23 } })

db.students.find(
    { course: "Cybersecurity" },
    { name: 1, age: 1, _id: 0 }
)

db.students.updateOne(
    { name: "Charles" },
    { $set: { age: 25 } }
)

db.students.deleteOne({ name: "Alice" })
```

## 5.4 Direct installation on Ubuntu/WSL

MongoDB Community Server may not be in Ubuntu's default APT
repositories. Follow MongoDB's official instructions for the exact
Ubuntu release:

https://www.mongodb.com/docs/manual/administration/install-on-linux/

The general procedure is to add MongoDB's official signing key and
repository, update APT, and install the server package.

Typical service commands after installation:

``` bash
sudo systemctl status mongod
sudo systemctl start mongod
sudo systemctl enable mongod
```

If systemd is unavailable in WSL:

``` bash
sudo service mongod status
sudo service mongod start
```

Open the shell:

``` bash
mongosh
```

## 5.5 MongoDB Python connection

``` bash
python -m pip install pymongo
```

``` python
from pymongo import MongoClient

client = MongoClient(
    "mongodb://admin:YourPassword@localhost:27017/?authSource=admin"
)

try:
    database = client["school_db"]
    students = database["students"]

    result = students.insert_one({
        "name": "Charles",
        "age": 24,
    })

    print("Inserted ID:", result.inserted_id)
    print(list(students.find()))
finally:
    client.close()
```

------------------------------------------------------------------------

# 6. Redis

Redis is an in-memory data store commonly used for caching, counters,
sessions, queues and fast temporary data. Persistence can be configured.

-   Default port: `6379`
-   Shell: `redis-cli`
-   Docker image: `redis`
-   Python client: `redis`

## 6.1 Docker installation

``` bash
docker pull redis:8
```

Run with AOF persistence:

``` bash
docker run -d \
  --name my-redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:8 \
  redis-server --appendonly yes
```

Check:

``` bash
docker ps
docker logs my-redis
```

Enter Redis CLI inside the container:

``` bash
docker exec -it my-redis redis-cli
```

Connect using a host-installed client:

``` bash
redis-cli -h 127.0.0.1 -p 6379
```

This example has no password configured. Keep development Redis bound to
localhost and do not expose an unauthenticated server to an untrusted
network.

## 6.2 Redis CLI commands

  Command              Meaning
  -------------------- ---------------------------------------------------------
  `PING`               Test server response
  `INFO`               Server information
  `DBSIZE`             Number of keys in selected DB
  `SELECT 0`           Select logical database 0
  `SET name Charles`   Store a string
  `GET name`           Read a string
  `EXPIRE name 60`     Expire key after 60 seconds
  `TTL name`           Remaining expiry time
  `DEL name`           Delete key
  `EXISTS name`        Check whether key exists
  `KEYS *`             Find matching keys; avoid on large production databases
  `SCAN 0`             Iterate keys incrementally
  `FLUSHDB`            Delete all keys in selected DB
  `FLUSHALL`           Delete keys in all logical databases
  `QUIT`               Exit

## 6.3 Redis practice

``` redis
PING

SET username Charles
GET username

SET session:123 active EX 60
TTL session:123

HSET user:1 name Charles age 24
HGETALL user:1

LPUSH tasks "send email"
LPUSH tasks "create report"
LRANGE tasks 0 -1

SADD skills Python SQL Linux
SMEMBERS skills

DEL username
```

Common data structures:

  Type         Commands                     Typical use
  ------------ ---------------------------- ------------------------
  String       `SET`, `GET`, `INCR`         Cache values, counters
  Hash         `HSET`, `HGETALL`            Object-like fields
  List         `LPUSH`, `RPUSH`, `LRANGE`   Ordered items, queues
  Set          `SADD`, `SMEMBERS`           Unique values
  Sorted set   `ZADD`, `ZRANGE`             Rankings
  Stream       `XADD`, `XREAD`              Event streams

## 6.4 Direct installation on Ubuntu/WSL

``` bash
sudo apt update
sudo apt install redis-server redis-tools
```

Manage:

``` bash
sudo systemctl status redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

If systemd is unavailable:

``` bash
sudo service redis-server status
sudo service redis-server start
```

Connect:

``` bash
redis-cli
```

Test:

``` redis
PING
```

Expected response:

``` text
PONG
```

## 6.5 Redis Python connection

``` bash
python -m pip install redis
```

``` python
import redis

client = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True,
)

try:
    print(client.ping())
    client.set("username", "Charles", ex=60)
    print(client.get("username"))
finally:
    client.close()
```

------------------------------------------------------------------------

# 7. MariaDB

MariaDB is a relational database and a MySQL-compatible alternative.
Many everyday MySQL clients and SQL statements work with MariaDB, but
compatibility is not identical for every feature and version.

-   Default port: `3306`
-   Shell: `mariadb` (sometimes `mysql` is also available)
-   Docker image: `mariadb`
-   Python drivers: `mariadb` or `PyMySQL`

## 7.1 Docker installation

``` bash
docker pull mariadb:11
```

Run:

``` bash
docker run -d \
  --name my-mariadb \
  -e MARIADB_ROOT_PASSWORD='RootPassword123!' \
  -e MARIADB_DATABASE=shop_db \
  -e MARIADB_USER=app_user \
  -e MARIADB_PASSWORD='AppPassword123!' \
  -p 3307:3306 \
  -v mariadb_data:/var/lib/mysql \
  mariadb:11
```

Host port `3307` maps to container port `3306`. This avoids a host-port
conflict if MySQL already uses `3306`.

Check:

``` bash
docker ps
docker logs my-mariadb
```

Enter the shell:

``` bash
docker exec -it my-mariadb mariadb -u app_user -p shop_db
```

As root:

``` bash
docker exec -it my-mariadb mariadb -u root -p
```

Connect from a host client:

``` bash
mariadb -h 127.0.0.1 -P 3307 -u app_user -p shop_db
```

## 7.2 MariaDB shell and SQL

``` sql
SHOW DATABASES;
USE shop_db;
SHOW TABLES;
DESCRIBE students;
SELECT VERSION();
SELECT USER();
```

Practice:

``` sql
CREATE TABLE students (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INT
);

INSERT INTO students (name, age)
VALUES ('Charles', 24);

SELECT * FROM students;
```

Exit:

``` sql
EXIT;
```

## 7.3 Direct installation on Ubuntu/WSL

``` bash
sudo apt update
sudo apt install mariadb-server mariadb-client
```

Manage:

``` bash
sudo systemctl status mariadb
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

If systemd is unavailable:

``` bash
sudo service mariadb status
sudo service mariadb start
```

Open the local administrative shell:

``` bash
sudo mariadb
```

Create a database and application user:

``` sql
CREATE DATABASE app_db;

CREATE USER 'app_user'@'localhost'
IDENTIFIED BY 'UseA_Strong_Password';

GRANT ALL PRIVILEGES ON app_db.*
TO 'app_user'@'localhost';

FLUSH PRIVILEGES;
```

Connect:

``` bash
mariadb -u app_user -p app_db
```

## 7.4 MariaDB Python connection

``` bash
python -m pip install mariadb
```

``` python
import mariadb

connection = mariadb.connect(
    host="127.0.0.1",
    port=3307,  # Use 3306 for a normal direct installation
    user="app_user",
    password="YourPassword",
    database="app_db",
)

try:
    cursor = connection.cursor()
    cursor.execute("SELECT VERSION()")
    print(cursor.fetchone())
finally:
    cursor.close()
    connection.close()
```

------------------------------------------------------------------------

# 8. Compare the databases

  -----------------------------------------------------------------------------------
  Database      Type               Default port Shell         Common data directory
                                                              in Docker
  ------------- ------------- ----------------- ------------- -----------------------
  PostgreSQL    Relational                 5432 `psql`        `/var/lib/postgresql`
                SQL                                           for image version 18

  MySQL         Relational                 3306 `mysql`       `/var/lib/mysql`
                SQL                                           

  MongoDB       Document                  27017 `mongosh`     `/data/db`
                NoSQL                                         

  Redis         In-memory                  6379 `redis-cli`   `/data`
                data store                                    

  MariaDB       Relational                 3306 `mariadb`     `/var/lib/mysql`
                SQL                                           
  -----------------------------------------------------------------------------------

Suggested learning order for a Python/Django developer:

1.  PostgreSQL --- relational design, joins, constraints, indexes,
    transactions and Django.
2.  MySQL or MariaDB --- compare another SQL database.
3.  MongoDB --- documents, collections and nested data.
4.  Redis --- caching, expiry, counters, queues and background tasks.

You do not need to run all five at once.

------------------------------------------------------------------------

# 9. Troubleshooting connections

## 9.1 Connection refused

Possible causes:

-   Container or service is stopped.
-   Wrong host or port.
-   Docker port was not published.
-   Server is listening on a different network interface.

Check Docker:

``` bash
docker ps
docker logs CONTAINER_NAME
```

Check Ubuntu services:

``` bash
sudo systemctl status postgresql
sudo systemctl status mysql
sudo systemctl status mongod
sudo systemctl status redis-server
sudo systemctl status mariadb
```

Check listening ports:

``` bash
sudo ss -lntp
```

## 9.2 Authentication failed

Check the username, password, database name, MongoDB authentication
database, MySQL/MariaDB host-based account rules and PostgreSQL
`pg_hba.conf`.

Do not fix authentication by disabling it or granting unrestricted
access.

## 9.3 Address already in use

If local MySQL already uses host port `3306`, map a Docker MySQL
container to host port `3307`:

``` bash
docker run -d \
  --name second-mysql \
  -e MYSQL_ROOT_PASSWORD='AnotherStrongPassword!' \
  -p 3307:3306 \
  mysql:8.4
```

Connect using the host port:

``` bash
mysql -h 127.0.0.1 -P 3307 -u root -p
```

The database still listens on port `3306` inside the container.

## 9.4 Container exits

Inspect:

``` bash
docker logs CONTAINER_NAME
```

Possible causes include missing required environment variables, invalid
configuration, incompatible data directories or permissions.

## 9.5 Data seems missing after removing a container

Inspect volumes:

``` bash
docker volume ls
docker inspect CONTAINER_NAME
```

A named volume normally survives container removal. Recreate the
container with the same volume and compatible database image/version. Do
not delete the volume unless you intend to erase the stored data.

------------------------------------------------------------------------

# 10. Docker Compose: PostgreSQL and Redis together

Compose describes multiple services in one YAML file. This is useful for
a Django project that needs PostgreSQL and Redis.

Create a folder:

``` bash
mkdir database-lab
cd database-lab
```

Create a file named `compose.yaml`:

``` yaml
services:
  postgres:
    image: postgres:18
    container_name: lab-postgres
    environment:
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: change_this_password
      POSTGRES_DB: app_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql
    restart: unless-stopped

  redis:
    image: redis:8
    container_name: lab-redis
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

For real projects, keep passwords in an environment file excluded from
Git or use a secrets-management solution. Do not commit production
credentials.

Start:

``` bash
docker compose up -d
```

Inspect:

``` bash
docker compose ps
docker compose logs
docker compose logs -f postgres
```

Enter PostgreSQL:

``` bash
docker compose exec postgres psql -U app_user -d app_db
```

Enter Redis:

``` bash
docker compose exec redis redis-cli
```

Stop containers but keep volumes:

``` bash
docker compose down
```

Stop and delete named volumes (this deletes the stored database data):

``` bash
docker compose down -v
```

**Warning:** `docker compose down -v` is destructive. Use it only when
you deliberately want to erase the persistent data.

Within this Compose network, an application container connects to
PostgreSQL at host `postgres`, port `5432`, and Redis at host `redis`,
port `6379`. An application running directly on Windows connects through
`localhost` and the published ports.

------------------------------------------------------------------------

# 11. Security and good practices

1.  Use strong, unique passwords.
2.  Do not use a database superuser in your application; create a
    restricted application account.
3.  Do not expose database ports publicly without a deliberate security
    design.
4.  Use parameterized SQL; never concatenate untrusted input into SQL.
5.  Store Docker database data in named volumes.
6.  Back up important data. A volume is persistence, not a backup.
7.  Keep credentials out of source code; use environment variables or a
    secrets manager.
8.  Use supported image versions and plan upgrades.
9.  Do not casually delete volumes.
10. Apply least privilege: give each application only the permissions it
    needs.

## Parameterized SQL examples

Unsafe:

``` python
# Do not build SQL this way with untrusted input.
query = f"SELECT * FROM users WHERE username = '{username}'"
```

Safer with PostgreSQL `psycopg`:

``` python
cursor.execute(
    "SELECT * FROM users WHERE username = %s",
    (username,),
)
```

Safer with MySQL Connector:

``` python
cursor.execute(
    "SELECT * FROM users WHERE username = %s",
    (username,),
)
```

Placeholder syntax depends on the driver. Follow the documentation for
the driver you use.

------------------------------------------------------------------------

# 12. Hands-on learning sequence

## Exercise 1 --- PostgreSQL

-   Run the Docker container.
-   Enter `psql`.
-   Create a database and table.
-   Insert, select, update and delete records.
-   Connect from Python.
-   Stop and restart the container.
-   Confirm your data remains.

## Exercise 2 --- MySQL and MariaDB

-   Run MySQL on host port `3306`.
-   Run MariaDB on host port `3307`.
-   Connect to each using its client.
-   Compare SQL statements and user creation.

## Exercise 3 --- MongoDB

-   Run MongoDB.
-   Create a collection.
-   Insert and query documents.
-   Update and delete documents.
-   Connect from Python with PyMongo.

## Exercise 4 --- Redis

-   Run Redis.
-   Practise strings, hashes, lists, sets and expiry.
-   Connect from Python.
-   Store a value with a TTL and observe it expire.

## Exercise 5 --- Compose

-   Run PostgreSQL and Redis together.
-   Connect to both from a Python application.
-   Stop and restart the services.
-   Verify that persistent data remains.

------------------------------------------------------------------------

# 13. Official documentation

-   Docker Desktop: https://docs.docker.com/desktop/
-   Docker `run`:
    https://docs.docker.com/reference/cli/docker/container/run/
-   Docker volumes: https://docs.docker.com/engine/storage/volumes/
-   Docker Compose: https://docs.docker.com/compose/
-   PostgreSQL: https://www.postgresql.org/docs/
-   PostgreSQL Docker image: https://hub.docker.com/\_/postgres
-   MySQL: https://dev.mysql.com/doc/
-   MySQL Docker image: https://hub.docker.com/\_/mysql
-   MongoDB: https://www.mongodb.com/docs/
-   MongoDB Linux installation:
    https://www.mongodb.com/docs/manual/administration/install-on-linux/
-   MongoDB Docker image: https://hub.docker.com/\_/mongo
-   Redis: https://redis.io/docs/latest/
-   Redis Docker image: https://hub.docker.com/\_/redis
-   MariaDB: https://mariadb.com/docs/
-   MariaDB Docker image: https://hub.docker.com/\_/mariadb

------------------------------------------------------------------------

## Final concept to remember

A database has two separate parts:

1.  **Server:** stores and manages data.
2.  **Client:** connects to the server and sends commands.

Docker changes how you run and isolate the server. It does not change
the basic idea of connecting to a database and issuing SQL, MongoDB or
Redis commands.
