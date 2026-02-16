# DataManager - 数据管理器详解

## 概述

DataManager 是 RPG Maker MV 的数据加载和存档核心。它负责：
- 加载所有 JSON 数据库文件
- 创建游戏运行时对象 ($gameXxx)
- 处理存档/读档

## 一、数据加载流程详解

### 1.1 启动时的完整加载链

```
index.html 加载
    │
    ▼
main.js 执行
    │
    ├── PluginManager.setup($plugins)  // 加载插件配置
    │
    └── window.onload 触发
            │
            ▼
        SceneManager.run(Scene_Boot)
            │
            ▼
        Scene_Boot.create()
            │
            ▼
        DataManager.loadDatabase()  // 【关键入口】
```

### 1.2 loadDatabase 逐行解析

```javascript
DataManager.loadDatabase = function() {
    // 第1步：判断是否为测试模式（战斗测试或事件测试）
    // 在编辑器中点击"战斗测试"时，this.isBattleTest() 返回 true
    var test = this.isBattleTest() || this.isEventTest();
    
    // 第2步：测试模式使用 Test_ 前缀的文件
    // 正常模式加载 data/Actors.json
    // 测试模式加载 data/Test_Actors.json
    var prefix = test ? 'Test_' : '';
    
    // 第3步：遍历所有数据库文件定义
    for (var i = 0; i < this._databaseFiles.length; i++) {
        var name = this._databaseFiles[i].name;  // 如 '$dataActors'
        var src = this._databaseFiles[i].src;    // 如 'Actors.json'
        
        // 异步加载每个文件
        this.loadDataFile(name, prefix + src);
    }
    
    // 第4步：事件测试时额外加载测试地图
    if (this.isEventTest()) {
        this.loadDataFile('$testMap', 'Map001.json');
    }
};
```

**实际案例**：打开浏览器控制台，输入：
```javascript
// 查看 $dataActors 是否已加载
console.log($dataActors);
// 输出: [null, {id:1, name:"哈罗德", ...}, {id:2, name:"特蕾莎", ...}, ...]

// 注意：数组第0位是 null，因为数据库ID从1开始
```

### 1.3 loadDataFile 异步加载详解

```javascript
DataManager.loadDataFile = function(name, src) {
    // 创建 XMLHttpRequest 对象
    var xhr = new XMLHttpRequest();
    
    // 构建URL：'data/Actors.json'
    var url = 'data/' + src;
    
    // 第3个参数 true 表示异步请求
    xhr.open('GET', url, true);
    
    // 设置 MIME 类型，确保服务器返回正确的 Content-Type
    xhr.overrideMimeType('application/json');
    
    // 定义加载完成回调
    xhr.onload = function() {
        // HTTP 状态码 < 400 表示成功（200, 201, 204 等）
        if (xhr.status < 400) {
            // 【关键】将 JSON 字符串解析为对象
            // 结果存入全局变量 window['$dataActors'] = ...
            window[name] = JSON.parse(xhr.responseText);
            
            // 调用后处理函数（提取数组数据）
            DataManager.onLoad(window[name]);
        }
    };
    
    // 发送请求
    xhr.send();
};
```

**为什么用 `window[name]` 而不是 `eval`？**
- `window['$dataActors']` 安全地创建全局变量
- `eval` 会执行任意代码，存在安全风险
- 这样设计允许插件访问 `$dataActors` 等全局变量

### 1.4 onLoad 后处理 - 提取数组元素

```javascript
DataManager.onLoad = function(object) {
    var array;
    
    // 检测是否为数组
    if (Array === object.constructor) {
        array = object;
    } else if (Array === object.data.constructor) {
        // 地图数据是对象，但内部 data 是数组
        array = object.data;
    }
    
    if (array) {
        // 提取数组中的元数据到对象属性
        // 例如：array[1].name 提取到 object._names[1] = "哈罗德"
        this.extractMetadata(array);
    }
    
    // 标记对象已加载
    object._loaded = true;
};

DataManager.extractMetadata = function(array) {
    // 遍历数组（跳过 null 的第0位）
    for (var i = 1; i < array.length; i++) {
        var data = array[i];
        if (data && data.note !== undefined) {
            // 解析备注栏中的 <tag:value> 格式
            // 例如：<price:500> 解析为 data._meta.price = "500"
            var meta = {};
            var note = data.note;
            var regex = /<([^<>:]+)(:?)([^>]*)>/g;
            var match;
            while ((match = regex.exec(note)) !== null) {
                var key = match[1];
                var value = match[3] || true;  // 无值则默认 true
                meta[key] = value;
            }
            data._meta = meta;
        }
    }
};
```

