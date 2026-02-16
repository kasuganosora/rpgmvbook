# Tilemap 类详解 - 从数据到画面

## 一个实际例子：玩家走到地图边缘时发生了什么？

当玩家在 Map001 中移动时，让我们追踪 Tilemap 的完整工作流程：

```
玩家按下右键
    ↓
Game_Player.moveStraight(6)
    ↓
$gamePlayer._x 从 8 变成 9
    ↓
Scene_Map 更新
    ↓
Spriteset_Map.update()
    ↓
Tilemap.update()  ← 我们在这里深入分析
    ↓
屏幕显示新位置
```

## Tilemap 核心属性详解

```javascript
// rpg_core.js 约 4515 行
function Tilemap() {
    this.initialize.apply(this, arguments);
}

Tilemap.prototype.initialize = function() {
    PIXI.Container.call(this);  // 继承 PIXI 容器
    
    //=== 基本尺寸 ===
    this._tileWidth = 48;   // 每个瓦片的像素宽度
    this._tileHeight = 48;  // 每个瓦片的像素高度
    
    //=== 地图尺寸 ===
    this._mapWidth = 0;     // 地图宽度（瓦片数）  Map001是17
    this._mapHeight = 0;    // 地图高度（瓦片数）  Map001是13
    
    //=== 视口控制 ===
    this._margin = 20;      // 额外绘制的边界瓦片数
                            // 为什么需要？防止滚动时边缘闪烁
                            // 实际绘制范围比屏幕大20个瓦片
    
    //=== 核心数据 ===
    this.bitmaps = [];      // 9个瓦片集图片 [A1, A2, A3, A4, A5, B, C, D, E]
    this.flags = [];        // 每个瓦片的属性标志（碰撞、优先级等）
    this._data = null;      // 地图瓦片数据数组
    
    //=== 渲染缓存 ===
    this._lowerBitmap = null;  // 下层位图（Layer 0-1）
    this._upperBitmap = null;  // 上层位图（Layer 2-3）
    
    //=== 动画状态 ===
    this._animationFrame = 0;   // 当前动画帧 0-3
    this._animationCount = 0;   // 帧计数器
    
    //=== 滚动状态 ===
    this._lastStartX = -1;  // 上次绘制的起始X
    this._lastStartY = -1;  // 上次绘制的起始Y
    this.origin = new Point(0, 0);  // 视口偏移（像素）
    
    //=== 循环地图 ===
    this.horizontalWrap = false;  // 水平循环
    this.verticalWrap = false;    // 垂直循环
};
```

## 地图数据是如何存储的？

### Map001.json 实际数据分析

```json
{
    "width": 17,
    "height": 13,
    "data": [2844, 2844, 2844, 0, 2844, 2844, 2844, 0, ...]
}
```

**data 数组长度计算**：
```
17 × 13 × 4 = 884 个元素
 ↑    ↑    ↑
 宽   高   4层
```

### 读取任意位置瓦片的数学原理

```javascript
// rpg_objects.js - Game_Map
Game_Map.prototype.tileId = function(x, y, z) {
    // 参数: x=地图列, y=地图行, z=图层(0-3)
    
    // 边界检查
    if (x < 0 || x >= this.width()) return 0;
    if (y < 0 || y >= this.height()) return 0;
    
    // 索引公式推导:
    // - 每层有 width × height 个瓦片
    // - Layer 0: 索引 0 ~ (width×height-1)
    // - Layer 1: 索引 (width×height) ~ (2×width×height-1)
    // - Layer 2: 索引 (2×width×height) ~ (3×width×height-1)
    // - Layer 3: 索引 (3×width×height) ~ (4×width×height-1)
    
    // 通用公式:
    var width = $dataMap.width;   // 17
    var height = $dataMap.height; // 13
    var index = (z * height + y) * width + x;
    
    return $dataMap.data[index];
};

//=== 实际例子 ===
// Map001 中读取位置 (5, 3) 的 Layer 0 瓦片:
// index = (0 * 13 + 3) * 17 + 5 = 3 * 17 + 5 = 56
// $dataMap.data[56] 的值就是该位置的瓦片ID
```

### 可视化数据布局

