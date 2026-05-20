# InvoiceShelf 付款记录对发票状态与余额的驱动链路分析

> 文档版本：v1.0  
> 代码版本：基于当前仓库 master 分支  
> 分析范围：付款创建/更新/删除 → 发票余额与状态 → 前端展示

---

## 一、核心数据模型定义

### 1.1 发票状态枚举

**文件**：`app/Models/Invoice.php:30-43`

```php
const STATUS_DRAFT = 'DRAFT';              // 草稿
const STATUS_SENT = 'SENT';                // 已发送
const STATUS_VIEWED = 'VIEWED';            // 已查看
const STATUS_COMPLETED = 'COMPLETED';      // 已完成（款项结清）

const STATUS_UNPAID = 'UNPAID';            // 未付款
const STATUS_PARTIALLY_PAID = 'PARTIALLY_PAID';  // 部分付款
const STATUS_PAID = 'PAID';                // 已付清
```

### 1.2 关键字段说明

| 字段 | 表 | 类型 | 说明 |
|------|-----|------|------|
| `total` | invoices | bigInteger | 发票总金额（分） |
| `due_amount` | invoices | bigInteger | 待付金额（分），2024.10 后允许负值 |
| `status` | invoices | string | 文档生命周期状态 |
| `paid_status` | invoices | string | 付款状态 |
| `amount` | payments | unsignedBigInteger | 付款金额（分） |
| `invoice_id` | payments | bigInteger | 关联发票ID（可为空） |

---

## 二、付款创建对发票的驱动链路

### 2.1 入口：PaymentsController::store()

**文件**：`app/Http/Controllers/V1/Admin/Payment/PaymentsController.php:47-54`

```php
public function store(PaymentRequest $request)
{
    $this->authorize('create', Payment::class);
    $payment = Payment::createPayment($request);
    return new PaymentResource($payment);
}
```

### 2.2 核心逻辑：Payment::createPayment()

**文件**：`app/Models/Payment.php:161-203`

```php
public static function createPayment($request)
{
    $data = $request->getPaymentPayload();

    // === 关键步骤：扣减发票余额 ===
    if ($request->invoice_id) {
        $invoice = Invoice::find($request->invoice_id);
        $invoice->subtractInvoicePayment($request->amount);  // 触发状态变更
    }

    $payment = Payment::create($data);
    // ... 序列号生成、汇率日志、自定义字段

    return Payment::with(['customer', 'invoice', 'paymentMethod', 'fields'])
        ->find($payment->id);
}
```

### 2.3 余额扣减：Invoice::subtractInvoicePayment()

**文件**：`app/Models/Invoice.php:689-695`

```php
public function subtractInvoicePayment($amount)
{
    $this->due_amount -= $amount;
    $this->base_due_amount = $this->due_amount * $this->exchange_rate;
    $this->changeInvoiceStatus($this->due_amount);  // 触发状态机
}
```

### 2.4 状态机：Invoice::changeInvoiceStatus()

**文件**：`app/Models/Invoice.php:734-743`

```php
public function changeInvoiceStatus($amount)
{
    $status = $this->getInvoiceStatusByAmount($amount);
    if (! empty($status)) {
        foreach ($status as $key => $value) {
            $this->setAttribute($key, $value);
        }
        $this->save();
    }
}
```

### 2.5 状态转换规则：Invoice::getInvoiceStatusByAmount()

**文件**：`app/Models/Invoice.php:702-727`

| 条件 | 返回数据 | 说明 |
|------|----------|------|
| `$amount < 0` | `[]` | 超额付款，**不更新状态** |
| `$amount == 0` | `{status: COMPLETED, paid_status: PAID, overdue: false}` | 款项结清 |
| `$amount == $this->total` | `{status: getPreviousStatus(), paid_status: UNPAID}` | 无付款 |
| 其他 | `{status: getPreviousStatus(), paid_status: PARTIALLY_PAID}` | 部分付款 |

---

## 三、付款更新对发票的驱动链路

### 3.1 入口：PaymentsController::update()

