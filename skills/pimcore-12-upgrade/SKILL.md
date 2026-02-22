---
name: pimcore-12-upgrade
description: Upgrade Pimcore 11 projects and bundles to Pimcore 12 - composer, config, PHP 8.3, Symfony 7, Docker, CI/CD, cors/dev, cors/saml
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Task
---

# Pimcore 11 to Pimcore 12 Upgrade

You are helping upgrade a Pimcore 11 codebase to Pimcore 12. First determine whether this is a **Project** or a **Bundle**, as the upgrade steps differ.

## Step 0: Determine Project vs Bundle

**Project** = A full Pimcore application (has `bin/console`, `config/`, `public/index.php`, `docker-compose.yaml`).
**Bundle** = A reusable Symfony/Pimcore bundle (has no `bin/console`, extends `AbstractPimcoreBundle`, published as composer package).

Check for indicators:

```bash
# Project indicators
ls bin/console config/bundles.php public/index.php docker-compose.yaml 2>/dev/null

# Bundle indicators
grep -r "AbstractPimcoreBundle\|AbstractBundle" src/ --include="*.php" -l
```

---

## Part A: Upgrade Steps for PROJECTS

### A1: Update PHP Requirement

Pimcore 12 requires PHP 8.3+.

**composer.json:**
```json
"php": ">=8.3"
```

**Docker:** Update PHP image to 8.3 or 8.4.

### A2: Update composer.json Dependencies

```bash
composer require pimcore/pimcore:^12.0
composer require pimcore/admin-ui-classic-bundle:^2.0
```

If using CORS packages:
```bash
composer require cors/dev:12.x-dev --dev
composer require cors/saml:12.x-dev  # if SAML is used
composer require cors/cors:^0.3.0    # if cors-bundle is used, check latest version
```

Symfony version support changes:
```json
"symfony/dotenv": "^6.4 | ^7.4",
"symfony/runtime": "^6.4 | ^7.4"
```

### A3: Pimcore License Configuration

Create `config/license.yaml`:
```yaml
pimcore:
    encryption:
        secret: '%env(PIMCORE_ENCRYPTION_SECRET)%'
    product_registration:
        instance_identifier: '<your-instance-id>'
        product_key: '%env(PIMCORE_PRODUCT_KEY)%'
```

Add to `.env` (or `.env.local`):
```
PIMCORE_ENCRYPTION_SECRET=your_secret_here
PIMCORE_PRODUCT_KEY=your_product_key_here
```

### A4: Docker Compose Updates

Pimcore 12 introduces new required services. Update `docker-compose.yaml`:

**Add these services:**
- `mercure` - Real-time notifications (required for Pimcore Studio)
- `opensearch:2.13.0` - Search engine (replaces Elasticsearch for Generic Data Index)
- `opensearch-dashboards:2.13.0` - OpenSearch UI (optional, for debugging)
- `rabbitmq:3-management` - Message queue

**Remove if present (enterprise-only):**
- `gotenberg`
- `pdfreactor`

If using `cors/dev`, update docker-compose include:
```yaml
include:
  - vendor/cors/dev/docker-compose-dev.yaml
```

### A5: Update Static Analysis & Code Quality Configs

Replace local configs with shared `cors/dev` imports:

**ecs.php:**
```php
<?php

declare(strict_types=1);

use Symplify\EasyCodingStandard\Config\ECSConfig;

return ECSConfig::configure()
    ->withSets([
        __DIR__ . '/vendor/cors/dev/ecs.php',
    ])
    ->withPaths([
        __DIR__ . '/src',
    ])
    ->withParallel()
;
```

**phpstan.neon:**
```neon
includes:
    - vendor/cors/dev/phpstan.neon

parameters:
    paths:
        - src
```

**psalm.xml:**
```xml
<?xml version="1.0"?>
<psalm xmlns:xi="http://www.w3.org/2001/XInclude">
    <xi:include
            href="vendor/cors/dev/psalm.xml"
            xpointer="xmlns(ps=https://getpsalm.org/schema/config)xpointer(/psalm/*[not(self::projectFiles)])"
    />
    <projectFiles>
        <directory name="src" />
        <ignoreFiles>
            <directory name="vendor" />
        </ignoreFiles>
    </projectFiles>
</psalm>
```

### A6: Update CI/CD Pipeline

Update `.gitlab-ci.yml` to reference the correct `cors/docker` version:
```yaml
include:
  - project: 'cors/docker'
    ref: "8.0"
    file: '.project-gitlab-ci.yml'
```

### A7: Pimcore Studio API Route Changes

If you have custom admin controllers, note the API prefix change:
- **Pimcore 11:** `/admin/...`
- **Pimcore 12:** `/pimcore-studio/api/...`

### A8: Deprecation Cleanup

Check for deprecations:
```bash
bin/console debug:container --deprecations
grep -rn "StaticRoutesBundle" config/ src/  # deprecated since 12.3
```

---

## Part B: Upgrade Steps for BUNDLES

Bundles follow a simpler process because they don't manage Docker, database, or application config directly.

### B1: Update composer.json

