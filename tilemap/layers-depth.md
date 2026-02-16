# 图层与深度

## 四层地图系统

RPG Maker MV 地图使用4层瓦片存储：

```
Layer 3 (上层装饰2)  ← 存储索引: 3
Layer 2 (上层装饰1)  ← 存储索引: 2
Layer 1 (下层覆盖)   ← 存储索引: 1  
Layer 0 (基础地形)   ← 存储索引: 0
```

### 数据存储格式

```javascript
// MapXXX.json 中的 data 数组
// 17x13 地图示例
"data": [
    // (0,0) 位置
    2844,  // Layer 0: 基础草地 (A2自动瓦片)
    0,     // Layer 1: 空
    0,     // Layer 2: 空
    0,     // Layer 3: 空
    
    // (1,0) 位置
    2844,  // Layer 0
    0, 0, 0,
    
    // ... 共 17*13*4 = 884 个元素
]
```

### 读取瓦片

```javascript
// Game_Map 中的读取方法
Game_Map.prototype.tileId = function(x, y, z) {
    var width = $dataMap.width;
    var height = $dataMap.height;
    
    if (x >= 0 && x < width && y >= 0 && y < height) {
        // 索引公式: (z * height + y) * width + x
        return $dataMap.data[(z * height + y) * width + x];
    }
    return 0;
};
```

## 瓦片标志系统

每个瓦片有一个32位标志值，控制各种属性：

```javascript
// 位掩码定义
var Flag = {
    BLOCK_LOWER:     0x0001,  // 阻挡下层瓦片显示
    BLOCK_UPPER:     0x0002,  // 阻挡上层瓦片显示
    LADDER:          0x0004,  // 梯子 (可上下通行)
    BUSH:            0x0008,  // 草丛 (角色半透明)
    COUNTER:         0x0020,  // 柜台 (可隔空交互)
    DAMAGE_FLOOR:    0x0100,  // 伤害地板
    PASSABLE_DIR:    0x0F00   // 方向通行位
};

// 方向位
// 0x0100: 下可通行
// 0x0200: 左可通行
// 0x0400: 右可通行
// 0x0800: 上可通行
```

### 标志检查

```javascript
Game_Map.prototype.checkPassage = function(x, y, bit) {
    var flags = this.tilesetFlags();
    var tiles = this.allTiles(x, y);
    
    for (var i = 0; i < tiles.length; i++) {
        var flag = flags[tiles[i]];
        if ((flag & 0x10) !== 0) continue;  // 跳过☆瓦片
        if ((flag & bit) === 0) return true;  // 可通行
        if ((flag & bit) === bit) return false;  // 不可通行
    }
    return false;
};
```

## 优先级系统

### 瓦片优先级

```javascript
// 检测瓦片是否为上层瓦片
Game_Map.prototype.isLayer2Tile = function(x, y) {
    var flags = this.tilesetFlags();
    var tiles = this.layeredTiles(x, y);
    for (var i = 0; i < tiles.length; i++) {
        var flag = flags[tiles[i]];
        if ((flag & 0x0001) !== 0) {
            return true;  // Block Lower = 上层瓦片
        }
    }
    return false;
};
```

### 角色优先级

```javascript
// Game_CharacterBase
this._priorityType = 1;  // 0=低, 1=普通, 2=高

Game_CharacterBase.prototype.screenZ = function() {
    // 低优先级: Z = 0*2 + 3 = 3 (与下层瓦片同级)
    // 普通优先级: Z = 1*2 + 3 = 5 (在上层瓦片之上)
    // 高优先级: Z = 2*2 + 3 = 7 (在最上层)
    return this._priorityType * 2 + 3;
};
```

## Z坐标详解

### 完整Z值表

| Z值 | 描述 | 示例 |
|-----|------|------|
| 0 | 下层瓦片 | 地面、水面 |
| 1 | 低优先级角色 | 特殊效果 |
| 2 | (保留) | - |
| 3 | 普通角色 (低优先级) | 某些事件 |
| 4 | 上层瓦片 | 树冠、屋顶 |
| 5 | 普通角色 | 玩家、NPC |
| 6 | 高优先级角色 | 飞行效果 |
| 7 | 飞船阴影 | - |
| 8 | 气球图标 | 感叹号、问号 |
| 9 | 动画效果 | 技能特效 |
| 10 | 目的地指示 | 鼠标点击位置 |

