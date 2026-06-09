# 发票行金额、税率叠加与最终合计算法全链梳理

## 概述

InvoiceShelf 的发票金额计算体系分为三层：**税率配置层** → **行级计算层** → **总额结算层**。所有金额均以**分**为最小单位存储（`integer` 类型），前端展示时除以 100。

---

## 一、税率配置层

### 1.1 税率类型模型

税率配置的核心模型为 `app/Models/TaxType.php`，支持两种计算方式：

| 计算方式 | 字段 | 说明 |
|---------|------|------|
| 百分比 | `calculation_type = 'percentage'` | 按 `percent` 字段的百分比计算 |
| 固定金额 | `calculation_type = 'fixed'` | 按 `fixed_amount` 字段的固定金额计算 |

### 1.2 关键税率属性

| 属性 | 类型 | 说明 |
|-----|------|------|
| `compound_tax` | boolean | **复合税**：仅在整体税模式下生效，在（折扣后小计 + 其他简单税）基础上计征 |
| `collective_tax` | boolean | 集体税标识（预留属性） |
| `percent` | float | 百分比税率值 |
| `fixed_amount` | integer | 固定税额（单位：分） |

> **重要**：`compound_tax` 仅在**整体税模式**（`tax_per_item = 'NO'`）下有意义。行级税模式下，所有税种均为简单税，不存在税上税的叠加。

税率配置的表单验证在 `app/Http/Requests/TaxTypeRequest.php` 中定义。

### 1.3 税实例模型

具体应用到发票或行项目上的税为 `app/Models/Tax.php` 模型实例，包含：

- `tax_type_id`：关联的税率类型
- `amount`：实际计算出的税额（分）
- `percent` / `fixed_amount`：税率快照
- `calculation_type`：计算方式快照
- `compound_tax`：是否复合税快照

### 1.4 TaxType 选择后的字段传递链

当用户在界面选择一个 TaxType 后，系统会将 TaxType 的部分属性快照到 Tax 实例中。字段携带情况如下：

| 字段 | TaxType → Tax（整单税） | Tax（行级税） | 说明 |
|-----|--------------------------|--------------|------|
| `name` | ✅ 携带 | ✅ 携带 | 税种名称 |
| `percent` | ✅ 携带 | ✅ 携带 | 税率百分比 |
| `calculation_type` | ✅ 携带 | ✅ 携带 | 计算方式 |
| `fixed_amount` | ✅ 携带 | ✅ 携带 | 固定税额 |
| `tax_type_id` | ✅ 携带 | ✅ 携带 | 关联 ID |
| `compound_tax` | ❌ **不携带** | ❌ **不携带** | 复合税标记 |

### 1.5 TaxStub 来源与导入 Bug

#### 1.5.1 行级税使用正确的 TaxStub

行级税的 stub 来源正确，来自 `resources/scripts/admin/stub/tax.js`：

```javascript
// CreateItemRow.vue 中的导入
import TaxStub from '@/scripts/admin/stub/tax'
```

**Tax stub 默认值**：
```javascript
export default {
  name: '',
  tax_type_id: 0,
  type: 'GENERAL',
  amount: null,
  percent: null,
  compound_tax: false,  // 明确为 false
}
```

#### 1.5.2 整单税的 TaxStub 导入错误（Bug）

> **严重 Bug**：`CreateTotal.vue` 中导入的 `TaxStub` 实际上是 `abilities.js`（权限常量对象），而不是 `tax.js`！

**错误导入代码**（`resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue`）：
```javascript
import TaxStub from '@/scripts/admin/stub/abilities'  // ⚠️ 错误！导入了权限常量
```

`abilities.js` 的内容是权限字符串常量的对象，例如：
```javascript
export default {
  DASHBOARD: 'dashboard',
  CREATE_CUSTOMER: 'create-customer',
  CREATE_TAX_TYPE: 'create-tax-type',
  // ... 共几十个权限字符串
}
```

#### 1.5.3 导入错误的实际影响

在 `onSelectTax()` 函数中：

