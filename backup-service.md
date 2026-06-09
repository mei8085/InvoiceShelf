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

### 3.1 校验层级总览

备份服务的校验分布在四个层级，各层的校验强度差异很大：

| 层级 | 校验内容 | 校验强度 | 位置 |
|------|---------|---------|------|
| 前端层 | option 必填、磁盘必填、下拉选值 | 🔒 严格 | BackupModal.vue（vuelidate + 下拉） |
| 权限层 | 用户是否有备份管理权限 | 🔒 严格 | 控制器 `authorize('manage backups')` |
| HTTP 输入层 | 请求参数格式校验 | ⚠️ 极弱 | 仅 destroy/download 校验 path，store 完全没有 |
| 业务层 | company、file_disk_id 非空 | ⚠️ 不完整 | 队列执行时 BackupConfigurationFactory |

**最大的特点**：前端做了完整的表单校验，后端 `store` 接口完全没有参数校验，直接入队。真正的业务校验延迟到队列任务执行时才做，而且校验本身也有漏洞。

---

### 3.2 前端校验链路（完整且严格）

前端 [BackupModal.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/components/modal-components/BackupModal.vue) 共有五层防护，确保正常使用时发出的数据一定合法。

#### 第一层：数据初始化有默认值

[backup.js:11-17](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/stores/backup.js#L11-L17)

```js
state: () => ({
  currentBackupData: {
    option: 'full',     // option 默认就是 'full'，不会为空
    selected_disk: null, // 磁盘默认 null，需要用户选择
  },
})
```

打开弹窗时还会默认选第一个磁盘：
```js
// BackupModal.vue:167-170
async function loadData() {
  let res = await diskStore.fetchDisks({ limit: 'all' })
  backupStore.currentBackupData.selected_disk = res.data.data[0]
}
```

#### 第二层：UI 组件约束

- `BaseMultiselect` 组件设置了 `:can-deselect="false"` 和 `:allow-empty="false"`，用户无法取消选择
- 下拉选择框限制了 `option` 只能是 `['full', 'only-db', 'only-files']` 三个值
- 磁盘选择也是下拉，只能从已有的磁盘列表中选

#### 第三层：vuelidate 校验规则

[BackupModal.vue:125-136](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/components/modal-components/BackupModal.vue#L125-L136)

```js
const rules = computed(() => {
  return {
    currentBackupData: {
      option: {
        required: helpers.withMessage(t('validation.required'), required),
      },
      selected_disk: {
        required: helpers.withMessage(t('validation.required'), required),
      },
    },
  }
})
```

两个字段都是必填的。

#### 第四层：提交前拦截

[BackupModal.vue:143-147](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/components/modal-components/BackupModal.vue#L143-L147)

```js
async function createNewBackup() {
  v$.value.currentBackupData.$touch()       // 触发所有校验
  if (v$.value.currentBackupData.$invalid) { // 校验不通过就 return
    return true
  }
  // 只有校验通过才会往下执行...
}
```

#### 第五层：组装数据并调用 API

[BackupModal.vue:149-155](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/components/modal-components/BackupModal.vue#L149-L155)

```js
let data = {
  option: backupStore.currentBackupData.option,
  file_disk_id: backupStore.currentBackupData.selected_disk.id,
  // selected_disk 是对象，取 .id 作为 file_disk_id
}
let res = await backupStore.createBackup(data)
```

最终通过 [backup.js:35-39](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/stores/backup.js#L35-L39) 发送 POST 请求：

```js
createBackup(data) {
  return http.post(`/api/v1/backups`, data)
}
```

> **前端小结**：从数据初始化 → 组件约束 → 校验规则 → 提交拦截 → API 调用，一共五层防护。正常使用前端界面的情况下，发到后端的数据一定是合法的。

---

### 3.3 后端 HTTP 层校验（store 接口完全没有）

对比前端的层层防护，后端 [BackupsController@store](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L75-L85) 的处理极其简单：

```php
public function store(Request $request)
{
    $this->authorize('manage backups');  // ← 唯一的校验：权限

    $data = $request->all();             // ← 直接拿所有参数，不验证
    $data['company'] = $request->header('company');

    dispatch(new CreateBackupJob($data)) // ← 直接入队
        ->onQueue(config('backup.queue.name'));

    return $this->respondSuccess();      // ← 立即返回成功
}
```

**store 接口没有的校验：**
- ❌ 没有 `$request->validate()`
- ❌ 没有 Form Request 类
- ❌ 没有 `option` 的白名单校验（in:full,only-db,only-files）
- ❌ 没有 `file_disk_id` 的整数类型校验
- ❌ 没有 `file_disk_id` 的存在性校验（exists:file_disks,id）
- ❌ 没有任何异常捕获

**其他接口有部分校验：**
- `destroy` 方法：校验 `path` 必须是 ZIP 路径（用 PathToZip 规则）
- `DownloadBackupController`：同样校验 `path` 必须是 ZIP 路径

> **设计意图推测**：开发者可能认为「有前端校验就够了」，或者是为了代码简洁。但对于 API 来说，后端不做校验是有风险的。

---

### 3.4 业务层校验：BackupConfigurationFactory（不完整，有漏洞）

参数的真正校验发生在队列任务执行时，由 [BackupConfigurationFactory::make()](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) 进行。但这里的校验**不完整**，存在明显漏洞。

#### 逐行分析执行路径

```php
// 第 12 行：入口方法
public static function make($data = []): Config
{
    // 第 14-16 行：校验 company 不为空
    if (blank($data['company'] ?? null)) {
        throw new Exception('The Company ID is missig');
    }
    // ✅ 这行是对的：company 为空就抛异常

    // 第 18-20 行：校验 file_disk_id 不为空
    if (blank($data['file_disk_id'] ?? null)) {
        throw new Exception('No file disk selected');
    }
    // ✅ 这行也是对的：file_disk_id 为空就抛异常

    // 第 22 行：查询 FileDisk 记录
    $fileDisk = FileDisk::find($data['file_disk_id']);
    // ⚠️ 问题在这里！find() 找不到记录会返回 null，但代码没检查

    // 第 24 行：直接调用 setConfig()
    $fileDisk->setConfig();
    // 💥 如果 $fileDisk 是 null，这里会报 Fatal Error！
    // 错误信息："Call to a member function setConfig() on null"

    // 第 26-28 行：设置备份目标磁盘
    $prefix = env('DYNAMIC_DISK_PREFIX', 'temp_');
    config(['backup.backup.destination.disks' => [$prefix.$fileDisk->driver]]);
    // 💥 如果 $fileDisk 是 null，这里也会报错
    // 错误信息："Attempt to read property 'driver' on null"

    // ... 后续代码
}
```

#### 校验漏洞总结表

| 校验项 | 是否有校验 | 失败时的结果 | 错误级别 |
|-------|-----------|-------------|---------|
| `company` 为空 | ✅ 有 | 抛出 `Exception`，队列任务失败 | 普通异常 |
| `file_disk_id` 为空 | ✅ 有 | 抛出 `Exception`，队列任务失败 | 普通异常 |
| `file_disk_id` 无效（找不到记录） | ❌ **没有** | 调用 `null->setConfig()`，Fatal Error | **致命错误** |

> **关键事实**：传一个不存在的 `file_disk_id`（比如 `9999`），比传空值的后果更严重。空值至少是个正常的 Exception，无效 ID 是 PHP Fatal Error，可能影响队列 Worker 的稳定性。

> **其他小问题**：第 15 行有拼写错误 `'The Company ID is missig'` → 应为 `missing`。

---

### 3.5 已定义但未被调用的规则类（死代码）

项目自定义了三个备份验证规则类，但**只有一个被实际使用**，另外两个是「死代码」。

#### （1）PathToZip - 路径必须指向 ZIP 文件 ✅ 实际使用

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
- [BackupsController@destroy](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L96-L98) - 删除备份时校验 path
- [DownloadBackupController](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/DownloadBackupController.php#L20-L22) - 下载备份时校验 path

#### （2）BackupDisk - 磁盘必须是已配置的备份磁盘 ❌ 未使用

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

> **事实**：此规则类在整个代码库中**从未被调用**，属于死代码。

#### （3）FilesystemDisks - 磁盘必须是已配置的文件系统磁盘 ❌ 未使用

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

> **事实**：此规则类也**从未被调用**，同样是死代码。

> **推测**：这两个规则类可能是早期版本留下来的，或者计划要用但还没接入。

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

在 [CreateBackupJob@handle](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Jobs/CreateBackupJob.php#L35-L58) 中，根据 `option` 参数决定备份内容和文件名。

前端 [BackupModal.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/components/modal-components/BackupModal.vue#L105) 定义了三个选项：

```js
const options = reactive(['full', 'only-db', 'only-files'])
```

| option 值 | 备份内容 | 文件名格式示例 |
|-----------|---------|--------------|
| `full` | 数据库 + 文件 | `full-2024-01-15-10-30-00.zip` |
| `only-db` | 仅数据库 | `only-db-2024-01-15-10-30-00.zip` |
| `only-files` | 仅文件 | `only-files-2024-01-15-10-30-00.zip` |

**关键代码逻辑**：

```php
// 控制备份内容
if ($this->data['option'] === 'only-db') {
    $backupJob->dontBackupFilesystem();  // 仅数据库：跳过文件系统
}

if ($this->data['option'] === 'only-files') {
    $backupJob->dontBackupDatabases();   // 仅文件：跳过数据库
}

// 控制文件名：只要 option 非空，就设置自定义文件名
if (! empty($this->data['option'])) {
    $prefix = str_replace('_', '-', $this->data['option']).'-';
    $backupJob->setFilename($prefix.date('Y-m-d-H-i-s').'.zip');
}
```

> **重要事实**：`full` 选项也会走 `! empty()` 判断（因为字符串 `'full'` 非空），所以 **full 备份也会设置自定义文件名前缀 `full-`**，不是使用 spatie 包的默认命名。只有当 option 参数完全不传或为空字符串时，才使用 spatie 默认命名。

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

---

### 4.5 各备份操作的目标磁盘差异（重要！）

这是备份服务最容易踩坑的地方：**前端给所有接口都传了 `file_disk_id`，但后端只有部分接口使用了它，各操作的目标磁盘不一致。**

#### 四个接口的磁盘处理对比

| 操作 | 前端传 file_disk_id | 后端是否使用 | 目标磁盘 | 代码位置 |
|------|---------------------|-------------|---------|---------|
| 获取列表 | ✅ 传 | ✅ 是 | 动态磁盘（按 file_disk_id） | [BackupsController@index:31-39](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L31-L39) |
| 创建备份 | ✅ 传 | ✅ 是（队列中） | 动态磁盘（按 file_disk_id） | [CreateBackupJob](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Jobs/CreateBackupJob.php) → [BackupConfigurationFactory](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Space/BackupConfigurationFactory.php) |
| 删除备份 | ✅ 传 | ❌ **完全忽略** | 系统默认文件系统（local） | [BackupsController@destroy:100](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L100) |
| 下载备份 | ✅ 传 | ❌ **完全忽略** | 系统默认文件系统（local） | [DownloadBackupController:24](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/DownloadBackupController.php#L24) |

#### 逐接口代码对照

**（1）列表接口 — 正确使用动态磁盘**

[BackupsController@index:31-39](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L31-L39)

```php
if ($request->file_disk_id) {          // 有 file_disk_id 才走动态配置
    $fileDisk = FileDisk::find($request->file_disk_id);
    if ($fileDisk) {                   // 找到了才设置
        $fileDisk->setConfig();        // ← 动态设置文件系统
        $prefix = env('DYNAMIC_DISK_PREFIX', 'temp_');
        config(['backup.backup.destination.disks' => [$prefix.$fileDisk->driver]]);
    }
}
// 然后用 config('filesystems.default') 创建 BackupDestination
```

> 列表接口的处理是完整的：有 ID → 查记录 → 找到了才设配置 → 用动态磁盘。

**（2）创建备份 — 在队列中使用动态磁盘**

不是在控制器里设置，而是把 `file_disk_id` 传给队列任务，由 `BackupConfigurationFactory` 在队列执行时设置。详见第三章的分析。

**（3）删除接口 — 完全忽略 file_disk_id**

[BackupsController@destroy:92-110](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/BackupsController.php#L92-L110)

```php
public function destroy($disk, Request $request)
{
    $this->authorize('manage backups');

    $validated = $request->validate([
        'path' => ['required', new PathToZip],  // ← 只校验 path
    ]);

    // 直接用默认文件系统创建！没有调用 setConfig()
    $backupDestination = BackupDestination::create(
        config('filesystems.default'),  // ← 用的是系统默认
        config('backup.backup.name')
    );

    $backupDestination->backups()
        ->first(fn (Backup $backup) => $backup->path() === $validated['path'])
        ->delete();

    return $this->respondSuccess();
}
```

> **关键事实**：
> - 路由参数 `$disk` 完全没用到（连变量都没读取）
> - `$request->file_disk_id` 也完全没用到
> - 直接用 `config('filesystems.default')`，也就是 `local` 磁盘
> - 也就是说：不管你在前端选的是哪个磁盘，删除的永远是本地磁盘上的文件

**（4）下载接口 — 同样完全忽略 file_disk_id**

[DownloadBackupController:16-35](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/app/Http/Controllers/V1/Admin/Backup/DownloadBackupController.php#L16-L35)

```php
public function __invoke(Request $request)
{
    $this->authorize('manage backups');

    $validated = $request->validate([
        'path' => ['required', new PathToZip],  // ← 只校验 path
    ]);

    // 同样直接用默认文件系统
    $backupDestination = BackupDestination::create(
        config('filesystems.default'),  // ← 系统默认
        config('backup.backup.name')
    );

    $backup = $backupDestination->backups()->first(
        fn (Backup $backup) => $backup->path() === $validated['path']
    );

    // ... 下载流
}
```

> 和删除接口一模一样的问题：完全忽略 `file_disk_id`，永远从本地磁盘下载。

#### 前端的传参方式

前端 [BackupSetting.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/views/settings/BackupSetting.vue) 给每个操作都传了 `file_disk_id`：

```js
// 列表查询
async function fetchBackupsData({ page, filter, sort }) {
  let data = {
    disk: filters.selected_disk.driver,
    file_disk_id: filters.selected_disk.id,  // ← 传了
  }
  let response = await backupStore.fetchBackups(data)
}

// 删除
function onRemoveBackup(backup) {
  let data = {
    disk: filters.selected_disk.driver,
    file_disk_id: filters.selected_disk.id,  // ← 传了
    path: backup.path,
  }
  let response = await backupStore.removeBackup(data)
}

// 下载
function onDownloadBckup(backup) {
  window.axios({
    method: 'GET',
    url: '/api/v1/download-backup',
    params: {
      disk: filters.selected_disk.driver,
      file_disk_id: filters.selected_disk.id,  // ← 传了
      path: backup.path,
    },
  })
}
```

前端的逻辑是一致的——哪个磁盘上的列表，就在哪个磁盘上删除/下载。但后端没跟上。

#### 这个不一致导致的问题

| 场景 | 表现 |
|------|------|
| 用户选了 S3 磁盘，看到 S3 上的备份列表 | ✅ 正常 |
| 用户在 S3 磁盘上创建备份 | ✅ 正常（队列中设置了动态磁盘） |
| 用户在 S3 磁盘列表上点「下载」 | ❌ 找不到文件！因为从本地磁盘找 |
| 用户在 S3 磁盘列表上点「删除」 | ❌ 删不掉！因为删的是本地磁盘上的 |
| 如果本地磁盘正好有同名文件 | ❗ 可能删错/下错文件（虽然概率很低，因为有时间戳） |

> **简单说**：列表看得到的，下载和删除都操作不到；下载删除能操作到的，列表里看不到（除非正好是本地磁盘）。

#### 各接口处理方式总结表

| 接口 | 有没有 file_disk_id 参数 | 有没有调用 setConfig | 有没有改 backup.destination.disks | 用的是哪个磁盘 |
|------|------------------------|---------------------|--------------------------------|--------------|
| index | ✅ 有 | ✅ 有 | ✅ 有 | 动态磁盘 |
| store（控制器） | ✅ 有 | ❌ 没有 | ❌ 没有 | 队列中处理 |
| store（队列中） | ✅ 有（data 里带的） | ✅ 有 | ✅ 有 | 动态磁盘 |
| destroy | ✅ 有（请求参数里） | ❌ **没有** | ❌ **没有** | 系统默认（local） |
| download | ✅ 有（请求参数里） | ❌ **没有** | ❌ **没有** | 系统默认（local） |

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
| [resources/scripts/admin/components/modal-components/BackupModal.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/components/modal-components/BackupModal.vue) | 前端备份弹窗（有校验） | 全文 |
| [resources/scripts/admin/stores/backup.js](file:///d:/fz/0508-2/solo-dogfeeding/code/121-InvoiceShelf/resources/scripts/admin/stores/backup.js) | 前端状态管理 | 全文 |
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

**Q: 前端有哪些校验？后端有哪些校验？**
A: 前端有五层防护：默认值初始化 → 下拉组件约束（不可取消） → vuelidate required 规则 → 提交前 $touch 校验拦截 → 组装数据调用 API。后端 store 接口只有权限校验，完全没有参数校验。真正的业务校验在队列里的 BackupConfigurationFactory，而且还不完整（没检查 file_disk_id 有效性）。

**Q: 传一个不存在的 file_disk_id 会怎么样？**
A: 会导致 PHP Fatal Error。`BackupConfigurationFactory` 只检查了 `file_disk_id` 不为空，但没有检查 `FileDisk::find()` 的结果是不是 null。找不到记录的话，`$fileDisk->setConfig()` 会直接报错："Call to a member function setConfig() on null"。

**Q: option 参数可以传任意字符串吗？有安全问题吗？**
A: 后端没有白名单校验，传什么字符串都会被用作文件名前缀。前端是下拉选择所以不会有问题，但如果绕过前端直接调用 API，可以传任意值。虽然用作文件名前缀的风险相对可控（因为会拼上时间戳），但理论上存在路径遍历的潜在风险。

**Q: file_disk_id 为空和无效，哪个后果更严重？**
A: 无效更严重。为空是正常的 Exception，队列任务失败但系统没事。无效是 Fatal Error，可能影响队列 Worker 的稳定性。按说应该反过来才对——无效 ID 至少应该优雅地抛异常。
