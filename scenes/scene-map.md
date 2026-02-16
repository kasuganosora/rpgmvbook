# Scene_Map - 地图场景

## 一、职责

Scene_Map 是游戏的核心场景，负责：
- 显示地图和角色
- 处理玩家移动
- 触发事件
- 处理遭遇战
- 管理消息窗口

## 二、生命周期方法

### 2.1 create - 创建显示对象

```javascript
Scene_Map.prototype.create = function() {
    Scene_Base.prototype.create.call(this);
    this._transfer = $gamePlayer.isTransferring();  // 是否正在传送
    this._mapLoaded = false;
    
    // 创建显示对象
    this.createDisplayObjects();
};

Scene_Map.prototype.createDisplayObjects = function() {
    // 1. 创建精灵集（地图+角色+效果）
    this.createSpriteset();
    
    // 2. 创建窗口层
    this.createWindowLayer();
    
    // 3. 创建所有窗口
    this.createAllWindows();
    
    // 4. 创建消息窗口（最重要）
    this.createMessageWindow();
};
```

### 2.2 start - 场景启动

```javascript
Scene_Map.prototype.start = function() {
    Scene_Base.prototype.start.call(this);
    
    // 触发地图自动事件
    $gameMap.refreshIfNeeded();
    
    // 传送时淡入
    if (this._transfer) {
        this.fadeInForTransfer();
    }
    
    // 开始消息处理
    this._mapNameWindow.open();
};
```

### 2.3 update - 每帧更新

```javascript
Scene_Map.prototype.update = function() {
    // 激活消息窗口
    this.updateMain();
    
    // 场景切换检测
    this.updateScene();
    
    Scene_Base.prototype.update.call(this);
};

Scene_Map.prototype.updateMain = function() {
    // 游戏暂停时不更新
    var active = this.isActive();
    $gameMap.update(active);      // 更新地图
    $gamePlayer.update(active);   // 更新玩家
    $gameTimer.update(active);    // 更新计时器
    $gameScreen.update();         // 更新画面效果
};
```

### 2.4 updateScene - 场景切换

```javascript
Scene_Map.prototype.updateScene = function() {
    this.checkGameover();       // 游戏结束检测
    this.updateDestination();   // 鼠标点击位置
    this.updateMainMultiply();  // 多次更新（加速模式）
    
    // 检测是否需要调用菜单
    if (this.isMenuCalled()) {
        this.callMenu();
    }
    
    // 检测是否需要切换场景
    if (this.isSceneChangeOk()) {
        this.updateTransferPlayer();  // 传送
        this.updateEncounter();       // 遭遇战
        this.updateCallDebug();       // 调试菜单
    }
};
```

## 三、精灵集创建

```javascript
Scene_Map.prototype.createSpriteset = function() {
    this._spriteset = new Spriteset_Map();
    this.addChild(this._spriteset);
};
```

**Spriteset_Map 结构**：
```
Spriteset_Map (Container)
├── _baseSprite
│   └── _backgroundFilter (模糊滤镜)
├── _parallax (远景图)
├── _tilemap (瓦片地图)
├── _characterContainer (角色层)
│   └── Sprite_Character × N
├── _destination (点击目标指示)
├── _weather (天气效果)
├── _pictureContainer (图片层)
└── _timerSprite (计时器)
```

## 四、窗口系统

```javascript
Scene_Map.prototype.createAllWindows = function() {
    this.createMapNameWindow();    // 地图名称
    this.createMessageWindow();    // 消息窗口
    this.createScrollTextWindow(); // 滚动文字
    this.createGoldWindow();       // 金币窗口
    this.createStatusWindow();     // 状态窗口
};

// 消息窗口是事件系统核心
Scene_Map.prototype.createMessageWindow = function() {
    this._messageWindow = new Window_Message();
    this.addWindow(this._messageWindow);
    
    // 关联子窗口
    this._messageWindow.setGoldWindow(this._goldWindow);
    this._messageWindow.setStatusWindow(this._statusWindow);
    
    // 创建选择窗口
    this._choiceWindow = new Window_ChoiceList();
    this._numberWindow = new Window_NumberInput();
    this._eventItemWindow = new Window_EventItem();
};
```

## 五、事件触发流程