```javascript
let data = {
  ...TaxStub,           // 展开的是 abilities 对象
  id: Guid.raw(),
  name: selectedTax.name,
  percent: selectedTax.percent,
  tax_type_id: selectedTax.id,
  amount,
  calculation_type: selectedTax.calculation_type,
  fixed_amount: selectedTax.fixed_amount
  // ⚠️ 没有 compound_tax！
}
```

**影响分析**：

| 影响点 | 具体说明 | 严重程度 |
|-------|---------|---------|
| `compound_tax` 缺失 | 新增整单税时，`compound_tax` 字段完全不存在（不是 `false`，是 `undefined`），导致复合税功能完全失效 | ⚠️ 高 |
| 多余属性混入 | tax 对象上会多出几十个权限常量属性（如 `DASHBOARD`、`CREATE_CUSTOMER` 等），但不会影响功能 | ⚠️ 低 |
| 类型安全破坏 | 本应是 Tax 类型的对象混入了大量无关属性，污染数据结构 | ⚠️ 中 |

#### 1.5.4 compound_tax 缺失对整单税的具体影响

`CreateTotalTaxes.vue` 中判断复合税的逻辑：
```javascript
if (props.tax.compound_tax && props.store.getSubtotalWithDiscount) {
  // 复合税计算
}
```

由于新增的 tax 对象上 `compound_tax` 是 `undefined`（甚至不是 `false`），`undefined && ...` 结果为 falsy，复合税分支永远不会执行。

**后果**：
- 所有新增的整单税都被当作简单税处理
- 即使在 TaxType 中配置了 `compound_tax = true`，添加到整单后也不会按复合税计算
- 只有从后端加载的已有数据（数据库中 `compound_tax = 1`）才可能触发复合税逻辑

### 1.6 数据库与模型层面

