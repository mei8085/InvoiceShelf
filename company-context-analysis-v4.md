# InvoiceShelf 公司上下文分析报告 v4

> 修正说明：本版本对极端数据场景进行了代码级验证，所有结论分为"代码已证实"和"仍需假设"两栏，区分确定行为与推测行为。

---

## 1. Slug 重复或缺失时的解析行为

### 1.1 路由模型绑定的基础机制

**代码已证实**：
- Laravel 的 `{company:slug}` 语法是**显式指定字段绑定**，不依赖 `getRouteKeyName()`
- 无自定义 `resolveRouteBinding()` 方法，使用 Laravel 默认实现
- `RouteServiceProvider.php` 中无自定义绑定逻辑
- 执行逻辑等价于：`Company::where('slug', $value)->firstOrFail()`

**仍需假设**：
- 无显式指定软删除排除逻辑（`withTrashed`），假设使用默认行为

---

### 1.2 Web 链路（客户门户 SPA 入口）

**路由**：`routes/web.php:129`
```php
Route::get('{company:slug}/customer/{vue?}', function (Company $company) {
    return view('app')->with([
        'customer_logo' => get_company_setting('customer_portal_logo', $company->id),
        'current_theme' => get_company_setting('customer_portal_theme', $company->id),
        'customer_page_title' => get_company_setting('customer_portal_page_title', $company->id),
    ]);
})->where('vue', '[\/\w\.-]*');
```

#### 场景 1.2.1：Slug 不存在（404）

| 结论 | 代码已证实 | 仍需假设 |
|------|-----------|---------|
| Laravel 抛出 `ModelNotFoundException` | ✅（firstOrFail() 行为） | |
| 渲染 404 页面而非 500 错误 | | ✅（依赖异常处理配置） |
| 无额外日志记录 | | ✅（默认行为） |

**失败模式**：`GET /non-existent-slug/customer/invoices` → HTTP 404

#### 场景 1.2.2：Slug 为 NULL（数据库中 slug 字段为空）

| 结论 | 代码已证实 | 仍需假设 |
|------|-----------|---------|
| 查询条件 `WHERE slug = NULL` 在 SQL 中永远为 false | ✅（SQL 三值逻辑） | |
| 等价于 slug 不存在，同样 404 | ✅ | |
| 两次迁移回填已确保历史数据无 NULL | ✅（迁移代码可见） | |

#### 场景 1.2.3：Slug 重复（两家公司相同 slug）

| 结论 | 代码已证实 | 仍需假设 |
|------|-----------|---------|
| `firstOrFail()` 返回第一条匹配记录 | ✅（Laravel 查询构建器行为） | |
| 后创建的公司永远无法通过 slug 访问 | ✅（先入为主） | |
| 数据库无唯一索引，允许重复插入 | ✅（迁移中无 unique 约束） | |
| 表单验证仅检查 `name` 唯一，不检查 `slug` | ✅（`CompaniesRequest.php:25-29`） | |

**失败模式**：
- 公司 A（id=1, name="Acme Inc"）→ slug="acme-inc"
- 公司 B（id=2, name="Acme Inc."）→ slug="acme-inc"（Str::slug 会去掉点号）
- 访问 `/acme-inc/customer/...` 永远返回公司 A，公司 B 的客户无法登录

---

### 1.3 API 链路（客户门户接口）

**路由**：`routes/api.php:489`
```php
Route::prefix('/{company:slug}/customer')->group(function () {
    Route::middleware(['auth:customer', 'customer-portal'])->group(function () {
        Route::get('/bootstrap', CustomerBootstrapController::class);
        // ...
    });
});
```

#### 场景 1.3.1：Slug 不存在

| 结论 | 代码已证实 | 仍需假设 |
|------|-----------|---------|
| 路由绑定阶段失败，不会到达中间件 | ✅（路由绑定在中间件之前执行） | |
| 返回 JSON 格式的 404 响应 | | ✅（API 路由的异常处理） |

**失败模式**：`GET /api/v1/non-existent/customer/bootstrap` → HTTP 404 JSON

#### 场景 1.3.2：Slug 重复

| 结论 | 代码已证实 | 仍需假设 |
|------|-----------|---------|
| 与 Web 链路相同，返回第一条匹配的公司 | ✅ | |
| 控制器拿到错误的 Company 实例 | ✅ | |
| 控制器若基于 `$company->id` 查询数据，会返回错误公司的数据 | ✅（见 `PaymentsController.php:49-54`） | |

**风险代码**：
```php
// app/Http/Controllers/V1/Customer/Payment/PaymentsController.php:49
public function show(Company $company, $id)
{
    $payment = $company->payments()  // 使用了绑定的 company
        ->whereCustomer(Auth::guard('customer')->id())
        ->where('id', $id)
        ->first();
}
```

