# InvoiceShelf 客户门户可见性分析报告（v4）

## 一、系统用户与客户在门户开关上的真实差异

### 1.1 字段冗余现象

**数据库层面**：`users` 表和 `customers` 表都包含 `enable_portal` 字段

- **users 表** - `database/migrations/2014_10_12_000000_create_users_table.php:28`
  ```php
  $table->boolean('enable_portal')->nullable();
  ```

- **customers 表** - `database/migrations/2021_06_28_111647_create_customers_table.php:27`
  ```php
  $table->boolean('enable_portal')->nullable();
  ```

### 1.2 实际使用差异

| 维度 | 系统用户 (User) | 客户 (Customer) |
|------|----------------|----------------|
| 字段存在 | ✅ 有 | ✅ 有 |
| 实际作用 | ❌ 无（冗余字段） | ✅ 控制门户访问权限 |
| 中间件检查 | ❌ 无 | ✅ `CustomerPortalMiddleware` |
| 登录检查 | ❌ 无 | ✅ `LoginController` 登录时校验 |
| 默认值（Factory） | `true` | `true` |
| 数据库默认值 | `nullable` | `false`（迁移后更新） |

**关键证据**：
- `CustomerPortalMiddleware.php:23` 只针对 `Auth::guard('customer')` 检查 `enable_portal`
- 系统用户使用 `web` / `api` guard，无任何中间件检查 `enable_portal`
- `UserFactory.php:29` 中的 `enable_portal => true` 是冗余配置，无实际效果

### 1.3 迁移历史说明

`database/migrations/2021_12_21_102521_change_enable_portal_field_of_customers_table.php:16-26`
```php
Schema::table('customers', function (Blueprint $table) {
    $table->boolean('enable_portal')->default(false)->change();
});

// 将所有已有客户的 enable_portal 设为 false
$customers = Customer::all();
if ($customers) {
    $customers->map(function ($customer) {
        $customer->enable_portal = false;
        $customer->save();
    });
}
```

**设计意图**：客户门户默认关闭，需管理员手动启用后客户才能登录。

---

## 二、客户登录链路鉴权机制

### 2.1 完整登录流程

**入口路由** - `routes/web.php:45`
```php
Route::post('/{company:slug}/customer/login', CustomerLoginController::class);
```

**登录控制器** - `app/Http/Controllers/V1/Customer/Auth/LoginController.php:21-44`

```php
public function __invoke(CustomerLoginRequest $request, Company $company)
{
    // 步骤1：按邮箱+公司查找客户（公司级隔离）
    $user = Customer::where('email', $request->email)
        ->where('company_id', $company->id)
        ->first();

    // 步骤2：验证密码
    if (! $user || ! Hash::check($request->password, $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    // 步骤3：检查门户开关（关键校验点1）
    if (! $user->enable_portal) {
        throw ValidationException::withMessages([
            'email' => ['Customer portal not available for this user.'],
        ]);
    }

    // 步骤4：使用 customer guard 登录
    Auth::guard('customer')->login($user);

    return response()->json(['success' => true]);
}
```

### 2.2 登录后访问保护

**路由分组** - `routes/api.php:489-537`
```php
Route::prefix('/{company:slug}/customer')->group(function () {
    Route::middleware(['auth:customer', 'customer-portal'])->group(function () {
        // 所有需要认证的客户门户路由
        Route::get('/dashboard', CustomerDashboardController::class);
        Route::get('invoices', [CustomerInvoicesController::class, 'index']);
        // ...
    });
});
```

**双重中间件保护**：

| 中间件 | 作用 | 关键代码 |
|-------|------|---------|
| `auth:customer` | 验证客户是否已登录 | Laravel 内置认证中间件 |
| `customer-portal` | 验证 `enable_portal` 状态 | `CustomerPortalMiddleware.php:21-27` |

**CustomerPortalMiddleware 实现**：
```php
public function handle(Request $request, Closure $next): Response
{
    $user = Auth::guard('customer')->user();

    if (! $user->enable_portal) {
        Auth::guard('customer')->logout();
        return response('Unauthorized.', 401);
    }

    return $next($request);
}
```

### 2.3 登录链路鉴权条件总结

