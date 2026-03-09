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

If using CoreShop packages (individual or full suite):
```bash
# CoreShop v4 → v5 (all coreshop/* packages)
# For x-dev packages: 4.1.x-dev → 5.0.x-dev
# For stable constraints: ^4.1 → ^5.0
composer require coreshop/theme-bundle:5.0.x-dev  # if used
composer require coreshop/menu-bundle:5.0.x-dev   # if used
composer require coreshop/messenger-bundle:5.0.x-dev  # if used
composer require coreshop/registry:^5.0            # if used
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
        instance_identifier: '%env(PIMCORE_INSTANCE_IDENTIFIER)%'
        product_key: '%env(PIMCORE_PRODUCT_KEY)%'
```

Import it in `config/config.yaml`:
```yaml
imports:
    - { resource: license.yaml }
```

Add to `.env` (or `.env.local`):
```
PIMCORE_ENCRYPTION_SECRET=your_secret_here
PIMCORE_INSTANCE_IDENTIFIER=your_instance_id_here
PIMCORE_PRODUCT_KEY=your_product_key_here
```

**Important:** `PIMCORE_ENCRYPTION_SECRET` must not be empty, otherwise you get:
```
`pimcore.encryption.secret` is not set.
Run `vendor/bin/generate-defuse-key` to generate a secret and set it as container parameter `pimcore.encryption.secret`.
```

Generate the secret with:
```bash
docker compose exec php vendor/bin/generate-defuse-key
```

Then set the output as `PIMCORE_ENCRYPTION_SECRET` in `.env.local`.

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

### A4b: Update .docker/composer-setup.sh

If the project has a `.docker/composer-setup.sh`, update the Docker image registry from the old GitLab registry to GitHub Container Registry:

**Before (Pimcore 11):**
```bash
docker run --rm --env COMPOSER_AUTH="$COMPOSER_AUTH" --volume "$(pwd)":/var/www/html:cached git.e-conomix.at:5050/cors/docker/php-alpine-${DOCKER_ALPINE_VERSION}-cli:${DOCKER_PHP_VERSION}-${DOCKER_BASE_IMAGE} composer install --no-scripts --no-interaction
```

**After (Pimcore 12):**
```bash
docker run --rm --env COMPOSER_AUTH="$COMPOSER_AUTH" --volume "$(pwd)":/var/www/html:cached ghcr.io/cors-gmbh/pimcore-docker/php-fpm:${DOCKER_PHP_VERSION}-alpine${DOCKER_ALPINE_VERSION}-${DOCKER_BASE_IMAGE} composer install --no-scripts --no-interaction
```

Key changes:
- Registry: `git.e-conomix.at:5050/cors/docker/` → `ghcr.io/cors-gmbh/pimcore-docker/`
- Image name format: `php-alpine-{ALPINE}-cli:{PHP}-{BASE}` → `php-fpm:{PHP}-alpine{ALPINE}-{BASE}`

### A5: Update Static Analysis & Code Quality Configs

Replace local configs with shared `cors/dev` imports:

**ecs.php:**
```php
<?php

declare(strict_types=1);

use Symplify\EasyCodingStandard\Config\ECSConfig;

return static function (ECSConfig $ecsConfig): void {
    $ecsConfig->import('vendor/cors/dev/ecs.php');
    $ecsConfig->parallel();
    $ecsConfig->paths(['src']);
};
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

### A7: Migrate Bundles from Kernel.php to config/bundles.php

Pimcore 11 projects often register bundles in `src/Kernel.php` via `registerBundlesToCollection()`. In Pimcore 12, migrate all bundles to `config/bundles.php` instead.

**Before (Pimcore 11 - src/Kernel.php):**
```php
class Kernel extends PimcoreKernel
{
    public function registerBundlesToCollection(BundleCollection $collection): void
    {
        $collection->addBundle(new SentryBundle(), 0, ['staging', 'prod']);
        $collection->addBundle(new CORSBundle());
        $collection->addBundle(new PimcoreAdminBundle(), 60);
        $collection->addBundle(new CoreShopCoreBundle(), 40);
    }
}
```

**After (Pimcore 12 - config/bundles.php):**
```php
<?php

