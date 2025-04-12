# Laravel Multilang Routes

[![Latest Version](https://img.shields.io/packagist/v/alexwaha/laravel-multilang-routes.svg?style=flat-square)](https://packagist.org/packages/alexwaha/laravel-multilang-routes)
[![Laravel Versions](https://img.shields.io/badge/laravel-10%20%7C%2011%20%7C%2012-blue?style=flat-square)](https://laravel.com/)
[![PHP Version](https://img.shields.io/badge/php-8.2%2B-blue?style=flat-square)](https://www.php.net/)
[![Downloads](https://img.shields.io/packagist/dt/alexwaha/laravel-multilang-routes.svg?style=flat-square)](https://packagist.org/packages/alexwaha/laravel-multilang-routes)
[![License](https://img.shields.io/packagist/l/alexwaha/laravel-multilang-routes.svg?style=flat-square)](LICENSE)

Simple and lightweight package to localize your Laravel routes.

## Features

- Localized versions of your routes without complex configurations
- Seamless support for route groups, slugs, and middleware
- Automatically switches URLs based on the active locale
- Redirects `/fallback_locale/...` to `/...` with 301 if needed
- Works with Laravel 10, 11, and 12

## Installation

Install the package via Composer:

```bash
composer require alexwaha/laravel-multilang-routes
```

Publish package config file:
```bash
php artisan vendor:publish --tag=multilang-localize-config
```

This will publish the config file to `config/multilang-routes.php`:

You can customize the list of supported languages and their URL slugs.

## Usage

Define your routes using the `localizedRoutes` macro.

```php
use Illuminate\Support\Facades\Route;

Route::localizedRoutes(function () {
    Route::get('/', function () {
        return view('welcome');
    });
});
```

Or routes with middlewares, for ex.: `web`
```php
use Illuminate\Support\Facades\Route;

Route::localizedRoutes(function () {
    Route::get('/', [HomeController::class, 'index'])->name('home');
}, ['web']);
```
Or named routes with prefix
```php
Route::localizedRoutes(function () {
    Route::name('blog.')->prefix('/blog')->group(function () {
        Route::get('', [BlogController::class, 'index'])->name('index');
        Route::get('/{post}', [BlogController::class, 'show'])->name('show');
    });
}, ['web']);
```

This automatically generates localized routes like:

- `/en/about`
- `/es/about`
- `/de/about`

>The slug in the URL is required for the `setLocale` middleware, which changes the application language based on the `locale` slug.

### Blade usage

Generate a localized URL in Blade templates using `Route::localize()` method:

```php
<a href="{{ Route::localize('about') }}">About</a>
```

Or generate URLs with parameters:

```php
<a href="{{ Route::localize('blog.show', ['post' => $post]) }}">View Post</a>
```

### Middleware: `localize.setLocale`
You must register the `localize.setLocale` middleware to automatically detect and set the application locale based on the first segment of the URL.

There are two ways to apply the middleware:

➊ Apply Middleware Globally
You can apply the middleware to a route group manually:
```php
Route::middleware(['localize.setLocale'])->group(function () {
    Route::localizedRoutes(function () {
        Route::get('/', function () {
            return view('welcome');
        });
    });
});
```
This method gives you full manual control over the middleware stack for your localized routes.

➋ Apply Middleware Directly in `localizedRoutes`
You can pass middleware as the second argument to the localizedRoutes macro:

```php
Route::localizedRoutes(function () {
    Route::get('/', function () {
        return view('welcome');
    });
}, ['web', 'localize.setLocale']);
```
This is a simpler and more compact approach, especially useful for smaller projects.

➌ Pagination-Aware Route Generation
When building localized URLs with pagination, you can define your routes like this:
```php
use Illuminate\Support\Facades\Route;

Route::localizedRoutes(function () {
    Route::name('blog.')->prefix('/blog')->group(function () {
        Route::get('', [BlogController::class, 'index'])->name('index');
        Route::get('/page/{page?}', [BlogController::class, 'index'])->name('paginated');
        Route::get('/{post}', [BlogController::class, 'show'])->name('show');
    });
}, ['web', 'localize.setLocale']);

```

If the segment does not match any configured slug, the application will fall back to the default locale.

### Default Locale Redirection

If a user visits a URL with the default fallback locale (e.g., `/en/about` when `en` is the fallback), they will be automatically redirected with a **301 Permanent Redirect** to the non-sluged version (`/about`).

This ensures clean URLs for your default language.


### Configuration

You can configure which languages are available and their corresponding URL slugs inside the `config/multilang-routes.php` file.

Each language entry should contain:

| Key    | Description                             | Example                              |
|--------|-----------------------------------------|--------------------------------------|
| locale | Application locale `(app()->setLocale)` | en,es,de                             |
| slug   | URL slug for routes                     | en,es,de \| english, spanish, german |

#### Example:
```php
return [
    'language_provider' => [
        'class' => \Alexwaha\Localize\LanguageProvider::class,
        'params' => [
            'languages' => [
                [
                    'locale' => 'en',
                    'slug' => 'en',
                ],
                [
                    'locale' => 'es',
                    'slug' => 'es',
                ],
            ],
        ],
    ],
];
```
You can customize the list of supported languages and their URL prefixes or create your own Language Provider.

If you want full control over how languages are loaded (for example, from a database), you can create a custom provider by implementing the `LanguageProviderInterface` and specify your class in the configuration.

Example `config/multilang-routes.php`:
```php
return [
    'language_provider' => [
        'class' => App\Providers\CustomLanguageProvider::class,
    ],
];
```
Your custom provider should implement:
```php
interface LanguageProviderInterface
{
    public function getLanguages(): array;
    public function getLocaleBySegment(string $segment): string;
}
```
This gives you the flexibility to load languages from the database, API, or any other source.

Example of `app/Providers/DatabaseLanguageProvider.php`

```php
<?php

namespace App\Providers;

use App\Contracts\LanguageProviderInterface;
use App\Contracts\LanguageInterface;
use App\Models\Language;

class DatabaseLanguageProvider implements LanguageProviderInterface
{
    /**
     * @return array<LanguageInterface>
     */
    public function getLanguages(): array
    {
        $languages = Language::all();

        return array_map(function (Language $language) {
            return new \Alexwaha\Localize\Language(
                $language->locale,
                $language->slug,
                $language->is_default
            );
        }, $languages->all());
    }

    public function getLocaleBySegment(string $segment): string
    {
        $language = Language::where('slug', $segment)->first();

        return $language?->locale ?? config('app.fallback_locale');
    }
}
```

## Requirements

- PHP >= 8.2
- Laravel 10.x, 11.x, or 12.x

## License

This package is open-sourced software licensed under the [MIT license](LICENSE).

## Credits

Developed by [Alex Waha](https://alexwaha.com)

---

Feel free to submit issues or contribute to this project!
