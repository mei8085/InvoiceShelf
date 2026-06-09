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

## 四、迁移事务机制深度分析

### 4.1 迁移执行链路概览

升级时的迁移调用链路：

```
Updater::migrateUpdate()
  → Artisan::call('migrate --force')
    → MigrateCommand
      → Migrator::run()
        → 遍历所有未执行的迁移
          → 对每个迁移：
            → 检查迁移类的 $withinTransaction 属性
            → 检查当前 Schema Grammar 是否支持事务 DDL
            → 两者都满足才在事务内执行 up()
            → 否则直接执行 up()
          → 写入 migrations 表，记录已执行
```

### 4.2 migrateUpdate 本身不包事务

文件位置：`app/Space/Updater.php`

```php
public static function migrateUpdate()
{
    Artisan::call('migrate --force');
    return true;
}
```

**核对结果：migrateUpdate 本身没有任何事务包裹。**

- ❌ 没有 `DB::transaction()` 包裹
- ❌ 没有 `beginTransaction()` / `commit()` / `rollBack()`
- ❌ 没有 try/catch 异常捕获
- ❌ 没有检查迁移执行结果（Artisan::call 返回值被忽略）

就是裸调了一下 `php artisan migrate --force`，成功失败全靠迁移自己扛。

### 4.3 withinTransaction：迁移类的事务开关

#### 基类默认值

Laravel 的 `Migration` 基类有一个 `$withinTransaction` 属性，**默认值为 `true`**：

```php
// Illuminate\Database\Migrations\Migration（框架基类）
abstract class Migration
{
    /**
     * Indicates if the migration should be run within a transaction.
     *
     * @var bool
     */
    public $withinTransaction = true;
    ...
}
```

#### 本项目迁移的实际情况

本项目所有迁移都继承自 `Migration` 基类，且**没有任何一个迁移覆盖了 `$withinTransaction` 属性**。

验证：在 `database/migrations/` 目录下搜索 `withinTransaction`，结果为 0 匹配。

所以：**所有迁移都使用默认值 `$withinTransaction = true`，即"希望"在事务中执行。**

但注意：这只是迁移类的"意愿"，最终能不能在事务里执行，还要看 Schema Grammar 支不支持。

### 4.4 Schema Grammar：事务支持的最终裁决者

`$withinTransaction = true` 只是迁移类的"意愿"，真正能不能在事务中执行，取决于当前数据库驱动对应的 **Schema Grammar** 是否支持事务性 DDL。

#### 判断逻辑（Laravel 框架 Migrator 中的逻辑）

```php
// 伪代码，来自 Migrator::runMigration()
protected function runMigration($migration, $method)
{
    $connection = $this->resolveConnection(
        $migration->getConnection()
    );

    $grammar = $connection->getSchemaBuilder()
        ->getConnection()
        ->getSchemaGrammar();

    if (
        $migration->withinTransaction
        && $grammar->supportsSchemaTransactions()
    ) {
        // 在事务中执行
        $connection->transaction(function () use ($migration, $method) {
            $migration->$method();
        });
    } else {
        // 直接执行，不包事务
        $migration->$method();
    }
}
```

两个条件必须**同时满足**才会包事务：
1. 迁移类的 `$withinTransaction = true`（本项目所有迁移都满足）
2. Schema Grammar 的 `supportsSchemaTransactions() = true`（取决于数据库驱动）

#### 各驱动的 Schema Grammar 事务支持

| 驱动 | Schema Grammar 类 | supportsSchemaTransactions() | 说明 |
|------|-------------------|----------------------------|------|
| MySQL | `MySqlGrammar` | ❌ **false** | DDL 会隐式提交事务 |
| MariaDB | `MariaDbGrammar` | ❌ **false** | 同 MySQL |
| PostgreSQL | `PostgresGrammar` | ✅ **true** | 完整支持事务 DDL |
| SQLite | `SQLiteGrammar` | ✅ **true** | 基本支持事务 DDL |
| SQL Server | `SqlServerGrammar` | ⚠️ 部分支持 | 仅部分 DDL 可事务化 |

**关键校正：MySQL 的 Schema Grammar 明确返回 `false`，所以 MySQL 下 Laravel 根本不会尝试把迁移包在事务里！**

不是"事务会被 DDL 打断"的问题，而是**Laravel 知道 MySQL 不支持事务 DDL，所以一开始就不会开事务**。

