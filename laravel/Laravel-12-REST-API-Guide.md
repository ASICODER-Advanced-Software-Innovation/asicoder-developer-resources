# Laravel 12 REST API Development Guide

**A practical Laravel 12 tutorial using PHP, Eloquent ORM, MySQL, validation and Sanctum.**

Maintained by **[ASICODER](https://www.asicoder.co.uk/)** — custom software development, Laravel development, API integration and developer resources.

> **Version:** Laravel 12.x requires PHP **8.2+**. This is a learning example: its write endpoints are public until authentication and authorization are configured. Do not expose them on a production server as-is.

## 1. Prerequisites

- PHP 8.2 or newer with Laravel's required extensions
- Composer
- MySQL, MariaDB, or SQLite
- Optional: Node.js and npm for frontend assets
- Postman or cURL for HTTP testing

Verify local versions:

```bash
php -v
composer --version
```

## 2. Create a Laravel 12 application

```bash
composer create-project laravel/laravel:^12.0 asicoder-api-12
cd asicoder-api-12
php artisan --version
```

This pins the **application skeleton** to the Laravel 12 major series rather than installing the latest major by accident.

## 3. Set up MySQL (optional)

Create database `asicoder_api_12`, then update `.env`:

```dotenv
APP_NAME="ASICODER Laravel 12 API"
APP_ENV=local
APP_DEBUG=true
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=asicoder_api_12
DB_USERNAME=root
DB_PASSWORD=
```

Use your own MySQL credentials. Keep `.env` out of Git.

```bash
php artisan migrate
```

## 4. Enable API routes and Sanctum

Fresh Laravel 12 installs do not necessarily have `routes/api.php` enabled. Run:

```bash
php artisan install:api
php artisan route:list --path=api
```

This installs Laravel Sanctum and enables stateless API routing under the `/api` prefix.

## 5. Create a Product model, migration, and API controller

```bash
php artisan make:model Product -m
php artisan make:controller Api/ProductController --api
```

In the generated `create_products_table` migration, replace the contents of the `Schema::create('products', function (Blueprint $table) { ... });` callback with:

```php
$table->id();
$table->string('name');
$table->text('description')->nullable();
$table->decimal('price', 10, 2);
$table->unsignedInteger('stock')->default(0);
$table->timestamps();
```

Then run:

```bash
php artisan migrate
```

In `app/Models/Product.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    protected $fillable = ['name', 'description', 'price', 'stock'];
}
```

## 6. Define REST routes

In `routes/api.php`, keep any existing framework-generated routes and add:

```php
use App\Http\Controllers\Api\ProductController;
use Illuminate\Support\Facades\Route;

Route::apiResource('products', ProductController::class);
```

| Method | URL | Operation |
|---|---|---|
| GET | `/api/products` | List products |
| GET | `/api/products/{product}` | Read product |
| POST | `/api/products` | Create product |
| PUT / PATCH | `/api/products/{product}` | Update product |
| DELETE | `/api/products/{product}` | Delete product |

## 7. Build the API controller

Replace `app/Http/Controllers/Api/ProductController.php` with:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Product;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    public function index(): JsonResponse
    {
        return response()->json(Product::query()->latest()->paginate(10));
    }

    public function store(Request $request): JsonResponse
    {
        $data = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string'],
            'price' => ['required', 'numeric', 'min:0'],
            'stock' => ['required', 'integer', 'min:0'],
        ]);

        return response()->json([
            'message' => 'Product created.',
            'data' => Product::create($data),
        ], 201);
    }

    public function show(Product $product): JsonResponse
    {
        return response()->json(['data' => $product]);
    }

    public function update(Request $request, Product $product): JsonResponse
    {
        $data = $request->validate([
            'name' => ['sometimes', 'required', 'string', 'max:255'],
            'description' => ['sometimes', 'nullable', 'string'],
            'price' => ['sometimes', 'required', 'numeric', 'min:0'],
            'stock' => ['sometimes', 'required', 'integer', 'min:0'],
        ]);

        $product->update($data);

        return response()->json([
            'message' => 'Product updated.',
            'data' => $product->fresh(),
        ]);
    }

    public function destroy(Product $product): JsonResponse
    {
        $product->delete();

        return response()->json(['message' => 'Product deleted.']);
    }
}
```

Route model binding retrieves a model for `{product}` or returns HTTP 404 when none matches. Invalid JSON requests that specify `Accept: application/json` receive HTTP 422 validation responses.

## 8. Test locally

```bash
php artisan serve
```

From another terminal:

```bash
curl -H "Accept: application/json" http://127.0.0.1:8000/api/products
```

Create a sample product:

```bash
curl -X POST http://127.0.0.1:8000/api/products \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"name":"Laravel API Demo","price":29.99,"stock":3}'
```

Modify an existing product (replace `1` with its real ID):

```bash
curl -X PATCH http://127.0.0.1:8000/api/products/1 \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"stock":5}'
```

## 9. Add authentication before production

For a token-authenticated API, move routes behind Sanctum's `auth:sanctum` middleware:

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('products', ProductController::class);
});
```

For personal-access-token clients, the relevant `User` model needs `Laravel\Sanctum\HasApiTokens`, and your authentication flow needs to issue tokens to authenticated users. A sample use of `createToken()` is **not** a complete login or account-verification implementation. API callers present an issued token via:

```http
Authorization: Bearer YOUR_ISSUED_TOKEN
Accept: application/json
```

Also add **policies / authorization** to control which users may manage products; authentication alone is insufficient. For first-party browser SPAs, consult Sanctum's stateful cookie authentication documentation rather than storing bearer tokens in browser storage.

## 10. Validation, JSON resources and tests

Useful artisan commands:

```bash
php artisan make:request StoreProductRequest
php artisan make:resource ProductResource
php artisan make:test ProductApiTest
php artisan test
php artisan route:list --path=api
```

For larger APIs, move validation rules to Form Request classes, use API Resources for stable JSON contracts, add pagination/filtering and write feature tests covering authorization, validation and CRUD operations.

## 11. Security and deployment checklist

- Enable HTTPS and set `APP_DEBUG=false` in production.
- Keep `.env` and secrets private; use environment-level secret storage.
- Authenticate sensitive routes and authorize each operation.
- Apply request validation and rate limiting.
- Restrict CORS to necessary origins.
- Use queues, database indexes and caching when appropriate.
- Add automated tests and monitor errors.
- Serve Laravel through the public web directory, never from your full project root.

## Laravel 12 official documentation

- [Installation](https://laravel.com/docs/12.x/installation)
- [Routing](https://laravel.com/docs/12.x/routing)
- [Validation](https://laravel.com/docs/12.x/validation)
- [Sanctum](https://laravel.com/docs/12.x/sanctum)
- [Deployment](https://laravel.com/docs/12.x/deployment)

## About ASICODER

ASICODER provides **custom PHP / Laravel development, React apps, API integration, AI tools, software projects and digital services**.

- [Website](https://www.asicoder.co.uk/)
- [Digital Services](https://www.asicoder.co.uk/our-service)
- **Email:** support@asicoder.co.uk

---

*Educational tutorial maintained by ASICODER. Validate and adapt the example to your project, framework patch version and security requirements.*