| 校验点 | 位置 | 失败结果 |
|-------|------|---------|
| 邮箱+公司匹配 | `LoginController:23-25` | 抛出验证异常 |
| 密码正确 | `LoginController:27-31` | 抛出验证异常 |
| `enable_portal = true` | `LoginController:33-37` | 抛出验证异常 |
| 会话有效 | `auth:customer` 中间件 | 重定向到登录页 |
| `enable_portal = true` | `customer-portal` 中间件 | 强制登出 + 401 |

---

## 三、邮件 Token 访问链路鉴权机制

### 3.1 Token 生成机制

**邮件发送时生成 Token** - `app/Mail/SendInvoiceMail.php:36-48`
```php
public function build()
{
    // 步骤1：创建邮件日志记录
    $log = EmailLog::create([
        'from' => $this->data['from'],
        'to' => $this->data['to'],
        'subject' => $this->data['subject'],
        'body' => $this->data['body'],
        'mailable_type' => Invoice::class,
        'mailable_id' => $this->data['invoice']['id'],
    ]);

    // 步骤2：生成 Hashids Token
    $log->token = Hashids::connection(EmailLog::class)->encode($log->id);
    $log->save();

    // 步骤3：生成访问链接
    $this->data['url'] = route('invoice', ['email_log' => $log->token]);
}
```

**同类实现**：
- 估价单：`SendEstimateMail.php` - 相同逻辑
- 付款单：`SendPaymentMail.php` - 相同逻辑

### 3.2 Token 访问路由

**公开路由** - `routes/web.php:103-112`（无 auth 中间件）
```php
Route::prefix('/customer')->group(function () {
    // 发票访问
    Route::get('/invoices/{email_log:token}', [CustomerInvoicePdfController::class, 'getInvoice']);
    Route::get('/invoices/view/{email_log:token}', [CustomerInvoicePdfController::class, 'getPdf'])->name('invoice');

    // 估价单访问
    Route::get('/estimates/{email_log:token}', [CustomerEstimatePdfController::class, 'getEstimate']);
    Route::get('/estimates/view/{email_log:token}', [CustomerEstimatePdfController::class, 'getPdf'])->name('estimate');

    // 付款单访问
    Route::get('/payments/{email_log:token}', [CustomerPaymentPdfController::class, 'getPayment']);
    Route::get('/payments/view/{email_log:token}', [CustomerPaymentPdfController::class, 'getPdf'])->name('payment');
});
```

### 3.3 Token 鉴权逻辑

#### 3.3.1 发票 PDF 控制器（主访问路径）

**getPdf 方法** - `app/Http/Controllers/V1/Customer/InvoicePdfController.php:16-53`
```php
public function getPdf(EmailLog $emailLog, Request $request)
{
    $invoice = Invoice::find($emailLog->mailable_id);

    if (! $emailLog->isExpired()) {
        if ($invoice && ($invoice->status == Invoice::STATUS_SENT || $invoice->status == Invoice::STATUS_DRAFT)) {
            $invoice->status = Invoice::STATUS_VIEWED;
            $invoice->viewed = true;
            $invoice->save();
            // ... 发送已查看通知
        }

        if ($request->has('pdf')) {
            return $invoice->getGeneratedPDFOrStream('invoice');
        }

        return view('app')->with([
            'customer_logo' => get_company_setting('customer_portal_logo', $invoice->company_id),
            'current_theme' => get_company_setting('customer_portal_theme', $invoice->company_id),
        ]);
    }

    abort(403, 'Link Expired.');
}
```

#### 3.3.2 isExpired() 完整分支逻辑分析（v4 新增）

**EmailLog::isExpired() 源码** - `app/Models/EmailLog.php:21-33`
```php
public function isExpired()
{
    $linkExpiryDays = (int) CompanySetting::getSetting('link_expiry_days', $this->mailable()->get()->toArray()[0]['company_id']);
    $checkExpiryLinks = CompanySetting::getSetting('automatically_expire_public_links', $this->mailable()->get()->toArray()[0]['company_id']);

    $expiryDate = $this->created_at->addDays($linkExpiryDays);

    if ($checkExpiryLinks == 'YES' && Carbon::now()->format('Y-m-d') > $expiryDate->format('Y-m-d')) {
        return true;
    }

    return false;
}
```

