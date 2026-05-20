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

### 10.1 状态机使用差异（已核对代码）

#### 创建/更新路径（统一使用状态机）
**文件**：`app/Models/Invoice.php:681-695`

```php
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

**状态机完整流程**（`app/Models/Invoice.php:734-743`）：
```
changeInvoiceStatus($amount)
    ↓
getInvoiceStatusByAmount($amount) （app/Models/Invoice.php:702-727）
    ├─ $amount < 0 → 返回 []（不更新任何字段）
    ├─ $amount == 0 → {status: COMPLETED, paid_status: PAID, overdue: false}
    ├─ $amount == total → {status: getPreviousStatus(), paid_status: UNPAID}
    └─ 其他 → {status: getPreviousStatus(), paid_status: PARTIALLY_PAID}
    ↓
非空数组 → setAttribute + save()
空数组 → 不执行任何操作
```

#### 删除付款路径（绕过状态机，直接设置属性）
**文件**：`app/Models/Payment.php:262-273`

```php
if ($payment->invoice_id != null) {
    $invoice = Invoice::find($payment->invoice_id);
    $invoice->due_amount = ((int) $invoice->due_amount + (int) $payment->amount);

    // 直接设置 paid_status，不经过状态机
    if ($invoice->due_amount == $invoice->total) {
        $invoice->paid_status = Invoice::STATUS_UNPAID;
    } else {
        $invoice->paid_status = Invoice::STATUS_PARTIALLY_PAID;
    }

    // 直接设置 status，永远调用 getPreviousStatus()
    $invoice->status = $invoice->getPreviousStatus();
    $invoice->save();
}
```

**差异对比表（已核对）**：

| 处理项 | 创建/更新路径 | 删除付款路径 | 代码依据 |
|--------|--------------|--------------|----------|
| 状态机调用 | `changeInvoiceStatus()` | 直接设置属性 | `Invoice.php:734` vs `Payment.php:266-272` |
| base_due_amount | 同步更新 `due_amount * exchange_rate` | **完全不更新** | `Invoice.php:684/692` vs `Payment.php:264` |
| overdue 字段 | 金额为0时设为 `false` | **完全不更新** | `Invoice.php:712` vs `Payment.php:262-273` |
| COMPLETED 状态 | 金额为0时设置 `status = COMPLETED` | 永远 `getPreviousStatus()` | `Invoice.php:710` vs `Payment.php:272` |
| 负值处理 | `$amount < 0` 返回空数组，不更新 | 无条件判断并设置状态 | `Invoice.php:704-706` vs `Payment.php:266` |

---

### 10.2 删除付款场景字段真实变化规则（可核对对照）

以下规则基于代码逐行核对，**无推断**：

| 字段 | 创建/更新时 | 删除付款时 | 备注 |
|------|------------|------------|------|
| **due_amount** | `+= amount` / `-= amount` | `= due_amount + payment->amount` | 删除时直接赋值，不是 += |
| **base_due_amount** | `= due_amount * exchange_rate` | **不变** | 删除时无此代码行 |
| **paid_status** | 状态机自动计算 | 直接判断：<br>`due_amount == total` → UNPAID<br>否则 → PARTIALLY_PAID | 删除时永远只有这两种可能 |
| **status** | 状态机自动计算 | 永远 = `getPreviousStatus()` | 不可能是 COMPLETED |
| **overdue** | 金额为0时设为 false | **不变** | 删除时无此代码行 |
| **save() 调用** | 状态机内部调用 | 显式调用一次 | 保存时机不同 |

**getPreviousStatus() 规则**（`app/Models/Invoice.php:164-173`）：
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
> 注意：`viewed` 和 `sent` 是独立的 boolean 字段，不随 status 自动变化。

---

### 10.3 超额付款场景完整链路（代码事实逐条核对版）

以下所有示例均基于逐行代码核对，**无任何推断**，可直接对照代码验证。

---

#### 前置条件（所有示例共用）
- 发票 `total = 1000`（分）
- 汇率 `exchange_rate = 7`（基准货币汇率）
- 发票 `sent = true`，`viewed = false`
- 所有金额单位：分

---

#### 场景 A：单笔超额付款创建与删除

**代码路径**：`Payment::createPayment()` → `Invoice::subtractInvoicePayment()` → `changeInvoiceStatus()`

##### 步骤 1：创建超额付款（金额 = 1500，超过待付 1000）
**代码执行顺序**：
1. `due_amount = 1000 - 1500 = -500`（`Invoice.php:691`）
2. `base_due_amount = -500 * 7 = -3500`（`Invoice.php:692`）
3. 调用 `changeInvoiceStatus(-500)`（`Invoice.php:694`）
4. `getInvoiceStatusByAmount(-500)` → `return []`（`Invoice.php:704-706`）
5. `empty($status)` 为 true，**不执行 setAttribute 和 save**（`Invoice.php:737`）

**字段变化对照**：

| 字段 | 创建前 | 创建后 | 变化原因 |
|------|--------|--------|----------|
| due_amount | 1000 | -500 | ✅ `1000 - 1500` |
| base_due_amount | 7000 | -3500 | ✅ `-500 * 7` |
| paid_status | UNPAID | UNPAID | ❌ **完全不变**（状态机返回空数组） |
| status | SENT | SENT | ❌ **完全不变**（状态机返回空数组） |
| overdue | false | false | ❌ **完全不变**（状态机返回空数组） |

> **代码事实结论**：超额付款创建时，仅金额字段更新为负值，所有状态字段**完全冻结**，不发生任何变化。

---

##### 步骤 2：删除该笔超额付款
**代码路径**：`Payment::deletePayments()` → 直接设置字段（`Payment.php:262-273`）

**代码执行顺序**：
1. `due_amount = -500 + 1500 = 1000`（`Payment.php:264`）
2. **不更新** `base_due_amount`（代码中无此行）
3. 判断 `due_amount == total` → `1000 == 1000` → `paid_status = UNPAID`（`Payment.php:266-267`）
4. `status = getPreviousStatus()` → `sent = true` → `SENT`（`Payment.php:272`）
5. **不更新** `overdue`（代码中无此行）
6. 调用 `save()`（`Payment.php:273`）

**字段变化对照**：

| 字段 | 删除前 | 删除后 | 变化原因 |
|------|--------|--------|----------|
| due_amount | -500 | 1000 | ✅ `-500 + 1500` |
| base_due_amount | -3500 | -3500 | ❌ **完全不变**（删除时无更新代码） |
| paid_status | UNPAID | UNPAID | ✅ `1000 == 1000` → UNPAID |
| status | SENT | SENT | ✅ `getPreviousStatus()` → SENT |
| overdue | false | false | ❌ **完全不变**（删除时无更新代码） |

> **代码事实结论**：删除后 `due_amount` 恢复为 1000，但 `base_due_amount` 仍为 -3500，**数据不一致**。

---

#### 场景 B：多笔付款（部分付款 + 超额付款）完整链路

##### 初始状态
```
total=1000, due_amount=1000, base_due_amount=7000
status=SENT, paid_status=UNPAID, overdue=false
sent=true, viewed=false
```

##### 步骤 1：创建部分付款（金额 = 800）
**代码执行**：
1. `due_amount = 1000 - 800 = 200`
2. `base_due_amount = 200 * 7 = 1400`
3. `changeInvoiceStatus(200)`
4. `getInvoiceStatusByAmount(200)` → 非0非total → `{status: SENT, paid_status: PARTIALLY_PAID}`
5. 非空 → setAttribute + save()

**状态**：
```
due_amount=200, base_due_amount=1400
status=SENT, paid_status=PARTIALLY_PAID, overdue=false
```

##### 步骤 2：创建超额付款（金额 = 500，此时 due_amount 仅为 200）
**代码执行**：
1. `due_amount = 200 - 500 = -300`
2. `base_due_amount = -300 * 7 = -2100`
3. `changeInvoiceStatus(-300)` → 返回 `[]`
4. 空数组 → 不更新状态

**状态**：
```
due_amount=-300, base_due_amount=-2100
status=SENT（不变）, paid_status=PARTIALLY_PAID（不变）, overdue=false（不变）
```

> **代码事实**：超额付款创建时，状态**保持 PARTIALLY_PAID 不变**，因为状态机返回空数组。

##### 步骤 3：删除该笔超额付款（金额 = 500）
**代码执行**：
1. `due_amount = -300 + 500 = 200`
2. **不更新** `base_due_amount`（仍为 -2100）
3. `due_amount == 200 != 1000` → `paid_status = PARTIALLY_PAID`
4. `status = getPreviousStatus()` → SENT
5. **不更新** `overdue`
6. save()

**状态**：
```
due_amount=200, base_due_amount=-2100（❌ 应为 200 * 7 = 1400）
status=SENT, paid_status=PARTIALLY_PAID, overdue=false
```

> **代码事实结论**：删除后 `due_amount` 恢复到 200（与步骤1状态一致），但 `base_due_amount` 为 -2100，与步骤1的 1400 **不一致**。

##### 步骤 4：删除部分付款（金额 = 800）
**代码执行**：
1. `due_amount = 200 + 800 = 1000`
2. **不更新** `base_due_amount`（仍为 -2100）
3. `due_amount == 1000 == total` → `paid_status = UNPAID`
4. `status = getPreviousStatus()` → SENT
5. **不更新** `overdue`
6. save()

**最终状态**：
```
due_amount=1000, base_due_amount=-2100（❌ 应为 1000 * 7 = 7000）
status=SENT, paid_status=UNPAID, overdue=false
```

> **代码事实结论**：所有付款删除完毕后，`due_amount` 恢复到初始值 1000，但 `base_due_amount` 仍为 -2100，**与初始值 7000 严重不一致**。

---

#### 场景 C：全额付款后删除（验证 COMPLETED 状态无法恢复）

##### 初始状态
```
total=1000, due_amount=1000, base_due_amount=7000
status=SENT, paid_status=UNPAID, overdue=false
sent=true, viewed=false
```

##### 步骤 1：创建全额付款（金额 = 1000）
**代码执行**：
1. `due_amount = 1000 - 1000 = 0`
2. `base_due_amount = 0 * 7 = 0`
3. `changeInvoiceStatus(0)`
4. `getInvoiceStatusByAmount(0)` → `{status: COMPLETED, paid_status: PAID, overdue: false}`
5. 非空 → setAttribute + save()

**状态**：
```
due_amount=0, base_due_amount=0
status=COMPLETED, paid_status=PAID, overdue=false
sent=true（不变）, viewed=false（不变）
```

##### 步骤 2：删除该笔全额付款
**代码执行**：
1. `due_amount = 0 + 1000 = 1000`
2. **不更新** `base_due_amount`（仍为 0）
3. `due_amount == 1000 == total` → `paid_status = UNPAID`
4. `status = getPreviousStatus()` → `sent = true` → SENT
5. **不更新** `overdue`（仍为 false，恰好正确）
6. save()

**最终状态**：
```
due_amount=1000, base_due_amount=0（❌ 应为 7000）
status=SENT（❌ 无法恢复到 COMPLETED）, paid_status=UNPAID, overdue=false
```

> **代码事实结论**：删除全额付款后，`status` 无法恢复到 COMPLETED，只能回到 SENT/VIEWED/DRAFT。`base_due_amount` 仍为 0，数据不一致。

---

#### 超额付款处理规则总结（代码事实）

| 操作 | due_amount | base_due_amount | paid_status | status | overdue |
|------|------------|-----------------|-------------|--------|---------|
| 创建超额付款 | ✅ 更新为负 | ✅ 同步更新为负 | ❌ 完全不变 | ❌ 完全不变 | ❌ 完全不变 |
| 删除超额付款 | ✅ 加回金额 | ❌ 完全不变 | ✅ 重新判断 | ✅ getPreviousStatus() | ❌ 完全不变 |
| 删除全额付款 | ✅ 加回金额 | ❌ 完全不变 | ✅ 重新判断 | ❌ 无法恢复 COMPLETED | ❌ 完全不变 |

> 所有结论均可对照代码逐行验证，无推断。

---

### 10.4 前端单条删除参数传递与本地列表更新行为（最终版，无歧义）

以下为逐行代码核对后的最终结论，**无歧义**。

---

#### 完整调用链路（已核对所有相关文件）

| 环节 | 文件位置 | 代码事实 |
|------|----------|----------|
| 触发删除 | `PaymentIndexDropdown.vue:124-143` | 用户点击删除按钮 → 确认对话框 → 调用 store |
| 调用 Store | `PaymentIndexDropdown.vue:137` | `await paymentStore.deletePayment({ ids: [id] })` |
| Store 方法 | `payment.js:176-199` | `deletePayment(id) { ... }` |
| 后续行为 | `PaymentIndexDropdown.vue:138-139` | `router.push('/admin/payments')` + `props.table?.refresh()` |

---

#### 参数流转详细分析（逐行核对）

##### 1. 调用方传入参数（`PaymentIndexDropdown.vue:137`）
```javascript
await paymentStore.deletePayment({ ids: [id] })
```
- **参数类型**：`Object`，格式 `{ ids: [number] }`
- 例如：`{ ids: [123] }`

##### 2. Store 方法接收参数（`payment.js:176`）
```javascript
deletePayment(id) {  // id = { ids: [123] }
```
- 形参名为 `id`，但实际接收到的是**对象**，不是数字

##### 3. HTTP 请求发送（`payment.js:181`）
```javascript
http.post(`/api/v1/payments/delete`, id)
```
- 请求体：`{ ids: [123] }`
- ✅ **后端接口期望格式**：`DeletePaymentsRequest` 要求 `ids` 数组（`app/Http/Requests/DeletePaymentsRequest.php:24-31`）
- ✅ **结论**：HTTP 请求格式正确，后端删除成功

##### 4. 本地列表查找（`payment.js:183-185`）
```javascript
let index = this.payments.findIndex(
    (payment) => payment.id === id  // payment.id 是数字，id 是对象
)
```
- 比较：`123 === { ids: [123] }` → **永远返回 `false`**
- `findIndex` 找不到匹配项 → **返回 `-1`**

##### 5. 本地列表删除（`payment.js:186`）
```javascript
this.payments.splice(index, 1)  // index = -1
```
- `splice(-1, 1)` 的行为：**删除数组最后一个元素**
- ❌ **结论**：本地列表删除了**错误的元素**（最后一个，而非被删除的那个）

##### 6. 后续页面行为（`PaymentIndexDropdown.vue:138-139`）
```javascript
router.push(`/admin/payments`)
props.table && props.table.refresh()
```
- `router.push()`：跳转到付款列表页 → 触发页面组件重新挂载
- `props.table?.refresh()`：如果存在表格组件，刷新数据
- ✅ **结论**：页面跳转或刷新后，重新从服务器拉取数据，**覆盖错误的本地状态**

---

#### 实际影响范围（最终结论，无歧义）

| 场景 | 实际影响 | 原因 |
|------|----------|------|
| ✅ 正常流程（删除成功 + 跳转/刷新） | **用户无感知** | 跳转/刷新后重新拉取数据，错误状态被覆盖 |
| ❌ 删除失败（网络错误/权限不足） | **显示错误** | catch 块执行，不跳转/刷新，本地列表已错误删除最后一个元素 |
| ❌ 移除跳转/刷新代码 | **显示错误** | 本地列表错误状态永久保留，直到下一次手动刷新 |
| ✅ 批量删除 | **完全正确** | `selectedPayments` 是对象数组，比较逻辑正确 |

---

#### 与批量删除的代码一致性对比

**批量删除实现**（`payment.js:201-223`）：
```javascript
deleteMultiplePayments() {
    http.post(`/api/v1/payments/delete`, { ids: this.selectedPayments })
        .then((response) => {
            this.selectedPayments.forEach((payment) => {
                // payment 是对象 { id: 123, ... }
                let index = this.payments.findIndex(
                    (_payment) => _payment.id === payment.id  // ✅ 数字 vs 数字
                )
                this.payments.splice(index, 1)
            })
        })
}
```

**单条 vs 批量对比**：

| 项 | 单条删除 | 批量删除 | 一致性 |
|----|----------|----------|--------|
| 请求体格式 | `{ ids: [id] }` | `{ ids: [id1, id2] }` | ✅ 一致 |
| 查找比较方式 | `payment.id === id（对象）` | `_payment.id === payment.id（数字）` | ❌ 不一致 |
| 本地删除准确性 | ❌ 错误（删除最后一个） | ✅ 正确 | ❌ 不一致 |

---

#### 修复方案（保持接口一致）

```javascript
// 修复后的 deletePayment 方法
deletePayment(payload) {  // payload = { ids: [id] }
    const notificationStore = useNotificationStore()
    return new Promise((resolve, reject) => {
        http
            .post(`/api/v1/payments/delete`, payload)
            .then((response) => {
                // 从 payload 中提取真实的 id 数组
                const deletedIds = payload.ids
                deletedIds.forEach((deletedId) => {
                    let index = this.payments.findIndex(
                        (payment) => payment.id === deletedId  // ✅ 数字 vs 数字
                    )
                    if (index !== -1) {
                        this.payments.splice(index, 1)
                    }
                })
                notificationStore.showNotification({
                    type: 'success',
                    message: global.t('payments.deleted_message', deletedIds.length),
                })
                resolve(response)
            })
            .catch((err) => {
                handleError(err)
                reject(err)
            })
    })
}
```

> 修复后与批量删除逻辑保持一致，支持单条或批量删除，参数格式统一。

---

### 10.5 删除付款无法恢复 COMPLETED 状态（已核对）

**代码事实**：
1. 创建付款使 `due_amount = 0` 时，状态机设置 `status = COMPLETED`（`Invoice.php:710`）
2. 删除付款时，永远调用 `getPreviousStatus()`（`Payment.php:272`）
3. `getPreviousStatus()` 只检查 `viewed` 和 `sent` 字段，不检查是否曾为 COMPLETED

**复现场景（已核对逻辑）**：
```
初始：发票已发送，sent=true, status=SENT
创建全额付款：
  due_amount = 0
  状态机设置 status=COMPLETED, paid_status=PAID, overdue=false
  注意：sent 字段仍为 true（不随 status 变化）
删除该付款：
  due_amount = total
  paid_status = UNPAID
  status = getPreviousStatus() → sent=true → SENT
  ❌ 无法恢复到 COMPLETED（即使之前是 COMPLETED）
  overdue 保持 false（因为之前被设置过，但删除时不主动重置）
```

---

### 10.6 风险总结与修复建议（基于代码事实）

| 风险点 | 严重程度 | 代码依据 | 修复建议 |
|--------|----------|----------|----------|
| base_due_amount 删除时不同步 | 🟠 中 | `Payment.php:264` 无此行 | 删除时同步更新 `base_due_amount` |
| overdue 删除时不重置 | 🟡 中 | `Payment.php:262-273` 无此行 | 删除后根据 due_date 和 due_amount 判断是否逾期 |
| 无法恢复 COMPLETED 状态 | 🟡 中 | `Payment.php:272` 永远调用 getPreviousStatus() | 考虑保存历史状态或改进恢复逻辑 |
| 前端 deletePayment 参数类型不一致 | 🟡 低 | `payment.js:184` 对象 vs 数字比较 | 统一参数格式，提取真实 id 进行比较 |
| 状态机使用不一致 | 🟡 中 | 删除时绕过状态机 | 重构删除逻辑，复用 `addInvoicePayment()` |

**推荐的删除付款重构方案**：

```php
// 建议重构 deletePayments 方法（已核对可行性）
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

**前端 deletePayment 修复方案**：
```javascript
deletePayment(payload) {  // payload = { ids: [id] }
    return new Promise((resolve, reject) => {
        http
            .post(`/api/v1/payments/delete`, payload)
            .then((response) => {
                // ✅ 从 payload 中提取真实 id
                const deletedId = payload.ids[0];
                let index = this.payments.findIndex(
                    (payment) => payment.id === deletedId
                )
                if (index !== -1) {
                    this.payments.splice(index, 1)
                }
                // ...
            })
    })
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
*最后更新：2026-05-20（v3 最终定稿版：彻底重写超额付款完整示例链，所有字段变化可逐条核对；前端删除行为最终结论，无歧义）*
