User Service Spring Boot Application
This document explains how to set up and run the User Service Spring Boot CRUD application with PostgreSQL on Ubuntu 22.04.

1. Prerequisites
Ubuntu 22.04
Internet connection
Sudo privileges
2. Install Java 21 (JDK21)
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
# should show: openjdk version "21.x.x"
3. Install Maven
sudo apt install -y maven
mvn -version
# should show Maven 4.x or similar
4. Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo -u postgres psql -c "SELECT version();"
5. Create Database and User
# Login as postgres
sudo -u postgres psql

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
6. Configure Spring Boot Application
In src/main/resources/application.properties:

spring.datasource.url=jdbc:postgresql://localhost:5432/springcrud
spring.datasource.username=springuser
spring.datasource.password=Spring@123
spring.jpa.hibernate.ddl-auto=create
spring.jpa.show-sql=true
Note: ddl-auto=create allows Hibernate to auto-create tables.

7. Build the Application
# Navigate to project root
cd ~/user-service

# Build using Maven
mvn clean package

# Check target directory
ls target/
# You should see user-service-1.0.jar
8. Run the Application
java -jar target/user-service-1.0.jar
Application will start on http://localhost:8080.
Hibernate will auto-create the users table in PostgreSQL.
Check in PostgreSQL:
sudo -u postgres psql -d springcrud
\dt
