# 场景系统

## 概述

场景是 RPG Maker MV 中最大的组织单元，每个场景代表一个完整的游戏状态（标题、地图、菜单、战斗等）。

## 场景生命周期

```
initialize() ─── create() ─── start() ─── update() ─── stop() ─── terminate()
     │              │            │           │            │            │
   初始化         创建子组件    开始动画    每帧更新     停止动画      销毁清理
```

## 场景继承关系

```
PIXI.Container
    └── Scene_Base
            ├── Scene_Boot (启动)
            ├── Scene_Title (标题)
            ├── Scene_Map (地图)
            ├── Scene_MenuBase
            │       ├── Scene_Menu (主菜单)
            │       ├── Scene_Item (物品)
            │       ├── Scene_Skill (技能)
            │       ├── Scene_Equip (装备)
            │       ├── Scene_Status (状态)
            │       ├── Scene_Options (选项)
            │       ├── Scene_Save (存档)
            │       └── Scene_Load (读档)
            ├── Scene_Battle (战斗)
            ├── Scene_Shop (商店)
            ├── Scene_Name (命名)
            └── Scene_Gameover (游戏结束)
```

## 场景切换流程

```
Scene_Boot
    │
    ▼
Scene_Title
    │
    ├── 新游戏 ──▶ Scene_Map
    │                 │
    │                 ├── 菜单 ──▶ Scene_Menu
    │                 │                │
    │                 │                ├── 物品
    │                 │                ├── 技能
    │                 │                ├── 装备
    │                 │                └── 状态
    │                 │
    │                 ├── 遭遇 ──▶ Scene_Battle
    │                 │
    │                 └── 商店 ──▶ Scene_Shop
    │
    └── 继续 ──▶ Scene_Load ──▶ Scene_Map
```

## 章节导航

- [Scene_Base - 场景基类](scene-base.md)
- [Scene_Boot - 启动场景](scene-boot.md)
- [Scene_Title - 标题场景](scene-title.md)
- [Scene_Map - 地图场景](scene-map.md)
- [Scene_Menu - 菜单场景](scene-menu.md)
- [Scene_Battle - 战斗场景](scene-battle.md)
