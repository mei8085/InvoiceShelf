# InvoiceShelf 发票渲染失败路径分析

本文档深入分析**在线预览**、**队列预生成**、**邮件附件发送**三条路径中，模板缺失或渲染失败时的异常处理机制与返回值行为。

---

## 一、失败场景分类

在进入各路径分析前，先明确两类主要失败场景：

| 失败类型 | 触发条件 | 抛出异常 |
|---------|---------|---------|
| **模板缺失** | `Invoice.template_name` 指向不存在的模板文件 | `ErrorException: Trying to access array offset on value of type null` |
| **渲染失败** | 驱动错误（Gotenberg 连接超时、Dompdf 内存溢出、字体缺失等） | 驱动特定异常（`Gotenberg\Exceptions\*`、`Dompdf\Exceptions\*`） |

两类失败均发生在 **`Invoice::getPDFData()`** 方法内部，且**默认无 try/catch 捕获**。

---

## 二、路径一：在线预览

### 2.1 调用链路

```
HTTP GET /invoices/pdf/{unique_hash}?preview=1
   ↓
InvoicePdfController::__invoke()
   ├─► request()->has('preview') == true
   └─► return $invoice->getPDFData();
       ├─► 构建视图数据
       ├─► 找模板: PdfTemplateUtils::findFormattedTemplate()
       ├─► 模板缺失 → ErrorException (无捕获)
       └─► 渲染: PDF::loadView($templatePath)
           └─► 渲染失败 → 驱动异常 (无捕获)
```

### 2.2 失败行为分析

| 失败场景 | 异常类型 | 捕获情况 | 最终行为 | 用户体验 |
|---------|---------|---------|---------|---------|
| 模板缺失 | `ErrorException` | ❌ 未捕获 | 冒泡到 Laravel Handler | **500 错误页面/JSON** |
| 驱动渲染失败 | 驱动特定异常 | ❌ 未捕获 | 冒泡到 Laravel Handler | **500 错误页面/JSON** |

### 2.3 关键代码

