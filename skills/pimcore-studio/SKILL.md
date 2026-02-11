---
name: pimcore-studio
description: Build Pimcore Studio v2 features - React/TypeScript, Ant Design, plugins, modules, dynamic types, DI container, widgets
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

# Pimcore Studio v2 Development

You are helping build features for Pimcore Studio v2 (React/TypeScript). Before writing code, load context about the architecture and patterns.

## Tech Stack

Pimcore Studio v2 is built with:
- **React 18** — UI components
- **TypeScript** — Type safety
- **Ant Design (antd)** — UI component library (Form, Table, Modal, Select, Input, Button, Tabs, Tree, etc.)
- **InversifyJS** — Dependency injection container
- **react-i18next** — Internationalization (`useTranslation()` hook)
- **Rsbuild** — Build tool with Module Federation for plugin loading

## Discovering Pimcore Studio Source

When unsure how to do something, **read the Studio source code**:

```
vendor/pimcore/studio-ui-bundle/assets/js/src/
├── core/
│   ├── app/
│   │   ├── depency-injection/     # DI container setup (InversifyJS)
│   │   ├── module-system/         # Module registration system
│   │   ├── plugin-system/         # Plugin lifecycle (IAbstractPlugin)
│   │   ├── config/services/       # Service IDs (serviceIds constant)
│   │   ├── i18n/                  # Translation setup (react-i18next)
│   │   ├── api/                   # API utilities
│   │   └── public-api/            # Public API gateway, element API
│   ├── components/                # Pimcore UI components (built on antd)
│   │   ├── form/                  # Form controls
│   │   ├── sidebar/               # Sidebar components
│   │   ├── split/                 # Split layouts
│   │   └── ...
│   └── utils/                     # Utility hooks and helpers
│       ├── hooks/                 # useDebounce, useClickOutside, etc.
│       └── ...
└── modules/                       # Pimcore feature modules
    ├── element/                   # Element handling, DynamicType registries
    ├── widget-manager/            # Widget registry
    └── ...
```

Search for examples:
```bash
grep -r "registerWidget\|registerDynamicType\|IAbstractPlugin" \
  vendor/pimcore/studio-ui-bundle/assets/js/src/
```

## Step 1: Plugin Architecture

### Plugin Lifecycle

```typescript
import { type IAbstractPlugin, container } from '@pimcore/studio-ui-bundle'

const plugin: IAbstractPlugin = {
  name: 'my-plugin',

  onInit() {
    // Phase 1: Runs first, before any modules
    // - Container bindings (InversifyJS)
    // - Dynamic type registration
    // - Registry creation
  },

  onStartup({ moduleSystem }) {
    // Phase 2: Runs after ALL plugins' onInit()
    // - Module registration
    // - Widget registration
    // - Can safely use container bindings from other plugins
  }
}

export default plugin
```

**Order matters**: All `onInit()` calls complete before any `onStartup()`. Bindings must happen in `onInit()`.

### Module System

Modules encapsulate feature registrations:
```typescript
import { type AbstractModule, container } from '@pimcore/studio-ui-bundle'

export const MyModule: AbstractModule = {
  onInit(): void {
    // Register services, form builders, extensions
  }
}

// In plugin's onStartup():
moduleSystem.registerModule(MyModule)
```

### DI Container (InversifyJS)

```typescript
import { container } from '@pimcore/studio-ui-bundle'

// Bind
container.bind('MyServiceId').to(MyClass).inSingletonScope()
container.bind('MyServiceId').toConstantValue(myInstance)

// Retrieve
const service = container.get<MyType>('MyServiceId')
```

## Step 2: Core Studio v2 Services

### Widget Registry
```typescript
import { serviceIds } from '@pimcore/studio-ui-bundle/app'
import { type WidgetRegistry } from '@pimcore/studio-ui-bundle/modules/widget-manager'

const widgetRegistry = container.get<WidgetRegistry>(serviceIds['WidgetRegistry'])
widgetRegistry.registerWidget({ name: 'my.widget', component: MyComponent })
```

### Dynamic Type Registry
```typescript
import { DynamicTypeObjectDataRegistry } from '@pimcore/studio-ui-bundle/modules/element'

const registry = container.get<DynamicTypeObjectDataRegistry>(
  serviceIds['DynamicTypes/ObjectDataRegistry']
)
registry.registerDynamicType(new MyDynamicType())
```

## Step 3: Dynamic Types (Pimcore Data Object Fields)

For custom field types in the class definition editor:

```typescript
import {
  DynamicTypeObjectDataAbstractSelect,
  DynamicTypeObjectDataAbstractMultiSelect,
  DynamicTypeObjectDataAbstract,
  DynamicTypeFieldFilterMultiselect,
} from '@pimcore/studio-ui-bundle/modules/element'

// Single select
export class DynamicTypeMyField extends DynamicTypeObjectDataAbstractSelect {
  readonly id = 'myFieldType'  // MUST match PHP CoreExtension $fieldtype
  readonly dynamicTypeFieldFilterType = new DynamicTypeFieldFilterMultiselect()
}

// Multi-select
export class DynamicTypeMyFieldMulti extends DynamicTypeObjectDataAbstractMultiSelect {
  readonly id = 'myFieldTypeMultiselect'
  readonly dynamicTypeFieldFilterType = new DynamicTypeFieldFilterMultiselect()
}

// Custom rendering
export class DynamicTypeMyCustom extends DynamicTypeObjectDataAbstract {
  readonly id = 'myCustomFieldType'
  getObjectDataComponent(props: any): React.ReactElement {
    return <MyCustomEditor {...props} />
  }
}
```

**Register in `onInit()`** — never in `onStartup()`.

## Step 4: Building Components with Ant Design

