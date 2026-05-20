# InvoiceShelf 公司上下文分析报告 v2

> 修正说明：本版本纠正了 v1 中关于外键删除行为、客户门户上下文解析、切换公司闭环以及无公司用户边界风险的错误描述。

---

## 1. 公司设置存储结构（修正版）

### 1.1 核心数据表

#### 1.1.1 `company_settings` 表 - 外键行为真相

**迁移文件**: `database/migrations/2019_09_26_145012_create_company_settings_table.php`

```php
$table->foreign('company_id')->references('id')->on('companies');
```

**⚠️ 关键纠正**：
- v1 错误："支持级联删除"
- **实际行为**：迁移中未指定 `onDelete()` 动作，默认使用数据库的 `RESTRICT`（拒绝删除）行为
- **实际删除方式**：通过 `Company::deleteCompany()` 方法**手动删除**：
  ```php
  // app/Models/Company.php:366
  public function deleteCompany($user)
  {
      // ... 先删除所有关联数据
      $this->settings()->delete();  // 手动删除公司设置
      $this->address()->delete();
      $this->delete();
  }
  ```

**风险点**：如果绕过 `deleteCompany()` 方法直接调用 `Company::destroy()`，数据库会因外键约束报错。

#### 1.1.2 `user_company` 表 - 多对多关联

**迁移文件**: `database/migrations/2021_07_01_060700_create_user_company_table.php`

```php
$table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
$table->foreign('company_id')->references('id')->on('companies')->onDelete('cascade');
```

**正确行为**：此表确实支持级联删除，删除用户或公司时自动解除关联。

#### 1.1.3 命名不一致：`slug` vs `unique_hash`

**路由定义** (`routes/web.php:45`, `routes/api.php:489`):
```php
Route::post('/{company:slug}/customer/login', CustomerLoginController::class);
Route::prefix('/{company:slug}/customer')->group(function () { ... });
```

**Company 模型字段**：
- 数据库字段：`unique_hash` (varchar, nullable)
- 生成方式：`Hashids::connection(Company::class)->encode($company->id)`
- **无 `slug` 字段**，也未定义 `getRouteKeyName()` 方法

**⚠️ 关键发现**：
Laravel 隐式模型绑定 `{company:slug}` 会尝试查找 `slug` 字段，但该字段不存在。实际能工作是因为：
1. Laravel 8+ 会优先使用模型的 `getRouteKeyName()` 方法
2. 如果未定义，则默认使用 `id` 字段
3. 但由于 `unique_hash` 是字符串，`id` 是整数，这里存在潜在的路由模型绑定失效风险

**正确的绑定方式**应该是：
```php
// 在 Company 模型中添加
public function getRouteKeyName()
{
    return 'unique_hash';
}
```
或者在路由中明确指定：`{company:unique_hash}`

---

## 2. 客户门户（Customer Portal）上下文解析链路

### 2.1 Web 路由入口

**文件**: `routes/web.php:129`

```php
Route::get('{company:slug}/customer/{vue?}', function (Company $company) {
    return view('app')->with([
        'customer_logo' => get_company_setting('customer_portal_logo', $company->id),
        'current_theme' => get_company_setting('customer_portal_theme', $company->id),
        'customer_page_title' => get_company_setting('customer_portal_page_title', $company->id),
    ]);
})->where('vue', '[\/\w\.-]*')->name('customer.dashboard')->middleware(['install']);
```

**解析流程**：
1. URL 格式：`https://example.com/abc123xyz/customer/invoices`
2. `abc123xyz` 被路由参数 `{company:slug}` 捕获
3. Laravel 尝试通过 `slug` 字段查找 Company（实际匹配不到，存在隐患）
4. 如果找到，注入到闭包函数的 `$company` 参数中
5. 从 `$company->id` 读取公司设置，传递给视图

### 2.2 API 路由分组

**文件**: `routes/api.php:489`

```php
Route::prefix('/{company:slug}/customer')->group(function () {
    Route::middleware(['auth:customer', 'customer-portal'])->group(function () {
        Route::get('/bootstrap', CustomerBootstrapController::class);
        Route::get('/dashboard', CustomerDashboardController::class);
        // ... 其他客户门户 API
    });
});
```

**⚠️ 重要差异**：
- 管理员 API 使用 **HTTP Header** (`company: <id>`) 传递公司上下文
- 客户门户 API 使用 **URL Path** (`/{company:slug}/customer/...`) 传递公司上下文
- 客户门户不经过 `CompanyMiddleware`，而是通过路由模型绑定直接解析

### 2.3 控制器中的使用模式

#### 方式 1：路由模型绑定（推荐）

**文件**: `app/Http/Controllers/V1/Customer/Invoice/InvoicesController.php:37`

