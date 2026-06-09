# 版本升级事件与监听器深度分析

## 一、核心结论（事务边界核对版）

经过逐行代码核对 + 事务边界分析，以下是确认后的结论：

1. **Listener 基类只是一个普通 PHP 类**，不是 Laravel 标准事件监听器
2. **migrateUpdate 本身不包事务**，只是裸调 `Artisan::call('migrate --force')`
3. **单个 migration 的事务保证取决于数据库驱动**，不能笼统说 MySQL 有事务
4. **整个升级流程没有全局事务**，文件系统操作 + 数据库操作完全脱节
5. **版本号双写有真实风险**，迁移失败 + 手动调用 finishUpdate 会导致版本号造假
6. **UpdateFinished 事件会触发，但没有任何监听器在监听**（事件是空转）

---

## 二、Listener 基类：就是个普通类

### 2.1 代码原貌

文件位置：`app/Listeners/Updates/Listener.php`

```php
<?php

namespace App\Listeners\Updates;

// Implementation taken from Akaunting - https://github.com/akaunting/akaunting
class Listener
{
    public const VERSION = '';

    protected function isListenerFired($event)
    {
        if (version_compare(static::VERSION, $event->old, '<=')) {
            return true;
        }
        return false;
    }
}
```

### 2.2 核对结果

| 特征 | 是否有 | 说明 |
|------|--------|------|
| 继承父类 | ❌ 无 | 没有 extends 任何类 |
| 实现接口 | ❌ 无 | 没有 implements |
| 使用 Trait | ❌ 无 | 没有 use 任何 trait |
| `__invoke()` 方法 | ❌ 无 | Laravel 事件监听器的标准入口 |
| `handle()` 方法 | ❌ 无 | 另一种常见的监听器入口 |
| `ShouldQueue` | ❌ 无 | 队列化监听器接口 |

**结论**：这就是一个纯普通类，只有一个 `isListenerFired()` 方法和一个 `VERSION` 常量。它本身不能作为 Laravel 事件监听器使用，必须由子类继承并实现 `handle()` 或 `__invoke()` 才能被 Laravel 事件系统调用。

### 2.3 设计意图（从代码反推）

这个基类是从 Akaunting 项目抄过来的脚手架代码，预期用法是：

```
子类继承 Listener → 定义 VERSION 常量 → 实现 handle() 方法
→ 在 handle() 中先调用 isListenerFired() 判断是否需要执行
→ 如果没触发过就执行该版本的升级逻辑
```

但当前代码库中**没有任何子类继承这个基类**。

---

## 三、升级事件：UpdateFinished

### 3.1 事件类定义

文件位置：`app/Events/UpdateFinished.php`

```php
class UpdateFinished
{
    use Dispatchable;

    public $new;   // 新版本号
    public $old;   // 旧版本号

    public function __construct($old, $new)
    {
        $this->old = $old;
        $this->new = $new;
    }
}
```

这是一个标准的 Laravel 事件类，使用了 `Dispatchable` trait。

### 3.2 事件触发位置

文件位置：`app/Space/Updater.php` 的 `finishUpdate()` 方法

```php
public static function finishUpdate($installed, $version)
{
    Setting::setSetting('version', $version);
    event(new UpdateFinished($installed, $version));  // ← 这里触发

    return [
        'success' => true,
        'error' => false,
        'data' => [],
    ];
}
```

### 3.3 事件是否真的触发监听器？

**答案：不会触发任何监听器，因为根本没有注册。**

核对过的注册方式全部缺失：

| 注册方式 | 是否存在 | 说明 |
|----------|----------|------|
| EventServiceProvider 的 $listen | ❌ | 项目中没有 EventServiceProvider.php |
| `Event::listen()` 动态注册 | ❌ | 代码中搜索不到 |
| `Event::subscribe()` 订阅者 | ❌ | 代码中搜索不到 |
| 事件自动发现 | ❌ | 没有 shouldDiscoverEvents 配置 |
| 服务提供者中注册 | ❌ | 所有 Provider 都查过了 |