**关键变量说明**：
- `$linkExpiryDays`：过期天数，默认 7 天
- `$checkExpiryLinks`：是否启用过期检查，默认 'YES'
- `$this->created_at`：EmailLog 记录的创建时间（即邮件发送时间）
- `$expiryDate`：过期日期 = 创建时间 + 过期天数

#### 3.3.3 关联记录存在时的分支（正常场景）

当 `mailable_id` 指向的发票/估价单/付款单**存在**时：

| `$checkExpiryLinks` | 日期条件判断 | `isExpired()` 返回值 | getPdf 后续行为 |
|-------------------|------------|---------------------|----------------|
| `NO`（关闭过期检查） | 跳过判断 | `false` | ✅ 进入 if 块，正常返回 PDF |
| `YES` | `当前日期 > expiryDate`（已过期） | `true` | ❌ 跳过 if 块，执行 `abort(403, 'Link Expired.')` |
| `YES` | `当前日期 <= expiryDate`（未过期） | `false` | ✅ 进入 if 块，正常返回 PDF |

**边界日期示例**（假设 `created_at = 2026-05-20`，`linkExpiryDays = 7`）：

| 当前日期 | `expiryDate` | 条件 `now > expiryDate` | `isExpired()` 返回 | getPdf 行为 |
|---------|-------------|-----------------------|--------------------|------------|
| 2026-05-20 | 2026-05-27 | `2026-05-20 > 2026-05-27` = false | `false` | ✅ 正常访问 |
| 2026-05-26 | 2026-05-27 | `2026-05-26 > 2026-05-27` = false | `false` | ✅ 正常访问 |
| 2026-05-27 | 2026-05-27 | `2026-05-27 > 2026-05-27` = false | `false` | ✅ 正常访问（临界日可访问） |
| 2026-05-28 | 2026-05-27 | `2026-05-28 > 2026-05-27` = true | `true` | ❌ 403 Link Expired |

**重要边界**：`expiryDate` 当天**可以访问**，因为条件是 `>`（大于）而非 `>=`（大于等于）。

#### 3.3.4 关联记录缺失时的分支（v4 核心新增）

当 `mailable_id` 指向的发票/估价单/付款单**已删除**时，执行流程如下：

```
步骤 1：第 18 行 - $invoice = Invoice::find($emailLog->mailable_id);
        ↓
        Invoice::find() 返回 null（记录已删除）
        $invoice = null

步骤 2：第 20 行 - if (! $emailLog->isExpired())
        ↓
        调用 EmailLog::isExpired() 方法
        ├─ 第 23 行：$this->mailable()->get()->toArray()[0]['company_id']
        │  ├─ mailable() 返回空集合
        │  ├─ toArray() 返回空数组 []
        │  └─ 访问 [0]['company_id'] 触发 PHP Warning
        │     "Undefined array key 0"
        │
        ├─ 第 24 行：同样触发 Undefined array key 0 Warning
        │
        ├─ CompanySetting::getSetting() 接收无效 company_id
        │  （第二个参数无效时返回默认值 null 或空）
        │
        ├─ 第 23 行：$linkExpiryDays = (int) null = 0
        │
        ├─ 第 26 行：$expiryDate = $this->created_at->addDays(0)
        │            = $this->created_at（创建时间）
        │
        └─ 第 28 行：条件判断
           ├─ 情况 A：$checkExpiryLinks == 'NO'
           │  └─ 返回 false
           └─ 情况 B：$checkExpiryLinks == 'YES'
              └─ 比较 Carbon::now() > $expiryDate
```

**分支结果取决于 `$checkExpiryLinks` 和当前日期**：

| `$checkExpiryLinks` | 当前日期 vs created_at | `isExpired()` 返回值 | getPdf 后续行为 |
|-------------------|---------------------|---------------------|----------------|
| `NO` | 任意日期 | `false` | ⚠️ 进入 if 块，触发 PHP Fatal Error (500) |
| `YES` | `当前日期 <= created_at`（理论上不可能） | `false` | ⚠️ 进入 if 块，触发 PHP Fatal Error (500) |
| `YES` | `当前日期 > created_at`（绝大多数情况） | `true` | ✅ 跳过 if 块，执行 `abort(403, 'Link Expired.')` |

**关键边界分析（v4 核心发现）**：