```php
public function show(Company $company, $id)
{
    $invoice = $company->invoices()
        ->whereCustomer(Auth::guard('customer')->id())
        ->where('id', $id)
        ->first();
    // ...
}
```

- 优点：`$company` 已通过路由绑定自动解析，直接使用
- 安全：通过 `$company->invoices()` 关联查询，天然数据隔离

#### 方式 2：从 Customer 模型关联

**文件**: `app/Http/Controllers/V1/Customer/General/BootstrapController.php:39`

```php
public function __invoke(Request $request)
{
    $customer = Auth::guard('customer')->user();
    $companyLanguage = CompanySetting::getSetting('language', $customer->company_id);
    // ...
}
```

- Customer 模型有 `company_id` 字段，直接关联到所属公司
- 不需要路由参数中的 `company`

#### 方式 3：登录时的范围限定

**文件**: `app/Http/Controllers/V1/Customer/Auth/LoginController.php:21`

```php
public function __invoke(CustomerLoginRequest $request, Company $company)
{
    $user = Customer::where('email', $request->email)
        ->where('company_id', $company->id)  // 关键：限定在当前公司范围内查找
        ->first();
    // ...
}
```

**安全机制**：即使邮箱正确，如果该客户不属于当前公司，也会登录失败。

### 2.4 客户门户完整上下文链路

```
客户访问 URL: /{company:slug}/customer/login
    ↓
[Web 路由] 隐式模型绑定解析 Company
    ↓
[视图渲染] 注入 company settings (logo, theme, page_title)
    ↓
[前端] 客户提交登录表单 (POST /{company:slug}/customer/login)
    ↓
[API 路由] 再次解析 Company 模型
    ↓
[LoginController] 在 $company->id 范围内查找 Customer
    ↓
[登录成功] 设置 customer guard session
    ↓
[后续请求] 通过 customer guard 认证，从 Customer 模型获取 company_id
```

---

## 3. 切换公司：前端 Header 注入 → 后端回退 → Bootstrap 回写的完整闭环

### 3.1 切换公司的完整时序

#### 步骤 1：用户触发切换

**文件**: `resources/scripts/components/CompanySwitcher.vue:210`

```javascript
async function changeCompany(company) {
  await companyStore.setSelectedCompany(company)  // ① 更新 localStorage
  router.push('/admin/dashboard')                 // ② 路由跳转
  await globalStore.setIsAppLoaded(false)
  await globalStore.bootstrap()                   // ③ 重新加载配置
}
```

#### 步骤 2：更新本地存储

**文件**: `resources/scripts/admin/stores/company.js:20`

```javascript
setSelectedCompany(data) {
  window.Ls.set('selectedCompany', data.id)  // 写入 localStorage
  this.selectedCompany = data                // 更新 Pinia 状态
}
```

#### 步骤 3：HTTP 拦截器注入 Header

**文件**: `resources/scripts/http/index.js:15`

```javascript
instance.interceptors.request.use(function (config) {
  const companyId = Ls.get('selectedCompany')  // 从 localStorage 读取
  
  if (companyId) {
    config.headers.company = companyId         // 注入请求头
  }
  
  return config
})
```

**⚠️ 时间差问题**：
- 调用 `setSelectedCompany()` 后立即发起的请求（如 `bootstrap()`）会使用新的 companyId
- 但如果有在切换前就已排队的请求，仍会携带旧的 company header
- 实际代码中通过先路由跳转再 bootstrap 规避了这个问题

#### 步骤 4：后端中间件验证与回退

**文件**: `app/Http/Middleware/CompanyMiddleware.php:17`

```php
public function handle(Request $request, Closure $next): Response
{
    if (Schema::hasTable('user_company')) {
        $user = $request->user();
        
        // 验证逻辑：
        // 1. header 中没有 company，或者
        // 2. 用户不属于该公司
        if ((! $request->header('company')) || (! $user->hasCompany($request->header('company')))) {
            // 回退到用户的第一个公司
            $request->headers->set('company', $user->companies()->first()->id);
        }
    }
    
    return $next($request);
}
```

#### 步骤 5：ScopeBouncer 设置权限范围

**文件**: `app/Http/Middleware/ScopeBouncer.php:32`

```php
public function handle(Request $request, Closure $next): Response
{
    $user = $request->user();
    $tenantId = $request->header('company')
        ? $request->header('company')
        : $user->companies()->first()->id;  // 再次回退

    $this->bouncer->scope()->to($tenantId);
    return $next($request);
}
```

#### 步骤 6：Bootstrap 控制器返回新公司上下文

**文件**: `app/Http/Controllers/V1/Admin/General/BootstrapController.php:38`

