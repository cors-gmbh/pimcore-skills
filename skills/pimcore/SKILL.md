---
name: pimcore
description: Pimcore platform development - bundles, data objects, class definitions, CoreExtensions, events, workflows, documents, assets
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Edit
  - Write
  - Task
---

# Pimcore Platform Development

You are helping develop on the Pimcore platform. Before writing any code, load context about Pimcore's architecture and patterns.

## Step 1: Understand Pimcore's Three Pillars

Pimcore has three fundamental element types, all extending `Pimcore\Model\Element\AbstractElement`:

1. **Assets** — File management (images, videos, PDFs). Uses League Flysystem for storage.
2. **Documents** — Web pages, emails, links, snippets. Template-based rendering with editables.
3. **Data Objects** — Structured data (products, customers, etc.). Defined by Class Definitions.

Every element has: `id`, `path`, `key`, `creationDate`, `modificationDate`, `userOwner`, `properties`, `dependencies`.

## Step 2: Bundle Architecture

### AbstractPimcoreBundle
All Pimcore bundles extend `Pimcore\Extension\Bundle\AbstractPimcoreBundle` (not plain Symfony `Bundle`):

```php
use Pimcore\Extension\Bundle\AbstractPimcoreBundle;

class MyBundle extends AbstractPimcoreBundle
{
    public function getNiceName(): string { return 'My Bundle'; }
    public function getDescription(): string { return 'Description'; }
    public function getInstaller(): ?InstallerInterface { return $this->container->get(Installer::class); }
}
```

### DependencyInjection Pattern
```php
class MyBundleExtension extends ConfigurableExtension implements PrependExtensionInterface
{
    public function loadInternal(array $config, ContainerBuilder $container): void
    {
        $loader = new YamlFileLoader($container, new FileLocator(__DIR__.'/../config'));
        $loader->load('services.yaml');
    }
}
```

### Compiler Passes
Used heavily for tag collection, registry building, service decoration:
```php
public function build(ContainerBuilder $container): void
{
    parent::build($container);
    $container->addCompilerPass(new MyRegistryPass());
}
```

## Step 3: Data Object Class Definitions

Class Definitions define the structure of Data Objects (like database schemas).

### Field Types (`Pimcore\Model\DataObject\ClassDefinition\Data\*`)

**Basic**: `Input`, `Textarea`, `Wysiwyg`, `Numeric`, `Slider`, `Date`, `DateTime`, `Checkbox`, `Select`, `Multiselect`, `Email`, `Country`, `Language`

**Relations**: `ManyToOneRelation`, `ManyToManyRelation`, `AdvancedManyToManyRelation`

**Complex**: `Block` (repeating groups), `Fieldcollection`, `Localizedfields` (i18n), `ObjectBrick` (extendable), `Classificationstore` (dynamic attributes), `QuantityValue`

### Custom Field Types (CoreExtensions)
Add custom field types to the Class Definition editor:

```php
namespace MyBundle\CoreExtension;

use Pimcore\Model\DataObject\ClassDefinition\Data\Select;

class MyCustomField extends Select
{
    public string $fieldtype = 'myCustomField';

    public function getFieldType(): string {
        return $this->fieldtype;
    }
}
```

The `$fieldtype` string must match the frontend dynamic type ID exactly.

## Step 4: Event System

Pimcore uses Symfony EventDispatcher with predefined event constants:

```php
use Pimcore\Event\DataObjectEvents;
use Pimcore\Event\Model\DataObjectEvent;

class MySubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            DataObjectEvents::POST_UPDATE => 'onPostUpdate',
            DataObjectEvents::PRE_DELETE => 'onPreDelete',
        ];
    }
}
```

**Event types per element**: `PRE_ADD`, `POST_ADD`, `PRE_UPDATE`, `POST_UPDATE`, `PRE_DELETE`, `POST_DELETE`, `POST_LOAD`, `PRE_COPY`, `POST_COPY`

**Event classes**: `AssetEvents`, `DataObjectEvents`, `DocumentEvents`

## Step 5: Workflow System

Built on Symfony Workflow Component, configured via YAML:

```yaml
pimcore:
    workflows:
        product_approval:
            enabled: true
            type: state_machine
            supports:
                - Pimcore\Model\DataObject\Product
            places: [draft, review, approved]
            transitions:
                submit_for_review:
                    from: draft
                    to: review
```

## Step 6: Pimcore Studio v2

The new admin UI is React/TypeScript with:
- **Backend**: `pimcore/studio-backend-bundle` — REST API (OpenAPI)
- **Frontend**: `pimcore/studio-ui-bundle` — React app with InversifyJS DI container
- **Real-time**: Symfony Mercure for live updates
- **Search**: `pimcore/generic-data-index-bundle`

### Plugin System
Studio plugins register via `IAbstractPlugin`:
```typescript
const plugin: IAbstractPlugin = {
  name: 'my-plugin',
  onInit() { /* DI bindings, dynamic types */ },
  onStartup({ moduleSystem }) { /* module/widget registration */ }
}
```

### Dynamic Types (Frontend)
Custom field types need frontend registration:
```typescript
import { DynamicTypeObjectDataAbstractSelect } from '@pimcore/studio-ui-bundle/modules/element'

export class DynamicTypeMyField extends DynamicTypeObjectDataAbstractSelect {
  readonly id = 'myCustomField' // Must match PHP $fieldtype
}
```

## Step 7: Configuration

Main config via `config/config.yaml`:
```yaml
pimcore:
    general:
        domain: "example.com"
    documents:
        default_controller: 'App\Controller\DefaultController::default'
    objects:
        class_definitions:
            data:
                map: {}
```

## Step 8: Caching

```php
use Pimcore\Cache;

// Core cache (Redis/Memcached/Filesystem)
Cache::save($data, 'my_key', ['tag1', 'tag2'], 3600);
Cache::load('my_key');
Cache::clearTag('tag1');

// In-memory runtime cache (current request only)
use Pimcore\Cache\RuntimeCache;
RuntimeCache::save('key', $data);
```

## Step 9: Installer Pattern

```php
use Pimcore\Extension\Bundle\Installer\AbstractInstaller;

class Installer extends AbstractInstaller
{
    public function install(): void { /* SQL migrations, permissions, configs */ }
    public function uninstall(): void { /* cleanup */ }
    public function isInstalled(): bool { /* check state */ }
}
```

## Step 10: Before Committing

- Validate YAML: `bin/console lint:yaml src`
- Validate Twig: `bin/console lint:twig src`
- Validate container: `bin/console lint:container`
- Clear cache: `bin/console cache:clear`

## Reference

See `.claude/skills/pimcore/reference.md` for directory structure templates and common patterns.