**文件**：`app/Http/Controllers/V1/Admin/Payment/PaymentsController.php:63-70`

```php
public function update(PaymentRequest $request, Payment $payment)
{
    $this->authorize('update', $payment);
    $payment = $payment->updatePayment($request);
    return new PaymentResource($payment);
}
```

### 3.2 核心逻辑：Payment::updatePayment()

**文件**：`app/Models/Payment.php:205-255`

处理三种场景：

#### 场景 A：新增发票关联
```php
if ($request->invoice_id && (!$this->invoice_id || $this->invoice_id !== $request->invoice_id)) {
    $invoice = Invoice::find($request->invoice_id);
    $invoice->subtractInvoicePayment($request->amount);
}
```

#### 场景 B：移除发票关联
```php
if ($this->invoice_id && (!$request->invoice_id || $this->invoice_id !== $request->invoice_id)) {
    $invoice = Invoice::find($this->invoice_id);
    $invoice->addInvoicePayment($this->amount);  // 回滚
}
```

#### 场景 C：同一发票但金额变更
```php
if ($this->invoice_id && $this->invoice_id === $request->invoice_id && $request->amount !== $this->amount) {
    $invoice = Invoice::find($this->invoice_id);
    $invoice->addInvoicePayment($this->amount);      // 先加回原金额
    $invoice->subtractInvoicePayment($request->amount); // 再扣除新金额
}
```

### 3.3 余额回滚：Invoice::addInvoicePayment()

**文件**：`app/Models/Invoice.php:681-687`

```php
public function addInvoicePayment($amount)
{
    $this->due_amount += $amount;
    $this->base_due_amount = $this->due_amount * $this->exchange_rate;
    $this->changeInvoiceStatus($this->due_amount);
}
```

---

## 四、付款删除对发票的驱动链路

### 4.1 入口：PaymentsController::delete()

**文件**：`app/Http/Controllers/V1/Admin/Payment/PaymentsController.php:72-85`

```php
public function delete(DeletePaymentsRequest $request)
{
    $this->authorize('delete multiple payments');
    $ids = Payment::whereCompany()->whereIn('id', $request->ids)->pluck('id');
    Payment::deletePayments($ids);
    return response()->json(['success' => true]);
}
```

### 4.2 核心逻辑：Payment::deletePayments()

**文件**：`app/Models/Payment.php:257-280`

```php
public static function deletePayments($ids)
{
    foreach ($ids as $id) {
        $payment = Payment::find($id);

        if ($payment->invoice_id != null) {
            $invoice = Invoice::find($payment->invoice_id);
            
            // === 关键步骤：加回付款金额 ===
            $invoice->due_amount = ((int) $invoice->due_amount + (int) $payment->amount);

            // === 状态恢复（不调用 changeInvoiceStatus）===
            if ($invoice->due_amount == $invoice->total) {
                $invoice->paid_status = Invoice::STATUS_UNPAID;
            } else {
                $invoice->paid_status = Invoice::STATUS_PARTIALLY_PAID;
            }

            $invoice->status = $invoice->getPreviousStatus();  // 恢复到发送/查看/草稿
            $invoice->save();
        }

        $payment->delete();
    }
    return true;
}
```

### 4.3 历史状态恢复：Invoice::getPreviousStatus()

**文件**：`app/Models/Invoice.php:164-173`

```php
public function getPreviousStatus()
{
    if ($this->viewed) {
        return self::STATUS_VIEWED;
    } elseif ($this->sent) {
        return self::STATUS_SENT;
    } else {
        return self::STATUS_DRAFT;
    }
}
```

> ⚠️ **注意**：删除付款时未调用 `changeInvoiceStatus()`，而是直接设置状态。这与创建/更新的处理方式不同。

---

## 五、部分付款与超额付款边界处理

### 5.1 部分付款（Partial Payment）

**触发条件**：`0 < 付款金额 < 发票 due_amount`

