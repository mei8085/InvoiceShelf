# 备份服务代码理解文档

## 一、整体架构概述

本项目的备份服务基于 **spatie/laravel-backup** 第三方包构建，提供了数据库和文件的备份、归档、清理和监控功能。整体采用 **队列异步执行** + **配置驱动** + **规则校验** 的设计模式。

核心组件关系图：

```
前端(BackupSetting.vue)
       ↓ HTTP 请求
BackupsController (触发 + 校验)
       ↓ dispatch
CreateBackupJob (队列任务)
       ↓ make
BackupConfigurationFactory (配置工厂)
       ↓ createFromConfig
Spatie BackupJob (第三方包核心)
       ↓ 执行
备份归档 (ZIP压缩 + 存储到磁盘)
```

---

## 二、备份触发机制

### 2.1 触发入口

备份有以下几种触发方式：

#### （1）用户手动触发（主要方式）
用户在前端设置页面点击「新建备份」按钮，通过 API 接口触发：

- **接口**：`POST /api/v1/backups`
- **控制器**：[BackupsController@store](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L75-L85)

```php
public function store(Request $request)
{
    $this->authorize('manage backups');

    $data = $request->all();
    $data['company'] = $request->header('company');

    dispatch(new CreateBackupJob($data))->onQueue(config('backup.queue.name'));

    return $this->respondSuccess();
}
```

#### （2）定时任务触发
通过 Webhook 端点触发 Laravel 调度器：

- **接口**：`POST /api/v1/cron-job` (假设)
- **控制器**：[CronJobController](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Webhook/CronJobController.php)

```php
public function __invoke(Request $request)
{
    Artisan::call('schedule:run');
    return response()->json(['success' => true]);
}
```

### 2.2 触发流程详解

1. **权限校验**：使用 `$this->authorize('manage backups')` 验证用户权限
2. **数据组装**：将请求数据与 company header 组合
3. **队列分发**：通过 `dispatch()` 将 `CreateBackupJob` 推送到指定队列
4. **异步执行**：队列 Worker 异步执行备份任务，请求立即返回成功

> **设计意图**：备份操作可能耗时较长（尤其是文件备份），采用队列异步执行可以避免请求超时，提升用户体验。

---

## 三、规则校验机制

### 3.1 校验层级

备份服务的校验分为三个层级：

| 层级 | 校验内容 | 位置 |
|------|---------|------|
| 权限层 | 用户是否有备份管理权限 | 控制器方法开头 `authorize()` |
| 输入层 | 请求参数格式校验 | 控制器中 `$request->validate()` |
| 业务层 | 业务逻辑校验（磁盘有效性等） | 配置工厂和备份任务中 |

### 3.2 核心校验规则类

项目自定义了三个备份相关的验证规则：

#### （1）PathToZip - 路径必须指向 ZIP 文件
[app/Rules/Backup/PathToZip.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/PathToZip.php)

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    if (! Str::endsWith($value, '.zip')) {
        $fail('The given value must be a path to a zip file.');
    }
}
```

**使用场景**：删除备份、下载备份时，验证 `path` 参数。

#### （2）BackupDisk - 磁盘必须是已配置的备份磁盘
[app/Rules/Backup/BackupDisk.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/BackupDisk.php)

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    $configuredBackupDisks = config('backup.backup.destination.disks');
    if (! in_array($value, $configuredBackupDisks)) {
        $fail('This disk is not configured as a backup disk.');
    }
}
```

