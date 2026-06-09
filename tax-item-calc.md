# 发票行金额、税率叠加与最终合计算法全链梳理

## 概述

InvoiceShelf 的发票金额计算体系分为三层：**税率配置层** → **行级计算层** → **总额结算层**。所有金额均以**分**为最小单位存储（`integer` 类型），前端展示时除以 100。

---

## 一、税率配置层

### 1.1 税率类型模型

税率配置的核心模型为 [TaxType.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/TaxType.php)，支持两种计算方式：

| 计算方式 | 字段 | 说明 |
|---------|------|------|
| 百分比 | `calculation_type = 'percentage'` | 按 `percent` 字段的百分比计算 |
| 固定金额 | `calculation_type = 'fixed'` | 按 `fixed_amount` 字段的固定金额计算 |

### 1.2 关键税率属性

| 属性 | 类型 | 说明 |
|-----|------|------|
| `compound_tax` | boolean | **复合税**：在（小计 + 其他简单税）基础上计征 |
| `collective_tax` | boolean | 集体税标识（预留属性） |
| `percent` | float | 百分比税率值 |
| `fixed_amount` | integer | 固定税额（单位：分） |

税率配置的表单验证在 [TaxTypeRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Http/Requests/TaxTypeRequest.php) 中定义。

### 1.3 税实例模型

具体应用到发票或行项目上的税为 [Tax.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/Tax.php) 模型实例，包含：

- `tax_type_id`：关联的税率类型
- `amount`：实际计算出的税额（分）
- `percent` / `fixed_amount`：税率快照
- `calculation_type`：计算方式快照
- `compound_tax`：是否复合税快照

---

## 二、行级计算层

行级计算由 [CreateItemRow.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue) 组件和 [CreateItemRowTax.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue) 组件共同完成。

### 2.1 行金额计算公式

```
行小计(subtotal) = 单价(price) × 数量(quantity)
行折扣值(discount_val) = 行小计 × 折扣率(%)  或  固定折扣金额
行总额(total) = 行小计 - 行折扣值
```

