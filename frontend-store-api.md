# 前端 Store 与后端资源类协作指南

本文档梳理 InvoiceShelf 项目中前端状态管理（Pinia Store）、后端 API 资源类（Laravel Eloquent API Resource）以及字段序列化之间的协作关系。

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

以 [InvoiceResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/InvoiceResource.php) 为例：

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
            // ...

            // 2. 访问器字段（Model 中 getXxxAttribute 方法）
            'formatted_created_at' => $this->formattedCreatedAt,
            'invoice_pdf_url' => $this->invoicePdfUrl,
            'formatted_invoice_date' => $this->formattedInvoiceDate,
            'allow_edit' => $this->allow_edit,
            'payment_module_enabled' => $this->payment_module_enabled,
            // ...

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
            // ...
        ];
    }
}
```

### 2.3 字段来源的三种类型

**字段不是"拉平"的，而是分层嵌套的。** 资源类中的字段按来源分为三类：

#### 类型 A：数据库直接字段
直接对应数据库表中的列，通过 `$this->字段名` 访问。
- 例：`id`, `invoice_number`, `invoice_date`, `customer_id`, `sub_total`, `total`

#### 类型 B：Eloquent 访问器（Accessors）
在 Model 中通过 `getXxxAttribute()` 方法定义的计算属性，在资源类中以驼峰/蛇形命名访问。
- 例：`formattedCreatedAt`, `invoicePdfUrl`, `allow_edit`, `payment_module_enabled`

以 [Invoice.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Models/Invoice.php) 中的访问器为例：

```php
// Model 中定义
protected $appends = [
    'formattedCreatedAt',
    'formattedInvoiceDate',
    'invoicePdfUrl',
];

public function getInvoicePdfUrlAttribute()
{
    return url('/invoices/pdf/'.$this->unique_hash);
}

public function getAllowEditAttribute()
{
    // 复杂业务逻辑...
    return $allowed;
}

