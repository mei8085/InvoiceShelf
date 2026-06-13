# 支出分类聚合链路代码分析

> 分析支出记录如何按分类聚合，以及公司隔离、筛选条件、金额口径三个因素分别如何影响分类列表、支出报表、损益报表和仪表盘四个统计视图的数值。

---

## 一、三个核心影响因素概览

| 因素 | 实现位置 | 作用 |
|------|---------|------|
| **公司隔离** | `scopeWhereCompany()` / `scopeWhereCompanyId()` | 确保数据仅返回当前公司 |
| **日期与分类筛选** | `scopeApplyFilters()` + `scopeExpensesBetween()` 等 | 按时间范围或特定分类过滤 |
| **原始金额 vs 基准金额** | `amount` vs `base_amount` | 统一/保留多币种下的货币口径 |

---

## 二、三个核心影响因素的代码实现

### 2.1 公司隔离

系统中存在两套公司隔离 scope，其选择取决于调用上下文（API 路由 vs Web 报表路由）。

#### scopeWhereCompany（隐式，从请求头读取）

```php
// Expense.php#L206-L209
public function scopeWhereCompany($query)
{
    $query->where('expenses.company_id', request()->header('company'));
}
```

- 依赖 `request()->header('company')`，由 `CompanyMiddleware` 在请求进入时注入
- 应用于所有通过 `auth:sanctum + company` 中间件组的 API 路由

```php
// CompanyMiddleware.php#L17-L28
public function handle(Request $request, Closure $next): Response
{
    if (Schema::hasTable('user_company')) {
        $user = $request->user();
        if ((! $request->header('company')) || (! $user->hasCompany($request->header('company')))) {
            $request->headers->set('company', $user->companies()->first()->id);
        }
    }
    return $next($request);
}
```

中间件从当前登录用户的 `user_company` 关联中自动选取第一个公司，填充到请求头。

#### scopeWhereCompanyId（显式，传入参数）

```php
// Expense.php#L211-L214
public function scopeWhereCompanyId($query, $company)
{
    $query->where('expenses.company_id', $company);
}
```

- 需要显式传入公司 ID 参数
- 用于 Web 路由（PDF 报表），因为这类请求不携带 `company` 请求头

#### 路由组对比

| 路由类型 | 文件 | 中间件 | 公司隔离方式 |
|---------|------|--------|------------|
| API 路由（中间件组内） | [api.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/routes/api.php) | `auth:sanctum` + `company` | `scopeWhereCompany()` — 读 `header('company')` |
| Web 路由（PDF 报表） | [web.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/routes/web.php) | `auth:sanctum` | `scopeWhereCompanyId($company->id)` — 显式传参 |

---

### 2.2 日期与分类筛选

所有筛选条件通过 `scopeApplyFilters()` 集中处理。

```php
// Expense.php#L153-L184
public function scopeApplyFilters($query, array $filters)
{
    $filters = collect($filters);

    if ($filters->get('expense_category_id')) {
        $query->whereCategory($filters->get('expense_category_id')); // 按分类 ID 精确过滤
    }

    if ($filters->get('customer_id')) {
        $query->whereUser($filters->get('customer_id'));
    }

    if ($filters->get('expense_id')) {
        $query->whereExpense($filters->get('expense_id'));
    }

    if ($filters->get('from_date') && $filters->get('to_date')) {
        $start = Carbon::createFromFormat('Y-m-d', $filters->get('from_date'));
        $end = Carbon::createFromFormat('Y-m-d', $filters->get('to_date'));
        $query->expensesBetween($start, $end); // 过滤日期范围
    }

    if ($filters->get('search')) {
        $query->whereSearch($filters->get('search')); // 按分类名称模糊搜索
    }
}
```

**日期范围过滤**的核心实现：

```php
// Expense.php#L121-L127
public function scopeExpensesBetween($query, $start, $end)
{
    return $query->whereBetween(
        'expenses.expense_date',
        [$start->format('Y-m-d'), $end->format('Y-m-d')]
    );
}
```

> 注意：`expense_date` 是 `expenses` 表的字段，用于标识这笔支出发生在哪一天，与 `created_at` 不同。报表的日期过滤基于 `expense_date` 而非创建时间。

---

### 2.3 原始金额（amount）vs 基准金额（base_amount）

#### 原始金额 amount

