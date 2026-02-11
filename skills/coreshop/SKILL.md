---
name: coreshop
description: Extend and customize CoreShop eCommerce - custom rules, payment gateways, entity extensions, workflows, Studio v2 plugins, notifications
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Task
---

# CoreShop Extension & Customization Guide

You are helping a developer extend or customize CoreShop, a Pimcore-based eCommerce platform. This skill covers how to build on top of CoreShop — adding custom rules, payment gateways, extending entities, customizing workflows, building Studio v2 plugins, etc.

## Step 1: Understand the Extension Model

CoreShop is designed to be extended at every layer:

- **PHP Backend**: Service decoration, tagged services, event listeners, entity extension
- **Rule Engine**: Custom conditions/actions for cart, product, shipping, notification rules
- **Payment**: Custom payment gateways via Payum
- **Workflows**: Extend order/shipment/payment state machines via YAML
- **Studio v2 Frontend**: 7 extension types + custom rule components + dynamic types
- **Templates/Controllers**: Override frontend controllers and Twig templates

### Key Principle: Tagged Services

CoreShop uses Symfony service tags extensively. Most extensions register via tags:

```yaml
# config/services.yaml
App\MyCondition:
    tags:
        - { name: coreshop.cart_price_rule.condition, type: my_condition, form-type: App\Form\Type\MyConditionType }
```

## Step 2: Extending Entities

### Extend a Doctrine Entity

Find the current class, create your extension, configure it:

```bash
# Find current model class
php bin/console debug:container --parameter=coreshop.model.currency.class
```

```php
namespace App\Entity;

use CoreShop\Component\Core\Model\Currency as BaseCurrency;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'coreshop_currency')]
class Currency extends BaseCurrency
{
    #[ORM\Column(type: 'boolean')]
    private bool $customFlag = false;

    public function getCustomFlag(): bool { return $this->customFlag; }
    public function setCustomFlag(bool $flag): void { $this->customFlag = $flag; }
}
```

```yaml
# config/config.yaml
core_shop_currency:
    resources:
        currency:
            classes:
                model: App\Entity\Currency
```

```yaml
# config/jms_serializer/Currency.yaml
App\Entity\Currency:
    exclusion_policy: ALL
    properties:
        customFlag:
            expose: true
            type: bool
            groups: [Detailed]
```

Then run migrations:
```bash
php bin/console doctrine:migrations:diff
php bin/console doctrine:migrations:migrate
```

### Register a Custom Pimcore Data Object

```yaml
# config/config.yaml
core_shop_resource:
    pimcore:
        app.my_entity:
            classes:
                model: Pimcore\Model\DataObject\MyEntity
                interface: CoreShop\Component\Resource\Model\ResourceInterface
```

## Step 3: Custom Rule Conditions & Actions

### Backend (PHP)

**Condition Checker:**
```php
namespace App\CoreShop\Rule\Condition;

use CoreShop\Component\Order\Cart\Rule\Condition\AbstractConditionChecker;
use CoreShop\Component\Resource\Model\ResourceInterface;
use CoreShop\Component\Rule\Model\RuleInterface;

final class MinimumItemCountCondition extends AbstractConditionChecker
{
    public function isValid(
        ResourceInterface $subject,
        RuleInterface $rule,
        array $configuration,
        array $params = []
    ): bool {
        return count($subject->getItems()) >= $configuration['min_items'];
    }
}
```

**Form Type:**
```php
namespace App\CoreShop\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\IntegerType;
use Symfony\Component\Form\FormBuilderInterface;

final class MinimumItemCountConditionType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add('min_items', IntegerType::class);
    }
}
```

**Register via service tag:**
```yaml
App\CoreShop\Rule\Condition\MinimumItemCountCondition:
    tags:
        - name: coreshop.cart_price_rule.condition
          type: minimum_item_count
          form-type: App\CoreShop\Form\Type\MinimumItemCountConditionType
```

### Available Rule Tags

| Tag | Rule Engine |
|-----|------------|
| `coreshop.cart_price_rule.condition` | Cart Price Rules |
| `coreshop.cart_price_rule.action` | Cart Price Rules |
| `coreshop.product_price_rule.condition` | Product Price Rules |
| `coreshop.product_price_rule.action` | Product Price Rules |
| `coreshop.product_specific_price_rule.condition` | Product Specific Prices |
| `coreshop.product_specific_price_rule.action` | Product Specific Prices |
| `coreshop.shipping_rule.condition` | Shipping Rules |
| `coreshop.shipping_rule.action` | Shipping Rules |
| `coreshop.notification_rule.condition` | Notification Rules |
| `coreshop.notification_rule.action` | Notification Rules |

