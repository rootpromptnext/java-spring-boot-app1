# User Service Spring Boot Application

This document explains how to set up and run the User Service Spring Boot CRUD application with PostgreSQL on Ubuntu 22.04.

---

## Prerequisites

- Ubuntu 22.04  
- Internet connection  
- Sudo privileges  

---

## Install Java 21 (JDK21)

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
# should show: openjdk version "21.x.x"
````

---

## Install Maven

```bash
sudo apt install -y maven
mvn -version
# should show Maven 4.x or similar
```

---

## Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo -u postgres psql -c "SELECT version();"
```

---

## Create Database and User

Login as `postgres`:

```bash
sudo -u postgres psql
```

Create database and user:

```sql
-- Create database
CREATE DATABASE springcrud;

-- Create user for application
CREATE USER springuser WITH PASSWORD 'Spring@123';

-- Grant privileges on database
GRANT ALL PRIVILEGES ON DATABASE springcrud TO springuser;

-- Grant privileges on public schema
\c springcrud
GRANT ALL ON SCHEMA public TO springuser;

-- Allow default privileges on future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO springuser;

-- Exit PostgreSQL
\q
```

---

## Configure Spring Boot Application

Edit `src/main/resources/application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/springcrud
spring.datasource.username=springuser
spring.datasource.password=Spring@123
spring.jpa.hibernate.ddl-auto=create
spring.jpa.show-sql=true
```

> **Note:** `ddl-auto=create` allows Hibernate to auto-create tables.

---

## Build the Application

Navigate to project root and build:

```bash
git clone https://github.com/rootpromptnext/java-spring-boot-app1
cd ~/java-spring-boot-app1
mvn clean package
```

Check the target directory:

```bash
ls target/
# You should see target/java-spring-boot-crud-0.0.1-SNAPSHOT.jar
```

---

## Run the Application manually

```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=springcrud
export DB_USER=springuser
export DB_PASSWORD=Spring@123

java -jar target/java-spring-boot-crud-0.0.1-SNAPSHOT.jar

```

* The application will start on [http://localhost:8080](http://localhost:8080).
* Hibernate will auto-create the `users` table in PostgreSQL.

Check the table in PostgreSQL:

```bash
sudo -u postgres psql -d springcrud
\dt
```
## Run the application using docker compose

```bash
# Build the Docker image locally
docker build -t java-springboot-app1:latest .
docker compose up -d
```

## Connect database
```bash
prayag@devops-vm:~/java-spring-boot-app1$ docker exec -it spring-postgres psql -U springuser -d springcrud
psql (16.13 (Debian 16.13-1.pgdg13+1))
Type "help" for help.

springcrud=# \du
                              List of roles
 Role name  |                         Attributes
------------+------------------------------------------------------------
 springuser | Superuser, Create role, Create DB, Replication, Bypass RLS

springcrud=# \l
                                                          List of databases
    Name    |   Owner    | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |     Access privileges
------------+------------+----------+-----------------+------------+------------+------------+-----------+---------------------------
 postgres   | springuser | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 springcrud | springuser | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0  | springuser | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/springuser            +
            |            |          |                 |            |            |            |           | springuser=CTc/springuser
 template1  | springuser | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/springuser            +
            |            |          |                 |            |            |            |           | springuser=CTc/springuser
(4 rows)

springcrud=#
```
