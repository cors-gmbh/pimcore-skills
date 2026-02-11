# Pimcore Studio v2 Code Templates & Patterns

All components use **Ant Design (antd)** as the UI library. When unsure about antd component APIs, check `vendor/pimcore/studio-ui-bundle/assets/js/src/` for how Pimcore itself uses them.

## Complete Plugin Entry Point

```typescript
// main.ts
import { type IAbstractPlugin, container } from '@pimcore/studio-ui-bundle'
import { serviceIds } from '@pimcore/studio-ui-bundle/app'
import { type WidgetRegistry } from '@pimcore/studio-ui-bundle/modules/widget-manager'
import { DynamicTypeObjectDataRegistry } from '@pimcore/studio-ui-bundle/modules/element'

import { DynamicTypeMyField } from './dynamic-types/DynamicTypeMyField'
import { IconLibraryModule } from './modules/icon-library'
import { MyFeatureModule } from './modules/my-feature'
import { MyComponent } from './modules/my-feature/MyComponent'

const plugin: IAbstractPlugin = {
  name: 'my-pimcore-plugin',

  onInit() {
    // Register dynamic types (must be in onInit)
    const objectDataRegistry = container.get<DynamicTypeObjectDataRegistry>(
      serviceIds['DynamicTypes/ObjectDataRegistry']
    )
    objectDataRegistry.registerDynamicType(new DynamicTypeMyField())
  },

  onStartup({ moduleSystem }) {
    // Register modules
    moduleSystem.registerModule(IconLibraryModule)
    moduleSystem.registerModule(MyFeatureModule)

    // Register widgets
    const widgetRegistry = container.get<WidgetRegistry>(serviceIds['WidgetRegistry'])
    widgetRegistry.registerWidget({
      name: 'my.feature',
      component: MyComponent,
    })
  },
}

export default plugin
```

## Dynamic Type Templates

### Single Select
```typescript
import {
  DynamicTypeObjectDataAbstractSelect,
  DynamicTypeFieldFilterMultiselect,
} from '@pimcore/studio-ui-bundle/modules/element'

export class DynamicTypeMyField extends DynamicTypeObjectDataAbstractSelect {
  readonly id = 'myField'  // Must match PHP $fieldtype
  readonly dynamicTypeFieldFilterType = new DynamicTypeFieldFilterMultiselect()
}
```

### Multi-Select
```typescript
import {
  DynamicTypeObjectDataAbstractMultiSelect,
  DynamicTypeFieldFilterMultiselect,
} from '@pimcore/studio-ui-bundle/modules/element'

export class DynamicTypeMyFieldMultiselect extends DynamicTypeObjectDataAbstractMultiSelect {
  readonly id = 'myFieldMultiselect'
  readonly dynamicTypeFieldFilterType = new DynamicTypeFieldFilterMultiselect()
}
```

### Custom Rendering
```typescript
import { DynamicTypeObjectDataAbstract } from '@pimcore/studio-ui-bundle/modules/element'

export class DynamicTypeMyCustomField extends DynamicTypeObjectDataAbstract {
  readonly id = 'myCustomField'

  getObjectDataComponent(props: any): React.ReactElement {
    return <MyCustomEditor {...props} />
  }

  getGridCellComponent(props: any): React.ReactElement {
    return <span>{props.value ?? '-'}</span>
  }

  getVersionPreviewComponent(props: any): React.ReactElement {
    return <span>{props.value}</span>
  }
}
```

## API Layer Template

```typescript
export interface MyEntity {
  id: number
  name: string
  active: boolean
  description?: string
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

  async save(entity: Partial<MyEntity>): Promise<MyEntity> {
    const method = entity.id ? 'PUT' : 'POST'
    const url = entity.id
      ? `/pimcore-studio/api/my-bundle/entities/${entity.id}`
      : '/pimcore-studio/api/my-bundle/entities'
    const res = await fetch(url, {
      method,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(entity),
    })
    return res.json()
  },

  async delete(id: number): Promise<void> {
    await fetch(`/pimcore-studio/api/my-bundle/entities/${id}`, { method: 'DELETE' })
  },
}
```

## Select with Module-Level Caching

```typescript
import React from 'react'
import { Select, type SelectProps } from 'antd'

let cachedOptions: Array<{ value: number; label: string }> | null = null
let loadPromise: Promise<typeof cachedOptions> | null = null

const loadOptions = async (): Promise<Array<{ value: number; label: string }>> => {
  if (cachedOptions) return cachedOptions
  if (loadPromise) return loadPromise

  loadPromise = (async () => {
    try {
      const items = await myApi.list()
      cachedOptions = items.map((item) => ({
        value: item.id,
        label: item.name,
      }))
      return cachedOptions
    } catch (err) {
      console.error('Failed to load options:', err)
      throw err
    } finally {
      loadPromise = null
    }
  })()

  return loadPromise
}

export const clearMyCache = () => {
  cachedOptions = null
  loadPromise = null
}

export const MySelect: React.FC<SelectProps> = (props) => {
  const [options, setOptions] = React.useState<Array<{ value: number; label: string }>>(
    cachedOptions || []
  )
  const [loading, setLoading] = React.useState(!cachedOptions)

  React.useEffect(() => {
    void (async () => {
      if (!cachedOptions) setLoading(true)
      try {
        const opts = await loadOptions()
        setOptions(opts)
      } finally {
        setLoading(false)
      }
    })()
  }, [])

  return (
    <Select
      {...props}
      loading={loading}
      options={options}
      placeholder={props.placeholder ?? 'Select...'}
      showSearch
      filterOption={(input, option) =>
        (option?.label ?? '').toLowerCase().includes(input.toLowerCase())
      }
    />
  )
}
```