```php
public function __invoke(Request $request)
{
    $current_user = $request->user();
    $companies = $current_user->companies;
    
    // 第三次验证：双重保险
    $current_company = Company::find($request->header('company'));
    if ((! $current_company) || ($current_company && ! $current_user->hasCompany($current_company->id))) {
        $current_company = $current_user->companies()->first();  // 第三次回退
    }
    
    $current_company_settings = CompanySetting::getAllSettings($current_company->id);
    $current_company_currency = Currency::find($current_company_settings->get('currency'));
    
    BouncerFacade::refreshFor($current_user);  // 刷新权限缓存
    
    return response()->json([
        'current_company' => new CompanyResource($current_company),
        'current_company_settings' => $current_company_settings,
        'current_company_currency' => $current_company_currency,
        'companies' => CompanyResource::collection($companies),
        // ... 其他数据
    ]);
}
```

#### 步骤 7：前端回写状态

**文件**: `resources/scripts/admin/stores/global.js:48`

```javascript
bootstrap() {
  return http.get('/api/v1/bootstrap').then((response) => {
    const companyStore = useCompanyStore()
    
    companyStore.companies = response.data.companies
    companyStore.selectedCompany = response.data.current_company
    
    // 关键：再次调用 setSelectedCompany 确保 localStorage 与后端一致
    companyStore.setSelectedCompany(response.data.current_company)
    
    companyStore.selectedCompanySettings = response.data.current_company_settings
    companyStore.selectedCompanyCurrency = response.data.current_company_currency
    
    this.isAppLoaded = true
  })
}
```

### 3.2 完整闭环图示

```
┌─────────────────────────────────────────────────────────────┐
│ 前端 (浏览器)                                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 用户点击切换公司                                       │  │
│  │  ↓                                                    │  │
│  │ companyStore.setSelectedCompany(company)              │  │
│  │   → 写入 localStorage.selectedCompany                 │  │
│  │   → 更新 Pinia 状态                                   │  │
│  │  ↓                                                    │  │
│  │ router.push('/admin/dashboard')                       │  │
│  │  ↓                                                    │  │
│  │ globalStore.bootstrap() → GET /api/v1/bootstrap       │  │
│  │   └─ axios 拦截器读取 localStorage 注入 company header │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↓                                │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTP 请求
┌────────────────────────────▼────────────────────────────────┐
│ 后端 (Laravel)                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ auth:sanctum 中间件 - 认证用户                         │  │
│  │  ↓                                                    │  │
│  │ CompanyMiddleware                                      │  │
│  │   → 验证 header 中的 company                           │  │
│  │   → 无效则回退到 users.companies.first()              │  │
│  │  ↓                                                    │  │
│  │ ScopeBouncer                                           │  │
│  │   → 设置 bouncer scope 到 company_id                   │  │
│  │  ↓                                                    │  │
│  │ BootstrapController                                    │  │
│  │   → 再次验证 company 归属                              │  │
│  │   → 无效则再次回退                                     │  │
│  │   → 查询新公司的 settings, currency                    │  │
│  │   → 返回新公司完整上下文                               │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↓                                │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTP 响应
┌────────────────────────────▼────────────────────────────────┐
│ 前端 (浏览器)                                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ globalStore.bootstrap() 接收响应                       │  │
│  │  ↓                                                    │  │
│  │ companyStore.selectedCompany = response.current_company│  │
│  │ companyStore.setSelectedCompany(...)                   │  │
│  │   → 再次写入 localStorage (确保与后端一致)             │  │
│  │  ↓                                                    │  │
│  │ companyStore.selectedCompanySettings = ...             │  │
│  │ companyStore.selectedCompanyCurrency = ...             │  │
│  │  ↓                                                    │  │
│  │ isAppLoaded = true → 渲染新公司的仪表盘                │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 三次回退机制的意义

| 层级 | 位置 | 回退触发条件 |
|------|------|-------------|
| 第一次 | CompanyMiddleware | header 为空或用户不属于该公司 |
| 第二次 | ScopeBouncer | header 为空（重复检查） |
| 第三次 | BootstrapController | company 不存在或用户无权限 |

**设计意图**：
1. 防御性编程：任何一层验证失败都有兜底
2. 用户体验：静默降级，不会报错
3. 数据安全：确保不会返回越权数据

---

## 4. 无公司关联用户的边界风险分析

### 4.1 风险场景

**场景**：用户存在于系统中，但 `user_company` 表中没有任何关联记录。

**可能原因**：
1. 系统管理员直接在数据库中创建用户，忘记关联公司
2. 用户的最后一个公司被删除（但 `CompaniesController::destroy()` 有保护逻辑）
3. 数据迁移错误导致关联丢失

### 4.2 代码中的风险点

#### 风险点 1：CompanyMiddleware

**文件**: `app/Http/Middleware/CompanyMiddleware.php:23`

```php
$request->headers->set('company', $user->companies()->first()->id);
// 如果用户没有任何公司：
// Error: Call to a member function id() on null
```

#### 风险点 2：ScopeBouncer

**文件**: `app/Http/Middleware/ScopeBouncer.php:37`

```php
$tenantId = $request->header('company')
    ? $request->header('company')
    : $user->companies()->first()->id;
