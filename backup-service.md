# 备份服务代码理解文档

## 一、整体架构概述

本项目的备份服务基于 **spatie/laravel-backup** 第三方包构建，提供数据库和文件的备份、归档功能。整体采用 **队列异步执行 + 配置驱动 + 延迟校验** 的设计模式。

核心组件关系图：

```
前端(BackupSetting.vue)
       ↓ HTTP 请求
BackupsController (权限校验 + 直接分发)
       ↓ dispatch (无参数校验)
CreateBackupJob (队列任务)
       ↓ handle
BackupConfigurationFactory (运行时校验 + 动态配置)
       ↓ createFromConfig
Spatie BackupJob (第三方包核心执行)
       ↓ 归档
ZIP 文件 (存储到目标磁盘)
```

---

## 二、备份触发机制

### 2.1 触发入口

备份有两种触发方式：

#### （1）用户手动触发（主要且唯一的备份发起方式）
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

**关键事实**：
- 该方法**仅做权限校验**，不对任何请求参数进行校验（没有 `$request->validate()`）
- 直接 `$request->all()` 取所有参数，原样传递给队列任务
- 校验延迟到队列任务执行时才进行

#### （2）Cron Webhook 入口（不直接触发备份）
项目提供了一个 Cron 入口路由，但**它不直接触发备份**：