**数据库表**：`taxes` 表有 `compound_tax` 字段（见 `database/migrations/2019_09_21_052548_create_taxes_table.php），类型为 `tinyInteger`，默认值为 `0`。

**模型层**：`app/Models/Tax.php` 的 `$casts` 中**未定义** `compound_tax` 的类型转换，读取时默认为 `integer` 类型的 `0` 或 `1`。

**保存行为**：由于 `Tax` 模型使用 `$guarded = ['id']`（非 `$fillable`），前端传来的所有字段都会被保存，包括 `compound_tax`。但由于前端根本没传这个字段，数据库中该字段永远是默认值 `0`。

---

## 二、行级计算层

行级计算由 `resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue` 组件和 `resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue` 组件共同完成。

### 2.1 行金额计算公式

```
行小计(subtotal) = 单价(price) × 数量(quantity)
行折扣值(discount_val) = 行小计 × 折扣率(%)  或  固定折扣金额
行总额(total) = 行小计 - 行折扣值
```

核心代码位于 `CreateItemRow.vue`：

```javascript
const subtotal = computed(() => Math.round(props.itemData.price * props.itemData.quantity))
const total = computed(() => subtotal.value - props.itemData.discount_val)
```

### 2.2 行级税计算

当 `tax_per_item === 'YES'` 时，每个行项目可独立配置多个税种。行税计算在 `CreateItemRowTax.vue` 中实现。

> **关键特性**：行级多税**分别基于行折扣后金额独立计算，然后求和**。行级税模式下没有复合税概念，所有税种均以行折扣后金额为基数，不存在税上税叠加。

#### 2.2.1 普通百分比税（税外）

```
单税税额 = 行折扣后总额 × 税率%
行总税 = 税1 + 税2 + ... + 税N  （每个税均以行折扣后总额为基数）
```

代码位于 `CreateItemRowTax.vue` 的 `taxAmount` computed 中：
```javascript
return (props.discountedTotal * localTax.percent) / 100
```

#### 2.2.2 税内包含（Tax Included）

当 `tax_included = true` 时，价格中已包含税，需反算税额：

```
不含税价格 = 含税总价 / (1 + 税率%)
税额 = 含税总价 - 不含税价格
```

代码位于 `CreateItemRowTax.vue`：
```javascript
return Math.round(props.discountedTotal - (props.discountedTotal / (1 + (localTax.percent / 100))))
```

#### 2.2.3 固定金额税

直接使用 `fixed_amount`，不随行金额变化。

#### 2.2.4 行税汇总

```
行总税 = Σ 该行所有税种的税额
```

代码位于 `CreateItemRow.vue`：
```javascript
const totalSimpleTax = computed(() => {
  return Math.round(
    sumBy(props.itemData.taxes, function (tax) {
      if (tax.amount) { return tax.amount }
      return 0
    })
  )
})
```

> **注意**：行级税的变量名为 `totalSimpleTax`，侧面印证了行级税不存在复合税，所有行税都是"简单税"。

### 2.3 行级折扣与整体折扣的分摊

当 `discount_per_item === 'NO'`（整体折扣）且 `tax_per_item === 'YES'`（行级税）时，计算行税需先按比例分摊整体折扣：

```
行占比 = 行总额 / 所有行总额之和
行分摊折扣 = 总折扣值 × 行占比
行折扣后金额 = 行总额 - 行分摊折扣
```

代码位于 `CreateItemRowTax.vue` 的 `getTaxAmount()` 函数中。

### 2.4 行级税计算分支执行顺序

`CreateItemRowTax.vue` 中 `taxAmount` computed 的判断顺序：

| 优先级 | 条件 | 计算公式 |
|-------|------|---------|
| 1 | `calculation_type === 'fixed'` | 直接返回 `fixed_amount` |
| 2 | `tax_per_item === 'YES'` 且 `discount_per_item === 'NO'` | 调用 `getTaxAmount()`（分摊整体折扣后计算） |
| 3 | `tax_included === true` | 税内包含公式：反算税额 |
| 4 | （默认） | 普通百分比税：`discountedTotal × percent / 100` |

> **注意**：行级税计算中**完全没有 `compound_tax` 判断**，复合税属性在行级税模式下不生效。

---

## 三、总额结算层

总额计算在 `resources/scripts/admin/stores/invoice.js` Store 的 getters 中集中实现，并由 `resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue` 组件展示。

### 3.1 核心 Getter 一览

| Getter 名称 | 说明 |
|------------|------|
| `getSubTotal` | 所有行项目的 `total` 之和（行折扣后） |
| `getSubtotalWithDiscount` | 小计 - 整体折扣值 |
| `getTotalSimpleTax` | 所有简单税（非复合税）之和（仅整体税模式） |
| `getTotalCompoundTax` | 所有复合税之和（仅整体税模式） |
| `getTotalTax` | 总税额 = 简单税 + 复合税（整体税模式）或 行税之和（行级税模式） |
| `getTotal` | 最终发票总额 |
| `getNetTotal` | 净总额 = 折扣后小计 - 总税（税内包含模式下展示） |

### 3.2 小计计算

```
小计(sub_total) = Σ 每个行项目的 total（行折扣后金额）
```

代码：`invoice.js` 的 `getSubTotal` getter

### 3.3 整体折扣计算

整体折扣支持两种模式：

- **百分比折扣**：`discount_val = sub_total × discount%`
- **固定金额折扣**：`discount_val = discount × 100`（输入为元，转为分）

代码位于 `CreateTotal.vue` 的 `setDiscount()` 函数中。

### 3.4 整体税计算（tax_per_item = 'NO'）

当 `tax_per_item === 'NO'` 时，税种应用于整单，支持简单税和复合税两种类型。

#### 3.4.1 简单税（Simple Tax）

```
简单税税额 = 折扣后小计 × 税率%
总简单税 = Σ 所有非复合税的税额
```

代码：`invoice.js` 的 `getTotalSimpleTax` getter：
```javascript
getTotalSimpleTax() {
  return _.sumBy(this.newInvoice.taxes, function (tax) {
    if (!tax.compound_tax) {
      return tax.amount
    }
    return 0
  })
}
```

#### 3.4.2 复合税（Compound Tax）

复合税在 **折扣后小计 + 所有简单税** 的基础上计征：

```
复合税税额 = (折扣后小计 + 总简单税) × 税率%
总复合税 = Σ 所有复合税的税额
```

代码位于 `resources/scripts/admin/components/estimate-invoice-common/CreateTotalTaxes.vue`：
```javascript
if (props.tax.compound_tax && props.store.getSubtotalWithDiscount) {
  return Math.round(
    ((props.store.getSubtotalWithDiscount + props.store.getTotalSimpleTax) *
      props.tax.percent) / 100
  )
}
```

> **关键点**：
> - 复合税的基数 = 折扣后小计 + **简单税总和**
> - 复合税的基数**不包括其他复合税**
> - 复合税仅在整体税模式下生效

#### 3.4.3 税内包含模式下的整体税

```
不含税小计 = 折扣后小计 / (1 + 税率%)
税额 = 折扣后小计 - 不含税小计
```

代码位于 `CreateTotalTaxes.vue`。

### 3.5 整体税计算分支执行顺序

`CreateTotalTaxes.vue` 中 `taxAmount` computed 的判断顺序：

| 优先级 | 条件 | 计算公式 |
|-------|------|---------|
| 1 | `calculation_type === 'fixed'` | 直接返回 `fixed_amount` |
| 2 | `compound_tax === true` 且有折扣后小计 | 复合税公式：`(折扣后小计 + 总简单税) × 税率%` |
| 3 | `tax_included === true` 且有税率 | 税内包含公式：反算税额 |
| 4 | （默认）有税率 | 普通百分比税：`折扣后小计 × 税率%` |
| 5 | （无基数） | 返回 0 |

> **重要结论**：
> 1. **复合税优先于税内包含**：当一个税被标记为复合税时，即使 `tax_included = true`，也会按复合税公式计算，不会使用税内包含公式。
> 2. **简单税受 tax_included 影响**：非复合的简单税，在 `tax_included = true` 时使用反算公式。
> 3. **固定金额税优先级最高**：直接取固定值，不受其他任何开关影响。

### 3.6 行级税汇总（tax_per_item = 'YES'）

当 `tax_per_item === 'YES'` 时，总税为所有行项目的税之和：

```
总税 = Σ 每个行项目的 tax（行税合计）
```

代码：`invoice.js` 的 `getTotalTax` getter：
```javascript
getTotalTax() {
  if (
    this.newInvoice.tax_per_item === 'NO' ||
    this.newInvoice.tax_per_item === null
  ) {
    return this.getTotalSimpleTax + this.getTotalCompoundTax
  }
  return _.sumBy(this.newInvoice.items, function (tax) {
    return tax.tax
  })
}
```

同时在 `CreateTotal.vue` 中按税种聚合展示：

```javascript
const itemWiseTaxes = computed(() => {
  let taxes = []
  props.store[props.storeProp].items.forEach((item) => {
    if (item.taxes) {
      item.taxes.forEach((tax) => {
        let found = taxes.find((_tax) => _tax.tax_type_id === tax.tax_type_id)
        if (found) {
          found.amount += tax.amount
        } else if (tax.tax_type_id) {
          taxes.push({ ... })
        }
      })
    }
  })
  return taxes
})
```

### 3.7 最终总额

根据 `tax_included` 开关有两种计算方式：

| 模式 | 公式 | 代码位置 |
|-----|------|---------|
| 税外（默认） | `total = 折扣后小计 + 总税` | `invoice.js` 的 `getTotal` |
| 税内包含 | `total = 折扣后小计`（税已包含在内） | `invoice.js` 的 `getTotal` |

税内包含模式下，还会展示**净总额**（不含税金额）：

```
净总额(net_total) = 折扣后小计 - 总税
```

代码：`invoice.js` 的 `getNetTotal` getter

---

## 四、完整计算链图

### 4.1 整体税模式（tax_per_item = 'NO'）

```
单价 × 数量 = 行小计
                  ↘