```json
{
  "require": {
    "php": ">=8.3",
    "pimcore/pimcore": "^11.0 | ^12.0"
  },
  "require-dev": {
    "cors/dev": "12.x-dev",
    "pimcore/admin-ui-classic-bundle": "^2.0"
  }
}
```

Key points:
- Support both `^11.0 | ^12.0` if possible for backwards compatibility
- `cors/dev` goes to `12.x-dev` (not `dev-main`)
- `pimcore/admin-ui-classic-bundle` bumps to `^2.0`
- Add Symfony 7 support: `"symfony/*": "^6.4 | ^7.4"`

### B2: Update CI/CD Pipeline

Bundle pipelines use a different CI template than projects:
```yaml
include:
  - project: 'cors/docker'
    ref: "8.0"
    file: '.bundle-gitlab-ci.yml'
```

Note: `.bundle-gitlab-ci.yml` (not `.project-gitlab-ci.yml`).

### B3: Update Static Analysis Configs

Same as project (Step A5) - use shared `cors/dev` imports for `ecs.php`, `phpstan.neon`, `psalm.xml`.

**Important for psalm.xml:** Make sure `<projectFiles>` is defined locally (not just inherited via xi:include), otherwise `vendor/bin/psalm` without arguments won't analyze any files. See Step A5 for the correct config.

### B4: Add `#[\Override]` Attributes

PHP 8.3 introduces `#[\Override]`. Add it to all methods that override a parent class or implement an interface method:

```php
final class MyBundle extends AbstractPimcoreBundle
{
    #[\Override]
    public function getNiceName(): string { return 'My Bundle'; }

    #[\Override]
    public function getDescription(): string { return 'Description'; }
}
```

Common methods that need `#[\Override]`:
- `getNiceName()`, `getDescription()` on bundle classes
- `load()` on DependencyInjection Extension classes
- `getConfigTreeBuilder()` on Configuration classes
- `getSubscribedEvents()` on EventSubscriber classes
- `getType()` on Document Editable classes
- `compile()` on Twig Node classes
- `parse()`, `getTag()` on Twig TokenParser classes
- `getTokenParsers()` on Twig Extension classes

### B5: Add `final` to Classes

Make classes `final` unless they are explicitly designed for extension:

```php
final class MyListener implements EventSubscriberInterface { ... }
final class MyExtension extends Extension { ... }
final class MyController extends UserAwareController { ... }
```

### B6: Add `declare(strict_types=1)`

Every PHP file must have `declare(strict_types=1);` after the opening `<?php` tag:

```php
<?php

declare(strict_types=1);
```

### B7: Update File Headers

Update from the old dual-license format to the GPL-only format:

**Old (Pimcore 11):**
```php
/**
 * CORS GmbH.
 *
 * This source file is available under two different licenses:
 * - GNU General Public License version 3 (GPLv3)
 * - Pimcore Commercial License (PCL)
 * ...
 */
```

**New (Pimcore 12):**
```php
/*
 * CORS GmbH
 *
 * This software is available under the GNU General Public License version 3 (GPLv3).
 *
 * @copyright  Copyright (c) CORS GmbH (https://www.cors.gmbh)
 * @license    https://www.cors.gmbh/license GPLv3
 */
```

### B8: Fix Common Psalm/PHPStan Issues

After upgrading, run static analysis and fix:

```bash
vendor/bin/psalm
vendor/bin/phpstan analyse
vendor/bin/ecs check src
```

Common fixes needed:
- `MissingOverrideAttribute` → add `#[\Override]` (see B4)
- `ClassMustBeFinal` → add `final` (see B5)
- `UnnecessaryVarAnnotation` → remove redundant `@var` annotations
- `UnusedVariable` → remove or use the variable
- `RiskyTruthyFalsyComparison` → use strict comparison (`!== false`, `> 0`)
- `UnusedClass` / `PossiblyUnusedMethod` → suppress in psalm.xml (false positives for DI-based bundles)
- `MissingConstructor` from vendor code → suppress in psalm.xml

Suppress bundle-typical false positives in `psalm.xml`:
```xml
<issueHandlers>
    <UnusedClass errorLevel="info" />
    <PossiblyUnusedMethod errorLevel="info" />
    <PossiblyUnusedParam errorLevel="info" />
    <MissingConstructor errorLevel="info" />
</issueHandlers>
```

---

## Quick Reference: Version Matrix

| Dependency | Pimcore 11 | Pimcore 12 |
|---|---|---|
| PHP | >=8.1 | >=8.3 |
| Symfony | ^6.4 | ^6.4 \| ^7.4 |
| pimcore/pimcore | ^11.0 | ^12.0 |
| pimcore/admin-ui-classic-bundle | ^1.3 | ^2.0 |
| cors/dev | dev-main | 12.x-dev |
| cors/saml | ^11.0 | 12.x-dev |
| cors/docker CI ref | varies | "8.0" |
| phpstan/phpstan | ^1.10 | ^1.10 \|\| ^2.0 |
| vimeo/psalm | ^5.0 | ^5.0 \|\| ^6.0 |