### 渲染排序

```javascript
// PIXI.js 内部按 Y 和 Z 排序
// 同 Z 值时，Y 值小的先渲染 (被遮挡)

// 角色间排序
Sprite_Character.prototype.updatePosition = function() {
    this.x = this._character.screenX();
    this.y = this._character.screenY();
    this.z = this._character.screenZ();
};

// 伪深度效果
// 角色A在 (5, 10) - Z=5
// 角色B在 (5, 12) - Z=5
// A 先渲染，B 在 A 前面
```

## 上下层分离渲染

### Tilemap 中的实现

```javascript
Tilemap.prototype._paintTiles = function(startX, startY, x, y) {
    // ... 获取4层瓦片
    
    // Layer 0-1: 绘制到下层位图
    this._drawTile(this._lowerBitmap, tileId0, dx, dy);
    this._drawTile(this._lowerBitmap, tileId1, dx, dy);
    
    // Layer 2-3: 绘制到上层位图
    this._drawTile(this._upperBitmap, tileId2, dx, dy);
    this._drawTile(this._upperBitmap, tileId3, dx, dy);
};
```

### 渲染顺序

```
1. 绘制 Lower Bitmap (Layer 0-1)
       ↓
2. 渲染角色层 (Z = 1, 3, 5)
       ↓
3. 绘制 Upper Bitmap (Layer 2-3)
       ↓
4. 渲染特效层 (Z = 7+)
```

## 视差滚动 (Parallax)

```javascript
// 地图可设置远景图
Game_Map.prototype.setupParallax = function() {
    this._parallaxName = $dataMap.parallaxName || '';
    this._parallaxZero = ImageManager.isZeroParallax(this._parallaxName);
    this._parallaxLoopX = $dataMap.parallaxLoopX;
    this._parallaxLoopY = $dataMap.parallaxLoopY;
};

// 计算远景偏移
Game_Map.prototype.parallaxOx = function() {
    if (this._parallaxZero) {
        return this._parallaxX * this.tileWidth();
    }
    return this._parallaxX;
};
```

## 区域系统

除了瓦片层，还有区域层和碰撞层：

```javascript
// 获取区域ID
Game_Map.prototype.regionId = function(x, y) {
    return $dataMap.data[(5 * this.height() + y) * this.width() + x] || 0;
};

// 区域用途:
// - 遭遇群设置
// - 事件条件检测
// - 特殊地形效果
```

### 数据层索引

```
索引 0-3: 瓦片层 (Layer 0-3)
索引 4: 碰撞层 (编辑器中)
索引 5: 区域层 (Region ID)
```

## 循环地图

```javascript
// 水平循环
Game_Map.prototype.isLoopHorizontal = function() {
    return $dataMap.scrollType === 2 || $dataMap.scrollType === 3;
};

// 垂直循环
Game_Map.prototype.isLoopVertical = function() {
    return $dataMap.scrollType === 1 || $dataMap.scrollType === 3;
};

// 坐标取模
Game_Map.prototype.roundX = function(x) {
    if (this.isLoopHorizontal()) {
        return x.mod(this.width());
    }
    return x;
};
```

## 图层绘制示例

```
坐标 (5, 5) 的4层瓦片:

Layer 0: 2844 (草地 A2)
Layer 1: 0 (空)
Layer 2: 3142 (花 B层)
Layer 3: 0 (空)

渲染结果:
┌─────────────────┐
│    🌸 花朵      │  ← Layer 2 (上层)
├─────────────────┤
│ 🟩 草地         │  ← Layer 0 (下层)
└─────────────────┘

玩家站立时:
- 玩家与花朵在同一位置
- 玩家 Z=5 > 花朵 Z=4
- 玩家渲染在花朵前面
```
