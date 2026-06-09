# 前端 Store 与后端资源类协作指南

本文档梳理 InvoiceShelf 项目中前端状态管理（Pinia Store）、后端 API 资源类（Laravel Eloquent API Resource）以及字段序列化之间的协作关系。

> **路径约定**：本文档中所有文件路径均为**仓库相对路径**，以项目根目录 `125-InvoiceShelf/` 为基准。

---

## 一、整体架构概览

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐
│  Database   │────▶│  Eloquent    │────▶│  API Resource │────▶│  JSON API   │
│  (数据表)    │     │  Model       │     │  (序列化)     │     │  Response   │
└─────────────┘     └──────────────┘     └──────────────┘     └──────┬──────┘
                                                                      │
                                                                      ▼
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐
│  Vue 组件   │◀────│  Pinia Store │◀────│  Axios HTTP  │◀────│  JSON Data  │
│  (视图层)    │     │  (状态层)    │     │  (请求层)     │     │  (响应体)    │
└─────────────┘     └──────────────┘     └──────────────┘     └─────────────┘
```

**核心目录对应关系：**

| 层级 | 目录路径 | 角色 |
|------|---------|------|
| 后端模型 | `app/Models/*.php` | 数据库 ORM 映射、业务逻辑、访问器 |
| 后端资源 | `app/Http/Resources/*.php` | 字段序列化、嵌套关系、数据塑形 |
| 后端控制器 | `app/Http/Controllers/V1/Admin/**/*.php` | 业务入口、权限校验、返回资源 |
| 前端 HTTP | `resources/scripts/http/index.js` | Axios 实例、拦截器、请求封装 |
| 前端 Store | `resources/scripts/admin/stores/*.js` | Pinia 状态管理、API 调用、数据转换 |
| 前端 Stub | `resources/scripts/admin/stub/*.js` | 表单初始化默认数据结构 |
| 前端视图 | `resources/scripts/admin/views/**/*.vue` | 界面渲染、用户交互 |

---

## 二、后端资源类（Resource）序列化机制

### 2.1 资源类的作用

Laravel 的 Eloquent API Resource 充当**数据转换层**，负责：
- 将 Eloquent Model 转换为前端需要的 JSON 结构
- 控制哪些字段暴露给前端
- 处理嵌套关系的序列化
- 添加计算属性、格式化字段

### 2.2 资源类结构示例

以 `app/Http/Resources/InvoiceResource.php` 为例，该类继承 `JsonResource`，通过 `toArray()` 方法定义序列化输出结构。

**代码证据**：`app/Http/Resources/InvoiceResource.php` 第 8-82 行

```php
class InvoiceResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            // 1. 基础字段（直接从 Model 属性读取）
            'id' => $this->id,
            'invoice_number' => $this->invoice_number,
            'invoice_date' => $this->invoice_date,
            'status' => $this->status,
            'sub_total' => $this->sub_total,
            'total' => $this->total,
            // ... 更多字段

            // 2. 访问器字段（Model 中 getXxxAttribute 方法）
            'formatted_created_at' => $this->formattedCreatedAt,
            'invoice_pdf_url' => $this->invoicePdfUrl,
            'formatted_invoice_date' => $this->formattedInvoiceDate,
            'allow_edit' => $this->allow_edit,
            'payment_module_enabled' => $this->payment_module_enabled,

            // 3. 条件加载的嵌套关系（when + 子 Resource）
            'items' => $this->when($this->items()->exists(), function () {
                return InvoiceItemResource::collection($this->items);
            }),
            'customer' => $this->when($this->customer()->exists(), function () {
                return new CustomerResource($this->customer);
            }),
            'taxes' => $this->when($this->taxes()->exists(), function () {
                return TaxResource::collection($this->taxes);
            }),
            'currency' => $this->when($this->currency()->exists(), function () {
                return new CurrencyResource($this->currency);
            }),
        ];
    }
}
```

### 2.3 字段来源的三种类型

**字段不是"拉平"的，而是分层嵌套的。** 资源类中的字段按来源分为三类：

#### 类型 A：数据库直接字段

直接对应数据库表中的列，通过 `$this->字段名` 访问。

**代码证据**：`app/Http/Resources/InvoiceResource.php` 第 18-59 行

```php
'id' => $this->id,
'invoice_date' => $this->invoice_date,
'due_date' => $this->due_date,
'invoice_number' => $this->invoice_number,
'status' => $this->status,
'paid_status' => $this->paid_status,
'tax_per_item' => $this->tax_per_item,
'sub_total' => $this->sub_total,
'total' => $this->total,
'customer_id' => $this->customer_id,
// ...
```

- 对应模型：`app/Models/Invoice.php` 第 52-74 行（`$guarded` 和 `$casts` 属性定义了字段和类型转换）

#### 类型 B：Eloquent 访问器（Accessors）

在 Model 中通过 `getXxxAttribute()` 方法定义的计算属性，在资源类中以驼峰/蛇形命名访问。这些属性会被追加到模型数组/JSON 输出中。

**代码证据**：`app/Models/Invoice.php` 第 56-62 行（注册访问器到 `$appends`）

```php
protected $appends = [
    'formattedCreatedAt',
    'formattedInvoiceDate',
    'formattedDueDate',
    'formattedDueAmount',
    'invoicePdfUrl',
];
```

**代码证据**：`app/Models/Invoice.php` 第 126-129 行（`getInvoicePdfUrlAttribute` 访问器）

```php
public function getInvoicePdfUrlAttribute()
{
    return url('/invoices/pdf/'.$this->unique_hash);
}
```

**代码证据**：`app/Models/Invoice.php` 第 140-162 行（`getAllowEditAttribute` 访问器，包含复杂业务逻辑）

```php
public function getAllowEditAttribute()
{
    $retrospective_edit = CompanySetting::getSetting('retrospective_edits', $this->company_id);
    $allowed = true;
    $status = [self::STATUS_DRAFT, self::STATUS_SENT, ...];
    // ... 多条件判断
    return $allowed;
}
```

在 Resource 中直接使用这些访问器字段：

**代码证据**：`app/Http/Resources/InvoiceResource.php` 第 51-56 行

```php
'formatted_created_at' => $this->formattedCreatedAt,
'invoice_pdf_url' => $this->invoicePdfUrl,
'formatted_invoice_date' => $this->formattedInvoiceDate,
'formatted_due_date' => $this->formattedDueDate,
'allow_edit' => $this->allow_edit,
'payment_module_enabled' => $this->payment_module_enabled,
```

#### 类型 C：嵌套关系资源

通过 `$this->when()` 条件加载，返回子 Resource 或 Resource 集合。
- 一对一关系用 `new XxxResource($this->relation)`
- 一对多关系用 `XxxResource::collection($this->relation)`

**代码证据**：`app/Http/Resources/InvoiceResource.php` 第 60-80 行

```php
'items' => $this->when($this->items()->exists(), function () {
    return InvoiceItemResource::collection($this->items);
}),
'customer' => $this->when($this->customer()->exists(), function () {
    return new CustomerResource($this->customer);
}),
'creator' => $this->when($this->creator()->exists(), function () {
    return new UserResource($this->creator);
}),
'taxes' => $this->when($this->taxes()->exists(), function () {
    return TaxResource::collection($this->taxes);
}),
'currency' => $this->when($this->currency()->exists(), function () {
    return new CurrencyResource($this->currency);
}),
```

对应的模型关系定义：

**代码证据**：`app/Models/Invoice.php` 第 86-124 行

```php
public function items(): HasMany
{
    return $this->hasMany(InvoiceItem::class);
}