```
Map001 (17×13) 的 data 数组结构:

Layer 0 (基础层):
索引范围: 0 - 220 (17×13-1)
┌────┬────┬────┬────┬────┬────┐
│  0 │  1 │  2 │  3 │... │ 16 │  y=0
├────┼────┼────┼────┼────┼────┤
│ 17 │ 18 │ 19 │ 20 │... │ 33 │  y=1
├────┼────┼────┼────┼────┼────┤
│... │... │... │... │... │... │
└────┴────┴────┴────┴────┴────┘
  x=0  x=1  x=2  x=3      x=16

Layer 1 (装饰层1):
索引范围: 221 - 441
┌────┬────┬────┐
│221 │222 │... │
└────┴────┴────┘

Layer 2 (装饰层2):
索引范围: 442 - 662

Layer 3 (装饰层3):
索引范围: 663 - 883
```

## 渲染流程完整解析

### 第一步：初始化 - setData()

```javascript
// Spriteset_Map 创建 Tilemap 时调用
var tilemap = new Tilemap();
tilemap.setData(
    $gameMap.width(),   // 17
    $gameMap.height(),  // 13
    $gameMap.data()     // data数组
);

// Tilemap.setData 实现
Tilemap.prototype.setData = function(width, height, data) {
    // 简单地保存数据
    this._mapWidth = width;   // 17
    this._mapHeight = height; // 13
    this._data = data;        // 884个元素的数组
};
```

### 第二步：创建位图缓存

```javascript
Tilemap.prototype._createBitmaps = function() {
    // 计算需要多大的位图
    // width: 屏幕宽度 + 两侧边距
    // height: 屏幕高度 + 上下边距
    
    var width = this._width + this._margin * 2;
    // 假设屏幕 816px: 816 + 20*2 = 856
    
    var height = this._height + this._margin * 2;
    // 假设屏幕 624px: 624 + 20*2 = 664
    
    // 创建两个大位图作为缓存
    this._lowerBitmap = new Bitmap(width, height);  // 856 × 664
    this._upperBitmap = new Bitmap(width, height);
    
    // 为什么是两个？
    // - lowerBitmap: 先绘制，角色在它上面
    // - upperBitmap: 后绘制，覆盖角色（如树冠）
};
```

### 第三步：每帧更新 - update()

```javascript
Tilemap.prototype.update = function() {
    //=== 动画更新 ===
    this._animationCount++;
    if (this._animationCount >= 30) {
        // 每30帧 = 0.5秒 更新动画
        this._animationCount = 0;
        this._animationFrame = (this._animationFrame + 1) % 4;
        // 动画帧: 0 → 1 → 2 → 3 → 0 → ...
    }
    
    //=== 位置更新（核心！）===
    this._updatePosition();
};
```

### 第四步：位置更新 - _updatePosition()

```javascript
Tilemap.prototype._updatePosition = function() {
    //=== 计算视口起始瓦片 ===
    // origin.x 是像素偏移，需要转换为瓦片坐标
    
    var startX = Math.floor(this.origin.x / this._tileWidth);
    var startY = Math.floor(this.origin.y / this._tileHeight);
    
    // 例如: origin.x = 100px
    // startX = Math.floor(100 / 48) = 2
    // 意味着视口从第3列开始显示
    
    //=== 计算像素级偏移（平滑滚动）===
    var ox = this.origin.x % this._tileWidth;
    var oy = this.origin.y % this._tileHeight;
    
    // 例如: origin.x = 100px
    // ox = 100 % 48 = 4
    // 意味着需要向左偏移4像素
    
    //=== 应用位图偏移 ===
    if (this._lowerBitmap) {
        this._lowerBitmap.origin.x = ox;
        this._lowerBitmap.origin.y = oy;
    }
    if (this._upperBitmap) {
        this._upperBitmap.origin.x = ox;
        this._upperBitmap.origin.y = oy;
    }
    
    //=== 检查是否需要重绘 ===
    if (startX !== this._lastStartX || startY !== this._lastStartY) {
        // 视口移动到了新的瓦片，需要重绘
        this._paintAllTiles(startX, startY);
        this._lastStartX = startX;
        this._lastStartY = startY;
    }
};
```

### 第五步：绘制所有瓦片 - _paintAllTiles()