**状态流转**：
```
初始状态：paid_status = UNPAID, due_amount = 1000
付款 300 → due_amount = 700
  → getInvoiceStatusByAmount(700)
  → 0 < 700 < 1000 → paid_status = PARTIALLY_PAID
```

**代码验证**：`app/Models/Invoice.php:719-723`
```php
} else {
    $data = [
        'status' => $this->getPreviousStatus(),
        'paid_status' => Invoice::STATUS_PARTIALLY_PAID,
    ];
}
```

### 5.2 超额付款（Over Payment）

**触发条件**：`付款金额 > 发票 due_amount`

**数据库支持**：`database/migrations/2024_10_09_103306_modify_invoices_to_allow_negative_values.php`
- `due_amount` 从 `unsignedBigInteger` → `bigInteger`（允许负值）

**状态处理**：`app/Models/Invoice.php:704-706`
```php
if ($amount < 0) {
    return [];  // 返回空数组，不更新任何状态字段
}
```

**实际效果**：
| 操作前 | 付款 | due_amount | paid_status | status |
|--------|------|------------|-------------|--------|
| due=200, PAID, COMPLETED | 500 | -300 | **PAID（不变）** | **COMPLETED（不变）** |
| due=500, PARTIALLY_PAID | 800 | -300 | **PARTIALLY_PAID（不变）** | **SENT（不变）** |

> 💡 设计意图：超额付款时金额字段如实记录（可作为客户预付款），但状态保持最后一次有效状态，避免出现异常状态。

### 5.3 前端限制

**文件**：`resources/scripts/admin/views/payments/Create.vue:346-351`

```javascript
amount: {
    required: ...,
    between: helpers.withMessage(
        t('validation.payment_greater_than_due_amount'),
        between(0, paymentStore.currentPayment.maxPayableAmount)
    ),
}
```

前端通过 `maxPayableAmount` 限制付款金额不超过待付金额，但后端仍允许超额付款（前端校验可绕过）。

---

## 六、前端 Store 感知变化的完整链路

### 6.1 API 资源层返回结构

#### PaymentResource
**文件**：`app/Http/Resources/PaymentResource.php:15-59`

```php
return [
    'id' => $this->id,
    'amount' => $this->amount,
    'invoice_id' => $this->invoice_id,
    // 嵌套返回完整的 InvoiceResource
    'invoice' => $this->when($this->invoice()->exists(), function () {
        return new InvoiceResource($this->invoice);
    }),
    // ... 其他字段
];
```

#### InvoiceResource
**文件**：`app/Http/Resources/InvoiceResource.php:15-82`

```php
return [
    'id' => $this->id,
    'status' => $this->status,
    'paid_status' => $this->paid_status,
    'total' => $this->total,
    'due_amount' => $this->due_amount,
    'allow_edit' => $this->allow_edit,
    // ... 其他字段
];
```

### 6.2 付款创建后的前端更新

**文件**：`resources/scripts/admin/stores/payment.js:129-147`

```javascript
addPayment(data) {
    return new Promise((resolve, reject) => {
        http.post('/api/v1/payments', data)
            .then((response) => {
                // 1. 付款列表追加新记录
                this.payments.push(response.data)
                
                // 2. response.data.invoice 包含最新的发票状态
                //    但 invoiceStore 不会自动更新，需手动刷新
                notificationStore.showNotification(...)
                resolve(response)
            })
    })
}
```

### 6.3 付款删除后的前端更新

**文件**：`resources/scripts/admin/stores/payment.js:176-199`

```javascript
deletePayment(id) {
    return new Promise((resolve, reject) => {
        http.post('/api/v1/payments/delete', id)
            .then((response) => {
                // 1. 从付款列表移除
                let index = this.payments.findIndex(p => p.id === id)
                this.payments.splice(index, 1)
                
                // 2. 发票状态已在后端更新，但前端发票列表需重新拉取
                notificationStore.showNotification(...)
                resolve(response)
            })
    })
}
```

### 6.4 前端展示链路

以发票信息卡片为例：

**文件**：`resources/scripts/components/InvoiceInformationCard.vue:26-37`