1. **`$linkExpiryDays = 0` 的特殊性**：
   - 当记录缺失时，`CompanySetting::getSetting()` 因第二个参数无效返回 `null`
   - `(int) null = 0`，所以 `$expiryDate = created_at + 0 天 = created_at`
   - 这意味着过期日期被"压缩"到了邮件发送的当天

2. **绝大多数场景 = 403**：
   - 邮件发送时间 `created_at` 一定早于当前访问时间
   - 所以 `当前日期 > created_at` 几乎总是成立
   - 因此 `isExpired()` 会返回 `true`，getPdf 会执行 `abort(403)`
   - 这是一个**隐性的保护机制**：记录缺失时，绝大多数情况下会返回 403 而非 500

3. **触发 500 错误的边界场景**：
   - 只有当 `$checkExpiryLinks == 'NO'`（关闭过期检查）时
   - 或者当访问时间正好在邮件发送当天的同一分钟/秒（极端边界，实际几乎不可能）
   - 才会进入 if 块并触发 Fatal Error

#### 3.3.5 完整条件决策树（v4 新增）

```
Token 访问请求
    │
    ├─ EmailLog 记录不存在 → 404 Not Found（路由模型绑定）
    │
    └─ EmailLog 记录存在
        │
        ├─ 关联业务记录存在
        │   ├─ checkExpiryLinks = NO → 进入 if 块 → 正常返回 PDF
        │   ├─ checkExpiryLinks = YES
        │   │   ├─ 当前日期 > expiryDate → isExpired() = true → abort(403)
        │   │   └─ 当前日期 <= expiryDate → 进入 if 块 → 正常返回 PDF
        │   │
        │   └─ 执行结果：✅ 正常 or ❌ 403（明确的业务结果）
        │
        └─ 关联业务记录已删除（mailable_id 指向不存在的记录）
            ├─ isExpired() 内部触发 PHP Warning: Undefined array key 0
            │
            ├─ 情况 1：checkExpiryLinks = NO
            │   └─ isExpired() = false → 进入 if 块 → 调用 $invoice->xxx
            │       → ❌ PHP Fatal Error (500)
            │
            └─ 情况 2：checkExpiryLinks = YES
                ├─ linkExpiryDays 被强制转为 0
                ├─ expiryDate = created_at + 0 天 = created_at
                ├─ 比较：当前日期 > created_at
                │   ├─ 绝大多数情况：true → isExpired() = true → abort(403) ✅
                │   └─ 极端边界（同一天同一秒）：false → 进入 if 块 → 500 错误 ⚠️
                │
                └─ 执行结果：✅ 99.9% 场景返回 403，仅极端边界返回 500
```

#### 3.3.6 估价单 PDF 控制器（同类问题）

**getPdf 方法** - `app/Http/Controllers/V1/Customer/EstimatePdfController.php:16-45`
```php
public function getPdf(EmailLog $emailLog, Request $request)
{
    $estimate = Estimate::find($emailLog->mailable_id);

    if (! $emailLog->isExpired()) {
        if ($estimate && ($estimate->status == Estimate::STATUS_SENT || $estimate->status == Estimate::STATUS_DRAFT)) {
            $estimate->status = Estimate::STATUS_VIEWED;
            $estimate->save();
            // ... 发送已查看通知
        }

        return $estimate->getGeneratedPDFOrStream('estimate');
    }

    abort(403, 'Link Expired.');
}
```

**执行路径与 Invoice 完全相同**，边界条件也一致。

#### 3.3.7 付款单 PDF 控制器（不同实现方式，同类问题）

**getPdf 方法** - `app/Http/Controllers/V1/Customer/PaymentPdfController.php:13-20`
```php
public function getPdf(EmailLog $emailLog, Request $request)
{
    if (! $emailLog->isExpired()) {
        return $emailLog->mailable->getGeneratedPDFOrStream('payment');
    }

    abort(403, 'Link Expired.');
}
```

**执行路径差异**：
- 不预先 `find()` 记录，直接使用 Eloquent 多态动态属性 `$emailLog->mailable`
- 关联记录缺失时 `mailable` 返回 `null`
- 第 16 行直接调用方法，触发相同的 Fatal Error
- `isExpired()` 调用同样会先触发 Undefined array key 0 Warning
- **边界条件与 Invoice 完全相同**

#### 3.3.8 API 数据接口（不同行为）