### Form Component
```typescript
import React from 'react'
import { Form, Input, Switch, InputNumber, Select } from 'antd'
import { useTranslation } from 'react-i18next'

interface MyFormProps {
  data: MyEntity
  onChange: (data: MyEntity) => void
}

export const MyForm: React.FC<MyFormProps> = ({ data, onChange }) => {
  const { t } = useTranslation()

  return (
    <Form layout="vertical">
      <Form.Item label={t('name')} required>
        <Input value={data.name}
          onChange={(e) => onChange({ ...data, name: e.target.value })} />
      </Form.Item>
      <Form.Item label={t('active')}>
        <Switch checked={data.active}
          onChange={(checked) => onChange({ ...data, active: checked })} />
      </Form.Item>
    </Form>
  )
}
```

### Table/List Component
```typescript
import React, { useEffect, useState } from 'react'
import { Table, Button, Space, Popconfirm } from 'antd'

export const MyList: React.FC = () => {
  const [data, setData] = useState<MyEntity[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    void (async () => {
      try { setData(await myApi.list()) }
      finally { setLoading(false) }
    })()
  }, [])

  return (
    <Table dataSource={data} loading={loading} rowKey="id" columns={[
      { title: 'Name', dataIndex: 'name' },
      { title: 'Active', dataIndex: 'active', render: (v: boolean) => v ? 'Yes' : 'No' },
      { title: 'Actions', render: (_, record) => (
        <Popconfirm title="Delete?" onConfirm={() => handleDelete(record.id)}>
          <Button danger size="small">Delete</Button>
        </Popconfirm>
      )},
    ]} />
  )
}
```

### Modal / Dialog
```typescript
import { Modal, Form, Input } from 'antd'

const [open, setOpen] = useState(false)

<Modal title="Add Entity" open={open} onOk={handleOk} onCancel={() => setOpen(false)}>
  <Form layout="vertical">
    <Form.Item label="Name"><Input /></Form.Item>
  </Form>
</Modal>
```

## Step 5: Select Components with Module-Level Caching

**ALWAYS use this pattern** for any Select that loads options from an API:

```typescript
import React from 'react'
import { Select, type SelectProps } from 'antd'

let cachedOptions: Array<{ value: number; label: string }> | null = null
let loadPromise: Promise<typeof cachedOptions> | null = null

const loadOptions = async () => {
  if (cachedOptions) return cachedOptions
  if (loadPromise) return loadPromise
  loadPromise = (async () => {
    try {
      const items = await myApi.list()
      cachedOptions = items.map(i => ({ value: i.id, label: i.name }))
      return cachedOptions
    } finally { loadPromise = null }
  })()
  return loadPromise
}

export const clearCache = () => { cachedOptions = null; loadPromise = null }

export const MySelect: React.FC<SelectProps> = (props) => {
  const [options, setOptions] = React.useState(cachedOptions || [])
  const [loading, setLoading] = React.useState(!cachedOptions)

  React.useEffect(() => {
    void (async () => {
      if (!cachedOptions) setLoading(true)
      try { setOptions(await loadOptions()) }
      finally { setLoading(false) }
    })()
  }, [])

  return <Select {...props} loading={loading} options={options} showSearch
    filterOption={(input, opt) =>
      (opt?.label ?? '').toLowerCase().includes(input.toLowerCase())
    } />
}
```

## Step 6: API Layer

```typescript
export interface MyEntity {
  id: number
  name: string
  active: boolean
}

export const myApi = {
  async list(): Promise<MyEntity[]> {
    const res = await fetch('/pimcore-studio/api/my-bundle/entities')
    return (await res.json()).data ?? (await res.json())
  },
  async get(id: number): Promise<MyEntity> {
    return (await fetch(`/pimcore-studio/api/my-bundle/entities/${id}`)).json()
  },
  async save(entity: Partial<MyEntity>): Promise<MyEntity> {
    const method = entity.id ? 'PUT' : 'POST'
    const url = entity.id
      ? `/pimcore-studio/api/my-bundle/entities/${entity.id}`
      : '/pimcore-studio/api/my-bundle/entities'
    return (await fetch(url, {
      method, headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(entity)
    })).json()
  },
  async delete(id: number): Promise<void> {
    await fetch(`/pimcore-studio/api/my-bundle/entities/${id}`, { method: 'DELETE' })
  },
}
```

## Step 7: Translations

**File**: `Resources/translations/studio.en.yaml`
```yaml
my_entity: 'My Entity'
my_entity_name: 'Name'
```

**Usage**:
```typescript
import { useTranslation } from 'react-i18next'
const { t } = useTranslation()
t('my_entity_name')
```

## Step 8: Build System

- **Rsbuild** with Module Federation for plugin loading
- `npm run build` — Production build
- `npm run dev` — Development with hot reload

### package.json Template
```json
{
  "name": "@my-scope/my-plugin",
  "version": "1.0.0",
  "main": "src/index.ts",
  "dependencies": {
    "@pimcore/studio-ui-bundle": "^1.0.0",
    "antd": "^5.12.0",
    "react": "18.3.x",
    "react-i18next": "^14.0.0"
  }
}
```

## Step 9: Directory Structure

```
Resources/assets/pimcore-studio/
├── package.json
└── src/
    ├── main.ts                    # Plugin entry (IAbstractPlugin)
    ├── modules/
    │   ├── icon-library/          # Icon registrations
    │   └── {feature}/
    │       ├── api.ts             # API layer
    │       ├── MyComponent.tsx    # React components (using antd)
    │       └── index.ts
    ├── components/                # Shared components
    │   └── MySelect.tsx           # Selects with caching
    └── dynamic-types/             # Pimcore field types
        └── DynamicTypeMyField.tsx
```

## Reference

See `.claude/skills/pimcore-studio/patterns.md` for complete code templates.