public function customer(): BelongsTo
{
    return $this->belongsTo(Customer::class, 'customer_id');
}

public function taxes(): HasMany
{
    return $this->hasMany(Tax::class);
}

public function currency(): BelongsTo
{
    return $this->belongsTo(Currency::class);
}

public function creator(): BelongsTo
{
    return $this->belongsTo(User::class, 'creator_id');
}
```

---

### 2.4 关系字段加载机制深度解析

这是理解前端 store 能否拿到嵌套字段的核心。项目中**没有使用 Laravel 标准的 `whenLoaded()`**，而是统一使用 `$this->when($this->relation()->exists(), ...)` 模式。

#### 2.4.1 `$relation()->exists()` 的工作原理

```php
'items' => $this->when($this->items()->exists(), function () {
    return InvoiceItemResource::collection($this->items);
}),
```

这段代码的执行分为两步：

**第一步：`$this->items()->exists()` — 检查存在性**
- `$this->items()` 返回关系查询构造器（`HasMany` 实例），**不触发查询**
- `->exists()` 执行一条独立的 SQL：`SELECT EXISTS(SELECT * FROM invoice_items WHERE invoice_id = ?)`
- 返回 `true` 或 `false`
- **注意**：即使关联已经通过 `with()` 预加载了，`exists()` 仍然会执行新的 SQL 查询！

**第二步：闭包内 `$this->items` — 获取关联数据**
- 如果 `exists()` 返回 `true`，闭包执行
- `$this->items` 访问 Eloquent 的动态属性
  - 如果已通过 `with()` 预加载：直接从模型的 `$relations` 属性中取，不查数据库
  - 如果未预加载：触发**懒加载**，执行 SQL 查询获取所有关联记录
- 然后用子 Resource 序列化返回

**代码证据**：全项目 grep 验证（`app/Http/Resources/` 目录下）

| 模式 | 使用次数 |
|------|---------|
| `when(...->exists(), ...)` | 100+ 处（所有 Resource 文件） |
| `whenLoaded` | 0 处 |

> **重要结论**：项目中嵌套字段出现与否，**不取决于接口类型（列表/详情），也不取决于是否预加载，而是取决于数据库中是否存在关联记录**。只要数据库里有关联数据，`exists()` 就返回 true，然后触发懒加载把数据查出来返回。

#### 2.4.2 控制器 `with()` 预加载的作用

控制器中的 `with()` 是**性能优化手段**，不是"控制字段是否返回"的开关。

**代码证据**：`app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php` 第 27-31 行（列表接口）

```php
$invoices = Invoice::whereCompany()
    ->applyFilters($request->all())
    ->with('customer')       // 只预加载了 customer
    ->latest()
    ->paginateData($limit);
