# InvoiceShelf 公司上下文分析报告 v5（最终版）

> 修正说明：本版本对中间件优先级、查询构建器语义、越权风险进行了代码级的最终严谨验证。

---

## 1. 中间件优先级与路由绑定顺序（最终严谨版）

### 1.1 全局中间件优先级定义

**代码位置**: `bootstrap/app.php:86-93`

```php
$middleware->priority([
    StartSession::class,              // 1. 启动会话
    ShareErrorsFromSession::class,    // 2. 共享错误
    Authenticate::class,              // 3. 认证（auth 中间件）
    AuthenticateSession::class,       // 4. 会话认证
    SubstituteBindings::class,        // 5. 路由模型绑定（bindings 中间件）
    Authorize::class,                 // 6. 授权
]);
```

**⚠️ 关键结论（代码已证实）**：
- `Authenticate`（第 3 位）**先于** `SubstituteBindings`（第 5 位）
- **认证中间件在路由绑定之前执行**

---

### 1.2 客户门户 API 的中间件执行顺序

**路由定义**: `routes/api.php:489,506`

```php
Route::prefix('/{company:slug}/customer')->group(function () {
    Route::middleware(['auth:customer', 'customer-portal'])->group(function () {
        Route::get('/bootstrap', CustomerBootstrapController::class);
        Route::get('invoices/{id}', [CustomerInvoicesController::class, 'show']);
    });
});
```

**实际执行顺序**（按优先级排序）：

| 顺序 | 中间件 | 行为 | 失败后果 |
|------|--------|------|---------|
| 1 | `auth:customer` | 验证客户会话 | 未登录 → 401/重定向 |
| 2 | `SubstituteBindings`（隐式） | 解析 `{company:slug}` 绑定 Company 模型 | slug 无效 → 404 |
| 3 | `customer-portal` | 检查 `enable_portal` 标志 | 未启用 → 401 |
| 4 | 控制器方法 | 执行业务逻辑 | |

**⚠️ 关键纠正（v4 错误）**：
- v4 断言："路由绑定在中间件之前执行"
- **实际顺序**：`auth:customer` 认证 **先于** `{company:slug}` 绑定
- 影响：未登录的客户访问无效 slug 时，会先收到 401 而不是 404

---

### 1.3 登录接口的特殊情况

**路由**: `routes/web.php:45`

```php
Route::post('/{company:slug}/customer/login', CustomerLoginController::class);
```

**无 `auth:customer` 中间件**，执行顺序：

| 顺序 | 中间件 | 行为 |
|------|--------|------|
| 1 | `SubstituteBindings` | 解析 `{company:slug}` |
| 2 | 控制器 | 执行登录逻辑 |

**行为**：
- slug 无效 → 404（不会到达登录逻辑）
- slug 有效 → 执行登录验证

---

### 1.4 管理员 API 的中间件顺序

**路由**: `routes/api.php:191`

```php
Route::middleware(['auth:sanctum', 'company'])->group(function () {
    Route::middleware(['bouncer'])->group(function () {
        // ...
    });
});
```

**执行顺序**：

| 顺序 | 中间件 | 行为 |
|------|--------|------|
| 1 | `auth:sanctum` | 认证管理员 |
| 2 | `company` | 验证公司归属，回退默认公司 |
| 3 | `SubstituteBindings` | 解析其他路由绑定（如有） |
| 4 | `bouncer` | 设置权限范围 |

**⚠️ 代码已证实**：
- `company` 中间件不在全局优先级列表中，按声明顺序在 `auth:sanctum` 之后执行
- `bouncer` 中间件也不在全局优先级列表中，在 `company` 之后执行

---

## 2. `where("slug", null)` 的真实 SQL 语义

### 2.1 Laravel 查询构建器的 null 处理机制

**Laravel 源码行为**（代码已证实）：
- `where('column', null)` 会被 Laravel 查询构建器**自动转换**为 `whereNull('column')`
- 生成的 SQL 是：`WHERE column IS NULL`
- **不是**：`WHERE column = NULL`（这在 SQL 中永远为 false）

**项目中的使用证据**：
```php
// app/Models/Item.php:120-121
->where('invoice_item_id', null)
->where('estimate_item_id', null);

// app/Services/SerialNumberFormatter.php:124
->where('sequence_number', '<>', null)
```

这些代码正常工作，证明 Laravel 正确处理了 null 值。

---

### 2.2 本项目中的 slug NULL 场景

**迁移回填**: `database/migrations/2022_01_06_103536_add_slug_to_companies.php:14`

