# InvoiceShelf 模块加载器深度解析

本文档深入分析 InvoiceShelf 应用的模块化扩展机制，涵盖模块发现、服务注册、前端挂载和菜单集成的完整流程。

## 目录

1. [整体架构概述](#整体架构概述)
2. [模块发现与加载器](#模块发现与加载器)
3. [服务提供者与模块注册](#服务提供者与模块注册)
4. [前端资源动态挂载](#前端资源动态挂载)
5. [主菜单生成与扩展](#主菜单生成与扩展)
6. [模块安装与启用流程](#模块安装与启用流程)
7. [完整调用链路](#完整调用链路)

---

## 整体架构概述

InvoiceShelf 采用 **nwidart/laravel-modules** 作为底层模块框架（通过 `invoiceshelf/modules` 包引入），配合自研的前端资源注册机制和菜单系统，实现了完整的模块化扩展能力。

```
                    ┌──────────────────────────┐
                    │   nwidart/laravel-modules │
                    │      (模块框架内核)        │
                    └───────────┬──────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   模块服务提供者 (SP)      │
                    │  - 注册路由/迁移/视图      │
                    │  - 注册脚本/样式          │
                    │  - 扩展菜单               │
                    └───────────┬──────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐  ┌───────▼───────┐  ┌──────────▼─────────┐
│  前端资源注册器    │  │  菜单系统     │  │   路由/数据库等    │
│  Module Service   │  │ lavary/menu   │  │  (Laravel 原生)    │
│  - Scripts        │  │  - main_menu  │  └────────────────────┘
│  - Styles         │  │  - setting_menu│
└─────────┬─────────┘  └───────┬───────┘
          │                     │
          ▼                     ▼
   Blade 模板注入         Bootstrap API
   (页面加载时)          (前端初始化)
```

---

## 模块发现与加载器

### 1.1 核心依赖包

项目通过 [composer.json](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/composer.json#L17) 引入模块框架：

```json
"invoiceshelf/modules": "^1.0.0"
```

这是 `nwidart/laravel-modules` 的定制版本，提供完整的模块生命周期管理。

### 1.2 模块配置

模块的核心配置位于 [config/modules.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/config/modules.php)：

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `namespace` | 模块命名空间 | `Modules` |
| `paths.modules` | 模块存放目录 | `base_path('Modules')` |
| `paths.assets` | 模块资源发布目录 | `public_path('modules')` |
| `activator` | 激活器类型 | `file` |
| `scan.enabled` | 是否扫描 vendor 目录 | `false` |

### 1.3 模块状态管理

使用 **FileActivator** 管理模块启用状态：

- 状态文件：`storage/app/modules_statuses.json`
- 缓存键：`activator.installed`
- 缓存有效期：604800 秒（7天）

同时，数据库中也有 [modules 表](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/database/migrations/2021_12_15_053223_create_modules_table.php) 对应 [Module 模型](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Models/Module.php)，存储模块的版本、安装状态、启用状态等信息。

### 1.4 自动加载

Composer 自动加载配置：

```json
"autoload": {
    "psr-4": {
        "Modules\\": "Modules/"
    }
}
```

这意味着 `Modules\` 命名空间下的所有类都会从 `Modules/` 目录自动加载。

---

## 服务提供者与模块注册

### 2.1 模块服务提供者的角色

每个模块都有自己的服务提供者（通常位于 `Modules/模块名/Providers/` 目录），它是模块与主应用交互的核心入口。服务提供者在 `register()` 和 `boot()` 阶段执行不同的逻辑：

- **register 阶段**：绑定服务到容器
- **boot 阶段**：执行启动逻辑，如注册路由、视图、菜单、前端资源等

### 2.2 前端资源注册器

应用自研了一套前端资源注册机制，核心类是 [app/Services/Module/Module.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Services/Module/Module.php)：

```php
class Module
{
    public static $scripts = [];  // 已注册的脚本
    public static $styles = [];   // 已注册的样式
    public static $settings = []; // 已注册的设置项

    // 注册脚本
    public static function script($name, $path)
    {
        static::$scripts[$name] = $path;
        return new static;
    }

    // 注册样式
    public static function style($name, $path)
    {
        static::$styles[$name] = $path;
        return new static;
    }
}
```

同时提供 [ModuleFacade](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Services/Module/ModuleFacade.php) 便于使用：

```php
// 在模块服务提供者中使用
\App\Services\Module\ModuleFacade::script('module-name', __DIR__ . '/../Resources/scripts/module.js');
\App\Services\Module\ModuleFacade::style('module-name', __DIR__ . '/../Resources/css/module.css');
```

> **设计要点**：使用静态属性存储注册信息，因为服务提供者的 boot 方法在请求生命周期早期执行，后续的控制器和视图都能访问到这些静态数据。

---

## 前端资源动态挂载

### 3.1 资源路由

模块的脚本和样式通过专属路由提供，定义在 [routes/web.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/routes/web.php#L24-L28)：

```php
// 模块样式
Route::get('/modules/styles/{style}', StyleController::class);

// 模块脚本
Route::get('/modules/scripts/{script}', ScriptController::class);
```

### 3.2 资源控制器

[ScriptController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Modules/ScriptController.php) 和 [StyleController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Modules/StyleController.php) 负责动态输出资源：

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

> **设计要点**：支持 Last-Modified 头，便于浏览器缓存；同时支持外部 URL（CDN 资源）。

### 3.3 Blade 模板注入

在 [resources/views/app.blade.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/views/app.blade.php) 中动态注入所有已注册的模块资源：

```html
<!-- 模块样式（注入到 <head> 中） -->
@foreach(\App\Services\Module\ModuleFacade::allStyles() as $name => $path)
    <link rel="stylesheet" href="/modules/styles/{{ $name }}">
@endforeach

<!-- 模块脚本（注入到 <body> 末尾，主应用脚本之前） -->
@foreach (\App\Services\Module\ModuleFacade::allScripts() as $name => $path)
    @if (\Illuminate\Support\Str::startsWith($path, ['http://', 'https://']))
        <script type="module" src="{!! $path !!}"></script>
    @else
        <script type="module" src="/modules/scripts/{{ $name }}"></script>
    @endif
@endforeach

<script type="module">
    window.InvoiceShelf.start()
</script>
```

### 3.4 前端应用启动钩子

[resources/scripts/InvoiceShelf.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/InvoiceShelf.js) 提供了模块化的前端启动机制：

```javascript
export default class InvoiceShelf {
  constructor() {
    this.bootingCallbacks = []  // 启动回调队列
  }

  // 模块注册启动回调
  booting(callback) {
    this.bootingCallbacks.push(callback)
  }

  // 执行所有回调
  executeCallbacks() {
    this.bootingCallbacks.forEach((callback) => {
      callback(app, router)  // 传入 Vue app 实例和 router 实例
    })
  }

  start() {
    this.executeCallbacks()  // 先执行模块回调
    // ... 然后初始化 Vue 应用
  }
}
```

模块的脚本可以这样扩展前端：

```javascript
// 在模块脚本中
window.InvoiceShelf.booting((app, router) => {
  // 注册全局组件
  app.component('module-component', MyComponent)
  
  // 添加路由
  router.addRoute('admin', {
    path: 'my-module',
    component: MyModulePage
  })
  
  // 添加 Pinia store
  // ...
})
```

> **设计要点**：使用 `booting` 钩子模式，模块脚本在主应用启动前注册回调，确保模块的路由、组件等在应用挂载前就已就绪。

---

## 主菜单生成与扩展

### 5.1 菜单系统基础

应用使用 **lavary/laravel-menu** 包作为菜单系统的底层。服务提供者注册在 [bootstrap/app.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/bootstrap/app.php#L30-L35)：

```php
->withProviders([
    ServiceProvider::class,  // Lavary\Menu\ServiceProvider
])
```

### 5.2 菜单初始化

菜单在 [AppServiceProvider::addMenus()](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Providers/AppServiceProvider.php#L84-L105) 方法中构建：

```php
public function addMenus()
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

### 5.3 菜单项数据结构

默认菜单项配置在 [config/invoiceshelf.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/config/invoiceshelf.php#L308-L434) 的 `main_menu` 数组中：

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

### 5.4 模块如何扩展菜单

模块可以在其服务提供者的 `boot()` 方法中获取菜单实例并添加菜单项：

```php
// 在模块的服务提供者中
public function boot(): void
{
    if (app()->bound('menu')) {
        $menu = \Menu::get('main_menu');
        
        if ($menu) {
            $menu->add('我的模块', '/admin/my-module')
                ->data('icon', 'PuzzlePieceIcon')
                ->data('name', 'MyModule')
                ->data('owner_only', false)
                ->data('ability', 'view-my-module')
                ->data('model', \Modules\MyModule\Entities\MyModule::class)
                ->data('group', 3);
        }
    }
}
```

> **注意**：模块服务提供者的 boot 顺序很重要。如果模块的服务提供者在 `AppServiceProvider` 之后 boot，就可以直接获取 `main_menu` 实例并添加项目。

### 5.5 菜单数据输出到前端

[BootstrapController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/General/BootstrapController.php) 使用 [GeneratesMenuTrait](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Traits/GeneratesMenuTrait.php) 将菜单转换为前端可用的数据：

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

Bootstrap 接口返回的数据中包含：

```php
return response()->json([
    // ...
    'main_menu' => $main_menu,
    'setting_menu' => $setting_menu,
    'modules' => Module::where('enabled', true)->pluck('name'),
    // ...
]);
```

### 5.6 前端菜单渲染

前端通过 Pinia store 管理菜单状态，核心逻辑在 [resources/scripts/admin/stores/global.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/stores/global.js)：

```javascript
// state
mainMenu: [],
settingMenu: [],

// getter - 按 group 分组
menuGroups: (state) => {
  return Object.values(_.groupBy(state.mainMenu, 'group'))
},

// bootstrap 时赋值
this.mainMenu = response.data.main_menu
this.settingMenu = response.data.setting_menu
```

侧边栏组件 [TheSiteSidebar.vue](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/layouts/partials/TheSiteSidebar.vue) 按组渲染菜单项：

```vue
<nav v-for="menu in globalStore.menuGroups" :key="menu">
  <router-link v-for="item in menu" :key="item.name" :to="item.link">
    <BaseIcon :name="item.icon" />
    {{ $t(item.title) }}
  </router-link>
</nav>
```

### 5.7 模块列表与启用状态

模块的启用状态通过 `moduleStore` 管理，见 [resources/scripts/admin/stores/module.js](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/resources/scripts/admin/stores/module.js)：

```javascript
state: () => ({
  enableModules: []  // 已启用的模块名称数组
}),

getters: {
  // 示例：检查特定模块是否启用
  salesTaxUSEnabled: (state) => state.enableModules.includes('SalesTaxUS'),
},
```

---

## 模块安装与启用流程

### 6.1 安装流程

[ModuleInstaller](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Space/ModuleInstaller.php) 类负责模块的安装，完整流程如下：

1. **下载/上传**：从市场下载或本地上传模块 ZIP 包
2. **解压**：解压到临时目录
3. **复制文件**：复制到 `Modules/模块名/` 目录
4. **完成安装**：
   - 调用 `Module::register()` 重新扫描注册模块
   - 执行 `module:migrate` 运行模块迁移
   - 执行 `module:seed` 运行模块填充
   - 执行 `module:enable` 启用模块
   - 更新数据库 `modules` 表记录
   - 触发 `ModuleInstalledEvent` 和 `ModuleEnabledEvent`

```php
public static function complete($module, $version)
{
    Module::register();  // 重新注册模块

    Artisan::call("module:migrate $module --force");
    Artisan::call("module:seed $module --force");
    Artisan::call("module:enable $module");

    $module = ModelsModule::updateOrCreate(
        ['name' => $module],
        ['version' => $version, 'installed' => true, 'enabled' => true]
    );

    ModuleInstalledEvent::dispatch($module);
    ModuleEnabledEvent::dispatch($module);

    return true;
}
```

### 6.2 启用/禁用流程

**启用** - [EnableModuleController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/Modules/EnableModuleController.php)：

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

**禁用** - [DisableModuleController](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Http/Controllers/V1/Admin/Modules/DisableModuleController.php) 逻辑类似。

### 6.3 事件系统

应用定义了三个模块相关事件：

| 事件 | 触发时机 | 文件 |
|------|----------|------|
| `ModuleInstalledEvent` | 模块安装完成时 | [ModuleInstalledEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Events/ModuleInstalledEvent.php) |
| `ModuleEnabledEvent` | 模块启用时 | [ModuleEnabledEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Events/ModuleEnabledEvent.php) |
| `ModuleDisabledEvent` | 模块禁用时 | [ModuleDisabledEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/122-InvoiceShelf/app/Events/ModuleDisabledEvent.php) |

模块和主应用都可以监听这些事件来执行自定义逻辑。

---

## 完整调用链路

### 7.1 后端请求生命周期中的模块流程

```
请求到达
   │
   ▼
Laravel 启动
   │
   ├─▶ 注册所有 ServiceProvider
   │     │
   │     ├─▶ nwidart/laravel-modules SP 注册
   │     │     └─▶ 扫描 Modules/ 目录
   │     │           └─▶ 注册各模块的 SP 到容器
   │     │
   │     └─▶ AppServiceProvider 注册
   │
   ├─▶ boot 所有 ServiceProvider
   │     │
   │     ├─▶ 模块 SP boot 阶段
   │     │     ├─▶ 注册模块路由
   │     │     ├─▶ 注册模块视图
   │     │     ├─▶ 注册模块迁移
   │     │     ├─▶ 调用 Module::script() 注册前端脚本
   │     │     ├─▶ 调用 Module::style() 注册前端样式
   │     │     └─▶ 向 Menu 实例添加菜单项
   │     │
   │     └─▶ AppServiceProvider boot 阶段
   │           ├─▶ 检查数据库是否就绪
   │           └─▶ addMenus() - 构建主菜单/设置菜单
   │
   ▼
路由匹配
   │
   ▼
中间件处理
   │
   ▼
控制器执行
   │
   ├─▶ BootstrapController
   │     └─▶ 生成 main_menu/setting_menu 数据
   │         └─▶ 权限过滤后返回给前端
   │
   ├─▶ ScriptController
   │     └─▶ 从 Module::$scripts 查找路径并输出 JS
   │
   └─▶ StyleController
         └─▶ 从 Module::$styles 查找路径并输出 CSS
```

### 7.2 前端启动流程

```
页面加载
   │
   ├─▶ 加载主应用 JS (Vite)
   │
   ├─▶ 依次加载模块脚本 (每个启用的模块一个)
   │     │
   │     └─▶ 模块脚本执行
   │           └─▶ window.InvoiceShelf.booting(callback) 注册回调
   │
   ▼
window.InvoiceShelf.start() 调用
   │
   ├─▶ executeCallbacks() - 执行所有模块回调
   │     ├─▶ 模块注册 Vue 组件
   │     ├─▶ 模块添加路由
   │     └─▶ 模块注册 Pinia store
   │
   ├─▶ 初始化 i18n
   ├─▶ 初始化 Pinia
   ├─▶ 安装路由
   │
   ▼
app.mount('body') - 应用挂载
   │
   ▼
LayoutBasic 组件 mounted
   │
   └─▶ globalStore.bootstrap()
         │
         ├─▶ 调用 /api/v1/bootstrap 接口
         │
         ├─▶ 获取 main_menu 数据存入 store
         ├─▶ 获取 setting_menu 数据存入 store
         ├─▶ 获取 modules (已启用列表) 存入 moduleStore
         │
         ▼
      侧边栏渲染菜单
```

### 7.3 模块开发的标准模式

一个典型的模块目录结构：

```
Modules/
  └─ MyModule/
     ├─ Providers/
     │  └─ MyModuleServiceProvider.php   # 服务提供者（核心）
     ├─ Http/
     │  └─ Controllers/                  # 控制器
     ├─ Entities/                        # 模型 (Eloquent)
     ├─ Database/
     │  ├─ Migrations/                   # 数据库迁移
     │  └─ Seeders/                      # 数据填充
     ├─ Routes/
     │  ├─ api.php                       # API 路由
     │  └─ web.php                       # Web 路由
     ├─ Resources/
     │  ├─ views/                        # Blade 视图
     │  ├─ scripts/                      # 前端脚本
     │  │  └─ my-module.js               # 模块入口脚本
     │  └─ lang/                         # 语言文件
     ├─ Config/
     │  └─ config.php                    # 模块配置
     └─ module.json                      # 模块元数据
```

模块服务提供者的标准写法：

```php
namespace Modules\MyModule\Providers;

use App\Services\Module\ModuleFacade;
use Illuminate\Support\ServiceProvider;

class MyModuleServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // 注册绑定到容器
    }

    public function boot(): void
    {
        // 注册前端资源
        ModuleFacade::script('my-module', __DIR__ . '/../Resources/scripts/my-module.js');
        ModuleFacade::style('my-module', __DIR__ . '/../Resources/css/my-module.css');

        // 扩展主菜单
        if (app()->bound('menu')) {
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
        }

        // 加载路由、视图、迁移等
        $this->loadRoutesFrom(__DIR__ . '/../Routes/api.php');
        $this->loadViewsFrom(__DIR__ . '/../Resources/views', 'mymodule');
        $this->loadMigrationsFrom(__DIR__ . '/../Database/Migrations');
        $this->loadTranslationsFrom(__DIR__ . '/../Resources/lang', 'mymodule');
    }
}
```

模块前端入口脚本的标准写法：

```javascript
// Modules/MyModule/Resources/scripts/my-module.js

import MyModulePage from './views/MyModulePage.vue'
import { defineStore } from 'pinia'

window.InvoiceShelf.booting((app, router) => {
  // 1. 添加路由
  router.addRoute('admin', {
    path: 'my-module',
    name: 'my-module.index',
    component: MyModulePage,
    meta: { ability: 'view-my-module' }
  })

  // 2. 注册全局组件
  // app.component('my-widget', MyWidget)

  // 3. 添加 i18n 消息
  // window.InvoiceShelf.addMessages({...})
})
```

---

## 总结

InvoiceShelf 的模块系统采用 **"后端服务提供者 + 前端启动钩子"** 的双层扩展架构：

1. **后端层面**：基于 `nwidart/laravel-modules`，模块通过服务提供者在 Laravel 生命周期内注册路由、迁移、视图、菜单和前端资源
2. **前端层面**：通过 `window.InvoiceShelf.booting()` 钩子，模块脚本可以在主应用启动前注入路由、组件和状态管理
3. **菜单系统**：使用 `lavary/laravel-menu` 作为菜单抽象层，模块可通过服务提供者扩展菜单，Bootstrap API 统一输出给前端
4. **资源加载**：自研 `Module` 静态注册器 + 专用控制器，实现模块前端资源的动态注入和按需加载

这种设计既保持了 Laravel 生态的惯用模式（服务提供者、门面、事件），又通过前端启动钩子实现了 Vue 应用的动态扩展，是一套相对完整且解耦的模块化方案。
