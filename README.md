# Employee CRUD REST API — PostgreSQL + Flyway Migrations

This project extends the Employee CRUD REST API by connecting it to a real relational database
(**PostgreSQL**) and managing the schema with **Flyway** migrations, including seed data.

## Tech Stack

- Java 17, Spring Boot 3.3.2
- Spring Web, Spring Data JPA (Hibernate)
- **PostgreSQL** (primary relational database)
- **Flyway** (schema migrations, versioned SQL files)
- H2 (in-memory, used only to run the same Flyway migrations quickly in tests)
- Jakarta Bean Validation, Lombok, JUnit 5 / MockMvc
- Docker Compose (spins up local PostgreSQL)

## Database Schema Design

Two related tables, managed entirely through Flyway migrations (`src/main/resources/db/migration`):

**`departments`**
| Column      | Type          | Constraints                  |
|-------------|---------------|-------------------------------|
| id          | BIGINT        | PK, identity                  |
| name        | VARCHAR(100)  | NOT NULL, UNIQUE               |
| location    | VARCHAR(100)  |                                |
| created_at  | TIMESTAMP     | NOT NULL, default now          |

**`employees`**
| Column         | Type           | Constraints                                      |
|----------------|----------------|---------------------------------------------------|
| id             | BIGINT         | PK, identity                                       |
| first_name     | VARCHAR(100)   | NOT NULL                                           |
| last_name      | VARCHAR(100)   | NOT NULL                                           |
| email          | VARCHAR(150)   | NOT NULL, UNIQUE                                   |
| salary         | NUMERIC(12,2)  | NOT NULL, CHECK (salary > 0)                       |
| department_id  | BIGINT         | FK → departments(id), ON DELETE SET NULL            |
| created_at     | TIMESTAMP      | NOT NULL, default now                              |
| updated_at     | TIMESTAMP      | NOT NULL, default now                              |

Indexes: `idx_employees_department_id`, `idx_employees_email`.

Entity relationship: **one Department → many Employees** (`@ManyToOne` on `Employee`).

## Migration Files

| File | Purpose |
|------|---------|
| `V1__create_departments_table.sql` | Creates `departments` table |
| `V2__create_employees_table.sql`   | Creates `employees` table with FK + indexes + check constraint |
| `V3__seed_sample_data.sql`         | Seeds 4 departments and 6 employees |

Flyway tracks applied migrations in the `flyway_schema_history` table and applies new, higher-versioned
scripts automatically on application startup — no manual DDL, no `ddl-auto=update` in production.
`spring.jpa.hibernate.ddl-auto` is set to `validate`, so Hibernate only checks the entity mapping
against the Flyway-managed schema and never generates or alters DDL itself.

## Configuration

`src/main/resources/application.properties` (production/default profile — PostgreSQL):
```properties
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/employeedb}
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=validate
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect

spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
```
Override `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` as environment variables for different environments
(staging, prod, CI) without touching code.

`src/test/resources/application.properties` (test profile — H2, same migrations):
```properties
spring.datasource.url=jdbc:h2:mem:employeedb_test;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
```
Tests run the **exact same** Flyway SQL scripts against H2 in PostgreSQL-compatibility mode, so
schema drift between test and production is caught automatically.

## Getting Started

### Prerequisites
- Java 17+, Maven 3.6+
- Docker (for local PostgreSQL) — or an existing PostgreSQL instance

### 1. Start PostgreSQL
```bash
docker compose up -d
```
This starts PostgreSQL 16 on `localhost:5432` with database `employeedb`, user `postgres`, password `postgres`.

### 2. Run the application
```bash
mvn spring-boot:run
```
On startup, Flyway automatically applies `V1`, `V2`, `V3` — creating the schema and inserting seed data.
The API is available at `http://localhost:8080`.

### 3. Run tests
```bash
mvn test
```
Tests spin up an H2 in-memory database, run the same Flyway migrations, and verify the seeded data plus
full CRUD behavior.

## API Endpoints