- **路由**：`GET /cron`
- **中间件**：`cron-job`（通过 `x-authorization-token` header 鉴权）
- **控制器**：[CronJobController](file:///d:/fz/0508-2\solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Webhook/CronJobController.php)

```php
public function __invoke(Request $request)
{
    Artisan::call('schedule:run');
    return response()->json(['success' => true]);
}
```

**关键事实**：
- 该路由只是调用 `schedule:run` 运行 Laravel 调度器
- 但 [routes/console.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/routes/console.php) 中**没有定义任何备份相关的调度任务**
- 当前调度器中只有：demo 环境重置、发票/估价状态检查、定期发票生成
- 结论：**当前代码中没有自动定时备份功能**，备份只能由用户手动触发

### 2.2 触发流程详解

1. **权限校验**：使用 `$this->authorize('manage backups')` 验证用户权限
2. **数据组装**：将所有请求数据 + company header 组装
3. **队列分发**：通过 `dispatch()` 将 `CreateBackupJob` 推送到队列
   - 注意：`config('backup.queue.name')` 在 [backup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/config/backup.php) 配置中**不存在**，实际返回 `null`，即使用默认队列
4. **立即返回**：HTTP 请求立即返回成功响应，备份在后台异步执行

> **设计意图**：备份操作可能耗时较长，采用队列异步执行避免请求超时。但参数校验全部后置，意味着调用方无法立即知道参数是否合法。

---

## 三、规则校验机制

### 3.1 校验层级与实际情况

| 层级 | 校验内容 | 实际存在性 | 位置 |
|------|---------|-----------|------|
| 权限层 | 用户是否有备份管理权限 | ✅ 是 | 控制器方法开头 `authorize()` |
| 输入层 | 请求参数格式校验 | ⚠️ 部分有 | 仅 `destroy` 和 `download` 校验 path |
| 业务层 | 业务逻辑校验 | ✅ 是 | 队列执行时由 `BackupConfigurationFactory` 校验 |

### 3.2 已定义但**未被调用**的规则类

项目自定义了三个备份验证规则类，但**只有一个被实际使用**：

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

**实际调用位置**：
- [BackupsController@destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L96-L98) - 删除备份时校验
- [DownloadBackupController](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/DownloadBackupController.php#L20-L22) - 下载备份时校验

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

> **重要事实**：此规则类**从未在代码中被调用**，属于"死代码"。

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

> **重要事实**：此规则类也**从未在代码中被调用**，属于"死代码"。

### 3.3 真正的业务校验：BackupConfigurationFactory

备份参数的实际校验发生在队列执行阶段，由 [BackupConfigurationFactory::make()](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) 进行：

```php
public static function make($data = []): Config
{
    if (blank($data['company'] ?? null)) {
        throw new Exception('The Company ID is missig');
    }

    if (blank($data['file_disk_id'] ?? null)) {
        throw new Exception('No file disk selected');
    }

    $fileDisk = FileDisk::find($data['file_disk_id']);
    // ... 后续配置组装
}
```

**校验内容**：
1. `company` 参数不能为空
2. `file_disk_id` 参数不能为空
3. 通过 `file_disk_id` 能找到有效的 `FileDisk` 记录

> **注意**：此处有拼写错误 `missig` → 应为 `missing`。
> **设计特点**：校验不是在 HTTP 请求阶段进行，而是延迟到队列任务执行时。如果参数错误，队列任务会失败，用户无法从 HTTP 响应中得知。

---

## 四、数据归档规则

### 4.1 归档配置

备份归档的规则在 [config/backup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/config/backup.php) 中定义。

#### （1）备份来源配置

```php
'source' => [
    'files' => [
        'include' => [base_path()],      // 包含的目录（整个项目根目录）
        'exclude' => [                    // 排除的目录
            base_path('vendor'),
            base_path('node_modules'),
            base_path('.git'),
        ],
        'follow_links' => false,          // 是否跟随符号链接
        'ignore_unreadable_directories' => false,
        'relative_path' => null,
    ],
    'databases' => [env('DB_CONNECTION', 'mysql')],  // 要备份的数据库连接
],
```

#### （2）归档目标配置

```php
'destination' => [
    'compression_method' => ZipArchive::CM_DEFAULT,  // ZIP 压缩算法
    'compression_level' => 9,                         // 压缩级别 0-9（9最高）
    'filename_prefix' => '',                          // 文件名前缀
    'disks' => ['local'],                             // 存储磁盘（默认 local）
],
```

#### （3）加密配置

```php
'password' => env('BACKUP_ARCHIVE_PASSWORD'),  // 归档加密密码
'encryption' => 'default',                     // 加密算法（默认 AES-256）
```

#### （4）临时目录

```php
'temporary_directory' => storage_path('app/backup-temp'),
```

### 4.2 备份类型（option 参数）

在 [CreateBackupJob@handle](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Jobs/CreateBackupJob.php#L35-L58) 中，根据 `option` 参数决定备份内容：

| option 值 | 备份内容 | 文件名前缀 |
|-----------|---------|-----------|
| 空值或 `full` | 数据库 + 文件 | 无（由 spatie 包自动命名） |
| `only-db` | 仅数据库 | `only-db-` |
| `only-files` | 仅文件 | `only-files-` |

```php
if ($this->data['option'] === 'only-db') {
    $backupJob->dontBackupFilesystem();  // 跳过文件系统备份
}

if ($this->data['option'] === 'only-files') {
    $backupJob->dontBackupDatabases();   // 跳过数据库备份
}

// 仅当 option 非空时，才设置自定义文件名
if (! empty($this->data['option'])) {
    $prefix = str_replace('_', '-', $this->data['option']).'-';
    $backupJob->setFilename($prefix.date('Y-m-d-H-i-s').'.zip');
}
```

### 4.3 清理策略（归档保留规则）

备份清理使用 `DefaultStrategy` 策略，按时间梯度保留历史备份：

```php
'cleanup' => [
    'strategy' => DefaultStrategy::class,
    'default_strategy' => [
        'keep_all_backups_for_days' => 7,          // 7天内：保留所有备份
        'keep_daily_backups_for_days' => 16,       // 7-16天：每天保留最新1个
        'keep_weekly_backups_for_weeks' => 8,      // 8周内：每周保留最新1个
        'keep_monthly_backups_for_months' => 4,    // 4个月内：每月保留最新1个
        'keep_yearly_backups_for_years' => 2,      // 2年内：每年保留最新1个
        'delete_oldest_backups_when_using_more_megabytes_than' => 5000,  // 超5GB删最老的
    ],
    'tries' => 1,       // 清理失败重试次数
    'retry_delay' => 0, // 重试延迟秒数
],
```

> **重要原则**：默认策略**永远不会删除最新的备份**，确保至少有一个可用备份。
> **重要事实**：项目代码中**没有主动调用清理命令的地方**。清理功能需要通过 `php artisan backup:clean` 手动执行，或在调度器中配置。

### 4.4 动态磁盘配置（多租户支持）

项目支持通过 `file_disk_id` 动态切换备份存储位置。核心逻辑在 [FileDisk](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Models/FileDisk.php) 模型中：

```php
// FileDisk@setConfig
public function setConfig()
{
    $driver = $this->driver;
    $credentials = collect(json_decode($this['credentials']));
    self::setFilesystem($credentials, $driver);
}

// FileDisk@setFilesystem
public static function setFilesystem($credentials, $driver)
{
    $prefix = env('DYNAMIC_DISK_PREFIX', 'temp_');
    config(['filesystems.default' => $prefix.$driver]);

    $disks = config('filesystems.disks.'.$driver);
    foreach ($disks as $key => $value) {
        if ($credentials->has($key)) {
            $disks[$key] = $credentials[$key];
        }
    }
    config(['filesystems.disks.'.$prefix.$driver => $disks]);
}
```

**工作流程**：
1. 从数据库的 `file_disks` 表读取磁盘配置（driver、credentials）
2. 以配置文件中的磁盘模板为基础，用数据库中存储的凭证覆盖
3. 动态注册一个新的文件系统磁盘（前缀 `temp_` + driver 名）
4. 将其设为默认文件系统
5. 备份目标磁盘也设置为这个动态磁盘

**在 BackupConfigurationFactory 中的调用**：
```php
$fileDisk = FileDisk::find($data['file_disk_id']);
$fileDisk->setConfig();

$prefix = env('DYNAMIC_DISK_PREFIX', 'temp_');
config(['backup.backup.destination.disks' => [$prefix.$fileDisk->driver]]);
```

---

## 五、异常处理与「回滚」机制

### 5.1 异常处理层次

备份服务的异常处理分布在三个层面：

#### （1）HTTP 层 - 列表查询异常捕获
[BackupsController@index](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L24-L68)

```php
try {
    // ... 读取备份列表
} catch (\Exception $e) {
    return response()->json([
        'backups' => [],
        'error' => 'invalid_disk_credentials',
        'error_message' => $e->getMessage(),
        'disks' => $configuredBackupDisks,
    ]);
}
```

**适用场景**：获取备份列表时磁盘连接失败（如凭证错误、网络不通）
**处理方式**：降级返回空列表 + 错误信息，前端可展示错误提示

#### （2）队列层 - 任务执行异常
备份任务在队列中执行时发生异常，由 Laravel 队列系统和 spatie/laravel-backup 共同处理：

**Laravel 队列层面**：
- 任务异常 → 标记为失败 → 存入 `failed_jobs` 表
- 可通过 `php artisan queue:failed` 查看失败任务
- 可通过 `php artisan queue:retry` 重试

**spatie/laravel-backup 内置重试**：
```php
// config/backup.php
'backup' => [
    'tries' => 1,       // 备份操作内部重试次数（当前为1，即不重试）
    'retry_delay' => 0, // 每次重试间隔秒数
],
```

> **注意**：这个 `tries` 是 spatie 包内部的重试机制，不是 Laravel 队列的重试。

#### （3）通知层 - 异常告警
当备份/清理失败时，spatie/laravel-backup 会自动发送通知：

```php
'notifications' => [
    'notifications' => [
        BackupHasFailedNotification::class => ['mail'],        // 备份失败
        CleanupHasFailedNotification::class => ['mail'],       // 清理失败
        UnhealthyBackupWasFoundNotification::class => ['mail'], // 备份不健康
        BackupWasSuccessfulNotification::class => ['mail'],     // 备份成功
        HealthyBackupWasFoundNotification::class => ['mail'],   // 备份健康
        CleanupWasSuccessfulNotification::class => ['mail'],    // 清理成功
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

### 5.2 「回滚」机制深度解析

> **重要澄清**：本备份服务**没有传统数据库事务意义上的「回滚」**。备份是一个生成 ZIP 文件的过程，失败时的保障体现在「不产生坏文件」和「不影响历史」两个方面。

#### （1）临时文件机制 = 隐式「回滚」
备份过程采用「先写临时目录，成功后移动」的策略：

1. 所有备份文件先写入 `temporary_directory`（默认：`storage/app/backup-temp`）
2. 数据库 dump 完成 → 写入临时目录
3. 文件打包完成 → 写入临时 ZIP 文件
4. 加密完成 → 仍在临时目录
5. **全部成功后** → 将 ZIP 移动到目标磁盘
6. **任何一步失败** → 临时文件被清理，目标磁盘不会出现不完整文件

这相当于一种「全有或全无」的原子性保障。

#### （2）历史备份不受影响
- 新备份失败时，**绝对不会删除或修改任何已有的成功备份**
- 这是一种「前向容错」设计：最坏情况只是本次备份失败，历史备份始终完好
- 即使新备份覆盖同名文件的情况也不存在，因为每个备份文件名都带时间戳

#### （3）失败后的状态
备份失败后系统的状态：
- **历史备份**：完整保留，可正常使用
- **正在创建的备份**：不完整的临时文件被清理，目标磁盘上没有残留
- **队列任务**：标记为 failed，存入 failed_jobs 表
- **通知**：发送失败邮件通知管理员
- **用户**：无法从 HTTP 响应感知失败（因为是异步队列），需要刷新列表或查邮件

#### （4）关于「自动重试」
- 当前配置 `tries => 1`，即 spatie 内部不做重试
- Laravel 队列层面也没有配置重试次数，默认执行一次后失败
- 如需自动重试，需要修改配置或在队列层面设置

### 5.3 监控与健康检查

备份服务内置健康检查机制，通过 `php artisan backup:monitor-healthy` 和 `backup:monitor-unhealthy` 命令运行：

```php
'monitor_backups' => [
    [
        'name' => env('APP_NAME', 'laravel-backup'),
        'disks' => ['local'],
        'health_checks' => [
            MaximumAgeInDays::class => 1,      // 最新备份不能超过1天
            MaximumStorageInMegabytes::class => 5000,  // 总存储不超过5GB
        ],
    ],
],
```

不健康时触发 `UnhealthyBackupWasFoundNotification` 通知。

> **注意**：项目代码中**没有定时调用这些检查命令**，健康检查功能需要手动触发或自行配置调度。

---

## 六、触发、校验、归档与异常处理的协作机制

### 6.1 完整备份执行时序

```
时间轴 →
│
├─ HTTP 请求阶段（同步）
│  ┌─────────────────────────────────────────┐
│  │ 1. 用户点击「新建备份」                   │
│  │ 2. POST /api/v1/backups                 │
│  │ 3. BackupsController@store              │
│  │    ├─ authorize('manage backups')       │ ← 权限校验（唯一的前置校验）
│  │    ├─ $request->all() 取所有参数        │ ← 无参数校验！
│  │    ├─ 添加 company header               │
│  │    └─ dispatch(CreateBackupJob)         │ ← 推入队列
│  │ 4. 返回 { success: true }               │
│  └─────────────────────────────────────────┘
│
├─ 队列执行阶段（异步，延迟发生）
│  ┌─────────────────────────────────────────┐
│  │ 5. CreateBackupJob@handle                │
│  │    ├─ BackupConfigurationFactory::make() │
│  │    │  ├─ 校验 company 不为空             │ ← 业务校验1
│  │    │  ├─ 校验 file_disk_id 不为空        │ ← 业务校验2
│  │    │  ├─ 查找 FileDisk 记录              │ ← 业务校验3
│  │    │  ├─ 动态设置文件系统配置             │
│  │    │  ├─ 设置通知邮箱                    │
│  │    │  └─ 返回 Config 对象               │
│  │    ├─ BackupJobFactory::createFromConfig │ ← 创建备份任务
│  │    ├─ 设置备份类型(only-db/only-files)   │
│  │    ├─ 设置文件名                         │
│  │    └─ $backupJob->run()                  │ ← 执行备份
│  └─────────────────────────────────────────┘
│
├─ spatie/laravel-backup 内部执行
│  ┌─────────────────────────────────────────┐
│  │ 6. 数据库 dump → 临时目录                │
│  │ 7. 文件收集 → 写入临时 ZIP               │
│  │ 8. ZIP 加密（如配置）                    │
│  │ 9. 移动到目标磁盘                        │ ← 只有全部成功才到这一步
│  └─────────────────────────────────────────┘
│
├─ 结果处理
│  ┌─────────────┐    ┌──────────────┐
│  │ 成功        │    │ 失败         │
│  │  ├─ 新备份出现在磁盘 │    │  ├─ 临时文件被清理│
│  │  └─ 发送成功通知  │    │  ├─ 发送失败通知  │
│  └─────────────┘    │  └─ 队列任务失败  │
│                     └──────────────┘
```

### 6.2 协作机制的关键特点

| 阶段 | 在哪里执行 | 有什么校验 | 失败怎么办 | 用户能否感知 |
|------|-----------|-----------|-----------|-------------|
| HTTP 请求 | 控制器同步 | 仅权限校验 | 直接返回错误 | ✅ 能 |
| 队列任务启动 | 队列 Worker | company、file_disk_id 校验 | 任务失败 + 通知 | ❌ 不能（需刷新或看邮件） |
| 备份执行 | 队列 Worker | spatie 内部完整性检查 | 任务失败 + 通知 + 清理临时文件 | ❌ 不能 |

### 6.3 设计权衡分析

**优点**：
- HTTP 响应快，用户无需等待备份完成
- 备份失败不会影响用户的其他操作
- 临时文件机制保证不产生坏文件

**缺点**：
- 参数校验后置，用户无法立即知道参数错误
- 备份失败后用户没有直观的反馈渠道
- 两个校验规则类（BackupDisk、FilesystemDisks）写了但没用，代码冗余
- 没有自动备份调度，全靠手动
- 没有自动清理调度，旧备份会越积越多

---

## 七、关键代码文件索引

| 文件 | 作用 | 关键行 |
|------|------|--------|
| [config/backup.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/config/backup.php) | 备份服务全局配置 | 全文 |
| [app/Jobs/CreateBackupJob.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Jobs/CreateBackupJob.php) | 备份队列任务 | 全文 |
| [app/Space/BackupConfigurationFactory.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) | 动态配置工厂 + 业务校验 | 全文 |
| [app/Http/Controllers/V1/Admin/Backup/BackupsController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php) | 备份管理控制器 | 全文 |
| [app/Http/Controllers/V1/Admin/Backup/DownloadBackupController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/DownloadBackupController.php) | 备份下载控制器 | 全文 |
| [app/Rules/Backup/PathToZip.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/PathToZip.php) | ZIP路径校验（实际使用） | 全文 |
| [app/Rules/Backup/BackupDisk.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/BackupDisk.php) | 备份磁盘校验（**未使用**） | 全文 |
| [app/Rules/Backup/FilesystemDisks.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Rules/Backup/FilesystemDisks.php) | 文件系统磁盘校验（**未使用**） | 全文 |
| [app/Models/FileDisk.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Models/FileDisk.php) | 文件磁盘模型（动态配置） | 83-112 |
| [app/Http/Controllers/V1/Webhook/CronJobController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Webhook/CronJobController.php) | Cron Webhook 入口 | 全文 |
| [routes/console.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/routes/console.php) | 调度任务定义（无备份任务） | 全文 |
| [tests/Feature/Admin/BackupTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/tests/Feature/Admin/BackupTest.php) | 备份功能测试 | 全文 |

---

## 八、常见问题解答

**Q: 备份创建接口有参数校验吗？**
A: HTTP 层面没有。`BackupsController@store` 只做权限校验，参数原样丢给队列。真正的参数校验在 `BackupConfigurationFactory` 中，是队列执行时才做的。

**Q: BackupDisk 和 FilesystemDisks 这两个规则在哪里用到？**
A: 实际上**没有被用到**。代码库中定义了这三个规则类，但只有 `PathToZip` 在删除和下载时被调用。另外两个是「死代码」。

**Q: 有自动定时备份吗？**
A: 没有。虽然有 `GET /cron` 路由和调度器，但 `routes/console.php` 中没有定义任何备份相关的调度任务。备份只能手动触发。

**Q: 备份失败了会回滚吗？历史备份会受影响吗？**
A: 没有传统意义上的「回滚」。但失败时：1) 不完整的临时文件会被清理，不会产生坏文件 2) 历史备份完好无损，绝对不会被删除或修改。

**Q: 备份失败了用户怎么知道？**
A: 两种方式：1) 刷新备份列表看有没有新备份 2) 配置了通知邮箱的话会收到失败邮件。HTTP 请求层面是感知不到的，因为任务是异步的。

**Q: 备份文件会自动清理吗？**
A: 不会自动清理。虽然配置了清理策略，但项目代码中没有调用 `backup:clean` 命令的地方，也没有在调度器中配置。需要手动执行或自行添加调度。

**Q: `config('backup.queue.name')` 配置在哪里？**
A: 这个配置项不存在。`backup.php` 中没有 `queue.name` 配置，调用它会返回 `null`，导致任务被推送到默认队列。
