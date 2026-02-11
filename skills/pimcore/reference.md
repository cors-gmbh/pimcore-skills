# Pimcore Reference — Directory Structures & Common Patterns

## Bundle Directory Structure

```
MyBundle/
├── MyBundle.php                    # Bundle class (extends AbstractPimcoreBundle)
├── DependencyInjection/
│   ├── MyBundleExtension.php       # Service loading
│   ├── Configuration.php           # Config tree builder
│   └── Compiler/                   # Compiler passes
│       └── MyRegistryPass.php
├── Resources/
│   ├── config/
│   │   ├── services.yaml           # Service definitions
│   │   ├── doctrine/               # Entity mappings (.orm.xml)
│   │   ├── serializer/             # JMS serializer configs
│   │   └── pimcore/
│   │       └── routing.yaml        # Route definitions
│   ├── translations/
│   │   ├── admin.en.yml            # Admin UI (ExtJS) translations
│   │   └── studio.en.yaml          # Studio v2 (React) translations
│   ├── public/                     # Public assets (JS, CSS, images)
│   └── views/                      # Twig templates
├── Controller/
│   └── Admin/                      # Admin controllers
├── CoreExtension/                  # Custom Pimcore field types
├── EventListener/                  # Event subscribers
├── Installer/
│   └── Installer.php               # Bundle installer
├── Model/                          # Bundle-specific models
├── Repository/                     # Doctrine repositories
└── assets/pimcore-studio/          # Studio v2 React frontend
    ├── package.json
    └── src/
        └── main.ts
```

## Data Object Lifecycle

```
              ┌──────────────┐
              │   PRE_ADD     │  Before first save
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │   POST_ADD    │  After first save (has ID)
              └──────┬───────┘
                     │
              ┌──────▼───────┐
  ┌──────────►│  POST_LOAD    │  After loading from DB
  │           └──────┬───────┘
  │                  │
  │           ┌──────▼───────┐
  │           │  PRE_UPDATE   │  Before each save
  │           └──────┬───────┘
  │                  │
  │           ┌──────▼───────┐
  └───────────│ POST_UPDATE   │  After each save
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │  PRE_DELETE   │  Before deletion
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │ POST_DELETE   │  After deletion
              └──────────────┘
```

## Element Service Utilities

```php
use Pimcore\Model\Element\Service;

// Get any element by type and ID
$element = Service::getElementById('object', 42);
$element = Service::getElementById('asset', 15);
$element = Service::getElementById('document', 3);

// Get element type string
$type = Service::getElementType($element); // 'object', 'asset', 'document'

// Path utilities
$idPath = Service::getIdPath($element);     // /1/5/42
$typePath = Service::getTypePath($element); // /folder/subfolder/my-object
```

## Data Object Field Type Implementation

```php
namespace Pimcore\Model\DataObject\ClassDefinition\Data;

// All field types implement these interfaces:
interface ResourcePersistenceAwareInterface {
    // How data is stored in the database
    public function getDataForResource($data, $object = null, $params = []);
    public function getDataFromResource($data, $object = null, $params = []);
}

interface QueryResourcePersistenceAwareInterface {
    // How data is stored in the query table (for search/filter)
    public function getDataForQueryResource($data, $object = null, $params = []);
}

interface TypeDeclarationSupportInterface {
    // PHP type declarations for generated classes
    public function getParameterTypeDeclaration(): ?string;
    public function getReturnTypeDeclaration(): ?string;
    public function getPhpdocInputType(): ?string;
    public function getPhpdocReturnType(): ?string;
}
```

## Service Registration Patterns

### Tagged Services
```yaml
services:
    # Auto-tag all implementations of an interface
    _instanceof:
        MyBundle\Handler\HandlerInterface:
            tags: ['my_bundle.handler']

    # Manual tagging
    MyBundle\Handler\SpecificHandler:
        tags:
            - { name: 'my_bundle.handler', type: 'specific' }
```

