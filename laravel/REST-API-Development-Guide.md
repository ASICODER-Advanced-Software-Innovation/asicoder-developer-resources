
# Laravel 11 REST API Development Guide

**Build RESTful APIs with PHP, Laravel, Eloquent ORM, JSON Responses and API Authentication**

Maintained by [ASICODER](https://www.asicoder.co.uk/).

## Introduction

Laravel is a PHP framework used to build web applications, backend services and REST APIs.

This guide demonstrates how to create a REST API using Laravel 11, including routing, controllers, database operations, request validation and consistent JSON responses.

The example uses a simple Product API.

## Requirements

- PHP 8.2 or later
- Laravel 11
- Composer
- MySQL or SQLite
- Postman, Insomnia or cURL
- Basic PHP knowledge

## 1. Create a Laravel Project

Install a new Laravel 11 application:

```bash
composer create-project laravel/laravel product-api "^11.0"

cd product-api
```

Configure your database in `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=product_api
DB_USERNAME=root
DB_PASSWORD=
```

Create the database before running migrations.

## 2. Enable API Routes

Laravel 11 does not include `routes/api.php` in a fresh installation by default.

Run:

```bash
php artisan install:api
```

This enables API routing and installs Laravel Sanctum.

API routes are automatically prefixed with `/api`.

## 3. Create Model, Migration and Controller

```bash
php artisan make:model Product -m

php artisan make:controller Api/ProductController --api
```

This generates:

- `app/Models/Product.php`
- A product migration
- `app/Http/Controllers/Api/ProductController.php`

## 4. Create Products Database Table

Open the generated product migration.

Inside the `Schema::create` callback, use:

```php
$table->id();
$table->string('name');
$table->text('description')->nullable();
$table->decimal('price', 10, 2);
$table->unsignedInteger('stock')->default(0);
$table->timestamps();
```

Run:

```bash
php artisan migrate
```

## 5. Configure the Product Model

Open `app/Models/Product.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    protected $fillable = [
        'name',
        'description',
        'price',
        'stock',
    ];
}
```

The `$fillable` array defines fields allowed for mass assignment.

## 6. Define REST API Routes

Open `routes/api.php`:

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\ProductController;

Route::apiResource('products', ProductController::class);
```

This registers the following endpoints:

| Method | Endpoint | Action |
|---|---|---|
| GET | `/api/products` | List products |
| GET | `/api/products/{product}` | Show product |
| POST | `/api/products` | Create product |
| PUT/PATCH | `/api/products/{product}` | Update product |
| DELETE | `/api/products/{product}` | Delete product |

**Security:** These routes are public for tutorial purposes. Protect write operations with authentication and authorization before using this API in production.

## 7. Build the Product API Controller

Open `app/Http/Controllers/Api/ProductController.php`:

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
        $products = Product::query()
            ->latest()
            ->paginate(10);

        return response()->json($products);
    }

    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'description' => ['nullable', 'string'],
            'price' => ['required', 'numeric', 'min:0'],
            'stock' => ['required', 'integer', 'min:0'],
        ]);

        $product = Product::create($validated);

        return response()->json([
            'message' => 'Product created successfully.',
            'data' => $product,
        ], 201);
    }

    public function show(Product $product): JsonResponse
    {
        return response()->json([
            'data' => $product,
        ]);
    }

    public function update(
        Request $request,
        Product $product
    ): JsonResponse {
        $validated = $request->validate([
            'name' => ['sometimes', 'required', 'string', 'max:255'],
            'description' => ['sometimes', 'nullable', 'string'],
            'price' => ['sometimes', 'required', 'numeric', 'min:0'],
            'stock' => ['sometimes', 'required', 'integer', 'min:0'],
        ]);

        $product->update($validated);

        return response()->json([
            'message' => 'Product updated successfully.',
            'data' => $product->fresh(),
        ]);
    }

    public function destroy(Product $product): JsonResponse
    {
        $product->delete();

        return response()->json([
            'message' => 'Product deleted successfully.',
        ]);
    }
}
```

Laravel automatically resolves the requested product using route model binding.

## 8. Start the Development Server

```bash
php artisan serve
```

Your API will be available at:

```text
http://127.0.0.1:8000/api/products
```

## 9. Test the API

### List products

```bash
curl -H "Accept: application/json" \
  http://127.0.0.1:8000/api/products
```

### Create a product

```bash
curl -X POST http://127.0.0.1:8000/api/products \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laravel Product Example",
    "description": "A sample API product",
    "price": 49.99,
    "stock": 10
  }'
```

### Update a product

```bash
curl -X PATCH http://127.0.0.1:8000/api/products/1 \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"price": 39.99}'
```

### Delete a product

```bash
curl -X DELETE http://127.0.0.1:8000/api/products/1 \
  -H "Accept: application/json"
```

These commands demonstrate local testing, not a hosted ASICODER API.

## 10. API Validation

Laravel validates incoming requests before database operations.

If the request contains invalid data and the client sends:

```http
Accept: application/json
```

Laravel returns a JSON validation response with HTTP status `422`.

Example:

```json
{
  "message": "The name field is required.",
  "errors": {
    "name": [
      "The name field is required."
    ]
  }
}
```

For larger applications, use Form Request classes to separate validation logic from controllers.

## 11. API Authentication with Laravel Sanctum

Laravel Sanctum supports API token authentication.

After enabling it with:

```bash
php artisan install:api
```

Protect the routes:

```php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\ProductController;

Route::middleware('auth:sanctum')
    ->apiResource('products', ProductController::class);
```

Requests must then be authenticated using a supported Sanctum authentication flow.

For token-based API clients, send an issued token in the Authorization header:

```http
Authorization: Bearer YOUR_API_TOKEN
```

**Important:** Authentication checks who is making the request. Authorization policies should also verify whether that user may create, update or delete a product.

Never expose production API tokens, application secrets or database credentials in a public GitHub repository.

## 12. REST API Security Best Practices

Before deploying a real API:

1. Enable HTTPS.
2. Use authentication for protected endpoints.
3. Apply authorization policies for sensitive actions.
4. Validate every incoming request.
5. Use rate limiting for abuse protection.
6. Configure CORS only for appropriate clients and origins.
7. Keep Laravel and dependencies updated.
8. Avoid exposing secrets in JSON responses.
9. Log errors without leaking sensitive information.
10. Add automated tests for API endpoints.

Do not deploy the tutorial CRUD routes publicly without appropriate access controls.

## 13. Useful Laravel Commands

```bash
php artisan route:list --path=api

php artisan migrate

php artisan make:request StoreProductRequest

php artisan make:resource ProductResource

php artisan test
```

## 14. Further Improvements

This example can be extended with:

- API Resources for consistent response formats
- Form Request validation
- Search and filtering
- Pagination and sorting
- Sanctum authentication
- Authorization policies
- Automated feature tests
- API versioning
- OpenAPI documentation
- Database indexing and query optimization

## Official Documentation

- [Laravel 11 Documentation](https://laravel.com/docs/11.x)
- [Laravel 11 Routing](https://laravel.com/docs/11.x/routing)
- [Laravel 11 Validation](https://laravel.com/docs/11.x/validation)
- [Laravel 11 Eloquent ORM](https://laravel.com/docs/11.x/eloquent)
- [Laravel 11 Sanctum](https://laravel.com/docs/11.x/sanctum)

## ASICODER Developer Resources

ASICODER provides custom software development, Laravel and PHP development, web applications, API integration, AI tools, automation and digital products.

**Explore our services:**

[ASICODER Software Development Services](https://www.asicoder.co.uk/our-service)

**Official website:**

https://www.asicoder.co.uk/

**Business contact:**

support@asicoder.co.uk

---

**Maintained by ASICODER — Advanced Software Innovation Coder**

This guide is an educational example, not a complete production-ready application.