```
玩家接触事件位置
    │
    ▼
Game_Player.checkEventTriggerHere()
    │ 检测碰撞的事件
    ▼
Game_Event.start()
    │ 启动事件
    ▼
Game_Interpreter.setup(list)
    │ 解析事件指令
    ▼
Window_Message.startMessage()
    │ 显示文字
    ▼
等待玩家按键
    │
    ▼
执行下一个指令
```

## 六、遭遇战系统

```javascript
Scene_Map.prototype.updateEncounter = function() {
    if ($gamePlayer.canEncounter()) {
        // 检测是否触发战斗
        if ($gameMap.isEncounterActive()) {
            var encounter = $gameMap.makeEncounterTroopId();
            if (encounter) {
                // 保存地图状态
                SceneManager.snapForBackground();
                
                // 进入战斗场景
                BattleManager.setup(encounter, false, false);
                SceneManager.push(Scene_Battle);
            }
        }
    }
};

// 遭遇率计算
Game_Player.prototype.canEncounter = function() {
    return $gameMap.canEncounter() && !this._encounterCount--;
};

// 每步减少 encounterCount
Game_Player.prototype.updateEncounterCount = function() {
    if (this.canEncounter()) {
        this._encounterCount = this.makeEncounterCount();
    }
};

// 随机步数
Game_Player.prototype.makeEncounterCount = function() {
    var n = $gameMap.encounterStep;
    // 20步平均遭遇，实际10-30步
    return Math.randomInt(n) + Math.randomInt(n) + 1;
};
```

## 七、传送系统

```javascript
Scene_Map.prototype.updateTransferPlayer = function() {
    if ($gamePlayer.isTransferring()) {
        // 淡出
        this.fadeOutForTransfer();
        
        // 执行传送
        $gamePlayer.performTransfer();
        
        // 加载新地图
        DataManager.loadMapData($gameMap.mapId());
        
        // 等待地图加载完成
        this._mapLoaded = false;
    }
};

Game_Player.prototype.performTransfer = function() {
    if (this._transferring) {
        // 如果是不同地图
        if (this._newMapId !== $gameMap.mapId()) {
            $gameMap.setup(this._newMapId);  // 设置新地图
        }
        
        // 移动玩家位置
        this.locate(this._newX, this._newY);
        this.setDirection(this._newDirection);
        
        this._transferring = false;
    }
};
```

## 八、菜单调用

```javascript
Scene_Map.prototype.isMenuCalled = function() {
    // ESC 键或 X 键
    return Input.isTriggered('menu') || TouchInput.isCancelled();
};

Scene_Map.prototype.callMenu = function() {
    // 禁用菜单时不响应
    if ($gameSystem.isMenuEnabled()) {
        // 保存状态
        $gameTemp.clearDestination();
        this._mapNameWindow.hide();
        
        // 推入菜单场景
        SceneManager.push(Scene_Menu);
        
        // 播放音效
        SoundManager.playOk();
    }
};
```

## 九、调试案例

### 案例1：显示当前地图信息

```javascript
console.log('地图ID:', $gameMap.mapId());
console.log('地图尺寸:', $gameMap.width(), 'x', $gameMap.height());
console.log('玩家位置:', $gamePlayer.x, ',', $gamePlayer.y);
console.log('事件数量:', $dataMap.events.filter(e => e).length);
```

### 案例2：强制传送

```javascript
// 传送到地图2，坐标(5,5)
$gamePlayer.reserveTransfer(2, 5, 5, 2, 0);
// 参数：地图ID, X, Y, 方向(2=下), 淡入淡出类型(0=黑)
```

### 案例3：触发战斗

```javascript
// 强制进入战斗
var troopId = 1;
BattleManager.setup(troopId, false, false);
SceneManager.snapForBackground();
SceneManager.push(Scene_Battle);
```

### 案例4：获取所有事件

```javascript
$gameMap.events().forEach(function(event) {
    console.log('事件', event.eventId(), ':', event.event().name, 
                '位置:', event.x, ',', event.y);
});
```

### 案例5：跳过淡入淡出

```javascript
Scene_Map.prototype.fadeInForTransfer = function() {
    // 原来会淡入
    // this.startFadeIn(this.fadeSpeed(), $gameMap.isBgmAutoplayed());
    // 改为直接显示
    $gameMap.autoplay();
};

Scene_Map.prototype.fadeOutForTransfer = function() {
    // 直接切换，不淡出
};
```
