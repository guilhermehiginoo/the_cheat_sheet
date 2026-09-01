---
title: Laravel
description: Practical reference for setting up a Laravel project, everyday Artisan commands, MVC structure, and fixing the errors beginners hit most.
tags:
  - php
  - web
  - backend
  - mvc
---

## Starting a Project

```bash
composer create-project laravel/laravel my-app   # create a new project
cd my-app

composer global require laravel/installer   # or, install the installer once
laravel new my-app                          # then create projects with it
```

```bash
composer install          # install PHP dependencies (vendor/ is gitignored)
cp .env.example .env      # create the local environment file
php artisan key:generate  # write APP_KEY into .env
php artisan migrate       # create the database tables
```

```bash
composer run dev    # server + queue worker + Vite, all in one terminal
```

- The four `composer install` → `migrate` steps above are what a fresh clone
  of an existing project needs.
- `.env` is **not** committed, which is why a cloned project has none.
- `APP_KEY` encrypts sessions and cookies. Without it, nothing boots.
- `composer run dev` (or `php artisan serve` alone) is for development only;
  production runs behind Nginx/Apache or FrankenPHP.
- A new project defaults to SQLite: `laravel new` already creates
  `database/database.sqlite` and runs the migrations for you.

## Starter Kits

`laravel new` prompts you to pick a starter kit: a normal Laravel app that
already ships authentication and a scaffolded frontend, fully inside your
project for you to edit.

