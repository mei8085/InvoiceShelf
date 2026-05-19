# InvoiceShelf 队列执行模式失败传播差异分析

本文档深入分析 **同步队列（sync）** 与 **异步队列（database/redis 等）** 两种配置下，发票 PDF 渲染失败时的行为差异，覆盖创建/更新接口、队列执行进程、失败任务记录和用户返回值四个维度。

---

## 一、队列基础配置

### 1.1 默认配置

**config/queue.php:16**

```php
'default' => env('QUEUE_CONNECTION', 'sync'),
```

默认使用 `sync` 驱动（同步执行）。可通过 `.env` 中的 `QUEUE_CONNECTION` 切换为 `database`、`redis` 等异步驱动。

### 1.2 Job 定义特性

**app/Jobs/GenerateInvoicePdfJob.php:11-41**

```php
class GenerateInvoicePdfJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    // 未自定义: $tries, $maxExceptions, $backoff, $retryUntil
    // 未定义: failed() 方法
    
    public function handle(): int
    {
        $this->invoice->generatePDF('invoice', $this->invoice->invoice_number, $this->deleteExistingFile);
        return 0;
    }
}
```

**关键特性**：
- 未显式设置重试次数 → 使用 Laravel 默认值（通常 1 次，即不重试）
- 未定义 `failed()` 方法 → 依赖 Laravel 全局失败处理
- `handle()` 无 try/catch → 异常直接抛出给队列系统

---

## 二、同步队列（sync）失败传播分析

### 2.1 执行模型

同步队列在当前请求进程中**立即执行**，不经过队列 worker 进程：

```
HTTP Request
   ↓
InvoicesController::store()
   ├─► $invoice = Invoice::createInvoice($request);
   └─► GenerateInvoicePdfJob::dispatch($invoice);
       │
       ▼ 同步执行（sync 驱动）
   SyncQueue::push() → 立即调用 Job::handle()
       ↓
Invoice::generatePDF()
   ├─► 检查 save_pdf_to_disk
   ├─► Invoice::getPDFData()  ← 失败发生点
   │   ├─► 模板缺失 → ErrorException
   │   └─► 渲染失败 → 驱动异常
   └─► 异常未被捕获 → 向上冒泡
       ↓
InvoicesController
   ↓
Laravel Exception Handler
   ↓
HTTP 500 Response
```

### 2.2 各维度失败行为

| 维度 | 同步队列行为 |
|-----|-------------|
| **创建/更新接口** | 接口调用被阻塞，直到 Job 执行完成（成功或失败） |
| **执行进程** | 在 HTTP 请求进程中执行，无独立 worker |
| **失败任务记录** | ❌ **不记录** 到 `failed_jobs` 表。SyncQueue 不触发失败队列逻辑 |
| **用户返回值** | ❌ **HTTP 500 错误**。异常直接冒泡到异常处理器 |
| **数据一致性** | ✅ 发票已创建但 PDF 未生成。用户看到错误但数据已持久化 |

### 2.3 关键代码行为

**sync 驱动特性**：
- `SyncQueue::push()` 直接调用 `$job->fire()`，没有队列持久化
- 异常抛出后，`SyncQueue` 不会捕获，也不会写入 `failed_jobs`
- `dispatch()` 调用返回的 PendingDispatch 不会被使用
- 控制器中 `dispatch()` 之后的代码（如 `return new InvoiceResource(...)`）**永远不会执行**

### 2.4 静默处理场景（同步队列特有）

| 场景 | 行为 | 用户感知 |
|-----|------|---------|
| `save_pdf_to_disk = 'NO'` | `generatePDF()` 返回 0，Job 成功 | 无感知，发票正常返回 |
| medialibrary 关联失败 | catch 后返回错误字符串，Job 仍标记成功 | ❌ **严重静默失败**：用户收到成功响应，但 PDF 未保存 |

> **注意**：medialibrary 阶段的失败在同步队列下仍然是静默的，因为 `generatePDF()` 内部捕获了该异常并返回字符串，`handle()` 将其视为成功。

---

## 三、异步队列（database/redis）失败传播分析