**无防护直接调用** [app/Http/Controllers/V1/PDF/InvoicePdfController.php:19-21](app/Http/Controllers/V1/PDF/InvoicePdfController.php#L19-L21)：

```php
if ($request->has('preview')) {
    return $invoice->getPDFData();  // 无 try/catch
}
```

**`getPDFData()` 无错误处理** [app/Models/Invoice.php:603-610](app/Models/Invoice.php#L603-L610)：

```php
$template = PdfTemplateUtils::findFormattedTemplate('invoice', $invoiceTemplate, '');
// $template 可能为 null，下一行直接触发 ErrorException
$templatePath = $template['custom'] ? sprintf(...) : sprintf(...);

if (request()->has('preview')) {
    return view($templatePath);  // 模板不存在时这里也会抛出 InvalidArgumentException
}

return PDF::loadView($templatePath);
```

> **注意**：当 `preview` 参数存在时，`getPDFData()` 返回的是 View 实例而非 PDF 对象，由 Laravel 自动渲染为 HTML。此时如果模板文件不存在，会抛出 `InvalidArgumentException` 而非视图渲染错误。

### 2.4 静默处理场景

在线预览路径**无任何静默处理**，所有失败都会直接冒泡为 500 错误。

---

## 三、路径二：队列预生成

### 3.1 调用链路

```
Invoice 创建/更新
   ↓
InvoicesController::store()/update()
   └─► GenerateInvoicePdfJob::dispatch($invoice)
       ↓
Queue Worker 执行
   ↓
GenerateInvoicePdfJob::handle()
   └─► $invoice->generatePDF('invoice', $fileName)
       ├─► 检查 save_pdf_to_disk 设置
       ├─► $pdf = $this->getPDFData();
       │   ├─► 模板缺失 → ErrorException
       │   └─► 渲染失败 → 驱动异常
       ├─► 写入临时文件
       └─► try { 关联 medialibrary } catch (\Exception $e) { return $e->getMessage(); }
```

### 3.2 失败行为分析

| 失败阶段 | 异常类型 | 捕获情况 | 最终行为 | 可观测性 |
|---------|---------|---------|---------|---------|
| **getPDFData() 阶段**<br>（模板缺失/渲染失败） | `ErrorException` 或驱动异常 | ❌ **未捕获** | Job 失败，进入重试队列 | Laravel 队列失败日志 |
| **medialibrary 关联阶段**<br>（文件写入失败/权限问题） | 任意 `Exception` | ✅ 已捕获 | 返回错误消息字符串，Job 标记成功 | 无日志，静默失败 |

### 3.3 关键代码

**Job::handle() 无防护** [app/Jobs/GenerateInvoicePdfJob.php:36-41](app/Jobs/GenerateInvoicePdfJob.php#L36-L41)：

```php
public function handle(): int
{
    $this->invoice->generatePDF('invoice', $this->invoice->invoice_number, $this->deleteExistingFile);
    // 无 try/catch，getPDFData() 抛出异常会导致 Job 失败
    
    return 0;
}
```

**generatePDF() 部分捕获** [app/Traits/GeneratesPdfTrait.php:70-109](app/Traits/GeneratesPdfTrait.php#L70-L109)：

```php
public function generatePDF($collection_name, $file_name, $deleteExistingFile = false)
{
    $save_pdf_to_disk = CompanySetting::getSetting('save_pdf_to_disk', $this->company_id);

    if ($save_pdf_to_disk == 'NO') {
        return 0;  // 静默跳过，不生成 PDF
    }

    // ... 设置 locale ...

    $pdf = $this->getPDFData();  // BUG: 这里抛出的异常不会被下面的 catch 捕获
    \Storage::disk('local')->put('temp/.../temp.pdf', $pdf->output());

    // ... 清理旧文件 ...

    try {
        $this->addMedia($media)
            ->withCustomProperties(['file_disk_id' => $file_disk->id])
            ->usingFileName($file_name.'.pdf')
            ->toMediaCollection($collection_name, config('filesystems.default'));

        \Storage::disk('local')->deleteDirectory('temp/...');

        return true;
    } catch (\Exception $e) {
        return $e->getMessage();  // 只捕获 medialibrary 阶段的异常
    }
}
```

### 3.4 静默处理场景

| 场景 | 行为 |
|-----|------|
| `save_pdf_to_disk == 'NO'` | 直接 return 0，不生成 PDF，无任何日志 |
| medialibrary 关联失败 | catch 异常后返回错误消息字符串，Job 仍标记为成功，无日志 |

> **严重问题**：`getPDFData()` 发生在 try/catch 块外部，模板缺失或驱动渲染失败会导致 Job 不断重试（默认 3 次），浪费队列资源。

---

## 四、路径三：邮件附件发送

### 4.1 调用链路

```
HTTP POST /admin/invoices/{id}/send
   ↓
SendInvoiceController::__invoke()
   └─► $invoice->send($request->all())
       ├─► $data = $this->sendInvoiceData($data)
       │   └─► $data['attach']['data'] = $this->getEmailAttachmentSetting() ? $this->getPDFData() : null
       │       ├─► 模板缺失 → ErrorException
       │       └─► 渲染失败 → 驱动异常
       └─► \Mail::to(...)->send(new SendInvoiceMail($data))
           └─► SendInvoiceMail::build()
               ├─► 创建 EmailLog
               └─► if ($data['attach']['data']) { attachData(...) }
```

### 4.2 失败行为分析

| 失败场景 | 异常类型 | 捕获情况 | 最终行为 | 用户体验 |
|---------|---------|---------|---------|---------|
| 模板缺失 | `ErrorException` | ❌ 未捕获 | 邮件发送中断，返回 500 | **API 返回 500 JSON** |
| 驱动渲染失败 | 驱动特定异常 | ❌ 未捕获 | 邮件发送中断，返回 500 | **API 返回 500 JSON** |
| 附件设置关闭 (`NO`) | - | ✅ 逻辑跳过 | 邮件正常发送，无附件 | 无感知 |

### 4.3 关键代码

**sendInvoiceData() 无防护** [app/Models/Invoice.php:453-463](app/Models/Invoice.php#L453-L463)：

```php
public function sendInvoiceData($data)
{
    $data['invoice'] = $this->toArray();
    $data['customer'] = $this->customer->toArray();
    $data['company'] = Company::find($this->company_id);
    $data['subject'] = $this->getEmailString($data['subject']);
    $data['body'] = $this->getEmailString($data['body']);
    
    // 无 try/catch，getPDFData() 失败会直接冒泡
    $data['attach']['data'] = ($this->getEmailAttachmentSetting()) ? $this->getPDFData() : null;

    return $data;
}
```

**send() 无防护** [app/Models/Invoice.php:475-498](app/Models/Invoice.php#L475-L498)：

```php
public function send($data)
{
    $data = $this->sendInvoiceData($data);  // 异常在这里抛出

    $mail = \Mail::to($data['to']);
    // ... cc/bcc 设置 ...
    $mail->send(new SendInvoiceMail($data));

    // 这些状态更新永远不会执行
    if ($this->status == Invoice::STATUS_DRAFT) {
        $this->status = Invoice::STATUS_SENT;
        $this->sent = true;
        $this->save();
    }

    return ['success' => true];
}
```

**SendInvoiceMail::build() 检查** [app/Mail/SendInvoiceMail.php:56-61](app/Mail/SendInvoiceMail.php#L56-L61)：

```php
if ($this->data['attach']['data']) {
    $mailContent->attachData(
        $this->data['attach']['data']->output(),
        $this->data['invoice']['invoice_number'].'.pdf'
    );
}
```

### 4.4 静默处理场景

| 场景 | 行为 |
|-----|------|
| `getEmailAttachmentSetting() == 'NO'` | `$data['attach']['data']` 设为 null，邮件正常发送，无附件 |
| `$data['attach']['data']` 为 null | `build()` 中 if 判断跳过，邮件正常发送 |

> **问题**：邮件发送过程中，PDF 生成失败会导致整个邮件发送中断，且发票状态不会更新为 `SENT`。用户会看到 500 错误，但 EmailLog 也不会被创建（因为 EmailLog 在 `SendInvoiceMail::build()` 中创建，而 build() 在 send() 内部调用，异常发生在这之前）。

---

## 五、跨路径对照分析

### 5.1 失败行为对比表

| 维度 | 在线预览 | 队列预生成 | 邮件附件发送 |
|-----|---------|-----------|-------------|
| **入口** | `InvoicePdfController` | `GenerateInvoicePdfJob` | `SendInvoiceController` |
| **核心方法** | `getPDFData()` | `generatePDF()` → `getPDFData()` | `sendInvoiceData()` → `getPDFData()` |
| **模板缺失** | 500 错误 | Job 重试 3 次后永久失败 | 500 错误，邮件未发送 |
| **渲染失败** | 500 错误 | Job 重试 3 次后永久失败 | 500 错误，邮件未发送 |
| **静默场景** | 无 | `save_pdf_to_disk=NO`、medialibrary 失败 | 附件设置关闭、`attach.data=null` |
| **日志记录** | Laravel 异常日志 | 队列失败日志 | Laravel 异常日志 |
| **用户影响** | 无法查看预览 | PDF 不会预生成，首次访问实时生成 | 邮件发送失败，需重试 |

### 5.2 共同缺陷

三条路径共享同一组核心缺陷：

1. **`getPDFData()` 缺乏错误处理**：所有路径直接调用而无 try/catch
2. **模板 null 访问**：`$template['custom']` 未检查 null 直接访问
3. **错误信息不透明**：用户看到 500 错误，但不知道是模板问题还是驱动问题

### 5.3 路径特有缺陷

| 路径 | 特有缺陷 |
|-----|---------|
| 队列预生成 | `getPDFData()` 在 try/catch 外部，失败导致不必要的重试；medialibrary 失败静默无日志 |
| 邮件附件发送 | PDF 生成失败导致邮件发送完全中断，状态不更新，无 EmailLog |

### 5.4 失败传播链对比

```
在线预览:
Controller → getPDFData() → Exception → Laravel Handler → 500 Response

队列预生成:
Controller → dispatch Job → Worker → generatePDF() → getPDFData() → Exception 
→ Job 标记失败 → 重试 → ... → 永久失败 → 队列失败日志

邮件发送:
Controller → send() → sendInvoiceData() → getPDFData() → Exception
→ Laravel Handler → 500 Response
  (邮件未发送, 状态未更新, EmailLog 未创建)
```

---

## 六、改进建议

### 6.1 通用修复：加固 `getPDFData()`

```php
public function getPDFData()
{
    // ... 现有代码 ...

    $template = PdfTemplateUtils::findFormattedTemplate('invoice', $invoiceTemplate, '');
    
    // 修复 1: 模板缺失回退
    if ($template === null) {
        $template = PdfTemplateUtils::findFormattedTemplate('invoice', 'invoice1', '');
        $invoiceTemplate = 'invoice1';
    }
    
    $templatePath = $template['custom'] 
        ? sprintf('pdf_templates::invoice.%s', $invoiceTemplate)
        : sprintf('app.pdf.invoice.%s', $invoiceTemplate);

    try {
        if (request()->has('preview')) {
            return view($templatePath);
        }
        return PDF::loadView($templatePath);
    } catch (\Exception $e) {
        report($e);  // 记录日志
        throw new \RuntimeException('PDF 生成失败: ' . $e->getMessage(), 0, $e);
    }
}
```

### 6.2 队列路径修复

```php
public function generatePDF(...)
{
    // ...
    
    try {
        $pdf = $this->getPDFData();  // 移入 try 块
        \Storage::disk('local')->put('temp/...', $pdf->output());
        
        // ... medialibrary 逻辑 ...
        
        return true;
    } catch (\Exception $e) {
        report($e);  // 记录日志
        return $e->getMessage();
    }
}
```

### 6.3 邮件路径修复

```php
public function send($data)
{
    try {
        $data = $this->sendInvoiceData($data);
    } catch (\RuntimeException $e) {
        // PDF 生成失败但继续发送邮件（无附件）
        $data['attach']['data'] = null;
        // 可选择添加警告到邮件内容
    }
    
    // ... 继续发送邮件 ...
}
```

---

## 七、静默处理场景汇总

| 路径 | 静默场景 | 发生位置 | 对用户的影响 |
|-----|---------|---------|-------------|
| 所有路径 | `save_pdf_to_disk = 'NO'` | `generatePDF()` 第 74 行 | 不预生成 PDF，无任何提示 |
| 队列预生成 | medialibrary 关联失败 | `generatePDF()` catch 块 | PDF 未保存但 Job 成功，用户不知情 |
| 邮件发送 | `invoice_email_attachment = 'NO'` | `sendInvoiceData()` 第 460 行 | 邮件无附件发送，用户需在邮件正文中点击链接查看 |
| 邮件发送 | `$data['attach']['data']` 为 null | `SendInvoiceMail::build()` 第 56 行 | 邮件无附件发送 |

---

## 附录：关键方法调用关系

```
getPDFData() ── 所有路径的核心，无错误处理
   ▲
   ├─► InvoicePdfController (在线预览)
   ├─► GeneratesPdfTrait::getGeneratedPDFOrStream() (在线查看)
   ├─► GeneratesPdfTrait::generatePDF() (队列预生成)
   └─► Invoice::sendInvoiceData() (邮件附件)
```

所有路径最终都调用 `getPDFData()`，该方法的任何异常都会直接影响所有调用方。