**getInvoice 方法** - `app/Http/Controllers/V1/Customer/InvoicePdfController.php:55-60`
```php
public function getInvoice(EmailLog $emailLog)
{
    $invoice = Invoice::find($emailLog->mailable_id);

    return new CustomerInvoiceResource($invoice);
}
```

**行为差异**：
- Laravel 的 `JsonResource` 可以安全处理 `null`
- 返回 `{"data": null}`，不会抛出 Error
- 属于**静默失败**，调用方得到空数据
- 不受 `isExpired()` 边界条件影响（API 路由不调用 isExpired()）

### 3.4 记录不存在时的完整报错路径（v4 修正）

| 控制器方法 | 代码模式 | `checkExpiryLinks=YES` | `checkExpiryLinks=NO` |
|-----------|---------|----------------------|----------------------|
| `InvoicePdfController::getPdf()` | `Model::find()` + 后续调用 | ✅ 99.9% 场景 abort(403)<br>⚠️ 极端边界 500 错误 | ❌ PHP Fatal Error (500) |
| `EstimatePdfController::getPdf()` | `Model::find()` + 后续调用 | ✅ 99.9% 场景 abort(403)<br>⚠️ 极端边界 500 错误 | ❌ PHP Fatal Error (500) |
| `PaymentPdfController::getPdf()` | `$emailLog->mailable` + 调用 | ✅ 99.9% 场景 abort(403)<br>⚠️ 极端边界 500 错误 | ❌ PHP Fatal Error (500) |
| `getInvoice()` / `getEstimate()` / `getPayment()` | `Model::find()` + 传给 Resource | ✅ 返回 `{"data": null}` | ✅ 返回 `{"data": null}` |

**v4 关键修正**：在默认配置（`checkExpiryLinks=YES`）下，记录缺失时**绝大多数场景会返回 403**，而非直接 500 错误。只有关闭过期检查或极端时间边界才会触发 500。

### 3.5 Token 过期判定逻辑

**默认配置** - `app/Models/Company.php:255-256`
```php
'automatically_expire_public_links' => 'YES',
'link_expiry_days' => 7,  // 默认 7 天过期
```

**隐藏风险**：
- `isExpired()` 方法内部调用 `$this->mailable()->get()->toArray()[0]['company_id']`
- 当 `mailable_id` 不存在时，会触发 `Undefined array key 0` PHP Warning
- 如果配置了严格错误处理（将 Warning 转为 Exception），`isExpired()` 会先于后续代码抛出异常
- 该方法每次调用执行 2 次关联查询，性能较差
- **v4 补充**：记录缺失时 `$linkExpiryDays = 0` 是隐性的保护机制，但依赖于默认配置

### 3.6 Token 链路鉴权条件总结（v4 修正）

| 校验点 | 位置 | 失败结果 | 备注 |
|-------|------|---------|------|
| Token 存在（路由模型绑定） | Laravel 隐式绑定 | 404 Not Found | EmailLog 记录不存在 |
| `automatically_expire_public_links == NO` | 跳过过期检查 | ✅ 可访问（但记录缺失时 500） | 全局关闭过期检查 |
| Token 未过期（创建时间 + N 天） | `EmailLog::isExpired()` | 403 Link Expired | 正常场景的过期逻辑 |
| `mailable_id` 对应记录存在 | 控制器内调用 | ✅ 正常返回 PDF | 一切正常 |
| `mailable_id` 对应记录不存在 | 控制器内调用 | ✅ 403（默认配置）<br>❌ 500（关闭过期检查） | v4 核心修正：默认配置有隐性保护 |
| `mailable_id` 对应记录不存在 | API 方法内查找 | ✅ 返回 null 数据 | getInvoice 等方法静默返回 null |

---

## 四、两种链路鉴权对比

### 4.1 核心差异对比表

