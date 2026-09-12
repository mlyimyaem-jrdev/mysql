# Building a Simple CRUD API in Spring Boot

The same product API as the Laravel guide, built with Spring Boot — minimal setup, XAMPP for MySQL, tested in Postman.

**Target version:** Spring Boot 4.1.x (Spring Framework 7)
**Java:** 17 minimum; 21 or 25 (both LTS) recommended
**Platform:** Windows 10 / 11, PowerShell
**Database:** MySQL/MariaDB from XAMPP
**No Docker, no Gradle install, no global Maven install**

---

## Table of Contents

1. [What you'll build](#1-what-youll-build)
2. [Prerequisites and downloads](#2-prerequisites-and-downloads)
3. [Generate the project](#3-generate-the-project)
4. [Project structure](#4-project-structure)
5. [Create the database](#5-create-the-database)
6. [Configure application.properties](#6-configure-applicationproperties)
7. [Write the entity](#7-write-the-entity)
8. [Write the repository](#8-write-the-repository)
9. [Write the request DTO](#9-write-the-request-dto)
10. [Write the controller](#10-write-the-controller)
11. [Handle validation errors](#11-handle-validation-errors)
12. [Run the application](#12-run-the-application)
13. [Test in Postman](#13-test-in-postman)
14. [Write an automated test (optional)](#14-write-an-automated-test-optional)
15. [Troubleshooting](#15-troubleshooting)
16. [Command cheat sheet](#16-command-cheat-sheet)
17. [Laravel to Spring Boot, side by side](#17-laravel-to-spring-boot-side-by-side)
18. [Where to go next](#18-where-to-go-next)

---

## 1. What you'll build

| Method | URI | Purpose | Success status |
|---|---|---|---|
| GET | `/api/products` | List products (paginated) | 200 |
| POST | `/api/products` | Create a product | 201 |
| GET | `/api/products/{id}` | Fetch one product | 200 |
| PUT | `/api/products/{id}` | Update a product | 200 |
| DELETE | `/api/products/{id}` | Delete a product | 204 |

**Six files total**, four of which you write by hand. No Docker, no Lombok, no XML config beyond the generated `pom.xml`.

---

## 2. Prerequisites and downloads

Three things. Note what's *not* on this list: Maven (the project ships a wrapper), Tomcat (embedded), and any IDE (optional).

### 2.1 JDK 21

Check what you have:

```powershell
java -version
```

Spring Boot 4.1 runs on Java 17 at minimum and supports up to Java 26. If you're already on 17, you can stay there. For a fresh install, take **21** — it's the most widely supported LTS right now.

Download the **Eclipse Temurin 21 (LTS) MSI** for Windows x64 from <https://adoptium.net/temurin/releases/>.

During install, expand the feature tree and enable:

- **Set JAVA_HOME variable**
- **Add to PATH**

Both are off by default in some builds, and skipping them causes the `mvnw` wrapper to fail later. Then open a **new** terminal:

```powershell
java -version
echo $env:JAVA_HOME
```

`JAVA_HOME` should print something like `C:\Program Files\Eclipse Adoptium\jdk-21.0.5.11-hotspot`. If it's blank, re-run the installer and tick the box.

### 2.2 XAMPP (MySQL only)

Download from <https://www.apachefriends.org/download.html> and install to `C:\xampp`.

Open the **XAMPP Control Panel** and click **Start** next to **MySQL**. That's the only component this project needs — Apache is only worth starting if you want the phpMyAdmin web UI at <http://localhost/phpmyadmin>. Spring Boot runs its own embedded Tomcat on port 8080, so nothing goes in `htdocs`.

Two things to know:

- XAMPP's "MySQL" is actually **MariaDB**. The MySQL JDBC driver talks to it fine; section 15 has a fallback if you hit an edge case.
- The default account is user `root` with an **empty password**.

> Unlike the Laravel guide, the PHP version inside XAMPP is irrelevant here. You're only using its database server.

### 2.3 Postman

Download the Windows 64-bit build from <https://www.postman.com/downloads/>.

You can skip the sign-in prompt with "Continue without an account" at the bottom of the window, though an account syncs your collections between machines.

### 2.4 An editor (optional but recommended)

Everything below works from PowerShell with Notepad. If you'd rather have autocomplete:

- **IntelliJ IDEA Community Edition** — <https://www.jetbrains.com/idea/download/> (free, best Spring support)
- **VS Code** + the *Extension Pack for Java* — lighter

### 2.5 Verification

```powershell
java -version      # 17 or higher
echo $env:JAVA_HOME  # must not be empty
```

Plus MySQL showing green in the XAMPP Control Panel.

---

## 3. Generate the project

Go to **<https://start.spring.io>** and set:

| Field | Value |
|---|---|
| Project | **Maven** |
| Language | **Java** |
| Spring Boot | the default (latest 4.1.x release — avoid SNAPSHOT/M builds) |
| Group | `com.example` |
| Artifact | `product-api` |
| Name | `product-api` |
| Package name | `com.example.productapi` |
| Packaging | **Jar** |
| Java | **21** (must match or be below your installed JDK) |

Click **ADD DEPENDENCIES** and add exactly four:

- **Spring Web** — REST controllers and embedded Tomcat
- **Spring Data JPA** — repositories and Hibernate
- **MySQL Driver** — JDBC connectivity
- **Validation** — the `@NotBlank` / `@Min` annotations

Click **GENERATE**. A `product-api.zip` downloads.

Extract it somewhere without spaces or OneDrive sync in the path — `C:\dev\product-api` is a good choice. OneDrive folders cause intermittent file-lock errors during builds.

```powershell
cd C:\dev\product-api
dir
```

You should see `mvnw`, `mvnw.cmd`, `pom.xml`, and `src`.

> **Why no Maven install:** `mvnw.cmd` is the Maven Wrapper. It downloads the correct Maven version on first run, so the build works identically on any machine. Always call it as `.\mvnw.cmd` in PowerShell — without the `.\`, PowerShell refuses to run executables from the current directory.

---

## 4. Project structure

You'll only touch two places:

```
product-api/
├── mvnw.cmd                      ← Maven wrapper (don't edit)
├── pom.xml                       ← dependencies (generated, no edits needed)
└── src/
    ├── main/
    │   ├── java/com/example/productapi/
    │   │   ├── ProductApiApplication.java   ← generated entry point
    │   │   ├── Product.java                 ← you write (§7)
    │   │   ├── ProductRepository.java       ← you write (§8)
    │   │   ├── ProductRequest.java          ← you write (§9)
    │   │   ├── ProductController.java       ← you write (§10)
    │   │   └── ApiExceptionHandler.java     ← you write (§11)
    │   └── resources/
    │       └── application.properties       ← you edit (§6)
    └── test/java/com/example/productapi/
        └── ProductApiApplicationTests.java
```

All new classes go in the **same package as `ProductApiApplication`**. Spring scans that package and its subpackages only — a class placed outside it is silently ignored, which produces a baffling 404.

Flat packaging like this is unusual for production code (you'd normally split into `controller`, `service`, `repository`, `model`), but it keeps the file count honest for a first build. Section 18 covers splitting it up.

---

## 5. Create the database

Hibernate will create the *table*, but not the *database*. Make it first, with MySQL running in XAMPP.

**Command line:**

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

No `-p` flag — the default XAMPP root account has no password.

**Or in phpMyAdmin:** start Apache too, open <http://localhost/phpmyadmin>, click **New**, enter `product_api`, set collation `utf8mb4_unicode_ci`, **Create**.

Confirm:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SHOW DATABASES;"
```

---

## 6. Configure application.properties

Open `src/main/resources/application.properties` and replace its contents:

```properties
spring.application.name=product-api
server.port=8080

# --- Database (XAMPP MySQL/MariaDB) ---
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/product_api?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=

# --- Hibernate ---
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false

# --- Cleaner error responses (RFC 9457) ---
spring.mvc.problemdetails.enabled=true
```

Line by line, the ones that matter:

**`127.0.0.1` not `localhost`** — on Windows, `localhost` frequently resolves to the IPv6 address `::1`, which MariaDB isn't listening on. The resulting error looks like the server is down when it isn't.

**`spring.datasource.password=`** — empty, matching the XAMPP default. Leave nothing after the `=`.

**`ddl-auto=update`** tells Hibernate to create and alter tables to match your entities at startup. That's what keeps this setup minimal: no migration tool, no SQL to write. It's a development convenience only — it never drops columns, happily accumulates cruft, and should be `validate` or `none` in production with Flyway or Liquibase handling schema changes.

**`open-in-view=false`** switches off a default that keeps a database session open for the whole request. It's on by default for historical reasons and hides lazy-loading bugs. Turning it off now saves confusion later.

**`problemdetails.enabled=true`** makes Spring return structured JSON error bodies instead of its default HTML "Whitelabel Error Page".

---

## 7. Write the entity

Create `src/main/java/com/example/productapi/Product.java`:

```java
package com.example.productapi;

import jakarta.persistence.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true, length = 64)
    private String sku;

    @Column(length = 2000)
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock = 0;

    @Column(name = "is_active", nullable = false)
    private Boolean active = true;

    @CreationTimestamp
    @Column(updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;

    // --- getters and setters ---

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getSku() { return sku; }
    public void setSku(String sku) { this.sku = sku; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }

    public Integer getStock() { return stock; }
    public void setStock(Integer stock) { this.stock = stock; }

    public Boolean getActive() { return active; }
    public void setActive(Boolean active) { this.active = active; }

    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}
```

Points worth understanding:

- **`BigDecimal` for money, never `double`.** Binary floating point can't represent `0.10` exactly; totals drift by cents. `precision = 10, scale = 2` maps to `DECIMAL(10,2)` in MySQL.
- **`@GeneratedValue(strategy = IDENTITY)`** uses MySQL's `AUTO_INCREMENT`. The default `AUTO` strategy on MySQL creates a separate sequence table — technically fine, surprising in practice.
- **`@CreationTimestamp` / `@UpdateTimestamp`** are Hibernate annotations that maintain those columns for you, equivalent to Laravel's `timestamps()`.
- **Getters are what Jackson serialises.** A field without a getter won't appear in your JSON. There's deliberately no setter for the timestamps, so a client can't forge them.
- **Lombok** would collapse all those accessors into `@Getter @Setter`. I've left it out because it needs an IDE plugin to avoid phantom compile errors, and "minimal setup" was the goal. Add it later if you want.

---

## 8. Write the repository

Create `ProductRepository.java`:

```java
package com.example.productapi;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {

    boolean existsBySku(String sku);

    boolean existsBySkuAndIdNot(String sku, Long id);

    Page<Product> findByNameContainingIgnoreCaseOrSkuContainingIgnoreCase(
            String name, String sku, Pageable pageable);
}
```

That's the whole data layer. You write **no implementation** — Spring Data generates one at startup by parsing the method names. `findByNameContainingIgnoreCase` becomes `WHERE LOWER(name) LIKE LOWER('%?%')`.

`JpaRepository` already supplies `findAll(Pageable)`, `findById`, `save`, `deleteById`, `existsById`, and about twenty more.

A typo in a derived method name fails **at startup**, not at runtime — you'll get a clear "No property 'nmae' found for type 'Product'" message rather than a surprise in production.

---

## 9. Write the request DTO

Never bind request JSON straight onto your entity. A client could otherwise set `id`, `createdAt`, or any future column you add. Create `ProductRequest.java`:

```java
package com.example.productapi;

import jakarta.validation.constraints.*;

import java.math.BigDecimal;

public record ProductRequest(

        @NotBlank(message = "Name is required")
        @Size(max = 255)
        String name,

        @NotBlank(message = "SKU is required")
        @Size(max = 64)
        String sku,

        @Size(max = 2000)
        String description,

        @NotNull(message = "Price is required")
        @DecimalMin(value = "0.0", message = "Price cannot be negative")
        BigDecimal price,

        @Min(value = 0, message = "Stock cannot be negative")
        Integer stock,

        Boolean active
) {}
```

A `record` gives you an immutable class with a constructor, accessors, `equals`, and `toString` in one line. Jackson deserialises JSON into records natively.

This is the direct equivalent of Laravel's `StoreProductRequest` — the same job, enforced by annotations instead of a `rules()` array.

---

## 10. Write the controller

Create `ProductController.java`:

```java
package com.example.productapi;

import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository repository;

    public ProductController(ProductRepository repository) {
        this.repository = repository;
    }

    // GET /api/products
    @GetMapping
    public Page<Product> index(
            @RequestParam(required = false) String search,
            @PageableDefault(size = 15, sort = "id", direction = Sort.Direction.DESC)
            Pageable pageable) {

        if (search == null || search.isBlank()) {
            return repository.findAll(pageable);
        }
        return repository.findByNameContainingIgnoreCaseOrSkuContainingIgnoreCase(
                search, search, pageable);
    }

    // POST /api/products
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product store(@Valid @RequestBody ProductRequest request) {
        if (repository.existsBySku(request.sku())) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "SKU already in use");
        }

        Product product = new Product();
        apply(request, product);
        return repository.save(product);
    }

    // GET /api/products/{id}
    @GetMapping("/{id}")
    public Product show(@PathVariable Long id) {
        return repository.findById(id).orElseThrow(this::notFound);
    }

    // PUT /api/products/{id}
    @PutMapping("/{id}")
    public Product update(@PathVariable Long id, @Valid @RequestBody ProductRequest request) {
        Product product = repository.findById(id).orElseThrow(this::notFound);

        if (repository.existsBySkuAndIdNot(request.sku(), id)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "SKU already in use");
        }

        apply(request, product);
        return repository.save(product);
    }

    // DELETE /api/products/{id}
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void destroy(@PathVariable Long id) {
        if (!repository.existsById(id)) {
            throw notFound();
        }
        repository.deleteById(id);
    }

    // --- helpers ---

    private void apply(ProductRequest request, Product product) {
        product.setName(request.name());
        product.setSku(request.sku());
        product.setDescription(request.description());
        product.setPrice(request.price());
        product.setStock(request.stock() == null ? 0 : request.stock());
        product.setActive(request.active() == null || request.active());
    }

    private ResponseStatusException notFound() {
        return new ResponseStatusException(HttpStatus.NOT_FOUND, "Product not found");
    }
}
```

### What's doing the work

**`@Valid`** triggers the annotations from section 9. If validation fails, the method body never runs — Spring throws `MethodArgumentNotValidException`, which section 11 turns into a clean 400.

**Constructor injection.** The repository arrives through the constructor; Spring supplies it automatically. No `@Autowired` annotation is needed on a single-constructor class, and constructor injection (rather than field injection) is what makes the class testable with a plain `new`.

**`Pageable` as a parameter** gives you `?page=0&size=10&sort=price,asc` for free. `@PageableDefault` sets the behaviour when the client sends nothing. **Pages are zero-indexed** — `page=0` is the first page. This trips up everyone coming from Laravel, where pages start at 1.

**`ResponseStatusException`** produces a proper HTTP status without needing custom exception classes.

**Returning the entity directly** keeps the file count down. It's acceptable here because `Product` has no sensitive fields and no lazy relationships. The moment you add either, introduce a response record — see section 18.

---

## 11. Handle validation errors

Without this, a validation failure returns a `ProblemDetail` that reports *that* something failed but not *which field*. Create `ApiExceptionHandler.java`:

```java
package com.example.productapi;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.LinkedHashMap;
import java.util.Map;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleValidation(MethodArgumentNotValidException ex) {

        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(error -> errors.putIfAbsent(error.getField(), error.getDefaultMessage()));

        return Map.of(
                "message", "Validation failed",
                "errors", errors
        );
    }
}
```

`@RestControllerAdvice` applies to every controller in the application. `putIfAbsent` keeps only the first message per field, so a field failing two rules doesn't produce a duplicate key.

---

## 12. Run the application

MySQL green in XAMPP, then:

```powershell
cd C:\dev\product-api
.\mvnw.cmd spring-boot:run
```

The **first run takes a few minutes** — Maven downloads the wrapper and every dependency into `C:\Users\<you>\.m2`. Later runs start in seconds.

Watch for these lines:

```
Hibernate: create table products (...)
Tomcat started on port 8080 (http)
Started ProductApiApplication in 3.412 seconds
```

That `create table` is `ddl-auto=update` doing its job. Check phpMyAdmin — `products` now exists inside `product_api`.

Stop the server with **Ctrl+C**.

> Running from IntelliJ instead? Open the folder, wait for Maven to import, then run `ProductApiApplication` — the green arrow beside `main()`.

---

## 13. Test in Postman

### 13.1 Collection and environment

**Collection:** left sidebar → **Collections** → **+** → name it `Spring Product API`.

**Environment:** left sidebar → **Environments** → **+** → name it `Local Spring`, then add:

| Variable | Initial value | Current value |
|---|---|---|
| `base_url` | `http://localhost:8080/api` | `http://localhost:8080/api` |

Save (Ctrl+S) and **select `Local Spring` in the top-right environment dropdown**. If it still says "No Environment", `{{base_url}}` won't resolve.

Note the port: **8080**, not 8000. If you're also running the Laravel version, both can run at once — different ports, different databases.

### 13.2 CREATE — POST /api/products

1. Right-click the collection → **Add request**, name it `Create product`
2. Method **POST**, URL `{{base_url}}/products`
3. **Body** tab → **raw** → change the dropdown on the right from *Text* to **JSON**
4. Paste:

```json
{
  "name": "Mechanical Keyboard",
  "sku": "KB-8700",
  "description": "Hot-swappable 75% layout",
  "price": 4250.00,
  "stock": 12
}
```

5. **Send**, then **Save**

Selecting **JSON** in step 3 is what sets `Content-Type: application/json`. Leave it on *Text* and Spring rejects the request with **415 Unsupported Media Type** — the single most common Postman mistake against a Spring API.

Expected: **201 Created**

```json
{
  "id": 1,
  "name": "Mechanical Keyboard",
  "sku": "KB-8700",
  "description": "Hot-swappable 75% layout",
  "price": 4250.00,
  "stock": 12,
  "active": true,
  "createdAt": "2026-09-12T08:31:04.221Z",
  "updatedAt": "2026-09-12T08:31:04.221Z"
}
```

Note there's **no `data` wrapper** — Laravel's API Resources add one, Spring returns the object bare. Neither is more correct; just be consistent.

#### Capture the ID automatically

In this request's **Scripts** tab (called **Tests** in older Postman versions):

```javascript
pm.test("Status is 201", function () {
    pm.response.to.have.status(201);
});

pm.collectionVariables.set("product_id", pm.response.json().id);
```

Now `{{product_id}}` works in the later requests.

### 13.3 Validation failure

Duplicate the create request, rename it `Create product (invalid)`, change the body to:

```json
{
  "name": "",
  "price": -5
}
```

Expected: **400 Bad Request**

```json
{
  "message": "Validation failed",
  "errors": {
    "name": "Name is required",
    "sku": "SKU is required",
    "price": "Price cannot be negative"
  }
}
```

Send the *valid* body a second time instead and you'll get **409 Conflict** with "SKU already in use", from the `existsBySku` check.

### 13.4 READ ALL — GET /api/products

Method **GET**, URL `{{base_url}}/products`. Expected **200 OK**:

```json
{
  "content": [
    { "id": 1, "name": "Mechanical Keyboard", "...": "..." }
  ],
  "pageable": { "pageNumber": 0, "pageSize": 15 },
  "totalElements": 1,
  "totalPages": 1,
  "first": true,
  "last": true,
  "numberOfElements": 1,
  "empty": false
}
```

Rows live under **`content`**, and the metadata is flattened rather than nested in a `meta` object the way Laravel does it.

Use the **Params** tab to exercise the query options:

| Key | Value | Effect |
|---|---|---|
| `search` | `keyboard` | Matches name or SKU, case-insensitive |
| `page` | `0` | **Zero-indexed** — first page is 0 |
| `size` | `5` | Page size |
| `sort` | `price,asc` | Any field name, `asc` or `desc` |

Untick a row to disable it without deleting it.

### 13.5 READ ONE — GET /api/products/{id}

URL `{{base_url}}/products/{{product_id}}` → **200 OK**.

Change it to `/products/9999` → **404 Not Found**:

```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "Product not found",
  "instance": "/api/products/9999"
}
```

That shape is RFC 9457 ProblemDetail, from the property you set in section 6.

### 13.6 UPDATE — PUT /api/products/{id}

Method **PUT**, URL `{{base_url}}/products/{{product_id}}`, Body → raw → JSON:

```json
{
  "name": "Mechanical Keyboard MK2",
  "sku": "KB-8700",
  "description": "Hot-swappable 75% layout, gasket mount",
  "price": 3999.00,
  "stock": 8,
  "active": true
}
```

**PUT replaces the whole resource, so send every field.** Omit `name` and validation rejects the request — that's PUT behaving correctly, not a bug. Section 18 covers adding PATCH for partial updates.

Expected **200 OK** with the updated object and a changed `updatedAt`.

### 13.7 DELETE — DELETE /api/products/{id}

Method **DELETE**, URL `{{base_url}}/products/{{product_id}}` → **204 No Content**, empty body. Send it again → **404**.

### 13.8 Run the collection

Right-click the collection → **Run collection**. Requests fire in sidebar order, so create must come before show/update/delete. With a `pm.test(...)` block in each, this becomes a regression suite you re-run after every change.

### 13.9 curl equivalents (optional)

```powershell
# CREATE
$body = '{"name":"Wireless Mouse","sku":"MS-2200","price":1250.00,"stock":30}'
curl.exe -X POST http://localhost:8080/api/products -H "Content-Type: application/json" -d $body

# READ ALL (quote the URL — PowerShell treats a bare & as an operator)
curl.exe "http://localhost:8080/api/products?size=5&sort=price,desc"

# READ ONE
curl.exe http://localhost:8080/api/products/1

# UPDATE
$update = '{"name":"Wireless Mouse","sku":"MS-2200","price":999.00,"stock":25,"active":true}'
curl.exe -X PUT http://localhost:8080/api/products/1 -H "Content-Type: application/json" -d $update

# DELETE
curl.exe -X DELETE http://localhost:8080/api/products/1 -i
```

Use `curl.exe`, not `curl` — plain `curl` is a PowerShell alias for `Invoke-WebRequest` and takes different flags. Put JSON in a variable first; PowerShell mangles quotes in inline `-d` arguments.

---

## 14. Write an automated test (optional)

`MockMvc` exercises the full request pipeline — routing, validation, serialisation — without starting a real server.

Create `src/test/java/com/example/productapi/ProductApiTest.java`:

```java
package com.example.productapi;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class ProductApiTest {

    @Autowired MockMvc mockMvc;
    @Autowired ProductRepository repository;
    @Autowired ObjectMapper objectMapper;

    private Product saved;

    @BeforeEach
    void setUp() {
        Product product = new Product();
        product.setName("Test Keyboard");
        product.setSku("TEST-0001");
        product.setPrice(new BigDecimal("1500.00"));
        product.setStock(4);
        saved = repository.save(product);
    }

    @Test
    void listsProducts() throws Exception {
        mockMvc.perform(get("/api/products"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.content").isArray())
               .andExpect(jsonPath("$.totalElements").value(1));
    }

    @Test
    void createsProduct() throws Exception {
        ProductRequest request = new ProductRequest(
                "New Mouse", "MS-9999", null, new BigDecimal("750.00"), 10, true);

        mockMvc.perform(post("/api/products")
                       .contentType(MediaType.APPLICATION_JSON)
                       .content(objectMapper.writeValueAsString(request)))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.sku").value("MS-9999"));
    }

    @Test
    void rejectsInvalidInput() throws Exception {
        mockMvc.perform(post("/api/products")
                       .contentType(MediaType.APPLICATION_JSON)
                       .content("{\"name\":\"\",\"price\":-5}"))
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.errors.name").exists())
               .andExpect(jsonPath("$.errors.sku").exists());
    }

    @Test
    void returns404ForMissingProduct() throws Exception {
        mockMvc.perform(get("/api/products/999999"))
               .andExpect(status().isNotFound());
    }

    @Test
    void deletesProduct() throws Exception {
        mockMvc.perform(delete("/api/products/" + saved.getId()))
               .andExpect(status().isNoContent());
    }
}
```

Run it:

```powershell
.\mvnw.cmd test
```

**`@Transactional` on the test class rolls back every test's database changes afterward**, so these run against your development database without polluting it. That's a meaningful difference from Laravel's `RefreshDatabase`, which drops and recreates every table and therefore needs a separate test schema.

If you'd still rather isolate completely, create `src/test/resources/application-test.properties` pointing at a `product_api_test` database and add `@ActiveProfiles("test")` to the class.

---

## 15. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `mvnw.cmd : The term 'mvnw.cmd' is not recognized` | PowerShell won't run from the current dir implicitly | Use `.\mvnw.cmd`, and confirm you're in the project root |
| `JAVA_HOME not found in your environment` | JDK installed without the env-var option | Re-run the Temurin installer and tick **Set JAVA_HOME**, then open a new terminal |
| `class file has wrong version 69.0, should be 61.0` | Project's Java version is higher than your JDK | Match `<java.version>` in `pom.xml` to your installed JDK, or install the newer JDK |
| `Web server failed to start. Port 8080 was already in use` | Another app has the port | Add `server.port=8081` to `application.properties`, or find the process: `netstat -ano \| findstr :8080` |
| `Communications link failure` / `Connection refused` | MySQL not started in XAMPP | Start MySQL in the Control Panel |
| Connection refused specifically on `localhost` | `localhost` resolving to IPv6 `::1` | Use `127.0.0.1` in the JDBC URL |
| `Unknown database 'product_api'` | Schema never created | Run the `CREATE DATABASE` command in §5 |
| `Access denied for user 'root'@'localhost'` | Wrong credentials | XAMPP default is `root` with an **empty** password |
| MySQL won't start; log mentions port 3306 | Another MySQL service holds the port | Stop it in `services.msc`, or change XAMPP's port and update the JDBC URL |
| `Table 'products' doesn't exist` | `ddl-auto` set to `none`/`validate` | Set `spring.jpa.hibernate.ddl-auto=update` |
| 404 on every endpoint, app starts fine | Classes outside the `ProductApiApplication` package | Move them into `com.example.productapi` |
| **415 Unsupported Media Type** | Postman body left on *Text* | Body → raw → dropdown → **JSON** |
| **400** with no field detail | `ApiExceptionHandler` missing or in the wrong package | See §11 |
| `Parameter 0 of constructor... required a bean of type ProductRepository` | Repository outside the scanned package, or missing `extends JpaRepository` | Check package and interface declaration |
| `No property 'xyz' found for type 'Product'` at startup | Typo in a derived repository method name | Fix the method name to match a field |
| Timestamps returned as huge numbers | Jackson serialising `Instant` as epoch | Add `spring.jackson.serialization.write-dates-as-timestamps=false` |
| Odd driver errors against MariaDB | MySQL Connector/J edge case | Swap in `org.mariadb.jdbc:mariadb-java-client` and change the URL to `jdbc:mariadb://...` |

When a build behaves strangely:

```powershell
.\mvnw.cmd clean
.\mvnw.cmd spring-boot:run
```

---

## 16. Command cheat sheet

```powershell
# Database (start MySQL in XAMPP first)
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
C:\xampp\mysql\bin\mysql.exe -u root product_api -e "SELECT * FROM products;"

# Run
.\mvnw.cmd spring-boot:run
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.arguments=--server.port=8081"

# Test
.\mvnw.cmd test
.\mvnw.cmd test "-Dtest=ProductApiTest"

# Build
.\mvnw.cmd clean package          # produces target\product-api-0.0.1-SNAPSHOT.jar
java -jar target\product-api-0.0.1-SNAPSHOT.jar

# Inspect
.\mvnw.cmd dependency:tree
java -version
```

---

## 17. Laravel to Spring Boot, side by side

| Concept | Laravel | Spring Boot |
|---|---|---|
| Create project | `laravel new product-api` | start.spring.io → download zip |
| Run dev server | `php artisan serve` (:8000) | `.\mvnw.cmd spring-boot:run` (:8080) |
| Routes | `routes/api.php`, `Route::apiResource` | `@RestController` + `@RequestMapping` |
| Model | Eloquent `Product extends Model` | JPA `@Entity class Product` |
| Schema | Migration files, `php artisan migrate` | `ddl-auto=update`, or Flyway |
| Queries | `Product::where(...)->get()` | `repository.findBy...()` |
| Validation | `StoreProductRequest::rules()` | `@Valid` + record annotations |
| Response shaping | `ProductResource` | Return entity, or a response record |
| Mass-assignment guard | `$fillable` | A separate request DTO |
| 404 on missing record | Route model binding, automatic | `orElseThrow(...)`, explicit |
| Pagination | `paginate()`, pages start at **1** | `Pageable`, pages start at **0** |
| JSON envelope | `{"data": {...}}` | bare object, no wrapper |
| Config | `.env` | `application.properties` |
| Test DB isolation | `RefreshDatabase` (drops tables) | `@Transactional` (rolls back) |

The biggest practical differences: Spring makes you write the 404 check that Laravel's route model binding does implicitly, and Spring's pagination is zero-indexed.

---

## 18. Where to go next

1. **Add a service layer.** Move logic out of the controller into a `@Service` class once methods grow past a few lines. Controllers should handle HTTP; services should hold business rules.
2. **Split the packages** into `controller`, `service`, `repository`, `model`, `dto` — standard once you pass a handful of classes.
3. **Add a response DTO.** Return a `ProductResponse` record instead of the entity, so a database column change can't silently alter your API contract.
4. **Add PATCH** for partial updates: a second DTO with all-nullable fields, applying only non-null values.
5. **Replace `ddl-auto` with Flyway.** Add `flyway-core` plus `flyway-mysql`, put versioned SQL in `src/main/resources/db/migration`, and set `ddl-auto=validate`. This is what you'd do before any real deployment.
6. **Add Spring Security** with JWT once the API needs authentication.
7. **Add springdoc-openapi** (`springdoc-openapi-starter-webmvc-ui`) for interactive docs at `/swagger-ui.html` generated from your annotations.
8. **Add Testcontainers** to run tests against a real MySQL in CI without depending on a local XAMPP install.

### Reference links

- Spring Boot reference: <https://docs.spring.io/spring-boot/index.html>
- Spring Data JPA query methods: <https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html>
- Jakarta Bean Validation constraints: <https://jakarta.ee/specifications/bean-validation/>
- Building a REST service (official guide): <https://spring.io/guides/gs/rest-service>
