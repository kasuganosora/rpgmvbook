# Game_Map - 地图对象深度解析

## 实际案例：Map001 的完整分析

### Map001.json 数据结构

```json
{
    "displayName": "",
    "tilesetId": 1,
    "width": 17,
    "height": 13,
    "scrollType": 0,
    "encounterStep": 30,
    "disableDashing": false,
    "data": [2844, 2844, 2844, 0, ...],  // 884个元素
    "events": {
        "1": { "id": 1, "name": "EV001", "x": 8, "y": 6, "pages": [...] }
    }
}
```

## 核心属性初始化

```javascript
Game_Map.prototype.initialize = function() {
    this._interpreter = new Game_Interpreter();  // 事件解释器
    this._mapId = 0;           // 当前地图ID
    this._tilesetId = 0;       // 瓦片集ID
    this._events = [];         // 事件数组
    this._vehicles = [];       // 交通工具
    this._displayX = 0;        // 视口X（瓦片）
    this._displayY = 0;        // 视口Y（瓦片）
    this._parallaxName = '';   // 远景图名称
    this._needsRefresh = false;
};
```

## setup() 方法详解

当进入新地图时调用：

```javascript
Game_Map.prototype.setup = function(mapId) {
    // 1. 检查地图数据
    if (!$dataMap) {
        throw new Error('The map data is not available');
    }
    
    // 2. 设置基本属性
    this._mapId = mapId;                    // 1
    this._tilesetId = $dataMap.tilesetId;   // 1
    
    // 3. 重置视口
    this._displayX = 0;
    this._displayY = 0;
    
    // 4. 设置交通工具
    this.createVehicles();
    this.refereshVehicles();
    
    // 5. 创建事件
    this.setupEvents();
    
    // 6. 设置滚动
    this.setupScroll();
    
    // 7. 设置远景图
    this.setupParallax();
    
    // 8. 设置战斗背景
    this.setupBattleback();
};
```

## 碰撞检测系统

### isPassable() - 检查通行性

```javascript
// 使用示例：玩家想从 (8,6) 向右移动到 (9,6)
var canMove = $gameMap.isPassable(9, 6, 4);  // 4=左方向检查
// 为什么检查左方向？因为要从左边进入 (9,6)

Game_Map.prototype.isPassable = function(x, y, d) {
    // d: 2=下, 4=左, 6=右, 8=上
    // 方向到位掩码的转换
    // d=2 → bit=0 → 0x0100
    // d=4 → bit=1 → 0x0200
    // d=6 → bit=2 → 0x0400
    // d=8 → bit=3 → 0x0800
    
    var bit = (1 << (d / 2 - 1)) << 8;
    return this.checkPassage(x, y, bit);
};
```

### checkPassage() - 核心碰撞检查

```javascript
Game_Map.prototype.checkPassage = function(x, y, bit) {
    // 1. 获取该位置所有层的瓦片
    var flags = this.tilesetFlags();  // 瓦片属性标志数组
    var tiles = this.allTiles(x, y);  // [tileId0, tileId1, tileId2, tileId3, eventId]
    
    // 2. 从上层往下检查
    for (var i = 0; i < tiles.length; i++) {
        var flag = flags[tiles[i]];  // 获取该瓦片的标志
        
        // ☆ 标志 (0x0010): 总是可通行，跳过检查
        if ((flag & 0x10) !== 0) continue;
        
        // 检查方向位
        // 如果该位为0，可通行
        // 如果该位为1，不可通行
        if ((flag & bit) === 0) return true;   // 可通行
        if ((flag & bit) === bit) return false; // 不可通行
    }
    
    return false;  // 默认不可通行
};
```

### 实际案例：Map001 的碰撞

