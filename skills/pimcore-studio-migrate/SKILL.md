---
name: pimcore-studio-migrate
description: Migrate any Pimcore ExtJS admin UI to Studio v2 React/TypeScript - panels, grids, forms, trees, stores, plugins
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

# Migrating Pimcore ExtJS Admin to Studio v2

You are helping migrate a Pimcore bundle's admin UI from ExtJS (Pimcore Admin UI Classic) to Studio v2 (React/TypeScript). This guide applies to any Pimcore bundle — first-party or third-party.

## Important: Studio v2 Tech Stack

Pimcore Studio v2 is built with:
- **React 18** — UI components
- **TypeScript** — Type safety
- **Ant Design (antd)** — UI component library (forms, tables, modals, buttons, etc.)
- **InversifyJS** — Dependency injection container
- **react-i18next** — Internationalization
- **Rsbuild** — Build tool (Module Federation for plugin loading)

**Studio source code** is at `vendor/pimcore/studio-ui-bundle/assets/js/src/` — read this to understand Pimcore's own components, hooks, services, and patterns when you're unsure how something works.

## Step 1: Identify What Exists in ExtJS

Before migrating, map the ExtJS implementation:

```bash
# Find all ExtJS JS files
find {bundle-path}/Resources/public/pimcore/js -name "*.js" | sort

# Find panel/grid/form/tree definitions
grep -rn "Ext.define\|Ext.panel.Panel\|Ext.grid.Panel\|Ext.form.Panel\|Ext.tree.Panel" \
  {bundle-path}/Resources/public/pimcore/js/

# Find data stores and their proxy URLs
grep -rn "Ext.data.Store\|Ext.data.TreeStore\|proxy:" \
  {bundle-path}/Resources/public/pimcore/js/

# Find plugin registrations
grep -rn "pimcore.registerNS\|pimcore.globalmanager\|pimcore.plugin" \
  {bundle-path}/Resources/public/pimcore/js/

# Find toolbar/menu registrations
grep -rn "pimcore.layout.Toolbar\|pimcore.navigation" \
  {bundle-path}/Resources/public/pimcore/js/
```

## Step 2: Map ExtJS Concepts to Studio v2

| ExtJS Concept | Studio v2 Equivalent | Notes |
|---------------|---------------------|-------|
| `Ext.define('My.Class', ...)` | TypeScript class or React component | No more class inheritance chains |
| `Ext.panel.Panel` | React component | Use `antd` `Card` for framed panels |
| `Ext.grid.Panel` | `antd` `<Table>` | Ant Design Table with columns |
| `Ext.form.Panel` | `antd` `<Form>` with `<Form.Item>` | Use FormBuilder for extensible forms |
| `Ext.form.field.ComboBox` | `antd` `<Select>` with module-level caching | Always cache API-backed selects |
| `Ext.form.field.Text` | `antd` `<Input>` | |
| `Ext.form.field.Number` | `antd` `<InputNumber>` | |
| `Ext.form.field.Checkbox` | `antd` `<Switch>` or `<Checkbox>` | `Switch` for on/off toggles |
| `Ext.form.field.TextArea` | `antd` `<Input.TextArea>` | |
| `Ext.form.field.Date` | `antd` `<DatePicker>` | |
| `Ext.form.field.HtmlEditor` | Wysiwyg / rich text component | |
| `Ext.tree.Panel` | `antd` `<Tree>` | |
| `Ext.tab.Panel` | `antd` `<Tabs>` | Or entity tab extensions |
| `Ext.Window` | `antd` `<Modal>` | |
| `Ext.toolbar.Toolbar` | `antd` `<Space>` with `<Button>` | Or widget registration |
| `Ext.MessageBox.alert()` | `antd` `message.info()` or `notification` | |
| `Ext.MessageBox.confirm()` | `antd` `Modal.confirm()` | |
| `Ext.data.Store` | API wrapper (fetch) | REST API replaces data stores |
| `Ext.data.proxy.Ajax` | `fetch()` or API class | Direct HTTP calls |
| `Ext.Ajax.request` | `fetch()` | |
| `pimcore.globalmanager.get()` | `container.get()` (InversifyJS) | DI container replaces global registry |
| `pimcore.registerNS()` | ES module imports | No namespace registration needed |
| `t('key')` translation | `useTranslation()` hook → `t('key')` | Same concept, React hook |
| ExtJS events (`on`, `fireEvent`) | React props (`onChange`, callbacks) | Or custom event emitters |
| ExtJS plugins | `IAbstractPlugin` with `onInit`/`onStartup` | |
| `Ext.Loader.require` | ES `import` statements | |