return [
    Pimcore\Bundle\AdminBundle\PimcoreAdminBundle::class => ['all' => true],
    Sentry\SentryBundle\SentryBundle::class => ['staging' => true, 'prod' => true],
    CORS\Bundle\CORSBundle\CORSBundle::class => ['all' => true],
    CoreShop\Bundle\CoreBundle\CoreShopCoreBundle::class => ['all' => true],
];
```

**After (Pimcore 12 - src/Kernel.php):**
```php
class Kernel extends PimcoreKernel
{
}
```

Migration rules:
- Bundles with no environment restriction → `['all' => true]`
- Bundles with `['staging', 'prod']` → `['staging' => true, 'prod' => true]`
- Priority parameter from `addBundle()` is no longer needed (Symfony handles order via bundles.php)
- Remove all `use` imports and the `registerBundlesToCollection` method from Kernel.php
- Remove bundles for packages that were removed during the upgrade

### A8: Migrate Annotations to Attributes

Symfony 7 removed support for `annotation` route loaders. Update `config/routes.yaml`:

**Before:**
```yaml
app:
    resource: "../src/Controller/"
    type: annotation
```

**After:**
```yaml
app:
    resource: "../src/Controller/"
    type: attribute
```

Also update `config/services.yaml` if it references the old `Symfony\Component\Security\Core\Security` class (removed in Symfony 7):

**Before:**
```yaml
- '@Symfony\Component\Security\Core\Security'
```

**After:**
```yaml
- '@Symfony\Bundle\SecurityBundle\Security'
```

### A8b: Remove Deprecated Security Config Options

Symfony 7 removed `enable_authenticator_manager` (it's now always enabled). Remove it from `config/packages/security.yaml`:

**Before:**
```yaml
security:
    enable_authenticator_manager: true
```

**After:**
```yaml
security:
```

### A8c: Fix Serializer Interface Changes

Symfony 7 enforces strict signatures on `NormalizerInterface`. Update all custom normalizers:

**`normalize()` method:**
```php
// Before (Symfony 6):
public function normalize($object, string $format = null, array $context = [])

// After (Symfony 7):
public function normalize(mixed $data, ?string $format = null, array $context = []): \ArrayObject|array|string|int|float|bool|null
```

**`supportsNormalization()` method:**
```php
// Before (Symfony 6):
public function supportsNormalization($data, $format = null)

// After (Symfony 7):
public function supportsNormalization(mixed $data, ?string $format = null, array $context = []): bool
```

**New required method `getSupportedTypes()`:**
```php
public function getSupportedTypes(?string $format): array
{
    return [
        MyClass::class => true,
    ];
}
```

### A9: Pimcore Studio API Route Changes

If you have custom admin controllers, note the API prefix change:
- **Pimcore 11:** `/admin/...`
- **Pimcore 12:** `/pimcore-studio/api/...`

### A10: Deprecation Cleanup

Check for deprecations:
```bash
bin/console debug:container --deprecations
grep -rn "StaticRoutesBundle" config/ src/  # deprecated since 12.3
```

### A11: Clean Up and Run Migrations (LAST STEP)

**Important:** Only run migrations after everything else works (cache:clear, manual browser test).

1. **Delete all existing app migrations** in `src/Migrations/` - they reference removed bundles and use deprecated `ContainerAwareTrait` (removed in Symfony 7).

2. **Create a cleanup migration** that removes old entries from the `migration_versions` table:
```php
final class VersionXXXX extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Pimcore 12 upgrade: clean up old app migrations from migration_versions table';
    }

    public function up(Schema $schema): void
    {
        $this->addSql("DELETE FROM migration_versions WHERE version LIKE 'App\\\\Migrations\\\\%' AND version != 'App\\\\Migrations\\\\VersionXXXX'");
    }

    public function down(Schema $schema): void
    {
    }
}
```

3. **Run migrations** (Pimcore's own migrations handle data conversions like serialized PHP to JSON):
```bash
docker compose exec php bin/console doctrine:migrations:migrate --no-interaction
```

**Before running migrations, verify:**
- `docker compose exec php bin/console cache:clear` works without errors
- Manual test in browser shows no critical errors
- Ask the user for confirmation before executing

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

### B4b: Fix Symfony 7 Return Type Declarations

Symfony 7 enforces strict return type declarations. Methods that previously had no return type now require one:

**Command `execute()` method:**
```php
// Before (Pimcore 11 / Symfony 6):
protected function execute(InputInterface $input, OutputInterface $output)