### Compiler Pass for Tag Collection
```php
class HandlerRegistryPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->has('my_bundle.handler_registry')) {
            return;
        }

        $definition = $container->findDefinition('my_bundle.handler_registry');
        $taggedServices = $container->findTaggedServiceIds('my_bundle.handler');

        foreach ($taggedServices as $id => $tags) {
            foreach ($tags as $attributes) {
                $definition->addMethodCall('register', [
                    $attributes['type'],
                    new Reference($id),
                ]);
            }
        }
    }
}
```

## Common Pimcore Configuration

```yaml
# config/config.yaml
pimcore:
    # General settings
    general:
        domain: '%env(PIMCORE_DOMAIN)%'
        redirect_to_maindomain: false
        language: en

    # Document settings
    documents:
        default_controller: 'App\Controller\DefaultController::default'
        allow_trailing_slash: 'no'
        generate_preview: false

    # Asset settings
    assets:
        frontend_prefixes:
            source: ''
            thumbnail: ''
            thumbnail_deferred: ''

    # Data Object settings
    objects:
        class_definitions:
            data:
                map:
                    # Map custom field types
                    myCustomField: \MyBundle\CoreExtension\MyCustomField

    # Security
    security:
        password:
            algorithm: auto

    # Cache
    full_page_cache:
        enabled: false
```

## Translation Usage

### PHP (Backend)
```php
// In controllers (extends AbstractController)
$this->translator->trans('my_key', [], 'admin');

// In services
public function __construct(private TranslatorInterface $translator) {}
$this->translator->trans('my_key');
```

### Twig (Frontend)
```twig
{{ 'my_key'|trans }}
{{ 'my_key'|trans({}, 'admin') }}
```

### Translation File Formats
```yaml
# admin.en.yml (ExtJS admin)
my_key: "My Translation"

# studio.en.yaml (Studio v2 React)
my_key: "My Translation"
```

## Pimcore CLI Commands

```bash
# Installation
bin/console pimcore:install

# Class definitions
bin/console pimcore:deployment:classes-rebuild   # Rebuild class definitions
bin/console pimcore:deployment:class-export      # Export to JSON

# Cache
bin/console pimcore:cache:clear                  # Clear Pimcore cache
bin/console cache:clear                          # Clear Symfony cache

# Maintenance
bin/console pimcore:maintenance                  # Run maintenance tasks
bin/console pimcore:thumbnails:image             # Generate image thumbnails

# Migrations
bin/console doctrine:migrations:migrate          # Run DB migrations

# Debug
bin/console debug:container                      # List services
bin/console debug:router                         # List routes
bin/console debug:event-dispatcher               # List event listeners
```

## Object Brick Pattern

```php
// Definition: Create via admin UI or definition file
// Usage in code:
$product = DataObject\Product::getById(42);

// Get brick container
$bricks = $product->getSpecialAttributes(); // ObjectBrick container

// Get specific brick
$colorBrick = $bricks->getColorAttributes();
if ($colorBrick) {
    $color = $colorBrick->getColor();
}

// Set brick
$brick = new DataObject\Objectbrick\Data\ColorAttributes($product);
$brick->setColor('red');
$bricks->setColorAttributes($brick);
$product->save();
```

## Fieldcollection Pattern

```php
$product = DataObject\Product::getById(42);

// Get fieldcollections
$features = $product->getFeatures(); // Fieldcollection instance

// Iterate
foreach ($features->getItems() as $item) {
    if ($item instanceof DataObject\Fieldcollection\Data\Feature) {
        echo $item->getName();
    }
}

// Add new item
$feature = new DataObject\Fieldcollection\Data\Feature();
$feature->setName('Weight');
$feature->setValue('1.5kg');
$features->add($feature);
$product->save();
```

## Localized Fields

```php
$product = DataObject\Product::getById(42);

// Get in specific locale
$name = $product->getName('en');
$name = $product->getName('de');

// Set in specific locale
$product->setName('English Name', 'en');
$product->setName('Deutscher Name', 'de');
$product->save();

// Get all locales
$localizedFields = $product->getLocalizedfields();
$allData = $localizedFields->getItems();
```