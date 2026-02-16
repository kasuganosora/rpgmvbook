# Game_Character - 角色基类

## 一、继承关系

```
Game_CharacterBase (基础属性)
    └── Game_Character (移动逻辑)
         ├── Game_Player (玩家)
         ├── Game_Event (事件)
         └── Game_Vehicle (交通工具)
```

## 二、坐标系统

```javascript
// 瓦片坐标 vs 像素坐标
Game_CharacterBase.prototype.x          // 瓦片X (0, 1, 2...)
Game_CharacterBase.prototype.y          // 瓦片Y (0, 1, 2...)
Game_CharacterBase.prototype._realX     // 实际X (0.0, 0.5, 1.0...)
Game_CharacterBase.prototype._realY     // 实际Y (0.0, 0.5, 1.0...)

// 转换关系
// realX 在移动时会渐变，x 在到达时直接跳跃
// 例如：从 x=0 移动到 x=1
// realX: 0.0 → 0.5 → 1.0
// x: 0 → 0 → 1 (到达时)
```

### 实际案例

```javascript
// 玩家在 (5, 3) 开始向右移动
$gamePlayer.x      // 5
$gamePlayer._realX // 5.0

// 移动中（移动50%）
$gamePlayer.x      // 5 (还未到达)
$gamePlayer._realX // 5.5

// 到达后
$gamePlayer.x      // 6
$gamePlayer._realX // 6.0
```

## 三、方向系统

```javascript
// 方向常量
var Direction = {
    DOWN:  2,
    LEFT:  4,
    RIGHT: 6,
    UP:    8
};

// 方向转换：方向 → dx/dy
Game_CharacterBase.prototype.directionToDx = function(direction) {
    switch (direction) {
        case 2: return { dx: 0, dy: 1 };  // 下
        case 4: return { dx: -1, dy: 0 }; // 左
        case 6: return { dx: 1, dy: 0 };  // 右
        case 8: return { dx: 0, dy: -1 }; // 上
    }
};
```

## 四、移动系统

### 4.1 moveStraight - 直线移动

```javascript
Game_Character.prototype.moveStraight = function(d) {
    // 检查是否可以移动
    if (this.canPass(this.x, this.y, d)) {
        // 设置方向
        this.setDirection(d);
        
        // 更新坐标
        this._x = $gameMap.roundXWithDirection(this.x, d);
        this._y = $gameMap.roundYWithDirection(this.y, d);
        
        // 增加步数
        this._stepCount++;
        
        // 触发接触事件
        this.checkEventTriggerTouchFront(d);
    } else {
        // 不能移动时设置方向
        this.setDirection(d);
        this.checkEventTriggerTouchFront(d);
    }
};
```

### 4.2 canPass - 碰撞检测详解

```javascript
Game_CharacterBase.prototype.canPass = function(x, y, d) {
    // 获取目标坐标
    var x2 = $gameMap.roundXWithDirection(x, d);
    var y2 = $gameMap.roundYWithDirection(y, d);
    
    // 1. 检查地图边界（循环地图特殊处理）
    if (!$gameMap.isValid(x2, y2)) {
        return false;
    }
    
    // 2. 检查穿透状态
    if (this.isThrough() || this.isDebugThrough()) {
        return true;
    }
    
    // 3. 检查地图通行性
    if (!this.isMapPassable(x, y, d)) {
        return false;
    }
    
    // 4. 检查与事件的碰撞
    if (this.isCollidedWithCharacters(x2, y2)) {
        return false;
    }
    
    return true;
};
```

### 4.3 地图通行性检测

```javascript
Game_CharacterBase.prototype.isMapPassable = function(x, y, d) {
    var vehicle = this.vehicle();
    if (vehicle) {
        // 乘交通工具时使用交通工具的通行设置
        return vehicle.isMapPassable(x, y, d);
    }
    
    // 获取当前和目标瓦片
    var tileId1 = $gameMap.tileId(x, y, 0);
    var tileId2 = $gameMap.tileId(
        $gameMap.roundXWithDirection(x, d),
        $gameMap.roundYWithDirection(y, d),
        0
    );
    
    // 检查方向通行标志
    // 标志位：0b1111 = 下左右上
    var flag = 0x0f;
    switch (d) {
        case 2: flag = 0x01; break; // 下 → 检查第0位
        case 4: flag = 0x02; break; // 左 → 检查第1位
        case 6: flag = 0x04; break; // 右 → 检查第2位
        case 8: flag = 0x08; break; // 上 → 检查第3位
    }
    
    return $gameMap.isPassable(tileId1, flag) && $gameMap.isPassable(tileId2, flag);
};
```