## Step 3: Create Studio v2 Project Structure

```
{bundle-path}/Resources/assets/pimcore-studio/
├── package.json
└── src/
    ├── main.ts                    # Plugin entry point (IAbstractPlugin)
    ├── modules/
    │   ├── icon-library/          # Icon registration
    │   │   └── index.ts
    │   └── {feature}/             # Per feature/entity
    │       ├── api.ts             # API layer (replaces Ext.data.Store)
    │       ├── types.ts           # TypeScript interfaces
    │       ├── MyComponent.tsx    # React components (replace ExtJS panels)
    │       └── index.ts
    ├── components/                # Shared/reusable components
    │   └── MySelect.tsx           # Select components (replace ComboBoxes)
    └── dynamic-types/             # Custom Pimcore field types
        └── DynamicTypeMyField.tsx
```

## Step 4: Plugin Entry Point (main.ts)

Replace ExtJS plugin registration:

**ExtJS (before):**
```javascript
pimcore.registerNS('pimcore.plugin.myPlugin');
pimcore.plugin.myPlugin = Class.create(pimcore.plugin.admin, {
    initialize: function() { /* ... */ },
    pimcoreReady: function() { /* ... */ }
});
```

**Studio v2 (after):**
```typescript
import { type IAbstractPlugin, container } from '@pimcore/studio-ui-bundle'
import { serviceIds } from '@pimcore/studio-ui-bundle/app'
import { type WidgetRegistry } from '@pimcore/studio-ui-bundle/modules/widget-manager'
import { DynamicTypeObjectDataRegistry } from '@pimcore/studio-ui-bundle/modules/element'

const plugin: IAbstractPlugin = {
  name: 'my-bundle',

  onInit() {
    // Phase 1: Container bindings, dynamic types
    // Runs before any modules — equivalent to ExtJS initialize()
  },

  onStartup({ moduleSystem }) {
    // Phase 2: Module + widget registration
    // Runs after all plugins init — equivalent to ExtJS pimcoreReady()
  }
}

export default plugin
```

## Step 5: Migrate Data Stores → API Layer

Replace `Ext.data.Store` with fetch-based API wrappers:

**ExtJS (before):**
```javascript
var store = Ext.create('Ext.data.Store', {
    autoLoad: true,
    proxy: {
        type: 'ajax',
        url: '/admin/my-bundle/list',
        reader: { type: 'json', rootProperty: 'data' }
    },
    fields: ['id', 'name', 'active']
});
```

**Studio v2 (after):**
```typescript
export interface MyEntity {
  id: number
  name: string
  active: boolean
}

export const myApi = {
  async list(): Promise<MyEntity[]> {
    const res = await fetch('/pimcore-studio/api/my-bundle/entities')
    const json = await res.json()
    return json.data ?? json
  },
  async get(id: number): Promise<MyEntity> {
    const res = await fetch(`/pimcore-studio/api/my-bundle/entities/${id}`)
    return res.json()
  },
  async save(entity: MyEntity): Promise<MyEntity> {
    const res = await fetch(`/pimcore-studio/api/my-bundle/entities/${entity.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(entity),
    })
    return res.json()
  },
  async delete(id: number): Promise<void> {
    await fetch(`/pimcore-studio/api/my-bundle/entities/${id}`, { method: 'DELETE' })
  }
}
```

## Step 6: Migrate Grid Panels → antd Table

**ExtJS (before):**
```javascript
Ext.create('Ext.grid.Panel', {
    store: myStore,
    columns: [
        { text: 'ID', dataIndex: 'id', width: 60 },
        { text: 'Name', dataIndex: 'name', flex: 1 },
        { text: 'Active', dataIndex: 'active', width: 80,
          renderer: function(v) { return v ? 'Yes' : 'No'; } }
    ],
    tbar: [{ text: 'Add', handler: onAdd }]
});
```

**Studio v2 (after):**
```typescript
import React, { useEffect, useState } from 'react'
import { Table, Button, Space } from 'antd'

