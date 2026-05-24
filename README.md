# dokku-review-pg

A Dokku plugin that automatically provisions and destroys PostgreSQL databases using system PostgreSQL. Designed for review apps and ephemeral environments where each app needs one or more isolated databases.

Unlike [dokku-postgres](https://github.com/dokku/dokku-postgres) which runs a separate Docker container per database, this plugin uses a single system PostgreSQL instance — significantly reducing memory overhead when running multiple apps.

Supports Rails [multiple databases](https://guides.rubyonrails.org/active_record_multiple_databases.html) out of the box.

## Requirements

- Dokku 0.35+
- Ubuntu 22.04 or 24.04

PostgreSQL is installed automatically by the plugin (default: PostgreSQL 18 from the official PGDG repository).

## Installation

```bash
sudo dokku plugin:install https://github.com/tiramizoo/dokku-review-pg.git
```

To install with a specific PostgreSQL version:

```bash
sudo REVIEW_PG_VERSION=16 dokku plugin:install https://github.com/tiramizoo/dokku-review-pg.git
```

## Configuration

Configure the plugin using `dokku review-pg:set`:

```bash
# Required: secret used to derive deterministic per-app database passwords
dokku review-pg:set --global db-secret "$(openssl rand -hex 32)"

# Optional: which env vars to set (space-separated, default: DATABASE_URL)
# Useful for Rails multi-database setups (e.g. primary + cache + queue)
dokku review-pg:set --global databases "DATABASE_URL CACHE_DATABASE_URL QUEUE_DATABASE_URL"

# Optional: glob pattern for which apps trigger provisioning (default: *)
dokku review-pg:set --global app-pattern "review-pr-*"
```

View current configuration:

```bash
dokku review-pg:report
```

## How It Works

### On `dokku apps:create`

When an app is created that matches the `app-pattern`, the plugin automatically:

1. Creates a PostgreSQL user named after the app (e.g., `review_pr_42`)
2. Creates a database for each entry in `databases` (e.g., `review_pr_42`, `review_pr_42_cache`, `review_pr_42_queue`)
3. Sets the corresponding env vars on the app with full connection strings

```
$ dokku apps:create review-pr-42
-----> Creating review-pr-42...
-----> review-pg: Provisioning PostgreSQL for review-pr-42
       DATABASE_URL -> review_pr_42
       CACHE_DATABASE_URL -> review_pr_42_cache
       QUEUE_DATABASE_URL -> review_pr_42_queue
-----> Setting config vars
       DATABASE_URL:        postgres://review_pr_42:a1b2c3...@127.0.0.1:5432/review_pr_42
       CACHE_DATABASE_URL:  postgres://review_pr_42:a1b2c3...@127.0.0.1:5432/review_pr_42_cache
       QUEUE_DATABASE_URL:  postgres://review_pr_42:a1b2c3...@127.0.0.1:5432/review_pr_42_queue
```

These env vars can be referenced directly in `config/database.yml`:

```yaml
production:
  primary:
    url: <%= ENV["DATABASE_URL"] %>
  cache:
    url: <%= ENV["CACHE_DATABASE_URL"] %>
    migrations_paths: db/cache_migrate
  queue:
    url: <%= ENV["QUEUE_DATABASE_URL"] %>
    migrations_paths: db/queue_migrate
```

### On `dokku apps:destroy`

Databases and the PostgreSQL user are dropped automatically:

```
$ dokku apps:destroy review-pr-42 --force
-----> Destroying review-pr-42 (including all add-ons)
-----> review-pg: Dropping PostgreSQL resources for review-pr-42
       Dropped database: review_pr_42
       Dropped database: review_pr_42_cache
       Dropped database: review_pr_42_queue
       Dropped user: review_pr_42
```

## Database Naming

| `databases` entry | Database name |
|---|---|
| `DATABASE_URL` | `{app_name}` |
| `CACHE_DATABASE_URL` | `{app_name}_cache` |
| `QUEUE_DATABASE_URL` | `{app_name}_queue` |
| `ANALYTICS_DATABASE_URL` | `{app_name}_analytics` |

The suffix is derived from the env var name by stripping `_DATABASE_URL` and lowercasing. `DATABASE_URL` is the primary and gets no suffix.

## Password Derivation

Passwords are deterministic: `sha256(db-secret + ":" + app-name)`, truncated to 32 hex characters. This means:

- Same app name + same secret = same password (redeploys just work)
- No password storage needed
- Different apps get different passwords

## Commands

```bash
dokku review-pg:set <--global|app> <key> <value>   # Set a property
dokku review-pg:set <--global|app> <key>            # Delete a property
dokku review-pg:get <--global|app> <key>            # Get a property
dokku review-pg:report                              # Show configuration
```

## Properties

| Property | Scope | Default | Description |
|---|---|---|---|
| `db-secret` | global | *(none)* | Secret for password derivation. **Required.** |
| `databases` | global | `DATABASE_URL` | Space-separated list of env vars to provision |
| `app-pattern` | global | `*` | Glob pattern to match app names |

## Updating

```bash
sudo dokku plugin:update review-pg
```

## Upgrading PostgreSQL major versions

The plugin does not auto-migrate data when `REVIEW_PG_VERSION` changes. If a `main` cluster already exists at a different major version, the install hook refuses with an error rather than silently leaving you with an empty new cluster.

To upgrade in place (e.g. 17 → 18):

```bash
sudo systemctl stop postgresql
sudo apt-get install postgresql-18
sudo pg_upgradecluster 17 main
sudo pg_dropcluster 17 main --stop
sudo rm /var/lib/dokku/data/review-pg/.installed
sudo REVIEW_PG_VERSION=18 dokku plugin:update review-pg
```

**Take a backup first.** `pg_upgradecluster` uses `pg_upgrade` under the hood and is robust, but in-place migrations can fail in ways that leave the cluster unrecoverable without one.

Downgrades are not supported — PostgreSQL data files are forward-compatible only. To move to an older major, dump from the running cluster, drop it, install the older version, and restore.

## Debugging

Enable Dokku trace to see detailed plugin execution:

```bash
DOKKU_TRACE=1 dokku apps:create review-pr-99
```

## License

MIT - see [LICENSE](LICENSE).