```javascript
Tilemap.prototype._paintAllTiles = function(startX, startY) {
    //=== 清空缓存位图 ===
    this._lowerBitmap.clear();
    this._upperBitmap.clear();
    
    //=== 计算绘制范围 ===
    // 需要多少列瓦片才能填满屏幕?
    var tileCols = Math.ceil(this._width / this._tileWidth) + 1;
    // 816 / 48 = 17，+1 = 18列
    
    // 需要多少行?
    var tileRows = Math.ceil(this._height / this._tileHeight) + 1;
    // 624 / 48 = 13，+1 = 14行
    
    //=== 遍历绘制 ===
    for (var y = 0; y < tileRows; y++) {
        for (var x = 0; x < tileCols; x++) {
            this._paintTiles(startX, startY, x, y);
        }
    }
    
    // 总共绘制: 18 × 14 = 252 个位置
    // 每个位置有4层，所以调用 _drawTile 1008 次
};
```

### 第六步：绘制单个位置 - _paintTiles()

```javascript
Tilemap.prototype._paintTiles = function(startX, startY, x, y) {
    //=== 计算地图坐标 ===
    var mx = startX + x;  // 地图上的实际列
    var my = startY + y;  // 地图上的实际行
    
    //=== 处理循环地图 ===
    if (this.horizontalWrap) {
        mx = mx.mod(this._mapWidth);  // 超出范围时回绕
    }
    if (this.verticalWrap) {
        my = my.mod(this._mapHeight);
    }
    
    //=== 计算绘制位置 ===
    var dx = (x - 1) * this._tileWidth;   // 位图上的X
    var dy = (y - 1) * this._tileHeight;  // 位图上的Y
    // 为什么 -1？因为 _margin 的存在，实际绘制从 (-48, -48) 开始
    
    //=== 读取4层瓦片ID ===
    var tileId0 = this._readTileId(mx, my, 0);  // 基础层
    var tileId1 = this._readTileId(mx, my, 1);  // 装饰层1
    var tileId2 = this._readTileId(mx, my, 2);  // 装饰层2
    var tileId3 = this._readTileId(mx, my, 3);  // 装饰层3
    
    //=== 分层绘制 ===
    // Layer 0-1 → 下层（角色下面）
    this._drawTile(this._lowerBitmap, tileId0, dx, dy);
    this._drawTile(this._lowerBitmap, tileId1, dx, dy);
    
    // Layer 2-3 → 上层（角色上面）
    this._drawTile(this._upperBitmap, tileId2, dx, dy);
    this._drawTile(this._upperBitmap, tileId3, dx, dy);
};
```

### 第七步：绘制单个瓦片 - _drawTile()

```javascript
Tilemap.prototype._drawTile = function(bitmap, tileId, dx, dy) {
    // tileId = 0 表示空，跳过
    if (tileId <= 0) return;
    
    //=== 判断瓦片类型 ===
    if (Tilemap.isAutotile(tileId)) {
        // 自动瓦片（A1-A4）
        this._drawAutotile(bitmap, tileId, dx, dy);
    } else {
        // 普通瓦片（B-E 和 A5）
        this._drawNormalTile(bitmap, tileId, dx, dy);
    }
};
```

## 普通瓦片绘制详解

```javascript
Tilemap.prototype._drawNormalTile = function(bitmap, tileId, dx, dy) {
    //=== 计算瓦片集索引 ===
    // tileId 范围: 0-255 (B-E瓦片)
    // 
    // B瓦片: tileId 0-255   → bitmaps[5]
    // C瓦片: 需要特殊处理...
    // 
    // 实际公式:
    var setNumber = 4 + Math.floor(tileId / 256);
    // tileId=0-255: setNumber=4 (B)
    // tileId=256-511: 需要其他逻辑...
    
    // 简化版本：
    // B: setNumber=5, C: setNumber=6, D: setNumber=7, E: setNumber=8
    
    //=== 计算源图位置 ===
    var w = this._tileWidth;   // 48
    var h = this._tileHeight;  // 48
    
    // 每个瓦片集是 768×768
    // 16列 × 16行 = 256个瓦片
    var sx = (Math.floor(tileId / 128) % 2) * w;  // 列
    var sy = (tileId % 256) * h;                   // 行
    
    //=== 执行复制 ===
    var source = this.bitmaps[setNumber];
    if (source) {
        // bitmap.blt(源图, 源X, 源Y, 源宽, 源高, 目标X, 目标Y, 目标宽, 目标高)
        bitmap.blt(source, sx, sy, w, h, dx, dy, w, h);
    }
};
```