- **React** — Inertia, React 19, TypeScript, Tailwind, [shadcn/ui](https://ui.shadcn.com). Frontend in `resources/js/`.
- **Vue** — Inertia, Vue 3 Composition API, TypeScript, Tailwind, shadcn-vue. Frontend in `resources/js/`.
- **Svelte** — Inertia, Svelte 5, TypeScript, Tailwind, shadcn-svelte. Frontend in `resources/js/`.
- **Livewire** — Livewire, Tailwind, [Flux UI](https://fluxui.dev). No JS framework, all Blade and PHP. Frontend in `resources/views/`.
- **None** — the plain skeleton, for learning the framework or building an API.

```bash
laravel new my-app                            # prompts for the kit
laravel new my-app --using=vendor/starter-kit # a community kit from Packagist
```

- The React, Vue, and Svelte kits use Inertia to keep server-side routing and
  controllers, with no separate API layer.
- **Teams** — an option in the same prompt. Adds team management screens and
  team-scoped routes (`/{current_team}/dashboard`).
- **WorkOS AuthKit** — another option. Swaps the built-in auth for a hosted
  provider with social login, passkeys, magic links, and SSO. Needs
  `WORKOS_CLIENT_ID`, `WORKOS_API_KEY`, and `WORKOS_REDIRECT_URL` in `.env`.

Every kit uses **Laravel Fortify** for authentication, configured in
`config/fortify.php`:

```php
use Laravel\Fortify\Features;

'features' => [
    Features::registration(),          // remove this line to close public sign-ups
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication(['confirm' => true]),
],
```

- Registration logic lives in `app/Actions/Fortify/CreateNewUser.php`.
- On the Inertia kits, disabling a feature also means removing the routes it
  used from your frontend components, or the build fails.
- **Breeze** and **Jetstream** were the previous generation of starter kits;
  current docs no longer list them, so use the kits above for new projects.
- This kit generation shipped in **Laravel 12**; the Svelte kit and
  Fortify-powered auth came in **Laravel 13**.

## Local Environment: Herd and Sail

`php artisan serve` assumes PHP, Composer, and a database are already
installed and configured. Herd and Sail remove that setup work.

### Herd (macOS / Windows)

```bash
herd park ~/Sites          # serve every project in this folder
herd link my-app           # serve a single project as my-app.test
herd php -v                # the PHP binary Herd manages
herd open                  # open the current project in the browser
```

- [Herd](https://herd.laravel.com) is a native app bundling PHP, Composer,
  and nginx. Projects in a parked directory are served automatically at
  `<folder>.test`.
- Switch PHP versions per project from the UI or `herd use php@8.3`.
- Herd bundles the `php`, `composer`, `laravel`, `node`, `npm`, and `nvm`
  binaries; that's why `node` may be missing from your `PATH` elsewhere.
- The paid tier adds databases, a mail catcher, and log viewing.

### Sail (Docker)

```bash
php artisan sail:install     # add Sail to an existing project
./vendor/bin/sail up -d      # start the containers
./vendor/bin/sail down       # stop them
```

```bash
php artisan serve   →  sail artisan serve
php artisan migrate →  sail artisan migrate
composer install    →  sail composer install
npm run dev         →  sail npm run dev
php artisan tinker  →  sail artisan tinker
```

- Wraps Docker Compose with PHP, MySQL, Redis, and more, so nothing is
  installed on the host. Prefix every command with `sail`.
- Shorten it with a shell alias: `alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'`.
- Running a bare `php artisan migrate` on the host is a common mistake; it
  targets the host's PHP, not the container's database.
- In `.env`, `DB_HOST` is the **service name** (`mysql`), not `127.0.0.1`.

Herd and Sail change *where* the commands run, not what they do. Everything
else in this cheat sheet is identical either way.

## Architecture: MVC

Laravel follows the **MVC** pattern. A request flows through the app in one
direction:

```text
Request → routes/web.php → Controller → Model (Eloquent) → Database
                              ↓
                        View (Blade) → Response
```

- **Route** (`routes/web.php`, `api.php`) — maps a URL and HTTP verb to code.
- **Controller** (`app/Http/Controllers/`) — receives the request, orchestrates, responds.
- **Model** (`app/Models/`) — represents a table, holds queries and relations.
- **View** (`resources/views/`) — Blade templates that render HTML.

```text
app/Http/Middleware/     # code that runs before/after every request
bootstrap/app.php        # app bootstrap: routing, middleware, exceptions
config/                  # configuration files, read from .env
database/migrations/     # versioned schema changes
database/seeders/        # sample/initial data
public/                  # web root, entry point (index.php) and built assets
resources/js|css/        # frontend source, compiled by Vite
routes/                  # route definitions
storage/                 # logs, cache, compiled views, uploaded files
```

- Keep controllers thin: validate, delegate, return. Business logic belongs
  in models or dedicated classes, not in the controller.

## Everyday Artisan Commands

```bash
php artisan serve                   # run the dev server
php artisan serve --port=8001       # on another port
php artisan tinker                  # interactive REPL with the app booted
php artisan route:list              # every registered route
php artisan route:list --path=user  # filtered
php artisan about                   # environment, versions, cache status
```

- `artisan` is the CLI. `php artisan list` shows everything; `php artisan help <command>` explains one.

### Generators

```bash
php artisan make:model Post -mcr    # model + migration + controller + resource routes
php artisan make:model Post -m      # model + migration
php artisan make:controller PostController --resource
php artisan make:migration create_posts_table
php artisan make:request StorePostRequest
php artisan make:middleware EnsureUserIsAdmin
php artisan make:seeder PostSeeder
php artisan make:factory PostFactory
php artisan make:command SyncPosts
```

### Database

```bash
php artisan migrate                 # run pending migrations
php artisan migrate:status          # what ran and what didn't
php artisan migrate:rollback        # undo the last batch
php artisan migrate:fresh --seed    # drop everything, re-migrate, seed
php artisan db:seed                 # run seeders only
```

- `migrate:fresh` **deletes all data**. Never run it against production.

### Cache and Maintenance

```bash
php artisan optimize:clear   # clear config, route, view and app caches at once
php artisan config:clear
php artisan cache:clear
php artisan view:clear
php artisan storage:link     # link public/storage → storage/app/public
php artisan queue:work       # process queued jobs
php artisan down             # maintenance mode
php artisan up
```

- In production, cache instead: `php artisan config:cache route:cache view:cache`.
- In development, keep them cleared, since cached config ignores `.env` changes.

## Routing

```php
// routes/web.php
use App\Http\Controllers\PostController;

Route::get('/', fn () => view('welcome'));

Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::post('/posts', [PostController::class, 'store']);
Route::get('/posts/{post}', [PostController::class, 'show']);
Route::put('/posts/{post}', [PostController::class, 'update']);
Route::delete('/posts/{post}', [PostController::class, 'destroy']);

// All seven CRUD routes at once
Route::resource('posts', PostController::class);

// Grouping
Route::middleware('auth')->prefix('admin')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

- Type-hinting the model (`show(Post $post)`) enables **route model binding**:
  Laravel fetches the record and 404s automatically if it doesn't exist.
- Use named routes and `route('posts.index')` instead of hardcoded URLs.
- `routes/web.php` has sessions and CSRF; `routes/api.php` is stateless.

## Controllers

```php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index()
    {
        return view('posts.index', ['posts' => Post::latest()->paginate(15)]);
    }

    public function show(Post $post)   // route model binding
    {
        return view('posts.show', compact('post'));
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required'],
        ]);

        $post = Post::create($data);

        return redirect()->route('posts.show', $post)->with('status', 'Post created!');
    }
}
```

## Eloquent

```php
// Read
Post::all();
Post::find(1);
Post::findOrFail(1);                       // throws 404 if missing
Post::where('published', true)->get();
Post::where('views', '>', 100)->orderBy('views', 'desc')->take(5)->get();
Post::first();
Post::count();