// After (Pimcore 12 / Symfony 7):
protected function execute(InputInterface $input, OutputInterface $output): int
```

Search all Command classes and add `: int` return type to `execute()`:
```bash
grep -rl "protected function execute(InputInterface \$input, OutputInterface \$output)$" src/ --include="*.php"
```

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
| pimcore/admin-ui-classic-bundle | ^1.x | ^2.0 |
| pimcore optional bundles (file-explorer, google-marketing, newsletter, system-info, web-to-print) | ^1.x | ^2.0 |
| pimcore/ecommerce-framework-bundle | ^1.0 | removed (no P12 version) |
| pimcore/personalization-bundle | ^1.0 | removed (no P12 version) |
| coreshop/* (core-shop, theme-bundle, menu-bundle, messenger-bundle, registry, etc.) | 4.x / 4.1.x-dev | ^5.0 / 5.0.x-dev |
| cors/dev | dev-main / ^1.0@dev | 12.x-dev |
| cors/cors | ^0.2 | ^0.3.0 |
| cors/saml | ^11.0 | 12.x-dev |
| cors/web-care | ^11.0 | ^2.0 |
| cors/prometheus | ^11.0 | ^2.0 |
| cors/docker CI ref | 7.0 | 8.0 |
| phpstan/phpstan | ^1.10 | ^1.10 \|\| ^2.0 |
| vimeo/psalm | ^5.0 | ^5.0 \|\| ^6.0 |
| symfony/webpack-encore-bundle | ^1.17 | ^1.17 \| ^2.0 |

## Packages with NO Pimcore 12 Support (as of Feb 2026)

These packages must be removed when upgrading to Pimcore 12:

| Package | Notes |
|---|---|
| `dachcom-digital/dynamic-search` | Requires pimcore ^11.0 |
| `dachcom-digital/dynamic-search-data-provider-trinity` | Requires pimcore ^11.0 |
| `dachcom-digital/dynamic-search-index-provider-elasticsearch` | Requires pimcore ^11.0 |
| `dachcom-digital/emailizr` | Requires pimcore ^11.0 |
| `dachcom-digital/seo` | Requires pimcore ^11.0 |
| `cors/kubernetes-console` | Requires admin-ui-classic-bundle ^1.0 |

## Project require-dev Cleanup

When using `cors/dev:12.x-dev`, the following packages are bundled and should be **removed** from project-level `require-dev`:
- `phpstan/phpstan`
- `phpstan/phpstan-symfony`
- `symplify/easy-coding-standard`
- `vimeo/psalm`

## Replacing Removed Dachcom Bundles

### dachcom-digital/emailizr

The emailizr bundle provides CSS inlining for email templates via custom Twig tags (`{% emailizr_inline_style %}`, `{% end_emailizr_inline_style %}`), a `emailizr_style_collector` global, and an `emailizr_inline_style()` Twig function.

**Steps to replace:**

1. Install required dependencies:
```bash
composer require pelago/emogrifier twig/inky-extra
```

2. Create the following classes in `src/Emailizr/`:

**`src/Emailizr/Collector/CssCollector.php`** - Collects CSS file paths for inlining:
```php
<?php

declare(strict_types=1);

namespace App\Emailizr\Collector;

/**
 * @implements \IteratorAggregate<int, string>
 */
class CssCollector implements \IteratorAggregate
{
    /** @var list<string> */
    protected array $cssFiles = [];

    public function add(string $file): void
    {
        $this->cssFiles[] = $file;
    }