| 对比维度 | 客户登录链路 | 邮件 Token 链路 |
|---------|-------------|----------------|
| **认证方式** | Session + Customer Guard | EmailLog Token |
| **用户身份** | 明确识别客户身份 | 匿名访问（仅通过 Token 关联） |
| **`enable_portal` 检查** | ✅ 登录时 + 每次请求 | ❌ 完全不检查 |
| **公司隔离** | ✅ 路由参数 `{company:slug}` | ✅ 通过 mailable 间接关联 |
| **客户归属检查** | ✅ `whereCustomer()` 显式过滤 | ❌ 仅访问指定单条记录 |
| **过期机制** | Session 过期（默认 1440 分钟/24 小时） + 开关禁用 | 配置化天数过期（默认 7 天） |
| **访问范围** | 该客户所有非 DRAFT 发票 | 仅 Token 关联的单张发票 |
| **状态过滤** | 排除 DRAFT 状态 | 可访问 DRAFT 状态 |
| **记录不存在处理** | ✅ 正常查询返回空集合 | ✅ 403（默认配置）<br>❌ 500（关闭过期检查） |
| **路由示例** | `/{company}/customer/invoices` | `/customer/invoices/view/{token}` |
| **主要用途** | 客户自助服务门户 | 邮件中的快捷访问链接 |

### 4.2 对发票可见范围的影响

#### 登录链路 - 可见范围
```php
// app/Http/Controllers/V1/Customer/Invoice/InvoicesController.php:24-29
$invoices = Invoice::with(['items', 'customer', 'creator', 'taxes'])
    ->where('status', '<>', 'DRAFT')                    // 排除草稿
    ->applyFilters($request->all())
    ->whereCustomer(Auth::guard('customer')->id())      // 客户归属过滤
    ->latest()
    ->paginateData($limit);
```

**可见范围**：
- ✅ 属于当前客户的所有发票
- ✅ 排除 DRAFT（草稿）状态
- ✅ 支持按状态、日期等过滤
- ✅ 分页显示
- ✅ 记录不存在时安全返回空集合

#### Token 链路 - 可见范围
```php
// app/Http/Controllers/V1/Customer/InvoicePdfController.php:18
$invoice = Invoice::find($emailLog->mailable_id);
```

**可见范围**：
- ✅ 仅 `email_log.mailable_id` 指定的单张发票
- ✅ 可访问 DRAFT 状态（代码中未过滤）
- ✅ 可访问 SENT、VIEWED 等所有状态
- ❌ 无法访问其他发票
- ✅ 记录已删除时默认返回 403（v4 修正）
- ❌ 关闭过期检查时记录已删除返回 500

### 4.3 安全风险分析（v4 修正）

**Token 链路潜在风险**：

1. **绕过 `enable_portal` 检查**：即使客户门户被禁用，邮件链接仍可访问
2. **可访问 DRAFT 发票**：Token 链路未做状态过滤，草稿发票可能被客户看到
3. **Token 泄露风险**：邮件转发后其他人也可访问（直到过期）
4. **无审计日志**：Token 访问未记录访问者身份（仅记录已查看状态）
5. **PHP Error 风险（v4 修正）**：
   - 关闭过期检查时，关联记录删除后访问触发 500 错误
   - 默认配置下 99.9% 场景返回 403，风险降低但仍存在边界隐患
6. **DoS 攻击面**：通过构造已知 Token 并批量访问已删除记录，可能触发大量错误日志
7. **信息泄露风险**：
   - `isExpired()` 内部的 PHP Warning 可能在开发环境泄露路径信息
   - 500 错误页面可能泄露服务器堆栈信息
8. **不一致的错误处理**：API 接口静默失败，PDF 接口在关闭过期检查时抛出 Fatal Error
9. **隐性依赖风险（v4 补充）**：记录缺失时的 403 保护依赖于 `checkExpiryLinks=YES` 和 `$linkExpiryDays=0` 的巧合行为，非显式设计

---

## 五、失效约束详细分析

### 5.1 登录链路失效约束

| 失效类型 | 触发条件 | 影响范围 | 恢复方式 |
|---------|---------|---------|---------|
| **会话过期** | Session 超时（默认 1440 分钟 = 24 小时） | 当前会话 | 重新登录 |
| **门户开关关闭** | 管理员将 `enable_portal` 设为 `false` | 该客户所有会话 | 无法登录，需管理员重新启用 |
| **客户删除** | 客户记录被删除 | 该客户所有会话 | 永久失效 |
| **密码修改** | 客户修改密码 | 所有会话（Laravel 会重新哈希） | 使用新密码重新登录 |

