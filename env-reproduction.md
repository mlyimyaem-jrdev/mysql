# Building a Simple CRUD API in CodeIgniter 4

The same product API as the Laravel and Spring Boot guides — built with CodeIgniter 4, XAMPP's own PHP and MySQL, tested in Postman.

**Target version:** CodeIgniter 4.7.x
**PHP:** 8.2 minimum — **XAMPP's bundled PHP is enough**
**Platform:** Windows 10 / 11, PowerShell
**Database:** MySQL/MariaDB from XAMPP
**No Docker, no separate PHP install, no web server config**

> **Why this one is the lightest setup of the three.** Laravel 13 needs PHP 8.3+, which forced a standalone PHP install because XAMPP ships 8.2.12. CodeIgniter 4.7 runs on 8.2, so XAMPP's own PHP works as-is. Install XAMPP, add one folder to your PATH, done.

---

## Table of Contents

1. [What you'll build](#1-what-youll-build)
2. [Prerequisites and downloads](#2-prerequisites-and-downloads)
3. [Create the project](#3-create-the-project)
4. [Project structure](#4-project-structure)
5. [Create the database](#5-create-the-database)
6. [Configure the .env file](#6-configure-the-env-file)
7. [Write the migration](#7-write-the-migration)
8. [Write the model](#8-write-the-model)
9. [Write the controller](#9-write-the-controller)
10. [Register the routes](#10-register-the-routes)
11. [Run the application](#11-run-the-application)
12. [Test in Postman](#12-test-in-postman)
13. [Seed sample data (optional)](#13-seed-sample-data-optional)
14. [Troubleshooting](#14-troubleshooting)
15. [Command cheat sheet](#15-command-cheat-sheet)
16. [All three frameworks, side by side](#16-all-three-frameworks-side-by-side)
17. [Where to go next](#17-where-to-go-next)

---

## 1. What you'll build

| Method | URI | Controller method | Success status |
|---|---|---|---|
| GET | `/api/products` | `index` | 200 |
| POST | `/api/products` | `create` | 201 |
| GET | `/api/products/{id}` | `show` | 200 |
| PUT/PATCH | `/api/products/{id}` | `update` | 200 |
| DELETE | `/api/products/{id}` | `delete` | 204 |

Three files written by hand: a migration, a model, and a controller. Plus two lines in the routes file.

Note the method names — CodeIgniter's `ResourceController` expects `create` and `delete`, not Laravel's `store` and `destroy`. Rename them and the routes stop resolving.

---

## 2. Prerequisites and downloads

### 2.1 XAMPP

Download from <https://www.apachefriends.org/download.html> and install to `C:\xampp`.

Open the **XAMPP Control Panel** and **Start** MySQL. Apache is optional — CodeIgniter's `spark serve` runs its own development server, so this project does **not** go in `htdocs`. Start Apache only if you want phpMyAdmin at <http://localhost/phpmyadmin>.

XAMPP's "MySQL" is actually MariaDB; CodeIgniter's `MySQLi` driver handles it without changes. The default account is `root` with an **empty password**.

### 2.2 Put PHP on your PATH

XAMPP doesn't add its PHP to the PATH, so the `php` command won't work until you do. Check:

```powershell
php -v
```

**If that prints 8.2 or higher, you're done** — skip to 2.3. (If you followed the Laravel guide and have `C:\php` with 8.3/8.4 on your PATH, that works here too. CodeIgniter 4.7 supports PHP 8.2 through 8.5.)

**If `php` is not recognized:**

1. Press `Win`, search "Edit the system environment variables" → **Environment Variables**
2. Under **User variables**, select `Path` → **Edit** → **New**
3. Add `C:\xampp\php`
4. **OK**, then open a **new** terminal and run `php -v` again

#### Check the required extensions

CodeIgniter 4 needs `intl`, `mbstring`, and `json`, plus `mysqli` for the database:

```powershell
php -m | Select-String "intl|mbstring|json|mysqli|curl"
```

If **`intl`** is missing — it's the one XAMPP sometimes ships disabled — open `C:\xampp\php\php.ini` in Notepad, find this line:

```ini
;extension=intl
```

Delete the leading `;` so it reads `extension=intl`, save, and open a new terminal. CodeIgniter refuses to boot without it.

### 2.3 Composer

```powershell
composer -V
```

If it's missing, download and run `Composer-Setup.exe` from <https://getcomposer.org/download/>. Point it at `C:\xampp\php\php.exe` when it asks (or `C:\php\php.exe` if you have the standalone install).

> **Prefer to skip Composer entirely?** You can download the framework as a ZIP from <https://codeigniter.com/download> and extract it. You'll then update the framework by hand instead of running `composer update`, which is why Composer is the recommended route.

### 2.4 Postman

Download the Windows 64-bit build from <https://www.postman.com/downloads/>. "Continue without an account" at the bottom of the first screen skips the sign-in.

### 2.5 Verification

```powershell
php -v       # 8.2 or higher
composer -V  # Composer 2.x
```

Plus MySQL green in the XAMPP Control Panel.

---

## 3. Create the project

```powershell
cd C:\dev
composer create-project codeigniter4/appstarter product-api-ci
cd product-api-ci
```

`appstarter` is the project skeleton — it pulls in `codeigniter4/framework` as a dependency, which keeps the framework itself upgradable through Composer.

Verify the CLI works:

```powershell
php spark
```

You should get the CodeIgniter banner and a list of commands. If you see a fatal error about `intl`, revisit section 2.2.

Pick a folder without spaces or OneDrive sync in its path. OneDrive's file locking causes intermittent write errors in the `writable/` directory.

---

## 4. Project structure

```
product-api-ci/
├── app/
│   ├── Config/
│   │   ├── Routes.php          ← you edit (§10)
│   │   └── Database.php        ← leave alone; .env overrides it
│   ├── Controllers/
│   │   └── Products.php        ← you write (§9)
│   ├── Database/
│   │   ├── Migrations/         ← you write (§7)
│   │   └── Seeds/              ← optional (§13)
│   ├── Models/
│   │   └── ProductModel.php    ← you write (§8)
│   └── Filters/
├── public/
│   └── index.php               ← the real entry point
├── writable/                   ← logs, cache, sessions
├── env                         ← template; you copy this to .env (§6)
└── spark                       ← CLI tool
```

Two things that trip people up coming from CodeIgniter 3:

- **`index.php` lives in `public/`**, not the project root. That's why you don't drop this folder into `htdocs` — the document root has to be `public/`, and `spark serve` handles that for you.
- **The config file to edit is `.env`**, not `app/Config/Database.php`. Values in `.env` override the PHP config classes, and `.env` is git-ignored so credentials stay out of version control.

---

## 5. Create the database

CodeIgniter's migration creates the *table*, but not the *database*. With MySQL running in XAMPP:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api_ci CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

No `-p` — the XAMPP root account has no password by default.

**Or in phpMyAdmin:** start Apache too, open <http://localhost/phpmyadmin>, click **New**, enter `product_api_ci`, collation `utf8mb4_unicode_ci`, **Create**.

The name differs from the other guides on purpose (`product_api`, `product_api_ci`) so all three projects can coexist on one machine.

---

## 6. Configure the .env file

The skeleton ships a file literally named **`env`** — no dot. Copy it:

```powershell
copy env .env
```

PowerShell won't let you rename to `.env` in Explorer easily, so use the command. Then open `.env` in your editor and change these lines — **every line in that file starts commented out with `#`, so you must delete the `#` as well as set the value**:

```env
CI_ENVIRONMENT = development

app.baseURL = 'http://localhost:8080/'

database.default.hostname = 127.0.0.1
database.default.database = product_api_ci
database.default.username = root
database.default.password =
database.default.DBDriver = MySQLi
database.default.port = 3306
```

Why each matters:

- **`CI_ENVIRONMENT = development`** turns on detailed error pages. Leave it at `production` and every mistake returns a generic "something went wrong" with no stack trace — the single biggest time-waster when starting out.
- **`127.0.0.1` not `localhost`** — on Windows, `localhost` often resolves to IPv6 `::1`, which MariaDB isn't listening on. The error that produces looks like the database is down when it isn't.
- **`database.default.password =`** is deliberately empty. Nothing after the `=`.
- **`app.baseURL`** must match how you access the app, including the trailing slash.

---

## 7. Write the migration

Generate the file:

```powershell
php spark make:migration CreateProducts
```

That creates `app/Database/Migrations/2026-09-12-083104_CreateProducts.php` (your timestamp will differ). Open it and replace the contents:

```php
<?php

namespace App\Database\Migrations;

use CodeIgniter\Database\Migration;

class CreateProducts extends Migration
{
    public function up(): void
    {
        $this->forge->addField([
            'id' => [
                'type'           => 'INT',
                'constraint'     => 11,
                'unsigned'       => true,
                'auto_increment' => true,
            ],
            'name' => [
                'type'       => 'VARCHAR',
                'constraint' => 255,
            ],
            'sku' => [
                'type'       => 'VARCHAR',
                'constraint' => 64,
            ],
            'description' => [
                'type' => 'TEXT',
                'null' => true,
            ],
            'price' => [
                'type'       => 'DECIMAL',
                'constraint' => '10,2',
            ],
            'stock' => [
                'type'       => 'INT',
                'constraint' => 11,
                'unsigned'   => true,
                'default'    => 0,
            ],
            'is_active' => [
                'type'       => 'TINYINT',
                'constraint' => 1,
                'default'    => 1,
            ],
            'created_at' => [
                'type' => 'DATETIME',
                'null' => true,
            ],
            'updated_at' => [
                'type' => 'DATETIME',
                'null' => true,
            ],
        ]);

        $this->forge->addKey('id', true);        // primary key
        $this->forge->addUniqueKey('sku');
        $this->forge->addKey('is_active');
        $this->forge->createTable('products');
    }

    public function down(): void
    {
        $this->forge->dropTable('products');
    }
}
```

Notes:

- **`DECIMAL(10,2)` for money.** Never `FLOAT` — binary floating point can't represent `0.10` exactly and totals drift by cents.
- **`created_at` / `updated_at` must be nullable.** The model fills them automatically, but they're empty at insert time before the model writes them.
- **`down()`** is the rollback. Fill it in so `php spark migrate:rollback` works.

Run it:

```powershell
php spark migrate
```

```
Running all new migrations...
	Running: (App) 2026-09-12-083104_App\Database\Migrations\CreateProducts
Migrations complete.
```

Check phpMyAdmin — `products` now exists inside `product_api_ci`.

---

## 8. Write the model

```powershell
php spark make:model ProductModel
```

Open `app/Models/ProductModel.php` and replace it:

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class ProductModel extends Model
{
    protected $table      = 'products';
    protected $primaryKey = 'id';
    protected $returnType = 'array';

    protected $allowedFields = [
        'name',
        'sku',
        'description',
        'price',
        'stock',
        'is_active',
    ];

    // Maintain created_at / updated_at automatically
    protected $useTimestamps = true;
    protected $dateFormat    = 'datetime';
    protected $createdField  = 'created_at';
    protected $updatedField  = 'updated_at';

    protected $validationRules = [
        'name'        => 'required|string|max_length[255]',
        'sku'         => 'required|string|max_length[64]|is_unique[products.sku,id,{id}]',
        'description' => 'permit_empty|string|max_length[2000]',
        'price'       => 'required|decimal|greater_than_equal_to[0]',
        'stock'       => 'permit_empty|is_natural',
        'is_active'   => 'permit_empty|in_list[0,1]',
    ];

    protected $validationMessages = [
        'sku' => [
            'is_unique' => 'That SKU is already in use.',
        ],
        'price' => [
            'greater_than_equal_to' => 'Price cannot be negative.',
        ],
    ];

    protected $skipValidation = false;
}
```

Three mechanisms doing real work here:

**`$allowedFields`** is mass-assignment protection — the exact equivalent of Laravel's `$fillable`. Anything not listed is silently dropped from inserts and updates, so a crafted request body can't set columns you never intended to expose. A field missing from this array is also the most common cause of "my update ran but nothing changed".

**`is_unique[products.sku,id,{id}]`** enforces a unique SKU while ignoring the row being edited. The `{id}` placeholder is replaced with the `id` value **from the data being validated** — which is why the controller's update method has to merge the ID into the payload. Without that, updating a product without changing its SKU fails, because the record collides with itself.

**`$cleanValidationRules`** defaults to `true` in CodeIgniter 4, meaning rules for fields absent from an update payload are skipped. That's what makes partial updates work without a second set of rules — Laravel needs a separate `UpdateProductRequest` with `sometimes` for the same behaviour.

---

## 9. Write the controller

```powershell
php spark make:controller Products --restful
```

The `--restful` flag generates a `ResourceController` with the five method stubs already in place. Open `app/Controllers/Products.php` and replace it:

```php
<?php

namespace App\Controllers;

use App\Models\ProductModel;
use CodeIgniter\RESTful\ResourceController;

class Products extends ResourceController
{
    protected $modelName = ProductModel::class;
    protected $format    = 'json';

    /**
     * GET /api/products
     */
    public function index()
    {
        $perPage = (int) ($this->request->getGet('per_page') ?: 15);
        $search  = $this->request->getGet('search');
        $active  = $this->request->getGet('active');

        $query = $this->model;

        if ($search !== null && $search !== '') {
            $query = $query->groupStart()
                           ->like('name', $search)
                           ->orLike('sku', $search)
                           ->groupEnd();
        }

        if ($active !== null && $active !== '') {
            $query = $query->where('is_active', $active === 'true' ? 1 : 0);
        }

        $products = $query->orderBy('id', 'DESC')->paginate($perPage);
        $pager    = $this->model->pager;

        return $this->respond([
            'data' => $products,
            'meta' => [
                'current_page' => $pager->getCurrentPage(),
                'per_page'     => $perPage,
                'last_page'    => $pager->getPageCount(),
                'total'        => $pager->getTotal(),
            ],
        ]);
    }

    /**
     * POST /api/products
     */
    public function create()
    {
        $data = $this->request->getJSON(true) ?? [];

        if (! $this->model->insert($data)) {
            return $this->failValidationErrors($this->model->errors());
        }

        $product = $this->model->find($this->model->getInsertID());

        return $this->respondCreated(['data' => $product]);
    }

    /**
     * GET /api/products/{id}
     */
    public function show($id = null)
    {
        $product = $this->model->find($id);

        if ($product === null) {
            return $this->failNotFound("No product found with ID {$id}");
        }

        return $this->respond(['data' => $product]);
    }

    /**
     * PUT|PATCH /api/products/{id}
     */
    public function update($id = null)
    {
        if ($this->model->find($id) === null) {
            return $this->failNotFound("No product found with ID {$id}");
        }

        $data = $this->request->getJSON(true) ?? [];
        $data['id'] = $id;   // required by the is_unique[...,{id}] placeholder

        if (! $this->model->update($id, $data)) {
            return $this->failValidationErrors($this->model->errors());
        }

        return $this->respond(['data' => $this->model->find($id)]);
    }

    /**
     * DELETE /api/products/{id}
     */
    public function delete($id = null)
    {
        if ($this->model->find($id) === null) {
            return $this->failNotFound("No product found with ID {$id}");
        }

        $this->model->delete($id);

        return $this->respondNoContent();
    }
}
```

### What's doing the work

**`$modelName` and `$format`** are all the configuration `ResourceController` needs. It instantiates the model into `$this->model` for you and serialises every response as JSON.

**`$this->request->getJSON(true)`** is the line to remember. CodeIgniter's `getPost()` reads form-encoded bodies only — it returns nothing for a raw JSON body. If you use `getPost()` here, every field comes back as "required" no matter what you send, and the cause is invisible. The `true` argument returns an associative array instead of an object.

**The response helpers** come from `ResponseTrait` and set the status code for you:

| Helper | Status |
|---|---|
| `respond($data)` | 200 |
| `respondCreated($data)` | 201 |
| `respondNoContent()` | 204 |
| `failValidationErrors($errors)` | 400 |
| `failNotFound($message)` | 404 |

**Validation lives in the model, not the controller.** `insert()` and `update()` return `false` when it fails, and `$this->model->errors()` holds the field-by-field messages. That's a structural difference from Laravel, where validation sits in a Form Request class, and from Spring, where it's annotations on a DTO.

---

## 10. Register the routes

Open `app/Config/Routes.php`. Below the existing `$routes->get('/', 'Home::index');` line, add:

```php
$routes->group('api', static function ($routes) {
    $routes->resource('products', ['except' => 'new,edit']);
});
```

`resource()` generates all five routes with correct verbs and URIs. The `except` is important: without it CodeIgniter also creates `products/new` and `products/{id}/edit`, which exist to return HTML forms and are meaningless in an API.

Verify:

```powershell
php spark routes
```

```
+--------+-------------------------+------------------------------------+
| Method | Route                   | Handler                            |
+--------+-------------------------+------------------------------------+
| GET    | api/products            | \App\Controllers\Products::index   |
| GET    | api/products/(.*)       | \App\Controllers\Products::show/$1 |
| POST   | api/products            | \App\Controllers\Products::create  |
| PUT    | api/products/(.*)       | \App\Controllers\Products::update  |
| PATCH  | api/products/(.*)       | \App\Controllers\Products::update  |
| DELETE | api/products/(.*)       | \App\Controllers\Products::delete  |
+--------+-------------------------+------------------------------------+
```

If the handlers say something other than `Products`, the controller filename or class name doesn't match. `resource('products')` looks for a controller class named `Products`.

---

## 11. Run the application

```powershell
php spark serve
```

```
CodeIgniter v4.7.x Command Line Tool

CodeIgniter development server started on http://localhost:8080
Press Control-C to stop.
```

**Leave this terminal open.** MySQL must also still be running in XAMPP.

Port 8080 collides with the Spring Boot guide, so if both are running:

```powershell
php spark serve --port 8081
```

Quick check in your browser: <http://localhost:8080/api/products> should return `{"data":[],"meta":{...}}`. A 404 here means the routes didn't register — re-run `php spark routes`.

---

## 12. Test in Postman

### 12.1 Collection and environment

**Collection:** left sidebar → **Collections** → **+** → name it `CodeIgniter Product API`.

**Environment:** **Environments** → **+** → name it `Local CI`, add:

| Variable | Initial value | Current value |
|---|---|---|
| `base_url` | `http://localhost:8080/api` | `http://localhost:8080/api` |

Save (Ctrl+S) and **select `Local CI` in the top-right dropdown**. If it says "No Environment", `{{base_url}}` won't resolve.

**Collection headers:** right-click the collection → **Edit** → **Headers** tab → add `Accept` = `application/json`. Save.

### 12.2 CREATE — POST /api/products

1. Right-click the collection → **Add request**, name it `Create product`
2. Method **POST**, URL `{{base_url}}/products`
3. **Body** tab → **raw** → change the dropdown from *Text* to **JSON**
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

**Step 3 is not optional.** Selecting JSON sets `Content-Type: application/json`, which is what makes `getJSON()` in the controller return anything at all. Leave it on *Text* and you'll get a 400 claiming every field is required, with a perfectly valid body sitting right there in the request pane.

Expected: **201 Created**

```json
{
  "data": {
    "id": "1",
    "name": "Mechanical Keyboard",
    "sku": "KB-8700",
    "description": "Hot-swappable 75% layout",
    "price": "4250.00",
    "stock": "12",
    "is_active": "1",
    "created_at": "2026-09-12 08:31:04",
    "updated_at": "2026-09-12 08:31:04"
  }
}
```

**Notice the quotes around the numbers.** The MySQLi driver returns every column as a string, and CodeIgniter doesn't cast them back. Laravel's `$casts` and Spring's typed fields both handle this; CodeIgniter leaves it to you. Section 17 covers fixing it with an Entity class if your API consumers care.

#### Capture the ID automatically

In this request's **Scripts** tab (**Tests** in older Postman versions):

```javascript
pm.test("Status is 201", function () {
    pm.response.to.have.status(201);
});

pm.collectionVariables.set("product_id", pm.response.json().data.id);
```

`{{product_id}}` now works in the later requests.

### 12.3 Validation failure

Duplicate the request, rename it `Create product (invalid)`, body:

```json
{
  "name": "",
  "price": -5
}
```

Expected: **400 Bad Request**

```json
{
  "status": 400,
  "error": 400,
  "messages": {
    "name": "The name field is required.",
    "sku": "The sku field is required.",
    "price": "Price cannot be negative."
  }
}
```

Send the valid body twice and the second attempt returns "That SKU is already in use." — the `is_unique` rule from the model.

### 12.4 READ ALL — GET /api/products

Method **GET**, URL `{{base_url}}/products` → **200 OK**:

```json
{
  "data": [
    { "id": "1", "name": "Mechanical Keyboard", "...": "..." }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 15,
    "last_page": 1,
    "total": 1
  }
}
```

The **Params** tab drives the filters:

| Key | Value | Effect |
|---|---|---|
| `search` | `keyboard` | Matches name or SKU |
| `active` | `true` | Only active products |
| `per_page` | `5` | Page size |
| `page` | `2` | **One-indexed**, like Laravel (Spring starts at 0) |

### 12.5 READ ONE — GET /api/products/{id}

URL `{{base_url}}/products/{{product_id}}` → **200 OK**.

Change it to `/products/9999` → **404 Not Found**:

```json
{
  "status": 404,
  "error": 404,
  "messages": { "error": "No product found with ID 9999" }
}
```

That's your explicit `failNotFound()` call. Unlike Laravel's route model binding, CodeIgniter has no automatic 404 on a missing record — remove the check and `find()` returns `null`, which serialises to `{"data":null}` with a 200 status.

### 12.6 UPDATE — PATCH /api/products/{id}

Method **PATCH**, URL `{{base_url}}/products/{{product_id}}`, Body → raw → JSON:

```json
{
  "price": 3999.00,
  "stock": 8
}
```

Expected **200 OK** with the updated record and a new `updated_at`.

PATCH with a partial body works because of `$cleanValidationRules` — rules for absent fields are skipped. PUT hits the same method; the difference is convention only.

### 12.7 DELETE — DELETE /api/products/{id}

Method **DELETE**, URL `{{base_url}}/products/{{product_id}}` → **204 No Content**, empty body. Send again → **404**.

### 12.8 Run the whole collection

Right-click the collection → **Run collection**. Requests fire in sidebar order, so `Create product` must come before the requests that depend on `{{product_id}}`. Add a `pm.test(...)` block to each and this becomes a regression suite.

### 12.9 curl equivalents (optional)

```powershell
# CREATE
$body = '{"name":"Wireless Mouse","sku":"MS-2200","price":1250.00,"stock":30}'
curl.exe -X POST http://localhost:8080/api/products -H "Content-Type: application/json" -d $body

# READ ALL (quote the URL — PowerShell treats a bare & as an operator)
curl.exe "http://localhost:8080/api/products?per_page=5&search=mouse"

# READ ONE
curl.exe http://localhost:8080/api/products/1

# UPDATE
$update = '{"price":999.00}'
curl.exe -X PATCH http://localhost:8080/api/products/1 -H "Content-Type: application/json" -d $update

# DELETE
curl.exe -X DELETE http://localhost:8080/api/products/1 -i
```

Use `curl.exe`, not `curl` — plain `curl` is a PowerShell alias for `Invoke-WebRequest`. Put JSON in a variable first; PowerShell mangles quotes in inline `-d` arguments.

---

## 13. Seed sample data (optional)

Testing pagination with one row proves little.

```powershell
php spark make:seeder ProductSeeder
```

Open `app/Database/Seeds/ProductSeeder.php`:

```php
<?php

namespace App\Database\Seeds;

use CodeIgniter\Database\Seeder;

class ProductSeeder extends Seeder
{
    public function run()
    {
        $faker = \Faker\Factory::create();

        for ($i = 0; $i < 50; $i++) {
            $this->db->table('products')->insert([
                'name'        => $faker->words(3, true),
                'sku'         => strtoupper($faker->unique()->bothify('??-####')),
                'description' => $faker->sentence(12),
                'price'       => $faker->randomFloat(2, 100, 50000),
                'stock'       => $faker->numberBetween(0, 200),
                'is_active'   => $faker->boolean(85) ? 1 : 0,
                'created_at'  => date('Y-m-d H:i:s'),
                'updated_at'  => date('Y-m-d H:i:s'),
            ]);
        }
    }
}
```

Faker ships with the CodeIgniter appstarter as a dev dependency. Run it:

```powershell
php spark db:seed ProductSeeder
```

Now `GET /api/products?per_page=5` returns real pagination metadata.

---

## 14. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `'php' is not recognized` | XAMPP's PHP not on PATH | Add `C:\xampp\php`, open a **new** terminal (§2.2) |
| `Class "IntlDateFormatter" not found` / intl errors on boot | `intl` extension disabled | Uncomment `extension=intl` in `C:\xampp\php\php.ini` |
| `Could not find driver` | `mysqli` disabled | Uncomment `extension=mysqli` in `php.ini` |
| `Unable to connect to the database` | MySQL not started | Start MySQL in the XAMPP Control Panel |
| Connection fails only on `localhost` | Resolving to IPv6 `::1` | Use `127.0.0.1` in `.env` |
| `Unknown database 'product_api_ci'` | Schema never created | Run the `CREATE DATABASE` command in §5 |
| `Access denied for user 'root'@'localhost'` | Wrong credentials | XAMPP default is `root` with an **empty** password |
| `.env` edits have no effect | The `#` was left in front of the line | Delete the leading `#` as well as setting the value |
| Generic error page, no detail | `CI_ENVIRONMENT = production` | Set it to `development` in `.env` |
| `404 Not Found` on `/api/products` | Routes not registered | `php spark routes`; check the group in `app/Config/Routes.php` |
| Routes exist but handler is wrong | Controller class name mismatch | `resource('products')` needs a class named `Products` |
| Every field reports "required" with a valid body | Body type left on **Text** in Postman, or `getPost()` used instead of `getJSON()` | Body → raw → **JSON**; use `$this->request->getJSON(true)` |
| Update runs, nothing changes | Field missing from `$allowedFields` | Add it to the model's `$allowedFields` |
| "SKU already in use" when editing without changing it | `{id}` placeholder got no value | Merge `$data['id'] = $id;` before `update()` (§9) |
| Numbers returned as strings | MySQLi driver returns strings | Expected; use an Entity with `$casts` (§17) |
| `Failed to listen on localhost:8080` | Port in use | `php spark serve --port 8081` |
| `The migration file must be a class` | Class name doesn't match the file | Keep the generated filename; only edit the body |

When things behave oddly:

```powershell
php spark cache:clear
php spark migrate:status
```

`writable/logs/` holds the detailed error logs.

---

## 15. Command cheat sheet

```powershell
# Database (start MySQL in XAMPP first)
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api_ci CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
C:\xampp\mysql\bin\mysql.exe -u root product_api_ci -e "SELECT * FROM products;"

# Project setup
composer create-project codeigniter4/appstarter product-api-ci
copy env .env

# Generators
php spark make:migration CreateProducts
php spark make:model ProductModel
php spark make:controller Products --restful
php spark make:seeder ProductSeeder
php spark make:entity Product

# Migrations
php spark migrate
php spark migrate:status
php spark migrate:rollback
php spark migrate:refresh

# Seeding
php spark db:seed ProductSeeder

# Inspect
php spark routes
php spark list

# Run
php spark serve
php spark serve --port 8081

# Maintenance
php spark cache:clear
composer update
```

---

## 16. All three frameworks, side by side

| Concept | Laravel 13 | Spring Boot 4 | CodeIgniter 4 |
|---|---|---|---|
| Minimum PHP/Java | PHP 8.3 | Java 17 | **PHP 8.2 — XAMPP's own works** |
| Create project | `laravel new` | start.spring.io | `composer create-project` |
| Dev server | `php artisan serve` (8000) | `.\mvnw.cmd spring-boot:run` (8080) | `php spark serve` (8080) |
| Config file | `.env` | `application.properties` | `.env` |
| Schema | Migration + `migrate` | `ddl-auto=update` | Migration + `spark migrate` |
| Model | Eloquent | JPA `@Entity` | `CodeIgniter\Model` |
| Mass-assignment guard | `$fillable` | Request DTO | `$allowedFields` |
| Validation lives in | Form Request class | DTO annotations | **The model** |
| Read JSON body | automatic | `@RequestBody` | `$this->request->getJSON(true)` |
| 404 on missing record | automatic (route binding) | `orElseThrow()` | `failNotFound()` |
| Status codes | `response()->noContent()` etc. | `@ResponseStatus` | `respondCreated()` etc. |
| Pagination index | page 1 | page 0 | page 1 |
| JSON envelope | `{"data": ...}` | bare object | whatever you return |
| Types in responses | cast via `$casts` | native Java types | **strings unless you cast** |
| Controller method names | `store` / `destroy` | any | **`create` / `delete`** |

The practical summary: CodeIgniter asks for the least setup and gives you the least automatic behaviour. You write the 404 checks and the type casting yourself. That's a fair trade when the app is small, and it's why CodeIgniter remains a common choice for teaching — there's less framework magic between the request and the SQL.

---

## 17. Where to go next

1. **Add an Entity for type casting.** `php spark make:entity Product`, set `protected $casts = ['id' => 'integer', 'price' => 'float', 'stock' => 'integer', 'is_active' => 'boolean'];`, then set `protected $returnType = \App\Entities\Product::class;` in the model. Your JSON stops returning numbers as strings.
2. **Add authentication with Shield** — `composer require codeigniter4/shield`, the official auth package, which supports API tokens.
3. **Add a service layer** once controller methods grow past a few lines. CodeIgniter has no `app/Services` convention, so create one and register it in `app/Config/Services.php`.
4. **Add CORS** if a browser front end will call this API: `composer require agungsugiarto/codeigniter4-cors`.
5. **Add soft deletes** — set `$useSoftDeletes = true` in the model and add a nullable `deleted_at DATETIME` column via a migration.
6. **Add filters** for rate limiting or API keys — `php spark make:filter ApiKeyFilter`, then register it in `app/Config/Filters.php`.
7. **Write tests** — CodeIgniter bundles PHPUnit with `FeatureTestTrait`: `$this->call('get', 'api/products')->assertStatus(200)`.

### Reference links

- CodeIgniter 4 user guide: <https://codeigniter.com/user_guide/index.html>
- Model and validation: <https://codeigniter.com/user_guide/models/model.html>
- ResourceController and ResponseTrait: <https://codeigniter.com/user_guide/incoming/restful.html>
- Validation rule reference: <https://codeigniter.com/user_guide/libraries/validation.html>
- Shield (authentication): <https://shield.codeigniter.com/>
