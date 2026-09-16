# Spring Boot Quick-Start Tutorial

Welcome to this beginner-friendly reference guide for **Spring Boot**. This document covers the foundational elements needed to bootstrap, construct, and run a production-ready REST API.

---

## 1. Project Initialization

The fastest way to scaffold a Spring Boot application is via [Spring Initializr](https://start.spring.io/).

### Recommended Project Settings:
* **Project:** Maven Project
* **Language:** Java
* **Packaging:** Jar
* **Java Version:** 17 or higher
* **Dependencies:**
  * `Spring Web` (for building REST APIs)
  * `Spring Data JPA` (for SQL database access)
  * `H2 Database` (for an in-memory testing database)

---

## 2. Main Entry Point

Every Spring Boot application requires a driver class marked with `@SpringBootApplication`. This single annotation combines configuration, component scanning, and auto-configuration defaults.

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

## 3. Creating a REST Controller

Controllers process incoming HTTP requests and map them directly into domain responses. Use `@RestController` alongside HTTP verb mappings like `@GetMapping` and `@PostMapping`.

```java
package com.example.demo.controller;

import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/v1/messages")
public class MessageController {

    @GetMapping
    public List<String> getMessages() {
        return List.of("Welcome to Spring Boot!", "Markdown Tutorial File Generated Successfully.");
    }
    
    @PostMapping
    public String createMessage(@RequestBody String newMessage) {
        return "Received: " + newMessage;
    }
}
```

---

## 4. Key Core Concepts

| Concept / Annotation | Target Layer | Core Responsibility |
| :--- | :--- | :--- |
| **`@Component`** | Class | Tells the Spring container to auto-detect and manage this class as a Bean. |
| **`@Autowired`** | Field/Constructor | Triggers implicit Dependency Injection (DI) to connect mandatory collaborative objects. |
| **`@Service`** | Business Logic | Represents stateless specialized services implementing business routines. |
| **`@Repository`** | Data Access | Abstracts infrastructure querying; acts as a broker to database storage. |

---

## 5. Application Properties

Configure your runtime traits (like system ports or database credentials) cleanly inside `src/main/resources/application.properties`:

```properties
# System Configuration
server.port=8080

# In-Memory Database Configurations
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=true
```


# SPRING API — Spring Boot, MySQL, Session Auth

A complete build: session-based login, user registration, role-based access, product and user CRUD, and image upload with server-side compression. Every response uses a custom envelope (`code`, `message`, `data`).

**Stack:** Spring Boot 4.1 · Spring Security 7 · Spring Data JPA · MySQL (XAMPP) · Thumbnailator · springdoc 3.x
**Database:** `ShopNowDB`
**Tested with:** Postman
**Platform:** Windows, PowerShell

---

## Table of Contents

1. [What you'll build](#1-what-youll-build)
2. [Prerequisites](#2-prerequisites)
3. [Create the project](#3-create-the-project)
4. [Create the database](#4-create-the-database)
5. [application.properties](#5-applicationproperties)
6. [Package structure](#6-package-structure)
7. [Entities](#7-entities)
8. [Repositories](#8-repositories)
9. [The response envelope](#9-the-response-envelope)
10. [Exception handling](#10-exception-handling)
11. [Security configuration](#11-security-configuration)
12. [Authentication endpoints](#12-authentication-endpoints)
13. [Image upload with compression](#13-image-upload-with-compression)
14. [Product CRUD](#14-product-crud)
15. [User CRUD](#15-user-crud)
16. [Categories](#16-categories)
17. [Seed the reference data](#17-seed-the-reference-data)
18. [Swagger UI](#18-swagger-ui)
19. [Run and test in Postman](#19-run-and-test-in-postman)
20. [Troubleshooting](#20-troubleshooting)
21. [Where to go next](#21-where-to-go-next)

---

## 1. What you'll build

### Endpoints

| Method | URI | Access | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | public | Create an account |
| POST | `/api/auth/login` | public | Start a session |
| POST | `/api/auth/logout` | authenticated | End the session |
| GET | `/api/auth/me` | authenticated | Current user |
| POST | `/api/auth/me/photo` | authenticated | Upload own profile picture |
| GET | `/api/products` | public | List (paginated, searchable) |
| GET | `/api/products/{id}` | public | One product |
| POST | `/api/products` | SELLER, ADMIN | Create, with image |
| PUT | `/api/products/{id}` | owner or ADMIN | Update fields |
| POST | `/api/products/{id}/image` | owner or ADMIN | Replace image |
| DELETE | `/api/products/{id}` | owner or ADMIN | Delete |
| GET | `/api/users` | ADMIN | List users |
| GET | `/api/users/{id}` | ADMIN | One user |
| PUT | `/api/users/{id}` | ADMIN | Update fullname/role |
| DELETE | `/api/users/{id}` | ADMIN | Delete |
| GET | `/api/categories` | public | List categories |
| POST | `/api/categories` | ADMIN | Create category |

### The response shape

Every response — success or failure — uses the same envelope:

```json
{
  "code": 201,
  "message": "New user registered",
  "data": { "id": 4, "username": "sarah", "fullname": "Sarah Dev", "roleId": 2, "roleName": "SELLER", "profilePictureUrl": null }
}
```

Validation failures add an `errors` map and omit `data`:

```json
{
  "code": 422,
  "message": "Validation failed",
  "errors": { "username": "Username is required", "password": "Password must be at least 8 characters" }
}
```

> **One design note up front.** A `DELETE` conventionally returns `204 No Content` — an empty body. That's incompatible with an envelope that always carries `code` and `message`, so deletes here return **200** with the envelope instead. Consistency is worth more than convention when clients parse one shape for everything.

---

## 2. Prerequisites

Same three as the earlier Spring guide — if you already have them, skip ahead.

| Need | Check | Get it |
|---|---|---|
| JDK 21 | `java -version` | <https://adoptium.net/temurin/releases/> — tick **Set JAVA_HOME** and **Add to PATH** |
| XAMPP | MySQL green in Control Panel | <https://www.apachefriends.org/download.html> |
| Postman | — | <https://www.postman.com/downloads/> |

Maven isn't needed; the project ships `mvnw.cmd`.

---

## 3. Create the project

Go to **<https://start.spring.io>**:

| Field | Value |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | latest 4.1.x (not SNAPSHOT/M) |
| Group | `com.example` |
| Artifact | `shopnow` |
| Package name | `com.example.shopnow` |
| Packaging | Jar |
| Java | 21 |

Dependencies:

- **Spring Web**
- **Spring Data JPA**
- **MySQL Driver**
- **Validation**
- **Spring Security**

**GENERATE**, extract to `C:\dev\shopnow`, then add two more to `pom.xml` by hand — these aren't on start.spring.io:

```xml
<!-- Image compression -->
<dependency>
    <groupId>net.coobird</groupId>
    <artifactId>thumbnailator</artifactId>
    <version>0.4.20</version>
</dependency>

<!-- Swagger UI / OpenAPI -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.1.1</version>
</dependency>
```

springdoc **3.x** is the line built for Spring Boot 4 — a 2.x version from an older tutorial will fail to start here.

---

## 4. Create the database

Start MySQL in XAMPP, then:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE ShopNowDB CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

Hibernate creates the tables from your entities, so there's no SQL schema to write.

---

## 5. application.properties

Replace `src/main/resources/application.properties`:

```properties
spring.application.name=shopnow
server.port=8080

# --- Database ---
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/ShopNowDB?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=

# --- JPA ---
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.open-in-view=false

# --- Sessions ---
server.servlet.session.timeout=30m
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.same-site=lax

# --- Uploads ---
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
app.upload.dir=uploads
app.upload.max-dimension=1000
app.upload.quality=0.8

# --- OpenAPI ---
springdoc.paths-to-match=/api/**
springdoc.swagger-ui.path=/swagger-ui.html

# --- Errors ---
server.error.include-message=always
```

`127.0.0.1` rather than `localhost`: on Windows `localhost` often resolves to IPv6 `::1`, where MariaDB isn't listening, and the failure looks like the server is down.

Add `uploads/` to `.gitignore`.

---

## 6. Package structure

This project is big enough that a flat package stops helping:

```
com.example.shopnow
├── ShopnowApplication.java
├── config/
│   ├── SecurityConfig.java
│   ├── WebConfig.java
│   ├── OpenApiConfig.java
│   └── DataSeeder.java
├── model/
│   ├── Role.java  Category.java  User.java  Product.java
├── repository/
│   ├── RoleRepository.java  CategoryRepository.java
│   ├── UserRepository.java  ProductRepository.java
├── dto/
│   ├── ApiResponse.java
│   ├── RegisterRequest.java  LoginRequest.java
│   ├── ProductCreateRequest.java  ProductUpdateRequest.java
│   ├── UserUpdateRequest.java  CategoryRequest.java
│   ├── UserResponse.java  ProductResponse.java  CategoryResponse.java
├── security/
│   ├── AppUserPrincipal.java
│   └── AppUserDetailsService.java
├── service/
│   ├── ImageStorageService.java
├── exception/
│   ├── ResourceNotFoundException.java
│   ├── DuplicateResourceException.java
│   └── GlobalExceptionHandler.java
└── controller/
    ├── AuthController.java  ProductController.java
    ├── UserController.java  CategoryController.java
```

All of it sits under `com.example.shopnow`, the same package as the application class — Spring only scans that package and below.

---

## 7. Entities

### `model/Role.java`

```java
package com.example.shopnow.model;

import jakarta.persistence.*;

@Entity
@Table(name = "role")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, unique = true, length = 50)
    private String name;

    public Role() {}

    public Role(String name) { this.name = name; }

    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

### `model/Category.java`

```java
package com.example.shopnow.model;

import jakarta.persistence.*;

@Entity
@Table(name = "category")
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(nullable = false, unique = true, length = 100)
    private String name;

    public Category() {}

    public Category(String name) { this.name = name; }

    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

### `model/User.java`

```java
package com.example.shopnow.model;

import jakarta.persistence.*;

@Entity
@Table(name = "user")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false, length = 150)
    private String fullname;

    @Column(nullable = false, length = 100)
    private String password;          // BCrypt hash — never plain text

    @ManyToOne(fetch = FetchType.EAGER, optional = false)
    @JoinColumn(name = "roleID", nullable = false)
    private Role role;

    @Column(name = "profilePictureUrl", length = 255)
    private String profilePictureUrl;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getFullname() { return fullname; }
    public void setFullname(String fullname) { this.fullname = fullname; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public Role getRole() { return role; }
    public void setRole(Role role) { this.role = role; }

    public String getProfilePictureUrl() { return profilePictureUrl; }
    public void setProfilePictureUrl(String profilePictureUrl) { this.profilePictureUrl = profilePictureUrl; }
}
```

Two notes on this one:

**`password` is a BCrypt hash, 60 characters.** Never store what the user typed. The column is 100 to leave room if you switch algorithms.

**`user` is a valid table name in MySQL/MariaDB** but it's a reserved word in PostgreSQL and a few others. Keeping your schema exactly as specified, it stays `user` — if you ever move to Postgres, change the annotation to `@Table(name = "users")` and nothing else breaks.

### `model/Product.java`

```java
package com.example.shopnow.model;

import jakarta.persistence.*;

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String name;

    @ManyToOne(fetch = FetchType.EAGER, optional = false)
    @JoinColumn(name = "categoryID", nullable = false)
    private Category category;

    @Column(name = "imageUrl", length = 255)
    private String imageUrl;

    @ManyToOne(fetch = FetchType.EAGER, optional = false)
    @JoinColumn(name = "sellerID", nullable = false)
    private User seller;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Category getCategory() { return category; }
    public void setCategory(Category category) { this.category = category; }

    public String getImageUrl() { return imageUrl; }
    public void setImageUrl(String imageUrl) { this.imageUrl = imageUrl; }

    public User getSeller() { return seller; }
    public void setSeller(User seller) { this.seller = seller; }
}
```

`@JoinColumn(name = "categoryID")` keeps your exact column names. Without it Hibernate would generate `category_id` from the field name.

---

## 8. Repositories

Four interfaces, no implementations.

```java
// repository/RoleRepository.java
package com.example.shopnow.repository;

import com.example.shopnow.model.Role;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface RoleRepository extends JpaRepository<Role, Integer> {
    Optional<Role> findByNameIgnoreCase(String name);
}
```

```java
// repository/CategoryRepository.java
package com.example.shopnow.repository;

import com.example.shopnow.model.Category;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CategoryRepository extends JpaRepository<Category, Integer> {
    boolean existsByNameIgnoreCase(String name);
}
```

```java
// repository/UserRepository.java
package com.example.shopnow.repository;

import com.example.shopnow.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
}
```

```java
// repository/ProductRepository.java
package com.example.shopnow.repository;

import com.example.shopnow.model.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByNameContainingIgnoreCase(String name, Pageable pageable);
    Page<Product> findByCategoryId(Integer categoryId, Pageable pageable);
}
```

---

## 9. The response envelope

### `dto/ApiResponse.java`

```java
package com.example.shopnow.dto;

import com.fasterxml.jackson.annotation.JsonInclude;

import java.time.Instant;
import java.util.Map;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record ApiResponse<T>(
        int code,
        String message,
        T data,
        Map<String, String> errors,
        Instant timestamp
) {

    public static <T> ApiResponse<T> success(int code, String message, T data) {
        return new ApiResponse<>(code, message, data, null, Instant.now());
    }

    public static <T> ApiResponse<T> message(int code, String message) {
        return new ApiResponse<>(code, message, null, null, Instant.now());
    }

    public static <T> ApiResponse<T> error(int code, String message, Map<String, String> errors) {
        return new ApiResponse<>(code, message, null, errors, Instant.now());
    }
}
```

`@JsonInclude(NON_NULL)` is what keeps the output clean — `errors` disappears from success responses and `data` disappears from failures, instead of both showing as `null` everywhere.

### Request DTOs

```java
// dto/RegisterRequest.java
package com.example.shopnow.dto;

import jakarta.validation.constraints.*;

public record RegisterRequest(
        @NotBlank(message = "Username is required")
        @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
        @Pattern(regexp = "^[a-zA-Z0-9._-]+$", message = "Username may only contain letters, numbers, dot, underscore and hyphen")
        String username,

        @NotBlank(message = "Full name is required")
        @Size(max = 150)
        String fullname,

        @NotBlank(message = "Password is required")
        @Size(min = 8, message = "Password must be at least 8 characters")
        String password,

        @NotNull(message = "Role is required")
        Integer roleId
) {}
```

```java
// dto/LoginRequest.java
package com.example.shopnow.dto;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
        @NotBlank(message = "Username is required") String username,
        @NotBlank(message = "Password is required") String password
) {}
```

```java
// dto/ProductUpdateRequest.java
package com.example.shopnow.dto;

import jakarta.validation.constraints.*;

public record ProductUpdateRequest(
        @NotBlank(message = "Name is required") @Size(max = 200) String name,
        @NotNull(message = "Category is required") Integer categoryId
) {}
```

```java
// dto/UserUpdateRequest.java
package com.example.shopnow.dto;

import jakarta.validation.constraints.*;

public record UserUpdateRequest(
        @NotBlank(message = "Full name is required") @Size(max = 150) String fullname,
        @NotNull(message = "Role is required") Integer roleId
) {}
```

```java
// dto/CategoryRequest.java
package com.example.shopnow.dto;

import jakarta.validation.constraints.*;

public record CategoryRequest(
        @NotBlank(message = "Name is required") @Size(max = 100) String name
) {}
```

`ProductCreateRequest` is a **class**, not a record, because it binds from `multipart/form-data` and carries the file:

```java
// dto/ProductCreateRequest.java
package com.example.shopnow.dto;

import jakarta.validation.constraints.*;
import org.springframework.web.multipart.MultipartFile;

public class ProductCreateRequest {

    @NotBlank(message = "Name is required")
    @Size(max = 200)
    private String name;

    @NotNull(message = "Category is required")
    private Integer categoryId;

    private MultipartFile image;   // optional

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Integer getCategoryId() { return categoryId; }
    public void setCategoryId(Integer categoryId) { this.categoryId = categoryId; }

    public MultipartFile getImage() { return image; }
    public void setImage(MultipartFile image) { this.image = image; }
}
```

### Response DTOs

Never return the entities — `User` carries a password hash, and `Product` carries a `User`.

```java
// dto/UserResponse.java
package com.example.shopnow.dto;

import com.example.shopnow.model.User;

public record UserResponse(
        Long id, String username, String fullname,
        Integer roleId, String roleName, String profilePictureUrl
) {
    public static UserResponse from(User u) {
        return new UserResponse(
                u.getId(), u.getUsername(), u.getFullname(),
                u.getRole().getId(), u.getRole().getName(), u.getProfilePictureUrl());
    }
}
```

```java
// dto/ProductResponse.java
package com.example.shopnow.dto;

import com.example.shopnow.model.Product;

public record ProductResponse(
        Long id, String name,
        Integer categoryId, String categoryName,
        String imageUrl,
        Long sellerId, String sellerUsername
) {
    public static ProductResponse from(Product p) {
        return new ProductResponse(
                p.getId(), p.getName(),
                p.getCategory().getId(), p.getCategory().getName(),
                p.getImageUrl(),
                p.getSeller().getId(), p.getSeller().getUsername());
    }
}
```

```java
// dto/CategoryResponse.java
package com.example.shopnow.dto;

import com.example.shopnow.model.Category;

public record CategoryResponse(Integer id, String name) {
    public static CategoryResponse from(Category c) {
        return new CategoryResponse(c.getId(), c.getName());
    }
}
```

---

## 10. Exception handling

Two small exceptions and one handler that turns everything into the envelope.

```java
// exception/ResourceNotFoundException.java
package com.example.shopnow.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

```java
// exception/DuplicateResourceException.java
package com.example.shopnow.exception;

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) { super(message); }
}
```

```java
// exception/GlobalExceptionHandler.java
package com.example.shopnow.exception;

import com.example.shopnow.dto.ApiResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.multipart.MaxUploadSizeExceededException;

import java.util.LinkedHashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.putIfAbsent(e.getField(), e.getDefaultMessage()));

        return ResponseEntity.unprocessableEntity()
                .body(ApiResponse.error(422, "Validation failed", errors));
    }

    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<ApiResponse<Void>> handleBadCredentials(BadCredentialsException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(ApiResponse.message(401, "Invalid username or password"));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiResponse<Void>> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(ApiResponse.message(403, "You do not have permission to perform this action"));
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.message(404, ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiResponse<Void>> handleDuplicate(DuplicateResourceException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(ApiResponse.message(409, ex.getMessage()));
    }

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<ApiResponse<Void>> handleTooLarge(MaxUploadSizeExceededException ex) {
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
                .body(ApiResponse.message(413, "Image is too large. Maximum size is 10MB"));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ApiResponse<Void>> handleIllegalArgument(IllegalArgumentException ex) {
        return ResponseEntity.badRequest()
                .body(ApiResponse.message(400, ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleEverythingElse(Exception ex) {
        ex.printStackTrace();   // replace with a logger in production
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.message(500, "Something went wrong"));
    }
}
```

**"Invalid username or password" is deliberately vague.** Saying "no such user" instead tells an attacker which usernames exist, which is half of a credential-stuffing attack.

The catch-all `Exception` handler stops a stack trace ever reaching the client — but it also hides real bugs, so keep `ex.printStackTrace()` (or a logger) or you'll debug blind.

---

## 11. Security configuration

### `security/AppUserPrincipal.java`

Spring Security's own `User` class doesn't carry your entity's ID, and you need it to stamp `sellerID` on products. So wrap your entity:

```java
package com.example.shopnow.security;

import com.example.shopnow.model.User;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

public class AppUserPrincipal implements UserDetails {

    private final User user;

    public AppUserPrincipal(User user) { this.user = user; }

    public User getUser() { return user; }
    public Long getId() { return user.getId(); }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole().getName().toUpperCase()));
    }

    @Override public String getPassword() { return user.getPassword(); }
    @Override public String getUsername() { return user.getUsername(); }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return true; }
}
```

**The `ROLE_` prefix is not optional.** `hasRole("ADMIN")` looks for an authority literally named `ROLE_ADMIN`. Store `ADMIN` in the `role` table, prefix it here, and both sides are happy. Forgetting this produces 403s that make no sense.

### `security/AppUserDetailsService.java`

```java
package com.example.shopnow.security;

import com.example.shopnow.repository.UserRepository;
import org.springframework.security.core.userdetails.*;
import org.springframework.stereotype.Service;

@Service
public class AppUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public AppUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepository.findByUsername(username)
                .map(AppUserPrincipal::new)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }
}
```

### `config/SecurityConfig.java`

```java
package com.example.shopnow.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.example.shopnow.dto.ApiResponse;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.http.MediaType;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.security.web.context.SecurityContextRepository;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final ObjectMapper objectMapper;

    public SecurityConfig(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityContextRepository securityContextRepository() {
        return new HttpSessionSecurityContextRepository();
    }

    @Bean
    public AuthenticationManager authenticationManager(UserDetailsService userDetailsService,
                                                       PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return new ProviderManager(provider);
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                           SecurityContextRepository securityContextRepository) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .securityContext(sc -> sc.securityContextRepository(securityContextRepository))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/register", "/api/auth/login").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**", "/api/categories/**").permitAll()
                .requestMatchers("/uploads/**").permitAll()
                .requestMatchers("/swagger-ui.html", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/api/users/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, authException) ->
                        write(response, 401, "Authentication required"))
                .accessDeniedHandler((request, response, deniedException) ->
                        write(response, 403, "You do not have permission to perform this action")))
            .logout(logout -> logout
                .logoutUrl("/api/auth/logout")
                .deleteCookies("JSESSIONID")
                .invalidateHttpSession(true)
                .logoutSuccessHandler((request, response, authentication) ->
                        write(response, 200, "Logout successful")));

        return http.build();
    }

    private void write(HttpServletResponse response, int code, String message) throws java.io.IOException {
        response.setStatus(code);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(ApiResponse.message(code, message)));
    }
}
```

### What each piece is for

**`csrf.disable()` — understand this one before shipping.** Session cookies are sent automatically by browsers, which is exactly what CSRF attacks exploit. Disabling CSRF is fine while the only client is Postman, and fine for a graded build, but a browser front end on this API needs it back on:

```java
.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
```

The front end then reads the `XSRF-TOKEN` cookie and echoes it in an `X-XSRF-TOKEN` header. The `same-site=lax` cookie setting in `application.properties` already blocks the simplest cross-site attempts.

**The custom entry point and denied handler** exist so that an unauthenticated call returns your envelope, not Spring's default HTML page. Without them, a client parsing `code`/`message` breaks on exactly the responses it most needs to read.

**`.logout()` is configured, not hand-written.** Spring Security's logout filter clears the context, saves it, and invalidates the session in the right order — all three matter, and hand-rolled logout usually misses one.

**Rule order is top to bottom, first match wins.** `/api/users/**` must come before `anyRequest()`, and the `GET` product rule before any broader product rule.

---

## 12. Authentication endpoints

### `controller/AuthController.java`

```java
package com.example.shopnow.controller;

import com.example.shopnow.dto.*;
import com.example.shopnow.exception.DuplicateResourceException;
import com.example.shopnow.exception.ResourceNotFoundException;
import com.example.shopnow.model.Role;
import com.example.shopnow.model.User;
import com.example.shopnow.repository.RoleRepository;
import com.example.shopnow.repository.UserRepository;
import com.example.shopnow.security.AppUserPrincipal;
import com.example.shopnow.service.ImageStorageService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.context.SecurityContextRepository;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;
    private final AuthenticationManager authenticationManager;
    private final SecurityContextRepository securityContextRepository;
    private final ImageStorageService imageStorageService;

    public AuthController(UserRepository userRepository,
                          RoleRepository roleRepository,
                          PasswordEncoder passwordEncoder,
                          AuthenticationManager authenticationManager,
                          SecurityContextRepository securityContextRepository,
                          ImageStorageService imageStorageService) {
        this.userRepository = userRepository;
        this.roleRepository = roleRepository;
        this.passwordEncoder = passwordEncoder;
        this.authenticationManager = authenticationManager;
        this.securityContextRepository = securityContextRepository;
        this.imageStorageService = imageStorageService;
    }

    /** POST /api/auth/register */
    @PostMapping("/register")
    public ResponseEntity<ApiResponse<UserResponse>> register(@Valid @RequestBody RegisterRequest request) {

        if (userRepository.existsByUsername(request.username())) {
            throw new DuplicateResourceException("Username is already taken");
        }

        Role role = roleRepository.findById(request.roleId())
                .orElseThrow(() -> new ResourceNotFoundException("Role " + request.roleId() + " does not exist"));

        User user = new User();
        user.setUsername(request.username());
        user.setFullname(request.fullname());
        user.setPassword(passwordEncoder.encode(request.password()));   // hash, never store raw
        user.setRole(role);

        User saved = userRepository.save(user);

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(201, "New user registered", UserResponse.from(saved)));
    }

    /** POST /api/auth/login */
    @PostMapping("/login")
    public ResponseEntity<ApiResponse<UserResponse>> login(@Valid @RequestBody LoginRequest request,
                                                           HttpServletRequest httpRequest,
                                                           HttpServletResponse httpResponse) {

        Authentication authentication = authenticationManager.authenticate(
                UsernamePasswordAuthenticationToken.unauthenticated(request.username(), request.password()));

        // Create a context, put it in the holder, and persist it to the session.
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);
        securityContextRepository.saveContext(context, httpRequest, httpResponse);

        AppUserPrincipal principal = (AppUserPrincipal) authentication.getPrincipal();

        return ResponseEntity.ok(
                ApiResponse.success(200, "Login successful", UserResponse.from(principal.getUser())));
    }

    /** GET /api/auth/me */
    @GetMapping("/me")
    public ResponseEntity<ApiResponse<UserResponse>> me(@AuthenticationPrincipal AppUserPrincipal principal) {
        return ResponseEntity.ok(
                ApiResponse.success(200, "Current user", UserResponse.from(principal.getUser())));
    }

    /** POST /api/auth/me/photo */
    @PostMapping(value = "/me/photo", consumes = "multipart/form-data")
    public ResponseEntity<ApiResponse<UserResponse>> uploadOwnPhoto(
            @AuthenticationPrincipal AppUserPrincipal principal,
            @RequestParam("image") MultipartFile image) {

        User user = userRepository.findById(principal.getId())
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        String previous = user.getProfilePictureUrl();
        user.setProfilePictureUrl(imageStorageService.storeCompressed(image));
        imageStorageService.delete(previous);

        return ResponseEntity.ok(
                ApiResponse.success(200, "Profile picture updated", UserResponse.from(userRepository.save(user))));
    }
}
```

### The three lines that make session login work

```java
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(authentication);
SecurityContextHolder.setContext(context);
securityContextRepository.saveContext(context, httpRequest, httpResponse);
```

Since Spring Security 6, setting the `SecurityContextHolder` **no longer persists anything**. `SecurityContextHolderFilter` reads the context at the start of a request but never writes it back, so a custom login endpoint must call `saveContext()` itself.

Leave that line out and the symptom is maddening: login returns 200, Postman shows a `JSESSIONID` cookie, and the very next request comes back 401. Nearly every "my Spring session login doesn't stick" question online is this one missing call.

There's no `/register` authentication step — registration creates the account, login starts the session. Keeping them separate means registration failures can't leave a half-authenticated session behind.

---

## 13. Image upload with compression

### `service/ImageStorageService.java`

```java
package com.example.shopnow.service;

import net.coobird.thumbnailator.Thumbnails;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import org.springframework.web.multipart.MultipartFile;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.nio.file.*;
import java.util.List;
import java.util.Locale;
import java.util.UUID;

@Service
public class ImageStorageService {

    private static final List<String> ALLOWED = List.of("jpg", "jpeg", "png", "webp", "gif");

    private final Path root;
    private final int maxDimension;
    private final double quality;

    public ImageStorageService(@Value("${app.upload.dir}") String uploadDir,
                               @Value("${app.upload.max-dimension}") int maxDimension,
                               @Value("${app.upload.quality}") double quality) {
        this.root = Paths.get(uploadDir).toAbsolutePath().normalize();
        this.maxDimension = maxDimension;
        this.quality = quality;
        try {
            Files.createDirectories(root);
        } catch (IOException e) {
            throw new UncheckedIOException("Could not create upload directory: " + root, e);
        }
    }

    /**
     * Validates, compresses and stores the upload.
     * @return the public URL path, e.g. /uploads/9f1c....jpg
     */
    public String storeCompressed(MultipartFile file) {
        if (file == null || file.isEmpty()) {
            throw new IllegalArgumentException("No image was uploaded");
        }

        String original = StringUtils.cleanPath(
                file.getOriginalFilename() == null ? "" : file.getOriginalFilename());
        String extension = StringUtils.getFilenameExtension(original);

        if (extension == null || !ALLOWED.contains(extension.toLowerCase(Locale.ROOT))) {
            throw new IllegalArgumentException("Only JPG, PNG, WEBP or GIF images are allowed");
        }

        // Confirm the bytes really are an image — an extension proves nothing
        try (InputStream probe = file.getInputStream()) {
            BufferedImage check = ImageIO.read(probe);
            if (check == null) {
                throw new IllegalArgumentException("That file is not a readable image");
            }
        } catch (IOException e) {
            throw new IllegalArgumentException("That file could not be read as an image");
        }

        String filename = UUID.randomUUID() + ".jpg";
        Path target = root.resolve(filename).normalize();

        if (!target.getParent().equals(root)) {
            throw new IllegalArgumentException("Invalid file name");
        }

        try (InputStream in = file.getInputStream()) {
            Thumbnails.of(in)
                      .size(maxDimension, maxDimension)   // fits inside the box, aspect ratio kept
                      .outputFormat("jpg")
                      .outputQuality(quality)
                      .toFile(target.toFile());
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to store image", e);
        }

        return "/uploads/" + filename;
    }

    /** Accepts the stored URL path; safe with null or a file that's already gone. */
    public void delete(String urlPath) {
        if (urlPath == null || urlPath.isBlank()) return;

        String filename = urlPath.substring(urlPath.lastIndexOf('/') + 1);
        try {
            Files.deleteIfExists(root.resolve(filename).normalize());
        } catch (IOException ignored) {
            // A leftover file isn't worth failing a request over
        }
    }
}
```

### What the compression actually does

`Thumbnails.of(...).size(1000, 1000).outputFormat("jpg").outputQuality(0.8)` does four useful things in one chain:

1. **Scales down** so the longest side is at most 1000px, preserving aspect ratio. A 4000×3000 phone photo becomes 1000×750. `size()` only shrinks by default, so a small image isn't upscaled and blurred.
2. **Re-encodes as JPEG at 80% quality** — visually near-identical, typically 10–20× smaller. A 4 MB photo lands around 150–250 KB.
3. **Strips EXIF metadata** as a side effect of re-encoding. Phone photos carry GPS coordinates; publishing those unintentionally is a real privacy leak.
4. **Neutralises a malicious file.** Decoding to a `BufferedImage` and re-encoding means whatever was in the original bytes doesn't survive.

Tune with `app.upload.max-dimension` and `app.upload.quality` in `application.properties` — no recompile needed.

Two caveats: **PNG transparency is lost** when converting to JPEG (transparent areas turn black), so if you need transparent logos, branch on the source format rather than forcing `.jpg`. And **animated GIFs lose animation** — only the first frame survives.

The `ImageIO.read()` probe is worth the extra few lines. A file named `payload.jpg` can contain anything; if `ImageIO` can't decode it, it never reaches disk.

### `config/WebConfig.java` — serve the stored files

```java
package com.example.shopnow.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.nio.file.Path;
import java.nio.file.Paths;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final String uploadDir;

    public WebConfig(@Value("${app.upload.dir}") String uploadDir) {
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

**That trailing slash is mandatory.** Without it Spring treats the location as a file rather than a directory and every image 404s, with nothing in the logs to explain why.

---

## 14. Product CRUD

### `controller/ProductController.java`

```java
package com.example.shopnow.controller;

import com.example.shopnow.dto.*;
import com.example.shopnow.exception.ResourceNotFoundException;
import com.example.shopnow.model.Category;
import com.example.shopnow.model.Product;
import com.example.shopnow.model.User;
import com.example.shopnow.repository.CategoryRepository;
import com.example.shopnow.repository.ProductRepository;
import com.example.shopnow.repository.UserRepository;
import com.example.shopnow.security.AppUserPrincipal;
import com.example.shopnow.service.ImageStorageService;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final UserRepository userRepository;
    private final ImageStorageService imageStorageService;

    public ProductController(ProductRepository productRepository,
                             CategoryRepository categoryRepository,
                             UserRepository userRepository,
                             ImageStorageService imageStorageService) {
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
        this.userRepository = userRepository;
        this.imageStorageService = imageStorageService;
    }

    /** GET /api/products?search=&categoryId=&page=&size= */
    @GetMapping
    public ResponseEntity<ApiResponse<Map<String, Object>>> list(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) Integer categoryId,
            @PageableDefault(size = 10, sort = "id", direction = Sort.Direction.DESC) Pageable pageable) {

        Page<Product> page;
        if (search != null && !search.isBlank()) {
            page = productRepository.findByNameContainingIgnoreCase(search, pageable);
        } else if (categoryId != null) {
            page = productRepository.findByCategoryId(categoryId, pageable);
        } else {
            page = productRepository.findAll(pageable);
        }

        List<ProductResponse> items = page.getContent().stream()
                .map(ProductResponse::from)
                .toList();

        Map<String, Object> body = Map.of(
                "items", items,
                "page", page.getNumber(),
                "size", page.getSize(),
                "totalItems", page.getTotalElements(),
                "totalPages", page.getTotalPages());

        return ResponseEntity.ok(ApiResponse.success(200, "Products retrieved", body));
    }

    /** GET /api/products/{id} */
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponse>> show(@PathVariable Long id) {
        Product product = findOrThrow(id);
        return ResponseEntity.ok(ApiResponse.success(200, "Product retrieved", ProductResponse.from(product)));
    }

    /** POST /api/products — multipart: name, categoryId, image */
    @PostMapping(consumes = "multipart/form-data")
    @PreAuthorize("hasAnyRole('SELLER','ADMIN')")
    public ResponseEntity<ApiResponse<ProductResponse>> create(
            @Valid @ModelAttribute ProductCreateRequest request,
            @AuthenticationPrincipal AppUserPrincipal principal) {

        Category category = categoryRepository.findById(request.getCategoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category " + request.getCategoryId() + " does not exist"));

        User seller = userRepository.findById(principal.getId())
                .orElseThrow(() -> new ResourceNotFoundException("Seller not found"));

        Product product = new Product();
        product.setName(request.getName());
        product.setCategory(category);
        product.setSeller(seller);

        MultipartFile image = request.getImage();
        if (image != null && !image.isEmpty()) {
            product.setImageUrl(imageStorageService.storeCompressed(image));
        }

        Product saved = productRepository.save(product);

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(201, "New product created", ProductResponse.from(saved)));
    }

    /** PUT /api/products/{id} — JSON body, fields only */
    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponse>> update(
            @PathVariable Long id,
            @Valid @RequestBody ProductUpdateRequest request,
            @AuthenticationPrincipal AppUserPrincipal principal) {

        Product product = findOrThrow(id);
        assertCanModify(product, principal);

        Category category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category " + request.categoryId() + " does not exist"));

        product.setName(request.name());
        product.setCategory(category);

        return ResponseEntity.ok(ApiResponse.success(200, "Product updated",
                ProductResponse.from(productRepository.save(product))));
    }

    /** POST /api/products/{id}/image — multipart, replaces the image */
    @PostMapping(value = "/{id}/image", consumes = "multipart/form-data")
    public ResponseEntity<ApiResponse<ProductResponse>> uploadImage(
            @PathVariable Long id,
            @RequestParam("image") MultipartFile image,
            @AuthenticationPrincipal AppUserPrincipal principal) {

        Product product = findOrThrow(id);
        assertCanModify(product, principal);

        String previous = product.getImageUrl();
        product.setImageUrl(imageStorageService.storeCompressed(image));
        imageStorageService.delete(previous);

        return ResponseEntity.ok(ApiResponse.success(200, "Product image updated",
                ProductResponse.from(productRepository.save(product))));
    }

    /** DELETE /api/products/{id} */
    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> delete(@PathVariable Long id,
                                                    @AuthenticationPrincipal AppUserPrincipal principal) {
        Product product = findOrThrow(id);
        assertCanModify(product, principal);

        imageStorageService.delete(product.getImageUrl());
        productRepository.delete(product);

        return ResponseEntity.ok(ApiResponse.message(200, "Product deleted"));
    }

    // --- helpers ---

    private Product findOrThrow(Long id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No product found with ID " + id));
    }

    /** A seller may only touch their own products; an admin may touch any. */
    private void assertCanModify(Product product, AppUserPrincipal principal) {
        boolean isAdmin = principal.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        boolean isOwner = product.getSeller().getId().equals(principal.getId());

        if (!isAdmin && !isOwner) {
            throw new AccessDeniedException("You can only modify your own products");
        }
    }
}
```

### Three decisions worth explaining

**Create is multipart; update is JSON; the image has its own endpoint.** Tomcat only reliably parses `multipart/form-data` on POST — a `PUT` with form-data is a known source of "all my fields are null" on Windows. Splitting image replacement into `POST /{id}/image` avoids that entirely and gives you an endpoint the client can call on its own when only the photo changed.

**`@ModelAttribute`, not `@RequestPart`.** With `@RequestPart("product")` for a JSON part, Postman requires you to set a content type per part, which people forget constantly. `@ModelAttribute` binds plain form-data text fields straight onto `ProductCreateRequest`, so the Postman body is just three rows.

**`sellerID` comes from the session, never the request body.** If the client supplied it, any seller could publish products under another seller's name. `@AuthenticationPrincipal` gives you the logged-in user, which is the only trustworthy source.

`@PreAuthorize` works because `@EnableMethodSecurity` is on `SecurityConfig`. The ownership rule in `assertCanModify` is a check that URL patterns can't express — it depends on the row, not the path.

---

## 15. User CRUD

### `controller/UserController.java`

Admin-only; the whole path is locked down in `SecurityConfig` by `.requestMatchers("/api/users/**").hasRole("ADMIN")`.

```java
package com.example.shopnow.controller;

import com.example.shopnow.dto.*;
import com.example.shopnow.exception.ResourceNotFoundException;
import com.example.shopnow.model.Role;
import com.example.shopnow.model.User;
import com.example.shopnow.repository.RoleRepository;
import com.example.shopnow.repository.UserRepository;
import com.example.shopnow.security.AppUserPrincipal;
import com.example.shopnow.service.ImageStorageService;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final ImageStorageService imageStorageService;

    public UserController(UserRepository userRepository,
                          RoleRepository roleRepository,
                          ImageStorageService imageStorageService) {
        this.userRepository = userRepository;
        this.roleRepository = roleRepository;
        this.imageStorageService = imageStorageService;
    }

    /** GET /api/users */
    @GetMapping
    public ResponseEntity<ApiResponse<Map<String, Object>>> list(
            @PageableDefault(size = 10, sort = "id", direction = Sort.Direction.ASC) Pageable pageable) {

        Page<User> page = userRepository.findAll(pageable);

        List<UserResponse> items = page.getContent().stream().map(UserResponse::from).toList();

        Map<String, Object> body = Map.of(
                "items", items,
                "page", page.getNumber(),
                "size", page.getSize(),
                "totalItems", page.getTotalElements(),
                "totalPages", page.getTotalPages());

        return ResponseEntity.ok(ApiResponse.success(200, "Users retrieved", body));
    }

    /** GET /api/users/{id} */
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<UserResponse>> show(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(200, "User retrieved", UserResponse.from(findOrThrow(id))));
    }

    /** PUT /api/users/{id} — fullname and role only */
    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<UserResponse>> update(@PathVariable Long id,
                                                            @Valid @RequestBody UserUpdateRequest request) {
        User user = findOrThrow(id);

        Role role = roleRepository.findById(request.roleId())
                .orElseThrow(() -> new ResourceNotFoundException("Role " + request.roleId() + " does not exist"));

        user.setFullname(request.fullname());
        user.setRole(role);

        return ResponseEntity.ok(ApiResponse.success(200, "User updated",
                UserResponse.from(userRepository.save(user))));
    }

    /** POST /api/users/{id}/photo */
    @PostMapping(value = "/{id}/photo", consumes = "multipart/form-data")
    public ResponseEntity<ApiResponse<UserResponse>> uploadPhoto(@PathVariable Long id,
                                                                 @RequestParam("image") MultipartFile image) {
        User user = findOrThrow(id);

        String previous = user.getProfilePictureUrl();
        user.setProfilePictureUrl(imageStorageService.storeCompressed(image));
        imageStorageService.delete(previous);

        return ResponseEntity.ok(ApiResponse.success(200, "Profile picture updated",
                UserResponse.from(userRepository.save(user))));
    }

    /** DELETE /api/users/{id} */
    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> delete(@PathVariable Long id,
                                                    @AuthenticationPrincipal AppUserPrincipal principal) {
        User user = findOrThrow(id);

        if (user.getId().equals(principal.getId())) {
            throw new IllegalArgumentException("You cannot delete your own account while logged in");
        }

        imageStorageService.delete(user.getProfilePictureUrl());
        userRepository.delete(user);

        return ResponseEntity.ok(ApiResponse.message(200, "User deleted"));
    }

    private User findOrThrow(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("No user found with ID " + id));
    }
}
```

**There's no password field on `UserUpdateRequest` on purpose.** Password changes want their own endpoint that requires the current password — folding them into a general update means an admin (or an XSS payload) can silently take over any account. Section 21 covers adding it.

**The self-delete guard** stops an admin deleting the account they're logged in as, which leaves a live session pointing at a row that no longer exists and produces confusing 500s on the next request.

Deleting a user with products will fail on the foreign key — that's the database protecting referential integrity, and it's the right default. Decide deliberately whether to reassign or cascade before changing it.

---

## 16. Categories

### `controller/CategoryController.java`

```java
package com.example.shopnow.controller;

import com.example.shopnow.dto.*;
import com.example.shopnow.exception.DuplicateResourceException;
import com.example.shopnow.model.Category;
import com.example.shopnow.repository.CategoryRepository;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/categories")
public class CategoryController {

    private final CategoryRepository categoryRepository;

    public CategoryController(CategoryRepository categoryRepository) {
        this.categoryRepository = categoryRepository;
    }

    /** GET /api/categories — public */
    @GetMapping
    public ResponseEntity<ApiResponse<List<CategoryResponse>>> list() {
        List<CategoryResponse> items = categoryRepository.findAll().stream()
                .map(CategoryResponse::from)
                .toList();

        return ResponseEntity.ok(ApiResponse.success(200, "Categories retrieved", items));
    }

    /** POST /api/categories — admin only */
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<CategoryResponse>> create(@Valid @RequestBody CategoryRequest request) {

        if (categoryRepository.existsByNameIgnoreCase(request.name())) {
            throw new DuplicateResourceException("That category already exists");
        }

        Category saved = categoryRepository.save(new Category(request.name()));

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(201, "New category created", CategoryResponse.from(saved)));
    }
}
```

---

## 17. Seed the reference data

You can't register anyone until roles exist, and there's no admin account to create the first one. Seed both at startup.

### `config/DataSeeder.java`

```java
package com.example.shopnow.config;

import com.example.shopnow.model.Category;
import com.example.shopnow.model.Role;
import com.example.shopnow.model.User;
import com.example.shopnow.repository.CategoryRepository;
import com.example.shopnow.repository.RoleRepository;
import com.example.shopnow.repository.UserRepository;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.List;

@Configuration
public class DataSeeder {

    @Bean
    public CommandLineRunner seed(RoleRepository roleRepository,
                                  CategoryRepository categoryRepository,
                                  UserRepository userRepository,
                                  PasswordEncoder passwordEncoder) {
        return args -> {

            for (String name : List.of("ADMIN", "SELLER", "BUYER")) {
                roleRepository.findByNameIgnoreCase(name)
                        .orElseGet(() -> roleRepository.save(new Role(name)));
            }

            for (String name : List.of("Electronics", "Fashion", "Home & Living", "Groceries", "Books")) {
                if (!categoryRepository.existsByNameIgnoreCase(name)) {
                    categoryRepository.save(new Category(name));
                }
            }

            if (!userRepository.existsByUsername("admin")) {
                Role adminRole = roleRepository.findByNameIgnoreCase("ADMIN").orElseThrow();

                User admin = new User();
                admin.setUsername("admin");
                admin.setFullname("System Administrator");
                admin.setPassword(passwordEncoder.encode("admin12345"));
                admin.setRole(adminRole);

                userRepository.save(admin);
                System.out.println(">>> Seeded admin account: admin / admin12345");
            }
        };
    }
}
```

Every block checks before inserting, so restarting doesn't duplicate anything.

**The seeded password is a development convenience and nothing more.** Before this is reachable by anyone else, change it, or read it from an environment variable and fail startup if it's unset.

---

## 18. Swagger UI

### `config/OpenApiConfig.java`

```java
package com.example.shopnow.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI shopNowApiDefinition() {
        return new OpenAPI().info(new Info()
                .title("ShopNow API")
                .version("1.0.0")
                .description("Marketplace API — session authentication, product and user management, image upload."));
    }
}
```

Open <http://localhost:8080/swagger-ui.html> once the app is running. The spec itself is at `/v3/api-docs`.

Swagger's "Try it out" works for authenticated endpoints too: log in through `POST /api/auth/login` on that same page and the browser holds the `JSESSIONID` cookie for the rest of the session.

---

## 19. Run and test in Postman

```powershell
cd C:\dev\shopnow
.\mvnw.cmd spring-boot:run
```

Watch for the seeded admin line and `Tomcat started on port 8080`.

### 19.1 Collection and environment

**Collection:** `ShopNow API`.

**Environment** `ShopNow Local` with `base_url` = `http://localhost:8080/api`. Select it in the top-right dropdown.

Add `Accept: application/json` as a collection-level header (right-click the collection → Edit → Headers).

> **Cookies are handled for you.** Postman stores `JSESSIONID` from the login response and sends it automatically on later requests to the same host. You don't add an Authorization header anywhere. To inspect or clear it, use the **Cookies** link under the Send button.

### 19.2 Register — POST {{base_url}}/auth/register

Body → **raw** → **JSON**:

```json
{
  "username": "sarah",
  "fullname": "Sarah Dev",
  "password": "secret12345",
  "roleId": 2
}
```

`roleId` 2 is SELLER from the seeder (1 ADMIN, 2 SELLER, 3 BUYER — confirm with `GET /api/categories`-style listing or in phpMyAdmin).

**201 Created:**

```json
{
  "code": 201,
  "message": "New user registered",
  "data": {
    "id": 2,
    "username": "sarah",
    "fullname": "Sarah Dev",
    "roleId": 2,
    "roleName": "SELLER",
    "profilePictureUrl": null
  },
  "timestamp": "2026-09-17T04:12:33.918Z"
}
```

Check phpMyAdmin — the `password` column holds a `$2a$10$...` BCrypt hash, not `secret12345`.

Send it again → **409** "Username is already taken".

Send `{"username":"ab","password":"123"}` → **422**:

```json
{
  "code": 422,
  "message": "Validation failed",
  "errors": {
    "username": "Username must be 3-50 characters",
    "fullname": "Full name is required",
    "password": "Password must be at least 8 characters",
    "roleId": "Role is required"
  }
}
```

### 19.3 Login — POST {{base_url}}/auth/login

```json
{ "username": "sarah", "password": "secret12345" }
```

**200** with `"message": "Login successful"`. Open **Cookies** under Send and you'll see `JSESSIONID` for `localhost`.

Wrong password → **401** "Invalid username or password".

### 19.4 Confirm the session — GET {{base_url}}/auth/me

**200** with your user. If this returns **401** right after a successful login, the `saveContext()` call in section 12 is missing or the cookie isn't being sent.

### 19.5 Create a product — POST {{base_url}}/products

Body → **form-data** (not raw):

| Key | Type | Value |
|---|---|---|
| `name` | Text | `Mechanical Keyboard` |
| `categoryId` | Text | `1` |
| `image` | **File** | pick a large photo |

Change a row's type from Text to File using the dropdown that appears when you hover over the Key cell.

**201:**

```json
{
  "code": 201,
  "message": "New product created",
  "data": {
    "id": 1,
    "name": "Mechanical Keyboard",
    "categoryId": 1,
    "categoryName": "Electronics",
    "imageUrl": "/uploads/6f2a91c4-....jpg",
    "sellerId": 2,
    "sellerUsername": "sarah"
  }
}
```

**Now check the compression** — this is the part worth seeing:

```powershell
dir uploads
```

A 4 MB source photo lands around 150–250 KB. Open <http://localhost:8080/uploads/6f2a91c4-....jpg> in a browser to confirm it's a valid, correctly-oriented image.

### 19.6 The rest of the product endpoints

| Request | Method | URL | Body |
|---|---|---|---|
| List | GET | `{{base_url}}/products?page=0&size=5` | — |
| Search | GET | `{{base_url}}/products?search=keyboard` | — |
| One | GET | `{{base_url}}/products/1` | — |
| Update | PUT | `{{base_url}}/products/1` | raw JSON: `{"name":"Keyboard MK2","categoryId":1}` |
| Replace image | POST | `{{base_url}}/products/1/image` | form-data: `image` (File) |
| Delete | DELETE | `{{base_url}}/products/1` | — |

Pages are **zero-indexed** — `page=0` is the first.

### 19.7 Test the authorization rules

This is the part most worth verifying, because a passing happy path proves nothing about access control.

1. **Logged out:** POST **{{base_url}}/auth/logout**, then try `GET {{base_url}}/auth/me` → **401** "Authentication required". `GET {{base_url}}/products` still works — it's public.
2. **Wrong role:** log in as `sarah` (SELLER) and call `GET {{base_url}}/users` → **403** "You do not have permission to perform this action".
3. **Admin:** log in as `admin` / `admin12345`, call `GET {{base_url}}/users` → **200**.
4. **Ownership:** register a second seller, log in as them, and try `DELETE {{base_url}}/products/1` (owned by sarah) → **403** "You can only modify your own products". As `admin`, the same call succeeds.

Every one of those responses uses the same envelope, which is the point of the whole design.

### 19.8 Save the collection as a regression suite

Add a check in each request's **Scripts** tab:

```javascript
pm.test("Envelope is well formed", function () {
    const body = pm.response.json();
    pm.expect(body).to.have.property("code");
    pm.expect(body).to.have.property("message");
    pm.expect(body.code).to.eql(pm.response.code);
});
```

Then right-click the collection → **Run collection**. Order matters: register → login → create product → the rest.

---

## 20. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Login returns 200 but `/auth/me` returns 401 | `securityContextRepository.saveContext(...)` missing | See §12 — the context isn't persisted automatically since Security 6 |
| Every request 401 after restart | Sessions are in memory and die with the process | Log in again, or add Spring Session + JDBC to survive restarts |
| 403 on an endpoint the role should reach | Authority missing the `ROLE_` prefix | `AppUserPrincipal` must emit `ROLE_ADMIN`, not `ADMIN` |
| 403 on every POST from a browser client | CSRF enabled without a token being sent | Either disable for the API or send `X-XSRF-TOKEN` (§11) |
| `Encoded password does not look like BCrypt` | A plain-text password was inserted directly into the table | Register through the API, or hash before inserting |
| All form fields null on create | Body sent as raw JSON instead of form-data | Product create is **multipart**; use form-data |
| Image field ignored | Key type left as Text | Switch the row's type to **File** in Postman |
| `MaxUploadSizeExceededException` | File over the 10MB limit | Raise `spring.servlet.multipart.max-file-size` |
| Image saves but URL 404s | Missing trailing slash in `addResourceLocations` | `"file:" + path + "/"` (§13) |
| `That file is not a readable image` | Not actually an image, or an unsupported codec | Expected — the `ImageIO` probe rejected it |
| Transparent PNG turns black | JPEG has no alpha channel | Branch on source format instead of forcing `.jpg` |
| `Cannot delete or update a parent row` | Deleting a user or category still referenced by products | Reassign or delete the products first |
| `Unknown database 'ShopNowDB'` | Schema not created | Run the `CREATE DATABASE` in §4 |
| `Communications link failure` | MySQL not started | Start MySQL in the XAMPP Control Panel |
| Connection refused on `localhost` | Resolving to IPv6 `::1` | Use `127.0.0.1` |
| `Table 'product' doesn't exist` | `ddl-auto` not set to `update` | Check `application.properties` |
| springdoc `NoClassDefFoundError` | springdoc 2.x with Spring Boot 4 | Use springdoc **3.x** |
| `Port 8080 already in use` | Another app holds it | `server.port=8081`, or `netstat -ano \| findstr :8080` |

---

## 21. Where to go next

Roughly in the order you'll want them:

1. **Add a password-change endpoint** — `POST /api/auth/change-password` taking the current password and the new one, verified with `passwordEncoder.matches()`. Deliberately left out of the user update endpoint.
2. **Move the logic into services.** The controllers here hold repository calls directly, which is fine at this size. Once a method passes ~20 lines, `ProductService` and `UserService` earn their place — and they make the ownership rules testable without HTTP.
3. **Add Spring Session + JDBC** (`spring-session-jdbc`) so sessions live in MySQL and survive restarts. That's also what makes session auth work across more than one server.
4. **Turn CSRF back on** before any browser front end touches this.
5. **Add `createdAt` / `updatedAt`** to both entities with `@CreationTimestamp` and `@UpdateTimestamp` — you will want them the first time you debug a data question.
6. **Replace `ddl-auto=update` with Flyway** before deploying anywhere real. `update` never drops columns and silently accumulates cruft.
7. **Write tests** — `@SpringBootTest` + `MockMvc` with `@WithMockUser(roles = "ADMIN")` to exercise the access rules without logging in each time.
8. **Add rate limiting on login** (Bucket4j, or a failed-attempt counter). An unthrottled login endpoint is a standing invitation to credential stuffing.
9. **Move uploads to object storage** (S3 or similar) if this ever runs on more than one instance — local files don't exist on the next server.
10. **Soft-delete products** with a `deleted` flag rather than removing rows, so order history keeps working.