```vue
<dt class="text-sm font-medium text-gray-500 capitalize">
    {{ $t('invoices.paid_status').toLowerCase() }}
</dt>
<dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">
    <BaseInvoiceStatusBadge :status="invoice.paid_status" class="px-3 py-1">
        <BaseInvoiceStatusLabel :status="invoice.paid_status" />
    </BaseInvoiceStatusBadge>
</dd>
```

**状态展示机制**：
1. `invoice.paid_status` 是响应式属性（来自 Pinia store）
2. 当 store 中的 invoice 对象更新时，组件自动重新渲染
3. `BaseInvoiceStatusBadge` 根据状态值显示不同颜色

### 6.5 跨页面状态同步问题

**当前实现的局限**：
- 付款操作后，`paymentStore` 更新了付款列表
- 但 `invoiceStore` 中的发票数据**不会自动同步**
- 需用户手动刷新发票列表页面，或重新进入发票详情页

**典型场景**：
1. 用户在「发票详情页」看到 due_amount = 1000, paid_status = UNPAID
2. 用户跳转到「创建付款页」，创建 600 元付款
3. 用户返回「发票详情页」（未刷新）→ 仍显示旧数据
4. 用户刷新页面 → 显示 due_amount = 400, paid_status = PARTIALLY_PAID

---

## 七、完整调用链汇总

### 7.1 创建付款调用链

```
前端: paymentStore.addPayment(data)
  ↓ POST /api/v1/payments
PaymentsController::store()
  ↓
Payment::createPayment($request)
  ├─ 有 invoice_id? → 是
  │   └─ Invoice::subtractInvoicePayment(amount)
  │       ├─ due_amount -= amount
  │       ├─ base_due_amount = due_amount * exchange_rate
  │       └─ Invoice::changeInvoiceStatus(due_amount)
  │           ├─ Invoice::getInvoiceStatusByAmount(due_amount)
  │           │   ├─ due_amount == 0 → COMPLETED + PAID
  │           │   ├─ due_amount == total → UNPAID
  │           │   ├─ 0 < due_amount < total → PARTIALLY_PAID
  │           │   └─ due_amount < 0 → []（不更新状态）
  │           └─ 非空 → save()
  ├─ Payment::create($data)
  └─ 返回 Payment + 关联 Invoice
  ↓
前端: paymentStore.payments.push(response.data)
     (invoiceStore 需手动刷新)
```

### 7.2 删除付款调用链

```
前端: paymentStore.deletePayment(id)
  ↓ POST /api/v1/payments/delete
PaymentsController::delete()
  ↓
Payment::deletePayments(ids)
  ├─ 循环每个付款
  │   ├─ 有 invoice_id? → 是
  │   │   ├─ due_amount += payment.amount
  │   │   ├─ due_amount == total?
  │   │   │   ├─ 是 → paid_status = UNPAID
  │   │   │   └─ 否 → paid_status = PARTIALLY_PAID
  │   │   ├─ status = getPreviousStatus()（VIEWED/SENT/DRAFT）
  │   │   └─ save()
  │   └─ payment->delete()
  └─ 返回 true
  ↓
前端: paymentStore.payments.splice(index, 1)
     (invoiceStore 需手动刷新)
```

---

## 八、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 发票状态常量 | `app/Models/Invoice.php` | 30-43 |
| 付款创建 | `app/Models/Payment.php` | 161-203 |
| 付款更新 | `app/Models/Payment.php` | 205-255 |
| 付款删除 | `app/Models/Payment.php` | 257-280 |
| 扣减余额 | `app/Models/Invoice.php` | 689-695 |
| 加回余额 | `app/Models/Invoice.php` | 681-687 |
| 状态转换规则 | `app/Models/Invoice.php` | 702-727 |
| 应用状态 | `app/Models/Invoice.php` | 734-743 |
| 历史状态恢复 | `app/Models/Invoice.php` | 164-173 |
| 付款资源 | `app/Http/Resources/PaymentResource.php` | 15-59 |
| 发票资源 | `app/Http/Resources/InvoiceResource.php` | 15-82 |
| 前端付款 Store | `resources/scripts/admin/stores/payment.js` | 129-199 |
| 前端发票 Store | `resources/scripts/admin/stores/invoice.js` | 122-136 |
| 付款创建页面 | `resources/scripts/admin/views/payments/Create.vue` | 全文件 |
| 发票信息卡片 | `resources/scripts/components/InvoiceInformationCard.vue` | 全文件 |
| 支持负数迁移 | `database/migrations/2024_10_09_103306_modify_invoices_to_allow_negative_values.php` | 全文件 |

