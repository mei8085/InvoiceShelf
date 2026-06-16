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

### 4.3 临时文件管理

- 临时目录: `storage_path('app/backup-temp')` [config/backup.php#L173](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L173)
- 备份完成后自动清理

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

## 六、恢复防护机制 (Restore Guard)

### 6.1 设计理念：**无内置恢复 API = 最强防护**

代码库**未提供任何恢复/导入 API**，这是避免恢复时覆盖运行数据的核心设计。恢复必须通过运维人员手动操作，从源头防止误操作。

### 6.2 多层防护体系

#### 第一层：类型隔离 - 避免整体覆盖

三种备份类型 [AdminBackupModal.vue#L40-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/scripts/features/admin/components/settings/AdminBackupModal.vue#L40-L44):
```typescript
const backupTypeOptions: BackupTypeOption[] = [
  { id: 'full', label: 'full' },        // 完整备份
  { id: 'only-db', label: 'only-db' },  // 仅数据库
  { id: 'only-files', label: 'only-files' }, // 仅文件
]
```

**文件名前缀标记** [CreateBackupJob.php#L51-L55](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php#L51-L55):
```php
if (! empty($this->data['option'])) {
    $prefix = str_replace('_', '-', $this->data['option']).'-';
    $backupJob->setFilename($prefix.date('Y-m-d-H-i-s').'.zip');
}
```

生成的文件名示例:
- `only-db-2024-06-17-15-30-00.zip`
- `only-files-2024-06-17-15-30-00.zip`
- `2024-06-17-15-30-00.zip` (完整备份无前缀)

#### 第二层：加密防护 - 数据机密性

- AES-256 加密 ZIP 包
- 密码通过环境变量 `BACKUP_ARCHIVE_PASSWORD` 配置
- 无密码无法解压查看或恢复

#### 第三层：权限与访问控制

1. **API 权限** [BackupsController.php#L24](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Admin/BackupsController.php#L24):
   ```php
   $this->authorize('manage backups');
   ```

2. **路由中间件** [routes/api.php#L190-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/api.php#L190-L191):
   ```php
   Route::middleware(['auth:sanctum', 'company'])->group(function () {
       Route::middleware(['bouncer'])->group(function () {
           Route::apiResource('backups', BackupsController::class);
       });
   });
   ```

#### 第四层：路径安全 - 防止路径遍历攻击

**PathToZip 验证规则** [PathToZip.php#L24-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/PathToZip.php#L24-L29):
```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    if (! Str::endsWith($value, '.zip')) {
        $fail('The given value must be a path to a zip file.');
    }
}
```

**BackupDisk 验证规则** [BackupDisk.php#L23-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/BackupDisk.php#L23-L30):
```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    $configuredBackupDisks = config('backup.backup.destination.disks');
    if (! in_array($value, $configuredBackupDisks)) {
        $fail('This disk is not configured as a backup disk.');
    }
}
```

#### 第五层：网络安全 - SSRF 防护

**PrivateNetworkGuard** 在 `FileDiskService::validateCredentials()` 中使用，防止备份目标配置指向内部网络:
- 阻止私有IP段 (10/8, 172.16/12, 192.168/16)
- 阻止回环地址 (127.0.0.1, ::1)
- 阻止链路本地地址 (169.254/16, fe80::/10)
- 阻止云元数据服务 (169.254.169.254)
- 阻止非 HTTP/HTTPS 协议

#### 第六层：维护模式 - 恢复操作参考

虽然无内置恢复 API，但 `ResetApp` 命令展示了正确的恢复操作流程:

**ResetApp 流程** [ResetApp.php#L56-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Console/Commands/ResetApp.php#L56-L88):
```php
public function handle(): void
{
    // 1. 激活维护模式，阻止用户访问
    Artisan::call('down');
    
    // 2. 执行数据操作 (迁移/种子)
    Artisan::call('migrate:fresh --seed --force');
    Artisan::call('db:seed', ['--class' => 'DemoSeeder', '--force' => true]);
    
    // 3. 清理缓存
    Artisan::call('optimize:clear');
    
    // 4. 关闭维护模式
    Artisan::call('up');
}
```

> **恢复操作最佳实践**：应参照此模式，在恢复前启用 `down` 维护模式，恢复完成后再 `up`。

#### 第七层：自动清理 - 版本管理

**清理策略** [config/backup.php#L289-L339](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php#L289-L339):
```php
'strategy' => DefaultStrategy::class,
'default_strategy' => [
    'keep_all_backups_for_days' => 7,        // 7天内全部保留
    'keep_daily_backups_for_days' => 16,     // 16天内每日保留
    'keep_weekly_backups_for_weeks' => 8,    // 8周内每周保留
    'keep_monthly_backups_for_months' => 4,  // 4月内每月保留
    'keep_yearly_backups_for_years' => 2,    // 2年内每年保留
    'delete_oldest_backups_when_using_more_megabytes_than' => 5000, // 存储上限
],
```

> **重要保证**：DefaultStrategy 永远不会删除最新的备份，无论配置如何。

---

## 七、三块逻辑协同关系图

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
│                      恢复防护层 (被动防护)                       │
│                                                                 │
│  类型隔离    加密保护    权限控制    路径安全    SSRF防护       │
│  ────────   ────────   ────────   ────────   ────────          │
│  full       AES-256    manage     PathToZip   PrivateNetwork   │
│  only-db    env 密码   backups    BackupDisk    Guard          │
│  only-files           auth:      仅.zip路径                    │
│  文件名前缀            sanctum                                 │
│                                                                 │
│  自动清理策略    维护模式参考    无内置恢复API                   │
│  ──────────    ──────────    ─────────────                    │
│  7天全保留       down/up       人工操作，源头防误操作           │
│  16天每日                                                │
│  8周每周                                                 │
│  4月每月                                                 │
│  2年每年                                                 │
│  5000MB上限                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、关键安全要点总结

### 8.1 避免恢复时覆盖运行数据的核心机制

1. **无恢复 API**：最根本的防护，所有恢复必须人工执行
2. **类型分离**：`only-db` / `only-files` 允许选择性恢复，避免整体覆盖
3. **维护模式**：恢复操作应先 `down` 再操作，最后 `up`
4. **权限控制**：只有 `manage backups` 权限的用户可操作
5. **加密保护**：备份文件加密，避免未授权访问
6. **文件名时间戳**：便于版本追溯，避免恢复错误版本

### 8.2 风险点与改进建议

1. **缺少自动备份调度**：当前仅支持手动备份，建议根据业务需求添加定期自动备份
2. **缺少恢复校验流程**：无内置的备份完整性校验，建议定期执行备份恢复演练
3. **备份通知配置**：当前通知配置为示例值，应配置真实的通知渠道

---

## 九、代码文件索引

| 模块 | 核心文件 |
|------|----------|
| 控制器 | [BackupsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Http/Controllers/Admin/BackupsController.php) |
| 队列任务 | [CreateBackupJob.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Jobs/CreateBackupJob.php) |
| 配置工厂 | [BackupConfigurationFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/BackupConfigurationFactory.php) |
| 备份服务 | [BackupService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/BackupService.php) |
| 文件磁盘服务 | [FileDiskService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Services/Storage/FileDiskService.php) |
| 验证规则 | [PathToZip.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/PathToZip.php), [BackupDisk.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Rules/Backup/BackupDisk.php) |
| 调度配置 | [console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/routes/console.php) |
| 备份配置 | [config/backup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/config/backup.php) |
| 前端页面 | [AdminBackupView.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/script/features/admin/views/settings/AdminBackupView.vue), [AdminBackupModal.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/script/features/admin/components/settings/AdminBackupModal.vue) |
| API 服务 | [backup.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/resources/script/api/services/backup.service.ts) |
| 测试用例 | [BackupTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/tests/Feature/Admin/BackupTest.php), [BackupGlobalTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/tests/Feature/Admin/BackupGlobalTest.php) |
| 参考流程 | [ResetApp.php](file:///d:/fz/0601-2/solo-dogfeeding/code/12-InvoiceShelf/app/Console/Commands/ResetApp.php) |