### 3.1 执行模型

异步队列将 Job 序列化后存储到队列后端，由独立的 worker 进程拉取执行：

```
HTTP Request (同步部分)
   ↓
InvoicesController::store()
   ├─► $invoice = Invoice::createInvoice($request);
   └─► GenerateInvoicePdfJob::dispatch($invoice);
       │
       ▼ 序列化入队
   Queue::push() → 写入 jobs 表 / Redis
       ↓
   return new InvoiceResource($invoice);  ✅ 立即返回 200/201
       ↓
HTTP Response (用户看到成功)

=================================================================

Queue Worker Process (异步部分)
   ↓
php artisan queue:work
   ├─► 从队列拉取 Job
   ├─► 反序列化，调用 Job::handle()
   ├─► Invoice::generatePDF()
   │   ├─► Invoice::getPDFData()  ← 失败发生点
   │   │   ├─► 模板缺失 → ErrorException
   │   │   └─► 渲染失败 → 驱动异常
   │   └─► 异常未被捕获 → 抛出给 Worker
   ├─► Worker 捕获异常
   ├─► 检查重试次数（默认 1 次，即不重试）
   └─► 超过重试次数 → 标记为失败
       ↓
   写入 failed_jobs 表（含异常堆栈）
       ↓
   Worker 继续处理下一个 Job
```

### 3.2 各维度失败行为

| 维度 | 异步队列行为 |
|-----|-------------|
| **创建/更新接口** | ✅ 立即返回成功响应（200/201），不等待 Job 执行 |
| **执行进程** | 在独立的 Queue Worker 进程中执行 |
| **失败任务记录** | ✅ **记录** 到 `failed_jobs` 表，包含完整异常堆栈 |
| **用户返回值** | ✅ **HTTP 200/201**。用户看到发票创建成功，不知道 PDF 生成失败 |
| **数据一致性** | ⚠️ 发票创建成功但 PDF 生成失败。状态不一致，需后续补偿 |

### 3.3 关键代码行为

**异步队列特性**：
- `dispatch()` 将 Job 序列化后写入队列，立即返回
- 控制器后续代码正常执行，返回成功响应
- Worker 进程独立执行，异常不会影响 HTTP 请求
- 超过重试次数后，调用 `FailedJobProvider::log()` 写入 `failed_jobs`

**Laravel 默认重试行为**：
- Job 未设置 `$tries` → 默认 1 次尝试（即失败后不重试）
- 如果 Worker 使用 `--tries=3` 参数 → 最多尝试 3 次
- 每次失败后，`attempts` 计数递增

### 3.4 静默处理场景（异步队列特有）

| 场景 | 行为 | 用户感知 |
|-----|------|---------|
| `save_pdf_to_disk = 'NO'` | `generatePDF()` 返回 0，Job 成功 | 无感知 |
| medialibrary 关联失败 | catch 后返回错误字符串，Job 标记成功 | ❌ **静默失败**：无任何告警，仅能通过检查 PDF 是否存在发现 |
| 任何其他失败（模板/渲染） | 写入 `failed_jobs` 表 | ❌ 用户无感知，需管理员后台检查 |

---

## 四、与在线预览、邮件发送路径对比

### 4.1 失败传播矩阵

| 路径 | 队列模式 | 触发点 | 异常类型 | 捕获情况 | 用户返回 | 失败记录 | 静默程度 |
|-----|---------|-------|---------|---------|---------|---------|---------|
| **在线预览** | - | `InvoicePdfController` | `ErrorException` / 驱动异常 | ❌ 未捕获 | ❌ HTTP 500 | Laravel 日志 | 无 |
| **邮件发送** | - | `SendInvoiceController` | `ErrorException` / 驱动异常 | ❌ 未捕获 | ❌ HTTP 500 | Laravel 日志 | 无 |
| **队列预生成** | **sync** | `dispatch()` 同步执行 | `ErrorException` / 驱动异常 | ❌ 未捕获 | ❌ HTTP 500 | ❌ 不记录 failed_jobs | 低 |
| **队列预生成** | **async** | Worker 执行 | `ErrorException` / 驱动异常 | ❌ 未捕获 | ✅ HTTP 200 | ✅ 写入 failed_jobs | **高**（用户无感知） |