## Form Component (antd)

```typescript
import React from 'react'
import { Form, Input, Switch, InputNumber, Button, Space } from 'antd'
import { useTranslation } from 'react-i18next'

interface MyFormProps {
  data: MyEntity
  onChange: (data: MyEntity) => void
  onSave?: () => void
}

export const MyForm: React.FC<MyFormProps> = ({ data, onChange, onSave }) => {
  const { t } = useTranslation()

  const handleChange = <K extends keyof MyEntity>(key: K, value: MyEntity[K]) => {
    onChange({ ...data, [key]: value })
  }

  return (
    <Form layout="vertical">
      <Form.Item label={t('name')} required>
        <Input
          value={data.name}
          onChange={(e) => handleChange('name', e.target.value)}
        />
      </Form.Item>

      <Form.Item label={t('description')}>
        <Input.TextArea
          value={data.description}
          onChange={(e) => handleChange('description', e.target.value)}
          rows={4}
        />
      </Form.Item>

      <Form.Item label={t('active')}>
        <Switch
          checked={data.active}
          onChange={(checked) => handleChange('active', checked)}
        />
      </Form.Item>

      {onSave && (
        <Form.Item>
          <Button type="primary" onClick={onSave}>{t('save')}</Button>
        </Form.Item>
      )}
    </Form>
  )
}
```

## Table/List Component (antd)

```typescript
import React, { useEffect, useState, useCallback } from 'react'
import { Table, Button, Space, Popconfirm, message } from 'antd'
import { useTranslation } from 'react-i18next'

export const MyEntityList: React.FC = () => {
  const { t } = useTranslation()
  const [data, setData] = useState<MyEntity[]>([])
  const [loading, setLoading] = useState(true)

  const loadData = useCallback(async () => {
    setLoading(true)
    try { setData(await myApi.list()) }
    finally { setLoading(false) }
  }, [])

  useEffect(() => { void loadData() }, [loadData])

  const handleDelete = async (id: number) => {
    try {
      await myApi.delete(id)
      message.success(t('deleted_successfully'))
      await loadData()
    } catch {
      message.error(t('delete_failed'))
    }
  }

  return (
    <div>
      <Space style={{ marginBottom: 16 }}>
        <Button type="primary">{t('add')}</Button>
        <Button onClick={loadData}>{t('refresh')}</Button>
      </Space>
      <Table
        dataSource={data}
        columns={[
          { title: 'ID', dataIndex: 'id', width: 60 },
          { title: t('name'), dataIndex: 'name', sorter: true },
          { title: t('active'), dataIndex: 'active', width: 80,
            render: (v: boolean) => v ? t('yes') : t('no') },
          { title: t('actions'), width: 120,
            render: (_, record: MyEntity) => (
              <Popconfirm title={t('confirm_delete')} onConfirm={() => handleDelete(record.id)}>
                <Button danger size="small">{t('delete')}</Button>
              </Popconfirm>
            )},
        ]}
        loading={loading}
        rowKey="id"
        pagination={{ pageSize: 25 }}
      />
    </div>
  )
}
```

## Modal Component (antd)

```typescript
import React, { useState } from 'react'
import { Modal, Form, Input, message } from 'antd'
import { useTranslation } from 'react-i18next'

interface AddModalProps {
  open: boolean
  onClose: () => void
  onCreated: () => void
}

export const AddModal: React.FC<AddModalProps> = ({ open, onClose, onCreated }) => {
  const { t } = useTranslation()
  const [name, setName] = useState('')
  const [loading, setLoading] = useState(false)

  const handleOk = async () => {
    if (!name.trim()) {
      message.warning(t('name_required'))
      return
    }
    setLoading(true)
    try {
      await myApi.save({ name, active: true })
      message.success(t('created_successfully'))
      setName('')
      onCreated()
      onClose()
    } catch {
      message.error(t('create_failed'))
    } finally {
      setLoading(false)
    }
  }

  return (
    <Modal
      title={t('add_entity')}
      open={open}
      onOk={handleOk}
      onCancel={onClose}
      confirmLoading={loading}
    >
      <Form layout="vertical">
        <Form.Item label={t('name')} required>
          <Input value={name} onChange={(e) => setName(e.target.value)} />
        </Form.Item>
      </Form>
    </Modal>
  )
}
```

