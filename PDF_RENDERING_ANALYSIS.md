# InvoiceShelf 发票 PDF 渲染跨层代码路径分析

## 概述

InvoiceShelf 的发票 PDF 渲染系统采用**驱动抽象层 + 模板引擎 + 媒体库**的三层架构，支持 `dompdf` 和 `gotenberg` 两种渲染驱动，通过配置动态切换。本文从驱动选择策略、模板数据上下文构建、错误处理机制三个维度进行深度分析。

---

## 一、驱动选择策略

### 1.1 驱动抽象架构

系统通过接口抽象和工厂模式实现驱动的可插拔设计：

```
┌─────────────────────────────────────────────────────────┐
│                   PDFService Facade                     │
│              App\Facades\PDF (facade accessor)          │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                PDFService (Entry Point)                 │
│       App\Services\PDFService::loadView()               │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              PDFDriverFactory (Factory Pattern)         │
│       App\Services\PDFDriverFactory::create()           │
└─────────────────────────────┬───────────────────────────┘
          ┌───────────────────┴───────────────────┐
          ▼                                       ▼
┌──────────────────────┐             ┌──────────────────────────┐
│  Dompdf Driver       │             │  Gotenberg Driver        │
│  barryvdh/laravel-dompdf          │  App\Services\PDFDrivers\ │
│  (Third-party package)            │  GotenbergPDFDriver       │
└──────────────────────┘             └──────────────────────────┘
```

### 1.2 配置优先级

驱动配置采用**数据库配置优先，环境变量兜底**的两级策略：

| 配置层级 | 位置 | 说明 |
|---------|------|------|
| L1 数据库 | `settings` 表 | `pdf_driver`, `gotenberg_host`, `gotenberg_papersize` |
| L2 环境变量 | `.env` | `PDF_DRIVER`, `GOTENBERG_HOST`, `GOTENBERG_PAPERSIZE` |
| L3 默认值 | `config/pdf.php` | `dompdf` 作为默认驱动 |