---

## 十、删除付款分支深度分析：与创建/更新路径的差异对比

### 10.1 状态机使用差异

#### 创建/更新路径（统一使用状态机）
**文件**：`app/Models/Invoice.php:681-695`

```php
// addInvoicePayment 和 subtractInvoicePayment 都调用 changeInvoiceStatus
public function addInvoicePayment($amount)
{
    $this->due_amount += $amount;
    $this->base_due_amount = $this->due_amount * $this->exchange_rate;
    $this->changeInvoiceStatus($this->due_amount);  // ✅ 统一状态机入口
}

public function subtractInvoicePayment($amount)
{
    $this->due_amount -= $amount;
    $this->base_due_amount = $this->due_amount * $this->exchange_rate;
    $this->changeInvoiceStatus($this->due_amount);  // ✅ 统一状态机入口
}
```

**状态机流程**：
```
changeInvoiceStatus($amount)
    ↓
getInvoiceStatusByAmount($amount)
    ├─ $amount < 0 → 返回 []（不更新状态）
    ├─ $amount == 0 → COMPLETED + PAID + overdue=false
    ├─ $amount == total → getPreviousStatus() + UNPAID
    └─ 其他 → getPreviousStatus() + PARTIALLY_PAID
    ↓
非空 → setAttribute + save()
```

#### 删除付款路径（绕过状态机，直接设置）
**文件**：`app/Models/Payment.php:262-273`

```php
if ($payment->invoice_id != null) {
    $invoice = Invoice::find($payment->invoice_id);
    $invoice->due_amount = ((int) $invoice->due_amount + (int) $payment->amount);

    // ❌ 直接设置状态，绕过状态机
    if ($invoice->due_amount == $invoice->total) {
        $invoice->paid_status = Invoice::STATUS_UNPAID;
    } else {
        $invoice->paid_status = Invoice::STATUS_PARTIALLY_PAID;
    }

    $invoice->status = $invoice->getPreviousStatus();
    $invoice->save();
}
```

**差异对比表**：

| 处理项 | 创建/更新路径 | 删除付款路径 | 风险 |
|--------|--------------|--------------|------|
| 状态机调用 | `changeInvoiceStatus()` | 直接设置属性 | 状态逻辑分叉，维护困难 |
| 负值处理 | `$amount < 0` 返回空数组，不更新 | 无判断，可能设置错误状态 | 超额付款后删除可能导致状态异常 |
| overdue 字段 | 金额为 0 时设置 `overdue = false` | 不处理 overdue 字段 | 删除后 overdue 可能保持 true |
| COMPLETED 状态 | 金额为 0 时设置 `status = COMPLETED` | 永远调用 `getPreviousStatus()` | 无法恢复到 COMPLETED |

---

### 10.2 due_amount/base_due_amount 同步机制差异

#### 创建/更新路径（双字段同步更新）
**文件**：`app/Models/Invoice.php:681-695`

```php
$this->due_amount += $amount;
$this->base_due_amount = $this->due_amount * $this->exchange_rate;  // ✅ 同步更新
```

#### 删除付款路径（仅更新 due_amount）
**文件**：`app/Models/Payment.php:264`

```php
$invoice->due_amount = ((int) $invoice->due_amount + (int) $payment->amount);
// ❌ base_due_amount 未更新！
```

**风险分析**：