```

**代码证据**：`app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php` 第 65-70 行（详情接口）

```php
public function show(Request $request, Invoice $invoice)
{
    $this->authorize('view', $invoice);
    return new InvoiceResource($invoice);  // 没有 with() 预加载！
}
```

**预加载 vs 不预加载的对比：**

| 场景 | items 字段是否返回 | SQL 查询次数 | 性能 |
|------|-------------------|-------------|------|
| 列表接口 + `with('customer')` + 有 items 数据 | ✅ 返回 | 每条发票多 1 次查 items（懒加载） | 较差（N+1 问题） |
| 列表接口 + `with('customer')` + 无 items 数据 | ❌ 不返回 | 每条发票多 1 次 exists 查询 | 一般 |
| 详情接口 + 无 `with()` + 有 items 数据 | ✅ 返回 | 1 次查 items（懒加载） | 可接受 |
| 详情接口 + 无 `with()` + 无 items 数据 | ❌ 不返回 | 1 次 exists 查询 | 可接受 |
| 如果用 `with('items', 'customer', ...)` | ✅ 返回（有数据时） | 常量次数（eager loading） | 最好 |

#### 2.4.3 `whenLoaded()` 与 `when()->exists()` 的本质区别

Laravel 内置的 `whenLoaded()` 和项目使用的 `when()->exists()` 是两种完全不同的策略：

| 对比维度 | `whenLoaded('relation')` | `when($relation->exists(), ...)` |
|---------|--------------------------|---------------------------------|
| **判断依据** | 关联是否已被预加载（in-memory） | 数据库中是否存在关联记录 |
| **是否查库** | 不查库，只看 `$model->relations` 数组 | 每次都查库（`SELECT EXISTS`） |
| **控制方** | 控制器决定（通过 `with()`） | 数据决定（数据库有没有） |
| **列表接口行为** | 默认不返回嵌套数据（除非 with 了） | 只要有数据就返回（但有 N+1） |
| **性能** | 好（配合 with） | 差（每次 exists + 可能懒加载） |
| **语义** | "需要时才加载" | "有数据就带上" |
| **项目中是否使用** | ❌ 未使用 | ✅ 全部使用 |

#### 2.4.4 各接口的嵌套字段实际出现情况

以 Invoice 为例，实际各接口返回的嵌套字段情况：

| 嵌套字段 | 列表接口 (index) | 详情接口 (show) | 创建接口 (store) | 更新接口 (update) |
|---------|-----------------|----------------|----------------|----------------|
| customer | ✅ 有（预加载） | ✅ 有（懒加载） | ✅ 有（新建后查） | ✅ 有 |
| items | ✅ 有（懒加载，N+1） | ✅ 有（懒加载） | ✅ 有 | ✅ 有 |
| taxes | ✅ 有（懒加载，N+1） | ✅ 有（懒加载） | ✅ 有 | ✅ 有 |
| creator | ✅ 有（懒加载，N+1） | ✅ 有（懒加载） | ✅ 有 | ✅ 有 |
| currency | ✅ 有（懒加载，N+1） | ✅ 有（懒加载） | ✅ 有 | ✅ 有 |
| fields | ✅ 有（懒加载，N+1） | ✅ 有（懒加载） | ✅ 有 | ✅ 有 |
| company | ✅ 有（懒加载，N+1） | ✅ 有（懒加载） | ✅ 有 | ✅ 有 |

> **纠正之前的误解**：并不是"列表接口浅、详情接口深"，而是**所有接口都会返回所有存在的嵌套数据**。区别仅在于性能（是否预加载），不在于字段有无。列表接口因为是多条数据，懒加载导致的 N+1 问题更严重。

---

### 2.5 嵌套资源的递归结构

嵌套关系是**递归**的，每个子资源内部同样可能包含自身的嵌套关系：

```
InvoiceResource
├── id
├── invoice_number
├── customer (CustomerResource)
│   ├── id
│   ├── name
│   ├── email
│   ├── billing (AddressResource)
│   │   ├── id
│   │   ├── address_street_1
│   │   └── country (CountryResource)
│   └── currency (CurrencyResource)
├── items (InvoiceItemResource[])
│   ├── id
│   ├── name
│   ├── price
│   └── taxes (TaxResource[])
│       ├── id
│       ├── name
│       └── amount
└── taxes (TaxResource[])
```

**代码证据**：

- Invoice → items → taxes：`app/Http/Resources/InvoiceItemResource.php` 第 38-40 行
  ```php
  'taxes' => $this->when($this->taxes()->exists(), function () {
      return TaxResource::collection($this->taxes);
  }),
  ```

- Customer → billing：`app/Http/Resources/CustomerResource.php` 第 40-42 行
  ```php
  'billing' => $this->when($this->billingAddress()->exists(), function () {
      return new AddressResource($this->billingAddress);
  }),
  ```

- Address → country：`app/Http/Resources/AddressResource.php` 第 32-34 行
  ```php
  'country' => $this->when($this->country()->exists(), function () {
      return new CountryResource($this->country);
  }),
  ```

相关资源文件：
- `app/Http/Resources/CustomerResource.php`
- `app/Http/Resources/InvoiceItemResource.php`
- `app/Http/Resources/TaxResource.php`
- `app/Http/Resources/AddressResource.php`

### 2.6 资源类与集合类

- **单资源**：`new InvoiceResource($invoice)` — 返回单个对象，包装在 `data` 键下
- **资源集合**：`InvoiceResource::collection($invoices)` — 返回对象数组，包装在 `data` 键下，附带 `meta` 和 `links` 分页信息
- **自定义集合类**：`InvoiceCollection` — 通常用于添加额外的 meta 数据

**代码证据**：`app/Http/Resources/InvoiceCollection.php` 第 8-18 行

```php
class InvoiceCollection extends ResourceCollection
{
    public function toArray($request): array
    {
        return parent::toArray($request);
    }
}
```

### 2.7 控制器中的使用

控制器从数据库查询 Model（通常带着 `with()` 预加载关联），然后用 Resource 转换后返回。

**代码证据**：`app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php` 第 21-37 行（列表接口）

```php
public function index(Request $request)
{
    $this->authorize('viewAny', Invoice::class);
    $limit = $request->input('limit', 10);

    $invoices = Invoice::whereCompany()
        ->applyFilters($request->all())
        ->with('customer')       // 预加载关联，避免 N+1
        ->latest()
        ->paginateData($limit);

    return InvoiceResource::collection($invoices)
        ->additional(['meta' => [
            'invoice_total_count' => Invoice::whereCompany()->count(),
        ]]);
}
```

**代码证据**：`app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php` 第 65-70 行（详情接口）

```php
public function show(Request $request, Invoice $invoice)
{
    $this->authorize('view', $invoice);
    return new InvoiceResource($invoice);
}
```

---

## 三、前端 Store 的数据消费

### 3.1 HTTP 请求层

前端使用 Axios 封装的 HTTP 客户端发送请求，通过请求拦截器自动注入认证 token 和 company header。

**代码证据**：`resources/scripts/http/index.js` 第 1-39 行

```js
import axios from 'axios'
import Ls from '@/scripts/services/ls.js'