**配置加载流程** [app/Providers/AppConfigProvider.php:118-157](app/Providers/AppConfigProvider.php#L118-L157)：

```php
// 1. 从数据库读取配置
$pdfSettings = Setting::getSettings([
    'pdf_driver', 'gotenberg_host', 'gotenberg_papersize', 'gotenberg_margins',
]);

// 2. 动态覆盖 config/pdf.php
if (! empty($pdfSettings['pdf_driver'])) {
    Config::set('pdf.driver', $pdfSettings['pdf_driver']);
    // 驱动特有配置...
}
```

### 1.3 驱动工厂实现

[app/Services/PDFService.php:27-37](app/Services/PDFService.php#L27-L37)

```php
class PDFDriverFactory
{
    public static function create(string $driver)
    {
        return match ($driver) {
            'dompdf' => App::make('dompdf.wrapper'),
            'gotenberg' => new GotenbergPDFDriver,
            default => throw new \InvalidArgumentException('Invalid PDFDriver requested')
        };
    }
}
```

### 1.4 驱动对比

| 特性 | Dompdf | Gotenberg |
|------|--------|-----------|
| 依赖 | PHP 扩展，本地渲染 | 独立服务（Chrome 无头模式） |
| CSS 支持 | 有限（CSS 2.1 子集） | 完整（现代 Chromium 引擎） |
| 性能 | 单线程，适合中小量 | 可水平扩展，适合高并发 |
| 部署复杂度 | 低（Composer 包） | 高（Docker 容器） |
| JavaScript | 不支持 | 完整支持 |

---

## 二、模板数据上下文构建

### 2.1 数据流全景

```
HTTP Request
     │
     ▼
InvoicePdfController::__invoke()
     │  [app/Http/Controllers/V1/PDF/InvoicePdfController.php]
     ├─► 预览模式: 直接返回视图
     └─► 生产模式: getGeneratedPDFOrStream()
                      │
                      ▼
          Invoice::getPDFData()
              │
              ├─► 聚合关联数据 (items, taxes, customer, company)
              ├─► 构建视图共享数据
              ├─► 解析模板路径
              └─► PDF::loadView($templatePath)
                      │
                      ▼
                Blade 模板渲染
                      │
                      ▼
                驱动生成 PDF
```

### 2.2 核心数据构建方法

**`Invoice::getPDFData()`** [app/Models/Invoice.php:562-611] 是数据上下文构建的核心：

```php
public function getPDFData()
{
    // 1. 税费聚合（按税种分组）
    $taxes = collect();
    if ($this->tax_per_item === 'YES') {
        foreach ($this->items as $item) {
            foreach ($item->taxes as $tax) {
                // 合并同税种金额...
            }
        }
    }

    // 2. 本地化设置
    $locale = CompanySetting::getSetting('language', $company->id);
    App::setLocale($locale);

    // 3. 视图数据共享
    view()->share([
        'invoice' => $this,
        'customFields' => $customFields,
        'company_address' => $this->getCompanyAddress(),
        'shipping_address' => $this->getCustomerShippingAddress(),
        'billing_address' => $this->getCustomerBillingAddress(),
        'notes' => $this->getNotes(),
        'logo' => $logo ?? null,
        'taxes' => $taxes,
    ]);

    // 4. 模板路径解析（支持自定义模板）
    $template = PdfTemplateUtils::findFormattedTemplate('invoice', $invoiceTemplate, '');
    $templatePath = $template['custom'] 
        ? sprintf('pdf_templates::invoice.%s', $invoiceTemplate)
        : sprintf('app.pdf.invoice.%s', $invoiceTemplate);

    return PDF::loadView($templatePath);
}
```

### 2.3 地址格式化引擎

**`GeneratesPdfTrait::getFormattedString()`** [app/Traits/GeneratesPdfTrait.php:172-191] 实现了灵活的占位符替换系统：

```php
public function getFormattedString($format)
{
    // 1. 合并基础字段 + 扩展字段
    $values = array_merge($this->getFieldsArray(), $this->getExtraFields());
    
    // 2. 占位符替换 (例如: {INVOICE_NUMBER} → INV-0001)
    $str = strtr($format, $values);
    
    // 3. 清理未匹配的占位符
    $str = preg_replace('/{(.*?)}/', '', $str);
    
    // 4. HTML 安全过滤（XSS/SSRF 防护）
    return PdfHtmlSanitizer::sanitize($str);
}
```

**内置字段映射** [app/Traits/GeneratesPdfTrait.php:112-170]：

| 字段组 | 前缀 | 示例 |
|--------|------|------|
| 公司地址 | `{COMPANY_*}` | `{COMPANY_NAME}`, `{COMPANY_PHONE}` |
| 账单地址 | `{BILLING_*}` | `{BILLING_ADDRESS_NAME}`, `{BILLING_ZIP_CODE}` |
| 收货地址 | `{SHIPPING_*}` | `{SHIPPING_COUNTRY}`, `{SHIPPING_CITY}` |
| 客户信息 | `{CONTACT_*}` | `{CONTACT_DISPLAY_NAME}`, `{CONTACT_EMAIL}` |
| 发票字段 | `{INVOICE_*}` | `{INVOICE_NUMBER}`, `{INVOICE_DATE}` |
| 自定义字段 | `{SLUG}` | 动态注入 |

### 2.4 模板解析策略

**`PdfTemplateUtils::findFormattedTemplate()`** [app/Space/PdfTemplateUtils.php:17-26] 支持双模板源：

```
模板查找顺序:
1. 自定义模板 → storage/app/pdf_templates/invoice/[name].blade.php
2. 系统内置模板 → resources/views/app/pdf/invoice/[name].blade.php
```

内置模板：
- `invoice1.blade.php` - 经典布局（左侧公司信息，右侧发票详情）
- `invoice2.blade.php` - 现代布局
- `invoice3.blade.php` - 简约布局

### 2.5 货币格式化

**`format_money_pdf()`** [app/Space/helpers.php:128-151] 处理 PDF 中的货币显示：

```php
function format_money_pdf($money, $currency = null)
{
    $money = $money / 100;  // 分转元
    
    // 根据货币配置格式化
    $format_money = number_format(
        $money,
        $currency->precision,
        $currency->decimal_separator,
        $currency->thousand_separator
    );
    
    // 处理货币符号位置（DejaVu Sans 字体确保 PDF 兼容性）
    if ($currency->swap_currency_symbol) {
        return $format_money.'<span style="font-family: DejaVu Sans;">'.$currency->symbol.'</span>';
    }
    return '<span style="font-family: DejaVu Sans;">'.$currency->symbol.'</span>'.$format_money;
}
```

---

## 三、错误处理机制

### 3.1 错误分层处理策略

系统采用**分层捕获、降级处理**的错误策略：

| 层级 | 处理方式 | 位置 |
|------|---------|------|
| 配置层 | 静默失败，使用默认值 | `AppConfigProvider::configurePDFFromDatabase()` |
| 驱动层 | 抛出异常，由上层处理 | `PDFDriverFactory::create()` |
| 模型层 | 捕获异常，返回 false/错误消息 | `GeneratesPdfTrait::generatePDF()` |
| 控制器层 | 直接返回响应，无额外包装 | `InvoicePdfController` |
| 队列层 | Laravel 队列重试机制 | `GenerateInvoicePdfJob` |

### 3.2 关键错误处理点

#### 3.2.1 配置加载容错

[app/Providers/AppConfigProvider.php:154-157](app/Providers/AppConfigProvider.php#L154-L157)

```php
try {
    $pdfSettings = Setting::getSettings([...]);
    // 配置逻辑...
} catch (\Exception $e) {
    // 静默失败 - 安装/迁移阶段数据库不可用时
    // 使用 config/pdf.php 中的默认配置
}
```

#### 3.2.2 驱动参数校验

[app/Services/PDFDrivers/GotenbergPDFDriver.php:40-43](app/Services/PDFDrivers/GotenbergPDFDriver.php#L40-L43)

```php
$papersize = explode(' ', config('pdf.connections.gotenberg.papersize'));
if (count($papersize) != 2) {
    throw new \InvalidArgumentException('Invalid Gotenberg Papersize specified');
}
```

#### 3.2.3 PDF 生成与存储容错

[app/Traits/GeneratesPdfTrait.php:98-109](app/Traits/GeneratesPdfTrait.php#L98-L109)

```php
try {
    $this->addMedia($media)
        ->withCustomProperties(['file_disk_id' => $file_disk->id])
        ->usingFileName($file_name.'.pdf')
        ->toMediaCollection($collection_name, config('filesystems.default'));

    \Storage::disk('local')->deleteDirectory('temp/'.$collection_name.'/'.$this->id);

    return true;
} catch (\Exception $e) {
    return $e->getMessage();  // 返回错误消息供上层处理
}
```

#### 3.2.4 已生成 PDF 读取容错

[app/Traits/GeneratesPdfTrait.php:38-68](app/Traits/GeneratesPdfTrait.php#L38-L68)

```php
public function getGeneratedPDF($collection_name)
{
    try {
        $media = $this->getMedia($collection_name)->first();
        
        if ($media) {
            $file_disk = FileDisk::find($media->custom_properties['file_disk_id']);
            
            if (! $file_disk) {
                return false;  // 文件磁盘配置丢失
            }
            
            $file_disk->setConfig();
            
            // 根据磁盘类型返回路径或临时 URL
            $path = $file_disk->driver == 'local' 
                ? $media->getPath() 
                : $media->getTemporaryUrl(Carbon::now()->addMinutes(5));
            
            return collect(['path' => $path, 'file_name' => $media->file_name]);
        }
    } catch (\Exception $e) {
        return false;  // 任何异常都降级为重新生成
    }
    
    return false;
}
```

### 3.3 安全防护：HTML  sanitizer

**`PdfHtmlSanitizer::sanitize()`** [app/Support/PdfHtmlSanitizer.php:16-68] 是关键的安全层：

```php
public static function sanitize(string $html): string
{
    // 1. 标准化换行标签
    $html = str_replace('</br>', '<br />', $html);
    
    // 2. 白名单标签过滤
    $allowedTags = '<br><br/><p><b><strong><i><em><u><ol><ul><li><table><tr><td><th><thead><tbody><tfoot><h1><h2><h3><h4><blockquote>';
    $html = strip_tags($html, $allowedTags);
    
    // 3. DOM 解析并移除危险属性
    $doc = new DOMDocument;
    $doc->loadHTML($wrapped, LIBXML_HTML_NOIMPLIED | LIBXML_HTML_NODEFDTD);
    
    $xpath = new DOMXPath($doc);
    foreach ($xpath->query('.//*', $root) as $element) {
        foreach ($element->attributes as $attr) {
            if (self::shouldRemoveAttribute($attr->name)) {
                $element->removeAttribute($attr->name);
            }
        }
    }
    
    // 危险属性列表: on*, style, src, href, srcset, formaction 等
}
```

**防护目的**：
- 防止 SSRF（通过 `src`/`href` 发起内部网络请求）
- 防止 XSS（通过 `on*` 事件处理器）
- 防止样式注入破坏 PDF 布局

---

## 四、异步生成流程

### 4.1 队列 Job 设计

[app/Jobs/GenerateInvoicePdfJob.php](app/Jobs/GenerateInvoicePdfJob.php)

```php
class GenerateInvoicePdfJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public $invoice;
    public $deleteExistingFile;
    
    public function __construct($invoice, $deleteExistingFile = false)
    {
        $this->invoice = $invoice;
        $this->deleteExistingFile = $deleteExistingFile;
    }
    
    public function handle(): int
    {
        $this->invoice->generatePDF('invoice', $this->invoice->invoice_number, $this->deleteExistingFile);
        return 0;
    }
}
```

### 4.2 触发时机

PDF 生成在以下场景被触发：
1. **实时预览** - 控制器直接调用 `getPDFData()`，不存储
2. **发票发送** - `SendInvoiceMail` 中按需生成作为附件
3. **后台预生成** - 发票创建/更新后 dispatch Job 异步生成
4. **首次访问** - `getGeneratedPDFOrStream()` 中按需生成

---

## 五、关键代码路径汇总

### 5.1 同步渲染路径（预览/在线查看）

```
1. GET /invoices/pdf/{unique_hash}
   ↓ routes/web.php:89
2. InvoicePdfController::__invoke(Invoice $invoice)
   ↓ app/Http/Controllers/V1/PDF/InvoicePdfController.php:17-24
3. Invoice::getGeneratedPDFOrStream('invoice')
   ├─► 检查已生成的 PDF (getGeneratedPDF)
   └─► 无缓存则调用 Invoice::getPDFData()
       ├─► 聚合数据、设置本地化
       ├─► 共享视图数据
       ├─► 解析模板路径
       └─► PDF::loadView($templatePath)
           ↓ app/Facades/PDF.php
           ↓ app/Services/PDFService.php:41-46
           ↓ app/Services/PDFService.php:29-36 (工厂选择驱动)
           ↓ 驱动 loadView() 实现
           ↓ Blade 渲染 → PDF 生成
4. 返回 Response (Content-Type: application/pdf)
```

### 5.2 异步生成路径（队列）

```
1. Invoice 创建/更新后
   ↓
2. GenerateInvoicePdfJob::dispatch($invoice)
   ↓
3. Queue Worker 执行 Job::handle()
   ↓
4. Invoice::generatePDF('invoice', $fileName)
   ├─► 检查 save_pdf_to_disk 设置
   ├─► 调用 getPDFData() 生成 PDF
   ├─► 写入临时文件: storage/app/temp/invoice/{id}/temp.pdf
   ├─► 通过 spatie/medialibrary 关联到 Invoice 模型
   └─► 清理临时文件
```

---

## 六、扩展点与优化建议

### 6.1 当前架构优势

1. **驱动抽象良好** - 新增驱动只需实现 `loadView()` 接口
2. **模板灵活** - 支持自定义模板覆盖系统模板
3. **安全到位** - HTML sanitizer 有效防止注入攻击
4. **容错设计** - 分层错误处理，降级策略清晰

### 6.2 潜在改进点

1. **驱动接口标准化** - 当前 Dompdf 通过 `App::make('dompdf.wrapper')` 获取，与 `GotenbergPDFDriver` 没有共同接口，建议：
   ```php
   interface PDFDriver {
       public function loadView(string $viewname): ResponseStream;
   }
   ```
   （文件中已定义接口但未强制实现）

2. **错误日志缺失** - 多处 `catch (\Exception $e)` 后直接返回 false，未记录日志，建议增加 `report($e)`

3. **配置验证时机** - Gotenberg 连接可用性可在配置保存时预检查

4. **缓存粒度优化** - 当前模板变更后需清除视图缓存，可考虑增加模板版本戳

---

## 附录：核心文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 配置定义 | `config/pdf.php` |
| 驱动工厂 | `app/Services/PDFService.php` |
| Gotenberg 驱动 | `app/Services/PDFDrivers/GotenbergPDFDriver.php` |
| PDF Facade | `app/Facades/PDF.php` |
| 服务提供者 | `app/Providers/PDFServiceProvider.php` |
| 配置动态加载 | `app/Providers/AppConfigProvider.php` |
| PDF 生成 Trait | `app/Traits/GeneratesPdfTrait.php` |
| HTML 安全过滤 | `app/Support/PdfHtmlSanitizer.php` |
| 模板工具 | `app/Space/PdfTemplateUtils.php` |
| 发票模型 | `app/Models/Invoice.php` |
| 异步生成 Job | `app/Jobs/GenerateInvoicePdfJob.php` |
| PDF 控制器 | `app/Http/Controllers/V1/PDF/InvoicePdfController.php` |
| PDF 配置 API | `app/Http/Controllers/V1/Admin/Settings/PDFConfigurationController.php` |
| 内置模板 | `resources/views/app/pdf/invoice/invoice{1,2,3}.blade.php` |
