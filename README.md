# PostgreStore: Product & Inventory Catalog REST API

REST API for managing a product and inventory catalog (CRUD), built with **Java 21**, **Spring Boot 3.3** and **PostgreSQL 16**. It uses Spring Data JPA, Flyway migrations, HikariCP, Docker Compose and RFC 7807 `ProblemDetail` error responses.

---

## Features

- **Layered architecture** (`controller` → `service` → `repository` → `entity` / `dto`): each layer has its own responsibility. Dependencies are injected through constructors.

- **Database migrations with Flyway** (`db/migration/V1__create_products_table.sql`): the schema is defined in SQL, with UUID primary keys, `NUMERIC(10,2)` for prices, check constraints (`price >= 0`, `stock_quantity >= 0`) and indexes. Hibernate runs with `ddl-auto: validate`, so it only checks the schema and never changes it.

- **HikariCP connection pool** (`application.yml`): pool of 10 connections with `connection-timeout: 30s`, `idle-timeout: 10m` and `max-lifetime: 30m`.

- **Optimistic locking** (`@Version`): a `version BIGINT` column managed by Hibernate detects conflicting concurrent updates of the same row.

- **JPA auditing** (`@EnableJpaAuditing`): `createdAt` and `updatedAt` are filled in automatically with `@CreatedDate` and `@LastModifiedDate`. The configuration is in a separate `JpaConfig` class, so it does not affect web slice tests.

- **Spring Data JPA repositories**:
  - derived queries: `findBySku()`, `existsBySku()`, `findByActiveTrue()`, `findByPriceBetween()`
  - case-insensitive keyword search over name and description using JPQL (`@Query`)
  - pagination and sorting with `Pageable` and `Page<T>`

- **DTOs as Java records** (`dto/`): `CreateProductRequest`, `UpdateProductRequest` and `ProductResponse` keep the entity model separate from the API. `ProductResponse.fromEntity(Product)` maps an entity to a response.

- **Validation** (Jakarta Bean Validation): `@NotBlank`, `@Size(max = 255)`, `@NotNull`, `@DecimalMin("0.0")` and `@Min(0)` on request payloads, so invalid prices, stock values and blank fields are rejected before reaching the business logic.

- **Error handling** (`exception/GlobalExceptionHandler.java`): a `@RestControllerAdvice` returns RFC 7807 `ProblemDetail` responses for `400`, `404`, `409` and `422`. Validation errors include a map of the failed fields under `errors`.

- **Tests** (25 in total):
  - **Unit tests** (`ProductServiceTest`, 8): service logic with JUnit 5, Mockito and AssertJ.
  - **Repository tests** (`ProductRepositoryTest`, 6): `@DataJpaTest` against PostgreSQL, covering derived queries, JPQL search and indexes.
  - **Controller tests** (`ProductControllerTest`, 8): `@WebMvcTest` with MockMvc, covering routing, JSON payloads, headers and RFC 7807 error responses.
  - **Integration tests** (`ProductIntegrationTest`, 2): `@SpringBootTest` with `TestRestTemplate`, sending real HTTP requests through the whole stack to PostgreSQL.
  - **Context test** (`PostgreStoreApplicationTests`): checks that the application context starts.

---

## Tech Stack