所以 `UpdateFinished` 事件每次升级都会触发，但都是"对空喊话"，没有任何监听者响应。

---

## 四、迁移执行与版本号写入的事务保证

### 4.1 迁移是怎么执行的

文件位置：`app/Space/Updater.php`

```php
public static function migrateUpdate()
{
    Artisan::call('migrate --force');
    return true;
}
```

就是调了一下 `php artisan migrate --force`，没有包事务，没有异常捕获。

### 4.2 版本号写在哪里

版本号写入分散在两个地方：

**地点 1：迁移文件内部**

每个版本对应一个 `update_version_xxx.php` 迁移文件，在 `up()` 方法里调用 `Setting::setSetting('version', 'x.x.x')`。

```php
// 典型的纯版本迁移
public function up(): void
{
    Setting::setSetting('version', '1.3.0');
}
```

**地点 2：finishUpdate() 方法**

```php
public static function finishUpdate($installed, $version)
{
    Setting::setSetting('version', $version);  // ← 再写一次
    event(new UpdateFinished($installed, $version));
    ...
}
```

### 4.3 事务保证分析

**单迁移文件内：有事务（取决于数据库驱动）**

- Laravel 的 `Migration` 基类默认在支持事务的数据库中（MySQL、PostgreSQL 等）会把整个 `up()` 方法包在一个事务里
- 也就是说，如果一个迁移文件里既有数据操作又有版本号更新，它们在同一个事务里
- 迁移失败会回滚，版本号也不会更新

**跨迁移文件：没有事务**

- 每个迁移文件是独立的事务
- 升级过程通常会执行多个迁移文件，依次提交
- 如果第 3 个迁移失败，前 2 个已经提交了，不会回滚

**版本号写入的位置不一致**

这是最坑的地方。有的迁移版本号写在最后，有的写在中间：

```
update_version_130.php:  版本号在 up() 最后一行（只有版本号）
update_version_110.php:  版本号在 up() 最后一行（前面有数据替换）
update_crater_version_400.php:  版本号在 up() 中间，后面还有一大堆操作！
```

以 `update_crater_version_400.php` 为例：

```php
public function up(): void
{
    $this->fileDiskSeed();          // 先 seed file disk

    Setting::setSetting('version', '4.0.0');  // ← 中间就更新版本号了！

    $user = User::where('role', 'admin')->first();

    if ($user && $user->role == 'admin') {
        $user->update(['role' => 'super admin']);  // ← 后面还有操作
        $this->updateCompanySettings($user);
        $this->updateCreatorId($user);
        // ... 更多操作
    }
}
```

虽然在单迁移事务内这些操作是原子的（要么都成功要么都失败），但版本号写在中间的设计很容易让人误解。

### 4.4 迁移和 finishUpdate 的双写问题

版本号会被写两次：

```
第一次：在最后一个 update_version_xxx 迁移的 up() 方法中
第二次：在 Updater::finishUpdate() 方法中
```

为什么写两次？推测是：
- 迁移里写是为了保证"迁移执行成功 = 版本号更新"
- finishUpdate 里再写一次是冗余保险，万一有人绕过迁移直接调用 finishUpdate 呢

但这也带来了问题：如果迁移执行了一半（部分迁移成功部分失败），然后有人手动调用 finishUpdate，版本号就会被强行更新到最新版本，但数据库结构其实不完整。

---

## 五、迁移和 finishUpdate 分别改了什么状态

### 5.1 迁移（migrateUpdate）改了什么

迁移改的东西很多，不只是版本号：

| 类型 | 具体内容 | 举例 |
|------|----------|------|
| 表结构变更 | 加表、加字段、改字段、删表 | 新增 tax_included 字段、创建 jobs 表 |
| 数据迁移 | 批量更新已有数据 | abilities/assigned_roles 的 entity_type 替换 |
| 数据填充 | 插入初始数据 | 新增币种、file disk 初始化 |
| 版本号更新 | 更新 settings.version | 每个版本迁移的 setSetting |

**每个版本迁移文件的职责不统一**：