#### （3）FilesystemDisks - 磁盘必须是已配置的文件系统磁盘
[app/Rules/Backup/FilesystemDisks.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/FilesystemDisks.php)

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    $configuredFileSystemDisks = array_keys(config('filesystems.disks'));
    if (! in_array($value, $configuredFileSystemDisks)) {
        $fail('This disk is not configured as a filesystem disk.');
    }
}
```

### 3.3 配置工厂中的校验

[BackupConfigurationFactory](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) 在创建配置时进行业务校验：

```php
public static function make($data = []): Config
{
    if (blank($data['company'] ?? null)) {
        throw new Exception('The Company ID is missig');
    }

    if (blank($data['file_disk_id'] ?? null)) {
        throw new Exception('No file disk selected');
    }

    // ... 后续配置组装
}
```

> **注意**：这里有个拼写错误 `missig` → 应为 `missing`。

---

## 四、数据归档规则

### 4.1 归档配置

备份归档的规则在 [config/backup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/config/backup.php) 中定义。

#### （1）备份来源配置

```php
'source' => [
    'files' => [
        'include' => [base_path()],      // 包含的目录
        'exclude' => [                    // 排除的目录
            base_path('vendor'),
            base_path('node_modules'),
            base_path('.git'),
        ],
        'follow_links' => false,          // 是否跟随符号链接
        'ignore_unreadable_directories' => false,
        'relative_path' => null,
    ],
    'databases' => [env('DB_CONNECTION', 'mysql')],  // 要备份的数据库
],
```

#### （2）归档目标配置

```php
'destination' => [
    'compression_method' => ZipArchive::CM_DEFAULT,  // 压缩算法
    'compression_level' => 9,                         // 压缩级别 0-9
    'filename_prefix' => '',                          // 文件名前缀
    'disks' => ['local'],                             // 存储磁盘
],
```

#### （3）加密配置

```php
'password' => env('BACKUP_ARCHIVE_PASSWORD'),  // 归档密码
'encryption' => 'default',                     // 加密算法 (AES-256)
```

### 4.2 备份类型（option 参数）

在 [CreateBackupJob](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Jobs/CreateBackupJob.php) 中，根据 `option` 参数决定备份内容：

| option 值 | 备份内容 | 文件名前缀 |
|-----------|---------|-----------|
| `full`（或空） | 数据库 + 文件 | 无前缀 |
| `only-db` | 仅数据库 | `only-db-` |
| `only-files` | 仅文件 | `only-files-` |

```php
if ($this->data['option'] === 'only-db') {
    $backupJob->dontBackupFilesystem();  // 不备份文件系统
}

if ($this->data['option'] === 'only-files') {
    $backupJob->dontBackupDatabases();   // 不备份数据库
}

// 设置自定义文件名
if (! empty($this->data['option'])) {
    $prefix = str_replace('_', '-', $this->data['option']).'-';
    $backupJob->setFilename($prefix.date('Y-m-d-H-i-s').'.zip');
}
```

### 4.3 清理策略（归档保留规则）

备份清理使用 `DefaultStrategy` 策略，按时间梯度保留备份：

```php
'cleanup' => [
    'strategy' => DefaultStrategy::class,
    'default_strategy' => [
        'keep_all_backups_for_days' => 7,          // 7天内：保留全部
        'keep_daily_backups_for_days' => 16,       // 7-16天：每天保留最新的一个
        'keep_weekly_backups_for_weeks' => 8,      // 8周内：每周保留最新的一个
        'keep_monthly_backups_for_months' => 4,    // 4个月内：每月保留最新的一个
        'keep_yearly_backups_for_years' => 2,      // 2年内：每年保留最新的一个
        'delete_oldest_backups_when_using_more_megabytes_than' => 5000,  // 超过5GB删最老的
    ],
],
```

> **重要原则**：默认策略永远不会删除最新的备份，确保至少有一个可用备份。

### 4.4 动态磁盘配置

项目支持多租户/多公司的动态磁盘配置。在 [BackupConfigurationFactory](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) 中：

```php
$fileDisk = FileDisk::find($data['file_disk_id']);
$fileDisk->setConfig();  // 动态设置文件系统配置