**实际案例**：查看备注解析结果
```javascript
// 假设某物品备注栏写了：<custom_effect:poison> <rare>
console.log($dataItems[1]._meta);
// 输出: {custom_effect: "poison", rare: true}
```

---

## 二、地图数据加载

### 2.1 地图文件的命名规则

```
Map001.json → Map%1.json → id=1
Map015.json → Map%15.json → id=15
Map999.json → Map%999.json → id=999
```

### 2.2 loadMapData 详解

```javascript
DataManager.loadMapData = function(mapId) {
    if (mapId > 0) {
        // padZero(3) 将数字补零到3位
        // 1 → "001", 15 → "015"
        var filename = 'Map%1.json'.format(mapId.padZero(3));
        this.loadDataFile('$dataMap', filename);
    } else {
        // mapId=0 时创建空地图（用于战斗场景）
        this.makeEmptyMap();
    }
};

DataManager.makeEmptyMap = function() {
    $dataMap = {};
    $dataMap.data = [];
    $dataMap.events = [];
    $dataMap.width = 0;
    $dataMap.height = 0;
    $dataMap.scrollType = 0;
};
```

**实际案例**：查看 Map001 的数据结构
```javascript
// 在 Scene_Map 中查看当前地图数据
console.log($dataMap);
// 输出结构：
// {
//   data: [0, 0, 0, 2816, 2816, ...],  // 瓦片ID数组
//   events: [null, {id:1, name:"EV001", ...}],
//   width: 17,
//   height: 13,
//   scrollType: 0,
//   tilesetId: 1
// }
```

### 2.3 地图加载状态检测

```javascript
// DataManager 提供的检测方法
DataManager.isMapLoaded = function() {
    return !!$dataMap;  // $dataMap 非空即已加载
};

// Scene_Map 中的等待逻辑
Scene_Map.prototype.isReady = function() {
    // 检测地图是否加载完成
    if (!this._mapLoaded && DataManager.isMapLoaded()) {
        this.onMapLoaded();  // 触发地图加载完成事件
        this._mapLoaded = true;
    }
    // 还需要检测图片资源是否就绪
    return this._mapLoaded && Scene_Base.prototype.isReady.call(this);
};

Scene_Map.prototype.onMapLoaded = function() {
    // 1. 设置地图数据到 Game_Map
    $gameMap.setup($dataMap);
    // 2. 创建地图精灵
    this.createSpriteset();
    // 3. 创建窗口
    this.createAllWindows();
};
```

---

## 三、游戏对象创建

### 3.1 createGameObjects 详解

```javascript
DataManager.createGameObjects = function() {
    // 临时数据（不存档）
    $gameTemp = new Game_Temp();
    
    // 系统数据（存档）
    $gameSystem = new Game_System();
    
    // 画面效果（存档）
    $gameScreen = new Game_Screen();
    
    // 计时器（存档）
    $gameTimer = new Game_Timer();
    
    // 消息窗口（不存档）
    $gameMessage = new Game_Message();
    
    // 开关（存档）- 数组，索引是开关ID
    $gameSwitches = new Game_Switches();
    
    // 变量（存档）- 数组，索引是变量ID
    $gameVariables = new Game_Variables();
    
    // 独立开关（存档）- 嵌套对象 {mapId: {eventId: {switchId: true}}}
    $gameSelfSwitches = new Game_SelfSwitches();
    
    // 角色数据（存档）
    $gameActors = new Game_Actors();
    
    // 队伍（存档）
    $gameParty = new Game_Party();
    
    // 敌群（不存档）
    $gameTroop = new Game_Troop();
    
    // 地图状态（存档）
    $gameMap = new Game_Map();
    
    // 玩家（存档）
    $gamePlayer = new Game_Player();
};
```

### 3.2 对象初始化顺序的意义