| Method | Endpoint                     | Description                    |
|--------|-------------------------------|----------------------------------|
| GET    | `/api/v1/departments`         | List all departments (for `departmentId` lookup) |
| POST   | `/api/v1/employees`           | Create employee — 201            |
| GET    | `/api/v1/employees`           | List all employees — 200         |
| GET    | `/api/v1/employees/{id}`      | Get one employee — 200           |
| PUT    | `/api/v1/employees/{id}`      | Update employee — 200            |
| DELETE | `/api/v1/employees/{id}`      | Delete employee — 204            |

`EmployeeRequestDTO` now takes `departmentId` (a foreign key reference) instead of a free-text department
string, so employees are always linked to a valid seeded/created department. An unknown `departmentId`
returns `404 Not Found`.

## Sample curl Commands

```bash
# List departments (to get a valid departmentId)
curl http://localhost:8080/api/v1/departments

# Create an employee
curl -X POST http://localhost:8080/api/v1/employees \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Jane","lastName":"Doe","email":"jane.newhire@example.com","salary":75000,"departmentId":1}'

# Get all employees
curl http://localhost:8080/api/v1/employees

# Update an employee
curl -X PUT http://localhost:8080/api/v1/employees/1 \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Jane","lastName":"Doe","email":"jane.newhire@example.com","salary":80000,"departmentId":2}'

# Delete an employee
curl -X DELETE http://localhost:8080/api/v1/employees/1 -i
```

Import [`postman_collection.json`](./postman_collection.json) for the same requests in Postman.

## Sample SQL Queries

```sql
-- All employees with their department name
SELECT e.id, e.first_name, e.last_name, e.email, e.salary, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
ORDER BY e.id;

-- Average salary per department
SELECT d.name AS department, ROUND(AVG(e.salary), 2) AS avg_salary, COUNT(e.id) AS headcount
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name
ORDER BY avg_salary DESC;

-- Highest paid employee per department
SELECT DISTINCT ON (d.name) d.name AS department, e.first_name, e.last_name, e.salary
FROM departments d
JOIN employees e ON e.department_id = d.id
ORDER BY d.name, e.salary DESC;

-- Employees hired (created) in the last 30 days
SELECT first_name, last_name, created_at
FROM employees
WHERE created_at >= NOW() - INTERVAL '30 days'
ORDER BY created_at DESC;

-- Departments with no employees
SELECT d.name
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
WHERE e.id IS NULL;
```

## Project Structure

```
src/main/java/com/example/employeeapi/
 ├── EmployeeApiApplication.java
 ├── controller/EmployeeController.java, DepartmentController.java
 ├── service/EmployeeService.java
 ├── repository/EmployeeRepository.java, DepartmentRepository.java
 ├── model/Employee.java, Department.java
 ├── dto/EmployeeRequestDTO.java, EmployeeResponseDTO.java, DepartmentResponseDTO.java
 └── exception/ResourceNotFoundException.java, DuplicateResourceException.java, ErrorResponse.java, GlobalExceptionHandler.java
src/main/resources/
 ├── application.properties         # PostgreSQL + Flyway config
 └── db/migration/
      ├── V1__create_departments_table.sql
      ├── V2__create_employees_table.sql
      └── V3__seed_sample_data.sql
src/test/
 ├── java/.../EmployeeControllerTests.java
 └── resources/application.properties   # H2 + same Flyway migrations
docker-compose.yml                  # local PostgreSQL
postman_collection.json
```

## Deliverables Checklist

- [x] DB schema design (`departments`, `employees`, FK relationship, indexes, constraints)
- [x] Spring Data JPA entities/repositories mapped to the migrated schema
- [x] Flyway migration files (`V1`–`V3`) applied automatically on startup
- [x] Seed data (4 departments, 6 employees) via `V3__seed_sample_data.sql`
- [x] Configuration details for PostgreSQL (prod) and H2 (test), documented above
- [x] Sample SQL queries and REST curl / Postman examples
# Employee-API-
