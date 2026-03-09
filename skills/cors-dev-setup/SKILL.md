---
name: cors-dev-setup
description: Set up or migrate a Pimcore project/bundle to use the cors/dev Docker Compose development environment
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Task
---

# CORS Dev Setup

You are helping set up or migrate a Pimcore project or bundle to use the `cors/dev` Docker Compose development environment.

## Overview

The `cors/dev` package provides a standardized Docker Compose setup for CORS Pimcore projects and bundles. Instead of maintaining a full `docker-compose.yaml` with all service definitions locally, projects include the shared `docker-compose-dev.yaml` from `cors/dev`.

## Step 1: Ensure cors/dev is in composer.json

Check that `cors/dev` is in `require-dev`:

```json
"require-dev": {
    "cors/dev": "12.x-dev"
}
```

If it's missing, add it.

Also ensure the `repositories` section includes the private packagist and disables the public one:

```json
"repositories": {
    "packagist.org": false,
    "private-packagist": {
        "type": "composer",
        "url": "https://repo.packagist.com/cors/"
    }
}
```

This is required because `cors/dev` and other CORS packages are hosted on the private packagist instance.

## Step 2: Replace docker-compose.yaml

Replace the existing `docker-compose.yaml` with the minimal include-based version:

```yaml
name: ${DOCKER_PROJECT_NAME:?error}
include:
  - vendor/cors/dev/docker-compose-dev.yaml
```

This replaces all manually defined services (php, php-debug, nginx, db, redis, supervisord, mercure, opensearch, rabbitmq, etc.) with the shared cors/dev definitions.

**Remove** from the old docker-compose.yaml:
- `version: '3.4'` or any version declaration (not needed with modern Docker Compose)
- All `services:` definitions (replaced by the include)
- All `networks:` definitions (`cors_dev` external network is defined in cors/dev)
- All `volumes:` definitions (defined in cors/dev)
- Hardcoded `container_name:` values
- Hardcoded image references like `git.e-conomix.at:5050/cors/docker/php-alpine-*`
- Traefik labels (handled by cors/dev using `DOCKER_PROJECT_NAME`)

## Step 3: Add Docker variables to .env

Add the Docker configuration variables to the **beginning** of the existing `.env` file. Do NOT replace existing entries - add the Docker block before them:

```env
###> docker ###
DOCKER_PROJECT_NAME=<project-name>
DOCKER_ALPINE_VERSION=3.22
DOCKER_PHP_VERSION=8.3
DOCKER_BASE_IMAGE=8.0-latest
###< docker ###
```

Keep all existing `.env` entries (`APP_ENV`, `APP_DEBUG`, `PIMCORE_KERNEL_CLASS`, etc.) as they are.

**Variable reference:**

| Variable | Description | Example |
|---|---|---|
| `DOCKER_PROJECT_NAME` | Kebab-case project identifier. Used for container names, traefik routing (`<name>.localhost`), network aliases | `cors-prometheus` |
| `DOCKER_PHP_VERSION` | PHP version for the Docker images | `8.3` or `8.4` |
| `DOCKER_BASE_IMAGE` | cors/docker base image version tag | `8.0-latest` |
| `DOCKER_ALPINE_VERSION` | Alpine Linux version for PHP images | `3.22` |

## Step 4: Remove obsolete files

### Files to DELETE

These files are now provided by `cors/dev` and must be removed:

- **`.docker/nginx.conf`** - cors/dev provides its own nginx config at `vendor/cors/dev/.docker/nginx.conf` with standardized upstream names (`fpm` / `fpm-debug` instead of project-specific names like `php-cors-prometheus`)
- **`.docker/php.ini`** - PHP config is baked into the cors/dev Docker images. Only keep if it has truly project-specific settings beyond the standard `upload_max_filesize`, `post_max_size`, `memory_limit`
- **`.docker/xdebug.ini`** - Xdebug configuration is handled by the cors/dev debug image
- **`.docker/composer-setup.sh`** or **`bin/composer-setup.sh`** - Initial composer install is handled differently with cors/dev (use `docker compose run php composer install` instead)

### Files to KEEP

- **`.docker/supervisord/*.conf`** - Supervisord worker configs are project-specific (define which messenger transports to consume). Keep these.
- **`.docker/rabbitmq.conf`** - If present and has project-specific RabbitMQ config, keep it

### How to check

Before deleting, verify the file only has default/standard content:

```bash
# Standard php.ini (safe to delete):
# upload_max_filesize = 512M
# post_max_size = 512M
# memory_limit = 512M

# Standard xdebug.ini (safe to delete):
# xdebug.mode=develop,debug
# xdebug.client_host=172.17.0.1
# xdebug.start_with_request=yes
```

## Step 5: Verify

After `composer update` (to install cors/dev), verify the setup works:

```bash
docker compose config  # validates the compose file
docker compose up -d   # starts all services
```

## What cors/dev provides

The shared `docker-compose-dev.yaml` includes these services:

| Service | Image | Purpose |
|---|---|---|
| `db` | mysql:8 | Database (alias: `database` on default network, `DOCKER_PROJECT_NAME` on cors_dev) |
| `redis` | redis:alpine | Cache/Session storage (password: `password`) |
| `php` | ghcr.io/cors-gmbh/pimcore-docker/php-fpm | PHP-FPM (alias: `fpm`) |
| `php-debug` | ghcr.io/cors-gmbh/pimcore-docker/php-fpm-debug | PHP-FPM with Xdebug (alias: `fpm-debug`) |
| `supervisord` | ghcr.io/cors-gmbh/pimcore-docker/php-supervisord | Background workers |
| `nginx` | nginx:stable-alpine | Web server with Traefik labels |
| `mercure` | dunglas/mercure | Real-time notifications (Pimcore Studio) |
| `opensearch` | opensearchproject/opensearch:2.13.0 | Search engine (Generic Data Index) |
| `opensearch-dashboards` | opensearchproject/opensearch-dashboards:2.13.0 | OpenSearch UI |
| `rabbitmq` | rabbitmq:3-management | Message queue (scale: 0 by default) |

### Networking

- All services are on a `default` network (project-scoped via `name:` in compose file)
- `cors_dev` external network is used for Traefik integration
- Database gets alias `DOCKER_PROJECT_NAME` on `cors_dev` network and `database` on default network
- Nginx uses Traefik labels with `DOCKER_PROJECT_NAME` for routing: `<name>.localhost` and `*.<name>.localhost`

### Nginx upstreams

cors/dev nginx config uses standardized upstream names:
- `fpm` (port 9000) - regular PHP-FPM
- `fpm-debug` (port 9000) - Xdebug-enabled PHP-FPM (activated via `XDEBUG_SESSION` cookie)

This is different from old setups that used project-specific names like `php-<project>` / `php-debug-<project>`.

### Install Environment

The `vendor/cors/dev/.install.env` provides default Pimcore installation parameters (used by php/php-debug via `env_file`):
- Admin user: `admin` / `admin`
- MySQL host: `database`
- MySQL user: `pimcore` / `pimcore`
- Database: `pimcore`

### Volumes

cors/dev defines these named volumes:
- `database` - MySQL data
- `redis` - Redis data
- `rabbitmq` - RabbitMQ data
- `opensearch` - OpenSearch data