### Frontend — Studio v2 (React)

```typescript
import React from 'react'
import { Form, InputNumber } from 'antd'
import { useTranslation } from 'react-i18next'
import type { ConditionComponentProps } from '@coreshop/rule/src/rules'

export const MinimumItemCountCondition: React.FC<ConditionComponentProps> = ({ data, onChange }) => {
  const { t } = useTranslation()

  return (
    <Form layout="vertical">
      <Form.Item label={t('minimum_items')}>
        <InputNumber
          value={data.min_items || 1}
          onChange={(v) => onChange({ ...data, min_items: v || 1 })}
          min={1}
          style={{ width: '100%' }}
        />
      </Form.Item>
    </Form>
  )
}
```

**Register in your plugin's `main.ts`:**
```typescript
import { type IAbstractPlugin, container } from '@pimcore/studio-ui-bundle'
import type { ConditionRegistry } from '@coreshop/rule/src/rules/registry'
import { coreshopOrderServiceIds } from '@coreshop/order'
import { MinimumItemCountCondition } from './conditions/MinimumItemCountCondition'

const plugin: IAbstractPlugin = {
  name: 'my-coreshop-plugin',

  onInit() {
    const registry = container.get<ConditionRegistry>(
      coreshopOrderServiceIds.cartPriceRuleConditionRegistry
    )
    registry.register('minimum_item_count', MinimumItemCountCondition)
  }
}

export default plugin
```

## Step 4: Custom Payment Gateways

CoreShop uses **Payum** for payments. To add a gateway:

**1. Configuration Form Type:**
```php
namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;

final class MyGatewayConfigurationType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('api_key', TextType::class)
            ->add('secret', TextType::class);
    }
}
```

**2. Register:**
```yaml
App\Form\Type\MyGatewayConfigurationType:
    tags:
        - { name: coreshop.gateway_configuration_type, type: my_gateway }
        - { name: form.type }
```

**3. Configure Payum factory:**
```yaml
# config/packages/payum.yaml
payum:
    gateways:
        my_gateway:
            factory: my_gateway
```

## Step 5: Extending Workflows

Add states and transitions to order/shipment/payment workflows:

```yaml
# config/config.yaml
core_shop_workflow:
    state_machine:
        coreshop_shipment:
            places:
                - reviewed          # New state
            transitions:
                review:             # New transition
                    from: [new, ready]
                    to: reviewed
            place_colors:
                reviewed: '#2f819e'
            transition_colors:
                review: '#2f819e'
            callbacks:
                after:
                    notify_after_review:
                        on: ['review']
                        do: ['@App\Listener\ShipmentReviewListener', 'onReview']
                        args: ['object']
```

**Add translations:**
```yaml
# translations/admin.en.yml
coreshop_workflow_transition_coreshop_shipment_review: 'Review'
coreshop_workflow_state_coreshop_shipment_reviewed: 'Reviewed'
```

## Step 6: Event Listeners

```php
namespace App\EventListener;

use CoreShop\Bundle\OrderBundle\Event\CartEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class CartListener implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            'coreshop.cart.post_add_item' => 'onItemAdded',
            'coreshop.cart.update' => 'onCartUpdate',
        ];
    }

    public function onItemAdded(CartEvent $event): void
    {
        $cart = $event->getCart();
        // Custom logic after item added
    }
}
```

### Key CoreShop Events

| Event | When |
|-------|------|
| `coreshop.cart.update` | Cart updated |
| `coreshop.cart.pre_add_item` / `post_add_item` | Item added to cart |
| `coreshop.cart.pre_remove_item` / `post_remove_item` | Item removed |
| `coreshop.customer.register` | New customer registered |
| `coreshop.customer.request_password_reset` | Password reset requested |
| `coreshop.*.pre_create` / `post_create` | Backend entity created |
| `coreshop.*.pre_save` / `post_save` | Backend entity saved |
| `coreshop.*.pre_delete` / `post_delete` | Backend entity deleted |
| `coreshop.payment_provider.supports` | Payment provider check |
| `coreshop.rule.availability_check` | Rule availability check |

## Step 7: Override Controllers & Templates

### Override a Frontend Controller