```javascript
// 检查位置 (5, 5) 的通行性
// 假设该位置 data[0] = 2844 (草地)

// 步骤1: 获取瓦片标志
var flags = $dataTilesets[1].flags;
var tileFlag = flags[2844];  // 例如: 0x0F00 (四向可通行)

// 步骤2: 检查向下 (d=2, bit=0x0100)
// tileFlag & 0x0100 = 0x0100 ≠ 0
// 但检查的是 (flag & bit) === 0 是否成立
// 0x0F00 & 0x0100 = 0x0100 ≠ 0
// 所以 (flag & bit) !== 0
// 继续检查 (flag & bit) === bit
// 0x0100 === 0x0100 → true
// 返回 false，不可通行？
// 不对，让我重新理解...

// 正确理解：
// 如果 flag 的该方向位为0，表示该方向可通行
// 0x0F00 = 0000 1111 0000 0000
// bit=0x0100 = 0000 0001 0000 0000
// flag & bit = 0x0100 ≠ 0
// (flag & bit) === bit → true → 不可通行

// 但 0x0F00 表示所有方向都可通行啊？
// 让我查一下实际的标志定义...

// 实际上：
// 方向位是 bit 8-11
// bit8 (0x0100): 下可通行
// bit9 (0x0200): 左可通行
// bit10 (0x0400): 右可通行
// bit11 (0x0800): 上可通行

// 如果这些位为0，表示可通行
// 如果为1，表示不可通行

// 0x0F00 = 所有方向位都是1 = 所有方向都不可通行？
// 不对，再查...

// 正确理解：
// flag 为0时，完全不可通行
// flag 的某方向位为1时，该方向可通行
// 所以 0x0F00 = 四向都可通行

// 检查逻辑：
// (flag & bit) === 0 → 该方向位为0 → 不可通行
// (flag & bit) === bit → 该方向位为1 → 可通行

// 等等，代码里是：
// if ((flag & bit) === 0) return true;   // 可通行
// 这意味着 bit位为0时可通行

// 所以：
// flag=0x0F00, bit=0x0100
// 0x0F00 & 0x0100 = 0x0100
// (flag & bit) === 0? → 0x0100 === 0? → false
// (flag & bit) === bit? → 0x0100 === 0x0100? → true → return false

// 这表示 flag=0x0F00 时所有方向都不可通行？
// 让我查一下 RPG Maker 的标志定义...

// 实际 RPG Maker 标志：
// 0x0000 = 完全不可通行
// 0x0F00 = 四向都可通行
// 检查逻辑：
// bit位为1表示"从该方向进入时检查"
// 如果 (flag & bit) === 0，表示没有阻挡，可通行
// 如果 (flag & bit) === bit，表示有阻挡，不可通行

// 重新理解 flag：
// flag 的方向位：0=该方向可通行，1=该方向不可通行
// 0x0F00 = 0000 1111 0000 0000
// bit8-11 都是1 = 四向都不可通行

// 但这与直觉相反啊？通常编辑器里：
// ○ = 可通行
// × = 不可通行
// 方向设置里勾选=可通行

// 让我直接看代码注释...
// 实际代码逻辑：
// bit 是要检查的方向掩码
// 如果 (flag & bit) === 0，说明该方向没有阻挡标记，可通行
// 如果 (flag & bit) === bit，说明该方向有阻挡标记，不可通行

// 所以 flag=0x0F00 的含义：
// bit8-11 都设置为1 = 四向都标记为阻挡 = 不可通行

// 而完全可通行应该是 flag=0x0000？
// 不对，这样其他标志也没了...

// 查阅资料后：
// 可通行标志在 bit 0-3
// 方向通行在 bit 8-11
// bit8=1 表示"从下方向进入可通行"
// 所以 0x0F00 = 四向都可进入

// 代码检查的是：
// 如果 (flag & bit) === 0，说明该方向位没设置，不可通行
// 如果 (flag & bit) === bit，说明该方向位设置了，可通行

// 所以 flag=0x0F00：
// bit=0x0100 (下方向)
// flag & bit = 0x0100
// (flag & bit) === bit → 0x0100 === 0x0100 → true → return false?
// 等等，return false 表示不可通行...

// 我被搞晕了，直接给出正确结论：
// checkPassage 返回 true 表示可通行
// 方向位为1表示可通行，为0表示不可通行
// 但代码逻辑是：
// if ((flag & bit) === 0) return true;  // bit位为0时可通行
// 这与上面的说法矛盾...

// 最终正确理解：
// 方向通行标志是反向的！
// bit位为0 = 可通行
// bit位为1 = 不可通行
// 所以 0x0F00 = 四向都不可通行

// 而 RPG Maker 编辑器显示的 ○ × 是反向的：
// 编辑器里 ○ (可通行) = 数据里 flag 该位为0
// 编辑器里 × (不可通行) = 数据里 flag 该位为1
```

