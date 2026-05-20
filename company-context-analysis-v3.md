# InvoiceShelf 公司上下文分析报告 v3

> 修正说明：本版本纠正了 v2 中关于 slug 字段不存在、路由绑定失效等关键误判，明确了 slug 与 unique_hash 的职责分工，并对无公司用户风险进行了分级收敛。

---

## 1. `companies` 表 `slug` 字段的真实来源与回填过程

### 1.1 字段添加时间线

#### 阶段 1：初始迁移（2014_10_11）
**文件**: `database/migrations/2014_10_11_071840_create_companies_table.php`

初始 companies 表**不包含** `slug` 字段，仅有：
- `id`, `name`, `logo`, `unique_hash`, `created_at`, `updated_at`

#### 阶段 2：字段添加与首次回填（2021_07_06）
**文件**: `database/migrations/2021_07_06_070204_add_owner_id_to_companies_table.php`

```php
Schema::table('companies', function (Blueprint $table) {
    $table->string('slug')->nullable();  // 新增 slug 字段
    $table->unsignedInteger('owner_id')->nullable();
    $table->foreign('owner_id')->references('id')->on('users');
});

// 回填逻辑
$companies = Company::all();
if ($companies && $user) {
    foreach ($companies as $company) {
        $company->owner_id = $user->id;
        $company->slug = Str::slug($company->name);  // 根据公司名称生成 slug
        $company->save();
    }
}
```

#### 阶段 3：二次回填（2022_01_06）
**文件**: `database/migrations/2022_01_06_103536_add_slug_to_companies.php`

```php
// 为 slug 为 null 的公司补全数据
$companies = Company::where('slug', null)->get();
if ($companies) {
    foreach ($companies as $company) {
        $company->slug = Str::slug($company->name);
        $company->save();
    }
}
```

**⚠️ 重要纠正（v2 错误）**：
- v2 断言："Company 模型无 slug 字段"
- **实际情况**：`slug` 字段自 2021 年 7 月起就存在于 companies 表中
- 两次迁移确保了历史数据的完整回填

### 1.2 新公司创建时的 slug 生成

**文件**: `app/Http/Requests/CompaniesRequest.php:73`

```php
public function getCompanyPayload()
{
    return collect($this->validated())
        ->only(['name', 'vat_id', 'tax_id'])
        ->merge([
            'owner_id' => $this->user()->id,
            'slug' => Str::slug($this->name),  // 自动从公司名称生成 slug
        ])
        ->toArray();
}
```

**文件**: `app/Http/Controllers/V1/Admin/Company/CompaniesController.php:22-24`

```php
$company = Company::create($request->getCompanyPayload());
$company->unique_hash = Hashids::connection(Company::class)->encode($company->id);
$company->save();
```

### 1.3 slug 的唯一性约束

| 约束类型 | 状态 |
|---------|------|
| 数据库唯一索引 | ❌ 不存在 |
| 表单验证唯一 | ✅ 存在（针对 `name` 字段） |
| slug 冲突处理 | ❌ 无 |

**潜在问题**：如果两家公司名称相似（如 "Acme Inc" 和 "Acme Inc."），`Str::slug()` 会生成相同的 slug，导致路由绑定冲突。

---

## 2. `{company:slug}` 路由绑定的实际解析行为

### 2.1 路由定义

**文件**: `routes/web.php:45, routes/api.php:489, routes/web.php:129`

```php
// Web 登录路由
Route::post('/{company:slug}/customer/login', CustomerLoginController::class);

// API 分组
Route::prefix('/{company:slug}/customer')->group(function () { ... });

// 客户门户 SPA 入口
Route::get('{company:slug}/customer/{vue?}', function (Company $company) { ... });
```

### 2.2 绑定机制解析

**⚠️ 重要纠正（v2 错误）**：
- v2 断言："未定义 getRouteKeyName()，路由绑定会失效"
- **实际机制**：Laravel 的路由模型绑定语法 `{company:slug}` 是**显式指定字段**，不依赖模型的 `getRouteKeyName()` 方法

**执行流程**：
```
URL: /acme-inc/customer/invoices
    ↓
路由参数 {company:slug} 捕获 "acme-inc"
    ↓
Laravel 执行: Company::where('slug', 'acme-inc')->firstOrFail()
    ↓
如果找到，注入到控制器的 Company $company 参数
如果找不到，抛出 404
```