// Write
$post = Post::create(['title' => 'Hi', 'body' => '...']);   // needs $fillable
$post->update(['title' => 'Updated']);
$post->delete();

// Relations
class Post extends Model
{
    protected $fillable = ['title', 'body'];

    public function user()     { return $this->belongsTo(User::class); }
    public function comments() { return $this->hasMany(Comment::class); }
}

$post->comments;                       // lazy load
Post::with('comments')->get();         // eager load, avoids the N+1 problem
```

- `create()` and `update()` only assign fields listed in `$fillable`. A
  silently missing column is almost always a missing `$fillable` entry.
- Model `Post` maps to table `posts` by convention (plural, snake_case).

## Migrations

```php
// database/migrations/xxxx_create_posts_table.php
public function up(): void
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->string('title');
        $table->text('body');
        $table->boolean('published')->default(false);
        $table->timestamps();
    });
}

public function down(): void
{
    Schema::dropIfExists('posts');
}
```

- Never edit a migration that already ran in production; add a new one instead.
- Locally, `php artisan migrate:fresh` is the fast way to redo everything.

## Blade Templates

```blade
{{-- resources/views/posts/index.blade.php --}}
@extends('layouts.app')

@section('content')
    <h1>{{ $title }}</h1>          {{-- escaped output --}}
    {!! $trustedHtml !!}           {{-- raw output, only for content you trust --}}

    @if ($posts->isEmpty())
        <p>No posts yet.</p>
    @else
        @foreach ($posts as $post)
            <a href="{{ route('posts.show', $post) }}">{{ $post->title }}</a>
        @endforeach
    @endif

    @auth
        <p>Hello, {{ auth()->user()->name }}</p>
    @endauth

    <form method="POST" action="{{ route('posts.store') }}">
        @csrf
        <input name="title" value="{{ old('title') }}">
        @error('title') <span>{{ $message }}</span> @enderror
        <button>Save</button>
    </form>
