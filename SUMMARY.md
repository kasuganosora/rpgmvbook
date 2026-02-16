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

## 窗口系统
- [Window_Base - 窗口基类](windows/window-base.md)
- [Window_Selectable - 可选择窗口](windows/window-selectable.md)

## 数据格式
- [地图数据格式](data/map-format.md)
- [数据库文件](data/database-files.md)
- [存档数据结构](data/save-format.md)

## 插件系统
- [插件开发指南](plugins/development.md)
