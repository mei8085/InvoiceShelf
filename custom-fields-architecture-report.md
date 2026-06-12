# InvoiceShelf 自定义字段三位一体架构分析报告

## 概述

InvoiceShelf 的自定义字段系统采用 **"存储-渲染-查询"三位一体** 架构，核心设计思想是通过一个 **类型 → 列映射函数** `getCustomFieldValueKey()` 在三个维度间建立统一桥梁，使不同字段类型的差异在存储层、表单渲染层、查询/输出层保持一致性。

---

## 一、存储结构：多态 EAV 模式

### 1.1 双表设计

系统使用两张核心表实现自定义字段的定义与赋值：

#### `custom_fields` 表 — 字段定义（Schema）

| 列名 | 类型 | 说明 |
|---|---|---|
| `id` | bigIncrements | 主键 |
| `name` | string | 字段内部名称 |
| `slug` | string | 唯一标识符，格式 `CUSTOM_{ModelType}_{Name}` |
| `label` | string | 用户可见标签 |
| `model_type` | string | 所属模型类型（Customer/Invoice/Estimate/Expense/Payment/Item） |
| `type` | string | 字段数据类型（Input/TextArea/Phone/Url/Number/Dropdown/Switch/Date/Time/DateTime） |
| `placeholder` | string | 可选占位符 |
| `options` | json | 下拉选项（仅 Dropdown 类型使用） |
| `boolean_answer` | boolean | 默认值列：布尔型 |
| `date_answer` | date | 默认值列：日期型 |
| `time_answer` | time | 默认值列：时间型 |
| `string_answer` | text | 默认值列：字符串型 |
| `number_answer` | unsignedBigInteger | 默认值列：数值型 |
| `date_time_answer` | dateTime | 默认值列：日期时间型 |
| `is_required` | boolean | 是否必填 |
| `order` | unsignedBigInteger | 排序序号 |
| `company_id` | unsignedBigInteger | 所属公司（多租户隔离） |