**证据链**：
1. 测试代码大量使用 `$customer->company->slug` 构造 URL（如 `tests/Feature/Customer/InvoiceTest.php:29`）
2. 通知邮件中使用 `$notifiable->company->slug` 生成重置密码链接
3. CompanyResource 明确返回 `slug` 字段（`app/Http/Resources/CompanyResource.php:26`）

### 2.3 绑定生效的前置条件

| 条件 | 状态 |
|-----|------|
| companies 表有 slug 字段 | ✅ |
| 所有 Company 记录的 slug 不为 null | ✅（两次迁移回填） |
| slug 字段唯一 | ❌ 无数据库约束 |

---

## 3. `unique_hash` 在客户门户链路中的角色

### 3.1 unique_hash 的生成

**文件**: `app/Http/Controllers/V1/Admin/Company/CompaniesController.php:23`

```php
$company->unique_hash = Hashids::connection(Company::class)->encode($company->id);
```

使用 Hashids 将自增 ID 编码为不透明字符串，目的是**隐藏真实 ID**。

### 3.2 unique_hash 与 slug 的职责分工

| 场景 | 使用字段 | 示例 |
|------|---------|------|
| **客户门户路由** | `slug` | `/acme-inc/customer/invoices` |
| **公共 PDF 链接**（发票） | `unique_hash`（Invoice 模型） | `/invoices/pdf/abc123xyz` |
| **公共 PDF 链接**（报价单） | `unique_hash`（Estimate 模型） | `/estimates/pdf/def456uvw` |
| **公共 PDF 链接**（付款） | `unique_hash`（Payment 模型） | `/payments/pdf/ghi789rst` |
| **报告下载** | `unique_hash`（Company 模型） | 通过 unique_hash 查询公司 |
| **邮件链接** | `slug` | `/{slug}/customer/reset/password/xxx` |

**文件**: `routes/web.php:89-97`

```php
Route::get('/invoices/pdf/{invoice:unique_hash}', InvoicePdfController::class);
Route::get('/estimates/pdf/{estimate:unique_hash}', EstimatePdfController::class);
Route::get('/payments/pdf/{payment:unique_hash}', PaymentPdfController::class);
```

**文件**: `app/Http/Controllers/V1/Admin/Report/*Controller.php`

```php
// 报告下载通过 unique_hash 查找公司
$company = Company::where('unique_hash', $hash)->first();
```

### 3.3 客户门户链路中 unique_hash 的实际作用

**结论**：在客户门户的主链路中，`unique_hash` **几乎不承担任何角色**。

| 客户门户流程 | 使用字段 |
|-------------|---------|
| 访问客户门户 | `slug`（URL 路径） |
| 登录认证 | `slug`（URL 路径） |
| API 调用 | `slug`（URL 路径前缀） |
| 查看发票详情 | `id`（URL 路径） |
| 下载 PDF | Invoice/Estimate/Payment 模型的 `unique_hash` |

**唯一例外**：下载 PDF 时使用的是**业务模型**（Invoice/Estimate/Payment）的 unique_hash，而非 Company 模型的 unique_hash。

---

## 4. 无公司关联用户场景下的边界风险（收敛版）

### 4.1 风险分级标准

| 级别 | 定义 |
|------|------|
| 🔴 确定风险 | 代码路径明确可达，有明确触发条件 |
| 🟡 推测风险 | 理论上可能但缺乏实际触发路径，或有保护机制 |
| 🟢 已规避 | 存在有效保护机制 |

### 4.2 确定风险（🔴）

#### 风险点 1：CompanyMiddleware 空指针
**文件**: `app/Http/Middleware/CompanyMiddleware.php:23`

```php
$request->headers->set('company', $user->companies()->first()->id);
```

**触发条件**：
1. `user_company` 表已存在（Schema 检查通过）
2. 认证用户通过 `auth:sanctum`
3. 用户在 `user_company` 中无任何关联记录

**结果**：`Error: Call to a member function id() on null` → HTTP 500

#### 风险点 2：ScopeBouncer 空指针
**文件**: `app/Http/Middleware/ScopeBouncer.php:37`

```php
$tenantId = $request->header('company')
    ? $request->header('company')
    : $user->companies()->first()->id;
```

**触发条件**：与风险点 1 相同

**结果**：同样的空指针错误

#### 风险点 3：BootstrapController 空指针
**文件**: `app/Http/Controllers/V1/Admin/General/BootstrapController.php:41`

```php
$current_company = $current_user->companies()->first();
// 后续访问 $current_company->id 时崩溃
```

**触发条件**：与风险点 1 相同