**潜在越权**：如果客户在两家公司都有账户，且两家公司 slug 相同，他可能通过访问错误的 slug 看到另一家公司的同名 payment id。

#### 场景 1.3.3：控制器中未使用绑定的 Company

部分控制器**不依赖**路由绑定的 Company，而是从 Auth 用户获取：

| 控制器方法 | 是否使用绑定 Company | 数据来源 |
|-----------|---------------------|---------|
| `InvoicesController@index` | ❌ | `Auth::guard('customer')->id()` |
| `PaymentsController@index` | ❌ | `Auth::guard('customer')->id()` |
| `BootstrapController@__invoke` | ❌ | `Auth::guard('customer')->user()->company_id` |
| `InvoicesController@show` | ✅ | `$company->invoices()` |
| `PaymentsController@show` | ✅ | `$company->payments()` |

**代码已证实**：
- `index` 方法不受 slug 重复影响（从 Auth 获取 company_id）
- `show` 方法受 slug 重复影响（使用绑定的 company）

---

### 1.4 登录接口（Web + API 混合）

**路由**：`routes/web.php:45`
```php
Route::post('/{company:slug}/customer/login', CustomerLoginController::class);
```

**控制器**：`app/Http/Controllers/V1/Customer/Auth/LoginController.php:21`
```php
public function __invoke(CustomerLoginRequest $request, Company $company)
{
    $user = Customer::where('email', $request->email)
        ->where('company_id', $company->id)  // 关键：使用绑定的 company
        ->first();
    
    // 验证密码...
}
```

| 场景 | 行为 | 代码已证实 |
|------|------|-----------|
| Slug 不存在 | 404，登录失败 | ✅ |
| Slug 重复 | 只能登录第一条匹配的公司，客户在第二条公司的账号无法登录 | ✅ |
| Slug 为 NULL | 404，登录失败 | ✅ |

---

## 2. Slug 改名、Unique_Hash 不变时的影响边界

### 2.1 各链路的标识使用情况

| 链路 | 主要标识 | 次要标识 |
|------|---------|---------|
| 客户门户路由 | `slug` | 无 |
| 邮件链接（重置密码） | `slug` | 无 |
| 客户门户 API | `slug` | 无 |
| 公共 PDF 链接（发票） | `invoice.unique_hash` | 无 |
| 公共 PDF 链接（报价单） | `estimate.unique_hash` | 无 |
| 公共 PDF 链接（付款） | `payment.unique_hash` | 无 |
| 报告下载 | `company.unique_hash` | 无 |

---

### 2.2 Slug 改名的影响范围

#### 2.2.1 立即失效的链接

| 链接类型 | 是否失效 | 代码已证实 |
|---------|---------|-----------|
| 客户门户主页 | ✅ 失效 | ✅（路由绑定使用 slug） |
| 书签保存的客户门户链接 | ✅ 失效 | ✅ |
| 历史邮件中的重置密码链接 | ✅ 失效 | ✅（`CustomerMailResetPasswordNotification.php:41` 使用 slug） |
| 客户分享的发票列表链接 | ✅ 失效 | ✅（前端从 URL 提取 slug） |

#### 2.2.2 不受影响的链接

| 链接类型 | 是否失效 | 代码已证实 |
|---------|---------|-----------|
| 已发送邮件中的 PDF 链接 | ❌ 不受影响 | ✅（使用 invoice/estimate/payment 的 unique_hash） |
| 报告下载链接 | ❌ 不受影响 | ✅（`Report/*Controller.php` 使用 company.unique_hash） |
| 管理员端 API | ❌ 不受影响 | ✅（使用 HTTP Header 传递 company id） |

**代码已证实**：PDF 链接路由不使用 company slug：
```php
// routes/web.php:89-97
Route::get('/invoices/pdf/{invoice:unique_hash}', InvoicePdfController::class);
Route::get('/estimates/pdf/{estimate:unique_hash}', EstimatePdfController::class);
Route::get('/payments/pdf/{payment:unique_hash}', PaymentPdfController::class);
```

---

### 2.3 Slug 改名后的业务中断分析

