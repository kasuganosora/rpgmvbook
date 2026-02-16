# 管理系统

## 概述

管理系统是 RPG Maker MV 的核心控制层，负责数据加载、场景切换、资源管理、战斗流程等全局功能。

## 管理器概览

| 管理器 | 文件位置 | 职责 |
|--------|---------|------|
| DataManager | rpg_managers.js | 数据库加载、存档管理 |
| SceneManager | rpg_managers.js | 场景生命周期、游戏循环 |
| BattleManager | rpg_managers.js | 战斗流程控制 |
| ImageManager | rpg_managers.js | 图像资源加载与缓存 |
| AudioManager | rpg_managers.js | 音频播放控制 |
| ConfigManager | rpg_managers.js | 游戏设置管理 |
| StorageManager | rpg_managers.js | 本地/云存储管理 |
| PluginManager | rpg_managers.js | 插件加载与配置 |

## 全局变量约定

### 数据库变量 ($data*)

```javascript
$dataSystem      // System.json
$dataActors      // Actors.json
$dataClasses     // Classes.json
$dataSkills      // Skills.json
$dataItems       // Items.json
$dataWeapons     // Weapons.json
$dataArmors      // Armors.json
$dataEnemies     // Enemies.json
$dataTroops      // Troops.json
$dataStates      // States.json
$dataAnimations  // Animations.json
$dataTilesets    // Tilesets.json
$dataCommonEvents// CommonEvents.json
$dataMapInfos    // MapInfos.json
$dataMap         // 当前地图
```

### 游戏对象变量 ($game*)

```javascript
$gameTemp        // 临时数据
$gameSystem      // 系统状态
$gameScreen      // 屏幕效果
$gameTimer       // 计时器
$gameMessage     // 消息框
$gameSwitches    // 开关
$gameVariables   // 变量
$gameSelfSwitches// 独立开关
$gameActors      // 角色集合
$gameParty       // 队伍
$gameTroop       // 敌群
$gameMap         // 地图
$gamePlayer      // 玩家
```

## 加载顺序

```
1. DataManager.loadDatabase()
   └── 加载所有 JSON 文件到 $data*

2. DataManager.selectSavefileForNewGame()
   └── 初始化新游戏

3. DataManager.setupNewGame()
   └── 创建所有 $game* 对象

4. SceneManager.goto(Scene_Map)
   └── 进入地图场景
```

## 章节导航

- [DataManager - 数据管理](data-manager.md)
- [SceneManager - 场景管理](scene-manager.md)
- [BattleManager - 战斗管理](battle-manager.md)
- [ImageManager - 图像管理](image-manager.md)
- [AudioManager - 音频管理](audio-manager.md)
- [ConfigManager - 配置管理](config-manager.md)
- [StorageManager - 存储管理](storage-manager.md)
- [PluginManager - 插件管理](plugin-manager.md)