单价 × 数量 = 行小计  →  Σ  →  小计(sub_total)  →  - 整体折扣  →  折扣后小计
                  ↗
单价 × 数量 = 行小计
                                          ↓
                     ┌────────────────────┴────────────────────┐
                     ↓                                         ↓
              简单税：折扣后小计 × 税率%                复合税：(折扣后小计 + Σ简单税) × 税率%
              （tax_included 时反算）                       （不受 tax_included 影响）
                     ↓                                         ↓
                     └────────────────────┬────────────────────┘
                                          ↓
                                     总税(getTotalTax)
                                          ↓
                              税外：折扣后小计 + 总税
                              税内：折扣后小计（税已含）
                                          ↓
                                     最终总额(total)
```

### 4.2 行级税模式（tax_per_item = 'YES'）

```
┌───────────────── 行1 ─────────────────┐
│  行小计 - 行折扣 = 行折扣后总额         │
│                          ↓             │
│                  税1: 行折扣后总额 × 税1%  │
│                  税2: 行折扣后总额 × 税2%  │
│                  税3: 行折扣后总额 × 税3%  │
│                          ↓             │
│                  行税合计 = 税1 + 税2 + 税3  │
│          （所有税均独立计算，无叠加）      │
└───────────────────────────────────────┘
              ↓
