# Migration Checklist — ExtJS to Pimcore Studio v2

Use this checklist when migrating any Pimcore bundle from ExtJS Admin UI Classic to Studio v2 (React/TypeScript/Ant Design).

---

## Pre-Migration Analysis

- [ ] Identify all ExtJS files (`Resources/public/pimcore/js/`)
- [ ] List all panels: grid, form, tree, tab, window
- [ ] List all stores and their proxy URLs
- [ ] List all form field types and map to Ant Design equivalents
- [ ] List all toolbar buttons and custom actions
- [ ] List all event listeners and inter-component communication
- [ ] Identify global manager registrations (`pimcore.globalmanager`)
- [ ] Identify plugin registration pattern
- [ ] Identify dynamic types / CoreExtensions (custom field types)
- [ ] Map all translation keys used → need `studio.en.yaml` entries
- [ ] Check backend API endpoints — do they exist for REST access?
- [ ] Identify cross-plugin dependencies (what other plugins does this extend?)
- [ ] Read Pimcore Studio source (`vendor/pimcore/studio-ui-bundle/assets/js/src/`) for relevant patterns

---

## Backend API Preparation

- [ ] Verify REST API endpoints exist for all CRUD operations
- [ ] Check route configuration
- [ ] Verify JSON response format is compatible
- [ ] Test endpoints: GET list, GET single, POST create, PUT update, DELETE
- [ ] Create new API controllers if needed (under `/pimcore-studio/api/`)
- [ ] Add proper serialization for API responses

---

## Frontend — Project Setup

- [ ] Create `Resources/assets/pimcore-studio/` directory
- [ ] Create `package.json` with dependencies (antd, react, react-i18next, inversify)
- [ ] Create `src/main.ts` with `IAbstractPlugin`
- [ ] Register in workspace root `package.json` (if monorepo)
- [ ] Verify build works

---

## Frontend — Plugin Registration

- [ ] Define `IAbstractPlugin` with `onInit()` and `onStartup()`
- [ ] Move DI bindings and dynamic types to `onInit()`
- [ ] Move module/widget registrations to `onStartup()`
- [ ] Export plugin as default export

---

## Frontend — API Layer

For each ExtJS data store:
- [ ] Create TypeScript interface for entity data
- [ ] Create API wrapper (fetch-based)
- [ ] Implement: `list()`, `get(id)`, `save(data)`, `delete(id)`
- [ ] Test API calls in browser console

---

## Frontend — Components

For each ExtJS panel:
- [ ] Create React component equivalent
- [ ] Use Ant Design components for UI (Table, Form, Modal, etc.)
- [ ] Convert `Ext.grid.Panel` → antd `<Table>` with columns
- [ ] Convert `Ext.form.Panel` → antd `<Form>` with `<Form.Item>`
- [ ] Convert `Ext.tree.Panel` → antd `<Tree>`
- [ ] Convert `Ext.tab.Panel` → antd `<Tabs>`
- [ ] Convert `Ext.Window` → antd `<Modal>`
- [ ] Convert `Ext.toolbar.Toolbar` → antd `<Space>` with `<Button>`
- [ ] Implement state management (useState/useReducer)
- [ ] Handle loading states (antd `loading` props, `<Spin>`)
- [ ] Handle error states (antd `message.error()`, `notification`)

---

## Frontend — Form Fields

For each form field:
- [ ] `textfield` → antd `<Input>`
- [ ] `numberfield` → antd `<InputNumber>`
- [ ] `checkbox` → antd `<Switch>` or `<Checkbox>`
- [ ] `combobox` → antd `<Select>` with module-level caching
- [ ] `textarea` → antd `<Input.TextArea>`
- [ ] `datefield` → antd `<DatePicker>`
- [ ] `htmleditor` → Rich text / Wysiwyg component
- [ ] `displayfield` → antd `<Typography.Text>` or plain text
- [ ] `hidden` → Hidden state value (no UI)
- [ ] `filefield` → antd `<Upload>`
- [ ] `radiogroup` → antd `<Radio.Group>`
- [ ] `tagfield` → antd `<Select mode="multiple">`
- [ ] `slider` → antd `<Slider>`
- [ ] `colorfield` → antd `<ColorPicker>`

---

## Frontend — Select Components (ComboBoxes)

For each ComboBox that loads from API:
- [ ] Implement module-level caching pattern (cachedOptions + loadPromise)
- [ ] Use API wrapper (not raw fetch)
- [ ] Add `showSearch` for searchability
- [ ] Add `filterOption` for client-side filtering
- [ ] Export `clearCache()` function for invalidation
- [ ] Test with multiple instances rendered simultaneously
- [ ] Verify only ONE API call is made regardless of instance count

---

## Frontend — Dynamic Types (CoreExtensions)

For each custom Pimcore field type:
- [ ] Create `DynamicTypeObjectData*` class
- [ ] Verify `id` matches PHP `$fieldtype` string exactly
- [ ] Choose correct base class:
  - Single select → `DynamicTypeObjectDataAbstractSelect`
  - Multi-select → `DynamicTypeObjectDataAbstractMultiSelect`
  - Custom rendering → `DynamicTypeObjectDataAbstract`
- [ ] Register in `onInit()` (NOT `onStartup()`)
- [ ] Test in Pimcore Data Object class definition editor
- [ ] Test in Data Object edit view

---

## Frontend — Toolbar/Menu Registration

- [ ] Register widgets via `WidgetRegistry` in `onStartup()`
- [ ] Map ExtJS menu items to widget registrations
- [ ] Verify widgets appear in Studio v2 navigation

---

## Frontend — Icons

- [ ] Register custom icons in icon library module
- [ ] Use SVG format for icons
- [ ] Register via icon library service

---

## Translations

- [ ] Create `Resources/translations/studio.en.yaml`
- [ ] Copy relevant keys from `admin.en.yml`
- [ ] Use `useTranslation()` hook in components
- [ ] Verify all user-visible strings use `t('key')` (no hardcoded strings)
- [ ] Add translations for new UI elements not in ExtJS version

---

## State Management Migration

- [ ] Replace ExtJS store state with React `useState`/`useReducer`
- [ ] Replace global state (`pimcore.globalmanager`) with DI container or React context
- [ ] Ensure data flows top-down (props) with callbacks up
- [ ] Convert `on('event', handler)` → React `onChange`/`onClick` props
- [ ] Convert `fireEvent()` → callback prop invocation

---

## Post-Migration Verification

- [ ] All ExtJS functionality is replicated in Studio v2
- [ ] CRUD operations work (create, read, update, delete)
- [ ] Form saves correctly via API
- [ ] List/grid loads and displays all data
- [ ] Select components don't make duplicate API calls
- [ ] Dynamic types work in class definition editor
- [ ] Translations display correctly in all locations
- [ ] No console errors in browser
- [ ] No TypeScript errors
- [ ] Build succeeds
- [ ] UI is responsive and usable

---

## Documentation

- [ ] Document the new Studio v2 components
- [ ] Document API endpoints
- [ ] Document extension points (if applicable)
- [ ] Update migration notes for users upgrading

---

## Cleanup

- [ ] Remove or deprecate ExtJS files (if full migration)
- [ ] Update bundle class (`getJsPaths()`/`getCssPaths()`) if needed
- [ ] Remove unused translation keys
- [ ] Remove unused backend admin controllers (if replaced by REST API)