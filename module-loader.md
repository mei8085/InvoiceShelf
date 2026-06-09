# InvoiceShelf 模块加载器深度解析

本文档深入分析 InvoiceShelf 应用的模块化扩展机制，重点解答三个核心问题：**Module 静态资源注册的生命周期**、**菜单构建的先后顺序**、以及**从模块发现到 Vue 挂载的完整调用链路**。

## 目录

1. [核心问题导读](#核心问题导读)
2. [模块发现与加载器内核](#模块发现与加载器内核)
3. [服务提供者注册与启动时序](#服务提供者注册与启动时序)
4. [Module 静态资源注册生命周期](#module-静态资源注册生命周期)
5. [主菜单构建顺序深度解析](#主菜单构建顺序深度解析)
6. [前端资源动态挂载](#前端资源动态挂载)
7. [从模块发现到 Vue 挂载的完整流程](#从模块发现到-vue-挂载的完整流程)
8. [模块安装与启用流程](#模块安装与启用流程)
9. [模块开发标准模式](#模块开发标准模式)

---

## 核心问题导读

在阅读本文前，先明确三个关键问题及其答案：

| 问题 | 简短答案 |
|------|----------|
| **Module 静态脚本/样式在哪个请求阶段生效？** | 在服务提供者 `boot()` 阶段注册，整个请求生命周期内有效，视图渲染时消费 |
| **AppServiceProvider 建菜单 vs 模块 SP 追加菜单，谁先谁后？** | 模块 SP 的 boot **先于** AppServiceProvider 的 boot，因此模块不能直接 `Menu::get()` 追加，需通过**配置合并**或 `app()->booted()` 回调 |
| **从模块发现到 Vue 挂载经过了多少环节？** | 后端 5 个阶段（发现 → 注册 → 启动 → 资源注入 → API输出） + 前端 4 个阶段（脚本加载 → 回调注册 → 应用启动 → 菜单渲染） |

---

## 模块发现与加载器内核

### 2.1 底层框架

InvoiceShelf 基于 **nwidart/laravel-modules**（通过 `invoiceshelf/modules` 包引入）实现模块化，见 [composer.json](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/composer.json#L17)。

模块框架通过 Composer 的 **package discovery** 机制自动注册服务提供者，无需手动添加到 providers 数组。

### 2.2 模块配置

核心配置在 [config/modules.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/config/modules.php)：

```php
return [
    'namespace' => 'Modules',        // 模块命名空间
    'paths' => [
        'modules' => base_path('Modules'),  // 模块存放目录
        'assets' => public_path('modules'), // 资源发布目录
    ],
    'activator' => 'file',           // 启用状态存储方式
    'scan' => [
        'enabled' => false,          // 不扫描 vendor 目录
    ],
    // ...
];
```

### 2.3 模块状态管理

**双重状态管理**：

1. **文件状态**（FileActivator）：`storage/app/modules_statuses.json`
   - 由 nwidart 框架管理
   - 缓存键：`activator.installed`，有效期 7 天

2. **数据库状态**：`modules` 表，对应 [Module 模型](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Models/Module.php)
   - 存储版本号、安装状态、启用状态、排序等
   - 由应用层管理

### 2.4 自动加载

```json
"autoload": {
    "psr-4": {
        "Modules\\": "Modules/"
    }
}
```

`Modules\` 命名空间下的所有类通过 Composer PSR-4 自动加载。

---

## 服务提供者注册与启动时序

### 3.1 Laravel 服务提供者生命周期

Laravel 的服务提供者分为两个阶段：

```
请求到达
   │
   ▼
register 阶段（按注册顺序）
   ├─ 所有 SP 的 register() 依次执行
   └─ 功能：绑定服务到容器
   │
   ▼
boot 阶段（按注册顺序）
   ├─ 所有 SP 的 boot() 依次执行
   └─ 功能：执行启动逻辑（路由、视图、事件等）
   │
   ▼
应用已启动（booted）
   │
   ▼
路由匹配 → 中间件 → 控制器 → 响应
```

### 3.2 nwidart 模块服务提供者的注册时机

nwidart 主服务提供者（`ModulesServiceProvider`）通过 package discovery 自动注册，**早于**应用服务提供者。

在 Laravel 11+ 中，服务提供者的注册顺序大致为：

```
1. 框架核心服务提供者
2. Package Discovery 发现的包服务提供者（nwidart 属于这一类）
3. bootstrap/providers.php 中的应用服务提供者
```

**因此，nwidart 主 SP 的 register 和 boot 都早于 AppServiceProvider。**

### 3.3 模块 SP 的注册与 boot 时机

nwidart 主 SP 在 **boot 阶段**扫描并启动所有启用的模块：

```
nwidart 主 SP boot 开始
   │
   ├─ 扫描 Modules/ 目录
   ├─ 读取 modules_statuses.json 状态
   │
   └─ 对每个启用的模块：
        │
        ├─ $module->register()
        │     └─ 注册模块的服务提供者（调用 $app->register()）
        │           └─ 执行模块 SP 的 register() 方法
        │
        └─ $module->boot()
              └─ 执行模块 SP 的 boot() 方法
```

### 3.4 完整时序图（后端启动阶段）

```
时间轴 ──────────────────────────────────────────────────────────▶

register 阶段
  ├─ nwidart 主 SP register()
  │     └─ 注册模块仓库、命令、Facade 等
  ├─ lavary/laravel-menu SP register()
  ├─ AppServiceProvider register()
  ├─ RouteServiceProvider register()
  └─ ... 其他应用 SP register()

boot 阶段
  ├─ nwidart 主 SP boot()
  │     ├─ 扫描所有模块
  │     └─ 对每个启用模块：
  │           ├─ 模块 SP register()    ← 模块 SP 的 register
  │           └─ 模块 SP boot()        ← 模块 SP 的 boot
  │                 ├─ 注册路由
  │                 ├─ 注册视图
  │                 ├─ Module::script() ← 注册前端脚本
  │                 ├─ Module::style()  ← 注册前端样式
  │                 └─ ... 其他模块启动逻辑
  │
  ├─ AppServiceProvider boot()
  │     ├─ 检查数据库是否就绪
  │     └─ addMenus()                  ← 构建 main_menu/setting_menu
  │           ├─ \Menu::make('main_menu', ...)
  │           └─ \Menu::make('setting_menu', ...)
  │
  ├─ RouteServiceProvider boot()
  │     └─ 加载路由文件
  │
  └─ ... 其他 SP boot()

应用 booted 阶段
  └─ 触发 app.booted 回调
```

> **关键结论**：模块服务提供者的 `boot()` **先于** AppServiceProvider 的 `boot()` 执行。
>
> 这意味着：
> - ✅ 模块可以在 boot 阶段注册前端资源（Module::script/style）
> - ❌ 模块不能直接通过 `Menu::get('main_menu')` 获取菜单（因为菜单还没创建）
> - ✅ 模块可以通过**配置合并**方式扩展菜单（在 AppServiceProvider 读取配置前合并）

---

## Module 静态资源注册生命周期

### 4.1 核心实现

[app/Services/Module/Module.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Services/Module/Module.php) 是资源注册器的核心：

```php
class Module
{
    /** @var array 已注册的脚本 */
    public static $scripts = [];

    /** @var array 已注册的样式 */
    public static $styles = [];

    /** @var array 已注册的设置项 */
    public static $settings = [];

    public static function script(string $name, string $path): self
    {
        static::$scripts[$name] = $path;
        return new static;
    }

    public static function style(string $name, string $path): self
    {
        static::$styles[$name] = $path;
        return new static;
    }

    public static function allScripts(): array
    {
        return static::$scripts;
    }

    public static function allStyles(): array
    {
        return static::$styles;
    }
}
```

同时提供 [ModuleFacade](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Services/Module/ModuleFacade.php) 方便使用：

```php
// 别名：ModuleFacade::script('name', $path)
// 等价于：Module::script('name', $path)
```

### 4.2 生命周期详解

**每一次 HTTP 请求**，Module 静态资源都会经历完整的生命周期：

| 阶段 | 时机 | 发生了什么 |
|------|------|------------|
| **初始化** | 请求开始，PHP 进程启动 | `Module::$scripts` 和 `Module::$styles` 为空数组 |
| **注册** | 服务提供者 boot 阶段 | 各模块的 SP 调用 `Module::script()` / `Module::style()` 注册资源 |
| **可用** | boot 阶段结束后至请求结束 | 静态属性中保存了所有已注册的资源路径 |
| **消费** | 视图渲染或资源控制器执行时 | 通过 `ModuleFacade::allScripts()` / `allStyles()` 获取 |
| **销毁** | 请求结束，PHP 进程退出 | 静态变量随进程销毁，下次请求重新注册 |

> **注意**：因为是静态属性，注册只在当前请求内有效，不跨请求持久化。每个请求都会重新注册所有启用模块的资源。

### 4.3 注册时机验证

从代码调用链可以确认注册发生在 boot 阶段：

1. nwidart 主 SP 在 boot 阶段启动模块
2. 模块 SP 在 boot 方法中调用 `ModuleFacade::script()`
3. 此时 `Module::$scripts` 被填充
4. 后续的控制器、视图都能访问到这些数据

### 4.4 消费场景

静态资源注册后，有两个消费场景：

**场景一：Blade 视图渲染时注入**

[resources/views/app.blade.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/views/app.blade.php) 中：

```html
<head>
    <!-- 主应用样式 -->
    @vite('resources/css/invoiceshelf.css')

    <!-- 模块样式（动态注入） -->
    @foreach(ModuleFacade::allStyles() as $name => $path)
        <link rel="stylesheet" href="/modules/styles/{{ $name }}">
    @endforeach
</head>
<body>
    <!-- 模块脚本（动态注入） -->
    @foreach (ModuleFacade::allScripts() as $name => $path)
        <script type="module" src="/modules/scripts/{{ $name }}"></script>
    @endforeach

    <!-- 主应用启动 -->
    <script type="module">
        window.InvoiceShelf.start()
    </script>
</body>
```

**场景二：资源控制器动态输出**

[ScriptController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Modules/ScriptController.php)：

```php
public function __invoke(Request $request, string $script)
{
    $path = Arr::get(ModuleFacade::allScripts(), $script);
    abort_if(is_null($path), 404);

    return response(
        file_get_contents($path),
        200,
        ['Content-Type' => 'application/javascript']
    )->setLastModified(DateTime::createFromFormat('U', filemtime($path)));
}
```

> **为什么不直接访问静态文件，而是通过控制器输出？**
> - 模块脚本可能存放在 `Modules/` 目录下，不对外公开
> - 通过控制器可以做权限检查、版本控制、CDN 转发等
> - 支持 Last-Modified 缓存协商

---

## 主菜单构建顺序深度解析

### 5.1 菜单系统基础

使用 **lavary/laravel-menu** 包，服务提供者注册在 [bootstrap/app.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/bootstrap/app.php#L30-L35)：

```php
->withProviders([
    ServiceProvider::class,  // Lavary\Menu\ServiceProvider
])
```

### 5.2 AppServiceProvider 中的菜单创建

[AppServiceProvider::addMenus()](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Providers/AppServiceProvider.php#L84-L105)：

```php
public function addMenus(): void
{
    // 主导航菜单
    \Menu::make('main_menu', function ($menu) {
        foreach (config('invoiceshelf.main_menu') as $data) {
            $this->generateMenu($menu, $data);
        }
    });

    // 设置菜单
    \Menu::make('setting_menu', function ($menu) {
        foreach (config('invoiceshelf.setting_menu') as $data) {
            $this->generateMenu($menu, $data);
        }
    });

    // 客户门户菜单
    \Menu::make('customer_portal_menu', function ($menu) {
        foreach (config('invoiceshelf.customer_menu') as $data) {
            $this->generateMenu($menu, $data);
        }
    });
}
```

关键特点：
- 菜单项**全部来自配置文件** `config('invoiceshelf.main_menu')`
- 只有在 `InstallUtils::isDbCreated()` 为 true 时才创建菜单
- 创建时机在 **AppServiceProvider 的 boot 阶段**

### 5.3 时序矛盾与解决方案

根据 3.4 节的时序分析，模块 SP boot **先于** AppServiceProvider boot。

这意味着模块 SP boot 时，`\Menu::get('main_menu')` 会返回 **null**（菜单还没创建）。

```
模块 SP boot  →  尝试 \Menu::get('main_menu')  →  null ❌
AppServiceProvider boot  →  addMenus()  →  \Menu::make('main_menu')  ✅
```

**那么模块如何扩展菜单？** 有两种可行方案：

#### 方案 A：配置合并（推荐用于静态菜单项）

模块在 **register 或 boot 阶段**将菜单项合并到配置数组中：

```php
// 模块服务提供者中
public function boot(): void
{
    // 合并菜单项到配置
    $existing = config('invoiceshelf.main_menu', []);
    $moduleMenu = [
        [
            'title' => 'mymodule::nav.title',
            'group' => 3,
            'link' => '/admin/my-module',
            'icon' => 'PuzzlePieceIcon',
            'name' => 'MyModule',
            'owner_only' => false,
            'ability' => 'view-my-module',
            'model' => \Modules\MyModule\Entities\MyModule::class,
        ],
    ];

    config()->set('invoiceshelf.main_menu', array_merge($existing, $moduleMenu));
}
```

工作原理：
1. 模块 SP boot 时合并配置
2. AppServiceProvider boot 时从配置读取菜单项（包含模块追加的）
3. 菜单创建时一并包含模块的菜单项

**优点**：简单直接，和核心菜单创建方式一致
**缺点**：只能添加静态菜单项，无法根据运行时条件动态添加

#### 方案 B：app->booted 回调（推荐用于动态菜单项）

模块在 boot 阶段监听 `app.booted` 事件，在应用启动完成后追加菜单：

```php
public function boot(): void
{
    $this->app->booted(function () {
        // 此时所有 SP 都已 boot，菜单已创建
        $menu = \Menu::get('main_menu');

        if ($menu) {
            $menu->add('mymodule::nav.title', '/admin/my-module')
                ->data('icon', 'PuzzlePieceIcon')
                ->data('name', 'MyModule')
                ->data('group', 3)
                ->data('owner_only', false)
                ->data('ability', 'view-my-module')
                ->data('model', \Modules\MyModule\Entities\MyModule::class);
        }
    });
}
```

工作原理：
1. 所有服务提供者 boot 完成后，触发 `app.booted`
2. 此时 AppServiceProvider 已经创建了菜单
3. 模块在回调中获取菜单实例并追加

**优点**：灵活，可以根据运行时条件动态添加
**缺点**：时机稍晚，但在控制器执行前仍然有效

### 5.4 菜单数据输出到前端

[BootstrapController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/General/BootstrapController.php) 使用 [GeneratesMenuTrait](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Traits/GeneratesMenuTrait.php) 输出菜单：

```php
public function generateMenu($key, $user)
{
    $new_items = [];
    $menu = \Menu::get($key);
    $items = $menu ? $menu->items->toArray() : [];

    foreach ($items as $data) {
        if ($user->checkAccess($data)) {  // 权限检查
            $new_items[] = [
                'title' => $data->title,
                'link' => $data->link->path['url'],
                'icon' => $data->data['icon'],
                'name' => $data->data['name'],
                'group' => $data->data['group'],
            ];
        }
    }

    return $new_items;
}
```

> **关键点**：BootstrapController 在**控制器阶段**执行，此时无论是方案 A 还是方案 B 添加的菜单项都已经在 Menu 对象中了。

### 5.5 菜单项数据结构

| 字段 | 说明 | 示例 |
|------|------|------|
| `title` | 菜单标题（翻译键） | `'navigation.dashboard'` |
| `link` | 路由路径 | `'/admin/dashboard'` |
| `icon` | 图标组件名 | `'HomeIcon'` |
| `name` | 唯一标识 | `'Dashboard'` |
| `group` | 分组编号 | `1` |
| `owner_only` | 是否仅所有者可见 | `false` |
| `ability` | 所需权限 | `'dashboard'` |
| `model` | 关联模型 | `Customer::class` |

---

## 前端资源动态挂载

### 6.1 前端启动钩子机制

[resources/scripts/InvoiceShelf.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/InvoiceShelf.js) 提供了模块化启动机制：

```javascript
export default class InvoiceShelf {
  constructor() {
    this.bootingCallbacks = []  // 启动回调队列
  }

  // 模块注册启动回调（在主应用启动前调用）
  booting(callback) {
    this.bootingCallbacks.push(callback)
  }

  // 执行所有回调（在 start() 中调用）
  executeCallbacks() {
    this.bootingCallbacks.forEach((callback) => {
      callback(app, router)  // 传入 Vue app 实例和 router 实例
    })
  }

  start() {
    this.executeCallbacks()  // 先执行模块回调
    // ... 然后初始化 Pinia、i18n、挂载应用
  }
}
```

### 6.2 为什么用 booting 钩子而不是直接执行？

因为模块脚本是在主应用脚本**之前**加载的。当模块脚本执行时，Vue 应用还没有创建，路由实例也不存在。

使用 `booting` 模式的好处：
1. **时序解耦**：模块脚本可以随时注册回调，不必关心主应用何时初始化
2. **统一入口**：所有模块都通过同一个钩子扩展，便于调试和管理
3. **依赖注入**：回调函数接收 `app` 和 `router` 参数，模块无需关心如何获取

### 6.3 模块脚本的加载顺序

在 Blade 模板中，脚本注入顺序为：

```html
<!-- 1. 模块脚本（按注册顺序） -->
@foreach (ModuleFacade::allScripts() as $name => $path)
    <script type="module" src="/modules/scripts/{{ $name }}"></script>
@endforeach

<!-- 2. 主应用启动 -->
<script type="module">
    window.InvoiceShelf.start()
</script>
```

因为是 `type="module"`，脚本会按顺序执行（尽管是异步加载）。

### 6.4 模块如何扩展前端

模块脚本可以在 `booting` 回调中做以下事情：

```javascript
window.InvoiceShelf.booting((app, router) => {
  // 1. 添加路由（Vue Router）
  router.addRoute('admin', {
    path: 'my-module',
    name: 'my-module.index',
    component: MyModulePage,
    meta: { ability: 'view-my-module' }
  })

  // 2. 注册全局组件
  app.component('my-module-widget', MyWidget)

  // 3. 添加 Pinia store
  // ...

  // 4. 添加 i18n 翻译
  // ...
})
```

---

## 从模块发现到 Vue 挂载的完整流程

### 7.1 后端完整链路（一次 HTTP 请求）

```
┌─────────────────────────────────────────────────────────────────┐
│                        阶段 1：请求开始                          │
│  - PHP 进程启动
│  - 加载 Composer 自动加载
│  - 创建 Laravel 应用实例
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      阶段 2：服务提供者注册                       │
│                                                                 │
│  nwidart 主 SP register()                                       │
│    └─ 注册模块仓库、命令、Facade                                 │
│                                                                 │
│  lavary/menu SP register()                                      │
│    └─ 注册 Menu 单例到容器                                       │
│                                                                 │
│  AppServiceProvider register()                                  │
│  ...其他应用 SP register()                                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      阶段 3：服务提供者启动                       │
│                                                                 │
│  nwidart 主 SP boot()                                           │
│    ├─ 扫描 Modules/ 目录                                        │
│    ├─ 读取模块状态文件                                           │
│    └─ 对每个启用模块：                                            │
│         ├─ 模块 SP register()                                   │
│         └─ 模块 SP boot()                                       │
│               ├─ 注册模块路由                                    │
│               ├─ 注册模块视图                                    │
│               ├─ Module::script('name', $path)  ← 脚本注册      │
│               ├─ Module::style('name', $path)   ← 样式注册      │
│               └─ [可选] 配置合并菜单                             │
│                                                                 │
│  AppServiceProvider boot()                                      │
│    ├─ 检查数据库是否就绪                                         │
│    └─ addMenus()  ← 从配置创建菜单（含模块合并的）                │
│                                                                 │
│  RouteServiceProvider boot()                                    │
│    └─ 加载核心路由文件                                           │
│                                                                 │
│  ...其他 SP boot()                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    阶段 4：应用启动完成 (booted)                  │
│                                                                 │
│  触发 app.booted 事件                                            │
│  模块如果注册了 booted 回调，在这里追加菜单                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      阶段 5：路由与控制器                         │
│                                                                 │
│  路由匹配 → 中间件 → 控制器                                       │
│                                                                 │
│  ├─ 如果是 Bootstrap API：                                       │
│  │     └─ 从 \Menu::get() 获取菜单数据                            │
│  │         └─ 权限过滤后返回 JSON                                │
│  │                                                               │
│  ├─ 如果是模块脚本请求 (/modules/scripts/{name})：               │
│  │     └─ ScriptController 从 Module::$scripts 查路径并输出       │
│  │                                                               │
│  ├─ 如果是页面请求 (HTML)：                                      │
│  │     └─ 渲染 Blade 视图                                        │
│  │           ├─ 遍历 Module::$styles 注入 <link>                │
│  │           └─ 遍历 Module::$scripts 注入 <script>              │
│  └─ ...其他请求                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Bootstrap API 路径核准

**重要结论**：Bootstrap 接口路径为 **`/api/v1/bootstrap`**，**没有** `admin` 前缀。

| 项目 | 真实值 | 所在文件 |
|------|--------|----------|
| 前端请求路径 | `GET /api/v1/bootstrap` | [global.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/stores/global.js#L51) |
| 后端路由定义 | `Route::get('/bootstrap', BootstrapController::class)` | [api.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/routes/api.php#L197) |
| 路由中间件 | `auth:sanctum`, `company`, `bouncer` | [api.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/routes/api.php#L191-L192) |
| 控制器命名空间 | `App\Http\Controllers\V1\Admin\General\BootstrapController` | [BootstrapController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/General/BootstrapController.php) |

> **容易混淆的点**：
> - ✅ API 路径：`/api/v1/bootstrap`（没有 admin 前缀）
> - ✅ 控制器目录：`V1/Admin/General/`（命名空间有 Admin）
> - ✅ 前端页面路由：`/admin/dashboard`（有 admin 前缀）
> - ✅ 菜单 link 字段：`/admin/dashboard`（与页面路由一致，有 admin 前缀）
>
> 命名空间的 `Admin` 只是代码组织方式，不反映在 URL 路径中。

#### 路由结构详解

在 [routes/api.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/routes/api.php) 中，路由层级为：

```
/api                          ← RouteServiceProvider 添加的 api 前缀
  /v1                          ← api.php 中的 Route::prefix('/v1')
    /bootstrap                 ← admin 端 bootstrap 接口（在 auth:sanctum + company + bouncer 中间件内）
    /dashboard
    /customers
    /invoices
    ...
    /modules                   ← 模块管理相关接口
      /
      /{module}/enable
      /{module}/disable
      ...
    /{company:slug}/customer   ← 客户端门户接口
      /bootstrap               ← 客户端 bootstrap 接口
      ...
```

两个 Bootstrap 接口对比：

| 接口 | 路径 | 用途 | 菜单类型 |
|------|------|------|----------|
| 管理端 Bootstrap | `GET /api/v1/bootstrap` | 后台管理初始化数据 | `main_menu`, `setting_menu` |
| 客户端 Bootstrap | `GET /api/v1/{company:slug}/customer/bootstrap` | 客户门户初始化数据 | `customer_portal_menu` |

### 7.3 前端完整链路（浏览器侧）

```
┌─────────────────────────────────────────────────────────────────┐
│                     阶段 1：HTML 解析                             │
│                                                                 │
│  浏览器接收 HTML 响应                                             │
│  解析 <head> 中的样式                                             │
│    ├─ Vite 主样式                                                │
│    └─ 模块样式（每个模块一个 <link>）                              │
│       路径：/modules/styles/{name}                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     阶段 2：脚本加载                              │
│                                                                 │
│  按顺序加载脚本（type="module"，保持顺序）                        │
│                                                                 │
│  1. Vite 主脚本                                                  │
│     └─ 创建 window.InvoiceShelf 实例                             │
│        └─ bootingCallbacks = []                                  │
│                                                                 │
│  2. 模块脚本（每个启用模块一个）                                  │
│     路径：/modules/scripts/{name}                                │
│     └─ 模块脚本执行                                               │
│        └─ window.InvoiceShelf.booting(callback)                  │
│           └─ callback 被推入 bootingCallbacks 队列               │
│                                                                 │
│  3. 内联启动脚本                                                 │
│     └─ window.InvoiceShelf.start()  ← 开始启动                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   阶段 3：应用启动 (start)                       │
│                                                                 │
│  InvoiceShelf.start() 执行                                       │
│    │
│    ├─ executeCallbacks()  ← 执行所有模块回调                     │
│    │     ├─ 模块A回调(app, router)                               │
│    │     │     ├─ 添加路由（router.addRoute）                     │
│    │     │     └─ 注册组件（app.component）                       │
│    │     ├─ 模块B回调(app, router)                               │
│    │     └─ ...                                                  │
│    │
│    ├─ 初始化 Pinia                                               │
│    ├─ 初始化 i18n                                                │
│    ├─ 安装路由                                                   │
│    └─ app.mount('body')  ← Vue 应用挂载                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              阶段 4：LayoutBasic mounted + Bootstrap             │
│                                                                 │
│  LayoutBasic 组件 onMounted                                       │
│    │
│    └─ globalStore.bootstrap()  ← 发起 Bootstrap 请求             │
│          │
│          ├─ HTTP 请求：GET /api/v1/bootstrap                     │
│          │   (axios 实例，相对路径，无 baseURL)                    │
│          │   请求头：Authorization, company                      │
│          │
│          │                                    ┌──────────────┐  │
│          │                                    │   后端处理    │  │
│          │                                    │  Bootstrap   │  │
│          │                                    │ Controller   │  │
│          │                                    │  生成菜单数据  │  │
│          │                                    └──────┬───────┘  │
│          │                                           │          │
│          ├─ 响应 JSON（部分关键字段）                  │          │
│          │     ├─ main_menu: [...]                    │          │
│          │     │   每项：{title, link, icon,          │          │
│          │     │          name, group}                │          │
│          │     ├─ setting_menu: [...]                 │          │
│          │     └─ modules: ["ModuleA", ...]           │          │
│          │
│          ├─ this.mainMenu = response.data.main_menu   │
│          ├─ this.settingMenu = response.data.setting_menu
│          └─ moduleStore.enableModules = response.data.modules
│                                                                 │
│  TheSiteSidebar 组件渲染（响应式）                               │
│    └─ globalStore.menuGroups                                    │
│       (getter: _.groupBy(mainMenu, 'group'))                   │
│       └─ 每组渲染一个 <nav>                                      │
│            └─ 每项渲染 <router-link :to="item.link">            │
│               （item.link 如 /admin/dashboard，带 admin 前缀）   │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4 菜单数据流向详解（从数据库到侧边栏）

菜单数据经过 **6 层转换** 才最终渲染到页面上：

```
  第 1 层          第 2 层          第 3 层          第 4 层
配置数组  ───▶  Menu 对象  ───▶  Bootstrap  ───▶  Pinia Store
config/invoiceshelf.php   lavary/menu       控制器 JSON       global store
                              items

  第 5 层          第 6 层
menuGroups  ───▶  <router-link>
getter 计算         侧边栏渲染
```

各层详细说明：

| 层级 | 数据形态 | 所在位置 | 关键操作 |
|------|----------|----------|----------|
| **第 1 层：配置源** | PHP 数组 | [config/invoiceshelf.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/config/invoiceshelf.php#L308) 的 `main_menu` | 静态定义菜单项的完整属性 |
| **第 2 层：Menu 对象** | `Lavary\Menu\Menu` 实例 | AppServiceProvider boot 阶段的 `\Menu::make('main_menu', ...)` | 从配置创建菜单对象，支持动态追加 |
| **第 3 层：API 输出** | JSON 数组 | [BootstrapController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/General/BootstrapController.php#L32-L34) 的 `generateMenu()` | 权限过滤，只返回用户有权限的菜单项 |
| **第 4 层：Pinia State** | JS 数组 | [global.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/stores/global.js#L30-L31) 的 `mainMenu` | 前端状态存储，响应式 |
| **第 5 层：Getter 分组** | 按 group 分组的二维数组 | [global.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/stores/global.js#L42-L44) 的 `menuGroups` | `_.groupBy(state.mainMenu, 'group')` 按组归类 |
| **第 6 层：侧边栏渲染** | Vue 组件 | [TheSiteSidebar.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/layouts/partials/TheSiteSidebar.vue#L74-L103) | 双层 v-for 渲染，生成 `<router-link>` |

#### 每层的数据结构示例

**第 1 层（配置）**：
```php
// config/invoiceshelf.php
[
    'title' => 'navigation.dashboard',
    'group' => 1,
    'link' => '/admin/dashboard',
    'icon' => 'HomeIcon',
    'name' => 'Dashboard',
    'owner_only' => false,
    'ability' => 'dashboard',
    'model' => '',
]
```

**第 2 层（Menu 对象）**：
```php
// 通过 Menu::get('main_menu')->items 访问
// Item 对象属性：title, link, data(icon/name/group/...)
```

**第 3 层（API JSON）**：
```json
{
  "main_menu": [
    {
      "title": "navigation.dashboard",
      "link": "/admin/dashboard",
      "icon": "HomeIcon",
      "name": "Dashboard",
      "group": 1
    }
  ]
}
```

**第 4-6 层（前端）**：
```javascript
// state.mainMenu (第4层)
[{ title: 'navigation.dashboard', link: '/admin/dashboard', icon: 'HomeIcon', name: 'Dashboard', group: 1 }, ...]

// getters.menuGroups (第5层)
[
  [ /* group 1 的菜单项 */ ],
  [ /* group 2 的菜单项 */ ],
  [ /* group 3 的菜单项 */ ],
]

// 渲染结果 (第6层)
// <nav> 包裹每组，内部是多个 <router-link>
```

### 7.5 关键节点对照表

| 节点 | 后端位置 | 前端位置 |
|------|----------|----------|
| 模块文件被发现 | nwidart 主 SP boot 阶段 | - |
| 模块 SP 被注册 | nwidart 主 SP boot 阶段 | - |
| 前端资源被注册 | 模块 SP boot 阶段（Module::script） | - |
| 菜单被创建 | AppServiceProvider boot 阶段，`addMenus()` | - |
| 模块菜单被合并 | 模块 SP boot 阶段（配置合并） | - |
| HTML 注入资源标签 | 视图渲染阶段 | HTML 解析阶段 |
| 模块脚本执行 | - | 脚本加载阶段 |
| 模块回调注册 | - | `InvoiceShelf.booting()` 调用时 |
| 模块路由/组件生效 | - | `start()` → `executeCallbacks()` |
| Bootstrap API 请求 | `GET /api/v1/bootstrap` | `globalStore.bootstrap()` |
| 菜单数据到达前端 | Bootstrap 控制器返回 | axios 响应 |
| 菜单存入 Store | - | `this.mainMenu = response.data.main_menu` |
| 菜单渲染到页面 | - | TheSiteSidebar.vue 中 v-for 渲染 |

### 7.3 关键节点对照表

| 节点 | 后端位置 | 前端位置 |
|------|----------|----------|
| 模块文件被发现 | nwidart 主 SP boot 阶段 | - |
| 模块 SP 被注册 | nwidart 主 SP boot 阶段 | - |
| 前端资源被注册 | 模块 SP boot 阶段（Module::script） | - |
| 菜单被创建 | AppServiceProvider boot 阶段 | - |
| 模块菜单被合并 | 模块 SP boot 阶段（配置合并） | - |
| HTML 注入资源标签 | 视图渲染阶段 | HTML 解析阶段 |
| 模块脚本执行 | - | 脚本加载阶段 |
| 模块回调注册 | - | booting() 调用时 |
| 模块路由/组件生效 | - | start() → executeCallbacks() |
| 菜单数据到达前端 | Bootstrap 控制器 | bootstrap API 响应 |
| 菜单渲染到页面 | - | 侧边栏组件 render |

---

## 模块安装与启用流程

### 8.1 安装流程

[ModuleInstaller](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Space/ModuleInstaller.php) 类负责模块安装：

1. **下载/上传** ZIP 包
2. **解压** 到临时目录
3. **复制** 到 `Modules/模块名/`
4. **完成安装**：
   - `Module::register()` — 重新扫描注册模块
   - `module:migrate` — 运行迁移
   - `module:seed` — 运行填充
   - `module:enable` — 启用模块
   - 更新数据库 `modules` 表
   - 触发 `ModuleInstalledEvent` 和 `ModuleEnabledEvent`

### 8.2 启用/禁用流程

**启用**：[EnableModuleController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/Modules/EnableModuleController.php)

```php
public function __invoke(Request $request, string $module)
{
    $this->authorize('manage modules');

    $module = ModelsModule::where('name', $module)->first();
    $module->update(['enabled' => true]);

    $installedModule = Module::find($module->name);
    $installedModule->enable();  // 更新文件状态

    ModuleEnabledEvent::dispatch($module);

    return response()->json(['success' => true]);
}
```

> **注意**：启用/禁用只修改状态，不影响当前请求的模块加载。下一次请求才会加载新启用的模块。

### 8.3 事件系统

| 事件 | 触发时机 | 文件 |
|------|----------|------|
| `ModuleInstalledEvent` | 模块安装完成时 | [ModuleInstalledEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Events/ModuleInstalledEvent.php) |
| `ModuleEnabledEvent` | 模块启用时 | [ModuleEnabledEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Events/ModuleEnabledEvent.php) |
| `ModuleDisabledEvent` | 模块禁用时 | [ModuleDisabledEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Events/ModuleDisabledEvent.php) |

---

## 模块开发标准模式

### 9.1 模块目录结构

```
Modules/
  └─ MyModule/
     ├─ Providers/
     │  └─ MyModuleServiceProvider.php   # 服务提供者（核心入口）
     ├─ Http/
     │  └─ Controllers/                  # 控制器
     ├─ Entities/                        # Eloquent 模型
     ├─ Database/
     │  ├─ Migrations/                   # 数据库迁移
     │  └─ Seeders/                      # 数据填充
     ├─ Routes/
     │  ├─ api.php                       # API 路由
     │  └─ web.php                       # Web 路由
     ├─ Resources/
     │  ├─ views/                        # Blade 视图
     │  ├─ scripts/                      # 前端脚本
     │  │  └─ my-module.js               # 模块前端入口
     │  └─ lang/                         # 语言文件
     ├─ Config/
     │  └─ config.php                    # 模块配置
     └─ module.json                      # 模块元数据
```

### 9.2 模块服务提供者模板

```php
<?php

namespace Modules\MyModule\Providers;

use App\Services\Module\ModuleFacade;
use Illuminate\Support\ServiceProvider;

class MyModuleServiceProvider extends ServiceProvider
{
    /**
     * 在 register 阶段可以做的事情：
     * - 绑定服务到容器
     * - 注册单例
     * - 合并配置（如果需要在其他 SP boot 前生效）
     */
    public function register(): void
    {
        // 合并模块配置
        $this->mergeConfigFrom(
            __DIR__ . '/../Config/config.php',
            'mymodule'
        );
    }

    /**
     * 在 boot 阶段可以做的事情：
     * - 注册路由、视图、迁移、翻译
     * - 注册前端资源（Module::script/style）
     * - 扩展菜单（配置合并方式）
     * - 注册事件监听器
     */
    public function boot(): void
    {
        // 1. 注册前端资源
        ModuleFacade::script('my-module', __DIR__ . '/../Resources/scripts/my-module.js');
        ModuleFacade::style('my-module', __DIR__ . '/../Resources/css/my-module.css');

        // 2. 扩展菜单（方案 A：配置合并）
        $this->extendMenu();

        // 3. 加载路由
        $this->loadRoutesFrom(__DIR__ . '/../Routes/api.php');
        $this->loadRoutesFrom(__DIR__ . '/../Routes/web.php');

        // 4. 加载视图
        $this->loadViewsFrom(__DIR__ . '/../Resources/views', 'mymodule');

        // 5. 加载迁移
        $this->loadMigrationsFrom(__DIR__ . '/../Database/Migrations');

        // 6. 加载翻译
        $this->loadTranslationsFrom(__DIR__ . '/../Resources/lang', 'mymodule');
    }

    /**
     * 通过配置合并方式扩展菜单
     */
    protected function extendMenu(): void
    {
        $mainMenu = config('invoiceshelf.main_menu', []);

        $moduleMenu = [
            [
                'title' => 'mymodule::nav.title',
                'group' => 3,
                'link' => '/admin/my-module',
                'icon' => 'PuzzlePieceIcon',
                'name' => 'MyModule',
                'owner_only' => false,
                'ability' => 'view-my-module',
                'model' => \Modules\MyModule\Entities\MyModule::class,
            ],
        ];

        config()->set('invoiceshelf.main_menu', array_merge($mainMenu, $moduleMenu));
    }
}
```

### 9.3 模块前端入口模板

```javascript
// Modules/MyModule/Resources/scripts/my-module.js

import MyModulePage from './views/MyModulePage.vue'
import { defineStore } from 'pinia'

/**
 * 使用 booting 钩子在主应用启动前注册扩展
 * @param {import('vue').App} app - Vue 应用实例
 * @param {import('vue-router').Router} router - 路由实例
 */
window.InvoiceShelf.booting((app, router) => {
  // 1. 添加路由到 admin 路由组
  router.addRoute('admin', {
    path: 'my-module',
    name: 'my-module.index',
    component: MyModulePage,
    meta: {
      ability: 'view-my-module',
      title: 'mymodule::nav.title',
    },
  })

  // 2. 注册全局组件（可选）
  // app.component('my-module-widget', MyWidget)

  // 3. 添加 Pinia store（可选）
  // const useMyModuleStore = defineStore('myModule', { ... })
})
```

### 9.4 动态菜单扩展模板（方案 B）

如果需要根据运行时条件动态添加菜单项，使用 `app.booted` 回调：

```php
public function boot(): void
{
    // ... 其他 boot 逻辑

    // 方案 B：通过 app.booted 回调动态追加菜单
    $this->app->booted(function () {
        if (! app()->bound('menu')) {
            return;
        }

        $menu = \Menu::get('main_menu');

        if (! $menu) {
            return;
        }

        // 可以根据条件动态添加
        if (someCondition()) {
            $menu->add('mymodule::nav.title', '/admin/my-module')
                ->data('icon', 'PuzzlePieceIcon')
                ->data('name', 'MyModule')
                ->data('group', 3)
                ->data('owner_only', false)
                ->data('ability', 'view-my-module')
                ->data('model', \Modules\MyModule\Entities\MyModule::class);
        }
    });
}
```

---

## 总结

### 核心设计模式

| 设计模式 | 用途 | 典型场景 |
|----------|------|----------|
| **静态注册器模式** | 模块资源（脚本/样式）注册 | Module::$scripts, Module::$styles |
| **启动钩子模式** | 前端模块扩展 | InvoiceShelf.booting(callback) |
| **配置合并模式** | 菜单等配置扩展 | config()->set('invoiceshelf.main_menu', ...) |
| **应用启动回调** | 需延后执行的逻辑 | $app->booted(function() { ... }) |

### 关键时序速查

```
后端（按时间顺序）：
  nwidart 主 SP register
  AppServiceProvider register
  nwidart 主 SP boot
    → 模块 SP register
    → 模块 SP boot
      → Module::script/style() 注册资源
      → 配置合并菜单
  AppServiceProvider boot
    → addMenus() 创建菜单
  app.booted 回调
    → 动态追加菜单（方案 B）
  路由匹配 → 控制器 → 响应

前端（按时间顺序）：
  HTML 解析 → 样式加载
  Vite 主脚本 → 创建 InvoiceShelf 实例
  模块脚本 → InvoiceShelf.booting() 注册回调
  start() → executeCallbacks() → 模块路由/组件生效
  Vue 挂载 → LayoutBasic 渲染
  Bootstrap API → 菜单数据 → 侧边栏渲染
```

### 为什么这样设计？

1. **后端层面**：基于 `nwidart/laravel-modules` + 自研静态注册器，充分利用 Laravel 服务提供者生命周期
2. **前端层面**：`booting` 钩子模式解决了模块脚本先加载、主应用后初始化的时序问题
3. **菜单系统**：配置驱动的菜单创建 + 多种扩展方式，兼顾简单性和灵活性
4. **资源加载**：通过控制器动态输出模块资源，支持缓存协商和权限控制

这套设计既保持了 Laravel 生态的惯用模式，又通过前端启动钩子实现了 Vue 应用的动态扩展，是一套相对完整且解耦的模块化方案。
