# RPG Maker MV 项目技术文档

## 概述

本文档详细分析了 RPG Maker MV 游戏引擎的完整架构和实现原理，涵盖所有核心子系统。项目基于 JavaScript 开发，使用 PIXI.js 作为底层渲染引擎。

## 技术栈

| 组件 | 技术选型 |
|------|----------|
| 语言 | JavaScript (ES5) |
| 渲染引擎 | PIXI.js v4.x |
| 地图渲染 | pixi-tilemap |
| 图片处理 | pixi-picture |
| 压缩算法 | lz-string |
| 数据格式 | JSON |

## 核心文件结构

```
Project1/
├── index.html          # 入口HTML
├── js/
│   ├── main.js         # 主入口脚本
│   ├── plugins.js      # 插件配置
│   ├── rpg_core.js     # 核心类 (Bitmap, Tilemap, Graphics等)
│   ├── rpg_managers.js # 管理器类 (DataManager, SceneManager等)
│   ├── rpg_objects.js  # 游戏对象 (Game_Map, Game_Player等)
│   ├── rpg_scenes.js   # 场景类 (Scene_Map, Scene_Battle等)
│   ├── rpg_sprites.js  # 精灵类 (Sprite_Character, Spriteset_Map等)
│   ├── rpg_windows.js  # 窗口类 (Window_Base, Window_MenuCommand等)
│   ├── libs/           # 第三方库
│   └── plugins/        # 插件目录
└── data/               # JSON数据库
    ├── System.json     # 系统设置
    ├── Actors.json     # 角色数据
    ├── MapXXX.json     # 地图数据
    └── Tilesets.json   # 瓦片集数据
```

## 启动流程

```javascript
// 1. 加载插件配置
PluginManager.setup($plugins);

// 2. 页面加载完成后启动
window.onload = function() {
    SceneManager.run(Scene_Boot);
};

// 3. Scene_Boot 加载所有数据库文件
// 4. 进入 Scene_Title 标题画面
// 5. 开始新游戏/继续游戏 → Scene_Map
```

## 文档导航

- **[架构总览](architecture/README.md)** - 了解整体系统设计
- **[瓦片与地图系统](tilemap/README.md)** - 重点章节：深入理解地图渲染
- **[管理系统](managers/README.md)** - 数据、场景、资源管理器
- **[游戏对象系统](objects/README.md)** - 地图、玩家、事件等核心对象
- **[场景系统](scenes/README.md)** - 场景状态机与生命周期
- **[精灵系统](sprites/README.md)** - 渲染层实现细节
- **[窗口系统](windows/README.md)** - UI 框架
- **[数据格式](data/README.md)** - JSON 数据库结构
- **[插件系统](plugins/README.md)** - 扩展开发指南