export const MyEntityList: React.FC = () => {
  const [data, setData] = useState<MyEntity[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    void (async () => {
      try { setData(await myApi.list()) }
      finally { setLoading(false) }
    })()
  }, [])

  return (
    <div>
      <Space style={{ marginBottom: 16 }}>
        <Button type="primary" onClick={handleAdd}>Add</Button>
      </Space>
      <Table dataSource={data} loading={loading} rowKey="id" columns={[
        { title: 'ID', dataIndex: 'id', width: 60 },
        { title: 'Name', dataIndex: 'name' },
        { title: 'Active', dataIndex: 'active', width: 80,
          render: (v: boolean) => v ? 'Yes' : 'No' },
      ]} />
    </div>
  )
}
```

## Step 7: Migrate Forms → antd Form

**ExtJS (before):**
```javascript
Ext.create('Ext.form.Panel', {
    items: [
        { xtype: 'textfield', fieldLabel: 'Name', name: 'name' },
        { xtype: 'checkbox', fieldLabel: 'Active', name: 'active' },
        { xtype: 'combobox', fieldLabel: 'Type', name: 'type',
          store: typeStore, displayField: 'name', valueField: 'id' }
    ],
    buttons: [{ text: 'Save', handler: onSave }]
});
```

**Studio v2 (after):**
```typescript
import React from 'react'
import { Form, Input, Switch, Button } from 'antd'
import { useTranslation } from 'react-i18next'
import { TypeSelect } from '../components/TypeSelect'

export const MyForm: React.FC<{ data: MyEntity; onChange: (d: MyEntity) => void }> = ({
  data, onChange
}) => {
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
      <Form.Item label={t('type')}>
        <TypeSelect value={data.type}
          onChange={(value) => onChange({ ...data, type: value })} />
      </Form.Item>
    </Form>
  )
}
```

## Step 8: Migrate ComboBoxes → Cached antd Select

**Always use module-level caching** for API-backed Select components:

```typescript
import React from 'react'
import { Select, type SelectProps } from 'antd'

let cachedOptions: Array<{ value: number, label: string }> | null = null
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

export const TypeSelect: React.FC<SelectProps> = (props) => {
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
    filterOption={(input, opt) => (opt?.label ?? '').toLowerCase().includes(input.toLowerCase())} />
}
```

## Step 9: Migrate Dynamic Types (CoreExtensions)

For custom Pimcore Data Object field types:

```typescript
import {
  DynamicTypeObjectDataAbstractSelect,
  DynamicTypeObjectDataAbstractMultiSelect,
  DynamicTypeFieldFilterMultiselect
} from '@pimcore/studio-ui-bundle/modules/element'

// Single select
export class DynamicTypeMyField extends DynamicTypeObjectDataAbstractSelect {
  readonly id = 'myField'  // MUST match PHP $fieldtype exactly
  readonly dynamicTypeFieldFilterType = new DynamicTypeFieldFilterMultiselect()
}

// Multi-select
export class DynamicTypeMyFieldMultiselect extends DynamicTypeObjectDataAbstractMultiSelect {
  readonly id = 'myFieldMultiselect'
  readonly dynamicTypeFieldFilterType = new DynamicTypeFieldFilterMultiselect()
}
```

**Register in `onInit()`** (not `onStartup()`).

## Step 10: Discovering Pimcore Studio Patterns

When unsure how Pimcore itself handles something, **read the Studio source code**:

```
vendor/pimcore/studio-ui-bundle/assets/js/src/
├── core/
│   ├── app/
│   │   ├── depency-injection/     # DI container setup
│   │   ├── module-system/         # Module registration system
│   │   ├── plugin-system/         # Plugin lifecycle (IAbstractPlugin)
│   │   ├── config/services/       # Service IDs
│   │   ├── i18n/                  # Translation setup
│   │   ├── api/                   # API utilities
│   │   └── public-api/            # Public API gateway
│   ├── components/                # Pimcore UI components
│   │   ├── form/                  # Form controls
│   │   ├── sidebar/               # Sidebar components
│   │   ├── split/                 # Split layouts
│   │   └── ...
│   └── utils/                     # Utility hooks and helpers
│       ├── hooks/                 # useDebounce, useClickOutside, etc.
│       └── ...
└── modules/                       # Pimcore feature modules
    ├── element/                   # Element (Asset/Document/Object) components
    ├── widget-manager/            # Widget registry
    └── ...
```

Search for examples: `grep -r "registerWidget\|registerDynamicType" vendor/pimcore/studio-ui-bundle/assets/`

## Step 11: Migrate Translations

**ExtJS**: `Resources/translations/admin.en.yml`
**Studio v2**: `Resources/translations/studio.en.yaml`

Copy relevant keys. Usage: `const { t } = useTranslation()` → `t('key')`

## Step 12: Backend API Routes

Studio v2 uses REST APIs (typically under `/pimcore-studio/api/`). If the ExtJS backend used admin routes (`/admin/...`), you may need new API controllers. Check if `pimcore/studio-backend-bundle` already provides the endpoints you need.

## Migration Checklist

See `.claude/skills/pimcore-studio-migrate/migration-checklist.md` for a detailed per-entity checklist.