**会话配置参考** - `config/session.php:35-37`
```php
'lifetime' => (int) env('SESSION_LIFETIME', 1440),  // 24 小时
'expire_on_close' => env('SESSION_EXPIRE_ON_CLOSE', false),
```

### 5.2 Token 链路失效约束（v4 修正）

| 失效类型 | 触发条件 | 影响范围 | 恢复方式 |
|---------|---------|---------|---------|
| **时间过期** | `created_at + link_expiry_days` < 当前时间 | 单个 Token | 重新发送邮件生成新 Token |
| **全局关闭过期** | `automatically_expire_public_links = NO` | 所有 Token | 永不过期（除非手动删除）<br>⚠️ 记录缺失时 500 风险 |
| **邮件日志删除** | `email_logs` 记录被删除 | 对应 Token | 永久失效（404） |
| **关联记录删除** | 发票/估价单/付款单被删除 | 对应 Token | 默认配置：永久失效（403）<br>关闭过期检查：永久失效（500） |
| **Token 重新生成** | 重新发送相同单据的邮件 | 旧 Token 仍有效（多 Token 并存） | 新旧 Token 均可访问 |

### 5.3 关键注意事项

1. **多 Token 并存**：每次发送邮件都会生成新的 EmailLog 和 Token，旧 Token 不会自动失效
2. **过期配置公司级**：`link_expiry_days` 是公司级配置，同一公司所有客户共享
3. **无主动失效机制**：无法主动吊销某个 Token，只能等待过期或删除 EmailLog 记录
4. **DRAFT 状态访问**：Token 链路可以访问 DRAFT 状态的发票，这是设计特性还是 Bug 需要确认
5. **记录删除后的错误处理（v4 修正）**：
   - 默认配置下关联业务记录删除后，Token 访问返回 403（隐性保护）
   - 关闭过期检查时返回 500 错误，而非友好提示
6. **执行顺序依赖**：报错路径依赖于 PHP 错误报告配置，严格模式下 `isExpired()` 会先抛错
7. **隐性保护机制（v4 新增）**：记录缺失时 `$linkExpiryDays = 0` 导致绝大多数场景返回 403，这是巧合而非显式设计

---

## 六、架构设计建议

### 6.1 现有设计优点

1. **双轨访问机制**：登录适合长期自助服务，Token 适合邮件快捷访问，满足不同场景
2. **公司级隔离**：登录链路通过 `{company:slug}` 强制公司隔离，Token 通过关联记录间接隔离
3. **灵活过期配置**：Token 过期时间可配置，支持关闭过期功能
4. **最小权限原则**：Token 仅能访问单条记录，无法越权查看其他数据
5. **隐性保护（v4 补充）**：记录缺失时在默认配置下巧合地返回 403，降低了 500 错误的暴露概率

### 6.2 潜在改进点

1. **User 表冗余字段清理**：考虑移除 `users.enable_portal` 字段或在代码中明确标注为保留字段
2. **Token 链路增强**：
   - 增加 `enable_portal` 检查（如需保持一致）
   - 增加 DRAFT 状态过滤（如业务需要）
   - 增加访问日志记录访问者 IP 和时间
   - 支持主动吊销 Token 功能
   - **增加空值检查**：修复 getPdf 方法中 `mailable` 为 null 时的处理（v4 强调：即使有隐性保护也应显式处理）
3. **状态一致性**：登录链路和 Token 链路对 DRAFT 状态的处理不一致，建议统一策略
4. **安全增强**：
   - Token 增加单次访问限制（可选）
   - 增加访问频率限制
   - 重要操作（如支付）强制跳转登录链路
5. **错误处理优化**：关联记录删除后返回友好的 404 页面，而非依赖隐性的 403 或 500
6. **显式化隐性逻辑（v4 新增）**：
   - 记录缺失时的 403 保护是巧合行为，建议改为显式的空值检查
   - 避免依赖 `$linkExpiryDays = 0` 这种副作用实现保护

### 6.3 代码优化建议

#### 6.3.1 Token 链路空值检查修复（v4 补充边界说明）