const instance = axios.create({
  withCredentials: true,
  headers: {
    common: {
      'X-Requested-With': 'XMLHttpRequest',
    },
  },
})

// 请求拦截器：自动注入 token 和 companyId
instance.interceptors.request.use(function (config) {
  const companyId = Ls.get('selectedCompany')
  const authToken = Ls.get('auth.token')
  if (authToken) config.headers.Authorization = authToken
  if (companyId) config.headers.company = companyId
  return config
})
```

### 3.2 Store 的基本结构

以 `resources/scripts/admin/stores/invoice.js` 为例，每个 Store 包含 state、getters、actions 三部分。

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 20-99 行

```js
import { defineStore } from 'pinia'
import http from '@/scripts/http'
import invoiceStub from '../stub/invoice'

export const useInvoiceStore = (useWindow = false) => {
  const defineStoreFunc = useWindow ? window.pinia.defineStore : defineStore

  return defineStoreFunc('invoice', {
    // state: 状态数据
    state: () => ({
      invoices: [],          // 列表数据
      selectedInvoices: [],
      invoiceTotalCount: 0,  // 总数
      newInvoice: {          // 当前编辑/新建的表单数据
        ...invoiceStub(),
      },
    }),

    // getters: 计算属性
    getters: {
      getInvoice: (state) => (id) => {
        let invId = parseInt(id)
        return state.invoices.find((invoice) => invoice.id === invId)
      },
      getSubTotal() { /* ... */ },
      getTotal() { /* ... */ },
      isEdit: (state) => (state.newInvoice.id ? true : false),
    },

    // actions: 方法（含 API 调用）
    actions: {
      fetchInvoices(params) { /* ... */ },
      fetchInvoice(id) { /* ... */ },
      addInvoice(data) { /* ... */ },
      updateInvoice(data) { /* ... */ },
      // ... 更多 actions
    },
  })()
}
```

### 3.3 从 API 到 Store 的数据流向

#### 列表数据获取

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 122-136 行

```js
fetchInvoices(params) {
  return new Promise((resolve, reject) => {
    http
      .get(`/api/v1/invoices`, { params })
      .then((response) => {
        // response.data.data 是后端 Resource 返回的数组
        this.invoices = response.data.data
        // response.data.meta 是额外元数据
        this.invoiceTotalCount = response.data.meta.invoice_total_count
        resolve(response)
      })
      .catch((err) => {
        handleError(err)
        reject(err)
      })
  })
}
```

后端响应的 JSON 结构（对应 `InvoiceResource::collection()` + `additional()`）：
```json
{
  "data": [
    { "id": 1, "invoice_number": "INV-001", "status": "SENT", "customer": { "id": 5, "name": "..." }, "items": [...] },
    { "id": 2, "invoice_number": "INV-002", "status": "DRAFT", "customer": { "id": 3, "name": "..." }, "items": [...] }
  ],
  "meta": {
    "invoice_total_count": 100
  },
  "links": {
    "first": "...", "last": "...", "prev": null, "next": "..."
  }
}
```

> **注意**：根据 2.4 节的分析，`items`、`taxes` 等嵌套字段在列表接口中也会出现（只要数据库中有关联数据），并不是只有详情接口才有。

#### 详情数据获取

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 138-152 行（`fetchInvoice`）

```js
fetchInvoice(id) {
  return new Promise((resolve, reject) => {
    http
      .get(`/api/v1/invoices/${id}`)
      .then((response) => {
        // 把 API 返回的数据设置到表单状态
        this.setInvoiceData(response.data.data)
        this.setCustomerAddresses(this.newInvoice.customer)
        resolve(response)
      })
      .catch((err) => {
        handleError(err)
        reject(err)
      })
  })
}
```

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 154-173 行（`setInvoiceData`）

```js
setInvoiceData(invoice) {
  // 关键：用 Object.assign 把后端数据合并到现有 newInvoice 对象
  Object.assign(this.newInvoice, invoice)

  // 额外的前端逻辑：补充空 tax 项
  if (this.newInvoice.tax_per_item === 'YES') {
    this.newInvoice.items.forEach((_i) => {
      if (_i.taxes && !_i.taxes.length)
        _i.taxes.push({ ...taxStub, id: Guid.raw() })
    })
  }

  // 折扣单位转换等前端特有逻辑
  if (this.newInvoice.discount_per_item === 'YES') {
    this.newInvoice.items.forEach((_i, index) => {
      if (_i.discount_type === 'fixed')
        this.newInvoice.items[index].discount = _i.discount / 100
    })
  } else {
    if (this.newInvoice.discount_type === 'fixed')
      this.newInvoice.discount = this.newInvoice.discount / 100
  }
}
```

**关键点**：`Object.assign(this.newInvoice, invoice)` 将后端返回的完整 JSON 对象（含嵌套的 customer、items、taxes 等）直接合并到前端状态对象中。**嵌套结构被完整保留，并没有被拉平。**

---

## 四、Stub（数据桩）的角色

### 4.1 什么是 Stub

Stub 是前端定义的**默认数据结构模板**，用于：
- 新建表单时的初始值
- 确保数据结构的完整性（即使后端没返回某些字段）
- 添加前端专用的字段（后端不需要）

### 4.2 Stub 示例

**代码证据**：`resources/scripts/admin/stub/invoice.js` 第 1-39 行

```js
import Guid from 'guid'
import invoiceItemStub from './invoice-item'
import taxStub from './tax'