    public function removeAll(): void
    {
        $this->cssFiles = [];
    }

    /**
     * @return \ArrayIterator<int<0, max>, string>
     */
    #[\Override]
    public function getIterator(): \ArrayIterator
    {
        return new \ArrayIterator($this->cssFiles); // @phpstan-ignore return.type
    }
}
```

**`src/Emailizr/Parser/InlineStyleParser.php`** - Parses and inlines CSS into HTML:
```php
<?php

declare(strict_types=1);

namespace App\Emailizr\Parser;

use Pelago\Emogrifier\CssInliner;
use Pimcore\Http\Request\Resolver\EditmodeResolver;
use Symfony\Component\CssSelector\Exception\ParseException;

class InlineStyleParser
{
    public function __construct(
        protected EditmodeResolver $editmodeResolver,
    ) {
    }

    /**
     * @throws ParseException
     */
    public function parseInlineHtml(string $html = '', string $css = '', bool $onlyBodyContent = false): string
    {
        if ($this->editmodeResolver->isEditmode()) {
            return $html;
        }

        if ($html === '') {
            return '';
        }

        $inliner = CssInliner::fromHtml($html)->inlineCss($css);

        if ($onlyBodyContent) {
            $mergedHtml = $inliner->renderBodyContent();
        } else {
            $mergedHtml = $inliner->render();
        }

        $mergedHtml = preg_replace_callback('/"(%7B%7B%20)(.*)(%20%7D%7D)"/', static function (array $hit): string {
            return '"{{' . $hit[2] . '}}"';
        }, $mergedHtml) ?? $mergedHtml;

        return str_replace(["\r\n", "\r", "\n", "\t", '  ', '    ', '    '], '', $mergedHtml);
    }
}
```

**`src/Emailizr/Twig/Extension/InlineStyleExtension.php`** - Registers Twig globals, functions, and token parsers:
```php
<?php

declare(strict_types=1);

namespace App\Emailizr\Twig\Extension;

use App\Emailizr\Collector\CssCollector;
use App\Emailizr\Parser\InlineStyleParser;
use App\Emailizr\Twig\Parser\InlineStyleTokenParser;
use Symfony\Component\HttpKernel\Config\FileLocator;
use Twig\Extension\AbstractExtension;
use Twig\Extension\GlobalsInterface;
use Twig\TwigFunction;

class InlineStyleExtension extends AbstractExtension implements GlobalsInterface
{
    public function __construct(
        protected InlineStyleParser $inlineStyleParser,
        protected FileLocator $fileLocator,
    ) {
    }

    #[\Override]
    public function getGlobals(): array
    {
        return [
            'emailizr_inline_style_parser' => $this->inlineStyleParser,
            'emailizr_locator' => $this->fileLocator,
            'emailizr_style_collector' => new CssCollector(),
        ];
    }

    #[\Override]
    public function getFunctions(): array
    {
        return [
            new TwigFunction('emailizr_inline_style', [$this, 'includeStyles']),
        ];
    }

    #[\Override]
    public function getTokenParsers(): array
    {
        return [
            new InlineStyleTokenParser(),
        ];
    }

    public function includeStyles(CssCollector $styles): string
    {
        $style = '';
        foreach ($styles as $styleFile) {
            /** @var string $path */
            $path = $this->fileLocator->locate($styleFile);
            $contents = file_get_contents($path);
            if ($contents !== false) {
                $style .= "\n\n" . $contents;
            }
        }

        return $style;
    }
}
```

**`src/Emailizr/Twig/Node/InlineStyleNode.php`** - Twig node for compiled inline style blocks:
```php
<?php

declare(strict_types=1);

namespace App\Emailizr\Twig\Node;

use Twig\Compiler;
use Twig\Node\Node;

class InlineStyleNode extends Node
{
    public function __construct(
        Node $html,
        int $line = 0,
        string $tag = 'inline_style',
    ) {
        parent::__construct(['html' => $html], [], $line, $tag);
    }