```
Game_Temp          → 最先，因为其他对象可能需要临时数据
Game_System        → 系统设置影响后续初始化
Game_Screen        → 画面效果
Game_Timer         → 计时器
Game_Message       → 消息显示
Game_Switches      → 开关状态
Game_Variables     → 变量状态
Game_SelfSwitches  → 独立开关
Game_Actors        → 角色数据（依赖 $dataActors）
Game_Party         → 队伍（依赖 $gameActors）
Game_Troop         → 敌群
Game_Map           → 地图（依赖开关/变量）
Game_Player        → 玩家（依赖 $gameMap）
```

### 3.3 新游戏初始化流程

```javascript
DataManager.setupNewGame = function() {
    // 第1步：创建所有游戏对象
    this.createGameObjects();
    
    // 第2步：选择存档槽位
    this.selectSavefileForNewGame();
    
    // 第3步：设置队伍初始成员
    // $dataSystem.partyMembers = [1, 2, 3, 4]
    $gameParty.setupStartingMembers();
    
    // 第4步：设置玩家初始位置
    // reserveTransfer 会触发地图加载和玩家传送
    $gamePlayer.reserveTransfer(
        $dataSystem.startMapId,  // 起始地图ID（通常是1）
        $dataSystem.startX,      // 起始X坐标
        $dataSystem.startY       // 起始Y坐标
    );
    
    // 第5步：重置游戏时间
    Graphics.frameCount = 0;
};

// reserveTransfer 的实际效果
Game_Player.prototype.reserveTransfer = function(mapId, x, y, d, fadeType) {
    this._transferring = true;      // 标记正在传送
    this._newMapId = mapId;         // 目标地图ID
    this._newX = x;                 // 目标X
    this._newY = y;                 // 目标Y
    this._newDirection = d || 0;    // 传送后朝向
    this._fadeType = fadeType || 0; // 淡入淡出类型
};
```

---

## 四、存档系统

### 4.1 存档文件结构

```
存档位置（本地）：
├── localStorage
│   ├── "RPG Save1" → Save1 的 JSON 数据
│   ├── "RPG Save2" → Save2 的 JSON 数据
│   └── "RPG Global" → 全局信息（存档列表）
```

### 4.2 保存数据详解

```javascript
// makeSaveContents 构建存档内容
DataManager.makeSaveContents = function() {
    var contents = {};
    
    // 系统设置（BGM音量、选项等）
    contents.system = $gameSystem;
    
    // 画面效果（色调、闪烁、天气等）
    contents.screen = $gameScreen;
    
    // 计时器状态
    contents.timer = $gameTimer;
    
    // 所有开关
    contents.switches = $gameSwitches;
    
    // 所有变量
    contents.variables = $gameVariables;
    
    // 独立开关
    contents.selfSwitches = $gameSelfSwitches;
    
    // 所有角色
    contents.actors = $gameActors;
    
    // 队伍状态
    contents.party = $gameParty;
    
    // 当前地图状态
    contents.map = $gameMap;
    
    // 玩家状态
    contents.player = $gamePlayer;
    
    return contents;
};

// saveGameWithoutRescue 实际保存
DataManager.saveGameWithoutRescue = function(savefileId) {
    // 1. 创建快照（用于存档预览）
    var json = JSON.stringify(this.makeSaveContents());
    
    // 2. 存储到 localStorage 或文件
    StorageManager.save(savefileId, json);
    
    // 3. 更新全局信息
    var globalInfo = this.loadGlobalInfo() || [];
    globalInfo[savefileId] = {
        title: $dataSystem.gameTitle,
        characters: $gameParty.charactersForSavefile(),
        faces: $gameParty.facesForSavefile(),
        playtime: $gameSystem.playtimeText(),
        timestamp: Date.now()
    };
    StorageManager.save(0, JSON.stringify(globalInfo));
    
    return true;
};
```

### 4.3 加载数据详解

