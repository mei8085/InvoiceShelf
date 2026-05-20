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

## 九、设计要点与注意事项

### 9.1 设计优点
1. **余额实时计算**：使用 `due_amount` 字段实时增减，性能高效
2. **状态集中管理**：`changeInvoiceStatus()` 统一处理状态转换
3. **超额付款支持**：允许 `due_amount` 为负，支持预付款场景
4. **API 嵌套返回**：PaymentResource 嵌套返回 Invoice，便于前端获取最新状态

### 9.2 潜在问题
1. **删除付款逻辑不一致**：删除时未调用 `changeInvoiceStatus()`，而是直接设置状态，可能导致逻辑分叉
2. **前端状态不同步**：付款操作后 invoiceStore 不会自动更新，需手动刷新
3. **超额付款状态停滞**：超额付款后状态保持不变，用户可能看不到明确提示

### 9.3 可优化点
1. 删除付款时统一调用 `changeInvoiceStatus()`，保持逻辑一致
2. 付款操作后通过事件或返回数据主动更新 invoiceStore
3. 超额付款时添加明确的 UI 提示（如"超额付款 XX 元"）

---

*文档生成时间：2026-05-20*