```php
$companies = Company::where('slug', null)->get();
```

**实际生成的 SQL**：
```sql
SELECT * FROM companies WHERE slug IS NULL
```

**⚠️ 关键纠正（v4 错误）**：
- v4 断言："`WHERE slug = NULL` 在 SQL 中永远为 false"
- **实际行为**：Laravel 自动转换为 `WHERE slug IS NULL`，能正确查询到 slug 为 NULL 的记录
- 两次迁移能正常工作正是依赖这个机制

---

### 2.3 slug 缺失时的真实行为

| 场景 | SQL 行为 | 结果 | 代码已证实 |
|------|---------|------|-----------|
| `Company::where('slug', null)->get()` | `WHERE slug IS NULL` | 返回 slug 为 NULL 的公司 | ✅ |
| 路由绑定 `{company:slug}` 值为 "null" | `WHERE slug = 'null'` | 查找 slug 等于字符串 "null" 的公司 | ✅ |
| 路由绑定 `{company:slug}` 值为 "acme" 但数据库 slug 为 NULL | `WHERE slug = 'acme'` | 0 条结果 → 404 | ✅ |

**结论**：
- 数据库中 slug 为 NULL 的公司**无法通过任何 slug 访问**（因为 URL 中的 slug 是字符串）
- 但查询构建器的 `where('slug', null)` **能正确工作**，用于数据迁移

---

## 3. 潜在越权风险的严谨收敛（基于 `whereCustomer` 约束）

### 3.1 show 方法的通用查询模式

**所有 Customer 控制器的 show 方法遵循相同模式**：

```php
public function show(Company $company, $id)
{
    $model = $company->{$modelName}()           // 1. 从绑定的 company 关联查询
        ->whereCustomer(Auth::guard('customer')->id())  // 2. 限定当前客户
        ->where('id', $id)                        // 3. 限定资源 ID
        ->first();
    
    if (! $model) {
        return response()->json(['error' => 'model_not_found'], 404);
    }
    
    return new ModelResource($model);
}
```

**关键约束**:
1. `$company->{$modelName}()` - 通过关联查询，自动添加 `WHERE company_id = ?`
2. `->whereCustomer(Auth::guard('customer')->id())` - 添加 `WHERE customer_id = ?`
3. `->where('id', $id)` - 添加 `WHERE id = ?`

---

### 3.2 `whereCustomer` 作用域定义

**代码位置**（统一模式）：
```php
// app/Models/Invoice.php:312
public function scopeWhereCustomer($query, $customer_id)
{
    $query->where('invoices.customer_id', $customer_id);
}

// app/Models/Payment.php:372
public function scopeWhereCustomer($query, $customer_id)
{
    $query->where('payments.customer_id', $customer_id);
}

// app/Models/Estimate.php:205
public function scopeWhereCustomer($query, $customer_id)
{
    $query->where('estimates.customer_id', $customer_id);
}
```

**代码已证实**：所有业务模型都有 `customer_id` 字段，`whereCustomer` 直接过滤该字段。

---

### 3.3 三种结果的严谨区分

基于 slug 重复场景下的查询行为，可能出现三种结果：

#### 🔴 结果 1：误路由（Wrong Company, Wrong Data）

**场景**：客户 C 属于公司 B（id=2），但访问了公司 A（id=1）的 slug

**查询条件**：
```sql
WHERE invoices.company_id = 1      -- 绑定的公司 A
  AND invoices.customer_id = C_id   -- 当前客户 C
  AND invoices.id = 123             -- 请求的发票 ID
```

**结果**：
- 客户 C 不属于公司 A，`customer_id` 不匹配
- 返回 404 `payment_not_found`（或对应模型的 not found）
- **没有越权**，因为客户看不到任何数据

**代码已证实**：`whereCustomer` 约束确保了这一点。

---

#### 🟡 结果 2：数据不可达（Correct Company, Inaccessible ID）

**场景**：客户 C 属于公司 A（id=1），访问公司 A 的 slug，但请求了不属于自己的发票 ID

**查询条件**：
```sql
WHERE invoices.company_id = 1      -- 正确的公司 A
  AND invoices.customer_id = C_id   -- 当前客户 C
  AND invoices.id = 999             -- 发票属于公司 A 的另一个客户
```

**结果**：
- `customer_id` 不匹配
- 返回 404
- **没有越权**

**代码已证实**：这是标准的资源所有者检查。

---

#### 🟢 结果 3：正常访问（Correct Company, Correct Data）

