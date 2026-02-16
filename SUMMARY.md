# 目录

- [概述](README.md)

## 架构总览
- [核心层次结构](architecture/layers.md)
- [模块依赖关系](architecture/dependencies.md)

## 管理系统
- [DataManager - 数据管理器](managers/data-manager.md)
- [SceneManager - 场景管理器](managers/scene-manager.md)
- [BattleManager - 战斗管理器](managers/battle-manager.md)
- [ImageManager - 图像管理器](managers/image-manager.md)
- [AudioManager - 音频管理器](managers/audio-manager.md)
- [StorageManager - 存储管理器](managers/storage-manager.md)
- [ConfigManager - 配置管理器](managers/config-manager.md)

## 瓦片与地图系统（重点）
- [Tilemap类详解](tilemap/tilemap-class.md)
- [自动瓦片系统](tilemap/autotile.md)
- [图层与深度](tilemap/layers-depth.md)
- [瓦片集数据结构](tilemap/tileset-data.md)
- [地图渲染流程](tilemap/rendering-pipeline.md)

## 游戏对象系统
- [Game_Map - 地图对象](objects/game-map.md)
- [Game_Character - 角色基类](objects/game-character.md)
- [Game_Player - 玩家对象](objects/game-player.md)
- [Game_Event - 事件对象](objects/game-event.md)

## 场景系统
- [Scene_Map - 地图场景](scenes/scene-map.md)
- [Scene_Battle - 战斗场景](scenes/scene-battle.md)

## 精灵系统
- [Sprite_Character - 角色精灵](sprites/sprite-character.md)
- [Spriteset_Map - 地图精灵集](sprites/spriteset-map.md)

## UI 系统（重点）
- [概述](ui/README.md)
- [窗口渲染原理](ui/ui-window.md)
- [Bitmap 绘图](ui/ui-bitmap.md)
- [输入系统](ui/ui-input.md)
- [窗口交互](ui/ui-interaction.md)
- [开发案例](ui/ui-examples.md)
- [从零实现控件教程](ui/ui-tutorial.md)

## 窗口系统
- [Window_Base - 窗口基类](windows/window-base.md)
- [Window_Selectable - 可选择窗口](windows/window-selectable.md)

## 数据格式
- [地图数据格式](data/map-format.md)
- [数据库文件](data/database-files.md)
- [存档数据结构](data/save-format.md)

## 插件系统
- [插件开发指南](plugins/development.md)

## 从零开发 RPG 引擎
- [概述与学习路线](engine/README.md)
- [01-什么是游戏引擎](engine/01-what-is-engine.md)
- [02-游戏主循环](engine/02-game-loop.md)
- [03-画布与坐标系统](engine/03-canvas-coordinates.md)
- [04-瓦片地图系统](engine/04-tilemap.md)
- [瓦片地图类型详解](engine/tilemap-types.md)
- [05-精灵与动画](engine/05-sprite-animation.md)
- [06-游戏对象系统](engine/06-game-objects.md)
- [07-碰撞检测](engine/07-collision.md)
- [08-地图与场景](engine/08-map-scene.md)
- [09-输入处理](engine/09-input.md)
- [10-事件系统](engine/10-event-system.md)
- [11-用户界面](engine/11-ui-system.md)
- [12-角色数据系统](engine/12-character-data.md)
- [13-物品与背包](engine/13-inventory.md)
- [14-战斗系统](engine/14-battle.md)
- [15-存档系统](engine/15-save-load.md)
- [瓦片地图类型详解](engine/tilemap-types.md)

### 扩展为 MMO 多人在线游戏
- [概述与学习路线](engine/mmo/README.md)
- [网络基础](engine/mmo/01-network-basics.md)
- [服务器架构](engine/mmo/02-server-architecture.md)
- [数据同步](engine/mmo/03-data-sync.md)
- [数据库设计](engine/mmo/04-database.md)
- [安全性](engine/mmo/05-security.md)
- [实战案例](engine/mmo/06-practice.md)
