# CoreShop Extension Reference — Configuration & Patterns

## Configuration Cheat Sheet

### Override an Entity Model

```yaml
# config/config.yaml
core_shop_{bundle}:
    resources:
        {entity}:
            classes:
                model: App\Entity\MyExtendedEntity
```

**Common entity config keys:**

| Key | Entity |
|-----|--------|
| `core_shop_currency.resources.currency` | Currency |
| `core_shop_currency.resources.exchange_rate` | Exchange Rate |
| `core_shop_address.resources.country` | Country |
| `core_shop_address.resources.state` | State |
| `core_shop_address.resources.zone` | Zone |
| `core_shop_taxation.resources.tax_rate` | Tax Rate |
| `core_shop_taxation.resources.tax_rule_group` | Tax Rule Group |
| `core_shop_store.resources.store` | Store |
| `core_shop_shipping.resources.carrier` | Carrier |
| `core_shop_payment.resources.payment_provider` | Payment Provider |
| `core_shop_order.resources.cart_price_rule` | Cart Price Rule |
| `core_shop_product.resources.product_price_rule` | Product Price Rule |
| `core_shop_notification.resources.notification_rule` | Notification Rule |

### Override a Frontend Controller

```yaml
core_shop_frontend:
    controllers:
        {name}: App\Controller\{Name}Controller
```

Available: `index`, `register`, `customer`, `currency`, `language`, `search`, `cart`, `checkout`, `category`, `product`, `quote`, `security`, `payment`

### Override Pimcore Data Object Configuration

```yaml
core_shop_{bundle}:
    pimcore:
        {entity}:
            path: coreshop/my-custom-path
            classes:
                model: Pimcore\Model\DataObject\MyCustomClass
                interface: CoreShop\Component\{Module}\Model\{Entity}Interface
                repository: App\Repository\MyRepository
```

### Add Admin JS/CSS (Legacy ExtJS)

```yaml
core_shop_{bundle}:
    pimcore_admin:
        js:
            my_script: '/my-bundle/js/my-script.js'
        css:
            my_style: '/my-bundle/css/my-style.css'
```

## Service Tag Reference

### Rule Engine Tags

```yaml
# Cart Price Rules
- { name: coreshop.cart_price_rule.condition, type: my_type, form-type: App\Form\Type\MyType }
- { name: coreshop.cart_price_rule.action, type: my_type, form-type: App\Form\Type\MyType }

# Product Price Rules
- { name: coreshop.product_price_rule.condition, type: my_type, form-type: App\Form\Type\MyType }
- { name: coreshop.product_price_rule.action, type: my_type, form-type: App\Form\Type\MyType }

# Product Specific Price Rules
- { name: coreshop.product_specific_price_rule.condition, type: my_type, form-type: App\Form\Type\MyType }
- { name: coreshop.product_specific_price_rule.action, type: my_type, form-type: App\Form\Type\MyType }

# Shipping Rules
- { name: coreshop.shipping_rule.condition, type: my_type, form-type: App\Form\Type\MyType }
- { name: coreshop.shipping_rule.action, type: my_type, form-type: App\Form\Type\MyType }

# Notification Rules
- { name: coreshop.notification_rule.condition, type: my_type, notification-type: order, form-type: App\Form\Type\MyType }
- { name: coreshop.notification_rule.action, type: my_type, notification-type: order, form-type: App\Form\Type\MyType }
```

### Context Tags

```yaml
- { name: coreshop.context.country }
- { name: coreshop.context.currency }
- { name: coreshop.context.store }
- { name: coreshop.context.locale }
- { name: coreshop.context.customer }
```

### Payment Gateway Tags

```yaml
- { name: coreshop.gateway_configuration_type, type: my_gateway }
```

### Filter Tags

```yaml
- { name: coreshop.filter.condition_type, type: my_filter, form-type: App\Form\Type\MyFilterType }
```

### Validator Tags

```yaml
- { name: validator.constraint_validator, alias: my_validator }
```

## Workflow Configuration

### Available Workflows

| Workflow | Key | States |
|----------|-----|--------|
| Order | `coreshop_order` | initialized, new, confirmed, cancelled, complete |
| Order Payment | `coreshop_order_payment` | new, awaiting, partially_paid, paid, cancelled, refunded, partially_refunded |
| Order Shipment | `coreshop_order_shipment` | new, ready, partially_shipped, shipped, cancelled |
| Order Invoice | `coreshop_order_invoice` | new, ready, partially_invoiced, invoiced, cancelled |
| Payment | `coreshop_payment` | new, authorized, processing, completed, failed, cancelled, refunded |
| Shipment | `coreshop_shipment` | new, ready, shipped, cancelled |
| Invoice | `coreshop_invoice` | new, ready, complete, cancelled |

### Extend a Workflow

```yaml
core_shop_workflow:
    state_machine:
        coreshop_order:
            places:
                - my_custom_state
            transitions:
                my_transition:
                    from: [confirmed]
                    to: my_custom_state
            place_colors:
                my_custom_state: '#ff6600'
            callbacks:
                after:
                    my_callback:
                        on: ['my_transition']
                        do: ['@App\Listener\MyListener', 'onTransition']
                        args: ['object']
```