@endsection
```

- Every non-GET form needs `@csrf`, otherwise the request is rejected with 419.
- Use `@method('PUT')` / `@method('DELETE')` for verbs HTML forms can't send.

## Validation

```php
$request->validate([
    'email'    => ['required', 'email', 'unique:users,email'],
    'password' => ['required', 'confirmed', 'min:8'],
    'age'      => ['nullable', 'integer', 'between:18,120'],
]);
```

```bash
php artisan make:request StorePostRequest
```

- On failure, Laravel redirects back automatically with errors and old
  input; no `try/catch` needed.
- For bigger rule sets, use a Form Request, generated above.

## Vite and Assets

```blade
{{-- In your layout's <head> --}}
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

```bash
npm run dev     # development: hot reload, must stay running
npm run build   # production: writes public/build/ + manifest.json
```

- In development, `npm run dev` runs **alongside** `php artisan serve`.
  `composer run dev` starts both, plus the queue worker, in one terminal.
- In production, or whenever the dev server isn't running, you must have
  run `npm run build` at least once.

## Common Errors

- `No application encryption key has been specified` — `APP_KEY` empty or
  `.env` missing. Fix: `cp .env.example .env && php artisan key:generate`.
- Blank page or a generic **500** — the real error is hidden. Set
  `APP_DEBUG=true` in `.env`, or read `storage/logs/laravel.log`.
- `SQLSTATE[HY000] [2002] Connection refused` — the database isn't running,
  or `DB_HOST`/`DB_PORT` in `.env` are wrong.
- `SQLSTATE[HY000] [1045] Access denied for user` — wrong `DB_USERNAME` /
  `DB_PASSWORD`. Fix them, then `php artisan config:clear`.
- `SQLSTATE[42S02] Base table or view not found` — migrations never ran.
  Fix: `php artisan migrate`.
- `Unable to open database file` (SQLite) — the file doesn't exist. Fix:
  `touch database/database.sqlite`.
- `Vite manifest not found at public/build/manifest.json` — assets were
  never built and the dev server is off. Fix: `npm run build`, or run
  `npm run dev`.
- Page loads but CSS/JS is missing — `npm run dev` stopped. Restart it, or
  run `npm run build`.
- **419** Page Expired — missing `@csrf` or an expired session. Add `@csrf`
  to the form, or clear cookies.
- **404** on a route you just added — cached routes, or the wrong verb.
  Run `php artisan route:clear`, then check `php artisan route:list`.
- `Target class [FooController] does not exist` — wrong namespace or a typo
  in the route. Import the controller and use `[FooController::class, 'method']`.
- `Class "App\Models\Post" not found` — autoloader out of date. Fix:
  `composer dump-autoload`.
- `.env` change has no effect — config is cached. Fix: `php artisan config:clear`
  (or `optimize:clear`).
- `The stream or file .../laravel.log could not be opened` — no write
  permission. Fix: `chmod -R 775 storage bootstrap/cache`.
- Uploaded images 404 in `/storage/...` — the symlink is missing. Fix:
  `php artisan storage:link`.
- `Add [title] to fillable property` — mass assignment blocked. Add the
  column to `$fillable` on the model.
- `Attempt to read property on null` — a query returned `null`. Use
  `findOrFail()`, or guard with `@if` / `?->`.
- `composer install` fails on `ext-...` — a PHP extension is missing.
  Install it (e.g. `php-mbstring`, `php-xml`, `php-curl`).
- Using Sail, `Connection refused` on `127.0.0.1` — the command ran on the
  host, or `DB_HOST` is wrong. Prefix with `sail`; set `DB_HOST=mysql`.

### The Debugging Order

```bash
tail -f storage/logs/laravel.log   # 1. read the actual error
php artisan optimize:clear         # 2. clear every cache
php artisan migrate:status         # 3. is the schema up to date?
php artisan route:list             # 4. does the route exist as you think?
php artisan about                  # 5. env, DB connection, cache state
```

- Most beginner 500s are one of three things: no `APP_KEY`, no database
  connection, or migrations that never ran.

## References

- [Laravel Documentation](https://laravel.com/docs)
- [Artisan Console](https://laravel.com/docs/artisan)
- [Eloquent ORM](https://laravel.com/docs/eloquent)
- [Blade Templates](https://laravel.com/docs/blade)
- [Asset Bundling (Vite)](https://laravel.com/docs/vite)
- [Starter Kits](https://laravel.com/docs/starter-kits)
- [Laravel Fortify](https://laravel.com/docs/fortify)
- [Laravel Herd](https://herd.laravel.com/docs)
- [Laravel Sail](https://laravel.com/docs/sail)
- [Laravel Bootcamp](https://bootcamp.laravel.com)