**场景**：客户 C 属于公司 A，访问公司 A 的 slug，请求属于自己的发票 ID

**查询条件**：
```sql
WHERE invoices.company_id = 1      -- 正确的公司 A
  AND invoices.customer_id = C_id   -- 当前客户 C
  AND invoices.id = 123             -- 发票属于客户 C
```

**结果**：
- 所有条件匹配
- 返回发票数据
- **正常行为**

---

### 3.4 越权可能性的最终结论

| 越权场景 | 是否可能 | 原因 | 代码已证实 |
|---------|---------|------|-----------|
| 通过错误 slug 看到其他公司的数据 | ❌ 不可能 | `whereCustomer` 约束过滤了 customer_id | ✅ |
| 通过错误 slug 看到同公司其他客户的数据 | ❌ 不可能 | `whereCustomer` 约束过滤了 customer_id | ✅ |
| slug 重复导致客户无法登录 | ✅ 可能 | 只能登录第一条匹配的公司 | ✅ |
| slug 重复导致客户无法访问自己的数据 | ✅ 可能 | 访问错误 slug 时 404 | ✅ |

**⚠️ 关键纠正（v4 错误）**：
- v4 断言："可能通过访问错误的 slug 看到另一家公司的同名 payment id"
- **实际情况**：`whereCustomer` 约束确保了客户只能看到自己的数据，即使 company_id 错误
- 实际风险是**可用性问题**（无法访问），而非**保密性问题**（越权查看）

---

## 4. 极端场景下的行为总表（最终版）

### 4.1 Slug 异常场景

| 场景 | Web 入口 | 登录接口 | API（已登录） |
|------|---------|---------|--------------|
| Slug 不存在 | 404 | 404 | 401（auth 先失败） |
| Slug 为 NULL（数据库） | 404 | 404 | 401 |
| Slug 重复（访问后创建的公司） | 404（绑定到先创建的） | 404 | 401 |
| Slug 重复（访问先创建的公司） | 正常 | 正常 | 正常 |

### 4.2 无公司关联用户场景

| 调用点 | 是否必然崩溃 | 触发条件 | 代码位置 |
|--------|-------------|---------|---------|
| `CompanyMiddleware.php:23` | ✅ 是 | 经过 auth + company 中间件 | 第 23 行 |
| `ScopeBouncer.php:37` | ✅ 是 | 经过 auth + bouncer 中间件 | 第 37 行 |
| `BootstrapController.php:41` | ✅ 是 | 调用 /api/v1/bootstrap | 第 41 行 |
| `User.php:96`（访问器） | ❌ 否 | 访问 formatted_created_at + header 无效 | 第 96 行 |
| `Installation/LoginController.php:26` | ❌ 否 | 手动创建 super admin 跳过 seed | 第 26 行 |

---

## 5. 结论可信度对照表

| 结论 | 代码已证实 | 仍需假设 |
|------|-----------|---------|
| auth:customer 先于路由绑定执行 | ✅（priority 列表第 3 位 vs 第 5 位） | |
| where('slug', null) 转换为 WHERE slug IS NULL | ✅（Laravel 查询构建器行为 + 项目代码证据） | |
| 登录接口无 auth 中间件，绑定先执行 | ✅（路由定义无 middleware） | |
| show 方法的 whereCustomer 约束阻止越权 | ✅（5 个控制器统一模式） | |
| slug 重复导致可用性问题而非保密性问题 | ✅（双重约束：company_id + customer_id） | |
| SubstituteBindings 的具体实现细节 | | ✅（依赖 Laravel 框架源码） |
| 异常处理对 404/500 的具体转换 | | ✅（依赖 Handler 配置） |
| 多数据库驱动下的 null 处理一致性 | | ✅（理论上统一，但未测试所有驱动） |

---

## 6. 最终架构风险清单

| 风险类型 | 具体描述 | 严重程度 | 可利用性 |
|---------|---------|---------|---------|
| 🔴 崩溃风险 | 无公司用户访问管理员 API → HTTP 500 | 高 | 低（需数据库操作失误） |
| 🟡 可用性风险 | slug 重复导致部分公司客户无法登录 | 中 | 中（需创建重名公司） |
| 🟡 可用性风险 | slug 改名导致所有客户书签/历史邮件失效 | 高 | 低（需主动改名操作） |
| 🟢 无越权风险 | show 方法的双重约束（company_id + customer_id）阻止跨客户数据访问 | - | - |
| 🟢 无绑定顺序风险 | 全局优先级确保认证在绑定前执行 | - | - |