## 实际案例：追踪一次完整绘制

### 场景设定
- 玩家在 Map001 的位置 (8, 6)
- 屏幕尺寸 816×624
- 视口中心在玩家位置

### 计算视口起始位置

```javascript
// Game_Player
var centerX = 8;  // 玩家X
var centerY = 6;  // 玩家Y

// Scene_Map / Spriteset_Map
var displayX = 8 - (816/48 - 1) / 2;  // 8 - 8 = 0
var displayY = 6 - (624/48 - 1) / 2;  // 6 - 6 = 0

// Tilemap.origin
origin.x = displayX * 48;  // 0 * 48 = 0
origin.y = displayY * 48;  // 0 * 48 = 0

// 起始瓦片
startX = Math.floor(0 / 48);  // 0
startY = Math.floor(0 / 48);  // 0
```

### 绘制范围

```
地图范围: (0,0) 到 (16,12)
绘制范围: 18列 × 14行 = 252个位置

因为 startX=0, startY=0:
- X方向: 0 到 17
- Y方向: 0 到 13

部分位置会超出地图边界(17×13)，需要边界检查
```

### 位置 (0, 0) 的绘制过程

```javascript
// 步骤1: 读取4层瓦片
var tileId0 = _readTileId(0, 0, 0);  // 2844 (草地)
var tileId1 = _readTileId(0, 0, 1);  // 0 (空)
var tileId2 = _readTileId(0, 0, 2);  // 0 (空)
var tileId3 = _readTileId(0, 0, 3);  // 0 (空)

// 步骤2: 绘制Layer 0
// tileId=2844, 是自动瓦片
_drawAutotile(_lowerBitmap, 2844, -48, -48);
// dx=-48 因为 x=0, dx = (0-1)*48 = -48
// 负坐标是正常的，会被 margin 区域覆盖

// 步骤3-5: 其他层都是0，跳过
```

## 性能优化原理

### 为什么用两个大位图而不是每个瓦片一个精灵？

```
方案A (错误): 每个瓦片一个Sprite
- 17×13×4 = 884 个 Sprite 对象
- 每帧需要更新884个对象的transform
- GPU需要884次draw call
- 结果: 卡顿

方案B (正确): 两个大位图缓存
- 只需2个 Sprite 对象
- 只在视口移动时重绘
- GPU只需2次draw call
- 结果: 流畅
```

### 增量更新策略

```javascript
// 只有视口移动到新瓦片时才重绘
if (startX !== this._lastStartX || startY !== this._lastStartY) {
    this._paintAllTiles(startX, startY);
}

// 例如: 玩家从 (8,6) 走到 (8.1, 6)
// origin.x = 8.1 * 48 = 388.8
// startX = floor(388.8 / 48) = 8
// _lastStartX = 8
// startX === _lastStartX → 不重绘，只偏移位图
```

## 调试技巧

### 可视化视口范围

```javascript
// 在游戏中按F8，输入:
var tilemap = SceneManager._scene._spriteset._tilemap;
console.log('视口起始:', tilemap._lastStartX, tilemap._lastStartY);
console.log('origin:', tilemap.origin.x, tilemap.origin.y);
console.log('位图尺寸:', tilemap._lowerBitmap.width, tilemap._lowerBitmap.height);
```

### 强制刷新

```javascript
// 强制重绘整个地图
$gameMap._tilemap.refresh();

// 这会清空缓存并重新绘制所有可见瓦片
```

### 查看瓦片集加载状态

```javascript
// 检查所有瓦片集是否加载完成
var tilemap = SceneManager._scene._spriteset._tilemap;
for (var i = 0; i < tilemap.bitmaps.length; i++) {
    var bitmap = tilemap.bitmaps[i];
    if (bitmap) {
        console.log('瓦片集' + i + ':', bitmap.isReady() ? '已加载' : '加载中');
    } else {
        console.log('瓦片集' + i + ': 未设置');
    }
}
```
