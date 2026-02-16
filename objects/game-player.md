# Game_Player - 玩家对象

## 一、职责

Game_Player 继承自 Game_Character，处理玩家特有逻辑：
- 跟随玩家输入
- 触发事件
- 管理遭遇战
- 处理传送

## 二、核心方法

### 2.1 update - 每帧更新

```javascript
Game_Player.prototype.update = function(sceneActive) {
    // 更新输入（移动）
    if (sceneActive) {
        this.updateInput();
    }
    
    // 更新跟随者
    this.updateFollowers();
    
    // 更新动画
    this.updateAnimation();
    
    // 调用父类更新
    Game_Character.prototype.update.call(this);
};
```

### 2.2 updateInput - 处理输入

```javascript
Game_Player.prototype.updateInput = function() {
    // 移动中不处理新输入
    if (this.isMoving()) return;
    
    // 执行强制移动路线时不处理
    if (this.isMoveRouteForcing()) return;
    
    // 获取方向
    var direction = this.getInputDirection();
    
    if (direction > 0) {
        // 移动
        this.executeMove(direction);
        
        // 更新遭遇计数
        this.updateEncounterCount();
    }
};

Game_Player.prototype.getInputDirection = function() {
    // 方向优先级：下 > 左 > 右 > 上
    if (Input.isPressed('down')) return 2;
    if (Input.isPressed('left')) return 4;
    if (Input.isPressed('right')) return 6;
    if (Input.isPressed('up')) return 8;
    return 0;
};

Game_Player.prototype.executeMove = function(direction) {
    // 冲刺
    if (this.isDashPressed()) {
        this.startDash();
    } else {
        this.endDash();
    }
    
    // 移动
    this.moveStraight(direction);
};
```

## 三、事件触发系统

### 3.1 checkEventTriggerHere - 当前位置

```javascript
// 触发当前位置的事件（自动/并行）
Game_Player.prototype.checkEventTriggerHere = function(triggers) {
    if (this.canStartLocalEvents()) {
        var events = $gameMap.eventsXy(this.x, this.y);
        
        events.forEach(function(event) {
            // 检查触发类型是否匹配
            if (triggers.contains(event.trigger())) {
                // 触发=2(自动)或3(并行)
                if (!event.isErased() && event.isTriggerIn(triggers)) {
                    event.start();
                }
            }
        });
    }
};
```

### 3.2 checkEventTriggerThere - 前方位置

```javascript
// 触发前方一格的事件（动作按钮）
Game_Player.prototype.checkEventTriggerThere = function(triggers) {
    if (this.canStartLocalEvents()) {
        // 计算前方坐标
        var x = $gameMap.roundXWithDirection(this.x, this._direction);
        var y = $gameMap.roundYWithDirection(this.y, this._direction);
        
        var events = $gameMap.eventsXy(x, y);
        
        events.forEach(function(event) {
            if (triggers.contains(event.trigger())) {
                if (!event.isErased() && event.isTriggerIn(triggers)) {
                    event.start();
                }
            }
        });
        
        // 如果没有事件，检查柜台
        if (events.length === 0) {
            this.checkEventTriggerCounter(triggers);
        }
    }
};
```

### 3.3 checkEventTriggerTouchFront - 接触触发

```javascript
// 玩家移动时检测前方接触事件
Game_Player.prototype.checkEventTriggerTouchFront = function(d) {
    if (this.canStartLocalEvents()) {
        var x = $gameMap.roundXWithDirection(this.x, d);
        var y = $gameMap.roundYWithDirection(this.y, d);
        this.checkEventTriggerTouch(x, y);
    }
};

// 检查接触触发
Game_Player.prototype.checkEventTriggerTouch = function(x, y) {
    if (this.canStartLocalEvents()) {
        var events = $gameMap.eventsXy(x, y);
        
        events.forEach(function(event) {
            // 触发类型=1(接触)
            if (event.trigger() === 1 && !event.isErased()) {
                if (!this.isInAirship()) {
                    event.start();
                }
            }
        }, this);
    }
};
```

## 四、跟随者系统

```javascript
Game_Player.prototype.updateFollowers = function() {
    // 更新所有跟随角色
    this.followers().update();
};

// Game_Followers 类
Game_Followers.prototype.update = function() {
    // 从后往前更新（后面的跟随前面的）
    for (var i = this._data.length - 1; i >= 0; i--) {
        var follower = this._data[i];
        follower.update();
        
        // 跟随前一个角色
        if (i > 0) {
            var target = this._data[i - 1];
            follower.chaseCharacter(target);
        } else {
            // 第一个跟随者跟随玩家
            follower.chaseCharacter($gamePlayer);
        }
    }
};

// 跟随逻辑
Game_Follower.prototype.chaseCharacter = function(character) {
    // 移动到目标位置
    var sx = this.deltaXFrom(character.x);
    var sy = this.deltaYFrom(character.y);
    
    if (sx !== 0 || sy !== 0) {
        this.moveTowardCharacter(character);
    }
};
```

## 五、传送系统

### 5.1 reserveTransfer - 预约传送