- 记录在 `expenses.amount` 字段
- 即用户在创建支出时输入的原始币种金额
- 不同支出可能使用不同币种

#### 基准金额 base_amount

- 由 [ExpenseRequest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Requests/ExpenseRequest.php#L74-L89) 在保存时计算：

```php
public function getExpensePayload()
{
    $company_currency = CompanySetting::getSetting('currency', $this->header('company'));
    $current_currency = $this->currency_id;
    $exchange_rate = $company_currency != $current_currency
        ? $this->exchange_rate
        : 1;

    return collect($this->validated())->merge([
        'creator_id' => $this->user()->id,
        'company_id' => $this->header('company'),
        'exchange_rate' => $exchange_rate,
        'base_amount' => $this->amount * $exchange_rate, // 统一转换为基准货币
        'currency_id' => $current_currency,
    ])->toArray();
}
```

- `base_amount = amount × exchange_rate`
- 即将所有金额统一转换到公司的基准货币
- **只有 `base_amount` 才适合跨币种求和**

#### 使用场景对照

| 字段 | 使用场景 | 原因 |
|------|---------|------|
| `base_amount` | Dashboard 总量、损益报表分组汇总、支出报表总金额 | 需要跨币种相加，必须统一货币口径 |
| `amount` | `ExpenseCategory.getAmountAttribute()` 分类列表的 amount | 仅展示原始币种金额，无跨币种求和需求 |

---

## 三、四个统计视图的链路分析

### 3.1 分类列表（ExpenseCategoriesController）

**涉及文件**：
- [ExpenseCategoriesController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Expense/ExpenseCategoriesController.php)
- [ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php)
- [ExpenseCategoryResource.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Resources/ExpenseCategoryResource.php)

**完整调用链**：

```
GET /api/v1/categories
  → ExpenseCategoriesController::index()
    → ExpenseCategory::applyFilters()       # 分类名搜索
    → ExpenseCategory::whereCompany()        # 公司隔离（读 header）
    → paginateData()
      → ExpenseCategoryResource::collection()
        → each resource: $category->amount   # 访问器动态计算
```

**三因素影响分析**：

| 因素 | 对分类列表的影响 |
|------|----------------|
| **公司隔离** | `ExpenseCategory::whereCompany()` 基于 `request()->header('company')` 过滤，公司 A 看不到公司 B 的分类 |
| **日期筛选** | ❌ 不支持。分类列表没有日期过滤参数，返回所有分类 |
| **分类筛选** | `ExpenseCategory::applyFilters()` 支持按 `category_id` 精确过滤或 `search` 模糊搜索分类名 |
| **金额口径** | `$category->amount` 访问器使用 `sum('amount')`（原始金额），而非 `base_amount`。多币种场景下，同一分类下多笔不同币种的支出，其 amount 累加值无法直接相加比较 |

```php
// ExpenseCategory.php#L41-L44
public function getAmountAttribute()
{
    return $this->expenses()->sum('amount'); // ⚠️ 使用原始金额，非基准金额
}
```

> ⚠️ **潜在问题**：如果某分类下有多笔美元支出和一笔人民币支出，`amount` 字段累加的是原始数值（100 USD + 50 USD + 200 CNY = 350），这个数字在实际业务中没有意义。应当使用 `base_amount` 才能正确反映该分类的支出总额。

---

### 3.2 支出报表（ExpensesReportController）

**涉及文件**：
- [ExpensesReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php)
- [ExpensesReport.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/reports/ExpensesReport.vue)
- [expenses.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/views/app/pdf/reports/expenses.blade.php)

**完整调用链**：

```
ExpensesReport.vue (iframe 加载)
  → GET /reports/expenses/{hash}?from_date=...&to_date=...
    → ExpensesReportController::__invoke($request, $hash)
      → Company::where('unique_hash', $hash)->first()      # 从 URL hash 获取公司
      → Expense::with('category')                           # 预加载分类
        → applyFilters(['from_date', 'to_date', 'expense_category_id'])  # 日期 + 分类筛选
        → whereCompanyId($company->id)                      # 公司隔离（显式 ID）
        → orderBy('expense_date', 'asc')
        → get() → Collection::groupBy('category.name')       # 内存中按分类名分组
      → 计算分组小计和总金额
      → domPDF 渲染 expenses.blade.php
```

**关键实现**：

```php
// ExpensesReportController.php#L36-L55
$expenses = Expense::with('category')
    ->whereCompanyId($company->id)
    ->applyFilters($request->only(['from_date', 'to_date', 'expense_category_id']))
    ->orderBy('expense_date', 'asc')
    ->get();

$totalAmount = $expenses->sum('base_amount'); // 使用基准金额

$grouped = $expenses->groupBy(function ($item) {
    return $item->category ? $item->category->name : trans('expenses.uncategorized');
});

$expenseGroups = collect();
foreach ($grouped as $categoryName => $group) {
    $expenseGroups->push([
        'name' => $categoryName,
        'expenses' => $group,
        'total' => $group->sum('base_amount'), // 每组小计使用基准金额
    ]);
}
```

**三因素影响分析**：

| 因素 | 对支出报表的影响 |
|------|----------------|
| **公司隔离** | `whereCompanyId($company->id)` 基于 URL 中的 `unique_hash` 解析出公司 ID 并过滤。公司 A 的 hash 只能查到 A 的数据 |
| **日期筛选** | ✅ 支持。`applyFilters()` 从请求中提取 `from_date` + `to_date`，调用 `scopeExpensesBetween()` 按 `expense_date` 字段过滤。只有落在日期范围内的支出才参与分组 |
| **分类筛选** | ✅ 支持。`expense_category_id` 参数传入后，`whereCategory()` 精确过滤到指定分类，全部分组结果只有这一个分类 |
| **金额口径** | `$expenses->sum('base_amount')` 和 `$group->sum('base_amount')` 全部使用基准金额。同一分类下不同币种的支出，按汇率转换后统一相加，数值正确 |

**PDF 展示结构**（每组下方逐条列出明细）：

```php
// expenses.blade.php#L238-L258
@foreach ($expenseGroups as $group)
    <p class="expense-title">{{ $group['name'] }}</p>   <!-- 分类名 -->
    @foreach ($group['expenses'] as $expense)
        <!-- 每笔明细：日期、备注、金额 -->
    @endforeach
    <p>小计：{!! format_money_pdf($group['total'], $currency) !!}</p> <!-- 基准金额 -->
@endforeach
```

---

### 3.3 损益报表（ProfitLossReportController）

**涉及文件**：
- [ProfitLossReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Report/ProfitLossReportController.php)
- [profit-loss.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/views/app/pdf/reports/profit-loss.blade.php)

**完整调用链**：

```
GET /reports/profit-loss/{hash}?from_date=...&to_date=...
  → ProfitLossReportController::__invoke($request, $hash)
    → Company::where('unique_hash', $hash)->first()   # URL hash 获取公司
    → Payment::whereCompanyId() → applyFilters() → sum('base_amount')   # 收入合计
    → Expense::with('category')
      → applyFilters(['from_date', 'to_date'])
      → whereCompanyId($company->id)
      → expensesAttributes()                            # SQL 层 GROUP BY
      → get()
    → 遍历分组累加总支出
    → 计算净利润
    → domPDF 渲染 profit-loss.blade.php
```

**关键实现**：

```php
// ProfitLossReportController.php#L35-L48
$paymentsAmount = Payment::whereCompanyId($company->id)
    ->applyFilters($request->only(['from_date', 'to_date']))
    ->sum('base_amount');

$expenseCategories = Expense::with('category')
    ->whereCompanyId($company->id)
    ->applyFilters($request->only(['from_date', 'to_date']))
    ->expensesAttributes()  // SQL: SUM + COUNT + GROUP BY
    ->get();

$totalAmount = 0;
foreach ($expenseCategories as $category) {
    $totalAmount += $category->total_amount; // base_amount 之和
}
```

`expensesAttributes()` scope 的 SQL 等价于：

```sql
SELECT
    expense_category_id,
    COUNT(*)          AS expenses_count,
    SUM(base_amount)  AS total_amount
FROM expenses
WHERE company_id = ?
  AND expense_date BETWEEN ? AND ?
GROUP BY expense_category_id
```

**三因素影响分析**：

| 因素 | 对损益报表的影响 |
|------|----------------|
| **公司隔离** | `whereCompanyId($company->id)` 与支出报表相同机制，确保只有当前公司的收支数据参与汇总 |
| **日期筛选** | ✅ 支持。`applyFilters()` 接收 `from_date` 和 `to_date`，过滤 `expense_date` 范围。收入侧（`payments`）和支出侧使用同一套日期过滤 |
| **分类筛选** | ❌ 不支持。损益报表按日期范围展示全部分类聚合结果，不支持单分类过滤 |
| **金额口径** | ✅ 严格使用 `base_amount`。`expensesAttributes()` 中的 `SUM(base_amount)` 直接在 SQL 层完成多币种统一转换。每行结果 `total_amount` 是该分类在基准货币下的总支出 |

**PDF 展示结构**：

```php
// profit-loss.blade.php#L199-L212
@foreach ($expenseCategories as $expenseCategory)
    <tr>
        <td>{{ $expenseCategory->category->name }}</td>
        <td>{!! format_money_pdf($expenseCategory->total_amount, $currency) !!}</td>
    </tr>
@endforeach
```

---

### 3.4 仪表盘总额（DashboardController）

**涉及文件**：
- [DashboardController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Dashboard/DashboardController.php)
- [dashboard.js](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/stores/dashboard.js)
- [DashboardChart.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/dashboard/DashboardChart.vue)

**完整调用链**：

```
Dashboard.vue → mounted
  → dashboard.js store.loadData()
    → GET /api/v1/dashboard (可选 ?previous_year=true)
      → DashboardController::__invoke($request)
        → Company::find(request()->header('company'))   # 从 header 获取公司
        → 计算财年起始月份
        → 循环 12 次，每次:
            → Expense::whereBetween('expense_date', [$start, $end])
              → whereCompany()  # 读 header
              → sum('base_amount')
            → 其他指标同理...
        → 财年总计 sum('base_amount')
        → 返回 JSON: chart_data.expense_totals[], total_expenses
      → 前端渲染折线图 + 右侧汇总数字
```

**关键实现**：

```php
// DashboardController.php#L62-L100
while ($monthCounter < 12) {
    // 每月支出
    array_push(
        $expense_totals,
        Expense::whereBetween('expense_date', [$start->format('Y-m-d'), $end->format('Y-m-d')])
            ->whereCompany()  // request()->header('company')
            ->sum('base_amount')
    );
    // 每月收入、回款、净收入同理...
    $start->addMonth();
    $end->addMonth()->endOfMonth();
}

// 财年总支出
$total_expenses = Expense::whereBetween('expense_date', [$startDate->format('Y-m-d'), $start->format('Y-m-d')])
    ->whereCompany()
    ->sum('base_amount');
```

**三因素影响分析**：

| 因素 | 对仪表盘的影响 |
|------|--------------|
| **公司隔离** | `whereCompany()` 读取 `request()->header('company')`，由 `CompanyMiddleware` 注入。公司 A 的用户只看到 A 的数据 |
| **日期筛选** | ✅ 支持。仪表盘使用财年（fiscal year）自动计算起始月份，支持切换"本年/去年"（通过 `?previous_year=true` 参数）。日期过滤基于 `expense_date` |
| **分类筛选** | ❌ 不支持。仪表盘只输出总支出和月度趋势，**不做分类维度的聚合**。所有支出按日期汇总，不区分分类 |
| **金额口径** | ✅ 使用 `base_amount`。所有月份和年度总计均使用基准货币金额，确保数值可加总。`total_net_income = total_receipts - total_expenses` 同样基于统一口径 |

---

## 四、四视图对比总览

| | 分类列表 | 支出报表 | 损益报表 | 仪表盘 |
|---|---------|---------|---------|-------|
| **公司隔离方式** | `whereCompany()` (header) | `whereCompanyId($id)` (URL hash) | `whereCompanyId($id)` (URL hash) | `whereCompany()` (header) |
| **日期筛选** | ❌ 不支持 | ✅ `expense_date` 范围 | ✅ `expense_date` 范围 | ✅ 财年自动范围 + 去年切换 |
| **分类筛选** | ✅ 分类名搜索 + ID | ✅ 单分类 ID 精确过滤 | ❌ 不支持 | ❌ 不支持 |
| **金额口径** | ❌ `amount`（原始金额） | ✅ `base_amount` | ✅ `base_amount` | ✅ `base_amount` |
| **聚合方式** | 访问器动态 sum | Collection groupBy | SQL GROUP BY | 循环 12 次 sum |
| **输出内容** | 分类名 + 金额 | 分类名 + 每笔明细 + 分组小计 + 总计 | 各分类合计 + 收入 + 净利润 | 月度趋势图 + 财年总计 |
| **路由类型** | API (`/api/v1/categories`) | Web PDF (`/reports/expenses/{hash}`) | Web PDF (`/reports/profit-loss/{hash}`) | API (`/api/v1/dashboard`) |

---

## 五、金额口径不一致问题

`ExpenseCategory.getAmountAttribute()` 使用 `sum('amount')`，与其他三处统一使用 `base_amount` 的做法不一致。这会导致：

1. **分类列表的金额与其他报表的同名分类金额不匹配**（多币种场景下）
2. 分类列表的金额不具备跨币种可加性

建议将 `ExpenseCategory.php` 第 43 行修改为：

```php
public function getAmountAttribute()
{
    return $this->expenses()->sum('base_amount');
}
```

---

## 六、三个深入问题的代码级解析

### 6.1 同公司下同名分类：支出报表 vs 损益报表为何汇总结果不同

#### 为什么允许同名分类存在

首先需要确认：系统**并未在数据库层或验证层禁止同公司下创建同名分类**。

[ExpenseCategoryRequest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Requests/ExpenseCategoryRequest.php#L20-L30) 中的验证规则：

```php
public function rules(): array
{
    return [
        'name' => ['required'],          // ❌ 没有 unique:expense_categories,name,company_id 约束
        'description' => ['nullable'],
    ];
}
```

- 数据库迁移 `create_ expense_categories_table.php` 也未对 `(company_id, name)` 建立唯一索引
- 因此同一公司下可以存在两条乃至多条 `name` 相同但 `id` 不同的分类记录

#### 同名分类场景示例

假设公司 A 下存在：

| id | name        | company_id |
|----|-------------|------------|
| 3  | 办公用品    | 1          |
| 7  | 办公用品    | 1          |

并存在四笔支出：

| id | expense_category_id | base_amount | expense_date |
|----|---------------------|-------------|--------------|
| E1 | 3                   | 100         | 2024-01-05   |
| E2 | 3                   | 200         | 2024-01-08   |
| E3 | 7                   | 50          | 2024-01-10   |
| E4 | 7                   | 150         | 2024-01-12   |

---

#### 支出报表的聚合结果（按名称合并）

支出报表在 [ExpensesReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php#L246-L248) 使用 Collection `groupBy()`，键值是**分类名称字符串**：

```php
$grouped = $expenses->groupBy(function ($item) {
    return $item->category ? $item->category->name : trans('expenses.uncategorized');
});
```

对于上面的示例数据：
- E1、E2、E3、E4 的分类 `name` 均为 `"办公用品"`
- Laravel Collection 将它们合并到同一分组
- 结果：**只有 1 个分组 `"办公用品"`，小计 = 100 + 200 + 50 + 150 = 500**

PDF 渲染（[expenses.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/views/app/pdf/reports/expenses.blade.php#L238-L258)）将只显示一个 "办公用品" 标题，其下按日期列出全部四笔支出。

---

#### 损益报表的聚合结果（按 ID 分开）

损益报表通过 `expensesAttributes()` scope 执行 **SQL 层 `GROUP BY expense_category_id`**：

```php
// Expense.php#L225-L234
public function scopeExpensesAttributes($query)
{
    $query->select(
        DB::raw('count(*) as expenses_count, sum(base_amount) as total_amount, expense_category_id')
    )->groupBy('expense_category_id');
}
```

对于上面的示例数据：
- E1、E2 → `expense_category_id = 3` → 行数 2，`total_amount = 300`
- E3、E4 → `expense_category_id = 7` → 行数 2，`total_amount = 200`

SQL 返回两条聚合记录：

| expense_category_id | expenses_count | total_amount |
|---------------------|----------------|--------------|
| 3                   | 2              | 300          |
| 7                   | 2              | 200          |

然后在 [profit-loss.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/views/app/pdf/reports/profit-loss.blade.php#L199-L212) 中通过 `$expenseCategory->category->name` 显示名称：

```php
@foreach ($expenseCategories as $expenseCategory)
    <tr>
        <td>{{ $expenseCategory->category->name }}</td>    <!-- 两条都显示 "办公用品" -->
        <td>{!! format_money_pdf($expenseCategory->total_amount, $currency) !!}</td>
    </tr>
@endforeach
```

- PDF 中会出现 **两行同名的 "办公用品"**，分别显示 300 和 200
- 但两个分类的总金额合计仍然是 500，与支出报表总计一致

---

#### 差异总结

| 维度 | 支出报表（Collection groupBy） | 损益报表（SQL GROUP BY） |
|------|-------------------------------|-------------------------|
| 分组键 | `category.name`（字符串） | `expense_category_id`（整数） |
| 同名分类行为 | **合并**为一个分组 | **拆开**为多个行 |
| 分类展示 | 分组标题显示名称，每笔明细列在下方 | 表格每行一个分类，名称重复出现 |
| 总计一致性 | 各组小计之和 = 500 | 各行 total_amount 之和 = 500（合计一致） |

> 💡 **根因**：支出报表侧重"按类别浏览每笔支出明细"，以用户可读的名称分组；损益报表侧重"财务维度的分类汇总"，严格按数据库主键分组以保证审计可追溯性。

---

### 6.2 分类管理页是否消费了分类金额字段

#### 前端分类管理页的列定义

[ExpenseCategorySetting.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/settings/ExpenseCategorySetting.vue#L62-L84) 中表格列的完整定义：

```js
const ExpenseCategoryColumns = computed(() => {
  return [
    {
      key: 'name',
      label: t('settings.expense_category.category_name'),
      // ...
    },
    {
      key: 'description',
      label: t('settings.expense_category.category_description'),
      // ...
    },
    {
      key: 'actions',
      label: '',
      sortable: false,
    },
  ]
})
```

**只有三列：分类名、描述、操作按钮**。没有 `amount` 列。

#### 后端 API 是否仍然返回 amount

虽然前端不展示，但 [ExpenseCategoryResource.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Resources/ExpenseCategoryResource.php#L15-L28) 始终包含 `amount` 字段：

```php
public function toArray($request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'description' => $this->description,
        'company_id' => $this->company_id,
        'amount' => $this->amount,           // ✅ 始终返回
        'formatted_created_at' => $this->formattedCreatedAt,
    ];
}
```

而 `$this->amount` 是 [ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php#L41-L44) 中的访问器：

```php
public function getAmountAttribute()
{
    return $this->expenses()->sum('amount');
}
```

#### 结论：存在 N+1 查询性能浪费

| 环节 | 是否消费 amount | 结论 |
|------|----------------|------|
| 前端分类管理表格 `ExpenseCategorySetting.vue` | ❌ 不渲染，`columns` 中无 amount 列 | 展示层面完全不需要 |
| 前端 category store `fetchCategories()` | ❌ 只把 `response.data.data` 赋值给 `this.categories`，未读取 `amount` | 数据层面也未消费 |
| 后端 Resource | ✅ 每次序列化都访问 `$this->amount` | 触发 N+1 查询 |
| 后端 `getAmountAttribute()` | ✅ 对每个分类执行一次 `SELECT SUM(amount) FROM expenses WHERE expense_category_id = ?` | 分页返回 5 条分类就执行 5 次额外查询 |

**因此分类管理页虽然没有消费 `amount`，但后端仍在无谓地执行 N+1 的聚合查询，是可优化点。**

---

### 6.3 公司隔离：请求参数 vs 当前公司上下文，谁真正决定结果

公司隔离涉及两套机制和三个数据来源，需要逐层分析优先级。

#### 来源一：当前登录用户的公司列表（最顶层权限）

用户属于哪些公司由 `user_company` 关联表决定，`$user->companies()` 返回该用户有权访问的所有公司。

#### 来源二：请求头 `header('company')`（API 路由）

由 [CompanyMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Middleware/CompanyMiddleware.php#L17-L28) 处理：

```php
public function handle(Request $request, Closure $next): Response
{
    if (Schema::hasTable('user_company')) {
        $user = $request->user();

        if ((! $request->header('company')) || (! $user->hasCompany($request->header('company')))) {
            $request->headers->set('company', $user->companies()->first()->id);
        }
    }
    return $next($request);
}
```

**决策逻辑（按优先级）**：

| 条件 | 最终 header('company') 的值 |
|------|---------------------------|
| 前端传入了合法的 `company` header 且用户拥有该公司 | ✅ **使用前端传入值** |
| 前端未传 header，或传入了用户无权访问的公司 ID | ❌ **被中间件覆盖为该用户的第一个公司** |

也就是说：前端请求参数（header）在"用户有权访问该公司"的前提下生效，否则回退到默认公司。

#### 来源三：URL 路径中的 `unique_hash`（Web PDF 路由）

PDF 报表路由定义在 [web.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/routes/web.php#L54-L75)：

```php
Route::middleware('auth:sanctum')->prefix('reports')->group(function () {
    Route::get('/expenses/{hash}', ExpensesReportController::class);
    Route::get('/profit-loss/{hash}', ProfitLossReportController::class);
    // ...
});
```

注意 `reports` 路由组只挂了 `auth:sanctum`，**没有 `company` 中间件**，因此 `request()->header('company')` 在这些请求中可能根本不存在。

对应控制器（以 [ExpensesReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php#L25-L29) 为例）：

```php
public function __invoke(Request $request, $hash)
{
    $company = Company::where('unique_hash', $hash)->first();

    $this->authorize('view report', $company);  // 策略授权
    // ...
}
```

`$this->authorize('view report', $company)` 会调用 [ReportPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Policies/ReportPolicy.php) 验证当前登录用户是否有权查看这个公司的报表。

**决策逻辑（按优先级）**：

| 条件 | 最终生效的公司 |
|------|-------------|
| URL hash 解析出的 company，且用户通过 `view report` 策略授权 | ✅ **使用 URL 中的 company** |
| URL hash 解析出的 company，用户无权访问 | ❌ 授权失败，抛出 403 |

`header('company')` 在 Web PDF 路由中**完全不参与裁决**，即使传了也被忽略，因为控制器显式用 `whereCompanyId($company->id)` 过滤。

---

#### 分类列表的决策链（API 路由示例）

```
GET /api/v1/categories
  │
  ├─ CompanyMiddleware
  │    ├─ 读取 header('company')
  │    └─ 如果为空或越权 → 回退为用户 companies()->first()
  │
  ├─ ExpenseCategoriesController::index()
  │    └─ ExpenseCategory::whereCompany()
  │         └─ 读取已被中间件"校正"后的 header('company')
  │
  └─ 返回：仅该 company_id 下的分类
```

谁最终决定结果？**是 `CompanyMiddleware` 校正后的 header 值**，前端原始 header 只有在合法且不越权时才能生效。

---

#### 四种视图的最终裁决者汇总

| 视图 | 路由类型 | 公司信息来源 | 最终裁决者 | 越权处理 |
|------|---------|------------|----------|---------|
| 分类列表 | API | `header('company')` | `CompanyMiddleware` + `scopeWhereCompany()` | 中间件静默回退到用户第一个公司 |
| 支出报表 | Web PDF | URL `{hash}` → `Company::where('unique_hash')` | `authorize('view report')` + `scopeWhereCompanyId()` | 授权失败抛 403 |
| 损益报表 | Web PDF | URL `{hash}` → `Company::where('unique_hash')` | `authorize('view report')` + `scopeWhereCompanyId()` | 授权失败抛 403 |
| 仪表盘 | API | `header('company')` | `CompanyMiddleware` + `scopeWhereCompany()` | 中间件静默回退到用户第一个公司 |

---

## 七、关键文件索引

| 角色 | 文件路径 |
|------|---------|
| 支出模型（含所有 scope） | `app/Models/Expense.php` |
| 分类模型（含 amount 访问器） | `app/Models/ExpenseCategory.php` |
| 公司中间件 | `app/Http/Middleware/CompanyMiddleware.php` |
| 支出请求（含 base_amount 计算） | `app/Http/Requests/ExpenseRequest.php` |
| 分类创建请求（无唯一约束） | `app/Http/Requests/ExpenseCategoryRequest.php` |
| 仪表盘控制器 | `app/Http/Controllers/V1/Admin/Dashboard/DashboardController.php` |
| 支出报表控制器 | `app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php` |
| 损益报表控制器 | `app/Http/Controllers/V1/Admin/Report/ProfitLossReportController.php` |
| 分类列表控制器 | `app/Http/Controllers/V1/Admin/Expense/ExpenseCategoriesController.php` |
| 报表授权策略 | `app/Policies/ReportPolicy.php` |
| 仪表盘前端 Store | `resources/scripts/admin/stores/dashboard.js` |
| 分类前端 Store | `resources/scripts/admin/stores/category.js` |
| 分类管理页视图 | `resources/scripts/admin/views/settings/ExpenseCategorySetting.vue` |
| API 路由 | `routes/api.php` |
| Web 报表路由 | `routes/web.php` |