| 业务场景 | 影响程度 | 说明 | 代码已证实 |
|---------|---------|------|-----------|
| 客户日常访问 | 🔴 高 | 所有客户书签失效 | ✅ |
| 密码重置流程 | 🔴 高 | 历史邮件中的重置链接失效 | ✅ |
| 发票 PDF 查看 | 🟢 无 | PDF 使用 unique_hash 永久链接 | ✅ |
| 报价单 PDF 查看 | 🟢 无 | 同上 | ✅ |
| 付款凭证下载 | 🟢 无 | 同上 | ✅ |
| 财务报告下载 | 🟢 无 | 报告使用 company.unique_hash | ✅ |
| 已有客户 Session | 🟡 中 | 已登录客户的 session 仍有效，直到登出 | |
| API 调用（客户端） | 🔴 高 | 前端硬编码 slug 的 API 全部失败 | ✅ |

---

### 2.4 Unique_Hash 不变的意义

| 特性 | 说明 | 代码已证实 |
|------|------|-----------|
| 生成方式 | `Hashids::connection(Company::class)->encode($company->id)` | ✅（`CompaniesController.php:23`） |
| 不可变性 | 创建后永不修改，无更新逻辑 | ✅（代码中无 setUniqueHash 或更新 unique_hash 的逻辑） |
| 作用范围 | 仅用于报告下载，不用于客户门户主链路 | ✅ |

**仍需假设**：
- 无业务场景需要修改 unique_hash
- Hashids salt 未修改（修改会导致所有历史链接失效）

---

## 3. 无公司关联用户风险的真实代码路径验证

### 3.1 风险触发的核心前提

**必须同时满足**：
1. `user_company` 表已存在（`Schema::hasTable('user_company')` 返回 true）
2. 用户已通过 `auth:sanctum` 认证
3. `$user->companies()->count() === 0`

**仍需假设**：
- `Schema::hasTable()` 在生产环境有稳定结果（无数据库连接问题）

---

### 3.2 5 处 `companies()->first()` 调用的逐一验证

#### 调用点 1：`CompanyMiddleware.php:23`

```php
if ((! $request->header('company')) || (! $user->hasCompany($request->header('company')))) {
    $request->headers->set('company', $user->companies()->first()->id);
}
```

| 分析项 | 结果 |
|--------|------|
| 执行路径 | 所有经过 `auth:sanctum` + `company` 中间件的请求 |
| 无公司时行为 | `first()` 返回 null，调用 `->id` 抛出 `Error` |
| HTTP 响应 | HTTP 500（默认错误处理） |
| 是否一定会触发 | ✅ 是 |
| 代码已证实 | ✅ |

**触发路径**：
```
用户认证通过 → CompanyMiddleware::handle()
    → header 无效 → companies()->first()->id
    → Error: Call to a member function id() on null
```

---

#### 调用点 2：`ScopeBouncer.php:37`

```php
$tenantId = $request->header('company')
    ? $request->header('company')
    : $user->companies()->first()->id;
```

| 分析项 | 结果 |
|--------|------|
| 执行路径 | 所有经过 `bouncer` 中间件的请求 |
| 无公司时行为 | `first()` 返回 null，调用 `->id` 抛出 `Error` |
| HTTP 响应 | HTTP 500 |
| 是否一定会触发 | ✅ 是 |
| 代码已证实 | ✅ |

**注意**：`bouncer` 中间件通常在 `company` 中间件之后，所以如果 CompanyMiddleware 已经崩溃，这里不会执行。但如果有路由只经过 `bouncer` 不经过 `company`，这里会独立崩溃。

---

#### 调用点 3：`BootstrapController.php:41`

```php
$current_company = $current_user->companies()->first();
// 后续在 CompanyResource 中访问 $current_company->id
```

| 分析项 | 结果 |
|--------|------|
| 执行路径 | 调用 `/api/v1/bootstrap` |
| 无公司时行为 | `$current_company` 为 null，`CompanyResource` 序列化时崩溃 |
| HTTP 响应 | HTTP 500 |
| 是否一定会触发 | ✅ 是 |
| 代码已证实 | ✅ |

**触发点**：
`CompanyResource.php:18` 访问 `$this->id` 时崩溃。

---

#### 调用点 4：`User.php:96`（访问器）

```php
public function getFormattedCreatedAtAttribute($value)
{
    $company_id = (CompanySetting::where('company_id', request()->header('company'))->exists())
        ? request()->header('company')
        : $this->companies()->first()->id;
}
```

| 分析项 | 结果 |
|--------|------|
| 执行路径 | 访问 `$user->formatted_created_at` 属性时 |
| 触发条件 | ① 访问属性 ② header 无效 ③ 无公司关联 |
| 是否一定会触发 | ❌ 否（需满足 3 个条件） |
| 代码已证实 | ✅ |

**仍需假设**：
- 哪个 API 会返回 User 模型并触发这个访问器

---

#### 调用点 5：`Installation/LoginController.php:26`

```php
'company' => $user->companies()->first(),
```