```javascript
DataManager.loadGameWithoutRescue = function(savefileId) {
    // 1. 从存储读取 JSON
    var json = StorageManager.load(savefileId);
    
    // 2. 解析为对象
    var contents = JSON.parse(json);
    
    // 3. 恢复游戏对象
    this.extractSaveContents(contents);
    
    // 4. 加载地图
    var mapId = $gamePlayer._mapId || $dataSystem.startMapId;
    this.loadMapData(mapId);
    
    return true;
};

DataManager.extractSaveContents = function(contents) {
    // 直接将存档对象赋值给全局变量
    $gameSystem = contents.system;
    $gameScreen = contents.screen;
    $gameTimer = contents.timer;
    $gameSwitches = contents.switches;
    $gameVariables = contents.variables;
    $gameSelfSwitches = contents.selfSwitches;
    $gameActors = contents.actors;
    $gameParty = contents.party;
    $gameMap = contents.map;
    $gamePlayer = contents.player;
};
```

### 4.4 存档检测

```javascript
// 检查是否有任何存档
DataManager.isAnySavefileExists = function() {
    var globalInfo = this.loadGlobalInfo();
    if (globalInfo) {
        // 从 1 开始（0 是全局信息）
        for (var i = 1; i < globalInfo.length; i++) {
            if (this.isThisGameFile(i)) {
                return true;
            }
        }
    }
    return false;
};

// 检查特定存档是否属于当前游戏
DataManager.isThisGameFile = function(savefileId) {
    var globalInfo = this.loadGlobalInfo();
    if (globalInfo && globalInfo[savefileId]) {
        // 验证游戏标题匹配
        return globalInfo[savefileId].title === $dataSystem.gameTitle;
    }
    return false;
};
```

---

## 五、实际调试案例

### 案例1：查看当前存档信息

```javascript
// 在浏览器控制台运行
var globalInfo = JSON.parse(localStorage.getItem('RPG Global'));
console.log(globalInfo);
// 输出：
// [null, 
//  {title: "Project1", playtime: "0:05:32", timestamp: 1708012345678, ...},
//  null,
//  {title: "Project1", playtime: "0:12:45", timestamp: 1708013456789, ...}
// ]
```

### 案例2：手动修改存档数据

```javascript
// 读取存档1
var save1 = JSON.parse(localStorage.getItem('RPG Save1'));

// 修改金币
save1.party._gold = 99999;

// 修改变量1的值
save1.variables._data[1] = 100;

// 保存回去
localStorage.setItem('RPG Save1', JSON.stringify(save1));
```

### 案例3：查看正在加载的文件

```javascript
// 在 Scene_Boot 中添加调试代码
var originalLoad = DataManager.loadDataFile;
DataManager.loadDataFile = function(name, src) {
    console.log('Loading:', src, '→', name);
    return originalLoad.call(this, name, src);
};
// 输出：
// Loading: Actors.json → $dataActors
// Loading: Classes.json → $dataClasses
// Loading: Skills.json → $dataSkills
// ...
```

### 案例4：监控加载完成状态

```javascript
// 检查所有数据库是否加载完成
function checkAllLoaded() {
    var files = [
        '$dataActors', '$dataClasses', '$dataSkills', '$dataItems',
        '$dataWeapons', '$dataArmors', '$dataEnemies', '$dataTroops',
        '$dataStates', '$dataAnimations', '$dataTilesets', 
        '$dataCommonEvents', '$dataSystem', '$dataMapInfos'
    ];
    
    files.forEach(function(name) {
        console.log(name + ':', window[name] ? '✓ loaded' : '✗ loading');
    });
}
checkAllLoaded();
```

---

## 六、数据库文件列表

| 全局变量 | 文件名 | 用途 |
|---------|--------|------|
| `$dataActors` | Actors.json | 角色数据库 |
| `$dataClasses` | Classes.json | 职业数据库 |
| `$dataSkills` | Skills.json | 技能数据库 |
| `$dataItems` | Items.json | 物品数据库 |
| `$dataWeapons` | Weapons.json | 武器数据库 |
| `$dataArmors` | Armors.json | 防具数据库 |
| `$dataEnemies` | Enemies.json | 敌人数据库 |
| `$dataTroops` | Troops.json | 敌群数据库 |
| `$dataStates` | States.json | 状态数据库 |
| `$dataAnimations` | Animations.json | 动画数据库 |
| `$dataTilesets` | Tilesets.json | 瓦片集数据库 |
| `$dataCommonEvents` | CommonEvents.json | 公共事件数据库 |
| `$dataSystem` | System.json | 系统设置 |
| `$dataMapInfos` | MapInfos.json | 地图列表 |
| `$dataMap` | MapXXX.json | 当前地图数据 |