export default function () {
  return {
    // 与后端对应的字段
    id: null,
    invoice_number: '',
    customer_id: null,
    invoice_date: '',
    due_date: '',
    sub_total: 0,
    total: 0,
    tax_per_item: null,
    tax_included: false,
    discount_per_item: null,

    // 嵌套结构（与后端 Resource 对应）
    customer: null,
    taxes: [],
    items: [
      {
        ...invoiceItemStub,
        id: Guid.raw(),
        taxes: [{ ...taxStub, id: Guid.raw() }],
      },
    ],

    // 前端专用字段（后端没有）
    customFields: [],
    fields: [],
    selectedNote: null,
    selectedCurrency: '',
  }
}
```

对应的子 stub：

**代码证据**：`resources/scripts/admin/stub/invoice-item.js` 第 1-18 行

```js
export default {
  invoice_id: null,
  item_id: null,
  name: '',
  title: '',
  description: null,
  quantity: 1,
  price: 0,
  discount_type: 'fixed',
  discount_val: 0,
  discount: 0,
  total: 0,
  totalTax: 0,
  totalSimpleTax: 0,
  totalCompoundTax: 0,
  tax: 0,
  taxes: [],
}
```

**代码证据**：`resources/scripts/admin/stub/address.js` 第 1-11 行

```js
export default {
  name: null,
  phone: null,
  address_street_1: null,
  address_street_2: null,
  city: null,
  state: null,
  country_id: null,
  zip: null,
  type: null,
}
```

### 4.3 Stub 与 Resource 的字段对比

| 来源 | 字段举例 | 说明 | 代码位置 |
|------|---------|------|---------|
| 两者共有 | `id`, `invoice_number`, `customer_id`, `sub_total`, `items`, `taxes` | 数据库字段，前后端对齐 | `app/Http/Resources/InvoiceResource.php` 第 18-50 行 / `resources/scripts/admin/stub/invoice.js` 第 6-27 行 |
| 仅后端 Resource | `formatted_created_at`, `invoice_pdf_url`, `allow_edit`, `payment_module_enabled`, `formatted_invoice_date` | 计算属性、格式化字段、权限字段 | `app/Http/Resources/InvoiceResource.php` 第 51-59 行 |
| 仅前端 Stub | `selectedNote`, `selectedCurrency`, `customFields`, `title` (item) | 前端 UI 状态、临时字段 | `resources/scripts/admin/stub/invoice.js` 第 35-38 行 |

### 4.4 数据合并机制

```
新建表单时：
  newInvoice = { ...invoiceStub() }  ← 完全使用 stub

编辑表单时：
  1. newInvoice = { ...invoiceStub() }  ← 先用 stub 初始化
  2. Object.assign(newInvoice, apiData)   ← 再用 API 数据覆盖
  3. setInvoiceData() 做额外前端处理       ← 补充/转换字段
