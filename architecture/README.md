# 架构总览

## 五层架构模型

RPG Maker MV 采用经典的分层架构设计：

```
┌─────────────────────────────────────────────────────────────┐
│                    Plugin Layer (插件层)                      │
│   Community_Basic.js, CustomLogo.js, AltMenuScreen.js...    │
├─────────────────────────────────────────────────────────────┤
│                   Window Layer (窗口层)                       │
│   Window_Base, Window_Selectable, Window_MenuCommand...     │
├─────────────────────────────────────────────────────────────┤
│                   Sprite Layer (精灵层)                       │
│   Sprite_Character, Spriteset_Map, Sprite_Battler...        │
├─────────────────────────────────────────────────────────────┤
│                   Scene Layer (场景层)                        │
│   Scene_Boot → Scene_Title → Scene_Map ↔ Scene_Battle       │
├─────────────────────────────────────────────────────────────┤
│                   Object Layer (对象层)                       │
│   Game_Map, Game_Player, Game_Event, Game_Actor...          │
├─────────────────────────────────────────────────────────────┤
│                  Manager Layer (管理层)                       │
│   DataManager, SceneManager, ImageManager, AudioManager...   │
├─────────────────────────────────────────────────────────────┤
│                    Core Layer (核心层)                        │
│   Bitmap, Tilemap, Graphics, Input, TouchInput, PIXI.js     │
└─────────────────────────────────────────────────────────────┘
```

## 核心层 (Core Layer)

位置：`rpg_core.js`

### 渲染相关

| 类名 | 职责 |
|------|------|
| `Graphics` | 画布创建、渲染循环、画面缩放 |
| `Bitmap` | 位图操作、绘制文本、色彩处理 |
| `Tilemap` | 瓦片地图渲染引擎 |
| `ShaderTilemap` | WebGL加速的瓦片渲染 |
| `Sprite` | PIXI.Sprite 封装，显示对象 |
| `ScreenSprite` | 全屏特效精灵 |
| `Window` | 窗口渲染基类 |

### 输入处理

| 类名 | 职责 |
|------|------|
| `Input` | 键盘输入处理 (方向键、确认、取消等) |
| `TouchInput` | 触摸/鼠标输入处理 |

### 资源与加密

| 类名 | 职责 |
|------|------|
| `Decrypter` | 资源解密处理 |
| `ResourceHandler` | 资源加载失败重试 |

## 管理层 (Manager Layer)

位置：`rpg_managers.js`

管理器采用单例模式，通过全局变量访问：

```javascript
$dataActors    // 数据库数据
$dataSkills    
$gameActors    // 运行时游戏对象
$gameMap       
$gamePlayer    
```

### 核心管理器

| 管理器 | 职责 |
|--------|------|
| `DataManager` | 数据库加载、存档/读档、游戏对象创建 |
| `SceneManager` | 场景栈管理、游戏循环、帧更新 |
| `BattleManager` | 战斗流程控制、回合管理、胜负判定 |
| `ImageManager` | 图像资源加载、缓存管理 |
| `AudioManager` | 音频播放控制 (BGM/BGS/ME/SE) |
| `ConfigManager` | 游戏设置 (音量、按键映射) |
| `StorageManager` | 本地存储、云存档 |
| `PluginManager` | 插件加载与参数解析 |

## 对象层 (Object Layer)

位置：`rpg_objects.js`

游戏对象是纯数据模型，不包含渲染逻辑：

### 核心游戏对象

```javascript
// 全局状态
$gameTemp       // Game_Temp - 临时数据
$gameSystem     // Game_System - 系统设置
$gameSwitches   // Game_Switches - 开关
$gameVariables  // Game_Variables - 变量
$gameScreen     // Game_Screen - 屏幕效果

// 地图相关
$gameMap        // Game_Map - 地图数据
$gamePlayer     // Game_Player - 玩家
$gameMap.events() // Game_Event[] - 事件

// 战斗相关
$gameActors     // Game_Actors - 角色管理
$gameParty      // Game_Party - 队伍
$gameTroop      // Game_Troop - 敌群
```

