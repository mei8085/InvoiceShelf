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

## 七、分类金额语境辨析：三个容易混淆的"分类列表"

用户口中的"分类列表"在代码中实际对应三个不同层次的概念，它们对 `amount` 字段的消费情况、承担的业务角色完全不同。本节把三者边界划清。

### 7.1 概念区分总览

| 概念 | 定义位置 | 业务角色 | 是否消费 amount |
|------|---------|---------|---------------|
| **分类接口返回值** | `GET /api/v1/categories` 的 HTTP 响应 JSON | 通用数据源，供多个前端页面消费 | ✅ Resource 始终序列化 amount |
| **分类管理页** | `ExpenseCategorySetting.vue` | 管理员维护分类字典（增删改） | ❌ 页面不渲染 amount 列 |
| **分类列表（本文语境）** | 分析文章中的术语，泛指分类维度的聚合结果 | 支撑报表与金额统计 | 视情况而定 |

---

### 7.2 分类接口返回值（通用数据源）

**路由**：[api.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/routes/api.php#L320)

```php
Route::apiResource('categories', ExpenseCategoriesController::class);
```

**控制器**：[ExpenseCategoriesController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Expense/ExpenseCategoriesController.php#L20-L32)

```php
public function index(Request $request)
{
    $this->authorize('viewAny', ExpenseCategory::class);

    $limit = $request->has('limit') ? $request->limit : 5;

    $categories = ExpenseCategory::applyFilters($request->all())
        ->whereCompany()
        ->latest()
        ->paginateData($limit);

    return ExpenseCategoryResource::collection($categories);
}
```

**Resource**：[ExpenseCategoryResource.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Resources/ExpenseCategoryResource.php#L15-L28)

```php
public function toArray($request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'description' => $this->description,
        'company_id' => $this->company_id,
        'amount' => $this->amount,           // ← 始终返回
        'formatted_created_at' => $this->formattedCreatedAt,
    ];
}
```

**结论**：接口响应体中 `amount` 字段是**无条件存在**的，无论调用方是否需要。这是导致"分类管理页没用 amount 但后端仍在算"的直接原因。

---

### 7.3 分类接口的三个实际消费者

分类接口并非只被分类管理页调用，实际上有三个前端页面通过 [category.js](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/stores/category.js) 的 `fetchCategories()` 访问它：

```js
// category.js#L26-L38
fetchCategories(params) {
  return new Promise((resolve, reject) => {
    http.get(`/api/v1/categories`, { params })
      .then((response) => {
        this.categories = response.data.data   // 把整段响应塞到 store
        resolve(response)
      })
  })
}
```

#### 消费者 1：分类管理页（设置页）

[ExpenseCategorySetting.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/settings/ExpenseCategorySetting.vue#L86-L103)

```js
async function fetchData({ page, filter, sort }) {
  let data = { orderByField: sort.fieldName || 'created_at', orderBy: sort.order || 'desc', page }
  let response = await categoryStore.fetchCategories(data)
  return {
    data: response.data.data,          // 数据给 BaseTable
    pagination: { totalPages: response.data.meta.last_page, ... }
  }
}
```

表格列定义（见 [ExpenseCategorySetting.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/settings/ExpenseCategorySetting.vue#L62-L84)）**只有 name、description、actions 三列**，`amount` 字段随响应体返回但被静默丢弃。

#### 消费者 2：支出创建/编辑页（下拉选择器）

[Create.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/expenses/Create.vue#L73-L88)

```html
<BaseMultiselect
  v-model="expenseStore.currentExpense.expense_category_id"
  value-prop="id"
  label="name"     <!-- 下拉只显示 name -->
  track-by="id"
  :options="searchCategory"
  ...
/>
```

[Create.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/expenses/Create.vue#L436-L446) 的 `searchCategory()` 把搜索结果作为下拉候选项，**只使用 `id` 和 `name`**。

#### 消费者 3：支出列表页（筛选下拉）

[Index.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/expenses/Index.vue#L53-L66)

```html
<BaseMultiselect
  v-model="filters.expense_category_id"
  value-prop="id"
  label="name"    <!-- 筛选器只显示 name -->
  track-by="name"
  :options="searchCategory"
  ...
/>
```

[Index.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/resources/scripts/admin/views/expenses/Index.vue#L328-L335) 在页面挂载时拉全部分类用于筛选下拉，**也只消费 `id` 和 `name`**。

---

### 7.4 三者关系与 amount 的尴尬位置

```
┌──────────────────────────────────────────────────────────────┐
│           GET /api/v1/categories （接口返回值）               │
│  { id, name, description, company_id, amount, formatted_... } │
│                      ↑ amount 无条件存在                       │
└──────────────┬─────────────────────────────┬─────────────────┘
               │                             │
               ▼                             ▼
  ┌─────────────────────────┐    ┌────────────────────────────┐
  │  分类管理页（设置页）     │    │  支出创建页 / 支出列表页     │
  │  渲染列：name/desc/操作  │    │  仅做下拉选择，读 id+name   │
  │  ❌ 完全不展示 amount    │    │  ❌ 也不展示 amount         │
  └─────────────────────────┘    └────────────────────────────┘
```

**`amount` 字段在整条链路中的处境**：
- Resource 层不管谁调用，一律序列化 `$this->amount`
- 这触发了 `ExpenseCategory::getAmountAttribute()` 访问器执行 `SELECT SUM(amount) FROM expenses WHERE expense_category_id = ?`
- 但前端三个消费者没有一个页面真正渲染或读取 `amount`
- 每分页返回 5 条分类就多执行 5 次 SQL，属于**典型的过度计算**

> 如果将来某页面确实需要展示分类金额，也应当把 `base_amount` 作为统一口径（参见 2.3 节和第五节），而非当前访问器中的 `sum('amount')`。

---

## 八、公司请求参数为何不真正参与分类结果裁决

"请求里带的 `company` 参数"这个说法容易让人以为是查询参数（如 `?company=5`）。实际代码中它是 **HTTP Header**（`company`），并且经历了"中间件校正 → scope 只读校正后的值"两层过滤，用户传入的原始值并不直接决定结果。本节把链路讲透。

### 8.1 请求参数进入系统的入口与路径

#### 参数形态

前端并不是发 `?company=5`，而是在 HTTP 请求头中携带：

```
GET /api/v1/categories
Headers:
  Authorization: Bearer <token>
  company: 5          ← 当前选中的公司 ID
  Accept: application/json
```

#### 中间件的挂载位置

[bootstrap/app.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/bootstrap/app.php#L70-L84) 先把中间件别名注册好：

```php
$middleware->alias([
    'auth' => Authenticate::class,
    'company' => CompanyMiddleware::class,   // ← 别名叫 company
    'bouncer' => ScopeBouncer::class,
    // ...
]);
```

[api.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/routes/api.php#L191-L192) 的路由组按顺序嵌套：

```php
Route::middleware(['auth:sanctum', 'company'])->group(function () {
    Route::middleware(['bouncer'])->group(function () {
        // ... 所有 API，包括 /categories、/dashboard
    });
});
```

**执行顺序**：`auth:sanctum` 先验证登录态 → `company` 中间件再校正公司 → `bouncer` → 进入控制器。

---

### 8.2 CompanyMiddleware 的校正逻辑

[CompanyMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Middleware/CompanyMiddleware.php#L17-L28) 是整个链路的关键裁决点：

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

把这段逻辑拆成真值表：

| `header('company')` 存在吗？ | 用户 `hasCompany(header 值)` 吗？ | 中间件动作 | 最终 header 值 |
|---------------------------|-------------------------------|----------|-------------|
| ✅ 存在 | ✅ 有权 | 什么都不做，原样放行 | ✅ 用户传的值生效 |
| ✅ 存在 | ❌ 无权（越权传他人公司 ID） | 覆盖为 `companies()->first()` | ❌ 用户传的值被丢弃 |
| ❌ 不存在 | — | 覆盖为 `companies()->first()` | ❌ 回退到默认公司 |

**这就是"请求参数不真正参与裁决"的根源**：用户传的 header 值先被当作"建议值"，只有通过了 `$user->hasCompany()` 权限校验才被采纳，否则被静默替换。

---

### 8.3 控制器与 scope 只读"校正后"的值

分类接口 [ExpenseCategoriesController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Expense/ExpenseCategoriesController.php#L20-L32)：

```php
public function index(Request $request)
{
    $categories = ExpenseCategory::applyFilters($request->all())
        ->whereCompany()    // ← 调 scope
        ->latest()
        ->paginateData($limit);
}
```

`whereCompany()` scope 定义在 [ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php#L46-L49)：

```php
public function scopeWhereCompany($query)
{
    $query->where('company_id', request()->header('company'));
    //                              ↑ 读的是 request 对象上"已被中间件改过"的 header
}
```

这里的 `request()->header('company')` 不是前端原始传进来的值，而是**经过 CompanyMiddleware 校正后的最终值**。

---

### 8.4 一条请求的完整决策链路

```
前端发起请求
  │
  │  Headers: company=5  （用户"想"看公司 5）
  ▼
auth:sanctum 中间件
  │
  │  验证 Bearer token，取出当前用户（假设用户 ID=42，属于公司 1 和 2）
  ▼
company 中间件 (CompanyMiddleware)
  │
  │  1. $request->header('company') → "5"
  │  2. $user->hasCompany(5)        → false （用户只属于 1、2）
  │  3. $request->headers->set('company', $user->companies()->first()->id)
  │                                    覆盖为 1（用户的第一个公司）
  │
  │  ⚠️  到这里，header('company') 的值已经从 "5" 变成 "1"
  ▼
bouncer 中间件 (可选)
  │
  ▼
ExpenseCategoriesController::index()
  │
  ├─ ExpenseCategory::applyFilters(...)
  │     → 处理 search、category_id 等业务筛选
  │
  └─ ExpenseCategory::whereCompany()
        → 读 request()->header('company') → "1"（校正后的值）
        → WHERE company_id = 1
  ▼
返回：公司 1 下的分类（而非用户传的 5）
```

**结果**：用户传了 `company=5`，但拿到的是公司 1 的数据。请求参数**只是建议**，中间件才是真正的裁决者。

---

### 8.5 两套机制的对比：为什么 PDF 路由不走 header

| 机制 | 适用路由 | 信息来源 | 是否经过中间件 | 越权后果 |
|------|---------|---------|-------------|---------|
| header + whereCompany() | API 路由（`/api/v1/...`） | `request()->header('company')` | ✅ `CompanyMiddleware` 静默校正 | 用户无感，回退默认公司 |
| URL hash + whereCompanyId() | Web PDF 路由（`/reports/...`） | URL 段 `{hash}` 反查 company | ❌ reports 路由组未挂 company 中间件 | `authorize()` 失败直接 403 |

PDF 路由不依赖 header 的根本原因：
1. [web.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/routes/web.php#L54-L75) 的 `reports` 组只挂了 `auth:sanctum`，**没有 `company` 中间件**，header 即使传了也不会被校正
2. 报表 URL 需要被嵌入 iframe 或被用户复制分享，header 难以随 URL 一起传递，而 `unique_hash` 天然存在于路径中
3. 显式 `authorize('view report', $company)` 策略比静默回退更严谨——报表数据敏感，越权时应直接拒绝而非降级展示

---

### 8.6 关键代码节点一览

| 节点 | 文件 | 做什么 |
|------|------|--------|
| header 别名 | `bootstrap/app.php` | `'company' => CompanyMiddleware::class` |
| 路由挂载顺序 | `routes/api.php` | `auth:sanctum` → `company` → `bouncer` → 控制器 |
| header 校正 | `app/Http/Middleware/CompanyMiddleware.php` | 校验 `hasCompany()`，非法则覆盖为默认 |
| scope 读 header | `app/Models/ExpenseCategory.php#L46-L49` | `where('company_id', request()->header('company'))` |
| PDF 读 URL hash | `app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php#L27` | `Company::where('unique_hash', $hash)` |
| PDF 策略授权 | `app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php#L29` | `$this->authorize('view report', $company)` |

---

## 九、公司隔离的四个易被忽视的代码级问题

公司隔离机制表面上通过 `whereCompany()` 和 `CompanyMiddleware` 实现了数据隔离，但在四个细节上存在容易被触发但难以排查的问题。本节逐一分析其触发条件、SQL 形式、隐蔽性及影响。

---

### 9.1 问题一：按 id 过滤分类时 `orWhere` 破坏公司隔离

#### 代码位置

[ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php#L51-L54)

```php
public function scopeWhereCategory($query, $category_id)
{
    $query->orWhere('id', $category_id);  // ⚠️  用了 orWhere，前面没有 where 条件
}
```

该 scope 被 [ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php#L61-L76) 的 `scopeApplyFilters()` 在 `category_id` 参数存在时调用：

```php
public function scopeApplyFilters($query, array $filters)
{
    $filters = collect($filters);

    if ($filters->get('category_id')) {
        $query->whereCategory($filters->get('category_id'));  // 触发 orWhere
    }
    // ...
}
```

#### 典型调用链（分类列表接口）

```php
// ExpenseCategoriesController::index()
$categories = ExpenseCategory::applyFilters($request->all())  // 先调 applyFilters
    ->whereCompany()                                           // 后调 whereCompany
    ->latest()
    ->paginateData($limit);
```

#### 生成的 SQL

请求：`GET /api/v1/categories?category_id=7`

当前公司 header：`company: 1`

```sql
SELECT *
FROM expense_categories
WHERE id = 7                  -- orWhere 展开后变成独立条件
   OR company_id = 1          -- 原本的公司隔离条件
ORDER BY created_at DESC
LIMIT 5 OFFSET 0
```

**致命问题**：`OR company_id = 1` 意味着 **id=7 的分类，即使不属于公司 1，也会被返回**。

#### 更广泛的影响：多个 Model 存在同一模式

`orWhere('id', xxx)` 不是孤立问题，整个项目至少有 **8 个 Model** 复制了同样的错误模式：

| Model | 代码位置 | 问题 scope |
|-------|---------|-----------|
| ExpenseCategory | `ExpenseCategory.php#L53` | `scopeWhereCategory` |
| Expense | `Expense.php#L188` | `scopeWhereExpense` |
| Item | `Item.php#L71` | `scopeWhereItem` |
| Payment | `Payment.php#L364` | `scopeWherePayment` |
| PaymentMethod | `PaymentMethod.php#L62` | `scopeWherePaymentMethod` |
| TaxType | `TaxType.php#L48` | `scopeWhereTaxType` |
| Unit | `Unit.php#L33` | `scopeWhereUnit` |
| Customer | `Customer.php#L293` | `scopeWhereCustomer` |
| Invoice | `Invoice.php#L299` | `scopeWhereInvoice` |
| Estimate | `Estimate.php#L144` | `scopeWhereEstimate` |

每一个的实现形式完全相同：

```php
public function scopeWhereXxx($query, $xxx_id)
{
    $query->orWhere('id', $xxx_id);  // 同样的 orWhere 问题
}
```

#### 触发条件

只要满足：
1. 前端请求中携带 `?category_id=xxx`（或对应 Model 的 `?xxx_id=xxx`）
2. 调用顺序为 `applyFilters()` → `whereCompany()`（先过滤后加公司隔离）

#### 为何平时不易发现

- 分类列表页的正常调用通常不传 `category_id` 参数，只传 `search`、`page`、`limit`
- 只有当通过 API 直接构造带 `category_id` 的请求时才会触发
- 返回结果中混入一条其他公司的数据，在 UI 上可能只是"多了一条"，不易立刻察觉
- 触发条件依赖"先 applyFilters 后 whereCompany"的调用顺序，若换过来则不触发

#### 对分类结果的影响

- **横向越权**：公司 A 的用户传入公司 B 的某个分类 ID，可以读取到该分类的 `id`、`name`、`description` 等全部字段
- **隔离失效**：公司隔离条件被 `OR` 运算符弱化，不再是硬约束
- **影响面广**：同样问题波及支出、产品、回款、客户、发票等几乎所有核心实体

---

### 9.2 问题二：`applyFilters` 中 `company_id` 被静默丢弃

#### 代码位置

[ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php#L69-L71)

```php
public function scopeApplyFilters($query, array $filters)
{
    $filters = collect($filters);

    // ...

    if ($filters->get('company_id')) {
        $query->whereCompany($filters->get('company_id'));  // ⚠️  传了参数但被忽略
    }

    // ...
}
```

看 `scopeWhereCompany` 的定义（[ExpenseCategory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/ExpenseCategory.php#L46-L49)）：

```php
public function scopeWhereCompany($query)
{
    $query->where('company_id', request()->header('company'));  // ⚠️  不接收参数，直接读 header
}
```

调用方传入的 `$filters->get('company_id')` 被 `whereCompany()` **完全忽略**。

#### 生成的 SQL

请求：`GET /api/v1/categories?company_id=999`（企图绕过 header 查公司 999）

当前 header：`company: 1`

```sql
SELECT *
FROM expense_categories
WHERE company_id = 1       -- 用的是 header 的值，不是 999
ORDER BY created_at DESC
LIMIT 5 OFFSET 0
```

#### 触发条件

只要请求中携带 `company_id` 查询参数，就会触发这个被静默丢弃的分支。

#### 为何平时不易发现

- 正常前端代码不会在查询参数里传 `company_id`，都是通过 header 传
- 攻击者即使发现这个参数并尝试传值，也不会得到预期结果（因为实际还是用 header），所以这个分支是"看似有效、实则无效"的死代码
- 不会报错，不会在日志中留下任何痕迹，难以通过常规手段发现

#### 对分类结果的影响

- **无害但具有迷惑性**：看似可以通过 `?company_id=` 来指定公司，实际仍由 header 决定
- **代码腐烂**：无效分支不仅误导后续维护者，还可能让真正需要按公司过滤的场景被错误实现
- **不一致性**：同样叫 `whereCompany()`，`Expense::scopeWhereCompanyId($company)` 是接收参数的，而 `ExpenseCategory::scopeWhereCompany()` 是不接收参数的，两套 API 风格不一致

---

### 9.3 问题三：仪表盘 12 个月计算触发大量重复 SQL

#### 代码位置

[DashboardController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Controllers/V1/Admin/Dashboard/DashboardController.php#L62-L100)

```php
while ($monthCounter < 12) {
    array_push(
        $invoice_totals,
        Invoice::whereBetween('invoice_date', [$start, $end])
            ->whereCompany()
            ->sum('base_total')
    );

    array_push(
        $expense_totals,
        Expense::whereBetween('expense_date', [$start, $end])
            ->whereCompany()
            ->sum('base_amount')
    );

    array_push(
        $receipt_totals,
        Payment::whereBetween('payment_date', [$start, $end])
            ->whereCompany()
            ->sum('base_amount')
    );

    array_push($net_income_totals, ($receipt_totals[$i] - $expense_totals[$i]));

    // ... 移动日期窗口
    $monthCounter++;
}
```

循环结束后还有 4 次年度总计查询：

```php
$total_sales    = Invoice::whereBetween(...)->whereCompany()->sum('base_total');
$total_receipts = Payment::whereBetween(...)->whereCompany()->sum('base_amount');
$total_expenses = Expense::whereBetween(...)->whereCompany()->sum('base_amount');
$total_net_income = $total_receipts - $total_expenses;
```

#### SQL 调用形式

**单次循环执行 3 条聚合查询**：

```sql
-- 第1个月
SELECT SUM(base_total)  FROM invoices WHERE invoice_date BETWEEN ? AND ? AND company_id = ?;
SELECT SUM(base_amount) FROM expenses WHERE expense_date BETWEEN ? AND ? AND company_id = ?;
SELECT SUM(base_amount) FROM payments  WHERE payment_date  BETWEEN ? AND ? AND company_id = ?;

-- 第2个月
SELECT SUM(base_total)  FROM invoices WHERE invoice_date BETWEEN ? AND ? AND company_id = ?;
SELECT SUM(base_amount) FROM expenses WHERE expense_date BETWEEN ? AND ? AND company_id = ?;
SELECT SUM(base_amount) FROM payments  WHERE payment_date  BETWEEN ? AND ? AND company_id = ?;

... 重复 12 次 ...

-- 年度总计（4次）
SELECT SUM(base_total)  FROM invoices WHERE invoice_date BETWEEN ? AND ? AND company_id = ?;
SELECT SUM(base_amount) FROM expenses WHERE expense_date BETWEEN ? AND ? AND company_id = ?;
SELECT SUM(base_amount) FROM payments  WHERE payment_date  BETWEEN ? AND ? AND company_id = ?;
```

**总查询次数**：12 月 × 3 表 + 4 次年度总计 = **40 次 SQL 查询**（每次 Dashboard 加载）。

更严重的是 `net_income_totals` 是用 PHP 计算的（`$receipt_totals[$i] - $expense_totals[$i]`），本身不需要数据库查询，但为了获得分子分母多执行了 24 次查询。

#### 触发条件

每次用户打开仪表盘页面，或前端 store 调用 `loadData()` 时触发。

#### 为何平时不易发现

- 开发环境数据量小（每个表几千条以内），40 次查询总耗时通常在 100-300ms，感知不明显
- 每个 `SUM()` 查询都走索引（`expense_date`、`invoice_date` 等字段通常有索引），单条查询很快
- 没有 N+1 明显的错误模式（如循环内查询关联模型），常规性能分析工具不容易标记为"慢查询"
- 只有当数据量增长到每个表几十万条、或并发用户多时，响应时间才会明显劣化

#### 对分类结果（及整体性能）的影响

- **响应时间线性增长**：每增加一个月的数据量，3 个 `SUM()` 的耗时都会增长
- **数据库压力**：40 次查询 / 次访问，10 个并发用户就是 400 次并发查询
- **本可优化**：理论上只需 3 条查询（按 `GROUP BY MONTH(expense_date)` 聚合一次，取出 12 个月数据），即可代替 36 次月度 `SUM()` 查询
- **分类层面的间接影响**：虽然问题三不直接影响分类结果，但仪表盘是用户高频访问页面，其性能问题会拖慢整个系统，间接导致分类列表等其他接口响应变慢

---

### 9.4 问题四：`CompanyMiddleware` 在无公司信息时触发空指针

#### 代码位置

[CompanyMiddleware.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Http/Middleware/CompanyMiddleware.php#L17-L28)

```php
public function handle(Request $request, Closure $next): Response
{
    if (Schema::hasTable('user_company')) {
        $user = $request->user();

        if ((! $request->header('company')) || (! $user->hasCompany($request->header('company')))) {
            // ⚠️  两处潜在空指针
            $request->headers->set('company', $user->companies()->first()->id);
        }
    }

    return $next($request);
}
```

#### 空指针触发点 1：`$user` 为 null

第 22 行 `$user->hasCompany(...)` 和第 23 行 `$user->companies()` 都假设 `$user` 非空。

但是 `$request->user()` 在以下情况会返回 `null`：
- Sanctum token 已过期但请求仍命中了 `auth:sanctum` 中间件组
- token 被撤销
- 测试环境中未正确设置用户上下文

此时调用 `$user->hasCompany()` 会抛出：
```
Error: Call to a member function hasCompany() on null
```

#### 空指针触发点 2：`$user->companies()->first()` 返回 null

第 23 行 `$user->companies()->first()->id` 假设 `first()` 一定有结果。

但 `companies()` 是多对多关联（[User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/User.php#L127-L130)）：

```php
public function companies(): BelongsToMany
{
    return $this->belongsToMany(Company::class, 'user_company', 'user_id', 'company_id');
}
```

如果 `user_company` 表中没有该用户的任何关联记录（新创建的用户还未分配公司，或所有关联被意外删除），`first()` 返回 `null`，调用 `->id` 会抛出：

```
Error: Attempt to read property "id" on null
```

#### 触发条件

| 场景 | 触发概率 | 现象 |
|------|---------|------|
| 用户账号刚创建，尚未分配任何公司 | 中 | 新员工首次登录时报 500 错误 |
| 通过后台删除了 `user_company` 中某用户的最后一条关联 | 低 | 该用户所有 API 请求全部 500 |
| Sanctum token 过期或被撤销 | 中（生产环境偶发） | 前端收到 500 而非正常的 401 |
| 单元测试/集成测试未设置用户上下文 | 高（开发期） | 测试 CI 报红 |

#### 为何平时不易发现

- 正常业务流程中，创建用户的同时会在 `user_company` 表插入关联记录，99.9% 的用户至少属于一个公司
- token 过期时 Sanctum 本身的 `auth:sanctum` 中间件通常先抛出 401，请求还没到 `company` 中间件
- 只有当 token 有效但用户权限刚好被清空、或者用户数据库被手工改动时才触发
- 问题属于"尾部场景"，在正常使用路径上难以覆盖

#### 对分类结果的影响

- **完全阻断**：一旦触发空指针，整个请求链提前终止于 500 错误，分类接口不会返回任何数据
- **用户体验差**：没有优雅降级（如跳转到"无公司权限"提示页），直接展示通用错误页
- **故障排查难**：日志中只显示 `Error: Call to a member function hasCompany() on null`，运维人员需要一定时间才能定位到是中间件的问题
- **同样问题波及其他文件**：[User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/49-InvoiceShelf/app/Models/User.php#L94-L96) 的 `getFormattedCreatedAtAttribute()` 中也有同样的 `$this->companies()->first()->id` 模式，同样面临空指针风险

---

### 9.5 四个问题对比汇总

| 问题 | 严重程度 | 触发概率 | 隐蔽性 | 直接影响 |
|------|---------|---------|-------|---------|
| 1. orWhere 破坏隔离 | ⚠️ 高 | 中 | 高 | 横向越权，跨公司读取分类 |
| 2. company_id 被丢弃 | ⚠️ 中 | 低 | 极高 | 死代码，误导开发者 |
| 3. 仪表盘重复 SQL | ⚠️ 中 | 极高（每次访问） | 中 | 性能随数据量线性劣化 |
| 4. 中间件空指针 | ⚠️ 高 | 低 | 低 | 500 错误阻断整个请求 |

---

## 十、关键文件索引

| 角色 | 文件路径 |
|------|---------|
| 支出模型（含所有 scope） | `app/Models/Expense.php` |
| 分类模型（含 amount 访问器、orWhere 问题） | `app/Models/ExpenseCategory.php` |
| 公司中间件（含空指针问题） | `app/Http/Middleware/CompanyMiddleware.php` |
| 支出请求（含 base_amount 计算） | `app/Http/Requests/ExpenseRequest.php` |
| 分类创建请求（无唯一约束） | `app/Http/Requests/ExpenseCategoryRequest.php` |
| 分类列表控制器 | `app/Http/Controllers/V1/Admin/Expense/ExpenseCategoriesController.php` |
| 仪表盘控制器（含循环查询问题） | `app/Http/Controllers/V1/Admin/Dashboard/DashboardController.php` |
| 支出报表控制器 | `app/Http/Controllers/V1/Admin/Report/ExpensesReportController.php` |
| 损益报表控制器 | `app/Http/Controllers/V1/Admin/Report/ProfitLossReportController.php` |
| 报表授权策略 | `app/Policies/ReportPolicy.php` |
| 分类 Resource | `app/Http/Resources/ExpenseCategoryResource.php` |
| 用户模型（含 companies() 关联） | `app/Models/User.php` |
| 分类前端 Store | `resources/scripts/admin/stores/category.js` |
| 仪表盘前端 Store | `resources/scripts/admin/stores/dashboard.js` |
| 分类管理页视图 | `resources/scripts/admin/views/settings/ExpenseCategorySetting.vue` |
| 支出创建页（消费分类下拉） | `resources/scripts/admin/views/expenses/Create.vue` |
| 支出列表页（消费分类筛选） | `resources/scripts/admin/views/expenses/Index.vue` |
| 中间件注册 | `bootstrap/app.php` |
| API 路由 | `routes/api.php` |
| Web 报表路由 | `routes/web.php` |