## Tabs Component (antd)

```typescript
import React from 'react'
import { Tabs } from 'antd'
import { useTranslation } from 'react-i18next'

export const MyEntityTabs: React.FC<{ data: MyEntity }> = ({ data }) => {
  const { t } = useTranslation()

  return (
    <Tabs items={[
      { key: 'general', label: t('general'), children: <GeneralTab data={data} /> },
      { key: 'settings', label: t('settings'), children: <SettingsTab data={data} /> },
      { key: 'history', label: t('history'), children: <HistoryTab data={data} /> },
    ]} />
  )
}
```

## Module Template

```typescript
import { type AbstractModule, container } from '@pimcore/studio-ui-bundle'

export const MyFeatureModule: AbstractModule = {
  onInit(): void {
    // Register services, form builders, extensions
  },
}
```

## Icon Library Module

```typescript
import { type AbstractModule, container } from '@pimcore/studio-ui-bundle'
import { serviceIds } from '@pimcore/studio-ui-bundle/app'
import type { IconLibrary } from '@pimcore/studio-ui-bundle/modules/icon-library'

import myIcon from '../../assets/icons/my-icon.svg'

export const IconLibraryModule: AbstractModule = {
  onInit(): void {
    const library = container.get<IconLibrary>(serviceIds['IconLibrary'])
    library.register({
      name: 'my-icon',
      component: () => <img src={myIcon} alt="" style={{ width: 16, height: 16 }} />,
    })
  },
}
```

## Translation File Template

```yaml
# Resources/translations/studio.en.yaml
---
my_entity: 'My Entity'
my_entities: 'My Entities'
my_entity_name: 'Name'
my_entity_description: 'Description'
my_entity_active: 'Active'
add_entity: 'Add Entity'
select_entity: 'Select an entity to view details.'
deleted_successfully: 'Deleted successfully'
delete_failed: 'Failed to delete'
created_successfully: 'Created successfully'
create_failed: 'Failed to create'
confirm_delete: 'Are you sure you want to delete this item?'
name_required: 'Name is required'
```

## Package.json Template

```json
{
  "name": "@my-scope/my-pimcore-plugin",
  "version": "1.0.0",
  "main": "src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./src/components": "./src/components/index.ts"
  },
  "dependencies": {
    "@pimcore/studio-ui-bundle": "^1.0.0",
    "antd": "^5.12.0",
    "react": "18.3.x",
    "react-i18next": "^14.0.0",
    "inversify": "^6.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "^18.2.0"
  }
}
```

## Common Ant Design Components Reference

| Component | Import | Usage |
|-----------|--------|-------|
| `Form`, `Form.Item` | `antd` | Form wrapper with labels/validation |
| `Input`, `Input.TextArea`, `Input.Password` | `antd` | Text inputs |
| `InputNumber` | `antd` | Numeric inputs |
| `Select` | `antd` | Dropdowns/selects |
| `Switch`, `Checkbox` | `antd` | Boolean toggles |
| `Button` | `antd` | Action buttons |
| `Table` | `antd` | Data tables with columns |
| `Modal` | `antd` | Dialog windows |
| `Tabs` | `antd` | Tab panels |
| `Tree` | `antd` | Tree views |
| `DatePicker`, `TimePicker` | `antd` | Date/time pickers |
| `Upload` | `antd` | File upload |
| `Spin` | `antd` | Loading spinner |
| `message` | `antd` | Toast notifications |
| `notification` | `antd` | Rich notifications |
| `Popconfirm` | `antd` | Confirmation popovers |
| `Space` | `antd` | Spacing wrapper |
| `Typography` | `antd` | Text/heading components |
| `Card` | `antd` | Card container |
| `Divider` | `antd` | Visual separator |
| `Tag` | `antd` | Tags/labels |
| `Badge` | `antd` | Status indicators |
| `Tooltip` | `antd` | Hover tooltips |
| `Radio`, `Radio.Group` | `antd` | Radio buttons |
| `Slider` | `antd` | Range sliders |
| `ColorPicker` | `antd` | Color selection |
| `Dropdown` | `antd` | Dropdown menus |

## Directory Structure

```
Resources/assets/pimcore-studio/
├── package.json
└── src/
    ├── main.ts                    # Plugin entry (IAbstractPlugin)
    ├── index.ts                   # Public exports
    ├── modules/
    │   ├── icon-library/
    │   │   └── index.ts
    │   └── {feature}/
    │       ├── api.ts             # API layer + TypeScript types
    │       ├── MyComponent.tsx    # React component (using antd)
    │       ├── MyForm.tsx         # Form (using antd Form)
    │       └── index.ts
    ├── components/                # Shared components
    │   ├── MySelect.tsx           # Select with module-level caching
    │   └── index.ts
    ├── dynamic-types/             # Pimcore Data Object field types
    │   ├── DynamicTypeMyField.tsx
    │   └── index.ts
    └── assets/
        └── icons/
            └── my-icon.svg
```