1. **数据不一致**：`due_amount` 和 `base_due_amount` 失去同步
2. **报表错误**：基准货币金额报表使用 `base_due_amount`，会显示错误数据
3. **汇率转换错误**：后续操作基于错误的 `base_due_amount` 计算

**复现场景**：
```
初始状态：due_amount = 800, base_due_amount = 800 * 7 = 5600 (汇率 7)
创建付款 300：
  due_amount = 500
  base_due_amount = 500 * 7 = 3500 ✅ 同步
删除该付款：
  due_amount = 500 + 300 = 800
  base_due_amount = 3500 (未更新) ❌ 不同步！
  应为 800 * 7 = 5600
```

---

### 10.3 边界值处理差异

#### 场景 A：超额付款后的删除

**创建超额付款**：
- 发票：total = 1000, due_amount = 1000
- 付款金额 = 1500（超额 500）
- 结果：due_amount = -500, 状态保持不变（不调用状态机更新）

**删除该超额付款**：
- 删除路径直接计算：due_amount = -500 + 1500 = 1000
- 然后判断：`due_amount == total` (1000 == 1000) → `paid_status = UNPAID` ✅ 正确

**但如果存在多笔付款**：
- 发票：total = 1000
- 付款1：800 → due_amount = 200, paid_status = PARTIALLY_PAID
- 付款2：500（超额）→ due_amount = -300, 状态保持 PARTIALLY_PAID
- 删除付款2：due_amount = -300 + 500 = 200
- 删除路径判断：`200 != 1000` → `paid_status = PARTIALLY_PAID` ✅ 正确

#### 场景 B：零金额付款删除

创建/更新路径通过状态机处理，但删除路径直接判断：
- 如果 `due_amount + 0 == total` → UNPAID
- 否则 → PARTIALLY_PAID

#### 场景 C：删除最后一笔付款后 due_amount < 0

**异常场景**：
- 发票：total = 1000
- 付款1：600 → due_amount = 400, PARTIALLY_PAID
- 付款2：500 → due_amount = -100, 状态保持 PARTIALLY_PAID
- 直接修改数据库将 due_amount 改为 -200（模拟数据错误）
- 删除付款1：due_amount = -200 + 600 = 400 → PARTIALLY_PAID ✅
- 删除付款2：due_amount = 400 + 500 = 900 → PARTIALLY_PAID ✅
- 结果：due_amount = 900 < total = 1000，但实际已付款 1100

---

### 10.4 前端删除调用参数流转风险

#### 参数不匹配 Bug

**调用方**：`resources/scripts/admin/components/dropdowns/PaymentIndexDropdown.vue:137`
```javascript
await paymentStore.deletePayment({ ids: [id] })  // 传入对象 { ids: [id] }
```

**Store 实现**：`resources/scripts/admin/stores/payment.js:176-199`
```javascript
deletePayment(id) {
    return new Promise((resolve, reject) => {
        http
            .post(`/api/v1/payments/delete`, id)  // id 实际上是 { ids: [id] }
            .then((response) => {
                let index = this.payments.findIndex(
                    (payment) => payment.id === id  // ❌ payment.id === { ids: [id] }
                )
                this.payments.splice(index, 1)  // ❌ index = -1，删除最后一个元素！
                // ...
            })
    })
}
```

**Bug 分析**：

1. **参数类型不匹配**：函数期望 `id` 是数字，但实际传入对象
2. **findIndex 永远失败**：`payment.id === { ids: [id] }` 恒为 false
3. **错误删除**：`index = -1`，`splice(-1, 1)` 删除数组最后一个元素
4. **数据不一致**：UI 显示删除了错误的付款记录，但后端实际删除了正确的记录

**批量删除对比**：
```javascript
deleteMultiplePayments() {
    http.post(`/api/v1/payments/delete`, { ids: this.selectedPayments })
        .then((response) => {
            this.selectedPayments.forEach((payment) => {
                let index = this.payments.findIndex(
                    (_payment) => _payment.id === payment.id  // ✅ 正确，payment 是对象
                )
                this.payments.splice(index, 1)
            })
        })
}
```