- 有的纯改版本号（如 v1.2.1、v1.2.2、v1.3.0）
- 有的夹带业务逻辑（如 v1.1.0 替换 entity_type、v4.0.0 一堆数据迁移）
- 有的甚至版本号写在中间（如 v4.0.0）

### 5.2 finishUpdate 改了什么

finishUpdate 只改两件事：

1. **更新版本号**：`Setting::setSetting('version', $version)`
2. **触发事件**：`event(new UpdateFinished($installed, $version))`

仅此而已，没有其他状态变更。

### 5.3 状态变更总览

```
升级前：
  settings.version = 旧版本
  数据库结构 = 旧版本结构
  
执行 migrateUpdate()：
  → 执行所有未执行的迁移（按时间戳顺序）
  → 每个迁移可能改表结构、改数据、改版本号
  → 最后一个版本迁移执行完后，settings.version = 新版本
  
执行 finishUpdate()：
  → settings.version = 新版本（再写一次，值相同）
  → 触发 UpdateFinished 事件（无人监听）

升级后：
  settings.version = 新版本
  数据库结构 = 新版本结构
```

---

## 六、完整升级流程（时序）

### 6.1 Artisan 命令方式

命令：`php artisan core:update`

执行顺序（`app/Console/Commands/UpdateCommand.php`）：

```
1. getInstalledVersion()    读取 version.md
2. getLatestVersionResponse()  调用远程 API 检查更新
3. download()               下载更新包 ZIP
4. unzip()                  解压到临时目录
5. copyFiles()              复制文件覆盖代码
   → 此时 version.md 已变成新版本
   → 新的迁移文件已就位
6. deleteFiles()            删除旧文件
7. migrateUpdate()          执行 php artisan migrate --force
   → 所有新迁移依次执行
   → 表结构变更
   → 数据迁移
   → settings.version 更新
8. finishUpdate()           收尾
   → settings.version 再写一次
   → 触发 UpdateFinished 事件
```

### 6.2 Web API 方式

控制器目录：`app/Http/Controllers/V1/Admin/Update/`

每个步骤对应一个控制器：
- CheckVersionController
- DownloadUpdateController
- UnzipUpdateController
- CopyFilesController
- DeleteFilesController
- MigrateUpdateController
- FinishUpdateController

另外还有个聚合控制器 `UpdateController.php` 把所有方法放一起了。

---

## 七、为什么让人"没把握"

结合代码，总结几个不确定的根源：

### 1. 事件体系似有实无

- 有 `UpdateFinished` 事件类
- 有 `Listeners/Updates/Listener.php` 基类
- 看起来像是有一整套事件监听机制
- 但实际上没有任何具体监听器，也没有注册

### 2. 版本号双写，职责不清

- 迁移里写一次
- finishUpdate 里再写一次
- 不知道该以哪个为准
- 不知道如果两个地方不一致会怎样

### 3. 迁移文件职责不统一

- 有的迁移只改版本号
- 有的迁移夹带大量业务逻辑
- 有的版本号写在末尾，有的写在中间
- 没有统一规范，不好预测

### 4. 没有事务包裹整个升级流程

- 单个迁移有事务
- 但整个升级过程没有全局事务
- 升级到一半失败了怎么办？没有回滚机制

### 5. 事件触发无声无息

- 事件触发了，但你不知道有没有监听者
- 想找监听者，发现没有 EventServiceProvider
- 想搜 `Event::listen`，搜不到
- 到底有没有东西在监听？心里没底

---

## 八、关键文件清单

| 文件路径 | 作用 |
|----------|------|
| `app/Events/UpdateFinished.php` | 升级完成事件类 |
| `app/Listeners/Updates/Listener.php` | 监听器基类（空架子） |
| `app/Space/Updater.php` | 升级核心逻辑类 |
| `app/Console/Commands/UpdateCommand.php` | Artisan 升级命令 |
| `app/Http/Controllers/V1/Admin/Update/` | Web 升级控制器组 |
| `app/Models/Setting.php` | 设置模型（存储版本号） |
| `version.md` | 代码版本号文件 |
| `database/migrations/` | 数据库迁移（含版本迁移） |
