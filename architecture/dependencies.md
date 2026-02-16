# 模块依赖关系

## 文件加载顺序

`index.html` 中定义的脚本加载顺序：

```html
<script type="text/javascript" src="js/libs/pi.js"></script>
<script type="text/javascript" src="js/libs/pixi.js"></script>
<script type="text/javascript" src="js/libs/pixi-tilemap.js"></script>
<script type="text/javascript" src="js/libs/pixi-picture.js"></script>
<script type="text/javascript" src="js/fpsmeter.js"></script>
<script type="text/javascript" src="js/lz-string.js"></script>
<script type="text/javascript" src="js/iphone-inline-video.browser.js"></script>
<script type="text/javascript" src="js/rpg_core.js"></script>
<script type="text/javascript" src="js/rpg_managers.js"></script>
<script type="text/javascript" src="js/rpg_objects.js"></script>
<script type="text/javascript" src="js/rpg_scenes.js"></script>
<script type="text/javascript" src="js/rpg_sprites.js"></script>
<script type="text/javascript" src="js/rpg_windows.js"></script>
<script type="text/javascript" src="js/plugins.js"></script>
<script type="text/javascript" src="js/main.js"></script>
```

## 依赖图

```
                    ┌─────────────┐
                    │   PIXI.js   │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
    │pixi-tilemap │ │pixi-picture │ │  lz-string  │
    └──────┬──────┘ └──────┬──────┘ └─────────────┘
           │               │
    ┌──────┴───────────────┴──────┐
    │        rpg_core.js          │
    │  (Graphics, Bitmap, Tilemap)│
    └──────────────┬───────────────┘
                   │
    ┌──────────────┴───────────────┐
    │       rpg_managers.js        │
    │(DataManager, SceneManager...)│
    └──────────────┬───────────────┘
                   │
    ┌──────────────┴───────────────┐
    │       rpg_objects.js         │
    │(Game_Map, Game_Player...)    │
    └───────┬──────────────┬───────┘
            │              │
    ┌───────┴───────┐ ┌────┴────────────┐
    │ rpg_scenes.js │ │  rpg_sprites.js │
    │(Scene_Map...) │ │(Sprite_Char...) │
    └───────┬───────┘ └────────┬────────┘
            │                  │
    ┌───────┴──────────────────┴───────┐
    │        rpg_windows.js            │
    │   (Window_Base, Window_Menu...)  │
    └────────────────┬─────────────────┘
                     │
    ┌────────────────┴─────────────────┐
    │         plugins/*.js             │
    │    (扩展所有上述模块)              │
    └──────────────────────────────────┘
```

## 全局变量

### 数据库变量 ($data*)

只读的静态数据，从JSON加载：

```javascript
// 管理器定义位置: rpg_managers.js
var $dataActors       = null;  // 角色数据
var $dataClasses      = null;  // 职业数据
var $dataSkills       = null;  // 技能数据
var $dataItems        = null;  // 道具数据
var $dataWeapons      = null;  // 武器数据
var $dataArmors       = null;  // 防具数据
var $dataEnemies      = null;  // 敌人数据
var $dataTroops       = null;  // 敌群数据
var $dataStates       = null;  // 状态数据
var $dataAnimations   = null;  // 动画数据
var $dataTilesets     = null;  // 瓦片集
var $dataCommonEvents = null;  // 公共事件
var $dataSystem       = null;  // 系统设置
var $dataMapInfos     = null;  // 地图列表
var $dataMap          = null;  // 当前地图
```

### 游戏对象变量 ($game*)

运行时状态，保存到存档：

```javascript
// 创建位置: DataManager.setupNewGame()
var $gameTemp      = null;  // 临时数据
var $gameSystem    = null;  // 系统状态
var $gameScreen    = null;  // 屏幕效果
var $gameTimer     = null;  // 计时器
var $gameMessage   = null;  // 消息框
var $gameSwitches  = null;  // 开关
var $gameVariables = null;  // 变量
var $gameSelfSwitches = null;  // 独立开关
var $gameActors    = null;  // 角色集合
var $gameParty     = null;  // 队伍
var $gameTroop     = null;  // 敌群
var $gameMap       = null;  // 地图
var $gamePlayer    = null;  // 玩家
```

## 模块间通信

### 订阅-通知模式

```javascript
// Game_Screen 保存效果状态
$gameScreen.startFadeOut(30);

// Scene_Base 读取并应用
Scene_Base.prototype.updateFade = function() {
    if ($gameScreen.isFading()) {
        // 应用淡入淡出效果
    }
};
```

### 单例访问

```javascript
// 不需要依赖注入，直接访问全局单例
Game_Event.prototype.updateSelfMovement = function() {
    if ($gameMap.isEventRunning()) return;
    // ...
};

Sprite_Character.prototype.updateVisibility = function() {
    if ($gameSystem.isBattleCamera()) {
        // ...
    }
};
```

## 插件扩展机制

### Alias 方法

插件通过保存原方法引用来扩展：

```javascript
// 原始方法
Game_Player.prototype.update = function(sceneActive) {
    // 原始实现
};

// 插件扩展
var _Game_Player_update = Game_Player.prototype.update;
Game_Player.prototype.update = function(sceneActive) {
    _Game_Player_update.call(this, sceneActive);  // 调用原方法
    // 扩展功能
    this.updateCustomMovement();
};
```

### 参数获取

```javascript
// plugins.js 定义插件参数
var $plugins = [
    {
        name: 'Community_Basic',
        status: true,
        description: 'Basic plugin...',
        parameters: {
            "cacheLimit": "10",
            "screenWidth": "816"
        }
    }
];

// 插件获取参数
var parameters = PluginManager.parameters('Community_Basic');
var cacheLimit = Number(parameters['cacheLimit']);
```

## 循环依赖处理

RPG Maker MV 通过**分层加载**避免循环依赖：

1. **下层不依赖上层** - Core 不依赖 Objects
2. **通过回调延迟绑定** - 场景创建时再绑定对象
3. **全局变量解耦** - 使用 `$gameXXX` 而非直接引用

```javascript
// Sprite 观察 $gamePlayer，而非持有引用
Sprite_Player.prototype.setCharacter = function(character) {
    this._character = character;  // 通常是 $gamePlayer
};
```