```javascript
Game_Player.prototype.reserveTransfer = function(mapId, x, y, d, fadeType) {
    this._transferring = true;      // 标记正在传送
    this._newMapId = mapId;         // 目标地图
    this._newX = x;                 // 目标X
    this._newY = y;                 // 目标Y
    this._newDirection = d || 0;    // 传送后方向
    this._fadeType = fadeType || 0; // 淡入淡出类型
};

// 检测是否正在传送
Game_Player.prototype.isTransferring = function() {
    return this._transferring;
};
```

### 5.2 performTransfer - 执行传送

```javascript
Game_Player.prototype.performTransfer = function() {
    if (this._transferring) {
        // 记录是否是不同地图
        var newMap = this._newMapId !== $gameMap.mapId();
        
        if (newMap) {
            // 设置新地图
            $gameMap.setup(this._newMapId);
        }
        
        // 设置位置
        this.locate(this._newX, this._newY);
        
        // 设置方向
        if (this._newDirection > 0) {
            this.setDirection(this._newDirection);
        }
        
        // 重置跟随者
        if (newMap) {
            this.followers().synchronize($dataMap.startX, $dataMap.startY, this.direction());
        }
        
        this._transferring = false;
    }
};
```

## 六、交通工具系统

### 6.1 乘坐/下船

```javascript
Game_Player.prototype.getOnOffVehicle = function() {
    if (this.isInVehicle()) {
        return this.getOffVehicle();
    } else {
        return this.getOnVehicle();
    }
};

Game_Player.prototype.getOnVehicle = function() {
    // 检查前方是否有交通工具
    var direction = this.direction();
    var x = $gameMap.roundXWithDirection(this.x, direction);
    var y = $gameMap.roundYWithDirection(this.y, direction);
    
    // 检测三种交通工具
    if ($gameMap.ship().pos(x, y)) {
        this._vehicleType = 'ship';
    } else if ($gameMap.boat().pos(x, y)) {
        this._vehicleType = 'boat';
    } else if ($gameMap.airship().pos(x, y)) {
        this._vehicleType = 'airship';
    }
    
    if (this.isInVehicle()) {
        this._vehicleGettingOn = true;
        this._vehicleType = this._vehicleType;
        return true;
    }
    return false;
};

Game_Player.prototype.getOffVehicle = function() {
    if (this.isInAirship()) {
        // 飞船只能在可通行位置降落
        if (!this.canLand()) {
            return false;
        }
    }
    
    this._vehicleGettingOff = true;
    return true;
};
```

## 七、遭遇战系统

```javascript
Game_Player.prototype.canEncounter = function() {
    // 检查是否禁用遭遇
    if ($gameSystem.isEncounterEnabled() === false) {
        return false;
    }
    
    // 乘坐交通工具时不遭遇
    if (this.isInVehicle()) {
        return false;
    }
    
    // 计数器归零时触发
    return this._encounterCount <= 0;
};

// 每步减少计数器
Game_Player.prototype.updateEncounterCount = function() {
    if (this.canEncounter()) {
        this._encounterCount = this.makeEncounterCount();
    }
};

// 计算遭遇步数
Game_Player.prototype.makeEncounterCount = function() {
    var n = $gameMap.encounterStep();
    // 两个随机值相加，范围是 n ~ 2n
    return Math.randomInt(n) + Math.randomInt(n) + 1;
};
```

## 八、调试案例

### 案例1：查看玩家状态

```javascript
console.log('位置:', $gamePlayer.x, $gamePlayer.y);
console.log('方向:', $gamePlayer.direction());
console.log('地图:', $gameMap.mapId());
console.log('移动中:', $gamePlayer.isMoving());
console.log('传送中:', $gamePlayer.isTransferring());
console.log('交通工具:', $gamePlayer.vehicleType());
```

### 案例2：强制传送

```javascript
// 传送到地图2，坐标(5,5)，朝向下方
$gamePlayer.reserveTransfer(2, 5, 5, 2, 0);

// 触发传送（通常由 Scene_Map 自动处理）
$gamePlayer.performTransfer();
```

### 案例3：修改移动速度

```javascript
// 速度范围 1-6
$gamePlayer.setMoveSpeed(6);  // 最快

// 频率范围 1-5
$gamePlayer.setMoveFrequency(5);
```

### 案例4：显示跟随者

```javascript
// 启用跟随者
$gamePlayer.showFollowers();
$gamePlayer.setTransparent(false);

// 隐藏跟随者
$gamePlayer.hideFollowers();

// 集合跟随者
$gamePlayer.followers().gather();
```

### 案例5：强制进入战斗

```javascript
// 触发战斗
var troopId = 1;
BattleManager.setup(troopId, false, false);
SceneManager.push(Scene_Battle);
```

### 案例6：获取前方位置

```javascript
var d = $gamePlayer.direction();
var x = $gameMap.roundXWithDirection($gamePlayer.x, d);
var y = $gameMap.roundYWithDirection($gamePlayer.y, d);
console.log('前方位置:', x, y);

// 检查前方是否有事件
var events = $gameMap.eventsXy(x, y);
console.log('前方事件:', events.map(e => e.event().name));
```