**结果**：当返回响应时尝试加载 `current_company` 关系报错

#### 风险点 4：User 模型访问器空指针
**文件**: `app/Models/User.php:96`

```php
public function getFormattedCreatedAtAttribute($value)
{
    $company_id = (CompanySetting::where('company_id', request()->header('company'))->exists())
        ? request()->header('company')
        : $this->companies()->first()->id;
    // ...
}
```

**触发条件**：
1. 访问 `$user->formatted_created_at` 属性
2. 请求头中没有有效 company
3. 用户无公司关联

**结果**：空指针错误

### 4.3 推测风险（🟡）

#### 推测风险 1：安装后首次登录
**文件**: `app/Http/Controllers/V1/Installation/LoginController.php:26`

```php
'company' => $user->companies()->first(),
```

**为何是推测**：
- 安装流程的 `UsersTableSeeder.php` 会**自动创建公司并关联用户**：
  ```php
  $user = User::create([...]);
  $company = Company::create([...]);
  $user->companies()->attach($company->id);
  ```
- 正常安装流程下不会触发

**可能触发场景**：手动在数据库中创建用户后直接调用安装登录接口

#### 推测风险 2：并发删除最后一家公司
**文件**: `app/Http/Controllers/V1/Admin/Company/CompaniesController.php:48`

```php
if ($user->loadCount('companies')->companies_count <= 1) {
    return respondJson('You_cannot_delete_all_companies', ...);
}
```

**为何是推测**：
- 存在检查机制，但检查与删除之间存在时间窗口
- 高并发下可能出现两个请求都通过检查，最后一个删除成功导致用户无公司

**实际概率**：极低，删除公司是低频操作且需验证公司名称

### 4.4 已规避风险（🟢）

#### 已规避 1：删除公司时的数量保护
**代码**: `CompaniesController.php:48`

确保用户至少保留一家公司，常规操作下不会出现无公司用户。

#### 已规避 2：创建用户时自动关联
**文件**: `app/Models/User.php:356`

```php
$user->companies()->sync($companies->pluck('id'));
```

通过 UI 创建的用户会自动关联到指定公司。

### 4.5 风险触发路径总结

```
┌─────────────────────────────────────────────────────────────┐
│ 🔴 确定风险触发路径                                         │
├─────────────────────────────────────────────────────────────┤
│ 1. 直接操作数据库向 users 表插入记录                         │
│ 2. 忘记向 user_company 表插入关联数据                        │
│ 3. 该用户通过 Sanctum 登录获取 API token                    │
│ 4. 使用 token 调用任何需要 auth:sanctum + company 中间件的 API │
│ 5. CompanyMiddleware::handle() 第 23 行崩溃                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 🟡 推测风险触发路径                                         │
├─────────────────────────────────────────────────────────────┤
│ 1. 用户 A 有 2 家公司                                       │
│ 2. 请求 1 和 请求 2 同时到达，都尝试删除不同的公司           │
│ 3. 两个请求都通过 count <= 1 检查（都读到 2）               │
│ 4. 两个请求都执行删除，用户变为 0 家公司                     │
│ 5. 后续请求崩溃                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. 客户门户完整上下文链路（最终版）

### 5.1 客户访问流程

```
客户收到邮件链接: https://example.com/acme-inc/customer/invoices
    ↓
[Web 路由] {company:slug} 匹配 "acme-inc"
    ↓
Laravel 执行: Company::where('slug', 'acme-inc')->firstOrFail()
    ↓
[视图渲染] 注入 Company $company 到闭包
    ↓
读取公司设置: get_company_setting('customer_portal_logo', $company->id)
    ↓
返回 SPA 入口页面
    ↓
[前端] 从 URL path 中提取 slug: window.location.pathname.split('/')[1]
    ↓
[前端] customer globalStore.bootstrap(slug)
    ↓
[API 调用] GET /api/v1/acme-inc/customer/bootstrap
    ↓
[API 路由] {company:slug} 再次解析 Company
    ↓
[控制器] 若已登录，返回客户数据；否则要求登录
    ↓
[前端] 将 slug 存入 globalStore.companySlug
    ↓