### 4.5 MySQL vs PostgreSQL：不能统一视为有事务保证

这是最关键的校正点。**MySQL 和 PostgreSQL 绝对不能统一视为有事务保证。**

#### PostgreSQL：完整事务保证

PostgreSQL 的 Schema Grammar 支持事务 DDL，所以：
- 每个迁移的 `up()` 方法整个包在一个事务里
- 迁移中所有操作（DDL + DML）都是原子的
- 任何一步失败，整个迁移全部回滚
- 版本号和表结构变更保持一致

#### MySQL：完全没有事务保证（迁移层面）

MySQL 的 Schema Grammar 不支持事务 DDL，所以：
- Laravel 不会给迁移包事务
- 迁移中的每个 SQL 语句都是独立执行、立即生效的
- 如果迁移执行到一半失败了，前面的操作已经生效，不会回滚
- 版本号可能更新了但后面的操作失败了，也可能版本号还没更新前面的 DDL 就失败了

**即使是纯 DML 的迁移（只改数据不改表结构），在 MySQL 下也不会被 Laravel 包在事务里！**

因为 `supportsSchemaTransactions()` 返回的是整体判断，Laravel 不会去分析迁移里有没有 DDL，直接就不包事务。

### 4.6 本项目迁移的实际事务保证

基于上面的分析，本项目迁移的事务保证情况：

| 数据库 | 事务保证 | 说明 |
|--------|----------|------|
| PostgreSQL | ✅ 单迁移完整事务 | 每个迁移的所有操作原子性 |
| MySQL / MariaDB | ❌ 完全没有 | 每个 SQL 独立执行，失败不回滚 |
| SQLite | ✅ 单迁移完整事务 | 每个迁移的所有操作原子性 |

本项目默认数据库配置是 SQLite（`config/database.php` 中 `'default' => env('DB_CONNECTION', 'sqlite')`），但实际生产环境大概率用 MySQL。

### 4.7 跨迁移文件：完全没有事务

不管什么数据库，跨迁移文件都没有事务：

- 迁移 1 执行 → 提交
- 迁移 2 执行 → 提交
- 迁移 3 执行 → 失败！
- 迁移 1 和 2 已经提交了，不会回滚

所以如果升级过程中执行到第 N 个迁移失败了：
- 前 N-1 个迁移的变更已经生效
- 数据库处于"半升级"状态
- 版本号停留在最后一个成功迁移对应的版本（如果那个迁移里有版本号更新的话）

### 4.8 整个升级流程：没有全局事务

整个升级流程涉及**文件系统操作**和**数据库操作**两大部分，完全没有全局事务包裹：

```
升级完整流程：
  1. 下载更新包 ZIP          ← 文件系统，无事务
  2. 解压到临时目录          ← 文件系统，无事务
  3. 复制文件覆盖代码        ← 文件系统，无事务（覆盖后无法回滚）
  4. 删除旧文件              ← 文件系统，无事务
  5. 执行迁移                ← 数据库，每个迁移独立（MySQL 下连单迁移事务都没有）
  6. finishUpdate            ← 数据库 + 事件
```

**最危险的阶段：第 3 步复制文件之后，第 5 步迁移完成之前。**

此时代码已经是新版本，但数据库结构还是旧版本。如果迁移失败了：
- 代码跑不起来（新旧不兼容）
- 没法回退文件（旧文件已经被覆盖/删除了）
- 数据库处于不确定状态
- MySQL 下连单个迁移内部的失败都没法回滚

### 4.9 版本号写入的位置不一致

版本号写入分散在多个迁移文件中，位置也不统一：

**写在末尾的（规范做法）：**
- `update_version_130.php` - 只有版本号，在最后
- `update_version_110.php` - 数据替换在前，版本号在后

**写在中间的（有坑）：**
- `update_crater_version_400.php` - 版本号在中间，后面还有大量操作

```php
// update_crater_version_400.php
public function up(): void
{
    $this->fileDiskSeed();          // 先 seed file disk

    Setting::setSetting('version', '4.0.0');  // ← 中间就更新版本号了！

    $user = User::where('role', 'admin')->first();
    if ($user && $user->role == 'admin') {
        $user->update(['role' => 'super admin']);
        $this->updateCompanySettings($user);
        $this->updateCreatorId($user);
        // ... 更多操作
    }
}
```

在 PostgreSQL 下（有事务），整个 `up()` 是原子的，版本号写在哪无所谓，反正要么都成功要么都失败。