// Resource 中直接使用
'invoice_pdf_url' => $this->invoicePdfUrl,
'allow_edit' => $this->allow_edit,
```

#### 类型 C：嵌套关系资源
通过 `$this->when()` 条件加载，返回子 Resource 或 Resource 集合。
- 一对一关系用 `new XxxResource($this->relation)`
- 一对多关系用 `XxxResource::collection($this->relation)`

### 2.4 嵌套资源的递归结构

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

相关资源文件：
- [CustomerResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/CustomerResource.php)
- [InvoiceItemResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/InvoiceItemResource.php)
- [TaxResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/TaxResource.php)
- [AddressResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/AddressResource.php)

### 2.5 资源类与集合类

- **单资源**：`new InvoiceResource($invoice)` — 返回单个对象，包装在 `data` 键下
- **资源集合**：`InvoiceResource::collection($invoices)` — 返回对象数组，包装在 `data` 键下，附带 `meta` 和 `links` 分页信息
- **自定义集合类**：`InvoiceCollection` — 通常用于添加额外的 meta 数据

参考：[InvoiceCollection.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/InvoiceCollection.php)

### 2.6 控制器中的使用

控制器从数据库查询 Model（通常带着 `with()` 预加载关联），然后用 Resource 转换后返回：

[InvoicesController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php)

```php
// 列表接口
public function index(Request $request)
{
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

// 详情接口
public function show(Request $request, Invoice $invoice)
{
    return new InvoiceResource($invoice);
}
```

---

## 三、前端 Store 的数据消费

### 3.1 HTTP 请求层

前端使用 Axios 封装的 HTTP 客户端发送请求：

[http/index.js](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/http/index.js)

```js
import axios from 'axios'

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

以 [invoice.js](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stores/invoice.js) 为例，每个 Store 包含三部分：

```js
import { defineStore } from 'pinia'
import http from '@/scripts/http'
import invoiceStub from '../stub/invoice'

export const useInvoiceStore = () => {
  return defineStore('invoice', {
    // state: 状态数据
    state: () => ({
      invoices: [],          // 列表数据
      invoiceTotalCount: 0,  // 总数
      newInvoice: {          // 当前编辑/新建的表单数据
        ...invoiceStub(),
      },
    }),

    // getters: 计算属性
    getters: {
      getInvoice: (state) => (id) => 
        state.invoices.find(invoice => invoice.id === id),
      getSubTotal() { /* ... */ },
    },

    // actions: 方法（含 API 调用）
    actions: {
      fetchInvoices(params) { /* ... */ },
      fetchInvoice(id) { /* ... */ },
      addInvoice(data) { /* ... */ },
    },
  })()
}
```

### 3.3 从 API 到 Store 的数据流向

#### 列表数据获取

```js
// Store action
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

后端响应的 JSON 结构：
```json
{
  "data": [
    { "id": 1, "invoice_number": "INV-001", "status": "SENT", ... },
    { "id": 2, "invoice_number": "INV-002", "status": "DRAFT", ... }
  ],
  "meta": {
    "invoice_total_count": 100
  },
  "links": { ... }
}
```

#### 详情数据获取

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
      .catch(...)
  })
}

setInvoiceData(invoice) {
  // 关键：用 Object.assign 把后端数据合并到现有 newInvoice 对象
  Object.assign(this.newInvoice, invoice)

  // 额外的前端逻辑：补充空 tax 项、转换折扣单位等
  if (this.newInvoice.tax_per_item === 'YES') {
    this.newInvoice.items.forEach((_i) => {
      if (_i.taxes && !_i.taxes.length)
        _i.taxes.push({ ...taxStub, id: Guid.raw() })
    })
  }
  // ...
}
```

**关键点：** `Object.assign(this.newInvoice, invoice)` 将后端返回的完整 JSON 对象（含嵌套的 customer、items、taxes 等）直接合并到前端状态对象中。**嵌套结构被完整保留，并没有被拉平。**

---

## 四、Stub（数据桩）的角色

### 4.1 什么是 Stub

Stub 是前端定义的**默认数据结构模板**，用于：
- 新建表单时的初始值
- 确保数据结构的完整性（即使后端没返回某些字段）
- 添加前端专用的字段（后端不需要）

### 4.2 Stub 示例

[invoice.js stub](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stub/invoice.js)

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
    // ...

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

### 4.3 Stub 与 Resource 的字段对比

| 来源 | 字段举例 | 说明 |
|------|---------|------|
| 两者共有 | `id`, `invoice_number`, `customer_id`, `sub_total`, `items`, `taxes` | 数据库字段，前后端对齐 |
| 仅后端 Resource | `formatted_created_at`, `invoice_pdf_url`, `allow_edit`, `payment_module_enabled`, `formatted_invoice_date` | 计算属性、格式化字段、权限字段 |
| 仅前端 Stub | `selectedNote`, `selectedCurrency`, `customFields` | 前端 UI 状态、临时字段 |

### 4.4 数据合并机制

```
新建表单时：
  newInvoice = { ...invoiceStub() }  ← 完全使用 stub

编辑表单时：
  1. newInvoice = { ...invoiceStub() }  ← 先用 stub 初始化
  2. Object.assign(newInvoice, apiData)   ← 再用 API 数据覆盖
  3. setInvoiceData() 做额外前端处理       ← 补充/转换字段
```

**重要：** `Object.assign` 是**浅合并**。如果 API 返回了 `customer` 对象，会完整替换 stub 中的 `customer: null`；如果 API 没有返回某个字段，stub 的默认值会保留。

---

## 五、字段"拉平"疑惑的澄清

### 5.1 字段不是被拉平的，而是分层嵌套的

后端 Resource 中的字段按层级组织，前端 Store 接收后也是同样的嵌套结构。并没有一个"拉平"的过程。

**举例：Invoice 的 customer 字段**

后端 InvoiceResource 定义：
```php
'customer' => $this->when($this->customer()->exists(), function () {
    return new CustomerResource($this->customer);
}),
```

前端接收到的数据：
```js
{
  id: 1,
  invoice_number: 'INV-001',
  customer: {           // 嵌套对象，不是 customer_id, customer_name 这种扁平字段
    id: 5,
    name: 'Acme Corp',
    email: 'acme@example.com',
    billing: { ... },   // 进一步嵌套
    shipping: { ... },
  }
}
```

### 5.2 容易混淆的点

1. **customer_id 与 customer 同时存在**
   - `customer_id` 是数据库外键字段（基础字段）
   - `customer` 是嵌套的关联资源对象（条件加载）
   - 两者在 Resource 中并列存在，各有用途

2. **列表接口与详情接口的差异**
   - 列表接口通常只加载少量关联（如 `with('customer')`），嵌套较浅
   - 详情接口会加载更多关联（items, taxes, fields 等），嵌套较深
   - 具体加载哪些关联，由控制器的 `with()` 和 Resource 的 `when()` 共同决定

3. **前端 store 中的"扁平化"操作**
   有些 store action 会把嵌套数据提取到顶层，这是**前端业务需要**，不是后端序列化的行为。

   例：[invoice.js](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stores/invoice.js) 中的 `selectCustomer`：
   ```js
   selectCustomer(id) {
     return http.get(`/api/v1/customers/${id}`)
       .then((response) => {
         this.newInvoice.customer = response.data.data    // 保存完整对象
         this.newInvoice.customer_id = response.data.data.id  // 同时保存 id 到顶层
       })
   }
   ```

---

## 六、完整数据流示例：发票详情页

### 6.1 后端流程

```
1. 请求到达 → GET /api/v1/invoices/123
   │
   ▼
2. 路由 → InvoicesController@show
   │
   ▼
3. 路由模型绑定 → 从数据库查询 Invoice Model
   │
   ▼
4. 策略授权 → 检查用户是否有权限查看
   │
   ▼
5. 返回 new InvoiceResource($invoice)
   │
   ├─ 基础字段：id, invoice_number, invoice_date, status...
   ├─ 访问器字段：formatted_invoice_date, invoice_pdf_url, allow_edit...
   └─ 条件嵌套：
      ├─ customer → CustomerResource（如果关联存在）
      ├─ items → InvoiceItemResource[]（如果关联存在）
      │   └─ items[].taxes → TaxResource[]
      └─ taxes → TaxResource[]
   │
   ▼
6. JSON 响应 → { "data": { ... } }
```

### 6.2 前端流程

```
1. 进入 /invoices/123 路由
   │
   ▼
2. 组件 mounted / onMounted → 调用 invoiceStore.fetchInvoiceInitialSettings(true)
   │
   ▼
3. fetchInvoice(route.params.id)
   │
   ├─ http.get(`/api/v1/invoices/${id}`)
   │  │
   │  ▼
   │  Axios 请求 → 带上 Authorization token 和 company header
   │
   ▼
4. 收到响应 response.data.data
   │
   ▼
5. setInvoiceData(response.data.data)
   │
   ├─ Object.assign(this.newInvoice, invoice)
   │  → 把后端数据合并到 stub 初始化的对象上
   │
   ├─ 补充前端 UI 需要的空 tax 项
   └─ 折扣单位转换等前端特有逻辑
   │
   ▼
6. setCustomerAddresses(customer)
   → 从 customer.customer_business 提取 billing/shipping address
   │
   ▼
7. 视图组件使用 newInvoice 渲染表单
   → v-model 双向绑定各字段
```

---

## 七、Customer 模块对比验证

以 Customer 为例验证上述模式是否一致：

### 后端 CustomerResource

[CustomerResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/CustomerResource.php)

```php
return [
    // 基础字段
    'id' => $this->id,
    'name' => $this->name,
    'email' => $this->email,
    // ...
    
    // 访问器字段
    'formatted_created_at' => $this->formattedCreatedAt,
    'password_added' => $this->password ? true : false,
    
    // 嵌套关系
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
```

### 前端 customer store

[customer.js store](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stores/customer.js)

```js
fetchCustomer(id) {
  return http.get(`/api/v1/customers/${id}`)
    .then((response) => {
      Object.assign(this.currentCustomer, response.data.data)
      this.setAddressStub(response.data.data)
    })
}
```

### 前端 customer stub

[customer.js stub](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stub/customer.js)

```js
export default function () {
  return {
    name: '',
    email: '',
    currency_id: null,
    billing: { ...addressStub },    // 与后端 billing 嵌套对应
    shipping: { ...addressStub },   // 与后端 shipping 嵌套对应
    customFields: [],               // 前端专用
    enable_portal: false,
    // ...
  }
}
```

**验证结论：** Customer 模块与 Invoice 模块遵循完全相同的模式。

---

## 八、关键注意事项

### 8.1 条件加载的陷阱

后端 Resource 使用 `$this->when($relation->exists(), ...)` 只在关联存在时返回嵌套数据。这意味着：
- 列表接口可能没有 `items` 字段（因为没预加载）
- 详情接口有 `items` 字段（因为预加载了）
- 前端访问时要做空值判断：`invoice.items?.forEach(...)`

### 8.2 前端额外字段

Stub 中定义但后端没有的字段，在 `Object.assign` 后会保留。常见的前端专用字段：
- `selectedNote`, `selectedCurrency` — UI 选中状态
- `customFields` — 自定义字段的前端表示
- 临时的 `id`（用 Guid 生成，前端用，后端会忽略）

### 8.3 数据提交方向

**从前端到后端**时，提交的数据结构不需要与 Resource 返回的完全一致。后端通过 Form Request 验证，只接受它需要的字段。

例：创建发票时，前端提交的数据包含 `items` 数组，后端 `Invoice::createInvoice()` 会处理嵌套的 items 和 taxes。

### 8.4 两套资源类

项目中有两套资源类：
- `app/Http/Resources/*.php` — 管理后台使用（Admin）
- `app/Http/Resources/Customer/*.php` — 客户门户使用（Customer Portal）

客户门户的资源类字段可能更少（安全考虑）。

---

## 九、文件索引

### 后端
| 文件 | 作用 |
|------|------|
| [InvoiceResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/InvoiceResource.php) | 发票资源序列化 |
| [CustomerResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/CustomerResource.php) | 客户资源序列化 |
| [InvoiceItemResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/InvoiceItemResource.php) | 发票行项目资源 |
| [TaxResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/TaxResource.php) | 税费资源 |
| [AddressResource.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Resources/AddressResource.php) | 地址资源 |
| [Invoice.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Models/Invoice.php) | 发票模型（含访问器） |
| [Customer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Models/Customer.php) | 客户模型 |
| [InvoicesController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/app/Http/Controllers/V1/Admin/Invoice/InvoicesController.php) | 发票控制器 |

### 前端
| 文件 | 作用 |
|------|------|
| [invoice.js store](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stores/invoice.js) | 发票 Pinia Store |
| [customer.js store](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stores/customer.js) | 客户 Pinia Store |
| [invoice.js stub](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stub/invoice.js) | 发票数据桩 |
| [customer.js stub](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stub/customer.js) | 客户数据桩 |
| [invoice-item.js stub](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stub/invoice-item.js) | 发票行项目数据桩 |
| [address.js stub](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/admin/stub/address.js) | 地址数据桩 |
| [http/index.js](file:///d:/fz/0508-2/solo-dogfeeding/code/125-InvoiceShelf/resources/scripts/http/index.js) | Axios HTTP 封装 |