$prefix = env('DYNAMIC_DISK_PREFIX', 'temp_');
config(['backup.backup.destination.disks' => [$prefix.$fileDisk->driver]]);
```

**工作流程**：
1. 根据 `file_disk_id` 从数据库获取磁盘配置
2. 调用 `setConfig()` 动态注册到 Laravel 文件系统
3. 将备份目标磁盘设置为动态磁盘
4. 支持不同公司使用不同的存储位置

---

## 五、异常处理与回滚机制

### 5.1 异常处理层次

备份服务的异常处理分为多个层次：

#### （1）控制器层 - 列表查询异常捕获
[BackupsController@index](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L24-L68)

```php
try {
    // ... 备份列表查询逻辑
} catch (\Exception $e) {
    return response()->json([
        'backups' => [],
        'error' => 'invalid_disk_credentials',
        'error_message' => $e->getMessage(),
        'disks' => $configuredBackupDisks,
    ]);
}
```

**特点**：
- 捕获磁盘连接异常（如凭证无效）
- 降级返回空列表和错误信息
- 不影响用户继续操作

#### （2）队列任务层 - 备份执行异常

队列任务的异常处理依赖于 **Laravel Queue** 的重试机制和 **spatie/laravel-backup** 内置的重试：

```php
// config/backup.php
'backup' => [
    'tries' => 1,           // 备份失败后的重试次数
    'retry_delay' => 0,     // 重试前等待秒数
],
'cleanup' => [
    'tries' => 1,           // 清理失败后的重试次数
    'retry_delay' => 0,     // 重试前等待秒数
],
```

#### （3）通知层 - 异常通知

当备份失败时，系统会自动发送通知：

```php
'notifications' => [
    'notifications' => [
        BackupHasFailedNotification::class => ['mail'],        // 备份失败
        UnhealthyBackupWasFoundNotification::class => ['mail'], // 备份不健康
        CleanupHasFailedNotification::class => ['mail'],       // 清理失败
        BackupWasSuccessfulNotification::class => ['mail'],     // 备份成功
        // ... 其他通知
    ],
],
```

通知接收邮箱可通过公司设置动态配置：
```php
$companyNotificationEmail = CompanySetting::getSetting('notification_email', $data['company']);
if ($companyNotificationEmail) {
    config(['backup.notifications.mail.to' => $companyNotificationEmail]);
}
```

### 5.2 「回滚」机制解析

> **重要澄清**：本备份服务 **没有传统意义上的数据库事务回滚**。备份过程是原子性的 ZIP 文件生成，失败时的「回滚」体现在以下方面：

#### （1）临时文件机制
- 备份过程中文件先写入 `temporary_directory`（默认：`storage/app/backup-temp`）
- 只有完整备份成功后，才会移动到目标位置
- 如果中途失败，临时文件会被清理，不会产生不完整的备份文件

```php
'temporary_directory' => storage_path('app/backup-temp'),
```

#### （2）备份完整性保障
spatie/laravel-backup 包内部保证：
- **数据库备份**：使用 `mysqldump` 等工具生成完整的 SQL dump 文件，失败则不保留
- **文件备份**：流式写入 ZIP，异常时不会生成损坏的 ZIP 文件
- **最终确认**：只有所有内容都备份完成后，才会将 ZIP 移动到目标磁盘

#### （3）失败不影响原有备份
- 新备份失败时，**不会删除或修改任何已有的成功备份**
- 这是一种「前向容错」设计：最坏情况只是本次备份失败，历史备份仍然可用

#### （4）自动重试机制
配置了 `tries > 1` 时，备份失败后会自动重试：
- 第一次失败 → 等待 `retry_delay` 秒 → 重新执行
- 所有尝试都失败后 → 发送失败通知 → 任务标记为失败

### 5.3 监控与健康检查

备份服务内置健康检查机制，定期检测备份是否健康：

```php
'monitor_backups' => [
    [
        'name' => env('APP_NAME', 'laravel-backup'),
        'disks' => ['local'],
        'health_checks' => [
            MaximumAgeInDays::class => 1,      // 备份文件不能超过1天
            MaximumStorageInMegabytes::class => 5000,  // 总存储不超过5GB
        ],
    ],
],
```

不健康时触发 `UnhealthyBackupWasFoundNotification` 通知。

---

## 六、核心协作流程

### 6.1 完整备份执行流程

```
1. 用户点击「新建备份」
   ↓
2. 前端调用 POST /api/v1/backups
   ↓
3. BackupsController@store
   ├─ 权限校验：authorize('manage backups')
   ├─ 数据组装：添加 company header
   └─ 分发任务：dispatch(CreateBackupJob) → 立即返回成功
   ↓