但在 MySQL 下（无事务），版本号写在中间就有问题了——如果版本号更新之后的操作失败了，版本号已经是 4.0.0 了，但后面的数据迁移没完成，数据库处于不一致状态。

### 4.10 版本号双写的真实风险

版本号会被写两次：

```
第一次：在最后一个 update_version_xxx 迁移的 up() 方法中
第二次：在 Updater::finishUpdate() 方法中
```

**风险 1：迁移半成功 + 手动调用 finishUpdate → 版本号造假**

如果迁移执行到一半失败了，此时：
- 数据库结构：部分新 + 部分旧
- 版本号：取决于最后一个成功的迁移有没有更新版本号

如果此时有人手动调用 `finishUpdate`（比如前端重试升级，跳过了迁移步骤），版本号会被强行更新到最新版本，但数据库结构其实不完整。

**风险 2：迁移里没写版本号，全靠 finishUpdate**

不是每个版本都有对应的 `update_version_xxx.php` 迁移。有些版本只有业务迁移，没有专门的版本号迁移。

如果最后一个迁移不是版本迁移，而是普通的业务迁移，那迁移执行完后版本号还是旧的，得等 `finishUpdate` 来更新。

**风险 3：两处版本号可能不一致**

迁移里写的版本号和 `finishUpdate` 传进来的版本号可能不一致：
- 迁移里是硬编码的字符串（如 `'1.3.0'`）
- finishUpdate 是从参数传进来的（从 version.md 读取）

如果开发时忘了同步修改两边，就会出现不一致。

**风险 4：MySQL 下迁移中途失败，版本号已更新（如果版本号写在前面）**

在 MySQL 下（无事务），如果迁移里版本号写在前面，后面的操作失败了：
- 版本号已经更新了
- 但后面的数据迁移/结构变更没完成
- 显示是新版本，实际是残缺的

---

## 五、迁移和 finishUpdate 分别改了什么状态

### 5.1 迁移（migrateUpdate）改了什么

迁移改的东西很多，不只是版本号：

| 类型 | 具体内容 | 举例 |
|------|----------|------|
| 表结构变更（DDL） | 加表、加字段、改字段、删表、重命名 | 新增 tax_included 字段、创建 jobs 表、重命名 password_resets 表 |
| 数据迁移（DML） | 批量更新已有数据 | abilities/assigned_roles 的 entity_type 替换、model namespace 替换 |
| 数据填充 | 插入初始数据 | 新增币种、file disk 初始化、默认设置写入 |
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

### 5.3 完整状态变更总览（含事务边界）

```
升级前：
  代码版本（version.md）= 旧版本
  数据库版本（settings.version）= 旧版本
  数据库结构 = 旧版本结构

─────────────────────────────────────────
  ↓  步骤 1-4：文件系统操作（无事务）
─────────────────────────────────────────

  1. 下载更新包 ZIP
  2. 解压到临时目录
  3. 复制文件覆盖代码
     → 代码版本（version.md）= 新版本
     → 新的迁移文件已就位
     ↑ 此步不可逆，覆盖后无法回滚
  4. 删除旧文件
     ↑ 此步不可逆

─────────────────────────────────────────
  ↓  步骤 5：数据库迁移（每个迁移独立）
─────────────────────────────────────────

  5. 执行所有未执行的迁移（按时间戳顺序）
     
     迁移 1 → 执行 → 提交（或隐式提交）
     迁移 2 → 执行 → 提交（或隐式提交）
     ...
     迁移 N → 执行 → 成功/失败
     
     每个迁移内部：
       - 如果只有 DML 且数据库支持事务 → 有事务保证
       - 如果包含 DDL 且是 MySQL → 无完整事务保证
       - DDL 执行时会隐式提交
     
     最后一个成功的迁移决定了当前版本号

─────────────────────────────────────────
  ↓  步骤 6：收尾（finishUpdate）
─────────────────────────────────────────

  6. finishUpdate()
     → settings.version = 新版本（再写一次）
     → 触发 UpdateFinished 事件（无人监听）

升级后（理想情况）：
  代码版本 = 新版本
  数据库版本 = 新版本
  数据库结构 = 新版本结构

升级后（迁移失败的情况）：
  代码版本 = 新版本（已经覆盖了）
  数据库版本 = 最后一个成功迁移对应的版本
  数据库结构 = 部分新 + 部分旧
  状态：不一致，可能无法正常运行
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