```

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 36-39 行（state 初始化用 stub）

```js
newInvoice: {
  ...invoiceStub(),
},
```

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 102-106 行（重置时用 stub）

```js
resetCurrentInvoice() {
  this.newInvoice = {
    ...invoiceStub(),
  }
},
```

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 155 行（编辑时 Object.assign 合并）

```js
Object.assign(this.newInvoice, invoice)
```

**重要**：`Object.assign` 是**浅合并**。如果 API 返回了 `customer` 对象，会完整替换 stub 中的 `customer: null`；如果 API 没有返回某个字段，stub 的默认值会保留。

---

## 五、字段"拉平"疑惑的澄清

### 5.1 字段不是被拉平的，而是分层嵌套的

后端 Resource 中的字段按层级组织，前端 Store 接收后也是同样的嵌套结构。并没有一个"拉平"的过程。

**举例：Invoice 的 customer 字段**

后端 InvoiceResource 定义：

**代码证据**：`app/Http/Resources/InvoiceResource.php` 第 63-65 行

```php
'customer' => $this->when($this->customer()->exists(), function () {
    return new CustomerResource($this->customer);
}),
```

CustomerResource 内部又有自己的嵌套字段：

**代码证据**：`app/Http/Resources/CustomerResource.php` 第 40-54 行

```php
'billing' => $this->when($this->billingAddress()->exists(), function () {
    return new AddressResource($this->billingAddress);
}),
'shipping' => $this->when($this->shippingAddress()->exists(), function () {
    return new AddressResource($this->shippingAddress);
}),
'currency' => $this->when($this->currency()->exists(), function () {
    return new CurrencyResource($this->currency);
}),
```

前端接收到的数据（嵌套结构）：
```js
{
  id: 1,
  invoice_number: 'INV-001',
  customer_id: 5,           // 外键字段（扁平）
  customer: {               // 嵌套对象（不是扁平的）
    id: 5,
    name: 'Acme Corp',
    email: 'acme@example.com',
    billing: {              // 进一步嵌套
      id: 10,
      address_street_1: '123 Main St',
      city: 'New York',
    },
    shipping: { ... },
    currency: { ... },
  }
}
```

### 5.2 容易混淆的点

1. **customer_id 与 customer 同时存在**
   - `customer_id` 是数据库外键字段（基础字段）
   - `customer` 是嵌套的关联资源对象（条件加载）
   - 两者在 Resource 中并列存在，各有用途

   **代码证据**：`app/Http/Resources/InvoiceResource.php` 第 40 行 + 第 63-65 行
   ```php
   'customer_id' => $this->customer_id,   // 外键（第 40 行）
   // ...
   'customer' => $this->when(...)          // 嵌套对象（第 63-65 行）
   ```

2. **列表接口与详情接口的嵌套字段差异 — 纠正**

   ❌ **之前的错误理解**：列表接口嵌套浅，详情接口嵌套深

   ✅ **实际情况**：列表和详情接口返回的嵌套字段种类**基本相同**（只要数据库中有关联数据就会返回），区别在于：
   - **性能不同**：列表如果没预加载会有 N+1 问题，详情只有一条数据影响小
   - **数据量不同**：列表返回多条，每条都带嵌套数据时 payload 更大
   - **深度相同**：嵌套深度都是递归的（如 items → taxes）

   列表页实际使用了嵌套的 customer 数据：

   **代码证据**：`resources/scripts/admin/views/invoices/Index.vue` 第 200 行
   ```vue
   <BaseText :text="row.data.customer.name" />
   ```

3. **前端 store 中的"扁平化"操作**
   有些 store action 会把嵌套数据提取到顶层，这是**前端业务需要**，不是后端序列化的行为。

   **代码证据**：`resources/scripts/admin/stores/invoice.js` 第 411-425 行（`selectCustomer`）

   ```js
   selectCustomer(id) {
     return new Promise((resolve, reject) => {
       http
         .get(`/api/v1/customers/${id}`)
         .then((response) => {
           this.newInvoice.customer = response.data.data      // 保存完整嵌套对象
           this.newInvoice.customer_id = response.data.data.id // 同时保存 id 到顶层
           resolve(response)
         })
     })
   }
   ```

---

## 六、前端读取 Optional 嵌套字段的影响

由于后端使用 `when + exists()` 模式，嵌套字段的出现取决于数据库中是否有关联数据，前端必须处理字段缺失的情况。

### 6.1 嵌套字段缺失的三种场景

| 场景 | 原因 | 前端表现 |
|------|------|---------|
| 关联数据不存在 | 数据库中没有关联记录（如新建发票还没加 items） | 字段在 JSON 中不存在（不是 null，是没有这个 key） |
| 关联数据存在但为空集合 | 有 items 关系但 items 表为空（罕见） | 字段存在，值为 `[]` 或 `null` |
| 外键为 null | 如 `customer_id` 为 null（未选择客户的发票） | `customer` 字段不存在 |

### 6.2 前端的防御式读取

前端代码中大量使用**可选链操作符（`?.`）**来安全访问嵌套字段：

**代码证据**（可选链使用实例）：

- `resources/scripts/admin/views/recurring-invoices/View.vue` 第 40 行：
  ```js
  recurringInvoiceStore.newRecurringInvoice?.customer?.name
  ```

- `resources/scripts/admin/views/payments/View.vue` 第 160 行：
  ```vue
  :text="payment?.customer?.name"
  ```

- `resources/scripts/admin/views/items/Create.vue` 第 182 行：
  ```js
  itemStore?.currentItem?.taxes?.map((tax) => { ... })
  ```

- `resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue` 第 453 行：
  ```js
  let itemTaxes = props.store[props.storeProp]?.items[props.index]?.taxes
  ```

### 6.3 Store 层的防御性处理

除了模板中的可选链，store 在处理数据时也做了防御性判断：

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 175-185 行（`setCustomerAddresses`）

```js
setCustomerAddresses(customer) {
  const customer_business = customer.customer_business

  // 用可选链安全访问嵌套字段
  if (customer_business?.billing_address)
    this.newInvoice.customer.billing_address =
      customer_business.billing_address

  if (customer_business?.shipping_address)
    this.newInvoice.customer.shipping_address =
      customer_business.shipping_address
},
```

**代码证据**：`resources/scripts/admin/stores/invoice.js` 第 157-162 行（处理 items.taxes 前先判断）

```js
if (this.newInvoice.tax_per_item === 'YES') {
  this.newInvoice.items.forEach((_i) => {
    if (_i.taxes && !_i.taxes.length)   // 先判断 taxes 是否存在
      _i.taxes.push({ ...taxStub, id: Guid.raw() })
  })
}
```

### 6.4 Stub 对可选字段的兜底作用

Stub 的一个重要作用是为可能不存在的嵌套字段提供**默认值兜底**：

| 字段 | Stub 默认值 | 后端无数据时 |
|------|------------|-------------|
| `customer` | `null` | Object.assign 后保持 `null` |
| `items` | 含 1 个空项目的数组 | 如果后端返回空数组或无此字段？ |
| `taxes` | `[]` | 保持空数组 |
| `billing` / `shipping` | 完整 addressStub 对象 | 需注意：customer 为 null 时访问会报错 |

> **注意一个隐患**：Stub 中 `customer: null`，但 `setCustomerAddresses` 等方法直接访问 `customer.customer_business`。如果后端返回的发票没有 customer（`customer_id` 为 null），直接访问会抛错。实际代码中是在选择客户后才调用这些方法，因此暂时安全。

---

## 七、完整数据流示例：发票详情页

### 7.1 后端流程

```
1. 请求到达 → GET /api/v1/invoices/123
   │
   ▼
2. 路由 → InvoicesController@show
   「代码位置」：routes/api.php → InvoicesController::show
   │
   ▼
3. 路由模型绑定 → 从数据库查询 Invoice Model
   「Laravel 隐式模型绑定」
   │
   ▼
4. 策略授权 → 检查用户是否有权限查看
   「代码位置」：app/Policies/InvoicePolicy.php
   │
   ▼