4. 队列 Worker 执行 CreateBackupJob@handle
   ├─ 调用 BackupConfigurationFactory::make($data)
   │   ├─ 校验 company ID 是否存在
   │   ├─ 校验 file_disk_id 是否存在
   │   ├─ 查找 FileDisk 并动态设置配置
   │   ├─ 设置通知邮箱
   │   └─ 返回 Config 对象
   ├─ 通过 BackupJobFactory 创建备份任务
   ├─ 根据 option 设置备份类型
   ├─ 设置备份文件名
   └─ 执行 $backupJob->run()
   ↓
5. spatie/laravel-backup 内部执行
   ├─ 数据库 dump → 写入临时目录
   ├─ 文件打包 → 写入 ZIP → 临时目录
   ├─ ZIP 加密（如配置）
   └─ 移动到目标磁盘
   ↓
6. 完成或失败
   ├─ 成功 → BackupWasSuccessfulNotification
   └─ 失败 → 重试(如有配置) → BackupHasFailedNotification
```

### 6.2 备份列表查询流程

```
1. 前端调用 GET /api/v1/backups?file_disk_id=xxx
   ↓
2. BackupsController@index
   ├─ 权限校验
   ├─ 动态设置磁盘配置（如有 file_disk_id）
   ├─ 创建 BackupDestination
   ├─ 从缓存读取（4秒缓存）
   │   └─ 遍历备份文件，返回 path/size/created_at
   └─ 返回 JSON 响应
   ↓
3. 异常时降级返回
   └─ 返回空列表 + error 信息
```

---

## 七、关键代码文件索引

| 文件 | 作用 | 关键行数 |
|------|------|---------|
| [config/backup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/config/backup.php) | 备份服务全局配置 | 全文 |
| [app/Jobs/CreateBackupJob.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Jobs/CreateBackupJob.php) | 备份队列任务 | 全文 |
| [app/Space/BackupConfigurationFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) | 动态配置工厂 | 全文 |
| [app/Http/Controllers/V1/Admin/Backup/BackupsController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php) | 备份管理控制器 | 全文 |
| [app/Rules/Backup/PathToZip.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/PathToZip.php) | ZIP路径校验规则 | 全文 |
| [app/Rules/Backup/BackupDisk.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/BackupDisk.php) | 备份磁盘校验规则 | 全文 |
| [app/Rules/Backup/FilesystemDisks.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/FilesystemDisks.php) | 文件系统磁盘校验 | 全文 |
| [tests/Feature/Admin/BackupTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/tests/Feature/Admin/BackupTest.php) | 备份功能测试 | 全文 |

---

## 八、设计特点与注意事项

### 8.1 设计优点

1. **异步队列执行**：避免备份操作阻塞 HTTP 请求
2. **配置驱动**：所有规则通过配置文件管理，灵活易扩展
3. **动态磁盘**：支持多租户/多公司的独立存储配置
4. **完善通知**：成功/失败/健康状态都有邮件通知
5. **缓存优化**：备份列表使用 4 秒缓存，减少磁盘 IO
6. **梯度清理**：按时间梯度保留备份，平衡历史数据与存储空间

### 8.2 注意事项

1. **无显式「回滚」**：备份失败不会「回滚」到之前状态，而是保证不产生坏文件。历史备份始终不受影响。
2. **重试次数为 1**：当前配置 `tries => 1`，即失败后不重试。如需更高可靠性，可调整此配置。
3. **拼写错误**：`BackupConfigurationFactory` 中有 `'The Company ID is missig'` 的拼写错误。
4. **信号禁用**：Windows 环境下禁用了 SIGINT 信号处理（`$backupJob->disableSignals()`）。
5. **缓存时间短**：备份列表只有 4 秒缓存，频繁刷新可能增加磁盘压力。

### 8.3 常见问题解答

**Q: 备份中途失败了怎么办？数据会丢失吗？**
A: 不会。备份使用「先写临时文件，成功后再移动」的策略，失败时临时文件会被清理，已有的历史备份完全不受影响。

**Q: 可以恢复到某个时间点的备份吗？**
A: 备份服务只负责生成备份文件，恢复操作需要手动下载 ZIP 文件并导入数据库/覆盖文件。

**Q: 备份文件存储在哪里？**
A: 默认存储在 `local` 磁盘（`storage/app/`），可通过 `file_disk_id` 参数选择其他已配置的磁盘。