### 4.2 静默失败对比

| 静默场景 | 在线预览 | 邮件发送 | 同步队列 | 异步队列 |
|---------|---------|---------|---------|---------|
| `save_pdf_to_disk = 'NO'` | - | - | ✅ 静默 | ✅ 静默 |
| medialibrary 关联失败 | - | - | ❌ **静默且用户成功** | ❌ **静默且用户成功** |
| 模板缺失 | ❌ 500 | ❌ 500 | ❌ 500 | ✅ 用户看到成功 |
| 驱动渲染失败 | ❌ 500 | ❌ 500 | ❌ 500 | ✅ 用户看到成功 |

---

## 五、关键差异点深度分析

### 5.1 用户返回值对比

```
同步队列失败时序:
  T0: 用户发起 POST /invoices
  T1: Invoice 创建，写入数据库
  T2: dispatch() → 同步执行 Job
  T3: getPDFData() 抛出异常
  T4: 异常冒泡到控制器
  T5: Laravel Handler 返回 500
  用户看到: ❌ 500 错误（但发票已创建）

异步队列失败时序:
  T0: 用户发起 POST /invoices
  T1: Invoice 创建，写入数据库
  T2: dispatch() → Job 写入队列
  T3: 返回 InvoiceResource (201 Created)
  T4: 用户看到: ✅ 成功
  T5: (稍后) Worker 拉取 Job 执行
  T6: getPDFData() 抛出异常
  T7: 写入 failed_jobs 表
  用户看到: 无任何反馈
```

### 5.2 失败可观测性对比

| 失败类型 | 同步队列 | 异步队列 |
|---------|---------|---------|
| 模板缺失 | ❶ 用户看到 500<br>❷ Laravel 日志<br>❸ ❌ failed_jobs 无记录 | ❶ 用户看到成功<br>❷ ✅ failed_jobs 有记录<br>❸ 管理员需主动检查 |
| 渲染失败 | ❶ 用户看到 500<br>❷ Laravel 日志<br>❸ ❌ failed_jobs 无记录 | ❶ 用户看到成功<br>❷ ✅ failed_jobs 有记录<br>❸ 管理员需主动检查 |
| medialibrary 失败 | ❶ 用户看到成功<br>❷ ❌ 无日志<br>❸ ❌ failed_jobs 无记录 | ❶ 用户看到成功<br>❷ ❌ 无日志<br>❸ ❌ failed_jobs 无记录 |

### 5.3 数据一致性影响

| 场景 | 同步队列 | 异步队列 |
|-----|---------|---------|
| 发票创建 | ✅ 已持久化 | ✅ 已持久化 |
| PDF 生成 | ❌ 失败 | ❌ 失败 |
| 用户认知 | 知道失败（看到 500） | 以为成功（看到 200） |
| 状态修复复杂度 | 低（用户重试即可） | 高（需后台重试失败 Job） |

---

## 六、改进建议

### 6.1 通用加固

**`getPDFData()` 增加错误处理**：

```php
public function getPDFData()
{
    // ... 现有代码 ...
    
    $template = PdfTemplateUtils::findFormattedTemplate('invoice', $invoiceTemplate, '');
    
    // 模板缺失回退
    if ($template === null) {
        $template = PdfTemplateUtils::findFormattedTemplate('invoice', 'invoice1', '');
        $invoiceTemplate = 'invoice1';
    }
    
    try {
        $templatePath = $template['custom'] 
            ? sprintf('pdf_templates::invoice.%s', $invoiceTemplate)
            : sprintf('app.pdf.invoice.%s', $invoiceTemplate);
            
        if (request()->has('preview')) {
            return view($templatePath);
        }
        return PDF::loadView($templatePath);
    } catch (\Exception $e) {
        report($e);
        throw new \RuntimeException('PDF 生成失败: ' . $e->getMessage(), 0, $e);
    }
}
```