#### 前端参数流转图

```
PaymentIndexDropdown.vue:removePayment(id)
    ↓
paymentStore.deletePayment({ ids: [id] })  // ❌ 包装成对象
    ↓
http.post('/api/v1/payments/delete', { ids: [id] })  // ✅ 请求体正确
    ↓
后端成功删除 id 对应的付款
    ↓
前端 findIndex(payment.id === { ids: [id] })  // ❌ 永远失败
    ↓
splice(-1, 1)  // ❌ 删除数组最后一个元素
```

---

### 10.5 删除付款无法恢复 COMPLETED 状态

**状态流转限制**：

```
getPreviousStatus() 的判断逻辑：
    if ($this->viewed) → VIEWED
    elseif ($this->sent) → SENT
    else → DRAFT
```

**问题**：当发票状态为 COMPLETED 时，删除唯一付款后：
1. `due_amount` 恢复为 `total`
2. `paid_status` 设置为 UNPAID ✅
3. `status` 通过 `getPreviousStatus()` 恢复，但 **永远不会是 COMPLETED** ❌

**示例**：
```
初始：发票已发送，状态 SENT
创建付款全额支付：
  due_amount = 0
  status = COMPLETED (通过状态机设置)
  paid_status = PAID
删除该付款：
  due_amount = total
  paid_status = UNPAID
  status = getPreviousStatus()
    → $this->sent = true → SENT
  ❌ 无法恢复到 COMPLETED（即使之前是 COMPLETED）
```

---

### 10.6 风险总结与修复建议

| 风险点 | 严重程度 | 影响范围 | 修复建议 |
|--------|----------|----------|----------|
| 前端参数不匹配导致错误删除 | 🔴 高 | 所有单条删除操作 | 修正 `deletePayment` 函数参数处理 |
| base_due_amount 不同步 | 🟠 中 | 多货币报表、汇率计算 | 删除时同步更新 base_due_amount |
| 状态机使用不一致 | 🟡 中 | 状态逻辑维护 | 删除时调用 `changeInvoiceStatus()` |
| overdue 字段未重置 | 🟡 中 | 逾期统计 | 删除时根据新的 due_amount 重置 overdue |
| 无法恢复 COMPLETED 状态 | 🟡 中 | 状态展示 | 考虑保存历史状态或改进恢复逻辑 |

**推荐的删除付款重构方案**：

```php
// 建议重构 deletePayments 方法
public static function deletePayments($ids)
{
    foreach ($ids as $id) {
        $payment = Payment::find($id);

        if ($payment->invoice_id != null) {
            $invoice = Invoice::find($payment->invoice_id);
            // ✅ 复用 addInvoicePayment，确保状态机和双字段同步
            $invoice->addInvoicePayment($payment->amount);
        }

        $payment->delete();
    }
    return true;
}
```

---

## 十一、设计要点与注意事项

### 11.1 设计优点
1. **余额实时计算**：使用 `due_amount` 字段实时增减，性能高效
2. **状态集中管理**：`changeInvoiceStatus()` 统一处理状态转换
3. **超额付款支持**：允许 `due_amount` 为负，支持预付款场景
4. **API 嵌套返回**：PaymentResource 嵌套返回 Invoice，便于前端获取最新状态

### 11.2 潜在问题
1. **删除付款逻辑不一致**：删除时未调用 `changeInvoiceStatus()`，而是直接设置状态，可能导致逻辑分叉
2. **前端状态不同步**：付款操作后 invoiceStore 不会自动更新，需手动刷新
3. **超额付款状态停滞**：超额付款后状态保持不变，用户可能看不到明确提示

### 11.3 可优化点
1. 删除付款时统一调用 `changeInvoiceStatus()`，保持逻辑一致
2. 付款操作后通过事件或返回数据主动更新 invoiceStore
3. 超额付款时添加明确的 UI 提示（如"超额付款 XX 元"）

---

*文档生成时间：2026-05-20*  
*最后更新：2026-05-20（新增删除付款分支深度分析）*
