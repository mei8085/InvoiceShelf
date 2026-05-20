# InvoiceShelf 客户门户可见性分析报告

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

**发票 PDF 控制器** - `app/Http/Controllers/V1/Customer/InvoicePdfController.php:16-53`
```php
public function getPdf(EmailLog $emailLog, Request $request)
{
    $invoice = Invoice::find($emailLog->mailable_id);

    // 仅检查 Token 是否过期，不检查客户登录状态
    if (! $emailLog->isExpired()) {
        // 标记为已查看（如果是 SENT 或 DRAFT 状态）
        if ($invoice && ($invoice->status == Invoice::STATUS_SENT || $invoice->status == Invoice::STATUS_DRAFT)) {
            $invoice->status = Invoice::STATUS_VIEWED;
            $invoice->viewed = true;
            $invoice->save();
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

### 3.4 Token 过期判定逻辑

**EmailLog 模型** - `app/Models/EmailLog.php:21-33`
```php
public function isExpired()
{
    $linkExpiryDays = (int) CompanySetting::getSetting(
        'link_expiry_days', 
        $this->mailable()->get()->toArray()[0]['company_id']
    );
    $checkExpiryLinks = CompanySetting::getSetting(
        'automatically_expire_public_links', 
        $this->mailable()->get()->toArray()[0]['company_id']
    );

    $expiryDate = $this->created_at->addDays($linkExpiryDays);

    if ($checkExpiryLinks == 'YES' && Carbon::now()->format('Y-m-d') > $expiryDate->format('Y-m-d')) {
        return true;
    }

    return false;
}
```

**默认配置** - `app/Models/Company.php:255-256`
```php
'automatically_expire_public_links' => 'YES',
'link_expiry_days' => 7,  // 默认 7 天过期
```

### 3.5 Token 链路鉴权条件总结

| 校验点 | 位置 | 失败结果 |
|-------|------|---------|
| Token 存在（路由模型绑定） | Laravel 隐式绑定 | 404 Not Found |
| `automatically_expire_public_links == NO` | 跳过过期检查 | ✅ 可访问 |
| Token 未过期（创建时间 + N 天） | `EmailLog::isExpired()` | 403 Link Expired |
| `mailable_id` 对应记录存在 | 控制器内查找 | 静默失败（返回 null） |

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
| **过期机制** | Session 过期 + 开关禁用 | 配置化天数过期（默认 7 天） |
| **访问范围** | 该客户所有非 DRAFT 发票 | 仅 Token 关联的单张发票 |
| **状态过滤** | 排除 DRAFT 状态 | 可访问 DRAFT 状态 |
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

### 4.3 安全风险分析

**Token 链路潜在风险**：
1. **绕过 `enable_portal` 检查**：即使客户门户被禁用，邮件链接仍可访问
2. **可访问 DRAFT 发票**：Token 链路未做状态过滤，草稿发票可能被客户看到
3. **Token 泄露风险**：邮件转发后其他人也可访问（直到过期）
4. **无审计日志**：Token 访问未记录访问者身份（仅记录已查看状态）

---

## 五、失效约束详细分析

### 5.1 登录链路失效约束

| 失效类型 | 触发条件 | 影响范围 | 恢复方式 |
|---------|---------|---------|---------|
| **会话过期** | Session 超时（Laravel 默认 120 分钟） | 当前会话 | 重新登录 |
| **门户开关关闭** | 管理员将 `enable_portal` 设为 `false` | 该客户所有会话 | 无法登录，需管理员重新启用 |
| **客户删除** | 客户记录被删除 | 该客户所有会话 | 永久失效 |
| **密码修改** | 客户修改密码 | 所有会话（Laravel 会重新哈希） | 使用新密码重新登录 |

### 5.2 Token 链路失效约束

| 失效类型 | 触发条件 | 影响范围 | 恢复方式 |
|---------|---------|---------|---------|
| **时间过期** | `created_at + link_expiry_days` < 当前时间 | 单个 Token | 重新发送邮件生成新 Token |
| **全局关闭过期** | `automatically_expire_public_links = NO` | 所有 Token | 永不过期（除非手动删除） |
| **邮件日志删除** | `email_logs` 记录被删除 | 对应 Token | 永久失效 |
| **关联记录删除** | 发票/估价单/付款单被删除 | 对应 Token | 永久失效 |
| **Token 重新生成** | 重新发送相同单据的邮件 | 旧 Token 仍有效（多 Token 并存） | 新旧 Token 均可访问 |

### 5.3 关键注意事项

1. **多 Token 并存**：每次发送邮件都会生成新的 EmailLog 和 Token，旧 Token 不会自动失效
2. **过期配置公司级**：`link_expiry_days` 是公司级配置，同一公司所有客户共享
3. **无主动失效机制**：无法主动吊销某个 Token，只能等待过期或删除 EmailLog 记录
4. **DRAFT 状态访问**：Token 链路可以访问 DRAFT 状态的发票，这是设计特性还是 Bug 需要确认

---

## 六、架构设计建议

### 6.1 现有设计优点

1. **双轨访问机制**：登录适合长期自助服务，Token 适合邮件快捷访问，满足不同场景
2. **公司级隔离**：登录链路通过 `{company:slug}` 强制公司隔离，Token 通过关联记录间接隔离
3. **灵活过期配置**：Token 过期时间可配置，支持关闭过期功能
4. **最小权限原则**：Token 仅能访问单条记录，无法越权查看其他数据

### 6.2 潜在改进点

1. **User 表冗余字段清理**：考虑移除 `users.enable_portal` 字段或在代码中明确标注为保留字段
2. **Token 链路增强**：
   - 增加 `enable_portal` 检查（如需保持一致）
   - 增加 DRAFT 状态过滤（如业务需要）
   - 增加访问日志记录访问者 IP 和时间
   - 支持主动吊销 Token 功能
3. **状态一致性**：登录链路和 Token 链路对 DRAFT 状态的处理不一致，建议统一策略
4. **安全增强**：
   - Token 增加单次访问限制（可选）
   - 增加访问频率限制
   - 重要操作（如支付）强制跳转登录链路

### 6.3 代码优化建议

**EmailLog::isExpired() 性能优化**：
```php
// 当前实现（性能较差）
public function isExpired()
{
    // 每次调用都执行查询获取 mailable，再取 company_id
    $companyId = $this->mailable()->get()->toArray()[0]['company_id'];
    // ...
}

// 建议优化（使用延迟加载或缓存）
public function isExpired()
{
    // EmailLog 增加 company_id 字段冗余，避免关联查询
    $companyId = $this->company_id;
    // ...
}
```