## 五、角色碰撞检测

```javascript
Game_CharacterBase.prototype.isCollidedWithCharacters = function(x, y) {
    // 检测与事件的碰撞
    if (this.isCollidedWithEvents(x, y)) {
        return true;
    }
    
    // 检测与交通工具的碰撞
    if (this.isCollidedWithVehicles(x, y)) {
        return true;
    }
    
    return false;
};

Game_CharacterBase.prototype.isCollidedWithEvents = function(x, y) {
    var events = $gameMap.eventsXy(x, y);
    return events.some(function(event) {
        // 可穿透或有特定优先级的事件不碰撞
        return !event.isThrough() && event.isNormalPriority();
    });
};
```

## 六、update - 每帧更新

```javascript
Game_CharacterBase.prototype.update = function() {
    // 每帧移动 1/16 格（60帧移动一格）
    this.updateMove();
    
    // 更新动画计数器
    this.updateAnimation();
};

Game_CharacterBase.prototype.updateMove = function() {
    // 检查是否正在移动
    if (this.isMoving()) {
        // 计算 _realX/Y 的增量
        // distancePerFrame = 1/16（正常速度）
        this._realX = (this._realX * 15 + this._x) / 16;
        this._realY = (this._realY * 15 + this._y) / 16;
        
        // 到达目的地时
        if (Math.abs(this._realX - this._x) < 0.001) {
            this._realX = this._x;
            this._realY = this._y;
        }
    }
};
```

**移动计算详解**：
```
每帧：realX = (realX * 15 + targetX) / 16

从 0 移动到 1：
帧1: (0.0 * 15 + 1) / 16 = 0.0625
帧2: (0.0625 * 15 + 1) / 16 = 0.121...
帧8: (0.512 * 15 + 1) / 16 = 0.542...
帧16: (0.956 * 15 + 1) / 16 = 0.998...

约16帧（0.27秒）完成一格移动
```

## 七、行走图动画

```javascript
Game_CharacterBase.prototype.updateAnimation = function() {
    this._animationCount++;
    
    if (this.isMoving()) {
        // 移动中：每12帧切换图案
        if (this._animationCount >= 12) {
            this._animationCount = 0;
            this._pattern = (this._pattern + 1) % 3;
        }
    } else {
        // 停止：每12帧检查是否需要眨眼
        if (this._animationCount >= 12) {
            this._animationCount = 0;
            this._pattern = 1;  // 中间图案
        }
    }
};
```

**图案索引**：
```
行走图结构（3列×4行）：
┌───┬───┬───┐
│ 0 │ 1 │ 2 │  ← pattern索引 (左脚-站立-右脚)
├───┼───┼───┤
│   │   │   │  ← direction=2 (下)
├───┼───┼───┤
│   │   │   │  ← direction=4 (左)
├───┼───┼───┤
│   │   │   │  ← direction=6 (右)
├───┼───┼───┤
│   │   │   │  ← direction=8 (上)
└───┴───┴───┘
```

## 八、调试案例

### 案例1：查看角色状态

```javascript
console.log('位置:', $gamePlayer.x, $gamePlayer.y);
console.log('实际位置:', $gamePlayer._realX, $gamePlayer._realY);
console.log('方向:', $gamePlayer.direction());
console.log('移动中:', $gamePlayer.isMoving());
console.log('图案:', $gamePlayer._pattern);
```

### 案例2：强制移动

```javascript
// 强制移动到指定位置（瞬间）
$gamePlayer.locate(10, 5);

// 平滑移动一格
$gamePlayer.moveStraight(6);  // 向右

// 随机移动
$gamePlayer.moveRandom();
```

### 案例3：设置穿透

```javascript
// 开启穿透模式（穿墙）
$gamePlayer._through = true;

// 关闭穿透
$gamePlayer._through = false;
```

### 案例4：修改移动速度

```javascript
// 速度范围 1-6
$gamePlayer.setMoveSpeed(6);  // 最快

// 频率范围 1-5（影响等待时间）
$gamePlayer.setMoveFrequency(5);
```

### 案例5：检查碰撞

```javascript
// 检查能否向上移动
var canMoveUp = $gamePlayer.canPass($gamePlayer.x, $gamePlayer.y, 8);
console.log('能向上移动:', canMoveUp);

// 检查某位置是否有事件
var events = $gameMap.eventsXy(5, 5);
console.log('(5,5)的事件:', events.map(e => e.event().name));
```