5. 返回 new InvoiceResource($invoice)
   「代码位置」：app/Http/Resources/InvoiceResource.php
   │
   ├─ 基础字段：id, invoice_number, invoice_date, status...
   │  「代码位置」：InvoiceResource.php 第 18-50 行
   ├─ 访问器字段：formatted_invoice_date, invoice_pdf_url, allow_edit...
   │  「代码位置」：InvoiceResource.php 第 51-59 行
   │                  + Invoice.php 第 126-162 行（访问器实现）
   └─ 条件嵌套（全部通过 exists() + 懒加载）：
      ├─ customer → CustomerResource（如果 customer_id 不为 null）
      │  「代码位置」：InvoiceResource.php 第 63-65 行
      ├─ items → InvoiceItemResource[]（如果有 items 记录）
      │  「代码位置」：InvoiceResource.php 第 60-62 行
      │  └─ items[].taxes → TaxResource[]
      │     「代码位置」：InvoiceItemResource.php 第 38-40 行
      ├─ taxes → TaxResource[]（如果有 taxes 记录）
      │  「代码位置」：InvoiceResource.php 第 69-71 行
      ├─ fields, creator, company, currency 等
      │
      ▼
6. JSON 响应 → { "data": { ... } }
```

### 7.2 前端流程

```
1. 进入 /invoices/123 路由
   「代码位置」：resources/scripts/admin/admin-router.js
   │
   ▼
2. 组件 onMounted → 调用 invoiceStore.fetchInvoiceInitialSettings(true)
   「代码位置」：resources/scripts/admin/views/invoices/View.vue
   │
   ▼
3. fetchInvoiceInitialSettings(true) → 调用 fetchInvoice(route.params.id)
   「代码位置」：resources/scripts/admin/stores/invoice.js 第 485-576 行
   │
   ▼
4. fetchInvoice(id)
   「代码位置」：resources/scripts/admin/stores/invoice.js 第 138-152 行
   │
   ├─ http.get(`/api/v1/invoices/${id}`)
   │  「代码位置」：resources/scripts/http/index.js
   │
   ▼
5. 收到响应 response.data.data
   │
   ▼
6. setInvoiceData(response.data.data)
   「代码位置」：resources/scripts/admin/stores/invoice.js 第 154-173 行
   │
   ├─ Object.assign(this.newInvoice, invoice)
   │  → 把后端数据合并到 stub 初始化的对象上
   │
   ├─ 补充前端 UI 需要的空 tax 项
   │  「代码位置」：invoice.js 第 157-162 行
   └─ 折扣单位转换等前端特有逻辑
      「代码位置」：invoice.js 第 164-172 行
   │
   ▼
7. setCustomerAddresses(customer)
   → 从 customer 对象中提取地址相关信息
   「代码位置」：invoice.js 第 175-185 行
   │
   ▼
8. 视图组件使用 newInvoice 渲染表单
   → v-model 双向绑定各字段
   → 嵌套字段用 ?. 安全访问
```

---

## 八、Customer 模块对比验证

以 Customer 为例验证上述模式是否一致，确认这是项目的统一模式。

### 8.1 后端 CustomerResource

**代码证据**：`app/Http/Resources/CustomerResource.php` 第 8-57 行

```php
class CustomerResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            // 基础字段（第 18-38 行）
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'phone' => $this->phone,
            'currency_id' => $this->currency_id,
            // ...

            // 访问器字段（第 33, 36-37 行）
            'formatted_created_at' => $this->formattedCreatedAt,
            'due_amount' => $this->due_amount,
            'password_added' => $this->password ? true : false,

            // 嵌套关系（第 40-54 行）
            'billing' => $this->when($this->billingAddress()->exists(), function () {
                return new AddressResource($this->billingAddress);
            }),
            'shipping' => $this->when($this->shippingAddress()->exists(), function () {
                return new AddressResource($this->shippingAddress);
            }),
            'currency' => $this->when($this->currency()->exists(), function () {
                return new CurrencyResource($this->currency);
            }),
        ];
    }
}
```

对应的模型关系：

**代码证据**：`app/Models/Customer.php` 第 87-120 行

```php
public function addresses(): HasMany
{
    return $this->hasMany(Address::class);
}

public function billingAddress(): HasOne
{
    return $this->hasOne(Address::class)->where('type', Address::BILLING_TYPE);
}

public function shippingAddress(): HasOne
{
    return $this->hasOne(Address::class)->where('type', Address::SHIPPING_TYPE);
}

public function currency(): BelongsTo
{
    return $this->belongsTo(Currency::class);
}
```

### 8.2 前端 customer store

**代码证据**：`resources/scripts/admin/stores/customer.js` 第 112-124 行

```js
fetchCustomer(id) {
  return new Promise((resolve, reject) => {
    http
      .get(`/api/v1/customers/${id}`)
      .then((response) => {
        Object.assign(this.currentCustomer, response.data.data)
        this.setAddressStub(response.data.data)
        resolve(response)
      })
  })
}
```

### 8.3 前端 customer stub

**代码证据**：`resources/scripts/admin/stub/customer.js` 第 1-19 行

```js
import addressStub from '@/scripts/admin/stub/address.js'