    #[\Override]
    public function compile(Compiler $compiler): void
    {
        $compiler
            ->write(sprintf('$inlineCssFiles = "";%s', \PHP_EOL))
            ->write(sprintf('foreach($context["emailizr_style_collector"] as $cssFile) {%s', \PHP_EOL))
            ->indent()
            ->write(sprintf('$path = $context["emailizr_locator"]->locate($cssFile);%s', \PHP_EOL))
            ->write(sprintf('if ($path) {%s', \PHP_EOL))
            ->indent()
            ->write(sprintf('$inlineCssFiles .= "\n".file_get_contents($path);%s', \PHP_EOL))
            ->outdent()
            ->write(sprintf('}%s', \PHP_EOL))
            ->outdent()
            ->write(sprintf('}%1$s%1$s', \PHP_EOL))
            ->write(sprintf('ob_start();%s', \PHP_EOL))
        ;

        $compiler->subcompile($this->getNode('html'));

        $compiler
            ->write(sprintf('$inlineHtml = ob_get_clean();%1$s%1$s', \PHP_EOL))
            ->write(sprintf('echo $context["emailizr_inline_style_parser"]->parseInlineHtml($inlineHtml, $inlineCssFiles);%s', \PHP_EOL))
            ->write(sprintf('$context["emailizr_style_collector"]->removeAll();%1$s%1$s', \PHP_EOL))
        ;
    }
}
```

**`src/Emailizr/Twig/Parser/InlineStyleTokenParser.php`** - Token parser for the `{% emailizr_inline_style %}` tag:
```php
<?php

declare(strict_types=1);

namespace App\Emailizr\Twig\Parser;

use App\Emailizr\Twig\Node\InlineStyleNode;
use Twig\Error\SyntaxError;
use Twig\Token;
use Twig\TokenParser\AbstractTokenParser;

class InlineStyleTokenParser extends AbstractTokenParser
{
    public const TAG = 'emailizr_inline_style';

    /**
     * @throws SyntaxError
     */
    #[\Override]
    public function parse(Token $token): InlineStyleNode
    {
        $parser = $this->parser;
        $stream = $parser->getStream();
        $stream->expect(Token::BLOCK_END_TYPE);
        $html = $this->parser->subparse([$this, 'decideEnd'], true);
        $stream->expect(Token::BLOCK_END_TYPE);

        return new InlineStyleNode($html, $token->getLine(), $this->getTag());
    }

    #[\Override]
    public function getTag(): string
    {
        return self::TAG;
    }

    public function decideEnd(Token $token): bool
    {
        return $token->test(sprintf('end_%s', self::TAG));
    }
}
```

3. Register the service in `config/services.yaml`:
```yaml
App\Emailizr\:
    resource: '../src/Emailizr'
```

4. Remove `dachcom-digital/emailizr` from `composer.json` if still present.

**Note:** This uses `ob_start()`/`ob_get_clean()` for Twig <3.9 compatibility. The original emailizr used `CaptureNode` which requires Twig 3.9+.

### dachcom-digital/seo

The `seo_update_metadata` Twig function comes from `dachcom-digital/seo` (not the Pimcore SEO bundle). Replace usage with Pimcore's built-in SEO helpers or remove the calls if not needed.

### dachcom-digital/formbuilder

The `form_builder_static` Twig function comes from `dachcom-digital/formbuilder`. Replace with Symfony form rendering or a custom implementation.

## Running Tests After Upgrade

After completing the upgrade, verify everything works:

```bash
# Twig template syntax
docker compose exec php bin/console lint:twig themes

# Static analysis
docker compose exec php vendor/bin/phpstan analyse src
docker compose exec php vendor/bin/psalm

# Code style
docker compose exec php vendor/bin/ecs check src

# Symfony container
docker compose exec php bin/console lint:container

# Cache clear
docker compose exec php bin/console cache:clear
```

## Docker Compose Setup

Projects using `cors/dev` should run commands via:
```bash
docker compose exec php <command>
```

For example:
```bash
docker compose exec php composer update
docker compose exec php bin/console cache:clear
```
