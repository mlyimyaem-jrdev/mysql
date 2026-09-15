# Building a Simple CRUD API in Spring Boot

The same product API as the Laravel guide, built with Spring Boot — minimal setup, XAMPP for MySQL, tested in Postman. Part 2 adds a Thymeleaf web UI with image upload; Part 3 adds Swagger UI.

**Target version:** Spring Boot 4.1.x (Spring Framework 7)
**Java:** 17 minimum; 21 or 25 (both LTS) recommended
**Platform:** Windows 10 / 11, PowerShell
**Database:** MySQL/MariaDB from XAMPP
**No Docker, no Gradle install, no global Maven install**

---

## Table of Contents

**Part 1 — the JSON API**

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

**Part 2 — the Thymeleaf form with image upload**

14. [Add Thymeleaf and upload configuration](#14-add-thymeleaf-and-upload-configuration)
15. [Write the form-backed bean](#15-write-the-form-backed-bean)
16. [Write the web controller](#16-write-the-web-controller)
17. [Write the templates](#17-write-the-templates)
18. [Try the form in the browser](#18-try-the-form-in-the-browser)

**Part 3 — Swagger UI**

19. [Add Swagger UI (OpenAPI documentation)](#19-add-swagger-ui-openapi-documentation)

**Reference**

20. [Write an automated test (optional)](#20-write-an-automated-test-optional)
21. [Troubleshooting](#21-troubleshooting)
22. [Command cheat sheet](#22-command-cheat-sheet)
23. [Laravel to Spring Boot, side by side](#23-laravel-to-spring-boot-side-by-side)
24. [Where to go next](#24-where-to-go-next)

---

## 1. What you'll build

| Method | URI | Purpose | Success status |
|---|---|---|---|
| GET | `/api/products` | List products (paginated) | 200 |
| POST | `/api/products` | Create a product | 201 |
| GET | `/api/products/{id}` | Fetch one product | 200 |
| PUT | `/api/products/{id}` | Update a product | 200 |
| DELETE | `/api/products/{id}` | Delete a product | 204 |

Then Part 2 adds a server-rendered UI over the same data:

| Method | URI | Page |
|---|---|---|
| GET | `/products` | List with thumbnails, search, pagination |
| GET | `/products/new` | Blank create form |
| GET | `/products/{id}/edit` | Pre-filled edit form |
| POST | `/products/save` | Handles create and update, including the image upload |
| POST | `/products/{id}/delete` | Delete |

Part 3 then generates interactive Swagger documentation at `/swagger-ui.html` from the code you already wrote.

**Eleven files total** by the end, nine of which you write by hand — four for the API, four for the web layer. No Docker, no Lombok, no XML config beyond the generated `pom.xml`.

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

- XAMPP's "MySQL" is actually **MariaDB**. The MySQL JDBC driver talks to it fine; section 21 has a fallback if you hit an edge case.
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

Click **ADD DEPENDENCIES** and add exactly five:

- **Spring Web** — REST controllers and embedded Tomcat
- **Spring Data JPA** — repositories and Hibernate
- **MySQL Driver** — JDBC connectivity
- **Validation** — the `@NotBlank` / `@Min` annotations
- **Thymeleaf** — the template engine for Part 2

Only the first four are needed for Part 1. Adding Thymeleaf now saves editing `pom.xml` later; §14.1 covers adding it after the fact if you'd rather.

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
├── pom.xml                       ← dependencies (§14.1 if adding Thymeleaf later)
├── uploads/                      ← created at runtime by ImageStorage (§14.5)
└── src/
    ├── main/
    │   ├── java/com/example/productapi/
    │   │   ├── ProductApiApplication.java   ← generated entry point
    │   │   │
    │   │   ├── Product.java                 ← you write (§7, §14.3)
    │   │   ├── ProductRepository.java       ← you write (§8)
    │   │   ├── ProductRequest.java          ← you write (§9)
    │   │   ├── ProductController.java       ← you write (§10)   — JSON API
    │   │   ├── ApiExceptionHandler.java     ← you write (§11)
    │   │   │
    │   │   ├── WebConfig.java               ← you write (§14.4) — Part 2
    │   │   ├── ImageStorage.java            ← you write (§14.5)
    │   │   ├── ProductForm.java             ← you write (§15)
    │   │   └── ProductWebController.java    ← you write (§16)   — HTML pages
    │   └── resources/
    │       ├── application.properties       ← you edit (§6, §14.2)
    │       └── templates/products/
    │           ├── list.html                ← you write (§17.1)
    │           └── form.html                ← you write (§17.2)
    └── test/java/com/example/productapi/
        └── ProductApiApplicationTests.java
```

All new classes go in the **same package as `ProductApiApplication`**. Spring scans that package and its subpackages only — a class placed outside it is silently ignored, which produces a baffling 404.

Flat packaging like this is unusual for production code (you'd normally split into `controller`, `service`, `repository`, `model`), but it keeps the file count honest for a first build. Section 24 covers splitting it up.

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
- **Part 2 adds one more field**, `imageFilename`, in §14.3. Skip it for now if you're only building the API.
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

**Returning the entity directly** keeps the file count down. It's acceptable here because `Product` has no sensitive fields and no lazy relationships. The moment you add either, introduce a response record — see section 24.

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

**PUT replaces the whole resource, so send every field.** Omit `name` and validation rejects the request — that's PUT behaving correctly, not a bug. Section 24 covers adding PATCH for partial updates.

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

# Part 2 — A Thymeleaf form with image upload

Everything so far is a JSON API. Part 2 adds a server-rendered web UI on top of the same repository: a product list, a create/edit form, and image upload to local disk.

The two live side by side. `/api/products` keeps returning JSON for Postman; `/products` serves HTML. Rows created through either one show up in the other.

---

## 14. Add Thymeleaf and upload configuration

### 14.1 Add the dependency

If you ticked **Thymeleaf** at start.spring.io, skip this. Otherwise open `pom.xml` and add it inside `<dependencies>`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

No `<version>` — the Spring Boot parent POM manages it. Stop the app (Ctrl+C) so Maven picks up the change on the next run.

### 14.2 Add upload settings

Append to `src/main/resources/application.properties`:

```properties
# --- File uploads ---
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=5MB
app.upload-dir=uploads

# --- Templates: reload without restarting (development only) ---
spring.thymeleaf.cache=false
```

`max-file-size` caps a single file; `max-request-size` caps the whole submission. Both default to 1MB, which a phone photo blows past instantly — that's the source of the mystifying `MaxUploadSizeExceededException` most people hit first.

`app.upload-dir=uploads` is a custom property of your own, read by the two classes below. The folder is created relative to the **working directory**, so running from the project root puts it at `C:\dev\product-api\uploads`.

Add it to `.gitignore` — uploaded files don't belong in version control:

```
uploads/
```

### 14.3 Add the image column to the entity

In `Product.java`, add the field alongside the others:

```java
    @Column(name = "image_filename")
    private String imageFilename;
```

and the accessors, next to the other getters and setters:

```java
    public String getImageFilename() { return imageFilename; }
    public void setImageFilename(String imageFilename) { this.imageFilename = imageFilename; }
```

Because `ddl-auto=update` is still set, Hibernate adds the `image_filename` column on the next startup. No migration to write.

**The database stores only the filename**, not the file and not a full path. Storing bytes in a `BLOB` bloats the table and slows every query that touches it; storing an absolute path breaks the moment the app moves to another machine.

### 14.4 Serve the uploaded files

Files written to `uploads/` sit outside `src/main/resources/static`, so Spring won't serve them by default. Create `WebConfig.java`:

```java
package com.example.productapi;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.nio.file.Path;
import java.nio.file.Paths;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final String uploadDir;

    public WebConfig(@Value("${app.upload-dir}") String uploadDir) {
        this.uploadDir = uploadDir;
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        Path path = Paths.get(uploadDir).toAbsolutePath().normalize();

        registry.addResourceHandler("/uploads/**")
                .addResourceLocations("file:" + path + "/");
    }
}
```

This maps the URL `/uploads/abc123.jpg` to the file `uploads\abc123.jpg` on disk.

**That trailing slash after `path` is mandatory.** Without it Spring treats the location as a file rather than a directory, and every image silently 404s — a genuinely annoying half-hour if you don't know to look for it.

### 14.5 Write the storage component

Create `ImageStorage.java`:

```java
package com.example.productapi;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.util.List;
import java.util.Locale;
import java.util.UUID;

@Component
public class ImageStorage {

    private static final List<String> ALLOWED_EXTENSIONS =
            List.of("jpg", "jpeg", "png", "webp", "gif");

    private final Path root;

    public ImageStorage(@Value("${app.upload-dir}") String uploadDir) {
        this.root = Paths.get(uploadDir).toAbsolutePath().normalize();
        try {
            Files.createDirectories(root);
        } catch (IOException e) {
            throw new UncheckedIOException("Could not create upload directory: " + root, e);
        }
    }

    /**
     * Saves the upload under a generated name and returns that name.
     */
    public String store(MultipartFile file) {
        String original = StringUtils.cleanPath(
                file.getOriginalFilename() == null ? "" : file.getOriginalFilename());

        String extension = StringUtils.getFilenameExtension(original);

        if (extension == null || !ALLOWED_EXTENSIONS.contains(extension.toLowerCase(Locale.ROOT))) {
            throw new IllegalArgumentException("Only JPG, PNG, WEBP or GIF images are allowed");
        }

        String filename = UUID.randomUUID() + "." + extension.toLowerCase(Locale.ROOT);
        Path target = root.resolve(filename).normalize();

        if (!target.getParent().equals(root)) {
            throw new IllegalArgumentException("Invalid file name");
        }

        try (InputStream in = file.getInputStream()) {
            Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to store " + original, e);
        }

        return filename;
    }

    /**
     * Removes a stored file. Safe to call with null or a name that no longer exists.
     */
    public void delete(String filename) {
        if (filename == null || filename.isBlank()) {
            return;
        }
        try {
            Files.deleteIfExists(root.resolve(filename).normalize());
        } catch (IOException e) {
            // A leftover file is not worth failing the request over.
        }
    }
}
```

Four deliberate safety decisions in that class, each guarding against a real attack or bug:

**Never reuse the client's filename.** `getOriginalFilename()` is attacker-controlled. A crafted value like `..\..\application.properties` would let an upload escape the folder and overwrite your config. Generating a `UUID` name sidesteps the whole class of problem and also stops two users' `photo.jpg` from overwriting each other.

**Allowlist the extension, don't blocklist.** Checking against a known-good list is the only version that stays correct as new dangerous types appear.

**Normalise and verify the target path.** `normalize()` resolves any `..` segments; comparing the parent against `root` rejects anything that escaped. Belt and braces, given the UUID name, but the cost is one line.

**Size is capped by the multipart config**, before a single byte reaches this class — which is why there's no size check here.

> A production version would go further: verify the actual file contents rather than trusting the extension (a `.jpg` can contain anything), strip EXIF metadata, and generate resized thumbnails. For a first build, the above is a reasonable floor.

---

## 15. Write the form-backed bean

The REST API binds to `ProductRequest`, a record. Thymeleaf can't use it: `th:field` needs a mutable JavaBean with getters *and* setters, and records are immutable. So the web form gets its own class.

Create `ProductForm.java`:

```java
package com.example.productapi;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import org.springframework.web.multipart.MultipartFile;

import java.math.BigDecimal;

public class ProductForm {

    private Long id;                 // null when creating

    @NotBlank(message = "Name is required")
    @Size(max = 255, message = "Name must be 255 characters or fewer")
    private String name;

    @NotBlank(message = "SKU is required")
    @Size(max = 64, message = "SKU must be 64 characters or fewer")
    private String sku;

    @Size(max = 2000, message = "Description must be 2000 characters or fewer")
    private String description;

    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.0", message = "Price cannot be negative")
    private BigDecimal price;

    @Min(value = 0, message = "Stock cannot be negative")
    private Integer stock = 0;

    private Boolean active = true;

    private MultipartFile image;     // the upload itself

    private String existingImage;    // filename already on the product, if any

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

    public MultipartFile getImage() { return image; }
    public void setImage(MultipartFile image) { this.image = image; }

    public String getExistingImage() { return existingImage; }
    public void setExistingImage(String existingImage) { this.existingImage = existingImage; }
}
```

`existingImage` is what makes editing behave sensibly: a file input can't be pre-filled with the current image, so the form carries the existing filename in a hidden field. Submit the form without choosing a new file and the product keeps the image it had.

---

## 16. Write the web controller

This is a `@Controller`, not a `@RestController`. The difference: `@RestController` serialises return values to the response body, while `@Controller` treats a returned `String` as the **name of a template to render**.

Create `ProductWebController.java`:

```java
package com.example.productapi;

import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.server.ResponseStatusException;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/products")
public class ProductWebController {

    private final ProductRepository repository;
    private final ImageStorage imageStorage;

    public ProductWebController(ProductRepository repository, ImageStorage imageStorage) {
        this.repository = repository;
        this.imageStorage = imageStorage;
    }

    /**
     * GET /products — the list page
     */
    @GetMapping
    public String list(@RequestParam(required = false) String search,
                       @PageableDefault(size = 10, sort = "id", direction = Sort.Direction.DESC)
                       Pageable pageable,
                       Model model) {

        Page<Product> products = (search == null || search.isBlank())
                ? repository.findAll(pageable)
                : repository.findByNameContainingIgnoreCaseOrSkuContainingIgnoreCase(
                        search, search, pageable);

        model.addAttribute("products", products);
        model.addAttribute("search", search);

        return "products/list";
    }

    /**
     * GET /products/new — blank form
     */
    @GetMapping("/new")
    public String createForm(Model model) {
        model.addAttribute("productForm", new ProductForm());
        return "products/form";
    }

    /**
     * GET /products/{id}/edit — form pre-filled from the database
     */
    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        Product product = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Product not found"));

        ProductForm form = new ProductForm();
        form.setId(product.getId());
        form.setName(product.getName());
        form.setSku(product.getSku());
        form.setDescription(product.getDescription());
        form.setPrice(product.getPrice());
        form.setStock(product.getStock());
        form.setActive(product.getActive());
        form.setExistingImage(product.getImageFilename());

        model.addAttribute("productForm", form);
        return "products/form";
    }

    /**
     * POST /products/save — handles both create and update
     */
    @PostMapping("/save")
    public String save(@Valid @ModelAttribute("productForm") ProductForm form,
                       BindingResult binding,
                       RedirectAttributes redirect) {

        // SKU uniqueness — annotations can't check the database
        if (form.getSku() != null && !form.getSku().isBlank()) {
            boolean duplicate = (form.getId() == null)
                    ? repository.existsBySku(form.getSku())
                    : repository.existsBySkuAndIdNot(form.getSku(), form.getId());

            if (duplicate) {
                binding.rejectValue("sku", "duplicate", "That SKU is already in use");
            }
        }

        if (binding.hasErrors()) {
            return "products/form";      // redisplay with messages
        }

        Product product = (form.getId() == null)
                ? new Product()
                : repository.findById(form.getId())
                            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Product not found"));

        product.setName(form.getName());
        product.setSku(form.getSku());
        product.setDescription(form.getDescription());
        product.setPrice(form.getPrice());
        product.setStock(form.getStock() == null ? 0 : form.getStock());
        product.setActive(form.getActive() != null && form.getActive());

        MultipartFile image = form.getImage();

        if (image != null && !image.isEmpty()) {
            String previous = product.getImageFilename();
            try {
                product.setImageFilename(imageStorage.store(image));
                imageStorage.delete(previous);          // clean up the replaced file
            } catch (IllegalArgumentException e) {
                binding.rejectValue("image", "invalid", e.getMessage());
                return "products/form";
            }
        }

        boolean creating = (form.getId() == null);
        repository.save(product);

        redirect.addFlashAttribute("message",
                creating ? "Product created." : "Product updated.");

        return "redirect:/products";
    }

    /**
     * POST /products/{id}/delete
     */
    @PostMapping("/{id}/delete")
    public String delete(@PathVariable Long id, RedirectAttributes redirect) {
        repository.findById(id).ifPresent(product -> {
            imageStorage.delete(product.getImageFilename());
            repository.delete(product);
        });

        redirect.addFlashAttribute("message", "Product deleted.");
        return "redirect:/products";
    }
}
```

### Things worth understanding here

**`BindingResult` must come immediately after the `@Valid` parameter.** Put anything between them and Spring throws the exception instead of populating the result object, so your form never redisplays with errors. This ordering rule catches almost everyone once.

**`return "products/form"` on error, not a redirect.** Returning the view name re-renders the page with the submitted values and the error messages still in memory. Redirecting would throw all of that away and hand the user an empty form.

**`redirect:/products` after a successful save** is the POST-Redirect-GET pattern. Without it, the browser's refresh button re-submits the form and creates a duplicate — and the user gets a "confirm form resubmission" dialog.

**`RedirectAttributes.addFlashAttribute`** survives exactly one redirect, then disappears. It's how the "Product created." message reaches the list page without ending up in the URL.

**Delete is a POST, not a DELETE.** HTML forms only support GET and POST. Your REST API uses a real `DELETE /api/products/{id}`; the web UI can't, so it posts to a distinct URL instead.

**The file is only touched after validation passes**, so a form with a bad price never writes anything to disk.

---

## 17. Write the templates

Both files go under `src/main/resources/templates/products/`. Create that folder — the path must match the strings returned by the controller exactly, or you get a `TemplateInputException`.

### 17.1 `templates/products/list.html`

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Products</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem auto; max-width: 60rem; color: #222; }
        table { border-collapse: collapse; width: 100%; margin-top: 1rem; }
        th, td { border-bottom: 1px solid #ddd; padding: .6rem; text-align: left; vertical-align: middle; }
        th { background: #f5f5f5; }
        img.thumb { width: 56px; height: 56px; object-fit: cover; border-radius: 4px; }
        .flash { background: #e6f4ea; border: 1px solid #b7dfc4; padding: .6rem 1rem; border-radius: 4px; }
        .muted { color: #999; }
        .toolbar { display: flex; gap: .5rem; align-items: center; margin-top: 1rem; }
        .pager a { padding: .3rem .6rem; }
        .pager a.current { font-weight: 700; text-decoration: none; }
        button { cursor: pointer; }
    </style>
</head>
<body>

<h1>Products</h1>

<p class="flash" th:if="${message}" th:text="${message}"></p>

<div class="toolbar">
    <form method="get" th:action="@{/products}">
        <input type="text" name="search" th:value="${search}" placeholder="Search name or SKU">
        <button type="submit">Search</button>
    </form>
    <a th:href="@{/products/new}">+ New product</a>
</div>

<table>
    <thead>
    <tr>
        <th>Image</th><th>Name</th><th>SKU</th><th>Price</th>
        <th>Stock</th><th>Active</th><th>Actions</th>
    </tr>
    </thead>
    <tbody>
    <tr th:each="product : ${products.content}">
        <td>
            <img class="thumb"
                 th:if="${product.imageFilename != null}"
                 th:src="@{'/uploads/' + ${product.imageFilename}}"
                 th:alt="${product.name}">
            <span class="muted" th:unless="${product.imageFilename != null}">none</span>
        </td>
        <td th:text="${product.name}">Name</td>
        <td th:text="${product.sku}">SKU</td>
        <td th:text="${#numbers.formatDecimal(product.price, 1, 2)}">0.00</td>
        <td th:text="${product.stock}">0</td>
        <td th:text="${product.active} ? 'Yes' : 'No'">Yes</td>
        <td>
            <a th:href="@{'/products/' + ${product.id} + '/edit'}">Edit</a>
            <form method="post"
                  th:action="@{'/products/' + ${product.id} + '/delete'}"
                  style="display:inline"
                  onsubmit="return confirm('Delete this product?');">
                <button type="submit">Delete</button>
            </form>
        </td>
    </tr>
    <tr th:if="${products.empty}">
        <td colspan="7" class="muted">No products yet.</td>
    </tr>
    </tbody>
</table>

<div class="pager" th:if="${products.totalPages > 1}">
    <span th:each="i : ${#numbers.sequence(0, products.totalPages - 1)}">
        <a th:href="@{/products(page=${i}, search=${search})}"
           th:text="${i + 1}"
           th:classappend="${i == products.number} ? 'current'">1</a>
    </span>
</div>

</body>
</html>
```

### 17.2 `templates/products/form.html`

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="${productForm.id} == null ? 'New product' : 'Edit product'">Product</title>
    <style>
        body { font-family: system-ui, sans-serif; margin: 2rem auto; max-width: 34rem; color: #222; }
        label { display: block; margin-top: 1rem; font-weight: 600; }
        input[type=text], input[type=number], textarea, input[type=file] {
            width: 100%; padding: .5rem; border: 1px solid #ccc; border-radius: 4px;
            box-sizing: border-box; font: inherit;
        }
        textarea { min-height: 5rem; }
        .error { color: #b3261e; font-size: .9rem; margin: .25rem 0 0; }
        .actions { margin-top: 1.5rem; display: flex; gap: .75rem; align-items: center; }
        .current-image { margin-top: .75rem; }
        .current-image img { max-width: 140px; border-radius: 4px; display: block; }
        .hint { color: #777; font-size: .85rem; margin-top: .25rem; }
        button { padding: .5rem 1rem; cursor: pointer; }
    </style>
</head>
<body>

<h1 th:text="${productForm.id} == null ? 'New product' : 'Edit product'">Product</h1>

<form method="post"
      th:action="@{/products/save}"
      th:object="${productForm}"
      enctype="multipart/form-data">

    <input type="hidden" th:field="*{id}">
    <input type="hidden" th:field="*{existingImage}">

    <label for="name">Name</label>
    <input type="text" id="name" th:field="*{name}">
    <p class="error" th:if="${#fields.hasErrors('name')}" th:errors="*{name}"></p>

    <label for="sku">SKU</label>
    <input type="text" id="sku" th:field="*{sku}">
    <p class="error" th:if="${#fields.hasErrors('sku')}" th:errors="*{sku}"></p>

    <label for="description">Description</label>
    <textarea id="description" th:field="*{description}"></textarea>
    <p class="error" th:if="${#fields.hasErrors('description')}" th:errors="*{description}"></p>

    <label for="price">Price</label>
    <input type="number" id="price" step="0.01" min="0" th:field="*{price}">
    <p class="error" th:if="${#fields.hasErrors('price')}" th:errors="*{price}"></p>

    <label for="stock">Stock</label>
    <input type="number" id="stock" min="0" th:field="*{stock}">
    <p class="error" th:if="${#fields.hasErrors('stock')}" th:errors="*{stock}"></p>

    <label>
        <input type="checkbox" th:field="*{active}"> Active
    </label>

    <label for="image">Image</label>
    <input type="file" id="image" name="image"
           accept="image/png, image/jpeg, image/webp, image/gif">
    <p class="hint">JPG, PNG, WEBP or GIF. 5 MB maximum.</p>
    <p class="error" th:if="${#fields.hasErrors('image')}" th:errors="*{image}"></p>

    <div class="current-image" th:if="*{existingImage} != null">
        <span class="hint">Current image — leave the field above empty to keep it:</span>
        <img th:src="@{'/uploads/' + *{existingImage}}" alt="Current product image">
    </div>

    <div class="actions">
        <button type="submit">Save</button>
        <a th:href="@{/products}">Cancel</a>
    </div>
</form>

</body>
</html>
```

### The four Thymeleaf pieces you actually need

**`enctype="multipart/form-data"`** on the form tag. Leave it off and the browser sends only the filename as text — `MultipartFile` arrives empty, no error is raised anywhere, and the upload just silently doesn't happen. This is the number one file-upload bug in every framework.

**`th:object` + `th:field`.** `th:object="${productForm}"` names the bean; `th:field="*{name}"` then generates the `id`, `name`, and `value` attributes from the `name` property in one go. The `*{...}` syntax means "a property of the current `th:object`".

**`th:errors` + `#fields.hasErrors`.** These read the `BindingResult` the controller handed back, so failed validation redisplays the form with each message under its own field.

**`th:if` vs `th:unless`** control whether an element renders at all — unlike CSS `display:none`, the markup is never sent.

Two smaller details: the checkbox works correctly with `th:field` because Thymeleaf emits a companion hidden input, so an unchecked box submits `false` rather than nothing at all. And `@{...}` is URL syntax — it prefixes your application's context path, which matters the day this deploys somewhere other than the root of a domain.

---

## 18. Try the form in the browser

Restart the application:

```powershell
.\mvnw.cmd spring-boot:run
```

Then open <http://localhost:8080/products>.

Walk through it:

1. **The list page.** Any rows you created in Postman are already here — same table, same repository.
2. **Click "+ New product"**, fill in the fields, choose an image, **Save**. You land back on the list with a green confirmation and a thumbnail in the first column.
3. **Check the disk.** A UUID-named file now sits in `C:\dev\product-api\uploads\`:
   ```powershell
   dir uploads
   ```
4. **Check the database.** `image_filename` holds just that name:
   ```powershell
   C:\xampp\mysql\bin\mysql.exe -u root product_api -e "SELECT id, name, image_filename FROM products;"
   ```
5. **Test validation.** Submit with an empty name and a negative price — the form comes back with messages under each field and your other input intact.
6. **Test the SKU check.** Reuse an existing SKU and you get "That SKU is already in use" under the SKU field.
7. **Test editing.** Click **Edit**. The form is pre-filled and shows the current image. Save without choosing a new file — the image is kept. Choose a new one and the old file is deleted from `uploads/`.
8. **Check the API still works.** `GET http://localhost:8080/api/products` in Postman now returns `imageFilename` on each product, since both layers share one entity.

> `spring.thymeleaf.cache=false` means template edits show up on browser refresh — no restart. Java changes still need one. Add `spring-boot-devtools` to `pom.xml` if you want automatic restarts on recompile too.

---

# Part 3 — Swagger UI

Swagger UI reads your controller annotations at runtime and renders a browsable, executable page for the API. No separate document to maintain, and no drift — the docs come from the code itself.

---

## 19. Add Swagger UI (OpenAPI documentation)

### 19.1 Add the dependency

Open `pom.xml` and add:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.1.1</version>
</dependency>
```

**The version matters more than usual here.** springdoc's major version tracks Spring Boot's:

| springdoc | Works with |
|---|---|
| **3.x** | **Spring Boot 4** ← this guide |
| 2.x | Spring Boot 3 |
| 1.x | Spring Boot 2 |

Most Swagger tutorials online still show `2.x`, because Boot 4 is recent. Paste one of those into this project and the app fails to start with a `NoClassDefFoundError` or `ClassNotFoundException` — springdoc 2.x is built against Spring Framework 6, this project runs Framework 7.

Two more things about this dependency:

- **SpringFox is dead.** If a tutorial tells you to add `io.springfox:springfox-boot-starter`, it's from 2020 and won't work on any Boot 3 or 4 project. springdoc is the maintained successor.
- **Check for a newer patch** at <https://central.sonatype.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui>. This dependency bundles Swagger UI, which is browser-facing JavaScript and has had XSS advisories — an old pinned version is a real (if small) exposure.

Unlike the Spring-managed starters, this one needs an explicit `<version>`, because it isn't part of the Spring Boot parent POM.

### 19.2 Start it and look

Restart:

```powershell
.\mvnw.cmd spring-boot:run
```

Open <http://localhost:8080/swagger-ui.html>.

That's the whole setup. Every endpoint on `ProductController` is already listed, with request and response schemas inferred from `ProductRequest` and `Product`, and the validation constraints from section 9 shown as field rules.

The raw specification sits at two URLs, useful for tooling:

- JSON: <http://localhost:8080/v3/api-docs>
- YAML: <http://localhost:8080/v3/api-docs.yaml>

### 19.3 Keep the web pages out of the docs

If you built Part 2, look at the endpoint list — the Thymeleaf pages are in there too, documented as endpoints that return a `string`. They aren't API endpoints and the noise is misleading.

springdoc inspects everything Spring routes, `@Controller` and `@RestController` alike. Limit it to the API in `application.properties`:

```properties
# --- OpenAPI / Swagger ---
springdoc.paths-to-match=/api/**
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.operationsSorter=method
springdoc.swagger-ui.tagsSorter=alpha
springdoc.swagger-ui.tryItOutEnabled=true
```

The alternative, if you prefer annotations to configuration, is `@Hidden` from `io.swagger.v3.oas.annotations` on `ProductWebController`. Either works; `paths-to-match` is one line and survives new controllers being added.

`operationsSorter=method` groups GET/POST/PUT/DELETE in a sensible order rather than the order Spring happened to register them.

### 19.4 Describe the API itself

Without this, the page header just says "OpenAPI definition". Create `OpenApiConfig.java`:

```java
package com.example.productapi;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI productApiDefinition() {
        return new OpenAPI().info(new Info()
                .title("Product API")
                .version("1.0.0")
                .description("CRUD API for managing products. Built with Spring Boot and MySQL.")
                .contact(new Contact().name("Your Name").email("you@example.com")));
    }
}
```

### 19.5 Annotate the endpoints

The defaults are usable, but a consumer reading them has to guess what a 409 means. Add descriptions to `ProductController` — the imports first:

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
```

Then annotate the class and its methods:

```java
@Tag(name = "Products", description = "Create, read, update and delete products")
@RestController
@RequestMapping("/api/products")
public class ProductController {

    // ...

    @Operation(
            summary = "List products",
            description = "Returns a paginated list. Use ?search= to filter by name or SKU.")
    @GetMapping
    public Page<Product> index(
            @Parameter(description = "Matches name or SKU, case-insensitive")
            @RequestParam(required = false) String search,
            @PageableDefault(size = 15, sort = "id", direction = Sort.Direction.DESC)
            Pageable pageable) {
        // ... unchanged
    }

    @Operation(summary = "Create a product")
    @ApiResponses({
            @ApiResponse(responseCode = "201", description = "Product created"),
            @ApiResponse(responseCode = "400", description = "Validation failed", content = @Content),
            @ApiResponse(responseCode = "409", description = "SKU already in use", content = @Content)
    })
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product store(@Valid @RequestBody ProductRequest request) {
        // ... unchanged
    }

    @Operation(summary = "Fetch one product by ID")
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Found"),
            @ApiResponse(responseCode = "404", description = "No product with that ID", content = @Content)
    })
    @GetMapping("/{id}")
    public Product show(
            @Parameter(description = "Product ID", example = "1")
            @PathVariable Long id) {
        // ... unchanged
    }
```

`@ApiResponse(..., content = @Content)` — importing `io.swagger.v3.oas.annotations.media.Content` — means "this status has no documented body", which stops Swagger inventing a `Product` schema for your error responses.

You can also document the fields themselves. In `ProductRequest`:

```java
import io.swagger.v3.oas.annotations.media.Schema;

public record ProductRequest(

        @Schema(description = "Display name", example = "Mechanical Keyboard")
        @NotBlank(message = "Name is required")
        @Size(max = 255)
        String name,

        @Schema(description = "Stock keeping unit, unique across products", example = "KB-8700")
        @NotBlank(message = "SKU is required")
        @Size(max = 64)
        String sku,

        // ...
) {}
```

Those `example` values become the pre-filled request body in Swagger UI, which makes "Try it out" usable without typing anything.

### 19.6 Execute a request from the browser

Restart, reload <http://localhost:8080/swagger-ui.html>, then:

1. Expand **POST /api/products**
2. Click **Try it out**
3. The body is pre-filled from your `@Schema(example = ...)` values — edit the SKU so it's unique
4. Click **Execute**

You get the response body, the status code, the response headers, and the equivalent curl command. The row is now in MySQL — check with **GET /api/products**, or in phpMyAdmin.

This is the part worth showing anyone consuming the API: they can exercise every endpoint without installing Postman or writing a line of code.

### 19.7 Import the spec into Postman

If you already built the Postman collection in section 13, you can now generate one instead:

**Import** → **Link** → `http://localhost:8080/v3/api-docs` → **Continue** → **Import**.

Postman builds a collection with every endpoint, method, and example body from the spec. Set its `baseUrl` variable to `http://localhost:8080` and it's ready. Re-import after adding endpoints and the collection stays in step with the code — which hand-maintained collections never do.

### 19.8 Turn it off in production

Swagger UI is a complete, executable map of your API. That's ideal in development and rarely what you want facing the public internet. For a production build:

```properties
springdoc.api-docs.enabled=false
springdoc.swagger-ui.enabled=false
```

Put those in `application-prod.properties` so they apply only under that profile, and your development setup is untouched. The alternative, if consumers need the docs, is to keep them enabled behind authentication.





## 20. Write an automated test (optional)

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

## 21. Troubleshooting

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
| `TemplateInputException: template might not exist` | Template path doesn't match the returned view name | File must be `src/main/resources/templates/products/form.html` for `return "products/form"` |
| **Uploaded image always empty, no error** | `enctype="multipart/form-data"` missing from the form tag | Add it — this is the most common upload bug |
| `MaxUploadSizeExceededException` | File larger than the 1MB default | Set `spring.servlet.multipart.max-file-size` (§14.2) |
| Image saves but shows as a broken icon | Missing trailing slash in `addResourceLocations` | Use `"file:" + path + "/"` (§14.4) |
| Can't find the `uploads` folder | Created relative to the working directory | Run from the project root, or set an absolute `app.upload-dir` |
| `Neither BindingResult nor plain target object for bean name 'productForm'` | Model attribute name doesn't match `th:object` | Both must say `productForm` |
| Validation errors never display; exception thrown instead | `BindingResult` not immediately after the `@Valid` parameter | Reorder the method parameters (§16) |
| Unchecked "Active" box saves as null | Plain `<input type="checkbox">` without `th:field` | Use `th:field="*{active}"` so Thymeleaf adds the hidden companion field |
| `405 Method Not Allowed` on delete | HTML forms can't send DELETE | Post to `/products/{id}/delete` (§16) |
| Price rejected as "Failed to convert" | Locale using a comma decimal separator | Type `1250.00`, or add a `@NumberFormat` annotation |
| Template edits need a restart | Template caching on | `spring.thymeleaf.cache=false` in development |
| Timestamps returned as huge numbers | Jackson serialising `Instant` as epoch | Add `spring.jackson.serialization.write-dates-as-timestamps=false` |
| Odd driver errors against MariaDB | MySQL Connector/J edge case | Swap in `org.mariadb.jdbc:mariadb-java-client` and change the URL to `jdbc:mariadb://...` |

When a build behaves strangely:

```powershell
.\mvnw.cmd clean
.\mvnw.cmd spring-boot:run
```

---

## 22. Command cheat sheet

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

# Docs (app running)
start http://localhost:8080/swagger-ui.html
curl.exe http://localhost:8080/v3/api-docs.yaml -o openapi.yaml

# Uploads
dir uploads
C:\xampp\mysql\bin\mysql.exe -u root product_api -e "SELECT id, name, image_filename FROM products;"

# Inspect
.\mvnw.cmd dependency:tree
java -version
```

---

## 23. Laravel to Spring Boot, side by side

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
| API docs | Scribe / L5-Swagger | springdoc-openapi |
| Templating | Blade (`.blade.php`) | Thymeleaf (`.html`) |
| Form binding | `old()` helper | `th:object` + `th:field` |
| Redisplay on error | automatic via session | return the view name + `BindingResult` |
| File upload | `$request->file('image')->store()` | `MultipartFile` + your own storage class |
| Serving uploads | `php artisan storage:link` | `WebMvcConfigurer` resource handler |
| Test DB isolation | `RefreshDatabase` (drops tables) | `@Transactional` (rolls back) |

The biggest practical differences: Spring makes you write the 404 check that Laravel's route model binding does implicitly, and Spring's pagination is zero-indexed.

---

## 24. Where to go next

1. **Add a service layer.** Move logic out of the controller into a `@Service` class once methods grow past a few lines. Controllers should handle HTTP; services should hold business rules.
2. **Split the packages** into `controller`, `service`, `repository`, `model`, `dto` — standard once you pass a handful of classes.
3. **Add a response DTO.** Return a `ProductResponse` record instead of the entity, so a database column change can't silently alter your API contract.
4. **Add PATCH** for partial updates: a second DTO with all-nullable fields, applying only non-null values.
5. **Replace `ddl-auto` with Flyway.** Add `flyway-core` plus `flyway-mysql`, put versioned SQL in `src/main/resources/db/migration`, and set `ddl-auto=validate`. This is what you'd do before any real deployment.
6. **Add Spring Security** with JWT once the API needs authentication.
7. **Generate a client from the OpenAPI spec.** Feed `/v3/api-docs` to openapi-generator and it produces a typed client for TypeScript, Java, Dart or whatever the consuming app uses.
8. **Add Testcontainers** to run tests against a real MySQL in CI without depending on a local XAMPP install.
9. **Harden the uploads.** Verify file contents rather than trusting the extension (a `.jpg` can hold anything), strip EXIF metadata, and generate thumbnails with Thumbnailator or imgscalr.
10. **Move uploads off local disk** before deploying to more than one server — S3 or any object store — since local files don't exist on the next instance.
11. **Extract a Thymeleaf layout fragment** so the list and form pages share one header and stylesheet instead of duplicating them.

### Reference links

- Spring Boot reference: <https://docs.spring.io/spring-boot/index.html>
- Spring Data JPA query methods: <https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html>
- Jakarta Bean Validation constraints: <https://jakarta.ee/specifications/bean-validation/>
- Building a REST service (official guide): <https://spring.io/guides/gs/rest-service>