### 继承关系

```
Game_BaseAction
    └── Game_Action (战斗动作)

Game_BattlerBase
    └── Game_Battler
            ├── Game_Actor (玩家角色)
            └── Game_Enemy (敌人)

Game_CharacterBase
    └── Game_Character
            ├── Game_Player (玩家)
            ├── Game_Event (事件)
            ├── Game_Vehicle (交通工具)
            └── Game_Follower (跟随者)
```

## 场景层 (Scene Layer)

位置：`rpg_scenes.js`

场景实现状态机模式，每个场景管理独立的子组件：

```
Scene_Boot (启动)
    │
    ▼
Scene_Title (标题画面)
    │
    ├─ 新游戏 ──▶ Scene_Map
    │                 │
    │                 ├─ 菜单 ──▶ Scene_Menu
    │                 │                │
    │                 │                ├─ Scene_Item
    │                 │                ├─ Scene_Skill
    │                 │                ├─ Scene_Equip
    │                 │                ├─ Scene_Status
    │                 │                └─ Scene_Options
    │                 │
    │                 └─ 遭遇 ──▶ Scene_Battle
    │
    └─ 继续 ──▶ Scene_Load (→ Scene_Map)
```

### 场景生命周期

```javascript
Scene_Base.prototype.initialize();   // 初始化
Scene_Base.prototype.create();       // 创建子组件
Scene_Base.prototype.start();        // 开始动画
Scene_Base.prototype.update();       // 每帧更新
Scene_Base.prototype.stop();         // 停止动画
Scene_Base.prototype.terminate();    // 销毁清理
```

## 精灵层 (Sprite Layer)

位置：`rpg_sprites.js`

精灵负责将游戏对象渲染到屏幕：

### 核心精灵类

| 精灵类 | 渲染目标 |
|--------|----------|
| `Sprite_Character` | 角色、事件、交通工具 |
| `Sprite_Battler` | 战斗角色基类 |
| `Sprite_Actor` | 玩家角色 (SV战斗) |
| `Sprite_Enemy` | 敌人 |
| `Sprite_Picture` | 图片 |
| `Sprite_StateIcon` | 状态图标 |
| `Sprite_Weapon` | 武器动画 |

### 精灵集 (Spriteset)

精灵集管理一组相关精灵：

```
Spriteset_Map
├── _tilemap (Tilemap)           // 瓦片地图
├── _parallax (TilingSprite)     // 远景图
├── _characterSprites[]          // 角色精灵数组
├── _weather (Weather)           // 天气效果
├── _pictureContainer            // 图片容器
└── _timerSprite                 // 计时器

Spriteset_Battle
├── _battleField                 // 战斗区域
├── _back1Sprite, _back2Sprite   // 战斗背景
├── _enemySprites[]              // 敌人精灵
├── _actorSprites[]              // 角色精灵
└── _effectContainer             // 特效容器
```

## 数据流

### 游戏循环

```
SceneManager.update()
    │
    ├── Graphics.update()           // 渲染帧
    ├── Input.update()              // 处理输入
    ├── TouchInput.update()         // 处理触摸
    │
    └── Scene.update()
            │
            ├── Scene.updateChildren()
            │       ├── Spriteset.update()
            │       └── WindowLayer.update()
            │
            └── Game_Object.update()
                    ├── Game_Map.update()
                    ├── Game_Player.update()
                    └── Game_Event[].update()
```

### 数据流向

```
JSON文件 (data/*.json)
    │
    ▼
DataManager.loadDatabase()
    │
    ▼
$dataXXX 全局变量 (只读数据库)
    │
    ▼
$gameXXX 游戏对象 (运行时状态)
    │
    ▼
Sprites/Windows 渲染显示
    │
    ▼
StorageManager 保存存档
```