> 参考：[2020_02_01_063235_create_custom_fields_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/database/migrations/2020_02_01_063235_create_custom_fields_table.php#L14-L34)

#### `custom_field_values` 表 — 字段赋值（Data）

| 列名 | 类型 | 说明 |
|---|---|---|
| `id` | bigIncrements | 主键 |
| `custom_field_valuable_type` | string | 多态类型（如 `App\Models\Invoice`） |
| `custom_field_valuable_id` | unsignedInteger | 多态 ID |
| `type` | string | 冗余存储字段类型（与 `custom_fields.type` 一致） |
| `boolean_answer` | boolean | 值列：布尔型 |
| `date_answer` | date | 值列：日期型 |
| `time_answer` | time | 值列：时间型 |
| `string_answer` | text | 值列：字符串型 |
| `number_answer` | unsignedBigInteger | 值列：数值型 |
| `date_time_answer` | dateTime | 值列：日期时间型 |
| `custom_field_id` | unsignedBigInteger | 关联字段定义 |
| `company_id` | unsignedBigInteger | 所属公司 |

> 参考：[2020_02_01_063509_create_custom_field_values_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/database/migrations/2020_02_01_063509_create_custom_field_values_table.php#L14-L29)

### 1.2 存储设计分析

这是经典的 **EAV（Entity-Attribute-Value）模式**变体，结合了 **Laravel 多态关系**：

```
┌─────────────────┐       1:N       ┌──────────────────────────┐
│  custom_fields   │───────────────▶│  custom_field_values      │
│  (字段定义)       │                │  (字段赋值)               │
│                  │                │                          │
│  type: "Date"    │                │  custom_field_valuable_*  │──▶ Invoice
│  date_answer:默认 │                │  type: "Date"             │──▶ Customer
│                  │                │  date_answer: 实际值       │──▶ Estimate ...
└─────────────────┘                └──────────────────────────┘
```

**关键设计决策**：

1. **多答案列（Multi-Column Value）**：不使用单一 `value TEXT` 列，而是为每种数据类型设置独立的类型化列（`string_answer`、`number_answer`、`date_answer` 等）。这保留了数据库层面的类型约束和索引能力。

2. **多态关联（Polymorphic Relation）**：通过 `custom_field_valuable_type` + `custom_field_valuable_id` 实现一张值表服务于多种业务模型（Invoice、Customer、Expense、Payment、Estimate、InvoiceItem 等）。

3. **`type` 冗余字段**：`custom_field_values` 表冗余存储了 `type`，避免每次读取值时都需要 JOIN `custom_fields` 表来确定应读取哪个 answer 列。

### 1.3 类型 → 列映射：核心桥梁函数

整个系统的类型适配由一个核心函数 [getCustomFieldValueKey()](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Space/helpers.php#L87-L123) 驱动：

```php
function getCustomFieldValueKey(string $type)
{
    switch ($type) {
        case 'Input':    return 'string_answer';
        case 'TextArea': return 'string_answer';
        case 'Phone':    return 'number_answer';
        case 'Url':      return 'string_answer';
        case 'Number':   return 'number_answer';
        case 'Dropdown': return 'string_answer';
        case 'Switch':   return 'boolean_answer';
        case 'Date':     return 'date_answer';
        case 'Time':     return 'time_answer';
        case 'DateTime': return 'date_time_answer';
        default:         return 'string_answer';
    }
}
```

**映射关系总览**：

| 字段类型 (type) | 存储列 (answer key) | 数据库列类型 |
|---|---|---|
| Input | `string_answer` | text |
| TextArea | `string_answer` | text |
| Phone | `number_answer` | unsignedBigInteger |
| Url | `string_answer` | text |
| Number | `number_answer` | unsignedBigInteger |
| Dropdown | `string_answer` | text |
| Switch | `boolean_answer` | boolean |
| Date | `date_answer` | date |
| Time | `time_answer` | time |
| DateTime | `date_time_answer` | dateTime |

> **注意**：Phone 类型使用 `number_answer` 而非 `string_answer`，这意味着电话号码存为整数，可能丢失前导零和特殊字符（如 `+`、`-`），这是一个已知的架构妥协。

### 1.4 模型层

#### [CustomField](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/CustomField.php) 模型

- 通过 `getDefaultAnswerAttribute()` 访问器动态读取默认值：根据 `type` 调用 `getCustomFieldValueKey()` 确定列名，返回 `$this->$value_type`
- `createCustomField()` 和 `updateCustomField()` 静态方法在写入时同样通过映射函数确定目标列
- `setTimeAnswerAttribute()` mutator 确保时间值格式化为 `H:i:s`

#### [CustomFieldValue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/CustomFieldValue.php) 模型

- 同样通过 `getDefaultAnswerAttribute()` 访问器实现值的多态读取
- `customFieldValuable()` 定义多态反向关联
- `customField()` 关联到字段定义

#### [HasCustomFieldsTrait](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Traits/HasCustomFieldsTrait.php)

这是业务模型（Invoice、Customer 等）接入自定义字段系统的 trait：

| 方法 | 作用 |
|---|---|
| `fields()` | 定义 `MorphMany` 关联到 `CustomFieldValue` |
| `addCustomFields($customFields)` | 创建新记录，通过 `getCustomFieldValueKey()` 映射写入对应列 |
| `updateCustomFields($customFields)` | `firstOrCreate` 查找现有值再更新 |
| `getCustomFieldBySlug($slug)` | 通过 slug 查找特定自定义字段的值记录 |
| `getCustomFieldValueBySlug($slug)` | 直接获取字段的 `defaultAnswer` |

使用此 Trait 的模型：[Invoice](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Invoice.php#L26)、[Customer](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Customer.php)、[Estimate](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Estimate.php)、[Expense](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Expense.php)、[Payment](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Payment.php)、[InvoiceItem](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/InvoiceItem.php#L15)、[EstimateItem](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/EstimateItem.php)、[RecurringInvoice](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/RecurringInvoice.php)

---

## 二、表单渲染：动态组件分发

### 2.1 管理端自定义字段配置（CustomFieldModal）

自定义字段的创建/编辑由 [CustomFieldModal.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/modal-components/custom-fields/CustomFieldModal.vue) 负责，其核心渲染逻辑：

```
用户选择类型 → selectedType 变更 → 动态加载对应 Type 组件
```

**默认值输入框的动态组件分发**（关键代码）：

```vue
<component
  :is="defaultValueComponent"
  v-model="customFieldStore.currentCustomField.default_answer"
  :options="customFieldStore.currentCustomField.options"
/>
```

其中 `defaultValueComponent` 计算属性：

```js
const defaultValueComponent = computed(() => {
  if (customFieldStore.currentCustomField.type) {
    return defineAsyncComponent(() =>
      import(`../../custom-fields/types/${customFieldStore.currentCustomField.type}Type.vue`)
    )
  }
  return false
})
```

### 2.2 单据表单中的自定义字段渲染（CreateCustomFields）

在单据创建/编辑页面（如 InvoiceCreate），自定义字段通过 [CreateCustomFields.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/CreateCustomFields.vue) 渲染：

1. 组件挂载时通过 `customFieldStore.fetchCustomFields({ type: props.type, limit: 'all' })` 获取该模型类型的所有自定义字段
2. 设置每个字段的 `value = default_answer`（默认值预填）
3. 如果是编辑模式，通过 `mergeExistingValues()` 将已保存的字段值合并覆盖默认值
4. 按 `order` 排序后渲染

每个字段通过 [CreateCustomFieldsSingle.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/CreateCustomFieldsSingle.vue) 渲染单个字段：

```vue
<component
  :is="getTypeComponent"
  v-model="field.value"
  :options="field.options"
  :invalid="v$.value.$error"
  :placeholder="field.placeholder"
/>
```

其中 `getTypeComponent` 同样采用动态导入：

```js
const getTypeComponent = computed(() => {
  if (props.field.type) {
    return defineAsyncComponent(() =>
      import(`./types/${props.field.type}Type.vue`)
    )
  }
  return false
})
```

### 2.3 十种类型组件详解

所有类型组件位于 [types/](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types) 目录下，每个组件遵循统一的 `v-model` 接口协议：

| 类型组件 | 基础组件 | 特殊处理 |
|---|---|---|
| [InputType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/InputType.vue) | `BaseInput type="text"` | 无 |
| [TextAreaType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/TextAreaType.vue) | `BaseTextarea` | 接收 `rows` 和 `inputName` props |
| [PhoneType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/PhoneType.vue) | `BaseInput type="tel"` | modelValue 支持 String/Number |
| [UrlType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/UrlType.vue) | `BaseInput type="url"` | 无 |
| [NumberType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/NumberType.vue) | `BaseInput type="number"` | modelValue 支持 String/Number |
| [DropdownType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/DropdownType.vue) | `BaseMultiselect` | 接收 `options`、`valueProp`、`label`、`object` props |
| [SwitchType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/SwitchType.vue) | `BaseSwitch` | **值转换**：`get() => modelValue === 1`，`set() => value ? 1 : 0`（前端 Boolean ↔ 后端 int 转换） |
| [DateType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/DateType.vue) | `BaseDatePicker` | 默认值 `moment().format('YYYY-MM-DD')` |
| [TimeType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/TimeType.vue) | `BaseTimePicker` | modelValue 支持 String/Date/Object |
| [DateTimeType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/DateTimeType.vue) | `BaseDatePicker enable-time` | 默认值 `moment().format('YYYY-MM-DD hh:MM')` |

### 2.4 前端数据流

```
┌──────────────────────┐
│  custom-field store   │  ←→  API: /api/v1/custom-fields
│  (Pinia)              │
└──────────┬───────────┘
           │ fetchCustomFields({ type: 'Invoice', limit: 'all' })
           ▼
┌──────────────────────┐
│ CreateCustomFields    │  按模型类型获取字段定义
│   ↓ 设置 value=default_answer
│   ↓ 编辑模式 mergeExistingValues()
└──────────┬───────────┘
           │ v-for field in customFields
           ▼
┌──────────────────────┐
│ CreateCustomFields-   │  单字段渲染
│ Single                │  → 动态组件 import(`./types/${field.type}Type.vue`)
│   ↓ v-model="field.value"
└──────────────────────┘
           │
           ▼ 提交时 store[storeProp].customFields 作为 customFields 参数
```

### 2.5 编辑模式下的值合并

[CreateCustomFields.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/CreateCustomFields.vue#L63-L91) 的 `mergeExistingValues()` 函数处理编辑场景：

1. 遍历已保存的 `fields`（CustomFieldValue 集合）
2. 通过 `custom_field_id` 匹配到对应的自定义字段定义
3. 读取 `field.default_answer`（即 `defaultAnswer` 访问器返回的值）
4. **DateTime 类型的特殊格式化**：`moment(field.default_answer, 'YYYY-MM-DD HH:mm:ss').format('YYYY-MM-DD HH:mm')`
5. 覆盖 `customFields` 数组中的 `value`、`label`、`options`、`is_required`、`placeholder`、`order`

---

## 三、查询过滤与输出

### 3.1 API 层

#### 自定义字段 CRUD

[CustomFieldsController](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Http/Controllers/V1/Admin/CustomField/CustomFieldsController.php) 提供标准 RESTful 接口：

- `GET /api/v1/custom-fields` — 列表，支持 `type`（模型类型）和 `search` 过滤
- `POST /api/v1/custom-fields` — 创建
- `GET /api/v1/custom-fields/{id}` — 详情
- `PUT /api/v1/custom-fields/{id}` — 更新
- `DELETE /api/v1/custom-fields/{id}` — 删除（级联删除关联值）

过滤逻辑由 [CustomField::applyFilters()](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/CustomField.php#L90-L106) 实现：

```php
public function scopeApplyFilters($query, array $filters)
{
    if ($filters->get('type')) {
        $query->whereType($filters->get('type'));
    }
    if ($filters->get('search')) {
        $query->whereSearch($filters->get('search'));
    }
}
```

其中 `whereType` 按 `model_type` 过滤（如只返回 Invoice 类型的自定义字段），`whereSearch` 在 `label` 和 `name` 上做模糊搜索。

#### 单据 CRUD 中的自定义字段处理

以 Invoice 为例，在 [Invoice::createInvoice()](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Invoice.php#L359-L361) 中：

```php
if ($request->customFields) {
    $invoice->addCustomFields($request->customFields);
}
```

在 [Invoice::updateInvoice()](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/Invoice.php#L437-L439) 中：

```php
if ($request->customFields) {
    $this->updateCustomFields($request->customFields);
}
```

### 3.2 API Resource 层

#### [CustomFieldResource](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Http/Resources/CustomFieldResource.php)

返回字段定义时，**同时返回所有 6 个 answer 列和计算属性 `default_answer`**：

```php
return [
    'boolean_answer'    => $this->boolean_answer,
    'date_answer'       => $this->date_answer,
    'time_answer'       => $this->time_answer,
    'string_answer'     => $this->string_answer,
    'number_answer'     => $this->number_answer,
    'date_time_answer'  => $this->date_time_answer,
    'default_answer'    => $this->default_answer,   // ← 计算属性，动态取正确列
];
```

前端使用 `default_answer` 作为默认值，无需关心底层存储在哪个列。

#### [CustomFieldValueResource](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Http/Resources/CustomFieldValueResource.php)

返回字段值时，提供两个值：

- `default_answer` — 原始值（由 `getDefaultAnswerAttribute()` 返回）
- `default_formatted_answer` — 格式化值（由 `dateTimeFormat()` 处理）
  - `date_time_answer` → `Carbon::parse()->format('Y-m-d H:i')`
  - `date_answer` → 按公司日期格式化
  - 其他 → 原值返回

#### 业务模型 Resource 中的嵌入

所有使用自定义字段的模型 Resource 都以相同模式嵌入字段值：

```php
'fields' => $this->when($this->fields()->exists(), function () {
    return CustomFieldValueResource::collection($this->fields);
}),
```

示例：[InvoiceResource](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Http/Resources/InvoiceResource.php#L72-L74)、[CustomerResource](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Http/Resources/CustomerResource.php#L46-L48)

### 3.3 PDF 模板中的查询

在 PDF 生成时，自定义字段通过两种方式输出：

1. **模板变量替换**：[GeneratesPdfTrait::getFieldsArray()](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Traits/GeneratesPdfTrait.php#L154-L163) 将自定义字段值映射为 `{SLUG}` 格式的模板变量：

```php
$customFields = $this->fields;
foreach ($customFields as $customField) {
    $fields['{'.$customField->customField->slug.'}'] = $customField->defaultAnswer;
}
```

2. **PDF 表格直接输出**：[table.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/views/app/pdf/invoice/partials/table.blade.php#L5-L7) 在 InvoiceItem 表格中遍历自定义字段列：

```blade
@foreach($customFields as $field)
    <th>{{ $field->label }}</th>
@endforeach
...
@foreach($customFields as $field)
    <td>{{ $item->getCustomFieldValueBySlug($field->slug) }}</td>
@endforeach
```

3. **富文本编辑器插入**：[BaseCustomInput.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/components/base/BaseCustomInput.vue) 提供可视化下拉菜单，允许用户在 Notes 等富文本字段中插入自定义字段占位符 `{SLUG}`。

### 3.4 按模型类型过滤查询

自定义字段在多处被按 `model_type` 过滤查询：

- 前端表单加载时：`fetchCustomFields({ type: 'Invoice', limit: 'all' })`
- PDF 生成时：`CustomField::where('model_type', 'Item')->get()`
- BaseCustomInput 中：按 `model_type` 分组显示可插入字段

---

## 四、不同类型差异的适配机制

### 4.1 三层适配总览

```
                  ┌─────────────────────────────────────────┐
                  │          getCustomFieldValueKey()         │
                  │    类型 → 列名映射（核心路由表）           │
                  └───────┬──────────┬──────────┬────────────┘
                          │          │          │
              ┌───────────▼──┐  ┌────▼─────┐  ┌─▼──────────────┐
              │  存储层适配    │  │ 渲染层适配 │  │ 输出层适配      │
              │  确定写入列    │  │ 确定组件   │  │ 确定格式化方式  │
              └──────────────┘  └───────────┘  └────────────────┘
```

### 4.2 各类型在三层中的完整适配对照

| 字段类型 | 存储层（列+类型） | 渲染层（Vue 组件） | 输出/格式化层 |
|---|---|---|---|
| **Input** | `string_answer` (text) | `BaseInput type="text"` | 直接输出 |
| **TextArea** | `string_answer` (text) | `BaseTextarea` | 直接输出 |
| **Phone** | `number_answer` (bigint) | `BaseInput type="tel"` | 直接输出 |
| **Url** | `string_answer` (text) | `BaseInput type="url"` | 直接输出 |
| **Number** | `number_answer` (bigint) | `BaseInput type="number"` | 直接输出 |
| **Dropdown** | `string_answer` (text) | `BaseMultiselect` | 直接输出（选项文本） |
| **Switch** | `boolean_answer` (bool) | `BaseSwitch` + `1/0 ↔ true/false` 转换 | 直接输出（1 或 0） |
| **Date** | `date_answer` (date) | `BaseDatePicker` | `Carbon::parse()->format(公司日期格式)` |
| **Time** | `time_answer` (time) | `BaseTimePicker` | `date('H:i:s', strtotime())` 格式化 |
| **DateTime** | `date_time_answer` (dateTime) | `BaseDatePicker enable-time` | `Carbon::parse()->format('Y-m-d H:i')` |

### 4.3 关键适配点详解

#### Switch 类型的双向转换

Switch 类型在前端和后端之间需要进行 Boolean ↔ Integer 的转换：

- **前端**：[SwitchType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/SwitchType.vue) 中 `get() => modelValue === 1`，`set() => value ? 1 : 0`
- **后端**：数据库存储为 `boolean` 类型（0/1），`defaultAnswer` 访问器直接返回原始值

#### Time 类型的格式化

- **前端**：[TimeType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/TimeType.vue) 使用 `BaseTimePicker`，modelValue 为对象 `{HH, mm, ss}`
- **提交时转换**：[CustomFieldModal.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/modal-components/custom-fields/CustomFieldModal.vue#L347-L362) 将 `{HH, mm, ss}` 对象转为 `HH:mm` 字符串
- **后端**：[CustomField](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Models/CustomField.php#L34-L38) 的 `setTimeAnswerAttribute()` mutator 将其格式化为 `H:i:s`

#### DateTime 类型的格式化

- **编辑回填**：[CreateCustomFields.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/CreateCustomFields.vue#L73-L76) 将 `YYYY-MM-DD HH:mm:ss` 转为 `YYYY-MM-DD HH:mm`（去掉秒）
- **API 输出**：[CustomFieldValueResource::dateTimeFormat()](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Http/Resources/CustomFieldValueResource.php#L43-L61) 将 `date_time_answer` 格式化为 `Y-m-d H:i`

#### Dropdown 类型的选项管理

- **存储**：`options` 列以 JSON 数组形式存储（如 `["选项1","选项2"]`）
- **前端配置**：[CustomFieldModal.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/DropdownType.vue) 中选项以 `{ name: "选项" }` 对象数组形式操作
- **提交转换**：提交时 `options.map(option => option.name)` 将对象数组还原为字符串数组
- **渲染**：[DropdownType.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types/DropdownType.vue) 使用 `BaseMultiselect`，通过 `valueProp="name"` 和 `label="name"` 确定显示和取值

---

## 五、数据流全链路追踪

以"创建一张 Invoice 并填写自定义字段"为例：

```
1. 页面加载
   InvoiceCreate.vue → CreateCustomFields(type="Invoice")
     → customFieldStore.fetchCustomFields({type:'Invoice', limit:'all'})
     → GET /api/v1/custom-fields?type=Invoice&limit=all
     → CustomFieldsController::index() → CustomField::applyFilters(['type'=>'Invoice'])
     → 返回 Invoice 类型的所有字段定义（含 default_answer）

2. 用户填写表单
   CreateCustomFieldsSingle.vue → 动态加载 ${field.type}Type.vue
     → v-model 绑定 field.value
     → Switch 类型自动 1/0 ↔ true/false 转换
     → Time 类型对象 → 字符串转换

3. 提交表单
   InvoiceCreate.vue → invoiceStore.addInvoice(data)
     → data.customFields = [{id: 1, value: "xxx"}, {id: 2, value: 1}, ...]
     → POST /api/v1/invoices

4. 后端存储
   Invoice::createInvoice($request)
     → $invoice->addCustomFields($request->customFields)
     → 遍历每个字段：
         CustomField::find($field['id'])           // 找到定义
         getCustomFieldValueKey($customField->type) // 确定列名
         $this->fields()->create([                 // 多态创建
             'type' => $customField->type,
             'custom_field_id' => $customField->id,
             getCustomFieldValueKey($type) => $field['value'],  // 写入对应列
         ])

5. 读取返回
   InvoiceResource::toArray()
     → 'fields' => CustomFieldValueResource::collection($this->fields)
     → 每个值包含 default_answer（动态取值）和 default_formatted_answer（格式化）
```

---

## 六、架构优缺点分析

### 优点

1. **类型安全存储**：多列 EAV 模式保留了数据库类型约束，比单列 TEXT 存储更安全
2. **统一映射函数**：`getCustomFieldValueKey()` 作为唯一路由表，避免散落的 if/else，新增类型只需修改一处
3. **前端组件自动分发**：通过 `defineAsyncComponent(() => import(...))` 按类型名动态加载，新增类型只需添加一个 Vue 文件
4. **多态关联复用**：一张 `custom_field_values` 表服务所有业务模型，无需为每个模型建表
5. **PDF 模板集成**：自定义字段值通过 slug 注入模板变量，支持富文本和表格两种输出形式

### 不足与改进空间

1. **Phone 类型存为整数**：丢失前导零和特殊字符（如国际区号 `+86`），应改为 `string_answer`
2. **type 冗余的一致性风险**：`custom_field_values.type` 与 `custom_fields.type` 冗余存储，若字段定义类型变更，已有值记录的 `type` 不会自动同步
3. **无自定义字段索引**：`custom_field_values` 表缺少 `custom_field_valuable_type` + `custom_field_valuable_id` 的联合索引，大数据量下查询可能较慢
4. **值更新策略为全量覆盖**：`updateCustomFields()` 使用 `firstOrCreate` + 逐条 save，不支持删除某个字段值（前端传空值仍会创建记录）
5. **组件命名约定耦合**：前端动态 `import(./types/${type}Type.vue)` 要求 Vue 文件名与后端 `type` 字符串严格一致，缺少注册表或验证机制

---

## 七、扩展新类型的步骤

若需新增一种自定义字段类型（如 `Email`），需修改以下位置：

| 步骤 | 位置 | 操作 |
|---|---|---|
| 1 | [helpers.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/app/Space/helpers.php) `getCustomFieldValueKey()` | 添加 `case 'Email': return 'string_answer';` |
| 2 | 新建 `EmailType.vue` | 放在 [types/](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/custom-fields/types) 目录，使用 `BaseInput type="email"` |
| 3 | [CustomFieldModal.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/resources/scripts/admin/components/modal-components/custom-fields/CustomFieldModal.vue) `dataTypes` | 添加 `{ label: 'Email', value: 'Email' }` |
| 4 | [CustomFieldFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/23-InvoiceShelf/database/factories/CustomFieldFactory.php) | 在 `type` 的 `randomElement` 中添加 `'Email'` |
| 5 | 测试 | 确保存储、渲染、输出三层一致 |

**无需修改**：数据库迁移（复用现有 `string_answer` 列）、Model、Trait、Resource — 这是该架构的核心优势。