export default function () {
  return {
    name: '',
    contact_name: '',
    email: '',
    phone: null,
    password: '',
    confirm_password: '',
    currency_id: null,
    website: null,
    billing: { ...addressStub },    // 与后端 billing 嵌套对应
    shipping: { ...addressStub },   // 与后端 shipping 嵌套对应
    customFields: [],               // 前端专用
    fields: [],
    enable_portal: false,
  }
}
```

**验证结论**：Customer 模块与 Invoice 模块遵循完全相同的模式：
- 后端 Resource 使用 `when + exists()` 定义嵌套字段
- 前端 stub 定义对应的数据模板结构
- Store 通过 `Object.assign` 合并 API 数据
- 前端有自己的额外字段和逻辑

---

## 九、关键注意事项

### 9.1 条件加载的真实行为（纠正）

后端 Resource 使用 `$this->when($relation->exists(), ...)` 检查关联是否存在。需要纠正之前的理解：

❌ **旧理解**：列表接口可能没有 `items` 字段（因为没预加载）

✅ **实际情况**：
- 只要数据库里有关联数据，就会返回嵌套字段
- 列表接口也不例外，只是存在 N+1 性能问题
- 如果数据库中没有关联数据，字段才不会出现
- 前端访问时要做空值判断：`invoice.items?.forEach(...)`

**代码证据**：`app/Http/Resources/InvoiceResource.php` 第 60-62 行

```php
'items' => $this->when($this->items()->exists(), function () {
    return InvoiceItemResource::collection($this->items);
}),
```

**性能影响**：列表接口（10 条数据）会触发：
- 10 次 `items` 的 `exists()` 查询
- 10 次 `customer` 的 `exists()` 查询（但 customer 已预加载，exists 还是额外查）
- 10 次 `taxes` 的 `exists()` 查询
- 10 次 `creator` 的 `exists()` 查询
- 10 次 `currency` 的 `exists()` 查询
- 10 次 `fields` 的 `exists()` 查询
- 10 次 `company` 的 `exists()` 查询
- ...以及对应的数据懒加载查询

> 列表接口总计：仅 1 个 `with('customer')` 预加载，其他 6+ 个关联每次都走 exists + 可能的懒加载，存在严重的 N+1 问题。

### 9.2 前端额外字段

Stub 中定义但后端没有的字段，在 `Object.assign` 后会保留。常见的前端专用字段：

| 字段 | 所在 stub | 用途 |
|------|----------|------|
| `selectedNote` | invoice stub | UI 选中的备注 |
| `selectedCurrency` | invoice stub | UI 选中的货币显示 |
| `customFields` | 多个 stub | 自定义字段的前端表示 |
| `title` | invoice-item stub | 行项目显示标题（内部用） |
| `totalTax` | invoice-item stub | 前端计算的总税额 |
| Guid 生成的 `id` | 多个子 stub | 前端临时 ID，新增项用 |

**代码证据**：`resources/scripts/admin/stub/invoice.js` 第 35-38 行

```js
customFields: [],
fields: [],
selectedNote: null,
selectedCurrency: '',
```

### 9.3 数据提交方向

**从前端到后端**时，提交的数据结构不需要与 Resource 返回的完全一致。后端通过 Form Request 验证，只接受它需要的字段。

例：创建发票时，前端提交的数据包含 `items` 数组，后端 `Invoice::createInvoice()` 会处理嵌套的 items 和 taxes。

**代码证据**：`app/Models/Invoice.php` 第 326-373 行（`createInvoice` 静态方法）

```php
public static function createInvoice($request)
{
    $data = $request->getInvoicePayload();
    $invoice = Invoice::create($data);
    // ...
    self::createItems($invoice, $request->items);   // 处理嵌套 items
    // ...
    if ($request->has('taxes') && (! empty($request->taxes))) {
        self::createTaxes($invoice, $request->taxes); // 处理嵌套 taxes
    }
    // ...
    return $invoice;
}
```

### 9.4 两套资源类

项目中有两套资源类，分别对应管理后台和客户门户：

| 路径 | 使用者 | 说明 |
|------|--------|------|
| `app/Http/Resources/*.php` | 管理后台（Admin） | 字段最全，用于内部管理 |
| `app/Http/Resources/Customer/*.php` | 客户门户（Customer Portal） | 字段可能更少，更安全 |

两套资源类都使用相同的 `when + exists()` 模式（grep 验证两套代码模式一致）。

---

## 十、文件索引

### 10.1 后端

| 文件路径 | 作用 | 关键行 |
|---------|------|--------|
| `app/Http/Resources/InvoiceResource.php` | 发票资源序列化 | 第 15-81 行（toArray 方法）、第 60-80 行（when + exists 嵌套） |
| `app/Http/Resources/CustomerResource.php` | 客户资源序列化 | 第 15-55 行（toArray 方法）、第 40-54 行（嵌套关系） |
| `app/Http/Resources/InvoiceItemResource.php` | 发票行项目资源 | 第 15-44 行 |
| `app/Http/Resources/TaxResource.php` | 税费资源 | 第 15-42 行 |
| `app/Http/Resources/AddressResource.php` | 地址资源 | 第 15-38 行 |
| `app/Models/Invoice.php` | 发票模型（含访问器） | 第 56-62 行（appends）、第 126-162 行（访问器）、第 86-124 行（关系） |
| `app/Models/Customer.php` | 客户模型 | 第 87-120 行（地址关系）、第 53-58 行（访问器） |
| `app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php` | 发票控制器 | 第 21-37 行（index + with）、第 65-70 行（show 无 with） |
| `app/Http/Resources/InvoiceCollection.php` | 发票资源集合类 | 第 15-17 行 |

### 10.2 前端

| 文件路径 | 作用 | 关键行 |
|---------|------|--------|
| `resources/scripts/admin/stores/invoice.js` | 发票 Pinia Store | 第 26-99 行（state/getters）、第 122-173 行（核心 actions） |
| `resources/scripts/admin/stores/customer.js` | 客户 Pinia Store | 第 112-124 行（fetchCustomer） |
| `resources/scripts/admin/stub/invoice.js` | 发票数据桩 | 第 5-39 行（默认结构） |
| `resources/scripts/admin/stub/customer.js` | 客户数据桩 | 第 3-18 行（默认结构） |
| `resources/scripts/admin/stub/invoice-item.js` | 发票行项目数据桩 | 第 1-17 行 |
| `resources/scripts/admin/stub/address.js` | 地址数据桩 | 第 1-10 行 |
| `resources/scripts/http/index.js` | Axios HTTP 封装 | 第 6-28 行（实例+拦截器） |
| `resources/scripts/admin/views/invoices/Index.vue` | 发票列表页 | 第 200 行（使用嵌套 customer.name） |
| `resources/scripts/admin/views/invoices/View.vue` | 发票详情页 | 第 192-197 行（loadInvoice） |