| 分析项 | 结果 |
|--------|------|
| 执行路径 | 调用 `/api/v1/installation/login` |
| 触发条件 | 安装完成后，有 super admin 用户但无公司关联 |
| 是否一定会触发 | ❌ 否（正常安装流程会 seed 公司） |
| 代码已证实 | ✅ |

**正常流程保护**：
`UsersTableSeeder.php:36` 会执行 `$user->companies()->attach($company->id)`，所以正常安装不会触发。

---

### 3.3 风险分级（最终版）

#### 🔴 确定风险（满足前提就一定会触发）

| 风险点 | 触发条件 | 影响范围 |
|--------|---------|---------|
| CompanyMiddleware 崩溃 | 通过 auth + company 中间件的任何请求 | 所有管理员 API |
| ScopeBouncer 崩溃 | 通过 auth + bouncer 中间件的任何请求 | 大部分管理员 API |
| BootstrapController 崩溃 | 调用 /api/v1/bootstrap | 前端应用加载失败 |

#### 🟡 条件风险（需要额外触发条件）

| 风险点 | 额外条件 | 影响范围 |
|--------|---------|---------|
| User 访问器崩溃 | 访问 formatted_created_at + header 无效 | 特定 API 响应 |
| 安装登录崩溃 | 手动创建 super admin 跳过 seed | 安装后首次登录 |

#### 🟢 已规避（有保护机制）

| 风险点 | 保护机制 | 代码位置 |
|--------|---------|---------|
| 删除导致无公司 | 删除前检查 companies_count > 1 | `CompaniesController.php:48` |
| UI 创建用户无公司 | 自动 sync 公司关联 | `User.php:356` |
| 安装后无公司 | Seeder 自动 attach | `UsersTableSeeder.php:36` |

---

### 3.4 真实攻击/故障路径

```
┌─────────────────────────────────────────────────────────────┐
│ 🔴 确定故障路径（数据库操作失误）                           │
├─────────────────────────────────────────────────────────────┤
│ 1. DBA 直接在 users 表插入 super admin 账号                  │
│ 2. 忘记向 user_company 表插入关联记录                        │
│ 3. 管理员尝试登录 → auth:sanctum 成功                         │
│ 4. 前端调用 /api/v1/bootstrap                                 │
│ 5. CompanyMiddleware.php:23 → first() 返回 null → 崩溃        │
│ 6. 前端显示 500 错误，无法进入系统                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 🟡 条件故障路径（迁移失败）                                  │
├─────────────────────────────────────────────────────────────┤
│ 1. 部署新版本时，user_company 表迁移失败                      │
│ 2. Schema::hasTable('user_company') 返回 false                │
│ 3. CompanyMiddleware 跳过检查，不设置 header                   │
│ 4. 后续逻辑仍可能因无 company 上下文崩溃                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 极端场景下的设计缺陷总结

### 4.1 Slug 相关缺陷

| 缺陷 | 严重程度 | 代码已证实 |
|------|---------|-----------|
| slug 无唯一索引，允许重复 | 高 | ✅ |
| slug 生成仅基于 name，无法处理重名公司 | 中 | ✅ |
| slug 改名导致所有客户链接失效 | 高 | ✅ |
| 无 slug 变更历史记录，改名后无法追溯 | 中 | |

### 4.2 无公司用户相关缺陷

| 缺陷 | 严重程度 | 代码已证实 |
|------|---------|-----------|
| 5 处 `first()->id` 调用无空值检查 | 高 | ✅ |
| 错误页面不友好（500 而非 403） | 中 | ✅ |
| 无审计日志记录无公司用户的登录尝试 | 低 | |

### 4.3 架构权衡

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| 客户门户使用 slug 作为路由标识 | URL 友好，便于分享和记忆 | 改名成本高，重复风险 |
| 管理员端使用 header 传递 id | 灵活，无改名问题，支持动态切换 | 每个请求都要验证 |
| PDF 使用 unique_hash | 永久链接，不受公司信息变更影响 | URL 不友好 |

---

## 5. 关键结论对照表（v4 vs v3）

| 问题 | v3 结论 | v4 修正 |
|------|---------|---------|
| slug 重复的影响 | 仅提到路由冲突 | 区分了 Web/API/登录/控制器方法的不同影响，明确 index 方法不受影响 |
| unique_hash 用途 | 仅用于 PDF/报告 | 明确了 slug 改名时哪些链接失效、哪些继续有效 |
| 无公司用户风险 | 4 确定 + 2 推测 | 逐一验证 5 个调用点，区分 3 个确定 + 2 个条件 + 2 个已规避 |
| 路由绑定机制 | 简单描述 | 详细分析 firstOrFail() 在各种边缘数据下的行为 |