### 6.2 队列路径加固

**`generatePDF()` 完整捕获**：

```php
public function generatePDF($collection_name, $file_name, $deleteExistingFile = false)
{
    $save_pdf_to_disk = CompanySetting::getSetting('save_pdf_to_disk', $this->company_id);

    if ($save_pdf_to_disk == 'NO') {
        return 0;
    }

    try {
        $locale = CompanySetting::getSetting('language', $this->company_id);
        App::setLocale($locale);

        $pdf = $this->getPDFData();
        
        \Storage::disk('local')->put('temp/'.$collection_name.'/'.$this->id.'/temp.pdf', $pdf->output());

        if ($deleteExistingFile) {
            $this->clearMediaCollection($this->collection_name);
        }

        $file_disk = FileDisk::whereSetAsDefault(true)->first();
        if ($file_disk) {
            $file_disk->setConfig();
        }

        $media = \Storage::disk('local')->path('temp/'.$collection_name.'/'.$this->id.'/temp.pdf');

        $this->addMedia($media)
            ->withCustomProperties(['file_disk_id' => $file_disk->id])
            ->usingFileName($file_name.'.pdf')
            ->toMediaCollection($collection_name, config('filesystems.default'));

        \Storage::disk('local')->deleteDirectory('temp/'.$collection_name.'/'.$this->id);

        return true;
    } catch (\Exception $e) {
        report($e);  // 记录所有异常
        return $e->getMessage();
    }
}
```

### 6.3 异步队列失败告警

在 Job 中增加 `failed()` 方法发送通知：

```php
public function failed(\Throwable $exception): void
{
    // 发送失败通知给管理员
    // 或更新 invoice 状态标记 PDF 生成失败
    $this->invoice->update(['pdf_generation_failed' => true]);
    
    // 可选：发送告警邮件/Slack 通知
    Notification::send(Admin::all(), new PdfGenerationFailed($this->invoice, $exception));
}
```

### 6.4 同步队列优化

如果同步队列是生产环境配置，控制器应捕获异常：

```php
public function store(InvoicesRequest $request)
{
    $this->authorize('create', Invoice::class);
    
    $invoice = Invoice::createInvoice($request);
    
    try {
        GenerateInvoicePdfJob::dispatch($invoice);
    } catch (\RuntimeException $e) {
        // PDF 生成失败但发票已创建，返回警告信息
        return (new InvoiceResource($invoice))
            ->additional(['warning' => '发票创建成功，但 PDF 生成失败: ' . $e->getMessage()]);
    }
    
    return new InvoiceResource($invoice);
}
```

---

## 七、结论汇总

### 7.1 核心差异表

| 维度 | 同步队列 (sync) | 异步队列 (database/redis) |
|-----|----------------|---------------------------|
| **执行时机** | HTTP 请求内同步执行 | Worker 进程异步执行 |
| **用户返回** | 失败时 500 错误 | 始终返回成功（200/201） |
| **失败记录** | ❌ 不写入 failed_jobs | ✅ 写入 failed_jobs |
| **用户感知** | 知道失败 | 以为成功 |
| **静默失败** | medialibrary 阶段 | 模板缺失、渲染失败、medialibrary 阶段 |
| **可观测性** | 高（用户报告 + 日志） | 低（仅管理员检查） |
| **数据一致性** | 发票存在，用户知道需要重试 | 发票存在，PDF 缺失，用户不知情 |

### 7.2 最严重问题

1. **异步队列下用户认知错位**：PDF 生成失败但用户看到成功响应，可能导致业务损失（如发送无附件的发票邮件）
2. **medialibrary 阶段静默失败**：所有队列模式下，文件存储关联失败都无日志、无告警
3. **同步队列下 500 但数据已创建**：用户看到错误但发票已存在，可能重复创建

### 7.3 部署建议

- **生产环境推荐使用异步队列**，避免 PDF 生成失败阻塞用户操作
- 配合失败 Job 监控告警（如 Laravel Horizon 或自定义 failed() 通知）
- 在发票列表页面显示 PDF 生成状态标识，让用户能直观看到失败情况
