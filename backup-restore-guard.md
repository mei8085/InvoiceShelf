# 备份恢复防护机制 - 代码协作分析

## 一、系统架构总览

InvoiceShelf 基于 `spatie/laravel-backup ^10.0` 构建备份系统，由三块核心逻辑协同工作：
**数据库快照**、**附件归集**、**还原校验/防护**。三者通过配置驱动、队列异步、动态磁盘注册等机制形成完整的备份恢复防护链。

---

## 二、数据库快照 (Database Snapshot)

### 2.1 核心代码位置

| 文件 | 说明 |
|------|------|
| [config/backup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L61-L94) | 数据库备份配置 |
| [CreateBackupJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php#L43-L49) | 数据库/文件备份开关 |
| [config/database.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/database.php#L32-L116) | 数据库连接配置 |

### 2.2 快照流程

```
请求发起 → 队列分发 → 动态配置 → DB Dump → 压缩 → 入包
```

**关键代码逻辑**：

1. **连接选择**：默认使用 `env('DB_CONNECTION', 'mysql')`，支持 MySQL/MariaDB/PostgreSQL/SQLite/SQL Server

2. **导出引擎**：由 `spatie/db-dumper` 提供底层支持
   - MySQL: `mysqldump`
   - PostgreSQL: `pg_dump`
   - SQLite: 直接复制数据库文件
   - 可通过连接配置的 `dump` 键自定义排除表等参数

3. **导出文件**：
   - 文件名基类: `database` (可配置为 `connection`)
   - 默认扩展名: `.sql`
   - 可选时间戳后缀 (配置 `database_dump_file_timestamp_format`)

4. **压缩选项**：
   - 数据库级压缩: `database_dump_compressor` (GzipCompressor)
   - ZIP 包级压缩: 见「压缩流水线」章节

5. **快照开关控制** [CreateBackupJob.php#L43-L49](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php#L43-L49):
   ```php
   if ($this->data['option'] === 'only-db') {
       $backupJob->dontBackupFilesystem();  // 仅数据库
   }
   if ($this->data['option'] === 'only-files') {
       $backupJob->dontBackupDatabases();   // 仅文件
   }
   ```

---

## 三、附件归集 (File Collection)

### 3.1 核心代码位置

| 文件 | 说明 |
|------|------|
| [config/backup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L23-L59) | 文件包含/排除配置 |
| [FileDiskService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/FileDiskService.php#L64-L103) | 动态磁盘注册 |
| [BackupConfigurationFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/BackupConfigurationFactory.php#L11-L24) | 备份配置工厂 |
| [config/filesystems.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/filesystems.php#L88-L91) | 媒体磁盘配置 |

### 3.2 归集范围

**包含目录** [config/backup.php#L28-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L28-L30):
```php
'include' => [
    base_path(),  // 整个项目根目录
],
```

**排除目录** [config/backup.php#L37-L41](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L37-L41):
```php
'exclude' => [
    base_path('vendor'),
    base_path('node_modules'),
    base_path('.git'),
],
```

> **自动排除**：备份过程使用的临时目录 (`storage/app/backup-temp`) 会被自动排除。

### 3.3 媒体文件存储

附件（发票附件、支出收据、公司Logo等）通过 `spatie/laravel-medialibrary` 管理：

1. **默认存储位置**: `public_path('media')` [filesystems.php#L88-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/filesystems.php#L88-L91)
2. **可配置存储**: 通过 `FileDisk` 模型支持多种驱动
   - Local / S3 / Dropbox / DigitalOcean Spaces
   - 管理员可在「Admin → File Disks → Disk Assignments」分配用途

### 3.4 动态磁盘注册机制

备份目标磁盘在运行时动态注册，支持多租户隔离：

**流程** [BackupConfigurationFactory.php#L11-L24](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/BackupConfigurationFactory.php#L11-L24):

```
接收 file_disk_id → 查找 FileDisk 模型 → FileDiskService::registerDisk()
    → 注入到 filesystems.disks 配置 → 设置 backup.destination.disks
    → 生成 Config 对象
```

**关键代码** [FileDiskService.php#L77-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/FileDiskService.php#L77-L103):
```php
public function registerDisk(FileDisk $disk): string
{
    $diskName = $this->getDiskName($disk);  // disk_{id}
    
    // 系统磁盘已预配置，直接返回
    if ($disk->isSystem()) {
        return $diskName;
    }
    
    // 动态注入配置
    $baseConfig = config('filesystems.disks.'.$disk->driver, []);
    // 合并凭证...
    config(['filesystems.disks.'.$diskName => $baseConfig]);
    
    return $diskName;
}
```

---

## 四、压缩流水线 (Compression Pipeline)

### 4.1 核心代码位置

| 文件 | 说明 |
|------|------|
| [config/backup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L130-L168) | 压缩配置 |
| [config/backup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L170-L188) | 加密配置 |

### 4.2 流水线步骤

```
DB Dump (可选gzip) → 文件收集 → ZIP打包 → AES-256加密 → 写入磁盘
```

**压缩参数** [config/backup.php#L145-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L145-L155):
```php
'compression_method' => ZipArchive::CM_DEFAULT,
'compression_level' => 9,  // 最高压缩级别
```

**加密参数** [config/backup.php#L179-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L179-L188):
```php
'password' => env('BACKUP_ARCHIVE_PASSWORD'),
'encryption' => 'default',  // ZipArchive::EM_AES_256
```

### 4.3 临时文件管理与递归打包防护

**核心问题**：为什么 temporary_directory 必须被排除，否则会发生什么？

#### 4.3.1 递归打包的真实场景

备份的归集范围配置如下 [config/backup.php#L28-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L28-L30)：

```php
'include' => [
    base_path(),   // = D:\fz\0601-2\solo-dogfeeding\code\12-InvoiceShelf
],
```

而临时打包目录的配置 [config/backup.php#L173](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L173)：

```php
'temporary_directory' => storage_path('app/backup-temp'),
// = D:\fz\0601-2\solo-dogfeeding\code\12-InvoiceShelf\storage\app\backup-temp
```

注意：`storage_path()` 展开后是 `base_path() . '/storage/...'`，也就是说 **temporary_directory 是 include 目录的子目录**。

#### 4.3.2 如果不排除会发生什么 — ZIP 递归膨胀路径

假设临时目录不被排除，BackupJob 的执行时序如下：

```
时间 t=0:
  include: base_path()  ← 整个项目根
  exclude: vendor, node_modules, .git  ← 无 temporary_directory

时间 t=1: BackupJob 开始收集文件
  遍历 base_path() 下所有文件和子目录
    ↓
  进入 storage/
    ↓
  进入 storage/app/
    ↓
  进入 storage/app/backup-temp/   ← 注意：这个目录此时是空的，还没创建 ZIP
    ↓
  没问题，backup-temp 是空目录，跳过

时间 t=2: 收集完文件后开始写 ZIP
  在 storage/app/backup-temp/ 下创建：
    2024-06-17-15-30-00.zip  ← 正在写入，大小从 0 开始增长

时间 t=3: 问题来了 —— spatie 的 FileSelection 是流式遍历
  某些场景下（第二次遍历、或 include 路径包含 backup-temp 的绝对路径）
  会再次扫描到 backup-temp/ 目录
    ↓
  发现 2024-06-17-15-30-00.zip 这个文件
    ↓
  把这个 ZIP 文件当作普通文件，打包进 ZIP 文件自身！

时间 t=4: 递归膨胀
  ZIP 正在写入的大小 = 已收集文件 + 这个 ZIP 自身的当前大小
    ↓
  每次 ZIP 增长一点，自身就多包含一点
    ↓
  大小指数级膨胀：1MB → 2MB → 4MB → 8MB → 16MB ...
    ↓
  直到：磁盘满 / PHP 内存耗尽 / 超时强制终止 / ZipArchive 报错
```

**这不是理论问题**。早期版本的 spatie/backup 就出过这个 bug — 项目包含自身目录时 ZIP 无限膨胀。正因为如此，spatie 现在在代码层面硬编码了 temporary_directory 的自动排除。

#### 4.3.3 include / exclude / 自动排除的优先级

spatie/laravel-backup v10 的 `FileSelection` 类中，三类排除项的**处理顺序**（优先级从低到高）：

```
优先级 1（最容易被覆盖）: 用户配置的 include
          ↓
优先级 2: 用户配置的 exclude  [config/backup.php#L37-L41]
          ← vendor / node_modules / .git
          ↓
优先级 3（最强，无法覆盖）: spatie 自动追加的 exclude
          ← temporary_directory
          ← 目标磁盘上已有的备份文件路径（防重复打包）
```

即使用户把 `storage/app/backup-temp` 手动加到 `include` 里也没用 — 优先级 3 的自动排除会强制把它从最终列表中剔除。

配置注释也明确说明了这一点 [config/backup.php#L35-L36](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L35-L36)：

> Directories used by the backup process will automatically be excluded.

#### 4.3.4 实际排除链路

```
BackupJob::run()
  → FileSelection::create()
    → 加载 config('backup.backup.source.files.include')
    → 加载 config('backup.backup.source.files.exclude')
    → 追加 temporary_directory 到 exclude
    → 追加 destination disks 的备份目录路径到 exclude
  → FileSelection 通过 Symfony Finder 遍历
    → Finder::exclude() 应用目录排除规则
    → 每个文件判断是否在排除路径下，是则跳过
  → 收集通过的文件列表
  → Zip::create() 写入 temporary_directory
  → 写入完成后，把 ZIP 从 temporary_directory 移动到目标磁盘
  → 删除 temporary_directory 中的残留文件
```

#### 4.3.5 三层防护总结

| 层 | 机制 | 代码位置 | 防护 |
|----|------|---------|------|
| 1 | spatie 自动排除 `temporary_directory` | BackupJob → FileSelection（spatie 源码硬编码） | 正在写入的 ZIP 不会被重新打包 |
| 2 | spatie 自动排除目标磁盘的备份目录 | BackupJob → FileSelection（spatie 源码硬编码） | 已有的旧备份 ZIP 不会被重新打包 |
| 3 | 用户配置排除 vendor/node_modules/.git | [config/backup.php#L37-L41](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L37-L41) | 大型依赖目录不会被包含 |

> **最终保证**：无论如何配置，第 1 层的自动排除是硬编码在 spatie/backup v10 源码中的，无法关闭。即使用户想递归也做不到。

---

### 4.4 备份任务信号开关与 Queue Worker 设计

**核心代码** [CreateBackupJob.php#L39-L41](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php#L39-L41)：

```php
if (! defined('SIGINT')) {
    $backupJob->disableSignals();
}
```

这段三行代码解决了**两层冲突问题**。下面逐层拆解。

#### 4.4.1 第一层冲突：spatie 信号处理器 vs Laravel Queue Worker 信号处理器

**spatie/backup 为什么要注册信号？**

spatie/laravel-backup v10 依赖 `spatie/laravel-signal-aware-command ^2.1`（实际版本 v2.1.2 [composer.lock#L5544-L5548](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/composer.lock#L5544-L5548)）。

这个包的设计目标是：在 **`php artisan backup:run` 命令行直接执行**时，处理 Ctrl+C（SIGINT）和 `kill`（SIGTERM），做到优雅中断：

```
用户在终端按 Ctrl+C
  → 系统向 artisan 进程发 SIGINT
  → signal-aware-command 的处理器被触发
  → BackupJob 收到信号
  → 停止收集文件，关闭 ZipArchive
  → 清理 temporary_directory 中的半写文件
  → 正常退出，返回非 0 状态码
  → 触发 BackupHasFailedNotification
```

这在命令行下是正确的行为。

**问题出在 queue worker 场景**：

当备份任务通过 `dispatch(new CreateBackupJob())` 投递给队列后，执行环境变了：

```
php artisan queue:work database
  → queue worker 启动，常驻内存
  → worker 自己用 pcntl_signal() 注册了 SIGINT / SIGTERM / SIGALRM 处理器
      • SIGINT / SIGTERM: 响应 queue:restart、queue:pause
      • SIGALRM:          实现 retry_after 超时机制
  → worker 取出 CreateBackupJob
  → 执行 handle()
  → BackupJobFactory::createFromConfig()
  → 如果不 disableSignals()，spatie 会再次调用 pcntl_signal()
      ↓
  此时发生：后注册的信号处理器 **覆盖** 先注册的
  ↓
  worker 原有的 SIGINT / SIGTERM 处理器被 BackupJob 的处理器替换！
```

**替换后的后果（极其严重）**：

| 操作 | 正常情况（worker 自己处理） | 冲突后（BackupJob 覆盖） |
|------|--------------------------|------------------------|
| `php artisan queue:restart` | worker 保存当前 Job 状态，释放回队列，然后退出 | BackupJob 的处理器收到 SIGINT，认为是"中断备份"，清理临时文件，Job 失败 |
| Worker 超时（retry_after=90s） | Worker 释放 Job，标记为可重试 | 超时由 SIGALRM 触发，BackupJob 没处理 → Worker 的 SIGALRM 被覆盖 → 超时失效，任务永远卡死 |
| 正常备份完成 | BackupJob 结束，Worker 取下一个 Job | 无问题 |

这就是为什么 `disableSignals()` 必须在 queue worker 场景下被调用。

#### 4.4.2 第二层冲突：跨平台差异

PHP 的 POSIX 信号（`pcntl_signal`、`SIGINT` 常量）只在**满足以下全部条件**时可用：

1. 操作系统是 Unix/Linux/macOS（不是 Windows）
2. PHP 编译时启用了 `--enable-pcntl`（CLI 模式一般默认启用，FPM 模式禁用）
3. 运行方式是 CLI（不是 FPM / mod_php）

在 **Windows** 环境下：
- `pcntl` 扩展根本不存在
- `SIGINT`、`SIGTERM` 常量**未定义**
- 如果 spatie/backup 内部执行类似 `if ($signal === SIGINT)` 的比较，会直接抛出 `Undefined constant SIGINT` 的 Fatal Error

所以代码用了 `defined('SIGINT')` 做判断，而不是判断操作系统或扩展是否存在。`defined()` 是最准确的判断方式 —— 因为即使在 Linux 上，如果 PHP 没启用 pcntl，`SIGINT` 同样未定义。

#### 4.4.3 判断逻辑的真实含义

```php
if (! defined('SIGINT')) {          // 信号常量不存在？
    $backupJob->disableSignals();   // 是 → 禁用 spatie 的信号注册
}
```

完整的决策表：

| 环境 | `SIGINT` 定义？ | `disableSignals()` 调用？ | 实际行为 |
|------|----------------|-------------------------|---------|
| Linux/macOS CLI + pcntl | 已定义 | **不调用** | BackupJob 注册自己的信号处理器；**注意 — 仍可能与 Worker 冲突**（见下方说明） |
| Linux CLI 无 pcntl | 未定义 | 调用 | BackupJob 不注册信号，Worker 自己处理 |
| Windows CLI | 未定义 | 调用 | BackupJob 不注册信号，Worker 用超时机制终止 |
| `sync` 队列（同步） | Linux=是 Windows=否 | 同上 | sync 无 Worker，spatie 信号正常工作 / 不注册 |

#### 4.4.4 残留问题：Linux 下信号冲突仍然可能发生

注意代码只判断了 `defined('SIGINT')`，**没有判断当前是否在 queue worker 中执行**。这意味着：

**Linux + pcntl + database/redis 队列**的场景下，即使运行在 queue worker 中，`SIGINT` 也是定义的，`disableSignals()` **不会被调用**，spatie 的信号处理器还是会注册，可能覆盖 Worker 的处理器。

这是一个**潜在的设计缺陷**，但实际影响较小，原因是：

1. **InvoiceShelf 默认队列是 `sync`** [config/queue.php#L16](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/queue.php#L16)：`env('QUEUE_CONNECTION', 'sync')`
   - `sync` 模式下，dispatch 后同步执行，没有独立的 worker 进程
   - spatie 注册信号处理器是正确的行为

2. 如果运维切换到 `database`/`redis` 异步队列：
   - 风险真实存在：`queue:restart` 可能被 BackupJob 的信号处理器当成"中断备份"
   - 但备份成功/失败有通知（见第六章），运维能感知到异常
   - 要彻底修复，应改为判断 `! $this->job`（即 Job 不是通过队列执行），或者直接永远 `disableSignals()`，交给 Worker 统一处理

> **设计权衡**：当前写法是"能用就用，不能用就禁"的保守策略 — 在支持信号的环境下尽量给备份优雅中断的能力，代价是异步队列场景下有潜在冲突。这个权衡成立的前提是项目默认用 `sync` 队列。

#### 4.4.5 两种中断方式的对比

| 中断方式 | 触发条件 | 谁来清理临时文件 | Job 最终状态 |
|---------|---------|----------------|------------|
| spatie 优雅中断 | Linux + 信号可用 + 收到 SIGINT/SIGTERM | BackupJob 的信号处理器主动删除 temporary_directory | `BackupHasFailed`，有通知 |
| Worker 超时终止 | 运行超过 `retry_after`（默认 90s [config/queue.php#L42](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/queue.php#L42)） | Worker 释放 Job，临时文件**可能残留** | Job 被放回队列，可重试；仍失败则写入 `failed_jobs` 表 |

**残留临时文件的风险**：两种方式都不会造成功能性问题 — 下一次备份执行时会重新创建 `temporary_directory`，旧的残留文件会被覆盖或被 `backup:clean` 清理任务处理。

---

## 五、备份调度 (Backup Scheduling)

### 5.1 调度入口

| 文件 | 说明 |
|------|------|
| [routes/console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/console.php#L1-L32) | 调度配置 |
| [CronJobController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Webhook/CronJobController.php#L17-L22) | Webhook触发 |
| [CronJobMiddleware.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Middleware/CronJobMiddleware.php#L16-L23) | 调度鉴权 |

### 5.2 调度方式

1. **Webhook 触发** [routes/api.php#L581](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/api.php#L581):
   ```
   GET /cron → CronJobController → Artisan::call('schedule:run')
   ```

2. **鉴权机制** [CronJobMiddleware.php#L18-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Middleware/CronJobMiddleware.php#L18-L19):
   ```php
   if ($request->header('x-authorization-token') == config('services.cron_job.auth_token')) {
       return $next($request);
   }
   ```

3. **当前调度任务** [console.php#L10-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/console.php#L10-L31):
   - Demo环境: 每日 `reset:app --force`
   - 发票/估价状态检查: 每日执行
   - 定期发票: 按各公司配置的 cron 表达式执行

> **注意**：当前代码库**未配置自动备份调度**，备份仅支持手动触发。如需自动备份，需在 `console.php` 中添加：
> ```php
> Schedule::command('backup:run')->daily();
> Schedule::command('backup:clean')->daily();
> ```

### 5.3 手动备份流程

```
AdminBackupView.vue → backupService.create() 
    → BackupsController@store → dispatch CreateBackupJob
    → 队列执行 → spatie/backup 核心逻辑
```

**关键代码** [BackupsController.php#L52-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Admin/BackupsController.php#L52-L61):
```php
public function store(Request $request): JsonResponse
{
    $this->authorize('manage backups');
    $data = $request->all();
    dispatch(new CreateBackupJob($data))
        ->onQueue(config('backup.queue.name'));
    return response()->json(['success' => true]);
}
```

---

## 六、监控健康检查与通知

### 6.1 健康检查：备份是否还"活着"

spatie/backup 在每次 `backup:run` 或 `backup:monitor` 执行时，会遍历 `monitor_backups` 配置中声明的磁盘，对每个磁盘上的备份做两项检查：

**配置入口** [config/backup.php#L267-L276](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L267-L276):
```php
'monitor_backups' => [
    [
        'name' => env('APP_NAME', 'laravel-backup'),
        'disks' => ['local'],
        'health_checks' => [
            MaximumAgeInDays::class => 1,
            MaximumStorageInMegabytes::class => 5000,
        ],
    ],
],
```

**检查逻辑**：

| 检查项 | 默认阈值 | 含义 |
|--------|---------|------|
| `MaximumAgeInDays` | 1 天 | 最新的备份不能超过 1 天前，否则视为"过期" |
| `MaximumStorageInMegabytes` | 5000 MB | 所有备份总大小不能超过 5 GB |

**判定流程**：

```
spatie 运行监控 → 遍历 monitor_backups 条目
  → 对每个磁盘读取备份列表 → 逐项检查 MaximumAgeInDays / MaximumStorageInMegabytes
    → 全部通过 → 触发 HealthyBackupWasFoundNotification
    → 任一不通过 → 触发 UnhealthyBackupWasFoundNotification
```

### 6.2 通知：备份出了什么事，谁会知道

spatie/backup 在 6 个事件点发出通知，每个通知可投递到 mail / slack / discord 三个渠道：

**配置入口** [config/backup.php#L209-L217](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L209-L217):
```php
'notifications' => [
    'notifications' => [
        BackupHasFailedNotification::class          => ['mail'],
        UnhealthyBackupWasFoundNotification::class  => ['mail'],
        CleanupHasFailedNotification::class         => ['mail'],
        BackupWasSuccessfulNotification::class      => ['mail'],
        HealthyBackupWasFoundNotification::class    => ['mail'],
        CleanupWasSuccessfulNotification::class     => ['mail'],
    ],
],
```

**6 种通知的触发场景**：

| 通知类 | 触发时机 | 严重程度 |
|--------|---------|----------|
| `BackupHasFailedNotification` | 备份过程中抛出异常 | 🔴 关键 |
| `UnhealthyBackupWasFoundNotification` | 健康检查不通过（过期/超量） | 🟡 警告 |
| `CleanupHasFailedNotification` | 清理旧备份时异常 | 🟡 警告 |
| `BackupWasSuccessfulNotification` | 备份成功完成 | 🟢 正常 |
| `HealthyBackupWasFoundNotification` | 健康检查通过 | 🟢 正常 |
| `CleanupWasSuccessfulNotification` | 清理完成 | 🟢 正常 |

**投递链路**：

```
事件触发 → spatie 创建 Notification 实例
  → 通过 Notifiable 类（默认）发送
    → 按 notifications 配置的渠道分发
      → mail:    发到 config('backup.notifications.mail.to')
      → slack:   发到 config('backup.notifications.slack.webhook_url')
      → discord: 发到 config('backup.notifications.discord.webhook_url')
```

> **当前状态**：mail.to 仍为 `your@example.com` 示例值，slack/discord webhook 均为空字符串。需要运维人员配置真实值后通知才会实际生效。

### 6.3 队列容错：备份任务挂了怎么办

`CreateBackupJob` 实现了 `ShouldQueue` 接口 [CreateBackupJob.php#L13](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php#L13)，由 Laravel 队列系统驱动。

**容错机制分两层**：

**第一层：spatie/backup 自身配置** [config/backup.php#L190-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L190-L199)：
```php
'tries' => 1,          // 备份命令遇到异常时的重试次数
'retry_delay' => 0,    // 重试间隔（秒）
```
清理任务同理 [config/backup.php#L344-L350](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L344-L350)。

**第二层：Laravel 队列系统**：

```
BackupsController@store
  → dispatch(new CreateBackupJob($data))
      ->onQueue(config('backup.queue.name'))
```

- 默认队列连接: `env('QUEUE_CONNECTION', 'sync')` [config/queue.php#L16](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/queue.php#L16)
- `sync` 连接 = 同步执行，无重试
- 若切换为 `database` / `redis` 等异步连接，Laravel 队列的 `retry_after`、`tries` 等机制自动生效

**两个 `tries` 的区别**：
- `config/backup.tries`：spatie 内部 artisan 命令级重试，控制的是 `backup:run` / `backup:clean` 命令本身
- Laravel 队列的 `$tries`：控制的是 `CreateBackupJob` 这个 Job 被队列 worker 重新执行的次数

**失败信号流转**：

```
备份失败 → spatie 内部 tries=1 不重试
  → 触发 BackupHasFailedNotification (mail)
  → Job 在队列中标记为 failed
  → 若配置了异步队列且有 tries > 1 → worker 自动重试
  → 仍失败 → 写入 failed_jobs 表
```

---

## 七、凭证校验与 SSRF 防护链路

### 7.1 问题：管理员配置备份磁盘时，endpoint 字段可以填什么？

S3 兼容存储和 DigitalOcean Spaces 都需要 `endpoint` 字段。如果攻击者（或被入侵的管理员账号）将 endpoint 设为 `http://169.254.169.254`（云元数据服务），服务器就会带着 S3 凭证去请求内部网络，这就是 SSRF。

### 7.2 两道拦截的完整代码链路

**第一道：表单验证时拦截（保存前）**

```
前端提交磁盘配置
  → DiskEnvironmentRequest::rules()
    → credentials.endpoint 字段挂了 PublicHttpUrl 规则
      → PublicHttpUrl::validate()
        → PrivateNetworkGuard::blockedReason($url)
          → 解析 URL → 检查 scheme (仅 http/https)
            → 提取 host → filter_var 是否 IP 直连
              → 是: ipIsBlocked() 直接检查
              → 否: resolveHost() 做 DNS 查询 → 逐 IP 检查
            → 返回 blocked 原因 或 null (允许)
          → blocked !== null → $fail() 拒绝
```

关键代码 [DiskEnvironmentRequest.php#L43-L48](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Requests/DiskEnvironmentRequest.php#L43-L48):
```php
'credentials.endpoint' => [
    'nullable',
    'string',
    'url',
    new PublicHttpUrl,   // ← SSRF 第一道拦截
],
```

[PublicHttpUrl.php#L21-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/PublicHttpUrl.php#L21-L29):
```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    if (! is_string($value) || trim($value) === '') {
        return;
    }
    if (PrivateNetworkGuard::blockedReason($value) !== null) {
        $fail('The :attribute must be a publicly reachable URL, not a private or reserved address.');
    }
}
```

**第二道：凭证验证时拦截（连通性测试前）**

即使表单验证通过了（比如 hostname 当时解析到公网 IP，后来 DNS 被篡改），`FileDiskService::validateCredentials()` 在真正尝试写入测试文件前，还会再次检查 endpoint：

```
DiskController@store
  → FileDiskService::validateCredentials($credentials, $driver)
    → 检查 credentials['endpoint'] 是否存在
      → PrivateNetworkGuard::blockedReason($endpoint)
        → 同样的 IP/DNS 检查逻辑
      → blocked !== null → return false (校验失败)
    → 通过 SSRF 检查后，才创建临时磁盘做连通性测试
      → Storage::disk('validation_temp')->put('invoiceshelf_temp.text', ...)
      → 成功写入再删除 → return true
      → 异常 → return false
```

关键代码 [FileDiskService.php#L105-L149](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/FileDiskService.php#L105-L149):
```php
public function validateCredentials(array $credentials, string $driver): bool
{
    // SSRF guard: 在真正发请求之前再次拦截
    if (isset($credentials['endpoint'])
        && is_string($credentials['endpoint'])
        && $credentials['endpoint'] !== ''
        && PrivateNetworkGuard::blockedReason($credentials['endpoint']) !== null) {
        return false;
    }

    // 通过后才做连通性测试...
    $tempDiskName = 'validation_temp';
    config(['filesystems.disks.'.$tempDiskName => $baseConfig]);
    \Storage::disk($tempDiskName)->put($root.'invoiceshelf_temp.text', 'Check Credentials');
    // ...
}
```

**两道拦截的分工**：

| 拦截点 | 时机 | 用途 |
|--------|------|------|
| `PublicHttpUrl` 规则 | 表单提交时 | 前端 UX 反馈，告诉用户"这个地址不可达" |
| `validateCredentials()` | 连通性测试前 | 运行时再次确认，防止 DNS rebinding 或直接绕过表单 |

### 7.3 完整 CIDR 黑名单

[PrivateNetworkGuard.php#L37-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php#L37-L68) 定义了 19 条 CIDR 规则，覆盖 IPv4、IPv6、IPv4-mapped IPv6 三条线：

**IPv4 黑名单（13 条）** [PrivateNetworkGuard.php#L37-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php#L37-L51)：

| CIDR | RFC | 覆盖范围 | 典型攻击向量 |
|------|-----|---------|------------|
| `0.0.0.0/8` | RFC1122 | "this" 网络 / 未指定 | 绕过防火墙 |
| `10.0.0.0/8` | RFC1918 | 私有 A 类 | 内网扫描 |
| `100.64.0.0/10` | RFC6598 | 运营商级 NAT (CGN) | 相邻租户 |
| `127.0.0.0/8` | RFC1122 | 回环 | 本地服务 |
| `169.254.0.0/16` | RFC3927 | 链路本地 | **云元数据 169.254.169.254** |
| `172.16.0.0/12` | RFC1918 | 私有 B 类 (172.16–172.31) | 内网扫描 |
| `192.0.0.0/24` | RFC6890 | IETF 协议分配 | 协议攻击 |
| `192.0.2.0/24` | RFC5737 | TEST-NET-1 文档 | 伪装 |
| `192.168.0.0/16` | RFC1918 | 私有 C 类 | 内网扫描 |
| `198.18.0.0/15` | RFC2544 | 基准测试 | 伪装 |
| `198.51.100.0/24` | RFC5737 | TEST-NET-2 文档 | 伪装 |
| `203.0.113.0/24` | RFC5737 | TEST-NET-3 文档 | 伪装 |
| `240.0.0.0/4` | RFC1112 | 保留 (含 255.255.255.255 广播) | 广播风暴 |

**IPv6 黑名单（6 条）** [PrivateNetworkGuard.php#L61-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php#L61-L68)：

| CIDR | 覆盖范围 | 典型攻击向量 |
|------|---------|------------|
| `::1/128` | 回环 | 本地服务 |
| `::/128` | 未指定 | 绕过 |
| `fc00::/7` | 唯一本地地址 (ULA) | 内网扫描 |
| `fe80::/10` | 链路本地 | 相邻节点 |
| `64:ff9b::/96` | NAT64 (嵌入 IPv4) | 间接内网访问 |
| `2001:db8::/32` | 文档 | 伪装 |

**IPv4-mapped IPv6 处理** [PrivateNetworkGuard.php#L137-L143](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php#L137-L143)：

```php
public static function ipIsBlocked(string $ip): bool
{
    $mapped = self::extractMappedIpv4($ip);
    if ($mapped !== null) {
        return self::ipIsBlocked($mapped);  // 递归到 IPv4 检查
    }
    // ...
}
```

`::ffff:10.0.0.1` 这种 IPv4-mapped IPv6 地址会被 [extractMappedIpv4()](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php#L229-L245) 拆出内嵌的 `10.0.0.1`，然后递归走 IPv4 黑名单。测试用例 [PrivateNetworkGuardTest.php#L23](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/tests/Unit/PrivateNetworkGuardTest.php#L23) 明确覆盖了这种情况：
```php
'ipv4-mapped v6' => 'http://[::ffff:10.0.0.1]',
```

### 7.4 五个出站调用方的防护边界

`PrivateNetworkGuard` 是一个**全系统复用**的 SSRF 守卫，不仅服务备份磁盘，还保护 AI、PDF、汇率三个出站链路。五个调用点按调用方式分为两类：

**第一类：`blockedReason()` — 返回 null / string，用于验证规则和保存前检查**

| # | 调用方 | 代码位置 | 防护的 URL 来源 | 检查时机 |
|---|--------|---------|---------------|---------|
| 1 | `PublicHttpUrl` 验证规则 | [PublicHttpUrl.php#L27](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/PublicHttpUrl.php#L27) | S3/Spaces endpoint（通过 `DiskEnvironmentRequest`） | 表单提交时 |
| 2 | `FileDiskService::validateCredentials()` | [FileDiskService.php#L112](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/FileDiskService.php#L112) | S3/Spaces endpoint | 连通性测试前 |

**第二类：`assertAllowed()` — 抛出 `BlockedUrlException`，用于运行时出站请求前**

| # | 调用方 | 代码位置 | 防护的 URL 来源 | 检查时机 |
|---|--------|---------|---------------|---------|
| 3 | `OpenRouterDriver::getBaseUrl()` | [OpenRouterDriver.php#L224](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Ai/OpenRouterDriver.php#L224) | AI base_url (公司级配置) | 每次 AI 请求前 |
| 4 | `GotenbergPdfDriver::loadView()` | [GotenbergPdfDriver.php#L25](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Pdf/GotenbergPdfDriver.php#L25) | Gotenberg host (env 配置) | 每次 PDF 生成前 |
| 5 | `CurrencyConverterDriver::getBaseUrl()` | [CurrencyConverterDriver.php#L61](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/ExchangeRate/CurrencyConverterDriver.php#L61) | DEDICATED 计划 URL (公司级配置) | 每次汇率查询前 |

**五个调用方的威胁模型对比**：

| 调用方 | 谁能设置 URL | 附着的凭证 | 攻击后果 |
|--------|------------|-----------|---------|
| FileDisk (1,2) | 管理员 | S3 Access Key + Secret | 泄露存储凭证 |
| OpenRouter (3) | 公司 owner | Bearer API Token | 泄露 AI 密钥 |
| Gotenberg (4) | 运维 (env) | 无（但可读取渲染内容） | SSRF 读内网 |
| CurrencyConverter (5) | 公司 owner | API Key | 泄露汇率密钥 |

**OpenRouterDriver 的 memoization 优化**：

[OpenRouterDriver.php#L29](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Ai/OpenRouterDriver.php#L29) 用 `$validatedBaseUrl` 缓存已校验的 URL，避免每次 AI 请求都重新做 DNS 解析：
```php
private ?string $validatedBaseUrl = null;

protected function getBaseUrl(): string
{
    if ($this->validatedBaseUrl !== null) {
        return $this->validatedBaseUrl;  // 缓存命中，跳过 DNS
    }
    // ... SSRF 检查 ...
    return $this->validatedBaseUrl = $url;
}
```

### 7.5 DNS Rebinding 的真实防护效果与 TOCTOU 局限

**威胁模型**：攻击者控制一个域名的 DNS，让它在检查时解析到公网 IP，在真正请求时解析到内网 IP。这就是经典的 Time-Of-Check-Time-Of-Use (TOCTOU) 攻击。

**当前代码的防护层级**：

```
时间线:  t0 (检查)                    t1 (请求)
          ↓                            ↓
DNS 响应: evil.com → 1.2.3.4 (公网)  evil.com → 10.0.0.1 (内网)

blockedReason() 检查时: 解析到 1.2.3.4 → 允许
真正 HTTP 请求时:        解析到 10.0.0.1 → 打到内网 ← 漏洞
```

**PrivateNetworkGuard 的 PHPDoc 明确承认此局限** [PrivateNetworkGuard.php#L25-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php#L25-L28)：

> Known limitation: this is not TOCTOU/DNS-rebinding-proof. A fully hardened implementation would pin the connection to the validated IP (cURL CURLOPT_RESOLVE). That is intentionally out of scope here — the realistic finding is direct private targeting, which this fully covers.

**各调用方的真实防护效果**：

| 调用方 | DNS rebinding 风险 | 缓解因素 |
|--------|-------------------|---------|
| FileDisk validateCredentials (2) | 中 — 检查与写入测试文件间有窗口 | 测试文件写入极快，窗口极小 |
| OpenRouterDriver (3) | **低** — memoization 把 URL 和检查绑定，每次请求前才检查 | 缓存了 `validatedBaseUrl`，但跨请求 DNS 可变 |
| GotenbergPdfDriver (4) | 中 — host 来自 env/config，非用户实时输入 | 攻击者需要先改 env 才能控制 URL |
| CurrencyConverterDriver (5) | 中 — DEDICATED URL 是用户配置的 | 检查与请求在同一方法调用内，窗口极小 |
| PublicHttpUrl 规则 (1) | **无运行时风险** — 仅做表单验证，不发请求 | 运行时由调用方 (2) 再次检查 |

**为什么"不够硬"但"足够用"**：

1. **直接内网瞄准**（把 endpoint 设为 `http://10.0.0.1`）→ 被 100% 拦截，这是最现实的攻击向量
2. **DNS rebinding** → 需要攻击者控制 DNS + 精确控制时序，攻击门槛极高
3. **完全硬化的方案**（cURL `CURLOPT_RESOLVE`）需要修改 PHP HTTP 客户端底层，侵入性大，项目选择有意识地不做

**如需加固**：最直接的方案是在 `assertAllowed()` 检查通过后，将解析到的 IP 传入 HTTP 客户端的 `CURLOPT_RESOLVE` 选项，强制连接到已验证的 IP，而非重新做 DNS 解析。

---

## 八、恢复防护机制 (Restore Guard)

### 核心设计：**无内置恢复 API = 源头防护**

代码库**未提供任何恢复/导入 API**。所有恢复操作必须由运维人员通过 `php artisan` 命令或数据库工具手动执行。这是避免恢复时覆盖运行数据的最根本防护。

下面按「挡覆盖、守机密、防误操作」三组展开。

---

### 8.1 挡覆盖：恢复时如何避免压坏正在运行的数据

#### 问题场景

恢复一个 `full` 备份意味着：用备份中的 SQL 覆盖当前数据库 + 用备份中的文件覆盖当前文件系统。如果应用正在服务请求，就会出现：
- 数据库刚覆盖一半，新请求写入的行与旧数据不一致
- 文件系统覆盖到一半，正在读取的 PDF 模板损坏
- 队列中的 Job 引用了已经不存在的记录

#### 防护手段 A：备份类型隔离 — 允许只恢复一半

**前端选择** [AdminBackupModal.vue#L40-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/scripts/features/admin/components/settings/AdminBackupModal.vue#L40-L44)：

```typescript
const backupTypeOptions = [
  { id: 'full', label: 'full' },
  { id: 'only-db', label: 'only-db' },
  { id: 'only-files', label: 'only-files' },
]
```

**后端执行** [CreateBackupJob.php#L43-L55](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php#L43-L55)：

```php
if ($this->data['option'] === 'only-db') {
    $backupJob->dontBackupFilesystem();   // ZIP 包中不含文件系统
}
if ($this->data['option'] === 'only-files') {
    $backupJob->dontBackupDatabases();    // ZIP 包中不含数据库 dump
}
```

**恢复 only-db 时避免压坏文件数据**：

`only-db-2024-06-17-15-30-00.zip` 内部结构：
```
only-db-2024-06-17-15-30-00.zip
  └── db-dumps/
        └── database.sql
```

恢复时只导入 `database.sql`，**不会触碰任何文件**。这意味着：
- `public/media/` 下所有发票附件、收据、Logo 完好无损
- `storage/app/templates/` 下的 PDF 模板完好无损
- `.env` 配置文件完好无损

**恢复 only-files 时避免压坏数据库**：

`only-files-2024-06-17-15-30-00.zip` 内部结构：
```
only-files-2024-06-17-15-30-00.zip
  └── (项目文件，不含 db-dumps/)
```

恢复时只覆盖文件系统，**不会触碰数据库**。这意味着：
- 正在运行的数据库记录（包括用户会话、待处理队列）完好无损
- 恢复后应用无需重新迁移

**文件名前缀是运维人员的"路标"**：

| 文件名前缀 | ZIP 内容 | 恢复影响范围 |
|-----------|---------|------------|
| `only-db-` | 仅 `db-dumps/database.sql` | 仅数据库行 |
| `only-files-` | 仅项目文件 | 仅文件系统 |
| 无前缀 | `db-dumps/` + 项目文件 | 全部 |

运维人员在解压前看到文件名就能判断这个包会影响什么，避免误操作"全套覆盖"。

#### 防护手段 B：维护模式 — 恢复前锁门

[ResetApp.php#L56-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Console/Commands/ResetApp.php#L56-L88) 展示了正确的数据操作流程：

```php
// 1. 先锁门 — 任何用户请求都返回 503
Artisan::call('down');

// 2. 执行数据操作
Artisan::call('migrate:fresh --seed --force');

// 3. 清理缓存
Artisan::call('optimize:clear');

// 4. 开门
Artisan::call('up');
```

**为什么恢复前必须 `down`**：

- 阻止所有 HTTP 请求，防止"数据库覆盖一半时有人写入"
- 阻止队列 worker 获取新 Job，防止 Job 操作的数据与备份不一致
- 阻止定时任务执行，防止 cron 任务在半恢复状态下修改数据

**only-db 恢复的推荐流程**：

```bash
php artisan down
# 导入 only-db 备份的 database.sql
mysql -u user -p database < database.sql
php artisan optimize:clear    # 清除所有缓存（config/route/view）
php artisan up
```

**only-files 恢复的推荐流程**：

```bash
php artisan down
# 解压 only-files 备份到项目目录
unzip only-files-2024-06-17-15-30-00.zip -d /path/to/project/
php artisan optimize:clear
php artisan up
```

#### 防护手段 C：自动清理 — 保证恢复点可用

[config/backup.php#L289-L339](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L289-L339) 的 `DefaultStrategy` 保证：
- **永远不删最新的备份**（无论配置如何）
- 按时间梯度保留：7天全保留 → 16天日保留 → 8周日保留 → 4月月保留 → 2年年保留
- 存储上限 5000 MB

这意味着即使恢复操作失败，最新的备份点一定还在。

---

### 8.2 守机密：备份文件如何防止未授权访问

#### 加密：AES-256 保护备份内容

[config/backup.php#L179-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L179-L188)：
```php
'password' => env('BACKUP_ARCHIVE_PASSWORD'),
'encryption' => 'default',  // 即 ZipArchive::EM_AES_256
```

- 密码通过 `.env` 注入，不进代码仓库
- 不设密码（`null`）则禁用加密
- 无密码无法解压 → 无法恢复 → 无法泄露数据

#### 权限：谁能操作备份 API

**三层权限锁** [routes/api.php#L190-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/api.php#L190-L191)：

```
请求 → auth:sanctum (必须有有效 API Token)
     → company (必须设置 company header)
     → bouncer (必须有 manage backups 权限)
     → BackupsController 中 $this->authorize('manage backups')
```

#### 路径安全：防止路径遍历

**PathToZip** [PathToZip.php#L24-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/PathToZip.php#L24-L29)：
- 删除/下载备份时，`path` 参数必须以 `.zip` 结尾
- 防止通过 `path=../../etc/passwd` 读取任意文件

**BackupDisk** [BackupDisk.php#L23-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/BackupDisk.php#L23-L30)：
- `disk` 参数必须在 `config('backup.backup.destination.disks')` 列表中
- 防止访问未授权的磁盘

#### SSRF 防护：防止备份目标指向内部网络

详见第七章「凭证校验与 SSRF 防护链路」。两道拦截确保备份磁盘的 endpoint 不会指向内部服务。

---

### 8.3 防误操作：如何阻止错误的人做错误的事

#### 无恢复 API — 最强的误操作防护

代码库中**没有任何** `restore` / `import` 端点。对比 `BackupsController` 只有 `index / store / destroy / download` 四个动作——能创建、能列出、能删除、能下载，**唯独不能通过 API 恢复**。

这意味着：
- 拥有 `manage backups` 权限的用户**也不能一键恢复**
- 恢复需要服务器命令行权限 + `BACKUP_ARCHIVE_PASSWORD` 密码
- 恢复操作天然需要运维级别的访问权限

#### 确认对话框 — 删除前的二次确认

[AdminBackupView.vue#L147-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/scripts/features/admin/views/settings/AdminBackupView.vue#L147-L155)：

```typescript
const confirmed = await dialogStore.openDialog({
  title: t('general.are_you_sure'),
  message: t('settings.backup.backup_confirm_delete'),
  variant: 'danger',
  // ...
})
if (!confirmed) return
```

#### 磁盘删除保护 — 有文件的磁盘不让删

[DiskController.php#L161-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Admin/Settings/DiskController.php#L161-L189)：

```php
// 系统磁盘不能删
if ($disk->type === 'SYSTEM') {
    return respondJson('not_allowed', 'System disks cannot be deleted.');
}

// 默认磁盘不能删
if ($disk->setAsDefault()) {
    return respondJson('not_allowed', 'The default disk cannot be deleted.');
}

// 有文件的磁盘不能删
$mediaCount = DB::table('media')
    ->where('disk', $diskName)->count();
if ($mediaCount > 0) {
    return respondJson('disk_has_files', 'Cannot delete this disk — it contains ...');
}
```

三重保护：系统盘锁 → 默认盘锁 → 有文件锁。防止误删备份目标磁盘导致备份丢失。

---

## 九、三块逻辑协同关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         备份请求发起                              │
│  AdminBackupView.vue → backupService.create(option, disk_id)    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       调度层 (CreateBackupJob)                   │
│  • 分发到队列 config('backup.queue.name')                       │
│  • 根据 option 开关: dontBackupFilesystem() / dontBackupDB()    │
│  • 设置文件名前缀: only-db- / only-files- / (无前缀full)        │
│  • 动态注册目标磁盘 (FileDiskService)                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
           ┌────────────────────┴────────────────────┐
           ▼                                         ▼
┌──────────────────────┐                  ┌──────────────────────┐
│   数据库快照模块      │                  │    附件归集模块       │
│                      │                  │                      │
│ • 连接: default      │                  │ • include: base_path │
│ • 导出: db-dumper    │                  │ • exclude: vendor/   │
│ • 可选 gzip 压缩     │                  │          node_modules│
│ • 文件名: database.sql│                  │          .git       │
│                      │                  │ • 媒体文件:          │
│                      │                  │   public/media/      │
│                      │                  │   FileDisk 动态磁盘  │
└───────────┬──────────┘                  └──────────┬───────────┘
            │                                        │
            └──────────────────┬─────────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │   压缩流水线模块      │
                    │                      │
                    │ 1. 收集DB dump + 文件│
                    │ 2. ZIP打包           │
                    │    • CM_DEFAULT      │
                    │    • 压缩级别: 9      │
                    │ 3. AES-256 加密      │
                    │ 4. 写入目标磁盘       │
                    └──────────┬───────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      恢复防护层                                  │
│                                                                 │
│  ┌─ 挡覆盖 ──────────────────────────────────────────────┐     │
│  │ • 备份类型隔离 (only-db / only-files / full)          │     │
│  │ • 文件名前缀标记 (only-db- / only-files-)             │     │
│  │ • 维护模式 (down → 操作 → up)                        │     │
│  │ • 自动清理保底 (永远不删最新备份)                      │     │
│  └───────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─ 守机密 ──────────────────────────────────────────────┐     │
│  │ • AES-256 加密 ZIP 包 (BACKUP_ARCHIVE_PASSWORD)      │     │
│  │ • 三层权限锁 (sanctum → company → bouncer)            │     │
│  │ • 路径安全 (PathToZip / BackupDisk 规则)              │     │
│  │ • SSRF 防护 (PublicHttpUrl + validateCredentials)     │     │
│  └───────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─ 防误操作 ────────────────────────────────────────────┐    │
│  │ • 无恢复 API (最根本的防护)                           │    │
│  │ • 删除前确认对话框                                    │    │
│  │ • 磁盘删除三重保护 (系统/默认/有文件)                  │    │
│  └───────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十、关键安全要点总结

### 10.1 only-db / only-files 单独恢复时避免压坏运行数据

| 恢复类型 | ZIP 包内容 | 不会影响 | 需要配合的操作 |
|---------|-----------|---------|-------------|
| `only-db` | `db-dumps/database.sql` | 文件系统（媒体、模板、.env） | `down` → 导入 SQL → `optimize:clear` → `up` |
| `only-files` | 项目文件（无 db-dumps/） | 数据库（记录、会话、队列） | `down` → 解压文件 → `optimize:clear` → `up` |
| `full` | 上述全部 | 无 | `down` → 导入 SQL + 解压文件 → `optimize:clear` → `up` |

**核心原则**：无论哪种类型，恢复前必须 `php artisan down`，恢复后必须 `php artisan optimize:clear`。

### 10.2 风险点与改进建议

1. **缺少自动备份调度**：当前仅支持手动备份，建议根据业务需求添加定期自动备份
2. **缺少恢复校验流程**：无内置的备份完整性校验，建议定期执行备份恢复演练
3. **备份通知配置**：当前通知配置为示例值，应配置真实的通知渠道

---

## 十一、代码文件索引

| 模块 | 核心文件 |
|------|----------|
| 控制器 | [BackupsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Admin/BackupsController.php) |
| 队列任务 | [CreateBackupJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php) |
| 配置工厂 | [BackupConfigurationFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/BackupConfigurationFactory.php) |
| 备份服务 | [BackupService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/BackupService.php) |
| 文件磁盘服务 | [FileDiskService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/FileDiskService.php) |
| 磁盘控制器 | [DiskController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Admin/Settings/DiskController.php) |
| 表单验证 | [DiskEnvironmentRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Requests/DiskEnvironmentRequest.php) |
| SSRF 守卫 | [PrivateNetworkGuard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/PrivateNetworkGuard.php) |
| SSRF 异常 | [BlockedUrlException.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Net/BlockedUrlException.php) |
| URL 验证规则 | [PublicHttpUrl.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/PublicHttpUrl.php) |
| AI 出站驱动 | [OpenRouterDriver.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Ai/OpenRouterDriver.php) |
| PDF 出站驱动 | [GotenbergPdfDriver.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/Pdf/GotenbergPdfDriver.php) |
| 汇率出站驱动 | [CurrencyConverterDriver.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Support/ExchangeRate/CurrencyConverterDriver.php) |
| 验证规则 | [PathToZip.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/PathToZip.php), [BackupDisk.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/BackupDisk.php) |
| 调度配置 | [console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/console.php) |
| 备份配置 | [config/backup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php) |
| 前端页面 | [AdminBackupView.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/scripts/features/admin/views/settings/AdminBackupView.vue), [AdminBackupModal.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/scripts/features/admin/components/settings/AdminBackupModal.vue) |
| API 服务 | [backup.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/scripts/api/services/backup.service.ts) |
| 测试用例 | [BackupTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/tests/Feature/Admin/BackupTest.php), [BackupGlobalTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/tests/Feature/Admin/BackupGlobalTest.php), [PrivateNetworkGuardTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/tests/Unit/PrivateNetworkGuardTest.php) |
| 参考流程 | [ResetApp.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Console/Commands/ResetApp.php) |
