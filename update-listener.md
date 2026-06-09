# 版本升级事件与监听器分析

## 一、核心结论

**当前版本（2.3.3）的 `UpdateFinished` 事件没有任何实际注册的监听器。** 代码库中只保留了监听器的基类框架，但没有具体的监听器实现。升级时的数据结构变更完全依赖 Laravel 的数据库迁移（Migration）机制，而非事件监听器。

---

## 二、升级事件体系

### 2.1 升级事件：UpdateFinished

**文件**: [`app/Events/UpdateFinished.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Events/UpdateFinished.php)

```php
class UpdateFinished
{
    use Dispatchable;

    public $new;      // 新版本号
    public $old;      // 旧版本号

    public function __construct($old, $new)
    {
        $this->old = $old;
        $this->new = $new;
    }
}
```

**触发位置**: [`app/Space/Updater.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Space/Updater.php#L140-L150) 的 `finishUpdate()` 方法

```php
public static function finishUpdate($installed, $version)
{
    Setting::setSetting('version', $version);
    event(new UpdateFinished($installed, $version));  // 触发事件
    // ...
}
```

### 2.2 监听器基类：Updates\Listener

**文件**: [`app/Listeners/Updates/Listener.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Listeners/Updates/Listener.php)

这是一个**抽象基类**，源自 Akaunting 项目设计模式，供具体版本监听器继承使用。

核心方法 `isListenerFired()`：

```php
protected function isListenerFired($event)
{
    // 如果监听器版本 <= 旧版本（即已经升级过了），返回 true（表示已触发，不再执行）
    if (version_compare(static::VERSION, $event->old, '<=')) {
        return true;
    }
    return false;
}
```

**设计意图**:
- 每个具体监听器定义自己的 `VERSION` 常量
- 通过版本比较判断该监听器是否需要执行
- 确保每个版本的升级逻辑只执行一次

**注意**: 当前 `VERSION` 常量为空字符串 `''`，因为这是基类，具体版本监听器应覆盖此常量。

### 2.3 实际监听器状态

**当前代码库中没有任何继承此基类的具体监听器。**

即：
- ❌ 没有 `V130Listener`、`V200Listener` 之类的具体实现
- ❌ `app/Listeners/Updates/` 目录下只有 `Listener.php` 一个文件
- ❌ 没有任何地方注册了 `UpdateFinished` 事件的监听者

---

## 三、事件注册机制

### 3.1 标准 Laravel 方式（未使用）

Laravel 通常通过 `EventServiceProvider` 的 `$listen` 属性注册事件-监听器映射，但本项目中：

- ❌ 不存在 `app/Providers/EventServiceProvider.php`
- ❌ `config/app.php` 中也没有注册事件服务提供者

### 3.2 动态发现（未使用）

代码库中没有发现以下动态注册模式：
- 扫描 `Listeners/Updates/` 目录自动发现监听器
- 通过 `Event::listen()` 或 `Event::subscribe()` 动态注册
- 服务提供者中的事件绑定

### 3.3 结论

**`UpdateFinished` 事件目前是"裸奔"状态**——被触发但没有任何监听者。这可能是：
1. 从 Akaunting 移植时保留了框架，但尚未实现具体监听器
2. 设计上预留了扩展点，供模块（Module）系统使用

---

## 四、完整升级流程

升级入口有两个：**Web API 方式** 和 **Artisan 命令方式**，核心逻辑都在 `Updater` 类中。

### 4.1 升级核心类：Updater

**文件**: [`app/Space/Updater.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Space/Updater.php)

| 方法 | 功能 |
|------|------|
| `checkForUpdate()` | 调用远程 API 检查是否有新版本 |
| `download()` | 下载更新包 ZIP |
| `unzip()` | 解压更新包到临时目录 |
| `copyFiles()` | 将解压后的文件复制到应用根目录 |
| `deleteFiles()` | 删除指定的旧文件 |
| `migrateUpdate()` | 执行数据库迁移 (`php artisan migrate --force`) |
| `finishUpdate()` | 更新版本号 + 触发 `UpdateFinished` 事件 |

### 4.2 Web API 升级流程

控制器目录: [`app/Http/Controllers/V1/Admin/Update/`](file:///d:/fz/0508-2\solo-dogfeeding/code/124-InvoiceShelf/app/Http/Controllers/V1/Admin/Update/)

| 控制器 | 对应 Updater 方法 |
|--------|------------------|
| `CheckVersionController` | `checkForUpdate()` |
| `DownloadUpdateController` | `download()` |
| `UnzipUpdateController` | `unzip()` |
| `CopyFilesController` | `copyFiles()` |
| `DeleteFilesController` | `deleteFiles()` |
| `MigrateUpdateController` | `migrateUpdate()` |
| `FinishUpdateController` | `finishUpdate()` |

此外还有一个聚合控制器 [`UpdateController.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Http/Controllers/V1/Admin/Update/UpdateController.php)，包含所有升级步骤的方法。

### 4.3 Artisan 命令升级

**文件**: [`app/Console/Commands/UpdateCommand.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Console/Commands/UpdateCommand.php)

命令: `php artisan core:update`

执行顺序（`handle()` 方法）:
```
1. 获取已安装版本
2. 检查最新版本
3. 下载更新包  →  Updater::download()
4. 解压        →  Updater::unzip()
5. 复制文件    →  Updater::copyFiles()
6. 删除旧文件  →  Updater::deleteFiles()
7. 执行迁移    →  Updater::migrateUpdate()
8. 完成更新    →  Updater::finishUpdate()  ← 触发 UpdateFinished 事件
```

---

## 五、数据库迁移与版本升级的衔接

### 5.1 版本号存储位置

**两处版本号**:

| 位置 | 说明 | 获取方式 |
|------|------|----------|
| `version.md` | 代码文件版本 | `File::get(base_path('version.md'))` |
| `settings` 表 | 数据库版本 | `Setting::getSetting('version')` |

`version.md` 是代码自带的版本号（当前 `2.3.3`），`settings.version` 是数据库中记录的已安装版本。

### 5.2 版本迁移文件模式

每个版本升级都有对应的迁移文件，命名模式：

- 旧版 Crater 时期: `2022_01_05_115423_update_crater_version_600.php`
- InvoiceShelf 时期: `2024_07_12_235756_update_version_130.php`

**示例**: [`database/migrations/2024_07_12_235756_update_version_130.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/database/migrations/2024_07_12_235756_update_version_130.php)

```php
public function up(): void
{
    Setting::setSetting('version', '1.3.0');  // 更新版本号
}

public function down(): void
{
    Setting::setSetting('version', '1.2.2');  // 回退版本号
}
```

### 5.3 数据迁移与业务逻辑分离

**纯数据迁移的迁移文件**:
- 只包含 `Setting::setSetting('version', 'x.x.x')`
- 例如: `update_version_130.php`、`update_version_122.php` 等

**包含业务逻辑的迁移文件**:
- 不仅更新版本号，还执行数据转换
- 例如: [`2024_02_11_075831_update_version_110.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/database/migrations/2024_02_11_075831_update_version_110.php)

```php
public function up(): void
{
    try {
        DB::update("UPDATE abilities SET entity_type = REPLACE(entity_type, 'Crater', 'InvoiceShelf')");
        DB::update("UPDATE assigned_roles SET entity_type = REPLACE(entity_type, 'Crater', 'InvoiceShelf')");
    } catch (Exception $e) {
    }

    Setting::setSetting('version', '1.1.0');
}
```

**设计分析**:
- 版本号更新内嵌在迁移文件的 `up()` 方法末尾
- 迁移成功执行后，版本号自动更新
- 利用 Laravel 迁移的事务性保证"迁移执行"和"版本号更新"的一致性

### 5.4 升级时迁移的执行方式

`Updater::migrateUpdate()` 直接调用:

```php
public static function migrateUpdate()
{
    Artisan::call('migrate --force');
    return true;
}
```

这意味着:
- 所有未执行的迁移都会按时间戳顺序执行
- 每个迁移文件负责自己的版本号更新
- 最后一个迁移文件执行完毕后，`settings.version` 就是最新版本号
- 然后 `finishUpdate()` 会再设置一次版本号（冗余但保险）

---

## 六、事件监听器的设计意图（推测）

虽然当前没有实际监听器，但从基类设计可以推测其预期用法：

### 6.1 预期监听器结构

```
app/Listeners/Updates/
├── Listener.php          ← 基类（已存在）
├── V100Listener.php      ← 1.0.0 版本升级逻辑（不存在）
├── V110Listener.php      ← 1.1.0 版本升级逻辑（不存在）
├── V120Listener.php      ← 1.2.0 版本升级逻辑（不存在）
└── ...
```

### 6.2 预期监听器实现

```php
// 示例：V130Listener.php（假设）
namespace App\Listeners\Updates;

class V130Listener extends Listener
{
    public const VERSION = '1.3.0';

    public function handle(UpdateFinished $event)
    {
        if ($this->isListenerFired($event)) {
            return; // 已经升级过，跳过
        }

        // 执行 1.3.0 版本的数据迁移逻辑
        // ...
    }
}
```

### 6.3 与 Migration 的分工

| 机制 | 适用场景 |
|------|----------|
| Migration | 表结构变更、索引、字段调整 |
| Listener | 复杂业务逻辑、文件操作、调用外部服务、数据清洗 |

当前项目所有版本升级逻辑都放在 Migration 中，没有使用 Listener 模式。

---

## 七、版本状态变更总览

升级过程中状态变更的完整时序：

```
初始状态:
  version.md = 旧版本（代码还没替换）
  settings.version = 旧版本
  
步骤 1: 下载并解压更新包
步骤 2: 复制文件 → 覆盖代码
  → version.md 变为新版本
  → 新的迁移文件被复制到 database/migrations/
  
步骤 3: 删除旧文件

步骤 4: 执行迁移 (php artisan migrate)
  → 按顺序执行所有新迁移文件
  → 每个迁移文件的 up() 方法执行
  → 最后一个迁移文件更新 settings.version
  
步骤 5: finishUpdate()
  → Setting::setSetting('version', $version)  ← 再设一次
  → event(new UpdateFinished($old, $new))     ← 触发事件（无人监听）

最终状态:
  version.md = 新版本
  settings.version = 新版本
```

---

## 八、为什么让人"没把握"

1. **事件触发但无监听**: `UpdateFinished` 事件会触发，但你不知道有没有监听者，因为注册方式不直观。

2. **版本号双写**: 迁移文件里写一次版本号，`finishUpdate()` 里又写一次，看似冗余。

3. **基类存在但无实现**: `Listeners/Updates/Listener.php` 看起来像是有一套监听器体系，但实际目录里只有这一个文件。

4. **迁移文件职责不清**: 有的迁移只改版本号，有的还夹带业务逻辑，没有统一规范。

5. **没有 EventServiceProvider**: 不符合 Laravel 标准做法，不容易找到事件-监听器的映射关系。

---

## 九、关键文件索引

| 文件 | 作用 |
|------|------|
| [`app/Events/UpdateFinished.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Events/UpdateFinished.php) | 升级完成事件类 |
| [`app/Listeners/Updates/Listener.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Listeners/Updates/Listener.php) | 监听器抽象基类 |
| [`app/Space/Updater.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Space/Updater.php) | 升级核心逻辑 |
| [`app/Console/Commands/UpdateCommand.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Console/Commands/UpdateCommand.php) | Artisan 升级命令 |
| [`app/Http/Controllers/V1/Admin/Update/`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Http/Controllers/V1/Admin/Update/) | Web API 升级控制器 |
| [`app/Models/Setting.php`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/app/Models/Setting.php) | 设置模型，存储版本号 |
| [`version.md`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/version.md) | 代码版本号文件 |
| [`database/migrations/`](file:///d:/fz/0508-2/solo-dogfeeding/code/124-InvoiceShelf/database/migrations/) | 数据库迁移文件（含版本迁移） |