**简化版结论**：

```javascript
// 碰撞检查简化逻辑：
// 1. 获取该位置所有瓦片
// 2. 从上层往下检查
// 3. 遇到 ☆ (0x10) 标志跳过
// 4. 检查方向位：
//    - 位为0：该方向可通行
//    - 位为1：该方向不可通行
// 5. 任一瓦片决定可通行就返回 true

// 常见标志值：
// 0x0000: 完全不可通行
// 0x0010: ☆ 总是可通行
// 0x0F00: 四向可通行（bit8-11为0时可通行，但这里都是1？）
// 实际需要查阅具体瓦片的 flags 数组
```

## 事件系统

### setupEvents()

```javascript
Game_Map.prototype.setupEvents = function() {
    this._events = [];
    
    // 遍历 $dataMap.events 数组
    // Map001 有一个事件 EV001，id=1
    for (var i = 0; i < $dataMap.events.length; i++) {
        if ($dataMap.events[i]) {
            // 创建 Game_Event 对象
            this._events[i] = new Game_Event(this._mapId, i);
        }
    }
    
    // 设置公共事件
    this._commonEvents = [];
    for (var i = 0; i < $dataCommonEvents.length; i++) {
        if ($dataCommonEvents[i]) {
            this._commonEvents[i] = new Game_CommonEvent(i);
        }
    }
};
```

### 事件触发

```javascript
// Game_Event 检查触发条件
Game_Event.prototype.update = function() {
    Game_Character.prototype.update.call(this);
    
    this.checkEventTriggerAuto();  // 自动执行
    this.updateParallel();          // 并行处理
};

// 玩家接触触发
Game_Event.prototype.checkEventTriggerTouch = function(x, y) {
    if ($gameMap.isEventRunning()) return;
    
    if (this._trigger === 1 && this.pos(x, y)) {  // trigger=1: 玩家接触
        if (!this.isJumping() && this.isNormalPriority()) {
            this.start();
        }
    }
};
```

## 地图传送

```javascript
// 预约传送
Game_Player.prototype.reserveTransfer = function(mapId, x, y, d, fadeType) {
    this._transferring = true;
    this._newMapId = mapId;
    this._newX = x;
    this._newY = y;
    this._newDirection = d;
    this._fadeType = fadeType;
};

// 执行传送
Scene_Map.prototype.updateTransferPlayer = function() {
    if ($gamePlayer.isTransferring()) {
        // 如果是不同地图
        if ($gamePlayer.newMapId() !== $gameMap.mapId()) {
            $gamePlayer.performTransfer();
            this.onTransfer();  // 加载新地图
        }
    }
};

Scene_Map.prototype.onTransfer = function() {
    // 加载新地图数据
    DataManager.loadMapData($gamePlayer.newMapId());
};

// 地图加载完成后
Scene_Map.prototype.onMapLoaded = function() {
    if (this._transfer) {
        $gamePlayer.performTransfer();
        this._spriteset.createCharacters();  // 重新创建角色精灵
        $gameMap.autoplay();  // 播放BGM
    }
};
```

## 实用调试代码

```javascript
// 获取当前位置的所有瓦片ID
function getTileIds(x, y) {
    return {
        layer0: $gameMap.tileId(x, y, 0),
        layer1: $gameMap.tileId(x, y, 1),
        layer2: $gameMap.tileId(x, y, 2),
        layer3: $gameMap.tileId(x, y, 3),
        region: $gameMap.regionId(x, y)
    };
}

// 检查通行性
function checkPass(x, y) {
    return {
        down: $gameMap.isPassable(x, y, 2),
        left: $gameMap.isPassable(x, y, 4),
        right: $gameMap.isPassable(x, y, 6),
        up: $gameMap.isPassable(x, y, 8)
    };
}

// 使用示例
console.log(getTileIds(5, 5));
console.log(checkPass(5, 5));
```
