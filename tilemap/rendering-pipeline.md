# 地图渲染流程 - 从玩家移动到画面更新

## 完整流程图（带时间线）

```
时间: 0ms                    16ms                   33ms
      │                       │                      │
      ▼                       ▼                      ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ 玩家按下右键  │       │ 第二帧开始   │       │ 渲染到屏幕   │
│ Input.update │ ────▶ │ 更新游戏逻辑 │ ────▶ │ GPU绘制     │
└──────────────┘       └──────────────┘       └──────────────┘
```

## 第一阶段：输入处理

### 玩家移动处理

```javascript
Game_Player.prototype.update = function(sceneActive) {
    var lastScrolledX = this.scrolledX();  // 保存旧位置
    var lastScrolledY = this.scrolledY();
    
    if (sceneActive) {
        this.moveByInput();  // 根据输入移动
    }
    
    Game_Character.prototype.update.call(this);  // 执行移动
    this.updateScroll(lastScrolledX, lastScrolledY);  // 更新视口
};
```

### 实际移动

```javascript
// Game_CharacterBase
Game_CharacterBase.prototype.moveStraight = function(d) {
    this.setDirection(d);  // 设置朝向
    
    // 计算新坐标
    this._x = $gameMap.roundXWithDirection(this._x, d);  // 8 → 9
    this._y = $gameMap.roundYWithDirection(this._y, d);
    
    // _realX 会逐渐追赶 _x，实现平滑效果
    this._realX = $gameMap.xWithDirection(this._x, this.reverseDir(d));
    this._realY = $gameMap.yWithDirection(this._y, this.reverseDir(d));
    
    this._isMoving = true;
};
```

## 第二阶段：视口滚动

```javascript
Game_Player.prototype.updateScroll = function(lastScrolledX, lastScrolledY) {
    var x1 = lastScrolledX;  // 旧值
    var y1 = lastScrolledY;
    var x2 = this.scrolledX();  // 新值
    var y2 = this.scrolledY();
    
    // 如果玩家超过屏幕中心，滚动地图
    if (x2 > x1 && x2 > this.centerX()) {
        $gameMap.scrollRight(x2 - x1);  // displayX 增加
    }
    // ... 其他方向
};
```

## 第三阶段：Tilemap更新

### 关键代码

```javascript
Spriteset_Map.prototype.updateTilemap = function() {
    // 将 displayX (瓦片) 转换为 origin.x (像素)
    this._tilemap.origin.x = $gameMap.displayX() * $gameMap.tileWidth();
    // displayX=0.1 → origin.x=4.8像素
    
    this._tilemap.update();
};

Tilemap.prototype._updatePosition = function() {
    var startX = Math.floor(this.origin.x / this._tileWidth);  // 起始瓦片
    var ox = Math.floor(this.origin.x % this._tileWidth);      // 像素偏移
    
    // 应用偏移
    this._lowerSprite.move(-ox, -oy);  // 向左偏移实现滚动
    this._upperSprite.move(-ox, -oy);
    
    // 只有 startX 变化时才重绘
    if (startX !== this._lastStartX) {
        this._paintAllTiles(startX, startY);
    }
};
```

## 平滑滚动的数学原理

```
玩家从 (8,6) 走到 (9,6):

帧1: _x=8, _realX=8     → 静止
帧2: _x=9, _realX=8     → 开始移动
帧3: _x=9, _realX=8.06  → 平滑过渡
...
帧17: _x=9, _realX=9    → 移动完成

每帧移动: distancePerFrame = 2^moveSpeed / 256
moveSpeed=4: 16/256 ≈ 0.0625 瓦片/帧
```

## 项目实例：Map001 分析

```javascript
// 玩家初始位置 (8, 6)
// 地图尺寸 17×13
// 屏幕尺寸 816×624 (17×13 瓦片)

// 玩家在中心:
// displayX = 8 - (17-1)/2 = 0
// displayY = 6 - (13-1)/2 = 0

// 当玩家走到 (9, 6):
// scrolledX = 9 - 0 = 9
// centerX = 8
// 9 > 8 → scrollRight(0.06)

// displayX 从 0 变成 0.06
// origin.x = 0.06 * 48 = 2.88 像素
// 地图向左滚动 2.88 像素
```

## 调试技巧

```javascript
// 查看当前状态
console.log('玩家位置:', $gamePlayer.x, $gamePlayer.y);
console.log('视口位置:', $gameMap.displayX(), $gameMap.displayY());
console.log('Tilemap origin:', $gameMap._tilemap.origin);

// 强制刷新地图
$gameMap._tilemap.refresh();
```
