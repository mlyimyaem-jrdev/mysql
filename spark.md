# Codeigniter 4
A powerful web application built using the [CodeIgniter 4 PHP Framework](https://codeigniter.com/).

## Features
* Built on the lightweight, high-performance MVC architecture of CodeIgniter 4.
* Comprehensive built-in security protections against CSRF and XSS attacks.
* Flexible routing, database migration tracking, and custom Entity mapping.

## Prerequisites
Before installing and running this application, ensure your environment meets the following requirements:
* **PHP:** version 8.1 or higher (with `intl`, `mbstring`, `json`, and database-specific extensions enabled).
* **Composer:** Installed globally for managing system dependencies.
* **Database:** MySQL/MariaDB or PostgreSQL server.

## Getting Started

Follow these steps to deploy and test the project locally.

### 1. Clone the Repository
```bash
git clone https://github.com
cd your-repo-name
```

### 2. Install Dependencies
Run Composer to fetch the required framework components:
```bash
composer install
```

### 3. Configure the Environment
The framework reads operational variables from an environmental layout. Copy the default template:
```bash
cp env .env
```
Open the newly created `.env` file and customize the following settings to match your machine setup:
```env
# Toggle between development and production modes
CI_ENVIRONMENT = development

# The primary destination URL for local testing
app.baseURL = 'http://localhost:8080/'

# Database credentials
database.default.hostname = localhost
database.default.database = project_db_name
database.default.username = root
database.default.password = secret_password
database.default.DBDriver = MySQLi
```

### 4. Run Database Migrations & Seeds
If your application relies on baseline schema tracking and initial test records, initialize them:
```bash
php spark migrate
php spark db:seed MainDatabaseSeeder
```

### 5. Launch the Local Development Server
Boot up CodeIgniter's internal PHP development engine:
```bash
php spark serve
```
Your application will now be live and accessible at **`http://localhost:8080`**.

## Running Tests
This project includes standard PHPUnit verification setups. Execute tests via:
```bash
vendor/bin/phpunit
```

# codeigniter API — CodeIgniter 4, MySQL, Session Auth

The same build as the Spring Boot version: session login, registration, role-based access, product and user CRUD, and image upload with compression. Every response uses a custom envelope (`code`, `message`, `data`).

**Stack:** CodeIgniter 4.7 · PHP 8.2 (XAMPP's own) · MySQL/MariaDB · GD image library
**Database:** `ShopNowDB`
**Tested with:** Postman
**Platform:** Windows, PowerShell

> **Two things are markedly easier here than in Spring.** Image compression is built into the framework — no Thumbnailator, no extra dependency. And uploads land in `public/uploads`, which is already the document root, so there's no resource handler to configure. In exchange you write the auth filters by hand that Spring Security gives you.

---

## Table of Contents

1. [What you'll build](#1-what-youll-build)
2. [Prerequisites](#2-prerequisites)
3. [Create the project](#3-create-the-project)
4. [Create the database](#4-create-the-database)
5. [Configure .env](#5-configure-env)
6. [Project structure](#6-project-structure)
7. [Migrations](#7-migrations)
8. [Models](#8-models)
9. [The response envelope](#9-the-response-envelope)
10. [Image upload with compression](#10-image-upload-with-compression)
11. [Authentication filters](#11-authentication-filters)
12. [Routes](#12-routes)
13. [Authentication controller](#13-authentication-controller)
14. [Product CRUD](#14-product-crud)
15. [User CRUD](#15-user-crud)
16. [Categories](#16-categories)
17. [Seed the reference data](#17-seed-the-reference-data)
18. [Run and test in Postman](#18-run-and-test-in-postman)
19. [Troubleshooting](#19-troubleshooting)
20. [CodeIgniter vs Spring Boot on this build](#20-codeigniter-vs-spring-boot-on-this-build)
21. [Where to go next](#21-where-to-go-next)

---

## 1. What you'll build

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
| POST | `/api/users/{id}/photo` | ADMIN | Upload a user's photo |
| DELETE | `/api/users/{id}` | ADMIN | Delete |
| GET | `/api/categories` | public | List categories |
| POST | `/api/categories` | ADMIN | Create category |

Every response, success or failure, uses one shape:

```json
{
  "code": 201,
  "message": "New user registered",
  "data": { "id": 2, "username": "sarah", "fullname": "Sarah Dev", "roleID": 2, "roleName": "SELLER", "profilePictureUrl": null },
  "timestamp": "2026-09-17T12:04:11+08:00"
}
```

Validation failures swap `data` for `errors`:

```json
{
  "code": 422,
  "message": "Validation failed",
  "errors": { "username": "Username is required", "password": "Password must be at least 8 characters" }
}
```

Deletes return **200** with the envelope rather than a bodyless 204, so clients parse one shape for everything.

---

## 2. Prerequisites

| Need | Check | Get it |
|---|---|---|
| PHP 8.2+ | `php -v` | XAMPP's bundled PHP is enough — add `C:\xampp\php` to your PATH |
| Composer | `composer -V` | <https://getcomposer.org/download/> |
| XAMPP MySQL | green in Control Panel | <https://www.apachefriends.org/download.html> |
| Postman | — | <https://www.postman.com/downloads/> |

### Required PHP extensions

```powershell
php -m | Select-String "intl|mbstring|json|mysqli|gd"
```

All five must appear. **`gd` is the one this project adds** — it's what does the image resizing, and CodeIgniter's image library is useless without it. If `intl` or `gd` is missing, open `C:\xampp\php\php.ini`, remove the leading `;` from:

```ini
extension=gd
extension=intl
```

Save and open a **new** terminal. Apache doesn't need restarting since you'll run `spark serve`.

---

## 3. Create the project

```powershell
cd C:\dev
composer create-project codeigniter4/appstarter shopnow-ci
cd shopnow-ci
php spark
```

The CodeIgniter banner means PHP and its extensions are fine.

---

## 4. Create the database

Start MySQL in XAMPP, then:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE ShopNowDB CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

---

## 5. Configure .env

The skeleton ships a file named `env` with no dot. Copy it:

```powershell
copy env .env
```

Open `.env` and set these — **remove the leading `#` on each line you change**, or the value is ignored:

```env
CI_ENVIRONMENT = development

app.baseURL = 'http://localhost:8080/'

database.default.hostname = 127.0.0.1
database.default.database = ShopNowDB
database.default.username = root
database.default.password =
database.default.DBDriver = MySQLi
database.default.port = 3306

# --- Sessions ---
session.driver = 'CodeIgniter\Session\Handlers\FileHandler'
session.expiration = 1800

# --- Uploads (read by App\Libraries\ImageStorage) ---
app.upload.maxDimension = 1000
app.upload.quality = 80
```

Three notes:

- **`CI_ENVIRONMENT = development`** gives you real error pages. On `production` every mistake is a generic message with no stack trace.
- **`127.0.0.1`, not `localhost`** — on Windows `localhost` often resolves to IPv6 `::1`, where MariaDB isn't listening. The error looks like the server is down.
- **`database.default.password =`** is deliberately empty, matching XAMPP's default root account.

---

## 6. Project structure

```
shopnow-ci/
├── app/
│   ├── Config/
│   │   ├── Routes.php          ← §12
│   │   └── Filters.php         ← §11
│   ├── Controllers/Api/
│   │   ├── BaseApiController.php   ← §9
│   │   ├── Auth.php                ← §13
│   │   ├── Products.php            ← §14
│   │   ├── Users.php               ← §15
│   │   └── Categories.php          ← §16
│   ├── Database/
│   │   ├── Migrations/  (4 files)  ← §7
│   │   └── Seeds/ShopNowSeeder.php ← §17
│   ├── Filters/
│   │   ├── AuthFilter.php          ← §11
│   │   └── RoleFilter.php
│   ├── Libraries/
│   │   └── ImageStorage.php        ← §10
│   └── Models/
│       ├── RoleModel.php  CategoryModel.php
│       ├── UserModel.php  ProductModel.php
├── public/
│   ├── index.php
│   └── uploads/                ← created at runtime, served directly
└── writable/logs/
```

---

## 7. Migrations

Generate four files. **Create them in this order** — the timestamps in the filenames control execution order, and the foreign keys need their parent tables to exist first.

```powershell
php spark make:migration CreateRole
php spark make:migration CreateCategory
php spark make:migration CreateUser
php spark make:migration CreateProduct
```

### `CreateRole`

```php
<?php

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateRole extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'   => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true],
            'name' => ['type' => 'VARCHAR', 'constraint' => 50],
        ]);

        $this->forge->addKey('id', true);
        $this->forge->addUniqueKey('name');
        $this->forge->createTable('role');
    }

    public function down(): void
    {
        $this->forge->dropTable('role');
    }
}
```

### `CreateCategory`

```php
<?php

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateCategory extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'   => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true],
            'name' => ['type' => 'VARCHAR', 'constraint' => 100],
        ]);

        $this->forge->addKey('id', true);
        $this->forge->addUniqueKey('name');
        $this->forge->createTable('category');
    }

    public function down(): void
    {
        $this->forge->dropTable('category');
    }
}
```

### `CreateUser`

```php
<?php

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateUser extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'       => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true],
            'username' => ['type' => 'VARCHAR', 'constraint' => 50],
            'fullname' => ['type' => 'VARCHAR', 'constraint' => 150],
            'password' => ['type' => 'VARCHAR', 'constraint' => 255],
            'roleID'   => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true],
            'profilePictureUrl' => ['type' => 'VARCHAR', 'constraint' => 255, 'null' => true],
        ]);

        $this->forge->addKey('id', true);
        $this->forge->addUniqueKey('username');
        $this->forge->addForeignKey('roleID', 'role', 'id', 'RESTRICT', 'RESTRICT');
        $this->forge->createTable('user');
    }

    public function down(): void
    {
        $this->forge->dropTable('user');
    }
}
```

**`password` is 255, not 60.** PHP's `password_hash()` with the default algorithm produces 60 characters today, but the documentation explicitly warns that length can grow as algorithms change. A too-short column truncates the hash silently and nobody can log in.

### `CreateProduct`

```php
<?php

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateProduct extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id'         => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true, 'auto_increment' => true],
            'name'       => ['type' => 'VARCHAR', 'constraint' => 200],
            'categoryID' => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true],
            'imageUrl'   => ['type' => 'VARCHAR', 'constraint' => 255, 'null' => true],
            'sellerID'   => ['type' => 'INT', 'constraint' => 11, 'unsigned' => true],
        ]);

        $this->forge->addKey('id', true);
        $this->forge->addKey('categoryID');
        $this->forge->addKey('sellerID');
        $this->forge->addForeignKey('categoryID', 'category', 'id', 'RESTRICT', 'RESTRICT');
        $this->forge->addForeignKey('sellerID', 'user', 'id', 'RESTRICT', 'RESTRICT');
        $this->forge->createTable('product');
    }

    public function down(): void
    {
        $this->forge->dropTable('product');
    }
}
```

Run them:

```powershell
php spark migrate
```

`RESTRICT` on both foreign keys means you can't delete a category or a seller while products reference them. That's the safe default — it turns a silent data-loss bug into an error you have to think about.

---

## 8. Models

### `app/Models/RoleModel.php`

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class RoleModel extends Model
{
    protected $table         = 'role';
    protected $primaryKey    = 'id';
    protected $returnType    = 'array';
    protected $allowedFields = ['name'];
    protected $useTimestamps = false;
}
```

### `app/Models/CategoryModel.php`

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class CategoryModel extends Model
{
    protected $table         = 'category';
    protected $primaryKey    = 'id';
    protected $returnType    = 'array';
    protected $allowedFields = ['name'];
    protected $useTimestamps = false;
}
```

### `app/Models/UserModel.php`
 protected $validationRules  = [
        'email' => 'required|valid_email|is_unique[users.email,id,{id}]',
    ];
```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class UserModel extends Model
{
    protected $table         = 'user';
    protected $primaryKey    = 'id';
    protected $returnType    = 'array';
    protected $useTimestamps = false;

    protected $allowedFields = [
        'username', 'fullname', 'password', 'roleID', 'profilePictureUrl',
    ];

    /** Fields safe to return over the API — never includes `password`. */
    private const PUBLIC_FIELDS =
        'user.id, user.username, user.fullname, user.roleID, user.profilePictureUrl, role.name AS roleName';

    public function findPublic(int $id): ?array
    {
        return $this->select(self::PUBLIC_FIELDS)
                    ->join('role', 'role.id = user.roleID')
                    ->where('user.id', $id)
                    ->first();
    }

    public function pagePublic(int $perPage): array
    {
        return $this->select(self::PUBLIC_FIELDS)
                    ->join('role', 'role.id = user.roleID')
                    ->orderBy('user.id', 'ASC')
                    ->paginate($perPage);
    }

    /** Includes the password hash — only for the login check. */
    public function findForLogin(string $username): ?array
    {
        return $this->select('user.*, role.name AS roleName')
                    ->join('role', 'role.id = user.roleID')
                    ->where('user.username', $username)
                    ->first();
    }
}
```

**`findPublic()` and `findForLogin()` are separate on purpose.** One query selects the password hash and one doesn't, and keeping them apart means you can't accidentally serialise a hash into a response. `SELECT *` on a user table is how password hashes end up in JSON.

### `app/Models/ProductModel.php`

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class ProductModel extends Model
{
    protected $table         = 'product';
    protected $primaryKey    = 'id';
    protected $returnType    = 'array';
    protected $useTimestamps = false;

    protected $allowedFields = ['name', 'categoryID', 'imageUrl', 'sellerID'];

    private const JOINED =
        'product.id, product.name, product.categoryID, product.imageUrl, product.sellerID,
         category.name AS categoryName, user.username AS sellerUsername';

    private function joined()
    {
        return $this->select(self::JOINED)
                    ->join('category', 'category.id = product.categoryID')
                    ->join('user', 'user.id = product.sellerID');
    }

    public function findJoined(int $id): ?array
    {
        return $this->joined()->where('product.id', $id)->first();
    }

    public function pageJoined(?string $search, ?int $categoryId, int $perPage): array
    {
        $builder = $this->joined();

        if ($search !== null && $search !== '') {
            $builder = $builder->groupStart()
                               ->like('product.name', $search)
                               ->groupEnd();
        }

        if ($categoryId !== null) {
            $builder = $builder->where('product.categoryID', $categoryId);
        }

        return $builder->orderBy('product.id', 'DESC')->paginate($perPage);
    }
}
```

---

## 9. The response envelope

### `app/Controllers/Api/BaseApiController.php`

```php
<?php

namespace App\Controllers\Api;

use App\Controllers\BaseController;
use CodeIgniter\HTTP\ResponseInterface;

abstract class BaseApiController extends BaseController
{
    protected function apiSuccess(int $code, string $message, $data = null): ResponseInterface
    {
        $body = ['code' => $code, 'message' => $message];

        if ($data !== null) {
            $body['data'] = $data;
        }

        $body['timestamp'] = date('c');

        return $this->response->setStatusCode($code)->setJSON($body);
    }

    protected function apiError(int $code, string $message, ?array $errors = null): ResponseInterface
    {
        $body = ['code' => $code, 'message' => $message];

        if (! empty($errors)) {
            $body['errors'] = $errors;
        }

        $body['timestamp'] = date('c');

        return $this->response->setStatusCode($code)->setJSON($body);
    }

    /**
     * Validates an array against rules, returning the error map or null.
     */
    protected function validateData(array $data, array $rules): ?array
    {
        $validation = \Config\Services::validation();

        if ($validation->setRules($rules)->run($data)) {
            return null;
        }

        return $validation->getErrors();
    }

    /** Body of a JSON request as an array. */
    protected function jsonBody(): array
    {
        return $this->request->getJSON(true) ?? [];
    }

    /** The logged-in user's ID, or null. */
    protected function currentUserId(): ?int
    {
        $id = session()->get('userId');

        return $id === null ? null : (int) $id;
    }

    protected function currentRole(): ?string
    {
        return session()->get('roleName');
    }

    protected function isAdmin(): bool
    {
        return $this->currentRole() === 'ADMIN';
    }
}
```

**Why `validateData()` instead of the usual `$this->validate()`.** CodeIgniter's controller helper reads from `getVar()`, which is populated from form-encoded input. For a raw JSON body that can quietly validate nothing at all, depending on version and content type. Running the validation service against an array you decoded yourself removes the ambiguity — it behaves identically for JSON and form-data callers.

---

## 10. Image upload with compression

CodeIgniter has an image manipulation service built in, so there's no dependency to add. It needs the **GD** extension from section 2.

### `app/Libraries/ImageStorage.php`

```php
<?php

namespace App\Libraries;

use CodeIgniter\HTTP\Files\UploadedFile;
use RuntimeException;

class ImageStorage
{
    private const ALLOWED_TYPES = [IMAGETYPE_JPEG, IMAGETYPE_PNG, IMAGETYPE_WEBP, IMAGETYPE_GIF];

    private string $directory;
    private int $maxDimension;
    private int $quality;

    public function __construct()
    {
        $this->directory    = FCPATH . 'uploads';
        $this->maxDimension = (int) (env('app.upload.maxDimension') ?: 1000);
        $this->quality      = (int) (env('app.upload.quality') ?: 80);

        if (! is_dir($this->directory)) {
            mkdir($this->directory, 0775, true);
        }
    }

    /**
     * Validates, compresses and stores an upload.
     *
     * @return string the public URL path, e.g. /uploads/9f1c....jpg
     */
    public function storeCompressed(?UploadedFile $file): string
    {
        if ($file === null || ! $file->isValid()) {
            throw new RuntimeException('No image was uploaded');
        }

        // Inspect the actual bytes — an extension proves nothing
        $info = @getimagesize($file->getTempName());

        if ($info === false || ! in_array($info[2], self::ALLOWED_TYPES, true)) {
            throw new RuntimeException('Only JPG, PNG, WEBP or GIF images are allowed');
        }

        [$width, $height] = $info;

        $filename = bin2hex(random_bytes(16)) . '.jpg';
        $target   = $this->directory . DIRECTORY_SEPARATOR . $filename;

        $image = service('image')->withFile($file->getTempName());

        // Only shrink — never upscale a small image into a blurry big one
        if ($width > $this->maxDimension || $height > $this->maxDimension) {
            $image->resize($this->maxDimension, $this->maxDimension, true, 'auto');
        }

        $image->convert(IMAGETYPE_JPEG)->save($target, $this->quality);

        return '/uploads/' . $filename;
    }

    /** Accepts the stored URL path. Safe with null or an already-deleted file. */
    public function delete(?string $urlPath): void
    {
        if (empty($urlPath)) {
            return;
        }

        $path = $this->directory . DIRECTORY_SEPARATOR . basename($urlPath);

        if (is_file($path)) {
            @unlink($path);
        }
    }
}
```

### What this actually does

**`getimagesize()` on the temp file is the security check.** It reads the real bytes and returns `false` for anything that isn't a decodable image, so a file named `payload.jpg` containing something else never reaches disk. Checking the extension alone — or `$file->getClientExtension()`, which is supplied by the client — proves nothing.

**`basename()` on delete** strips any path the stored value might contain, so a tampered database value can't make this unlink a file elsewhere.

**`bin2hex(random_bytes(16))` for the filename.** Never reuse the client's filename: `..\..\app\Config\Database.php` would escape the folder, and two users' `photo.jpg` would overwrite each other.

**`resize(..., true, 'auto')`** keeps the aspect ratio and lets CodeIgniter pick the master dimension, so a 4000×3000 photo becomes 1000×750. The size check around it matters — `resize()` will happily enlarge a 200px image to 1000px and produce a blurry mess.

**`convert(IMAGETYPE_JPEG)->save($target, 80)`** re-encodes at 80% quality. A 4 MB phone photo typically lands at 150–250 KB. Re-encoding also strips EXIF metadata as a side effect, which matters more than it sounds: phone photos carry GPS coordinates.

Two caveats: **PNG transparency is lost** converting to JPEG (transparent areas go black), so branch on `$info[2]` if you need transparent logos. And **animated GIFs lose animation** — only the first frame survives.

**Uploads go in `public/uploads`**, which is already the document root. Unlike the Spring version there's no resource handler to write — `/uploads/abc.jpg` just works.

---

## 11. Authentication filters

Spring Security gives you the filter chain; in CodeIgniter you write it. It's about 40 lines.

### `app/Filters/AuthFilter.php`

```php
<?php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class AuthFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if (! session()->get('isLoggedIn')) {
            return service('response')
                ->setStatusCode(401)
                ->setJSON([
                    'code'      => 401,
                    'message'   => 'Authentication required',
                    'timestamp' => date('c'),
                ]);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        // nothing to do
    }
}
```

### `app/Filters/RoleFilter.php`

```php
<?php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class RoleFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        if (! session()->get('isLoggedIn')) {
            return $this->deny(401, 'Authentication required');
        }

        $role = session()->get('roleName');

        if (! empty($arguments) && ! in_array($role, $arguments, true)) {
            return $this->deny(403, 'You do not have permission to perform this action');
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
    }

    private function deny(int $code, string $message)
    {
        return service('response')
            ->setStatusCode($code)
            ->setJSON(['code' => $code, 'message' => $message, 'timestamp' => date('c')]);
    }
}
```

**Returning a response object from `before()` short-circuits the request** — the controller never runs. Returning nothing lets it through. That's the whole contract.

The `$arguments` come from the route: `'filter' => 'role:ADMIN,SELLER'` arrives as `['ADMIN', 'SELLER']`.

Both filters emit the same envelope as everything else, so a client never has to parse two shapes.

### Register them — `app/Config/Filters.php`

Add to the `$aliases` array:

```php
public array $aliases = [
    'csrf'          => CSRF::class,
    'toolbar'       => DebugToolbar::class,
    'honeypot'      => Honeypot::class,
    'invalidchars'  => InvalidChars::class,
    'secureheaders' => SecureHeaders::class,
    'forcehttps'    => ForceHTTPS::class,
    'pagecache'     => PageCache::class,
    'performance'   => PerformanceMetrics::class,

    // added
    'auth' => \App\Filters\AuthFilter::class,
    'role' => \App\Filters\RoleFilter::class,
];
```

Leave `$globals` alone. CSRF is commented out there by default, which is what you want while Postman is the only client — see the note in section 12.

---

## 12. Routes

Replace the contents of `app/Config/Routes.php` below the default `$routes->get('/', 'Home::index');`:

```php
$routes->group('api', ['namespace' => 'App\Controllers\Api'], static function ($routes) {

    // --- public ---
    $routes->post('auth/register', 'Auth::register');
    $routes->post('auth/login', 'Auth::login');

    $routes->get('products', 'Products::index');
    $routes->get('products/(:num)', 'Products::show/$1');
    $routes->get('categories', 'Categories::index');

    // --- authenticated ---
    $routes->post('auth/logout', 'Auth::logout', ['filter' => 'auth']);
    $routes->get('auth/me', 'Auth::me', ['filter' => 'auth']);
    $routes->post('auth/me/photo', 'Auth::photo', ['filter' => 'auth']);

    $routes->put('products/(:num)', 'Products::update/$1', ['filter' => 'auth']);
    $routes->post('products/(:num)/image', 'Products::image/$1', ['filter' => 'auth']);
    $routes->delete('products/(:num)', 'Products::delete/$1', ['filter' => 'auth']);

    // --- role restricted ---
    $routes->post('products', 'Products::create', ['filter' => 'role:ADMIN,SELLER']);
    $routes->post('categories', 'Categories::create', ['filter' => 'role:ADMIN']);

    $routes->group('users', ['filter' => 'role:ADMIN'], static function ($routes) {
        $routes->get('', 'Users::index');
        $routes->get('(:num)', 'Users::show/$1');
        $routes->put('(:num)', 'Users::update/$1');
        $routes->post('(:num)/photo', 'Users::photo/$1');
        $routes->delete('(:num)', 'Users::delete/$1');
    });
});
```

Check it:

```powershell
php spark routes
```

**The update/delete product routes use `auth`, not `role`.** Ownership can't be expressed as a URL rule — whether you may touch product 7 depends on who owns row 7. The filter confirms you're logged in; the controller checks the row.

> **On CSRF.** Session cookies are sent automatically by browsers, which is what CSRF attacks exploit. CodeIgniter leaves the `csrf` filter out of `$globals` by default, so it's off, which is fine for Postman and for a graded build. Before a browser front end calls this API, uncomment `'csrf'` in `$globals['before']` and have the client echo the token — otherwise any site your user visits can POST here with their session.

---

## 13. Authentication controller

### `app/Controllers/Api/Auth.php`

```php
<?php

namespace App\Controllers\Api;

use App\Libraries\ImageStorage;
use App\Models\RoleModel;
use App\Models\UserModel;
use CodeIgniter\HTTP\ResponseInterface;
use RuntimeException;

class Auth extends BaseApiController
{
    /** POST /api/auth/register */
    public function register(): ResponseInterface
    {
        $data = $this->jsonBody();

        $errors = $this->validateData($data, [
            'username' => [
                'rules'  => 'required|alpha_dash|min_length[3]|max_length[50]|is_unique[user.username]',
                'errors' => [
                    'required'   => 'Username is required',
                    'alpha_dash' => 'Username may only contain letters, numbers, underscore and hyphen',
                    'min_length' => 'Username must be at least 3 characters',
                    'is_unique'  => 'Username is already taken',
                ],
            ],
            'fullname' => [
                'rules'  => 'required|max_length[150]',
                'errors' => ['required' => 'Full name is required'],
            ],
            'password' => [
                'rules'  => 'required|min_length[8]',
                'errors' => [
                    'required'   => 'Password is required',
                    'min_length' => 'Password must be at least 8 characters',
                ],
            ],
            'roleID' => [
                'rules'  => 'required|is_natural_no_zero',
                'errors' => ['required' => 'Role is required'],
            ],
        ]);

        if ($errors !== null) {
            return $this->apiError(422, 'Validation failed', $errors);
        }

        if ((new RoleModel())->find((int) $data['roleID']) === null) {
            return $this->apiError(404, 'Role ' . $data['roleID'] . ' does not exist');
        }

        $users = new UserModel();

        $id = $users->insert([
            'username' => $data['username'],
            'fullname' => $data['fullname'],
            'password' => password_hash($data['password'], PASSWORD_DEFAULT),
            'roleID'   => (int) $data['roleID'],
        ], true);

        return $this->apiSuccess(201, 'New user registered', $users->findPublic((int) $id));
    }

    /** POST /api/auth/login */
    public function login(): ResponseInterface
    {
        $data = $this->jsonBody();

        $errors = $this->validateData($data, [
            'username' => ['rules' => 'required', 'errors' => ['required' => 'Username is required']],
            'password' => ['rules' => 'required', 'errors' => ['required' => 'Password is required']],
        ]);

        if ($errors !== null) {
            return $this->apiError(422, 'Validation failed', $errors);
        }

        $users = new UserModel();
        $user  = $users->findForLogin($data['username']);

        if ($user === null || ! password_verify($data['password'], $user['password'])) {
            return $this->apiError(401, 'Invalid username or password');
        }

        // New session ID on privilege change — blocks session fixation
        session()->regenerate(true);

        session()->set([
            'isLoggedIn' => true,
            'userId'     => (int) $user['id'],
            'username'   => $user['username'],
            'roleName'   => $user['roleName'],
        ]);

        return $this->apiSuccess(200, 'Login successful', $users->findPublic((int) $user['id']));
    }

    /** POST /api/auth/logout */
    public function logout(): ResponseInterface
    {
        session()->destroy();

        return $this->apiSuccess(200, 'Logout successful');
    }

    /** GET /api/auth/me */
    public function me(): ResponseInterface
    {
        $user = (new UserModel())->findPublic($this->currentUserId());

        return $this->apiSuccess(200, 'Current user', $user);
    }

    /** POST /api/auth/me/photo */
    public function photo(): ResponseInterface
    {
        $users = new UserModel();
        $id    = $this->currentUserId();
        $user  = $users->findPublic($id);

        $storage = new ImageStorage();

        try {
            $url = $storage->storeCompressed($this->request->getFile('image'));
        } catch (RuntimeException $e) {
            return $this->apiError(400, $e->getMessage());
        }

        $storage->delete($user['profilePictureUrl']);
        $users->update($id, ['profilePictureUrl' => $url]);

        return $this->apiSuccess(200, 'Profile picture updated', $users->findPublic($id));
    }
}
```

### Three things worth understanding

**`password_hash()` / `password_verify()`, never anything else.** `PASSWORD_DEFAULT` is bcrypt today and will follow PHP's recommendations as they change. Don't reach for `md5()` or `sha1()` — they're fast, which is exactly wrong for passwords. Never write your own salt handling either; `password_hash()` generates and embeds one.

**`session()->regenerate(true)` immediately after a successful login.** Without it, an attacker who can set a victim's session ID before they log in still holds a valid ID afterwards — session fixation. The `true` argument destroys the old session data rather than carrying it over. This is the single most-skipped line in PHP login tutorials.

**"Invalid username or password" covers both cases deliberately.** Saying "no such user" tells an attacker which usernames exist, which is half of a credential-stuffing attack.

Registration doesn't log you in. Keeping the two separate means a registration failure can't leave a half-authenticated session behind.

---

## 14. Product CRUD

### `app/Controllers/Api/Products.php`

```php
<?php

namespace App\Controllers\Api;

use App\Libraries\ImageStorage;
use App\Models\CategoryModel;
use App\Models\ProductModel;
use CodeIgniter\HTTP\ResponseInterface;
use RuntimeException;

class Products extends BaseApiController
{
    /** GET /api/products?search=&categoryID=&page=&perPage= */
    public function index(): ResponseInterface
    {
        $perPage    = (int) ($this->request->getGet('perPage') ?: 10);
        $search     = $this->request->getGet('search');
        $categoryID = $this->request->getGet('categoryID');

        $products = new ProductModel();
        $items    = $products->pageJoined($search, $categoryID ? (int) $categoryID : null, $perPage);
        $pager    = $products->pager;

        return $this->apiSuccess(200, 'Products retrieved', [
            'items'      => $items,
            'page'       => $pager->getCurrentPage(),
            'perPage'    => $perPage,
            'totalItems' => $pager->getTotal(),
            'totalPages' => $pager->getPageCount(),
        ]);
    }

    /** GET /api/products/{id} */
    public function show(int $id): ResponseInterface
    {
        $product = (new ProductModel())->findJoined($id);

        if ($product === null) {
            return $this->apiError(404, 'No product found with ID ' . $id);
        }

        return $this->apiSuccess(200, 'Product retrieved', $product);
    }

    /** POST /api/products — multipart: name, categoryID, image */
    public function create(): ResponseInterface
    {
        $data = [
            'name'       => $this->request->getPost('name'),
            'categoryID' => $this->request->getPost('categoryID'),
        ];

        $errors = $this->validateData($data, [
            'name' => [
                'rules'  => 'required|max_length[200]',
                'errors' => ['required' => 'Name is required'],
            ],
            'categoryID' => [
                'rules'  => 'required|is_natural_no_zero',
                'errors' => ['required' => 'Category is required'],
            ],
        ]);

        if ($errors !== null) {
            return $this->apiError(422, 'Validation failed', $errors);
        }

        if ((new CategoryModel())->find((int) $data['categoryID']) === null) {
            return $this->apiError(404, 'Category ' . $data['categoryID'] . ' does not exist');
        }

        $imageUrl = null;
        $file     = $this->request->getFile('image');

        if ($file !== null && $file->isValid()) {
            try {
                $imageUrl = (new ImageStorage())->storeCompressed($file);
            } catch (RuntimeException $e) {
                return $this->apiError(400, $e->getMessage());
            }
        }

        $products = new ProductModel();

        $id = $products->insert([
            'name'       => $data['name'],
            'categoryID' => (int) $data['categoryID'],
            'imageUrl'   => $imageUrl,
            'sellerID'   => $this->currentUserId(),   // from the session, never the request
        ], true);

        return $this->apiSuccess(201, 'New product created', $products->findJoined((int) $id));
    }

    /** PUT /api/products/{id} — JSON body */
    public function update(int $id): ResponseInterface
    {
        $products = new ProductModel();
        $product  = $products->findJoined($id);

        if ($product === null) {
            return $this->apiError(404, 'No product found with ID ' . $id);
        }

        if (! $this->canModify($product)) {
            return $this->apiError(403, 'You can only modify your own products');
        }

        $data = $this->jsonBody();

        $errors = $this->validateData($data, [
            'name' => [
                'rules'  => 'required|max_length[200]',
                'errors' => ['required' => 'Name is required'],
            ],
            'categoryID' => [
                'rules'  => 'required|is_natural_no_zero',
                'errors' => ['required' => 'Category is required'],
            ],
        ]);

        if ($errors !== null) {
            return $this->apiError(422, 'Validation failed', $errors);
        }

        if ((new CategoryModel())->find((int) $data['categoryID']) === null) {
            return $this->apiError(404, 'Category ' . $data['categoryID'] . ' does not exist');
        }

        $products->update($id, [
            'name'       => $data['name'],
            'categoryID' => (int) $data['categoryID'],
        ]);

        return $this->apiSuccess(200, 'Product updated', $products->findJoined($id));
    }

    /** POST /api/products/{id}/image — multipart */
    public function image(int $id): ResponseInterface
    {
        $products = new ProductModel();
        $product  = $products->findJoined($id);

        if ($product === null) {
            return $this->apiError(404, 'No product found with ID ' . $id);
        }

        if (! $this->canModify($product)) {
            return $this->apiError(403, 'You can only modify your own products');
        }

        $storage = new ImageStorage();

        try {
            $url = $storage->storeCompressed($this->request->getFile('image'));
        } catch (RuntimeException $e) {
            return $this->apiError(400, $e->getMessage());
        }

        $storage->delete($product['imageUrl']);
        $products->update($id, ['imageUrl' => $url]);

        return $this->apiSuccess(200, 'Product image updated', $products->findJoined($id));
    }

    /** DELETE /api/products/{id} */
    public function delete(int $id): ResponseInterface
    {
        $products = new ProductModel();
        $product  = $products->findJoined($id);

        if ($product === null) {
            return $this->apiError(404, 'No product found with ID ' . $id);
        }

        if (! $this->canModify($product)) {
            return $this->apiError(403, 'You can only modify your own products');
        }

        (new ImageStorage())->delete($product['imageUrl']);
        $products->delete($id);

        return $this->apiSuccess(200, 'Product deleted');
    }

    /** A seller may only touch their own rows; an admin may touch any. */
    private function canModify(array $product): bool
    {
        return $this->isAdmin() || (int) $product['sellerID'] === $this->currentUserId();
    }
}
```

### Why create is multipart but update is JSON

**PHP does not populate `$_POST` or `$_FILES` for PUT requests** — only for POST. A `PUT` carrying `multipart/form-data` arrives as an unparsed raw body, and every field reads as null. So field updates go over PUT as JSON, and image replacement gets its own `POST /{id}/image`. That also gives the client an endpoint to call when only the photo changed.

**`sellerID` comes from the session.** If the client supplied it, any seller could publish products under another seller's name. `$this->currentUserId()` is the only trustworthy source.

---

## 15. User CRUD

### `app/Controllers/Api/Users.php`

The whole group is behind `role:ADMIN` in the routes, so there's no per-method check.

```php
<?php

namespace App\Controllers\Api;

use App\Libraries\ImageStorage;
use App\Models\RoleModel;
use App\Models\UserModel;
use CodeIgniter\HTTP\ResponseInterface;
use RuntimeException;

class Users extends BaseApiController
{
    /** GET /api/users */
    public function index(): ResponseInterface
    {
        $perPage = (int) ($this->request->getGet('perPage') ?: 10);

        $users = new UserModel();
        $items = $users->pagePublic($perPage);
        $pager = $users->pager;

        return $this->apiSuccess(200, 'Users retrieved', [
            'items'      => $items,
            'page'       => $pager->getCurrentPage(),
            'perPage'    => $perPage,
            'totalItems' => $pager->getTotal(),
            'totalPages' => $pager->getPageCount(),
        ]);
    }

    /** GET /api/users/{id} */
    public function show(int $id): ResponseInterface
    {
        $user = (new UserModel())->findPublic($id);

        if ($user === null) {
            return $this->apiError(404, 'No user found with ID ' . $id);
        }

        return $this->apiSuccess(200, 'User retrieved', $user);
    }

    /** PUT /api/users/{id} — fullname and role only */
    public function update(int $id): ResponseInterface
    {
        $users = new UserModel();

        if ($users->findPublic($id) === null) {
            return $this->apiError(404, 'No user found with ID ' . $id);
        }

        $data = $this->jsonBody();

        $errors = $this->validateData($data, [
            'fullname' => [
                'rules'  => 'required|max_length[150]',
                'errors' => ['required' => 'Full name is required'],
            ],
            'roleID' => [
                'rules'  => 'required|is_natural_no_zero',
                'errors' => ['required' => 'Role is required'],
            ],
        ]);

        if ($errors !== null) {
            return $this->apiError(422, 'Validation failed', $errors);
        }

        if ((new RoleModel())->find((int) $data['roleID']) === null) {
            return $this->apiError(404, 'Role ' . $data['roleID'] . ' does not exist');
        }

        $users->update($id, [
            'fullname' => $data['fullname'],
            'roleID'   => (int) $data['roleID'],
        ]);

        return $this->apiSuccess(200, 'User updated', $users->findPublic($id));
    }

    /** POST /api/users/{id}/photo */
    public function photo(int $id): ResponseInterface
    {
        $users = new UserModel();
        $user  = $users->findPublic($id);

        if ($user === null) {
            return $this->apiError(404, 'No user found with ID ' . $id);
        }

        $storage = new ImageStorage();

        try {
            $url = $storage->storeCompressed($this->request->getFile('image'));
        } catch (RuntimeException $e) {
            return $this->apiError(400, $e->getMessage());
        }

        $storage->delete($user['profilePictureUrl']);
        $users->update($id, ['profilePictureUrl' => $url]);

        return $this->apiSuccess(200, 'Profile picture updated', $users->findPublic($id));
    }

    /** DELETE /api/users/{id} */
    public function delete(int $id): ResponseInterface
    {
        $users = new UserModel();
        $user  = $users->findPublic($id);

        if ($user === null) {
            return $this->apiError(404, 'No user found with ID ' . $id);
        }

        if ((int) $user['id'] === $this->currentUserId()) {
            return $this->apiError(400, 'You cannot delete your own account while logged in');
        }

        (new ImageStorage())->delete($user['profilePictureUrl']);
        $users->delete($id);

        return $this->apiSuccess(200, 'User deleted');
    }
}
```

**There's no `password` field on the update endpoint, deliberately.** Password changes need their own endpoint that requires the current password. Folding them into a general update means an admin — or an XSS payload holding an admin session — can silently take over any account.

**The self-delete guard** stops an admin deleting the account they're logged in as, which would leave a live session pointing at a row that no longer exists.

Deleting a user who owns products fails on the foreign key. That's `RESTRICT` doing its job; decide deliberately whether to reassign or cascade.

---

## 16. Categories

### `app/Controllers/Api/Categories.php`

```php
<?php

namespace App\Controllers\Api;

use App\Models\CategoryModel;
use CodeIgniter\HTTP\ResponseInterface;

class Categories extends BaseApiController
{
    /** GET /api/categories — public */
    public function index(): ResponseInterface
    {
        $items = (new CategoryModel())->orderBy('name', 'ASC')->findAll();

        return $this->apiSuccess(200, 'Categories retrieved', $items);
    }

    /** POST /api/categories — admin only */
    public function create(): ResponseInterface
    {
        $data = $this->jsonBody();

        $errors = $this->validateData($data, [
            'name' => [
                'rules'  => 'required|max_length[100]|is_unique[category.name]',
                'errors' => [
                    'required'  => 'Name is required',
                    'is_unique' => 'That category already exists',
                ],
            ],
        ]);

        if ($errors !== null) {
            return $this->apiError(422, 'Validation failed', $errors);
        }

        $categories = new CategoryModel();
        $id         = $categories->insert(['name' => $data['name']], true);

        return $this->apiSuccess(201, 'New category created', $categories->find((int) $id));
    }
}
```

---

## 17. Seed the reference data

Nobody can register until roles exist, and there's no admin to create the first one.

```powershell
php spark make:seeder ShopNowSeeder
```

### `app/Database/Seeds/ShopNowSeeder.php`

```php
<?php

namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;

class ShopNowSeeder extends Seeder
{
    public function run()
    {
        foreach (['ADMIN', 'SELLER', 'BUYER'] as $name) {
            if ($this->db->table('role')->where('name', $name)->countAllResults() === 0) {
                $this->db->table('role')->insert(['name' => $name]);
            }
        }

        foreach (['Electronics', 'Fashion', 'Home & Living', 'Groceries', 'Books'] as $name) {
            if ($this->db->table('category')->where('name', $name)->countAllResults() === 0) {
                $this->db->table('category')->insert(['name' => $name]);
            }
        }

        if ($this->db->table('user')->where('username', 'admin')->countAllResults() === 0) {
            $adminRole = $this->db->table('role')->where('name', 'ADMIN')->get()->getRowArray();

            $this->db->table('user')->insert([
                'username' => 'admin',
                'fullname' => 'System Administrator',
                'password' => password_hash('admin12345', PASSWORD_DEFAULT),
                'roleID'   => $adminRole['id'],
            ]);

            echo ">>> Seeded admin account: admin / admin12345\n";
        }
    }
}
```

```powershell
php spark db:seed ShopNowSeeder
```

Every block checks before inserting, so re-running is safe.

**The seeded password is a development convenience.** Change it before this is reachable by anyone else.

---

## 18. Run and test in Postman

```powershell
php spark serve
```

`http://localhost:8080`. MySQL must be running in XAMPP.

### 18.1 Collection and environment

**Collection:** `ShopNow CI`. **Environment** `ShopNow CI Local` with `base_url` = `http://localhost:8080/api`, selected in the top-right dropdown. Add `Accept: application/json` as a collection header.

> **Cookies are automatic.** Postman stores the `ci_session` cookie from login and sends it on later requests to the same host. No Authorization header anywhere. Inspect or clear it via the **Cookies** link under Send.

### 18.2 Register — POST {{base_url}}/auth/register

Body → **raw** → **JSON**:

```json
{
  "username": "sarah",
  "fullname": "Sarah Dev",
  "password": "secret12345",
  "roleID": 2
}
```

**201:**

```json
{
  "code": 201,
  "message": "New user registered",
  "data": {
    "id": 2,
    "username": "sarah",
    "fullname": "Sarah Dev",
    "roleID": 2,
    "profilePictureUrl": null,
    "roleName": "SELLER"
  },
  "timestamp": "2026-09-17T12:04:11+08:00"
}
```

Check phpMyAdmin — `password` holds a `$2y$10$...` hash, not `secret12345`.

Send again → **422** with `"username": "Username is already taken"` (the `is_unique` rule catches it before the database does).

Send `{"username":"ab","password":"123"}` → **422** with four field messages.

### 18.3 Login — POST {{base_url}}/auth/login

```json
{ "username": "sarah", "password": "secret12345" }
```

**200**, `"message": "Login successful"`. Open **Cookies** under Send — `ci_session` is there.

Wrong password → **401** "Invalid username or password".

### 18.4 GET {{base_url}}/auth/me

**200** with your user. A 401 here right after a successful login means the cookie isn't being sent — check that `base_url` uses the same host you logged in against (`localhost` and `127.0.0.1` are different cookie jars).

### 18.5 Create a product — POST {{base_url}}/products

Body → **form-data**, not raw:

| Key | Type | Value |
|---|---|---|
| `name` | Text | `Mechanical Keyboard` |
| `categoryID` | Text | `1` |
| `image` | **File** | pick a large photo |

Switch a row's type from Text to File with the dropdown that appears when you hover the Key cell.

**201** with `"imageUrl": "/uploads/6f2a....jpg"`.

Now verify the compression — the part worth seeing:

```powershell
dir public\uploads
```

A 4 MB source photo lands around 150–250 KB. Open <http://localhost:8080/uploads/6f2a....jpg> in a browser to confirm it's valid and correctly oriented.

### 18.6 The rest

| Request | Method | URL | Body |
|---|---|---|---|
| List | GET | `{{base_url}}/products?page=1&perPage=5` | — |
| Search | GET | `{{base_url}}/products?search=keyboard` | — |
| One | GET | `{{base_url}}/products/1` | — |
| Update | PUT | `{{base_url}}/products/1` | raw JSON: `{"name":"Keyboard MK2","categoryID":1}` |
| Replace image | POST | `{{base_url}}/products/1/image` | form-data: `image` (File) |
| Delete | DELETE | `{{base_url}}/products/1` | — |

Pages here are **one-indexed** — `page=1` is the first. (Spring's are zero-indexed; easy to trip over if you're switching between the two.)

### 18.7 Test the authorization rules

A passing happy path proves nothing about access control.

1. **Logged out:** POST `{{base_url}}/auth/logout`, then `GET {{base_url}}/auth/me` → **401**. `GET {{base_url}}/products` still works — it's public.
2. **Wrong role:** logged in as `sarah` (SELLER), `GET {{base_url}}/users` → **403**.
3. **Admin:** log in as `admin` / `admin12345`, `GET {{base_url}}/users` → **200**.
4. **Ownership:** register a second seller, log in as them, `DELETE {{base_url}}/products/1` (sarah's) → **403** "You can only modify your own products". As `admin`, the same call succeeds.

### 18.8 Save it as a regression suite

In each request's **Scripts** tab:

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

## 19. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Class "IntlDateFormatter" not found` on boot | `intl` disabled | Uncomment `extension=intl` in `C:\xampp\php\php.ini` |
| `Call to undefined function imagecreatefromjpeg()` | **`gd` disabled** | Uncomment `extension=gd`, new terminal (§2) |
| Login works, `/auth/me` returns 401 | Cookie not sent — different host in `base_url` | Use one of `localhost` or `127.0.0.1` consistently |
| Session lost on every request | `.env` session lines still commented out, or `writable/` not writable | Remove the `#`; check `writable/session` exists |
| 403 on an endpoint the role should reach | `roleName` in session doesn't match the filter argument | Seeder stores `ADMIN`; filter expects `ADMIN` — same case |
| All fields null on product create | Sent as raw JSON instead of form-data | Create is **multipart** |
| All fields null on a PUT with form-data | PHP never populates `$_POST`/`$_FILES` for PUT | Use JSON for PUT; images go to `POST /{id}/image` |
| Image field ignored | Key type left as Text | Switch the row to **File** in Postman |
| `Only JPG, PNG, WEBP or GIF images are allowed` for a real photo | `gd` missing, or file over `upload_max_filesize` | Check `gd`; raise `upload_max_filesize` and `post_max_size` in `php.ini` |
| Upload silently fails over ~2 MB | PHP's default `upload_max_filesize = 2M` | Raise both it and `post_max_size`, then restart |
| `mkdir(): Permission denied` | `public/uploads` not creatable | Create it by hand once |
| Transparent PNG turns black | JPEG has no alpha channel | Branch on `$info[2]` instead of forcing `.jpg` |
| `Cannot delete or update a parent row` | FK `RESTRICT` protecting referenced rows | Delete or reassign the products first |
| Validation passes with an empty body | Using `$this->validate()` on a JSON body | Use `validateData()` from §9 |
| `Unknown database 'ShopNowDB'` | Schema not created | Run the `CREATE DATABASE` in §4 |
| `Unable to connect to the database` | MySQL not started | Start it in the XAMPP Control Panel |
| Generic error page, no detail | `CI_ENVIRONMENT = production` | Set `development` in `.env` |
| 404 on every `/api/...` | Routes not registered | `php spark routes` |
| `Port 8080 already in use` | Something else has it | `php spark serve --port 8081` |

Logs are in `writable/logs/`.

---

## 20. CodeIgniter vs Spring Boot on this build

Both guides build the identical API. Where they differ:

| Concern | CodeIgniter 4 | Spring Boot 4 |
|---|---|---|
| Runtime | XAMPP's PHP 8.2 as-is | Separate JDK 21 install |
| Dependencies added | none | Thumbnailator + springdoc |
| Image compression | built in (`service('image')`, GD) | Thumbnailator library |
| Serving uploads | `public/uploads` is already web-root | `WebMvcConfigurer` resource handler |
| Auth framework | you write two filters (~40 lines) | Spring Security filter chain |
| Password hashing | `password_hash()` / `password_verify()` | `BCryptPasswordEncoder` bean |
| Session persistence | automatic | explicit `securityContextRepository.saveContext()` |
| Role check | `'filter' => 'role:ADMIN'` on the route | `.hasRole("ADMIN")` + `@PreAuthorize` |
| Validation | model/service rules, string syntax | annotations on a DTO record |
| Error envelope | base controller helpers | `@RestControllerAdvice` |
| Hiding the password hash | separate `SELECT` lists | response DTO |
| Schema | migrations | `ddl-auto=update` |
| Pagination index | page **1** | page **0** |
| Startup | instant | a few seconds |

CodeIgniter asks less of the setup and gives less automatic behaviour — you hand-write the auth filter, the envelope helpers, and the field lists that keep the password hash out of responses. Spring hands you those and charges you configuration and startup time for them. For a small marketplace either is a reasonable answer; the CodeIgniter version is the one you can run on shared hosting.

---

## 21. Where to go next

1. **Add a password-change endpoint** — `POST /api/auth/change-password` taking the current and new password, verified with `password_verify()`. Deliberately absent from the user update endpoint.
2. **Turn CSRF back on** before any browser front end touches this (§12).
3. **Rate-limit login.** CodeIgniter has a throttler: `service('throttler')->check($ip, 5, MINUTE)` in the login method. An unthrottled login endpoint is an invitation to credential stuffing.
4. **Move sessions to the database** — `session.driver = 'CodeIgniter\Session\Handlers\DatabaseHandler'` plus the sessions table — so they survive across more than one server.
5. **Add `created_at` / `updated_at`** to both tables and set `$useTimestamps = true` on the models. You'll want them the first time you debug a data question.
6. **Add soft deletes** for products — `$useSoftDeletes = true` plus a nullable `deleted_at` column — so order history keeps working.
7. **Add Shield** (`composer require codeigniter4/shield`) if auth grows beyond this: it brings password reset, email verification, and API tokens as a maintained package rather than more hand-rolled code.
8. **Write tests** — CodeIgniter bundles PHPUnit with `FeatureTestTrait`: `$this->call('get', 'api/products')->assertStatus(200)`.
9. **Generate API docs.** CodeIgniter has no first-class OpenAPI support the way Spring does; the practical route is to export your Postman collection, or hand-write an `openapi.yaml` and serve Swagger UI as static files from `public/`.