- **Language:** [Java 21](https://openjdk.org/projects/jdk/21/)
- **Framework:** [Spring Boot 3.3.5](https://spring.io/projects/spring-boot)
- **Database:** [PostgreSQL 16](https://www.postgresql.org/)
- **Persistence:** [Spring Data JPA](https://spring.io/projects/spring-data-jpa) and [Hibernate 6](https://hibernate.org/)
- **Connection pool:** [HikariCP](https://github.com/brettwooldridge/HikariCP)
- **Migrations:** [Flyway](https://flywaydb.org/)
- **Containers:** [Docker and Docker Compose](https://www.docker.com/)
- **Build tool:** [Maven Wrapper (mvnw)](https://maven.apache.org/)
- **Validation:** [Jakarta Bean Validation](https://beanvalidation.org/) (Hibernate Validator)
- **Testing:** [JUnit 5](https://junit.org/junit5/), [AssertJ](https://assertj.github.io/doc/), [Mockito](https://site.mockito.org/), [MockMvc](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html), [TestRestTemplate](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/test/web/client/TestRestTemplate.html)
- **Error format:** [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807)

---

## Project Structure

```text
src/
├── main/
│   ├── java/com/devpulse/postgrestore/
│   │   ├── PostgreStoreApplication.java         # Application entry point
│   │   ├── config/
│   │   │   └── JpaConfig.java                   # JPA auditing configuration
│   │   ├── controller/
│   │   │   └── ProductController.java           # REST controller (8 endpoints)
│   │   ├── dto/
│   │   │   ├── CreateProductRequest.java        # Request record for POST
│   │   │   ├── UpdateProductRequest.java        # Request record for PUT
│   │   │   └── ProductResponse.java             # Response record with fromEntity mapper
│   │   ├── entity/
│   │   │   └── Product.java                     # JPA entity for the products table
│   │   ├── exception/
│   │   │   ├── DuplicateResourceException.java  # Maps to 409 Conflict
│   │   │   ├── GlobalExceptionHandler.java      # @RestControllerAdvice, ProblemDetail responses
│   │   │   ├── InsufficientStockException.java  # Maps to 422 Unprocessable Entity
│   │   │   └── ResourceNotFoundException.java   # Maps to 404 Not Found
│   │   ├── repository/
│   │   │   └── ProductRepository.java           # Derived and JPQL queries
│   │   └── service/
│   │       ├── ProductService.java              # Service interface
│   │       └── ProductServiceImpl.java          # Business logic, @Transactional
│   └── resources/
│       ├── application.yml                      # HikariCP, JPA, Flyway and PostgreSQL settings
│       └── db/migration/
│           └── V1__create_products_table.sql    # Flyway migration
└── test/
    └── java/com/devpulse/postgrestore/
        ├── PostgreStoreApplicationTests.java    # Application context test
        ├── ProductIntegrationTest.java          # Integration tests (TestRestTemplate)
        ├── controller/
        │   └── ProductControllerTest.java       # MockMvc web slice tests
        ├── repository/
        │   └── ProductRepositoryTest.java       # @DataJpaTest repository tests
        └── service/
            └── ProductServiceTest.java          # Unit tests (JUnit 5, Mockito, AssertJ)
```

---

## Database Schema (`products` table)

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY,
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL,
    version BIGINT NOT NULL DEFAULT 0
);

-- Indexes
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_active ON products(active);
CREATE INDEX idx_products_price ON products(price);
```

---

## API Endpoints

Base URL: `http://localhost:8080/api/v1/products`

| Method | Endpoint | Description | Status | Response |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/products` | Create a product | `201 Created` | `ProductResponse` + `Location` header |
| `GET` | `/api/v1/products` | List products, paginated (`?page=0&size=10&activeOnly=true`) | `200 OK` | `Page<ProductResponse>` |
| `GET` | `/api/v1/products/{id}` | Get a product by UUID | `200 OK` | `ProductResponse` (or `404 ProblemDetail`) |
| `GET` | `/api/v1/products/sku/{sku}` | Get a product by SKU | `200 OK` | `ProductResponse` (or `404 ProblemDetail`) |
| `GET` | `/api/v1/products/search?query=...` | Search products by keyword, paginated | `200 OK` | `Page<ProductResponse>` |
| `PUT` | `/api/v1/products/{id}` | Update a product | `200 OK` | `ProductResponse` (or `404 ProblemDetail`) |
| `PATCH` | `/api/v1/products/{id}/stock?delta=...` | Adjust stock quantity | `200 OK` | `ProductResponse` (or `422 ProblemDetail`) |
| `DELETE` | `/api/v1/products/{id}` | Delete a product | `204 No Content` | Empty body (or `404 ProblemDetail`) |

---

## Getting Started

### Prerequisites
- Java JDK 21 or newer
- Docker and Docker Compose
- Git

### Running the project

1. Clone the repository:
   ```bash
   git clone https://github.com/rlaktis/postgre-store-api.git
   cd postgre-store-api
   ```

2. Start PostgreSQL in Docker:
   ```bash
   docker compose up -d
   ```

3. Run the tests:
   ```bash
   # Windows PowerShell
   .\mvnw.cmd test

   # macOS / Linux
   ./mvnw test
   ```

4. Start the application:
   ```bash
   # Windows PowerShell
   .\mvnw.cmd spring-boot:run

   # macOS / Linux
   ./mvnw spring-boot:run
   ```

5. The API is then available at:
   ```text
   http://localhost:8080/api/v1/products
   ```

6. Optionally, build an executable JAR:
   ```bash
   # Windows PowerShell
   .\mvnw.cmd clean package
   java -jar target/postgre-store-0.0.1-SNAPSHOT.jar
   ```

---

## Limitations and Next Steps

- No authentication or authorization.
- No generated API documentation (OpenAPI / Swagger).
- No CI pipeline yet.