// 同样问题：first() 返回 null 时调用 ->id 报错
```

#### 风险点 3：BootstrapController

**文件**: `app/Http/Controllers/V1/Admin/General/BootstrapController.php:41`

```php
$current_company = $current_user->companies()->first();
// $current_company 为 null 时，后续访问 $current_company->id 报错
```

#### 风险点 4：User 模型访问器

**文件**: `app/Models/User.php:96`

```php
public function getFormattedCreatedAtAttribute($value)
{
    $company_id = (CompanySetting::where('company_id', request()->header('company'))->exists())
        ? request()->header('company')
        : $this->companies()->first()->id;  // 同样风险
    // ...
}
```

#### 风险点 5：安装登录控制器

**文件**: `app/Http/Controllers/V1/Installation/LoginController.php:26`

```php
'company' => $user->companies()->first(),
// 安装完成后首次登录时可能还没有关联公司
```

### 4.3 现有保护机制

#### 保护 1：删除公司时的数量检查

**文件**: `app/Http/Controllers/V1/Admin/Company/CompaniesController.php:48`

```php
if ($user->loadCount('companies')->companies_count <= 1) {
    return respondJson('You_cannot_delete_all_companies', 'You cannot delete all companies');
}
```

确保用户至少保留一个公司。

#### 保护 2：创建用户时自动关联

**文件**: `app/Models/User.php:356`

```php
public static function createFromRequest(UserRequest $request)
{
    $user = self::create($request->getUserPayload());
    // ...
    $user->companies()->sync($companies->pluck('id'));  // 自动关联公司
    // ...
}
```

### 4.4 未覆盖的风险

1. **直接数据库操作**：绕过模型的手动数据操作可能导致无公司用户
2. **首次安装场景**：安装流程中创建的第一个用户可能在关联公司前就被使用
3. **并发删除**：极端并发下可能出现检查与删除之间的竞态条件
4. **API 令牌用户**：使用 Sanctum 令牌的非交互式用户可能绕过前端保护

### 4.5 修复建议

```php
// 在 CompanyMiddleware 中增加空值检查
public function handle(Request $request, Closure $next): Response
{
    if (Schema::hasTable('user_company')) {
        $user = $request->user();
        $userCompanies = $user->companies;
        
        if ($userCompanies->isEmpty()) {
            // 终止请求并返回明确错误
            return response()->json([
                'error' => 'user_has_no_companies',
                'message' => 'Your account is not associated with any company. Please contact administrator.'
            ], 403);
        }
        
        // ... 原有逻辑
    }
    
    return $next($request);
}
```

---

## 5. 架构总结（修正版）

### 5.1 核心设计模式

| 模式 | 应用场景 |
|------|----------|
| **双轨上下文传递** | 管理员用 Header，客户门户用 URL Path |
| **三层验证回退** | CompanyMiddleware → ScopeBouncer → BootstrapController |
| **键值对配置存储** | `company_settings` 表支持灵活扩展 |
| **无状态后端** | 不保存用户当前公司，每次请求重新解析 |

### 5.2 已知问题清单

| 问题 | 严重程度 | 影响 |
|------|----------|------|
| `company_settings` 外键无 cascade | 中 | 直接删除 Company 会报错 |
| 路由绑定 `{company:slug}` 与实际字段 `unique_hash` 不一致 | 高 | 潜在的路由模型绑定失败 |
| 无公司用户会导致 500 错误 | 高 | 系统崩溃 |
| 公司设置无缓存 | 中 | 数据库查询压力 |
| `request()->header('company')` 全局依赖 | 中 | 测试困难、上下文污染 |

### 5.3 关键文件速查表（修正版）

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| 公司删除逻辑（手动级联） | `app/Models/Company.php` | 272-373 |
| 公司中间件（第一层回退） | `app/Http/Middleware/CompanyMiddleware.php` | 17-28 |
| 权限范围中间件（第二层回退） | `app/Http/Middleware/ScopeBouncer.php` | 32-42 |
| Bootstrap 控制器（第三层回退） | `app/Http/Controllers/V1/Admin/General/BootstrapController.php` | 38-42 |
| 前端 HTTP 拦截器 | `resources/scripts/http/index.js` | 15-28 |
| 切换公司组件 | `resources/scripts/components/CompanySwitcher.vue` | 210-215 |
| 全局 Store Bootstrap | `resources/scripts/admin/stores/global.js` | 48-117 |
| 客户登录控制器 | `app/Http/Controllers/V1/Customer/Auth/LoginController.php` | 21-44 |
| 删除公司数量保护 | `app/Http/Controllers/V1/Admin/Company/CompaniesController.php` | 48-50 |
