# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Lake is a single-file bash bootstrap script (`lake.setup`) that provisions a full Laravel + FrankenPHP development environment with zero host-level dependencies (no PHP, no Docker, no Composer required).

## Commands

After running `./lake.setup` once, the project is set up. Daily commands all go through the `.lake/` shims:

```bash
# Bootstrap (run once in an empty directory)
./lake.setup
FRANKEN_VERSION=1.11.2 LAKE_PORT=9000 ./lake.setup   # with overrides

# Start dev server (FrankenPHP + queue worker + log viewer + Vite)
.lake/composer run dev

# Artisan
.lake/php artisan <command>

# Composer
.lake/composer require <package>
.lake/composer install

# Tests
.lake/composer run test

# Reset
./lake.setup --clean        # Remove everything except lake.setup and .claude
./lake.setup --clean-app    # Remove Laravel files only; keep .lake/ (~170 MB of binaries)
```

## Architecture

The entire bootstrap logic lives in `lake.setup`. There is no build step — it is a standalone bash script.

**Key design decisions:**

- `.lake/php` is a shim that strips `-d` flags before delegating to `frankenphp php-cli`. This is necessary because Composer always passes `-d allow_url_fopen=1` and similar flags that FrankenPHP's php-cli mode does not accept.
- `.lake/composer` sets `PHP_BINARY` and prepends `.lake/` to `PATH` before invoking `frankenphp php-cli composer.phar`. This ensures `@php` hooks in Composer scripts resolve to the shim.
- `laravel new` is run into a temporary subdirectory (`.laravel_setup_tmp/`) and then moved to the project root — because the installer requires an empty directory.
- The `composer.json` dev script is patched post-install: `php artisan serve` → `.lake/frankenphp run` (FrankenPHP uses a `Caddyfile`; it does not support PHP's built-in `-S` server).
- SQLite is the default database. A `database/database.sqlite` file is created and `migrate` is run automatically.

**Known limitation:** The `PHP_BINARY` constant is empty under FrankenPHP php-cli mode. Packages that spawn PHP sub-processes via `PHP_BINARY` (e.g. Laravel Boost) will fail at that step, but the app runs normally otherwise.
