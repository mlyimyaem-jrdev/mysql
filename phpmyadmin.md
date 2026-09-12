# Building a Simple CRUD API in Laravel 13

A complete walkthrough — from installing PHP on a blank machine to testing five working endpoints.

**Target version:** Laravel 13.x (released March 2026, requires PHP 8.3+)
**Platform:** Windows 10 / 11, using PowerShell
**Example resource:** `Product`
**Estimated time:** 45–60 minutes on a fresh machine, ~15 minutes if PHP and Composer are already installed.

---

## Table of Contents

1. [What you'll build](#1-what-youll-build)
2. [Prerequisites and downloads](#2-prerequisites-and-downloads)
3. [Create the Laravel project](#3-create-the-laravel-project)
4. [Project structure tour](#4-project-structure-tour)
5. [Create the database and configure .env](#5-create-the-database-and-configure-env)
6. [Enable API routing](#6-enable-api-routing)
7. [Generate the CRUD scaffolding](#7-generate-the-crud-scaffolding)
8. [Write the migration](#8-write-the-migration)
9. [Configure the model](#9-configure-the-model)
10. [Add validation with Form Requests](#10-add-validation-with-form-requests)
11. [Shape the output with an API Resource](#11-shape-the-output-with-an-api-resource)
12. [Write the controller](#12-write-the-controller)
13. [Register the routes](#13-register-the-routes)
14. [Run the app and test in Postman](#14-run-the-app-and-test-in-postman)
15. [Seed sample data (optional)](#15-seed-sample-data-optional)
16. [Write an automated test (optional)](#16-write-an-automated-test-optional)
17. [Troubleshooting](#17-troubleshooting)
18. [Command cheat sheet](#18-command-cheat-sheet)
19. [Where to go next](#19-where-to-go-next)

---

## 1. What you'll build

A REST API for managing products. Five endpoints following standard REST conventions:

| Method | URI | Action | Purpose |
|---|---|---|---|
| GET | `/api/products` | `index` | List all products (paginated) |
| POST | `/api/products` | `store` | Create a product |
| GET | `/api/products/{id}` | `show` | Fetch one product |
| PUT/PATCH | `/api/products/{id}` | `update` | Update a product |
| DELETE | `/api/products/{id}` | `destroy` | Delete a product |

The product table will hold: `id`, `name`, `sku`, `description`, `price`, `stock`, `is_active`, `created_at`, `updated_at`.

---

## 2. Prerequisites and downloads

Four pieces of software. Install them in this order.

### 2.1 XAMPP (MySQL + phpMyAdmin)

Download from <https://www.apachefriends.org/download.html> and run the installer. Accept the defaults — it lands in `C:\xampp`.

The components that matter for this guide are **MySQL** and **phpMyAdmin**. Apache is optional: Laravel ships its own development server (`php artisan serve`), so your project will *not* live in `htdocs`. Start Apache anyway if you want the phpMyAdmin web UI.

After installing, open the **XAMPP Control Panel** and click **Start** next to **MySQL**. The label turns green and shows port 3306.

Three things to know about XAMPP's database:

- What XAMPP labels "MySQL" is actually **MariaDB**, a drop-in compatible fork. Laravel's `mysql` driver talks to it without any changes.
- The default account is user `root` with an **empty password**. Fine on your own machine, never on a server.
- phpMyAdmin is at <http://localhost/phpmyadmin> and needs Apache running as well as MySQL.

### 2.2 PHP 8.3 or higher (for the command line)

**Read this even though XAMPP already includes PHP.** Laravel 13 requires **PHP 8.3 minimum**, and the official XAMPP for Windows installer has been shipping PHP 8.2.12 as its newest build for a while now. If that's the PHP on your PATH, Composer will refuse to install Laravel 13 with a "requirements could not be resolved" error.

Check what you have:

```powershell
php -v
```

Pick the branch that matches the output:

#### If it shows 8.3 or higher

Nothing to do. Skip to 2.3.

#### If it shows 8.2 or lower, or `php` is not recognized — recommended fix

Install a standalone PHP for the command line and leave XAMPP's copy alone. They don't conflict, because XAMPP doesn't add its own PHP to your PATH by default.

1. Download the **Thread Safe x64 ZIP** of PHP 8.3 or 8.4 from <https://windows.php.net/download/>
2. Extract it to `C:\php`
3. In that folder, copy `php.ini-development` and rename the copy to `php.ini`
4. Open `php.ini` and uncomment (delete the leading `;` from) these lines:
   ```ini
   extension_dir = "ext"
   extension=curl
   extension=fileinfo
   extension=mbstring
   extension=openssl
   extension=pdo_mysql
   extension=mysqli
   extension=zip
   extension=intl
   ```
   `pdo_mysql` is the one Laravel actually connects to MySQL through — don't skip it.
5. Add `C:\php` to your **PATH**: press `Win`, search "Edit the system environment variables" → Environment Variables → select `Path` under **User variables** → Edit → New → `C:\php` → OK.
   **If `C:\xampp\php` is already in that list, select `C:\php` and click Move Up** until it sits above it. Windows uses the first match, so otherwise XAMPP's older PHP keeps winning.
6. Open a **new** terminal and run `php -v` again.

#### If you'd rather not install a second PHP

Build on **Laravel 12** instead — it only needs PHP 8.2, which XAMPP has. Create the project with `composer create-project laravel/laravel:^12.0 product-api` and read `12.x` wherever this guide links to `13.x` docs. Every other step is identical. Laravel 12 gets bug fixes until August 2026 and security fixes until February 2027.

#### Confirm the extensions

```powershell
php -m | Select-String "mbstring|openssl|pdo_mysql|tokenizer|xml|ctype|json|curl|fileinfo"
```

`pdo_mysql` **must** appear in that list. If it doesn't, revisit step 4 and open a fresh terminal.

### 2.3 Composer (PHP dependency manager)

Check first:

```powershell
composer -V
```

If it's missing, download and run `Composer-Setup.exe` from <https://getcomposer.org/download/>. It auto-detects your PHP — when it asks, point it at `C:\php\php.exe`, **not** `C:\xampp\php\php.exe`, so Composer uses the newer version.

Close your terminal, open a new one, and run `composer -V` again.

### 2.4 Postman (API testing)

Download the Windows 64-bit build from <https://www.postman.com/downloads/> and install it.

On first launch Postman asks you to sign in. You can skip that with the "Continue without an account" link at the bottom of the window, though an account lets your collections sync between machines and survive a reinstall.

Section 14 builds a full collection with all five endpoints.

### 2.5 Verification checklist

Open **Windows PowerShell** (or Windows Terminal) and run:

```powershell
php -v       # 8.3.x or higher
composer -V  # Composer 2.x
```

Then check the XAMPP Control Panel: **MySQL** should be green and running.

Every command in this guide is written for PowerShell. Where CMD behaves differently, I've noted it.

---

## 3. Create the Laravel project

Two ways. Both produce the same result.

### Option A — the Laravel installer (recommended)

```powershell
composer global require laravel/installer
```

Then add Composer's global bin directory to your PATH so the `laravel` command is found:

The path to add is:

```
%USERPROFILE%\AppData\Roaming\Composer\vendor\bin
```

Add it exactly the way you added PHP: `Win` → "Edit the system environment variables" → Environment Variables → select `Path` under **User variables** → Edit → New → paste the line above → OK.

Confirm the folder exists first — run `dir $env:APPDATA\Composer\vendor\bin` in PowerShell and you should see `laravel.bat`.

Open a **new** terminal, then create the project:

```powershell
laravel new product-api
```

The installer asks a few questions. For this guide:

- **Starter kit:** `None` (we only need an API)
- **Testing framework:** `PHPUnit` (or Pest, if you prefer its syntax)
- **Database:** `MySQL`
- **Run default migrations:** `No` — say no here. The database doesn't exist yet; you'll create it and run migrations in section 5.

### Option B — Composer directly

```powershell
composer create-project laravel/laravel product-api
```

### Then enter the project

```powershell
cd product-api
php artisan --version
```

You should see `Laravel Framework 13.x.x`.

> **What just happened:** Composer downloaded the framework and ~60 dependency packages into `vendor/`, generated an `APP_KEY` in `.env`, and set up a runnable skeleton app.

---

## 4. Project structure tour

You only need to care about a handful of folders:

```
product-api/
├── app/
│   ├── Http/
│   │   ├── Controllers/    ← your controllers live here
│   │   ├── Requests/       ← validation classes
│   │   └── Resources/      ← JSON output formatters
│   └── Models/             ← Eloquent models
├── bootstrap/
│   └── app.php             ← routing, middleware, exception config
├── config/                 ← framework config files
├── database/
│   ├── factories/          ← fake data generators for tests
│   ├── migrations/         ← version-controlled schema changes
│   └── seeders/            ← sample data loaders
├── routes/
│   ├── web.php             ← browser routes (session, CSRF)
│   └── api.php             ← API routes (NOT created by default — see §6)
├── tests/
├── .env                    ← your local secrets/config (git-ignored)
└── composer.json
```

Coming from Spring Boot, the rough mapping is: `Controller` → `@RestController`, `Model` → `@Entity`, `FormRequest` → bean validation on a DTO, `Resource` → a response DTO/mapper, and `migration` → a Flyway/Liquibase changeset.

---

## 5. Create the database and configure .env

### 5.1 Create the schema

Laravel will create *tables* for you, but not the database itself. Make it first, using either method.

**With phpMyAdmin (GUI):** start **Apache** in the XAMPP Control Panel alongside MySQL, open <http://localhost/phpmyadmin>, click **New** in the left sidebar, enter `product_api` as the database name, set collation to `utf8mb4_unicode_ci`, and click **Create**.

**With the command line (no Apache needed):**

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

No `-p` flag, because the default XAMPP root account has no password. If you've set one, add `-p` and it will prompt you.

Confirm it exists:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "SHOW DATABASES;"
```

> **Tip:** add `C:\xampp\mysql\bin` to your PATH and you can type just `mysql -u root` from anywhere.

### 5.2 Point Laravel at it

Open `.env` in the project root — this file holds environment-specific settings and is excluded from Git. Set these values:

```env
APP_NAME="Product API"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=product_api
DB_USERNAME=root
DB_PASSWORD=
```

Points worth noting:

- `DB_PASSWORD=` is deliberately empty — that's the XAMPP default. Leave nothing after the `=`.
- Use `127.0.0.1`, not `localhost`. On Windows, `localhost` can resolve to the IPv6 address `::1`, which MySQL isn't listening on, producing a confusing connection refused error.
- If you changed MySQL's port (see the troubleshooting table for why you might have), update `DB_PORT` to match.

### 5.3 Run the migrations

Make sure MySQL shows green in the XAMPP Control Panel, then:

```powershell
php artisan migrate
```

Expected output:

```
INFO  Preparing database.

INFO  Running migrations.

0001_01_01_000000_create_users_table ................. 42ms DONE
0001_01_01_000001_create_cache_table ................. 18ms DONE
0001_01_01_000002_create_jobs_table .................. 31ms DONE
```

Those are Laravel's built-in tables. If this completes without error, your connection works — refresh phpMyAdmin and you'll see them under `product_api`.

If you edited `.env` and nothing seems to change, Laravel is reading a cached copy:

```powershell
php artisan config:clear
```

---

## 6. Enable API routing

**This is the step most outdated tutorials skip.** Since Laravel 11, a fresh project does **not** include `routes/api.php`. Run:

```powershell
php artisan install:api
```

This command does three things:

1. Creates `routes/api.php`
2. Installs **Laravel Sanctum** (token authentication — you won't use it yet, but it's wired up for later)
3. Registers the API route file in `bootstrap/app.php` with the `/api` prefix and the `api` middleware group

It will prompt to run Sanctum's migration — answer **yes**.

Confirm the file now exists:

```powershell
type routes\api.php
```

> If you see `404 Not Found` on every `/api/...` URL later, this step is almost always the cause.

---

## 7. Generate the CRUD scaffolding

Artisan can generate the model, migration, controller, and validation classes in a single command:

```powershell
php artisan make:model Product --migration --controller --api --requests
```

Flag by flag:

| Flag | What it creates |
|---|---|
| *(none)* | `app/Models/Product.php` |
| `--migration` / `-m` | `database/migrations/xxxx_create_products_table.php` |
| `--controller` / `-c` | `app/Http/Controllers/ProductController.php` |
| `--api` | Makes the controller a resource controller **without** `create`/`edit` methods (those return HTML forms — useless for an API) |
| `--requests` | `StoreProductRequest.php` and `UpdateProductRequest.php` in `app/Http/Requests/` |

Then generate the API resource separately:

```powershell
php artisan make:resource ProductResource
```

Files created:

```
app/Models/Product.php
app/Http/Controllers/ProductController.php
app/Http/Requests/StoreProductRequest.php
app/Http/Requests/UpdateProductRequest.php
app/Http/Resources/ProductResource.php
database/migrations/2026_09_12_000000_create_products_table.php
```

---

## 8. Write the migration

Open the new migration file in `database/migrations/` (the filename is prefixed with a timestamp) and replace its contents:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('sku')->unique();
            $table->text('description')->nullable();
            $table->decimal('price', 10, 2);
            $table->unsignedInteger('stock')->default(0);
            $table->boolean('is_active')->default(true);
            $table->timestamps();

            $table->index('is_active');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

Notes on the column types:

- `$table->id()` — auto-incrementing big integer primary key
- `decimal('price', 10, 2)` — never use `float` for money; you'll get rounding errors
- `timestamps()` — adds `created_at` and `updated_at`, which Eloquent maintains automatically
- `down()` — the rollback. Always fill it in so `php artisan migrate:rollback` works.

Run it:

```powershell
php artisan migrate
```

Expected output:

```
INFO  Running migrations.
2026_09_12_000000_create_products_table .............. 12ms DONE
```

Useful related commands:

```powershell
php artisan migrate:status     # which migrations have run
php artisan migrate:rollback   # undo the last batch
php artisan migrate:fresh      # drop all tables and re-run everything (destroys data)
```

---

## 9. Configure the model

Open `app/Models/Product.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    use HasFactory;

    /**
     * Attributes that can be set via create() or update().
     */
    protected $fillable = [
        'name',
        'sku',
        'description',
        'price',
        'stock',
        'is_active',
    ];

    /**
     * Type conversion between database and PHP.
     */
    protected function casts(): array
    {
        return [
            'price' => 'decimal:2',
            'stock' => 'integer',
            'is_active' => 'boolean',
        ];
    }
}
```

Two things worth understanding:

**`$fillable`** is mass-assignment protection. Without it, a malicious request body could set any column — including ones you never intended to expose. Anything not in this list is silently ignored by `create()` and `update()`.

**`casts()`** converts types on the way in and out. MySQL stores booleans as `TINYINT(1)`, so without the boolean cast your JSON would return `1` instead of `true`, and `decimal` columns come back from the driver as strings unless cast.

> Laravel 13 also supports PHP attributes as an alternative to properties — e.g. `#[Table('products')]` above the class. The property style shown here still works and is what most existing code uses.

---

## 10. Add validation with Form Requests

Validation lives in its own class, keeping the controller clean. Open `app/Http/Requests/StoreProductRequest.php`:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // no auth yet — see §19
    }

    public function rules(): array
    {
        return [
            'name'        => ['required', 'string', 'max:255'],
            'sku'         => ['required', 'string', 'max:64', 'unique:products,sku'],
            'description' => ['nullable', 'string', 'max:2000'],
            'price'       => ['required', 'numeric', 'min:0'],
            'stock'       => ['nullable', 'integer', 'min:0'],
            'is_active'   => ['nullable', 'boolean'],
        ];
    }

    public function messages(): array
    {
        return [
            'sku.unique' => 'That SKU is already in use.',
            'price.min'  => 'Price cannot be negative.',
        ];
    }
}
```

> **`authorize()` returning `false` produces a 403.** This is the hook for permission checks later — leave it `true` while you're building.

Now `app/Http/Requests/UpdateProductRequest.php`. The differences: fields are `sometimes` (partial updates via PATCH), and the unique rule must ignore the record being edited.

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        $productId = $this->route('product')->id;

        return [
            'name'        => ['sometimes', 'required', 'string', 'max:255'],
            'sku'         => [
                'sometimes',
                'required',
                'string',
                'max:64',
                Rule::unique('products', 'sku')->ignore($productId),
            ],
            'description' => ['nullable', 'string', 'max:2000'],
            'price'       => ['sometimes', 'required', 'numeric', 'min:0'],
            'stock'       => ['sometimes', 'integer', 'min:0'],
            'is_active'   => ['sometimes', 'boolean'],
        ];
    }
}
```

Without `->ignore()`, updating a product without changing its SKU would fail — the record collides with itself.

When validation fails and the request carries `Accept: application/json`, Laravel returns **422 Unprocessable Entity** with a structured error body. No try/catch needed anywhere.

---

## 11. Shape the output with an API Resource

A Resource decides exactly which fields the client sees. Without it you'd dump the raw database row, which leaks internal columns the moment you add one. Open `app/Http/Resources/ProductResource.php`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'          => $this->id,
            'name'        => $this->name,
            'sku'         => $this->sku,
            'description' => $this->description,
            'price'       => (float) $this->price,
            'stock'       => $this->stock,
            'is_active'   => $this->is_active,
            'created_at'  => $this->created_at?->toIso8601String(),
            'updated_at'  => $this->updated_at?->toIso8601String(),
        ];
    }
}
```

---

## 12. Write the controller

Replace the contents of `app/Http/Controllers/ProductController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreProductRequest;
use App\Http\Requests\UpdateProductRequest;
use App\Http\Resources\ProductResource;
use App\Models\Product;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Http\Response;

class ProductController extends Controller
{
    /**
     * GET /api/products
     */
    public function index(Request $request): AnonymousResourceCollection
    {
        $products = Product::query()
            ->when($request->filled('search'), function ($query) use ($request) {
                $term = $request->string('search');
                $query->where('name', 'like', "%{$term}%")
                      ->orWhere('sku', 'like', "%{$term}%");
            })
            ->when($request->filled('active'), function ($query) use ($request) {
                $query->where('is_active', $request->boolean('active'));
            })
            ->latest()
            ->paginate($request->integer('per_page', 15));

        return ProductResource::collection($products);
    }

    /**
     * POST /api/products
     */
    public function store(StoreProductRequest $request): JsonResponse
    {
        $product = Product::create($request->validated());

        return ProductResource::make($product)
            ->response()
            ->setStatusCode(Response::HTTP_CREATED); // 201
    }

    /**
     * GET /api/products/{product}
     */
    public function show(Product $product): ProductResource
    {
        return ProductResource::make($product);
    }

    /**
     * PUT|PATCH /api/products/{product}
     */
    public function update(UpdateProductRequest $request, Product $product): ProductResource
    {
        $product->update($request->validated());

        return ProductResource::make($product);
    }

    /**
     * DELETE /api/products/{product}
     */
    public function destroy(Product $product): Response
    {
        $product->delete();

        return response()->noContent(); // 204
    }
}
```

### What's doing the work here

**Route model binding.** Notice `show(Product $product)` — you never call `Product::find($id)`. Laravel sees the type hint, matches the `{product}` route segment to the primary key, fetches the record, and **automatically returns 404** if it doesn't exist. This is the single biggest boilerplate saving in the whole file.

**Automatic dependency injection.** Type-hinting `StoreProductRequest` runs the validation before the method body executes. If validation fails, the method is never called.

**`$request->validated()`** returns only the fields that passed your rules — never the raw input. Combined with `$fillable`, that's two layers of mass-assignment protection.

**Status codes.** `store` returns 201 Created, `destroy` returns 204 No Content, the rest return 200. These are what REST clients expect.

**`when()`** applies a query condition only if the value is present, so `?search=` and `?active=` are optional filters that compose cleanly.

---

## 13. Register the routes

Open `routes/api.php` and add:

```php
<?php

use App\Http\Controllers\ProductController;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');

Route::apiResource('products', ProductController::class);
```

That one `apiResource` line registers all five routes with correct verbs, URIs, and names. (`Route::resource()` would add two extra HTML-form routes you don't want.)

If you need only some of them:

```php
Route::apiResource('products', ProductController::class)
     ->only(['index', 'show']);

// or
Route::apiResource('products', ProductController::class)
     ->except(['destroy']);
```

Verify:

```powershell
php artisan route:list --path=api
```

Expected:

```
GET|HEAD   api/products ............ products.index  › ProductController@index
POST       api/products ............ products.store  › ProductController@store
GET|HEAD   api/products/{product} .. products.show   › ProductController@show
PUT|PATCH  api/products/{product} .. products.update › ProductController@update
DELETE     api/products/{product} .. products.destroy › ProductController@destroy
```

If this list is empty, revisit section 6.

---

## 14. Run the app and test in Postman

Start the development server:

```powershell
php artisan serve
```

```
INFO  Server running on [http://127.0.0.1:8000].
```

**Leave this terminal open.** Closing it stops the server. MySQL must also still be running in the XAMPP Control Panel.

Quick smoke test before opening Postman: visit <http://localhost:8000/api/products> in your browser. You should get `{"data":[],"links":{...},"meta":{...}}`. A 404 here means section 6 was skipped.

### 14.1 Set up a collection and an environment

Doing this first saves retyping the base URL into every request.

**Create the collection:**

1. In Postman's left sidebar, click **Collections** → **+** (New Collection)
2. Name it `Product API`

**Create the environment:**

1. Click **Environments** in the left sidebar → **+**
2. Name it `Local`
3. Add a variable:

| Variable | Initial value | Current value |
|---|---|---|
| `base_url` | `http://localhost:8000/api` | `http://localhost:8000/api` |

4. **Save** (Ctrl+S), then select `Local` from the environment dropdown in the top-right corner. If that dropdown still says "No Environment", `{{base_url}}` won't resolve and your requests will fail.

**Set collection-wide headers** so you don't repeat them five times:

1. Right-click the `Product API` collection → **Edit**
2. Open the **Headers** tab (not Authorization)
3. Add: Key `Accept`, Value `application/json`
4. **Save**

> That `Accept` header matters more than it looks. Without it, Laravel returns an HTML error page on failures instead of a JSON error body — you'd see a wall of markup instead of a readable validation message.

### 14.2 CREATE — POST /api/products

1. Right-click the collection → **Add request**, name it `Create product`
2. Set the method dropdown to **POST**
3. URL: `{{base_url}}/products`
4. Open the **Body** tab → select **raw** → change the dropdown on the right from *Text* to **JSON**
5. Paste:

```json
{
  "name": "Mechanical Keyboard",
  "sku": "KB-8700",
  "description": "Hot-swappable 75% layout",
  "price": 4250.00,
  "stock": 12
}
```

6. Click **Send**, then **Save** (Ctrl+S)

Selecting **JSON** in step 4 makes Postman set `Content-Type: application/json` automatically. If you leave it on *Text*, Laravel receives the body as a plain string and every field comes back as "required".

Expected response — status **201 Created** in the top-right of the response pane:

```json
{
  "data": {
    "id": 1,
    "name": "Mechanical Keyboard",
    "sku": "KB-8700",
    "description": "Hot-swappable 75% layout",
    "price": 4250,
    "stock": 12,
    "is_active": true,
    "created_at": "2026-09-12T08:31:04+00:00",
    "updated_at": "2026-09-12T08:31:04+00:00"
  }
}
```

Open phpMyAdmin and look at the `products` table — the row is there.

#### Capture the new ID automatically (optional but convenient)

Rather than copying IDs by hand into the show/update/delete requests, have Postman remember it. In the **Scripts** tab of this request (older versions call it **Tests**), add:

```javascript
pm.test("Status is 201", function () {
    pm.response.to.have.status(201);
});

pm.collectionVariables.set("product_id", pm.response.json().data.id);
```

Now `{{product_id}}` works as a URL segment in the later requests, always pointing at whatever you last created.

### 14.3 Validation failure — what a 422 looks like

Duplicate the create request (right-click → **Duplicate**), rename it `Create product (invalid)`, and change the body to:

```json
{
  "name": "",
  "price": -5
}
```

Send it. Status **422 Unprocessable Entity**:

```json
{
  "message": "The name field is required. (and 2 more errors)",
  "errors": {
    "name": ["The name field is required."],
    "sku": ["The sku field is required."],
    "price": ["Price cannot be negative."]
  }
}
```

You wrote no error-handling code anywhere to get this. It comes from the Form Request in section 10.

### 14.4 READ ALL — GET /api/products

1. **Add request** → name it `List products`
2. Method **GET**, URL `{{base_url}}/products`
3. **Send**

Status **200 OK**, with pagination metadata Laravel adds for free:

```json
{
  "data": [ { "id": 1, "name": "Mechanical Keyboard", "...": "..." } ],
  "links": {
    "first": "http://localhost:8000/api/products?page=1",
    "last": "http://localhost:8000/api/products?page=1",
    "prev": null,
    "next": null
  },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 1,
    "per_page": 15,
    "to": 1,
    "total": 1
  }
}
```

**To test the filters**, open the **Params** tab and add key/value rows — Postman appends them to the URL as a query string:

| Key | Value | Effect |
|---|---|---|
| `search` | `keyboard` | Matches name or SKU |
| `active` | `true` | Only active products |
| `per_page` | `5` | Page size |

Untick a row's checkbox to disable it without deleting it. Section 15 seeds enough rows to make pagination visible.

### 14.5 READ ONE — GET /api/products/{id}

1. **Add request** → `Show product`
2. Method **GET**, URL `{{base_url}}/products/{{product_id}}` (or hardcode `/products/1`)
3. **Send** → **200 OK** with a single `data` object

Change the URL to `{{base_url}}/products/9999` and send again. Status **404 Not Found**:

```json
{ "message": "No query results for model [App\\Models\\Product] 9999" }
```

That 404 is route model binding doing its job — you never wrote a "not found" check.

### 14.6 UPDATE — PATCH /api/products/{id}

1. **Add request** → `Update product`
2. Method **PATCH**, URL `{{base_url}}/products/{{product_id}}`
3. **Body** → **raw** → **JSON**:

```json
{
  "price": 3999.00,
  "stock": 8
}
```

4. **Send** → **200 OK** with the updated record

PATCH sends only the fields you're changing; PUT conventionally sends the whole object. Both route to the same `update()` method here, and the `sometimes` rules in `UpdateProductRequest` are what make the partial version valid.

### 14.7 DELETE — DELETE /api/products/{id}

1. **Add request** → `Delete product`
2. Method **DELETE**, URL `{{base_url}}/products/{{product_id}}`
3. **Send**

Status **204 No Content** and an empty response body. That's correct, not a bug — 204 means "done, nothing to tell you". Re-send it and you'll get a 404, since the row is gone.

### 14.8 Run the whole collection at once

Once all five requests are saved: right-click the collection → **Run collection** → **Run Product API**. Postman fires them in order and shows a pass/fail summary. Add a `pm.test(...)` block in each request's **Scripts** tab (like the one in 14.2) and this becomes a genuine regression suite you can re-run after any code change.

Order matters — create must run before show, update, and delete, since they depend on `{{product_id}}`. Drag requests in the sidebar to reorder.

### 14.9 curl equivalents (optional)

If you'd rather stay in the terminal, curl ships with Windows 10 (build 1803) and 11. In PowerShell call it as **`curl.exe`** — plain `curl` is an alias for `Invoke-WebRequest`, which takes different flags.

```powershell
# CREATE
$body = '{"name":"Wireless Mouse","sku":"MS-2200","price":1250.00,"stock":30}'
curl.exe -X POST http://localhost:8000/api/products -H "Content-Type: application/json" -H "Accept: application/json" -d $body

# READ ALL (quotes are required — PowerShell treats a bare & as an operator)
curl.exe "http://localhost:8000/api/products?per_page=5" -H "Accept: application/json"

# READ ONE
curl.exe http://localhost:8000/api/products/1 -H "Accept: application/json"

# UPDATE
$update = '{"price":999.00}'
curl.exe -X PATCH http://localhost:8000/api/products/1 -H "Content-Type: application/json" -H "Accept: application/json" -d $update

# DELETE (-i shows the status line, since the body is empty)
curl.exe -X DELETE http://localhost:8000/api/products/1 -H "Accept: application/json" -i
```

Put JSON in a variable first as shown — PowerShell mangles quotes in inline `-d` arguments. To read the output comfortably:

```powershell
curl.exe -s http://localhost:8000/api/products -H "Accept: application/json" | ConvertFrom-Json | ConvertTo-Json -Depth 5
```

---

## 15. Seed sample data (optional)

Testing pagination with one record isn't informative. Create a factory:

```powershell
php artisan make:factory ProductFactory --model=Product
```

`database/factories/ProductFactory.php`:

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

class ProductFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name'        => fake()->words(3, true),
            'sku'         => strtoupper(fake()->unique()->bothify('??-####')),
            'description' => fake()->sentence(12),
            'price'       => fake()->randomFloat(2, 100, 50000),
            'stock'       => fake()->numberBetween(0, 200),
            'is_active'   => fake()->boolean(85),
        ];
    }
}
```

Generate 50 rows from the interactive shell:

```powershell
php artisan tinker
```

```php
App\Models\Product::factory()->count(50)->create();
exit
```

Now `GET /api/products` returns a real paginated list.

---

## 16. Write an automated test (optional)

Laravel ships with a test suite ready to go. Create one:

```powershell
php artisan make:test ProductApiTest
```

`tests/Feature/ProductApiTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Models\Product;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ProductApiTest extends TestCase
{
    use RefreshDatabase;

    public function test_it_lists_products(): void
    {
        Product::factory()->count(3)->create();

        $this->getJson('/api/products')
             ->assertOk()
             ->assertJsonCount(3, 'data')
             ->assertJsonStructure(['data' => [['id', 'name', 'sku', 'price']], 'meta']);
    }

    public function test_it_creates_a_product(): void
    {
        $payload = [
            'name'  => 'Test Mouse',
            'sku'   => 'MS-0001',
            'price' => 1250.50,
            'stock' => 5,
        ];

        $this->postJson('/api/products', $payload)
             ->assertCreated()
             ->assertJsonPath('data.name', 'Test Mouse');

        $this->assertDatabaseHas('products', ['sku' => 'MS-0001']);
    }

    public function test_it_rejects_invalid_input(): void
    {
        $this->postJson('/api/products', ['name' => '', 'price' => -1])
             ->assertStatus(422)
             ->assertJsonValidationErrors(['name', 'sku', 'price']);
    }

    public function test_it_updates_a_product(): void
    {
        $product = Product::factory()->create(['price' => 100]);

        $this->patchJson("/api/products/{$product->id}", ['price' => 250])
             ->assertOk()
             ->assertJsonPath('data.price', 250);
    }

    public function test_it_deletes_a_product(): void
    {
        $product = Product::factory()->create();

        $this->deleteJson("/api/products/{$product->id}")
             ->assertNoContent();

        $this->assertDatabaseMissing('products', ['id' => $product->id]);
    }

    public function test_it_returns_404_for_missing_product(): void
    {
        $this->getJson('/api/products/9999')->assertNotFound();
    }
}
```

Run it:

```powershell
php artisan test
```

#### Important: use a separate test database

`RefreshDatabase` **drops and re-creates every table** in whatever database is configured before wrapping each test in a transaction. Point it at `product_api` and `php artisan test` will wipe the data you've been working with.

Create a second schema:

```powershell
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

Then open `phpunit.xml` in the project root and, inside the `<php>` block, uncomment or add these lines:

```xml
<php>
    <env name="APP_ENV" value="testing"/>
    <env name="DB_CONNECTION" value="mysql"/>
    <env name="DB_DATABASE" value="product_api_test"/>
    <!-- ...existing entries... -->
</php>
```

Now tests run against `product_api_test` and your development data is left alone. Verify with `php artisan test` followed by a look at phpMyAdmin — `product_api` should still hold your rows.

---

## 17. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `'php' is not recognized...` | `C:\php` not on PATH, or terminal opened before you added it | Re-check the PATH entry, then open a **new** terminal |
| `'laravel' is not recognized...` | Composer's global bin folder not on PATH | Add it (§3), or skip it and use `composer create-project laravel/laravel product-api` |
| `Invoke-WebRequest: A parameter cannot be found that matches parameter name 'X'` | You typed `curl` instead of `curl.exe` in PowerShell | Use `curl.exe` — plain `curl` is a PowerShell alias |
| `The term '&' is not a valid...` or the URL truncates at `&` | Unquoted query string in PowerShell | Wrap the whole URL in double quotes |
| `Failed to listen on 127.0.0.1:8000` | Port already in use | `php artisan serve --port=8001` |
| `404 Not Found` on every `/api/*` route | `routes/api.php` doesn't exist | Run `php artisan install:api` |
| `Class "App\Http\Requests\StoreProductRequest" not found` | Missing `use` statement | Check the imports at the top of the controller |
| `SQLSTATE[HY000] [2002] No connection could be made...` | MySQL not started in XAMPP | Open the XAMPP Control Panel and Start MySQL |
| `SQLSTATE[HY000] [2002] Connection refused` on `localhost` | `localhost` resolving to IPv6 `::1` | Set `DB_HOST=127.0.0.1` in `.env`, then `php artisan config:clear` |
| `SQLSTATE[HY000] [1049] Unknown database 'product_api'` | Schema never created | Create it in phpMyAdmin or via the `mysql.exe` command in §5.1 |
| `SQLSTATE[HY000] [1045] Access denied for user 'root'` | Wrong credentials | XAMPP's default is user `root` with an **empty** `DB_PASSWORD=` |
| MySQL won't start in XAMPP; log mentions port 3306 | Another MySQL service is already using 3306 | Stop it in `services.msc`, or change XAMPP's port in `my.ini` and match it in `DB_PORT` |
| `Specified key was too long; max key length is 1000 bytes` | Very old MySQL/MariaDB with utf8mb4 | Update XAMPP, or add `Schema::defaultStringLength(191);` in `AppServiceProvider::boot()` |
| `.env` change has no effect | Cached config | `php artisan config:clear` |
| `Add [name] to fillable property` | Field missing from `$fillable` | Add it to the model's `$fillable` array |
| HTML error page instead of JSON | Missing header | Add `Accept: application/json` to the collection headers (§14.1) |
| Postman: every field reports "required" though you sent them | Body type left on **Text** instead of **JSON** | Body tab → raw → change the dropdown to **JSON** |
| Postman: URL shows a literal `{{base_url}}` / request fails | No environment selected | Pick `Local` in the top-right environment dropdown |
| `419 Page Expired` on POST | Route is in `web.php`, not `api.php` | Move the route; API routes are CSRF-exempt |
| `Route [products.index] not defined` | Route file not registered | `php artisan route:list`, then `php artisan route:clear` |
| `Your requirements could not be resolved` on install | PHP below 8.3 — usually XAMPP's bundled 8.2 winning on PATH | Install standalone PHP 8.3+ and move `C:\\php` above `C:\\xampp\\php` in PATH (§2.2) |
| `SKU already taken` when editing without changing it | Missing `Rule::unique()->ignore()` | See §10 |
| 500 error with no detail | `APP_DEBUG=false` | Set `APP_DEBUG=true` locally, check `storage/logs/laravel.log` |

General reset sequence when things are behaving strangely:

```powershell
php artisan optimize:clear   # clears config, route, view, and event caches
composer dump-autoload
```

---

## 18. Command cheat sheet

```powershell
# Database (XAMPP MySQL — start MySQL in the Control Panel first)
C:\xampp\mysql\bin\mysql.exe -u root -e "CREATE DATABASE product_api CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
C:\xampp\mysql\bin\mysql.exe -u root -e "SHOW DATABASES;"
C:\xampp\mysql\bin\mysql.exe -u root product_api -e "SELECT * FROM products;"

# Project setup
laravel new product-api
cd product-api
php artisan install:api

# Generators
php artisan make:model Product --migration --controller --api --requests
php artisan make:resource ProductResource
php artisan make:factory ProductFactory --model=Product
php artisan make:test ProductApiTest
php artisan make:middleware CheckApiKey

# Database
php artisan migrate
php artisan migrate:status
php artisan migrate:rollback
php artisan migrate:fresh --seed

# Inspect
php artisan route:list --path=api
php artisan about
php artisan tinker

# Run and test
php artisan serve
php artisan test

# Clear caches
php artisan optimize:clear
```

---

## 19. Where to go next

Ordered roughly by how soon you'll need them:

1. **Authentication** — Sanctum is already installed. Issue a token with `$user->createToken('api')->plainTextToken`, then protect routes: `Route::apiResource(...)->middleware('auth:sanctum');`
2. **Authorization** — `php artisan make:policy ProductPolicy --model=Product`, then replace the `return true` in your Form Requests with real permission checks.
3. **Rate limiting** — `->middleware('throttle:60,1')` limits a route group to 60 requests per minute.
4. **Soft deletes** — add `use SoftDeletes;` to the model and `$table->softDeletes();` to a migration, so `DELETE` marks rather than destroys.
5. **Service layer** — once controller methods grow past a few lines, move business logic into an `app/Services/` class. Same reasoning as a `@Service` bean in Spring.
6. **Eager loading** — when products gain relationships, `Product::with('category')` avoids N+1 queries. Install Laravel Telescope or Debugbar to spot them.
7. **API versioning** — group routes under a `v1` prefix before you have external consumers: `Route::prefix('v1')->group(...)`.
8. **Documentation** — Scribe or Laravel OpenAPI generates interactive docs from your annotations.

### Reference links

- Laravel 13 docs: <https://laravel.com/docs/13.x>
- Eloquent ORM: <https://laravel.com/docs/13.x/eloquent>
- Validation rules: <https://laravel.com/docs/13.x/validation>
- API Resources: <https://laravel.com/docs/13.x/eloquent-resources>
- Sanctum: <https://laravel.com/docs/13.x/sanctum>