## Event Reference

### Cart Events

```
coreshop.cart.update
coreshop.cart.pre_add_item
coreshop.cart.post_add_item
coreshop.cart.pre_remove_item
coreshop.cart.post_remove_item
```

### Customer Events

```
coreshop.customer.register
coreshop.customer.update_post
coreshop.customer.change_password_post
coreshop.customer.newsletter_confirm_post
coreshop.customer.request_password_reset
coreshop.customer.password_reset
```

### Address Events

```
coreshop.address.add_post
coreshop.address.update_post
coreshop.address.delete_pre
```

### Backend CRUD Events (per entity)

```
coreshop.{entity}.pre_create
coreshop.{entity}.post_create
coreshop.{entity}.pre_save
coreshop.{entity}.post_save
coreshop.{entity}.pre_delete
coreshop.{entity}.post_delete
```

### Other Events

```
coreshop.payment_provider.supports
coreshop.rule.availability_check
coreshop.workflow.valid_transitions
```

## Studio v2 Extension Slots

### Form Slots

```
coreshop.address.country.form
coreshop.address.state.form
coreshop.address.zone.form
coreshop.taxation.tax_rate.form
coreshop.taxation.tax_rule_group.form
coreshop.currency.currency.form
coreshop.store.store.form
coreshop.shipping.carrier.form
coreshop.payment.payment_provider.form
```

### Table Slots

```
coreshop.taxation.tax_rule_group.tax_rules
```

### Entity Keys (save decorators, validation, lifecycle, tabs, actions)

```
coreshop.address.country
coreshop.address.state
coreshop.address.zone
coreshop.taxation.tax_rate
coreshop.taxation.tax_rule_group
coreshop.currency.currency
coreshop.store.store
coreshop.shipping.carrier
coreshop.payment.payment_provider
```

### Rule Registry Service IDs (for registering conditions/actions)

```typescript
// Import from respective bundles
import { coreshopOrderServiceIds } from '@coreshop/order'
// → .cartPriceRuleConditionRegistry
// → .cartPriceRuleActionRegistry
// → .cartItemConditionRegistry
// → .cartItemActionRegistry

import { coreshopProductServiceIds } from '@coreshop/product'
// → .productPriceRuleConditionRegistry
// → .productPriceRuleActionRegistry
// → .productSpecificPriceRuleConditionRegistry
// → .productSpecificPriceRuleActionRegistry

import { coreshopShippingServiceIds } from '@coreshop/shipping'
// → .shippingRuleConditionRegistry
// → .shippingRuleActionRegistry

import { coreshopNotificationServiceIds } from '@coreshop/notification'
// → .notificationRuleConditionRegistry
// → .notificationRuleActionRegistry
```

## Common Extension Patterns

### Service Decoration

```yaml
# config/services.yaml
App\Service\MyPriceCalculator:
    decorates: 'CoreShop\Component\Core\Product\ProductPriceCalculator'
    arguments:
        - '@App\Service\MyPriceCalculator.inner'
```

### Custom Repository

```php
namespace App\Repository;

use CoreShop\Bundle\ResourceBundle\Doctrine\ORM\EntityRepository;

class MyEntityRepository extends EntityRepository
{
    public function findActiveByStore(int $storeId): array
    {
        return $this->createQueryBuilder('e')
            ->where('e.active = :active')
            ->andWhere('e.store = :store')
            ->setParameter('active', true)
            ->setParameter('store', $storeId)
            ->getQuery()
            ->getResult();
    }
}
```

### Custom Index Filter

```php
namespace App\Filter;

use CoreShop\Component\Index\Filter\FilterConditionProcessorInterface;

class MyFilterCondition implements FilterConditionProcessorInterface
{
    public function prepareValuesForRendering($condition, $filter, $list, $currentFilter)
    {
        // Prepare values for frontend rendering
    }

    public function addCondition($condition, $filter, $list, $currentFilter, $parameterBag, $isPrecondition = false)
    {
        // Add condition to the listing query
        return $currentFilter;
    }
}
```

```yaml
App\Filter\MyFilterCondition:
    autoconfigure: false
    tags:
        - { name: coreshop.filter.condition_type, type: my_filter, form-type: App\Form\Type\MyFilterType }
```

### Triggering Notifications

```php
// Inject CoreShop\Component\Notification\Processor\RulesProcessorInterface
$this->rulesProcessor->applyRules('order', $order, [
    'fromState' => $fromState,
    'toState' => $toState,
    '_locale' => $order->getLocaleCode(),
    'recipient' => $customer->getEmail(),
    'firstname' => $customer->getFirstname(),
    'lastname' => $customer->getLastname(),
    'orderNumber' => $order->getOrderNumber(),
]);
```

## Useful Commands

```bash
# Debug CoreShop parameters
php bin/console debug:container --parameter=coreshop.model.currency.class
php bin/console debug:container --tag=coreshop.cart_price_rule.condition

# Install frontend templates
php bin/console coreshop:frontend:install

# Rebuild class definitions
php bin/console pimcore:deployment:classes-rebuild

# Clear cache
php bin/console cache:clear

# Run migrations after entity changes
php bin/console doctrine:migrations:diff
php bin/console doctrine:migrations:migrate
```