┌───────────────── 行2 ─────────────────┐
│  同上...                               │
└───────────────────────────────────────┘
              ↓
         Σ 所有行的行折扣后总额  →  小计(sub_total)
         Σ 所有行的行税合计     →  总税(getTotalTax)
              ↓
         最终总额计算（同整体税模式的最终步骤）
```

> **关键对比**：
> - 整体税模式：存在复合税，简单税计入复合税基数
> - 行级税模式：无复合税概念，每行的所有税均独立基于行折扣后金额计算

---

## 五、关键配置开关

| 配置项 | 位置 | 说明 |
|-------|------|------|
| `tax_per_item` | CompanySetting | `YES`=行级税，`NO`=整体税（决定是否启用复合税） |
| `discount_per_item` | CompanySetting | `YES`=行级折扣，`NO`=整体折扣 |
| `tax_included` | 发票级 | `true`=税内包含，`false`=税外（复合税不受此开关影响） |
| `tax_included_by_default` | CompanySetting | 默认是否启用税内包含 |
| `compound_tax` | TaxType | 是否为复合税（仅整体税模式下生效） |
| `calculation_type` | TaxType | `percentage`=百分比，`fixed`=固定金额 |

---

## 六、后端存储与验证

### 6.1 后端接收

前端计算完成后，通过 `app/Http/Requests/InvoicesRequest.php` 提交以下关键字段：

- `sub_total`：小计
- `discount_val`：折扣值
- `tax`：总税
- `total`：最终总额
- `items[].total`：行总额
- `items[].tax`：行税
- `items[].taxes[]`：行税种明细
- `taxes[]`：整体税种明细

请求验证在 `app/Http/Requests/InvoicesRequest.php` 的 `rules()` 方法中，仅做字段存在性和类型校验，**不校验金额的计算逻辑正确性**。

### 6.2 后端存储

后端在 `app/Models/Invoice.php` 的 `createInvoice()` 和 `createItems()` 方法中存储：

- 基础金额字段直接存储
- 同时存储 `base_*` 字段（按汇率换算为公司本位币）
- 行级税存储在 `taxes` 表，关联 `invoice_item_id`
- 整体税存储在 `taxes` 表，关联 `invoice_id`

关键代码：`Invoice.php` 的 `createItems()` 和 `createTaxes()` 方法

### 6.3 后端是否重算金额？——完全信任前端

> **核心结论**：后端**完全信任**前端传过来的所有金额字段，不做任何重算、校验、核对。前端算多少，后端存多少。

**证据链**：

1. **Invoice 主表创建**：`createInvoice()` 中直接使用 `$request->getInvoicePayload()`，而 `getInvoicePayload()` 直接使用前端传来的 `total`、`sub_total`、`tax`、`discount_val` 等字段。

   代码：`app/Http/Requests/InvoicesRequest.php` 的 `getInvoicePayload()` 方法：
   ```php
   return collect($this->except('items', 'taxes'))
       ->merge([
           // ...
           'base_total' => $this->total * $exchange_rate,        // 直接用前端传的 total
           'base_discount_val' => $this->discount_val * $exchange_rate,
           'base_sub_total' => $this->sub_total * $exchange_rate,
           'base_tax' => $this->tax * $exchange_rate,
           // ...
       ])
   ```

2. **行项目创建**：`createItems()` 中直接 `$invoice->items()->create($invoiceItem)`，金额字段全部来自前端。

3. **税创建**：`createTaxes()` 和 `createItems()` 中的行税创建，直接 `$item->taxes()->create($tax)`，`amount` 字段来自前端。

4. **汇率换算仅乘以汇率**：后端唯一做的"计算"只有 `base_* = 金额 × exchange_rate`，完全是线性放大，不涉及业务逻辑重算。

### 6.4 安全边界与风险

| 风险点 | 说明 | 影响程度 |
|-------|------|---------|
| 前端篡改金额 | 用户可以通过浏览器控制台修改任意金额字段 | ⚠️ 高 |
| 计算不一致 | 前端不同入口计算结果可能与后端不一致（但后端不验证） | ⚠️ 中 |
| 舍入误差累积 | 前端多次舍入后的值被直接存储 | ⚠️ 中 |
| 汇率换算使用前端金额 | `base_*` 字段也基于前端金额计算 | ⚠️ 中 |

**验证空白区**：

- ❌ 没有验证 `sub_total - discount_val + tax = total` 是否成立
- ❌ 没有验证 `items` 的 `total` 之和是否等于 `sub_total`
- ❌ 没有验证 `taxes` 的 `amount` 之和是否等于 `tax`
- ❌ 没有验证每个税的 `amount` 是否与税率计算结果一致
- ❌ 没有验证行项目数量×单价是否等于行小计

### 6.5 金额单位约定

- 数据库存储：**分**（integer）
- 前端输入展示：**元**（float）
- 转换：`存储值 = Math.round(显示值 × 100)`

---

## 七、复合税与税内包含的交互说明

### 7.1 整体税模式下的分支优先级

在 `CreateTotalTaxes.vue` 中，单个税种的税额计算遵循以下优先级：

```
固定金额税 → 复合税 → 税内包含简单税 → 普通简单税
```

这意味着：

| 场景 | 行为 |
|-----|------|
| 固定金额税 | 直接取固定值，忽略所有其他设置 |
| 复合税 + tax_included | 按复合税公式计算（tax_included 对复合税无效） |
| 简单税 + tax_included | 按税内包含公式反算税额 |
| 简单税（无 tax_included） | 按普通百分比公式计算 |

### 7.2 行级税模式下的分支优先级

在 `CreateItemRowTax.vue` 中，单个税种的税额计算遵循以下优先级：

```
固定金额税 → （整体折扣分摊） → 税内包含 → 普通百分比税
```

> 注意：行级税完全没有复合税分支，`compound_tax` 属性在行级税模式下被忽略。

---

## 八、代码文件索引

### 模型层
- `app/Models/TaxType.php` - 税率类型模型
- `app/Models/Tax.php` - 税实例模型
- `app/Models/InvoiceItem.php` - 发票行项目模型
- `app/Models/Invoice.php` - 发票主模型

### 请求验证层
- `app/Http/Requests/TaxTypeRequest.php` - 税率类型请求验证
- `app/Http/Requests/InvoicesRequest.php` - 发票请求验证

### 前端计算层
- `resources/scripts/admin/stores/invoice.js` - 发票状态管理（核心计算 getters）
- `resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue` - 行项目组件
- `resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue` - 行税计算组件
- `resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue` - 总额展示组件
- `resources/scripts/admin/components/estimate-invoice-common/CreateTotalTaxes.vue` - 整体税计算组件
- `resources/scripts/admin/components/estimate-invoice-common/NetTotal.vue` - 净总额展示组件

### 测试文件
- `tests/Unit/InvoiceTest.php` - 发票模型测试
- `tests/Unit/InvoiceItemTest.php` - 发票行项目测试
- `tests/Unit/TaxTest.php` - 税测试
- `tests/Unit/TaxTypeTest.php` - 税率类型测试
