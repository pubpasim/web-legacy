# PUB website (legacy)

The original website of **PUB (Pemberdayaan Umat Berkelanjutan)**, the free-tuition and dormitory scholarship at Universitas Nasional PASIM Bandung, built by earlier PUB students. Live at **https://pubpasim.org**.

> [!NOTE]
> This site is in maintenance mode. New development happens in the new website ([new.pubpasim.org](https://new.pubpasim.org)), which takes over these pages step by step. Change this repo only for security fixes, bug fixes, or content that has not moved yet.

## What it does

**Public pages**: PUB profile and profile video, pembina profile, legal documents (legalitas), PUB activities, board structures (PUB, PPMB, alumni association), PPMB schedules (school visits, TPA, survey, final interview) and their documentation, selection results, downloadable registration form, monthly infaq, learning material, alumni listing, and a question form.

**Admin area** (after login): data management for students and alumni, batches (angkatan), majors, schools, regions, periods, PUB/PPMB organisation, schedules, selection results, trainings, documentation, infaq, terms, and user accounts.

Login levels: `admin`, `admin_pub`, `admin_ppmb`, `ikatan_alumni` (all land on the admin area) and `alumni` (only their own profile, education, experience, trainings, and activities).

## Stack

- PHP 8.5 (composer requires `^8.4`), Laravel 13, Blade views
- Vite + Tailwind CSS 3 for assets
- Database: PostgreSQL in production (the image also ships the MySQL/MariaDB PDO driver)
- Queue: Laravel database queue, worked by supervisor in the container
- Tests: Pest 4

## Project layout

```
app/Http/Controllers/   one controller per module (mostly Indonesian names, e.g. mahasiswaController, jad_tpaController)
app/Http/Middleware/    RequireLogin, RestrictAlumni (see "Access control")
app/Models/             Eloquent models over the tb_* tables
resources/views/        Blade views: user/ (public), admin/, alumni_admin/, ppmb/, layouts/, components/
routes/web.php          all routes
docker/                 nginx, php-fpm, php.ini, supervisor, and entrypoint for the production image
tests/Feature/          access-control tests
public/file, public/imgs  uploaded files (persistent volumes in production)
```

## Access control

Both middlewares are appended to the `web` group in `bootstrap/app.php` and **fail closed**:

- **`RequireLogin`** requires a logged-in session for every route. A new public page must be added to `PUBLIC_PATHS` in `app/Http/Middleware/RequireLogin.php`, or it will redirect to the login page.
- **`RestrictAlumni`** limits the `alumni` level to the routes listed in `RULES` (`app/Http/Middleware/RestrictAlumni.php`), and only for their own records. A new alumni route must be added there with the ids it has to match.

Other rules:

- Delete actions use `Route::delete()` (the `<x-delete-button>` component). Never add a GET route that changes data; the tests check for this.
- Login regenerates the session only after the password check passes.

## Running locally

Requirements: PHP 8.4+ with Composer, Node 24 with npm, and a database. The migrations in `database/migrations` do **not** build the current schema from scratch, so work against a restored copy of the database rather than `php artisan migrate`. Never point a local `.env` at production.

```sh
composer install
npm ci
cp .env.example .env
php artisan key:generate
# set DB_CONNECTION / DB_HOST / DB_PORT / DB_DATABASE / DB_USERNAME / DB_PASSWORD to your local copy
composer dev          # php artisan serve + queue listener + Vite
```

`php artisan route:list` currently fails because some routes point at controllers that no longer exist; this does not affect serving pages.

## Tests

The tests use an in-memory SQLite database and need no external services:

```sh
cp .env.example .env && php artisan key:generate   # once
vendor/bin/pest
```

Without a local PHP, run them in a container from a throwaway copy:

```sh
podman run --rm -v "$PWD":/src:ro docker.io/library/composer:2 sh -c \
  'cp -a /src /tmp/app && cd /tmp/app && rm -rf vendor .env && composer install -q &&
   cp .env.example .env && php artisan key:generate -q && vendor/bin/pest'
```

## Deployment

Pushes to `main` are deployed automatically by Dokploy, which builds the `Dockerfile`:

- Official `php:<PHP_VERSION>-fpm` image + nginx (nginx.org stable) + supervisor running nginx, php-fpm, and 12 queue workers.
- To patch PHP or nginx, bump `ARG PHP_VERSION` / `ARG NGINX_VERSION` in the `Dockerfile` and push.
- The container refuses to start without `APP_KEY`, and its health check calls Laravel's `/up` route.
- `public/file` and `public/imgs` are mounted from the host. Anything else written under `public/` is lost on redeploy.

`nixpacks.toml` is a leftover from the earlier Nixpacks build and is not used by the current deployment.

## Contributing

- Keep credentials, `.env`, and database dumps out of git.
- Run the tests before pushing, since a push goes straight to production.
- Write new code, commit messages, and comments in English.