核心代码位于 [CreateItemRow.vue L267-L281](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue#L267-L281)：

```javascript
const subtotal = computed(() => Math.round(props.itemData.price * props.itemData.quantity))
const total = computed(() => subtotal.value - props.itemData.discount_val)
```

### 2.2 行级税计算

当 `tax_per_item === 'YES'` 时，每个行项目可独立配置多个税种。行税计算在 [CreateItemRowTax.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue) 中实现。

#### 2.2.1 普通百分比税（税外）

```
单税税额 = 行折扣后总额 × 税率%
```

代码位于 [CreateItemRowTax.vue L175](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue#L175)：
```javascript
return (props.discountedTotal * localTax.percent) / 100
```

#### 2.2.2 税内包含（Tax Included）

当 `tax_included = true` 时，价格中已包含税，需反算税额：

```
不含税价格 = 含税总价 / (1 + 税率%)
税额 = 含税总价 - 不含税价格
```

代码位于 [CreateItemRowTax.vue L172-L174](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue#L172-L174)：
```javascript
return Math.round(props.discountedTotal - (props.discountedTotal / (1 + (localTax.percent / 100))))
```

#### 2.2.3 固定金额税

直接使用 `fixed_amount`，不随行金额变化。

#### 2.2.4 行税汇总

```
行总税 = Σ 该行所有税种的税额
```

代码位于 [CreateItemRow.vue L298-L307](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue#L298-L307)：
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

### 2.3 行级折扣与整体折扣的分摊

当 `discount_per_item === 'NO'`（整体折扣）且 `tax_per_item === 'YES'`（行级税）时，计算行税需先按比例分摊整体折扣：

```
行占比 = 行总额 / 所有行总额之和
行分摊折扣 = 总折扣值 × 行占比
行折扣后金额 = 行总额 - 行分摊折扣
```

代码位于 [CreateItemRowTax.vue L256-L283](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue#L256-L283) 的 `getTaxAmount()` 函数中。

---

## 三、总额结算层

总额计算在 [invoice.js](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js) Store 的 getters 中集中实现，并由 [CreateTotal.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue) 组件展示。

### 3.1 核心 Getter 一览

| Getter 名称 | 说明 |
|------------|------|
| `getSubTotal` | 所有行项目的 `total` 之和（行折扣后） |
| `getSubtotalWithDiscount` | 小计 - 整体折扣值 |
| `getTotalSimpleTax` | 所有简单税（非复合税）之和 |
| `getTotalCompoundTax` | 所有复合税之和 |
| `getTotalTax` | 总税额 = 简单税 + 复合税（整体税模式）或 行税之和（行级税模式） |
| `getTotal` | 最终发票总额 |
| `getNetTotal` | 净总额 = 折扣后小计 - 总税（税内包含模式下展示） |

### 3.2 小计计算

```
小计(sub_total) = Σ 每个行项目的 total（行折扣后金额）
```

代码：[invoice.js L47-L51](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js#L47-L51)

### 3.3 整体折扣计算

整体折扣支持两种模式：

- **百分比折扣**：`discount_val = sub_total × discount%`
- **固定金额折扣**：`discount_val = discount × 100`（输入为元，转为分）

代码位于 [CreateTotal.vue L323-L334](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue#L323-L334) 的 `setDiscount()` 函数中。

### 3.4 整体税计算（tax_per_item = 'NO'）

当 `tax_per_item === 'NO'` 时，税种应用于整单。

#### 3.4.1 简单税（Simple Tax）

```
简单税税额 = 折扣后小计 × 税率%
总简单税 = Σ 所有非复合税的税额
```

代码：[invoice.js L57-L64](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js#L57-L64)

#### 3.4.2 复合税（Compound Tax）

复合税在 **折扣后小计 + 所有简单税** 的基础上计征：

```
复合税税额 = (折扣后小计 + 总简单税) × 税率%
总复合税 = Σ 所有复合税的税额
```

代码位于 [CreateTotalTaxes.vue L64-L70](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotalTaxes.vue#L64-L70)：
```javascript
if (props.tax.compound_tax && props.store.getSubtotalWithDiscount) {
  return Math.round(
    ((props.store.getSubtotalWithDiscount + props.store.getTotalSimpleTax) *
      props.tax.percent) / 100
  )
}
```

#### 3.4.3 税内包含模式下的整体税

```
不含税小计 = 折扣后小计 / (1 + 税率%)
税额 = 折扣后小计 - 不含税小计
```

代码位于 [CreateTotalTaxes.vue L71-L77](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotalTaxes.vue#L71-L77)。

### 3.5 行级税汇总（tax_per_item = 'YES'）

当 `tax_per_item === 'YES'` 时，总税为所有行项目的税之和：

```
总税 = Σ 每个行项目的 tax（行税合计）
```

代码：[invoice.js L75-L85](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js#L75-L85)

同时在 [CreateTotal.vue L289-L313](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue#L289-L313) 中按税种聚合展示：

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

### 3.6 最终总额

根据 `tax_included` 开关有两种计算方式：

| 模式 | 公式 | 代码位置 |
|-----|------|---------|
| 税外（默认） | `total = 折扣后小计 + 总税` | [invoice.js L95](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js#L95) |
| 税内包含 | `total = 折扣后小计`（税已包含在内） | [invoice.js L92-L94](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js#L92-L94) |

税内包含模式下，还会展示**净总额**（不含税金额）：

```
净总额(net_total) = 折扣后小计 - 总税
```

代码：[invoice.js L53-L55](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js#L53-L55)

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
│                  行税1: 行折扣后总额 × 税1% │
│                  行税2: 行折扣后总额 × 税2% │
│                          ↓             │
│                  行税合计 = 行税1 + 行税2  │
└───────────────────────────────────────┘
              ↓
┌───────────────── 行2 ─────────────────┐
│  同上...                               │
└───────────────────────────────────────┘
              ↓
         Σ 所有行的行折扣后总额  →  小计(sub_total)
         Σ 所有行的行税合计     →  总税(getTotalTax)
              ↓
         最终总额计算（同整体税模式）
```

---

## 五、关键配置开关

| 配置项 | 位置 | 说明 |
|-------|------|------|
| `tax_per_item` | CompanySetting | `YES`=行级税，`NO`=整体税 |
| `discount_per_item` | CompanySetting | `YES`=行级折扣，`NO`=整体折扣 |
| `tax_included` | 发票级 | `true`=税内包含，`false`=税外 |
| `tax_included_by_default` | CompanySetting | 默认是否启用税内包含 |
| `compound_tax` | TaxType | 是否为复合税 |
| `calculation_type` | TaxType | `percentage`=百分比，`fixed`=固定金额 |

---

## 六、后端存储与验证

### 6.1 后端接收

前端计算完成后，通过 [InvoicesRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Http/Requests/InvoicesRequest.php) 提交以下关键字段：

- `sub_total`：小计
- `discount_val`：折扣值
- `tax`：总税
- `total`：最终总额
- `items[].total`：行总额
- `items[].tax`：行税
- `items[].taxes[]`：行税种明细
- `taxes[]`：整体税种明细

### 6.2 后端存储

后端在 [Invoice.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/Invoice.php) 的 `createInvoice()` 和 `createItems()` 方法中存储：

- 基础金额字段直接存储
- 同时存储 `base_*` 字段（按汇率换算为公司本位币）
- 行级税存储在 `taxes` 表，关联 `invoice_item_id`
- 整体税存储在 `taxes` 表，关联 `invoice_id`

关键代码：[Invoice.php L500-L560](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/Invoice.php#L500-L560)

### 6.3 金额单位约定

- 数据库存储：**分**（integer）
- 前端输入展示：**元**（float）
- 转换：`存储值 = Math.round(显示值 × 100)`

---

## 七、代码文件索引

### 模型层
- [TaxType.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/TaxType.php) - 税率类型模型
- [Tax.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/Tax.php) - 税实例模型
- [InvoiceItem.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/InvoiceItem.php) - 发票行项目模型
- [Invoice.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Models/Invoice.php) - 发票主模型

### 请求验证层
- [TaxTypeRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Http/Requests/TaxTypeRequest.php) - 税率类型请求验证
- [InvoicesRequest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/app/Http/Requests/InvoicesRequest.php) - 发票请求验证

### 前端计算层
- [invoice.js](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/stores/invoice.js) - 发票状态管理（核心计算 getters）
- [CreateItemRow.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRow.vue) - 行项目组件
- [CreateItemRowTax.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateItemRowTax.vue) - 行税计算组件
- [CreateTotal.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotal.vue) - 总额展示组件
- [CreateTotalTaxes.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/CreateTotalTaxes.vue) - 整体税计算组件
- [NetTotal.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/resources/scripts/admin/components/estimate-invoice-common/NetTotal.vue) - 净总额展示组件

### 测试文件
- [InvoiceTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/tests/Unit/InvoiceTest.php) - 发票模型测试
- [InvoiceItemTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/tests/Unit/InvoiceItemTest.php) - 发票行项目测试
- [TaxTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/tests/Unit/TaxTest.php) - 税测试
- [TaxTypeTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/123-InvoiceShelf/tests/Unit/TaxTypeTest.php) - 税率类型测试