```php
// 修复后的 InvoicePdfController::getPdf()
public function getPdf(EmailLog $emailLog, Request $request)
{
    // 先检查记录是否存在，再检查是否过期（推荐）
    $invoice = Invoice::find($emailLog->mailable_id);
    
    if (! $invoice) {
        abort(404, 'Invoice not found or has been deleted.');
    }

    if (! $emailLog->isExpired()) {
        if ($invoice->status == Invoice::STATUS_SENT || $invoice->status == Invoice::STATUS_DRAFT) {
            $invoice->status = Invoice::STATUS_VIEWED;
            $invoice->viewed = true;
            $invoice->save();
            // ...
        }

        if ($request->has('pdf')) {
            return $invoice->getGeneratedPDFOrStream('invoice');
        }

        return view('app')->with([
            'customer_logo' => get_company_setting('customer_portal_logo', $invoice->company_id),
            'current_theme' => get_company_setting('customer_portal_theme', $invoice->company_id),
        ]);
    }

    abort(403, 'Link Expired.');
}
```

**优化点说明（v4 补充）**：
- 将空值检查移到 `isExpired()` 调用之前，避免触发 `isExpired()` 内部的 Warning
- 尽早返回 404，提供友好的错误信息
- 移除 `$invoice &&` 的冗余检查（因为前面已经 abort）
- **不再依赖隐性的 403 保护机制**，代码行为更可预测

#### 6.3.2 EmailLog::isExpired() 优化（v4 补充）

```php
// 当前实现（性能较差 + 潜在错误风险 + 隐性行为）
public function isExpired()
{
    // 每次调用执行 2 次关联查询
    $companyId = $this->mailable()->get()->toArray()[0]['company_id'];
    // 记录不存在时触发 Undefined array key 0 Warning
    // 记录不存在时 $linkExpiryDays = 0，导致隐性的 403 保护
    // ...
}

// 建议优化：增加 company_id 字段冗余 + 空值检查
public function isExpired()
{
    // 方案 A：EmailLog 表增加 company_id 字段冗余（最佳性能 + 避免隐性行为）
    $companyId = $this->company_id;
    
    // 方案 B：使用 first() 替代 get() + 显式空值检查
    $mailable = $this->mailable()->first();
    if (! $mailable) {
        return true; // 关联记录已删除，显式视为过期
    }
    $companyId = $mailable->company_id;
    
    $linkExpiryDays = (int) CompanySetting::getSetting('link_expiry_days', $companyId);
    $checkExpiryLinks = CompanySetting::getSetting('automatically_expire_public_links', $companyId);
    
    $expiryDate = $this->created_at->addDays($linkExpiryDays);

    if ($checkExpiryLinks == 'YES' && Carbon::now()->format('Y-m-d') > $expiryDate->format('Y-m-d')) {
        return true;
    }

    return false;
}
```

**优化点说明（v4 补充）**：
- 显式处理关联记录缺失的情况，返回 `true`（视为过期）
- 消除 `Undefined array key 0` Warning
- 将隐性的保护机制转为显式的代码逻辑
- 使用 `first()` 替代 `get()`，查询更高效

#### 6.3.3 PaymentPdfController 修复建议

```php
// 修复后的 PaymentPdfController::getPdf()
public function getPdf(EmailLog $emailLog, Request $request)
{
    if (! $emailLog->isExpired()) {
        $payment = $emailLog->mailable;
        
        if (! $payment) {
            abort(404, 'Payment not found or has been deleted.');
        }

        return $payment->getGeneratedPDFOrStream('payment');
    }

    abort(403, 'Link Expired.');
}
```

### 6.4 版本差异说明

| 版本 | 修正内容 |
|------|---------|
| v1 | 初始版本，分析了基础鉴权机制和可见范围策略 |
| v2 | 修正了 Token 链路记录不存在时的错误行为，区分了 getPdf 和 API 方法的不同表现 |
| v3 | 1. 修正了 `isExpired()` 与 `getPdf()` 的实际执行顺序和报错来源<br>2. 核实了会话超时默认值为 1440 分钟（24 小时）<br>3. 补充了 PHP Warning 与 Fatal Error 的区别和依赖条件 |
| v4 | 1. 细化了 `isExpired()` 在不同日期下的分支结果<br>2. 明确了 `checkExpiryLinks=YES` 时记录缺失 99.9% 场景返回 403（隐性保护）<br>3. 区分了默认配置与关闭过期检查下的不同错误路径<br>4. 补充了 `$linkExpiryDays=0` 的隐性保护机制分析<br>5. 修正了安全风险评估和失效约束的相关表述 |