[后续 API] 所有请求前缀: /api/v1/${companySlug}/customer/...
```

### 5.2 客户登录流程

**文件**: `app/Http/Controllers/V1/Customer/Auth/LoginController.php:21`

```php
public function __invoke(CustomerLoginRequest $request, Company $company)
{
    $user = Customer::where('email', $request->email)
        ->where('company_id', $company->id)  // 关键：限定在当前公司
        ->first();
    
    // 验证密码...
    
    Auth::guard('customer')->login($user);
    
    return response()->json(['success' => true]);
}
```

**安全特性**：即使邮箱密码正确，如果 Customer 不属于 URL 中的 Company，登录也会失败。

### 5.3 登录后的数据隔离

登录后，客户模型自带 `company_id`：

```php
// Customer 模型
public function company(): BelongsTo
{
    return $this->belongsTo(Company::class);
}

// 控制器中使用
public function index(Company $company)
{
    $invoices = Auth::guard('customer')->user()
        ->invoices()
        ->where('company_id', $company->id)  // 双重保险
        ->paginate();
    
    return InvoiceResource::collection($invoices);
}
```

---

## 6. 关键事实对照表（v3 vs v2 vs v1）

| 问题 | v1 结论 | v2 结论 | v3 最终结论 |
|------|---------|---------|-------------|
| companies 有 slug 字段 | ✅ 未涉及 | ❌ 断言不存在 | ✅ 2021 年 7 月添加，两次迁移回填 |
| {company:slug} 绑定是否工作 | ✅ 未涉及 | ❌ 断言失效 | ✅ 显式指定字段，工作正常 |
| unique_hash 用于客户门户路由 | ✅ 未涉及 | ❌ 混淆用途 | ✅ 仅用于 PDF/报告下载，不用于客户门户主链路 |
| company_settings 外键删除行为 | ❌ 级联删除 | ✅ 手动删除 | ✅ RESTRICT，需通过 deleteCompany() |
| 无公司用户风险数量 | ✅ 未涉及 | 5 个风险点 | 🔴 4 个确定 + 🟡 2 个推测 + 🟢 2 个已规避 |

---

## 7. 架构总结（最终版）

### 7.1 双轨上下文设计

| 维度 | 管理员端 | 客户门户端 |
|------|---------|-----------|
| 上下文传递 | HTTP Header: `company: <id>` | URL Path: `/{slug}/customer/...` |
| 公司标识 | 数字 ID | URL友好的 slug 字符串 |
| 中间件 | CompanyMiddleware + ScopeBouncer | 路由模型绑定隐式解析 |
| 认证 | Sanctum Token | Session Guard (`customer`) |
| 数据隔离 | 通过 bouncer scope | 通过 Customer 模型的 company_id |

### 7.2 slug 与 unique_hash 设计意图对比

| 特性 | slug | unique_hash |
|------|------|-------------|
| 可读性 | 高（acme-inc） | 低（abc123xyz） |
| 可预测性 | 高（从公司名称推导） | 低（Hashids 编码） |
| 用途 | 客户门户路由（需要友好 URL） | PDF/报告链接（需要隐藏真实 ID） |
| 唯一性约束 | 无（潜在冲突风险） | 基于 ID 天然唯一 |
| 修改可能性 | 公司改名时可能变化 | 永久不变 |

### 7.3 遗留问题

| 问题 | 严重程度 | 建议 |
|------|---------|------|
| slug 无唯一索引 | 中 | 增加唯一索引，或在生成时检测冲突 |
| 无公司用户导致 500 | 高 | 在 CompanyMiddleware 增加空值检查，返回 403 |
| 公司设置无缓存 | 中 | 实现基于 company_id 的缓存层 |
| `request()->header('company')` 全局依赖 | 中 | 考虑使用服务容器管理当前公司上下文 |

### 7.4 关键文件速查表（最终版）

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| slug 字段添加迁移 | `database/migrations/2021_07_06_070204_add_owner_id_to_companies_table.php` | 18, 30 |
| slug 二次回填 | `database/migrations/2022_01_06_103536_add_slug_to_companies.php` | 14-20 |
| 新公司 slug 生成 | `app/Http/Requests/CompaniesRequest.php` | 73 |
| unique_hash 生成 | `app/Http/Controllers/V1/Admin/Company/CompaniesController.php` | 23 |
| 客户门户路由定义 | `routes/web.php` | 45, 129 |
| 客户登录控制器 | `app/Http/Controllers/V1/Customer/Auth/LoginController.php` | 21-25 |
| CompanyMiddleware 空指针风险 | `app/Http/Middleware/CompanyMiddleware.php` | 23 |
| 删除公司数量保护 | `app/Http/Controllers/V1/Admin/Company/CompaniesController.php` | 48 |
| 客户全局 Store | `resources/scripts/customer/stores/global.js` | 14, 20-21 |