```php
namespace App\Controller;

class ProductController extends \CoreShop\Bundle\FrontendBundle\Controller\ProductController
{
    public function detailAction(Request $request)
    {
        // Custom logic
        return parent::detailAction($request);
    }
}
```

```yaml
# config/config.yaml
core_shop_frontend:
    controllers:
        product: App\Controller\ProductController
```

### Override Templates

```bash
# Copy all CoreShop frontend templates to your project
php bin/console coreshop:frontend:install
```

This copies templates to `templates/coreshop/` where you can customize them.

**Available controllers to override:** index, register, customer, currency, language, search, cart, checkout, category, product, quote, security, payment

## Step 8: Studio v2 Extensions (for External Plugins)

### Build Setup for External Plugins

Your plugin needs `rsbuild.config.ts` with CoreShop aliases:

```typescript
// rsbuild.config.ts
import { defineConfig } from '@rsbuild/core'
import { pluginReact } from '@rsbuild/plugin-react'
import { pluginModuleFederation } from '@module-federation/rsbuild-plugin'

const coreshopPath = 'vendor/coreshop/core-shop/src/CoreShop/Bundle/'

export default defineConfig({
  resolve: {
    alias: {
      '@coreshop/rule': path.join(coreshopPath, 'RuleBundle/Resources/assets/pimcore-studio'),
      '@coreshop/resource': path.join(coreshopPath, 'ResourceBundle/Resources/assets/pimcore-studio'),
      '@coreshop/order': path.join(coreshopPath, 'OrderBundle/Resources/assets/pimcore-studio'),
      '@coreshop/product': path.join(coreshopPath, 'ProductBundle/Resources/assets/pimcore-studio'),
      // ... other bundles as needed
    }
  },
  plugins: [
    pluginReact(),
    pluginModuleFederation({
      name: 'my_coreshop_plugin',
      shared: {
        '@coreshop/rule': { singleton: true, eager: false },
        '@coreshop/resource': { singleton: true, eager: false },
        // ALL CoreShop bundles must be shared as singletons
      }
    })
  ]
})
```

### The 7 Extension Types

All imported from `@coreshop/resource/src/entities`:

**1. Form Extensions** — Add fields to any entity form:
```typescript
const formReg = container.get<EntityFormExtensionRegistry>(entityFormExtensionsServiceId)
formReg.add('coreshop.address.country.form', ({ data, onChange }) => (
  <Form.Item label="My Field"><Input onChange={(e) => onChange({ myField: e.target.value })} /></Form.Item>
))
```

**2. Table Column Extensions** — Add columns to nested tables
**3. Save Decorators** — Transform data before save
**4. Tab Extensions** — Add tabs to entity detail views
**5. Action Extensions** — Add toolbar/context-menu buttons
**6. Validation Extensions** — Custom validation before save
**7. Lifecycle Hooks** — beforeLoad/afterLoad/beforeSave/afterSave/beforeDelete/afterDelete

### Slot Naming Convention

- Forms: `coreshop.{bundle}.{entity}.form` (e.g. `coreshop.address.country.form`)
- Tables: `coreshop.{bundle}.{entity}.{table}` (e.g. `coreshop.taxation.tax_rule_group.tax_rules`)
- Entity key: `coreshop.{bundle}.{entity}` (for save, validation, lifecycle, tabs, actions)

## Step 9: Custom Notification Rules

```yaml
# Register a custom notification condition
App\Notification\MyCondition:
    tags:
        - name: coreshop.notification_rule.condition
          type: my_condition
          notification-type: order
          form-type: App\Form\Type\MyConditionType
```

**Trigger notifications programmatically:**
```php
$this->rulesProcessor->applyRules('order', $order, [
    'fromState' => $previousState,
    'toState' => $newState,
    '_locale' => $order->getLocaleCode(),
    'recipient' => $customer->getEmail(),
]);
```

## Step 10: Custom Resource with Admin API

Register a completely new entity with admin CRUD:

```yaml
# config/config.yaml
core_shop_resource:
    resources:
        app.custom_entity:
            classes:
                model: App\Entity\CustomEntity
                interface: CoreShop\Component\Resource\Model\ResourceInterface
                repository: CoreShop\Bundle\ResourceBundle\Doctrine\ORM\EntityRepository
```

```yaml
# config/routes.yaml
app_admin_custom_entity:
    type: coreshop.resources
    resource: |
        alias: app.custom_entity
```

This auto-generates admin API endpoints for list, get, add, save, delete.

## Reference

See `.claude/skills/coreshop/reference.md` for configuration cheat sheet and common patterns.