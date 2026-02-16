# 瓦片集数据结构

## Tilesets.json 概述

瓦片集文件定义了地图使用的所有图形资源和属性。

### 基本结构

```json
[
    null,  // 索引0保留
    {
        "id": 1,
        "name": "World",
        "mode": 0,
        "flags": [0, 0, 16, 16, ...],
        "tilesetNames": [
            "World_A1",
            "World_A2", 
            "World_A3",
            "World_A4",
            "World_A5",
            "World_B",
            "World_C",
            "World_D",
            "World_E"
        ]
    }
]
```

## 字段详解

### id
瓦片集的唯一标识符，从1开始。

### name
瓦片集的显示名称，在编辑器中显示。

### mode
```
0: 世界地图模式 (Area Type)
1: 区域类型模式 (Field Type)
```

### flags
每个瓦片的属性标志数组，索引对应瓦片ID：

```javascript
// 8192+ 个标志值 (覆盖所有可能的瓦片ID)
"flags": [
    // 索引 0-255: B-E 瓦片
    0, 0, 16, 16, ...  // 0=普通, 16=☆(可通行)
    
    // 索引 2048-2815: A1 瓦片
    // ...
    
    // 索引 2816-4351: A2 瓦片
    // ...
    
    // 以此类推
]
```

### tilesetNames
9个图块文件名的数组：

| 索引 | 图块类型 | 说明 |
|------|---------|------|
| 0 | A1 | 动画水域 |
| 1 | A2 | 地面自动瓦片 |
| 2 | A3 | 屋顶 |
| 3 | A4 | 墙壁 |
| 4 | A5 | 静态下层 |
| 5 | B | 上层1 |
| 6 | C | 上层2 |
| 7 | D | 上层3 |
| 8 | E | 上层4 |

## 标志位详解

### 位定义

```javascript
// 32位标志的各部分含义
// 低4位: 特殊属性
// 高4位: 方向通行性
// 中间位: 其他属性

var FLAGS = {
    // 低4位 (0-3)
    BLOCK_LOWER:     0x0001,  // 位0: 遮挡下层
    BLOCK_UPPER:     0x0002,  // 位1: 遮挡上层
    LADDER:          0x0004,  // 位2: 梯子
    BUSH:            0x0008,  // 位3: 草丛
    
    // 中间位 (4-7)
    COUNTER:         0x0020,  // 位5: 柜台
    DAMAGE:          0x0100,  // 位8: 伤害地板
    
    // 高4位 (8-11): 方向通行
    // 位8: ↓ (下)
    // 位9: ← (左)
    // 位10: → (右)
    // 位11: ↑ (上)
    
    // 特殊标志
    STAR:            0x0010,  // ☆ 瓦片 (总是可通行)
    ALL_PASSABLE:    0x0F00,  // 所有方向可通行
    IMPASSABLE:      0x0000   // 所有方向不可通行
};
```

### 常见标志值

| 值 | 二进制 | 含义 |
|----|--------|------|
| 0x0000 | 00000000 | 完全不可通行 |
| 0x0010 | 00010000 | ☆ 可通行 (忽略下层) |
| 0x0001 | 00000001 | 上层瓦片，遮挡下层 |
| 0x010F | 000100001111 | 可通行 + 伤害地板 |
| 0x0F10 | 111100010000 | 四向可通行 + ☆ |
| 0x0200 | 001000000000 | 仅左可通行 |

### 方向通行计算

```javascript
// 检查某方向是否可通行
Game_Map.prototype.isPassable = function(x, y, d) {
    // d: 2=下, 4=左, 6=右, 8=上
    return this.checkPassage(x, y, (1 << (d / 2 - 1)) << 8);
};

// 方向到位掩码的映射
// d=2 (下) → bit=0 → mask=0x0100
// d=4 (左) → bit=1 → mask=0x0200
// d=6 (右) → bit=2 → mask=0x0400
// d=8 (上) → bit=3 → mask=0x0800
```

## 瓦片集文件格式

### A1 (动画水域)

```
┌─────────────────────────────────────┐
│ 宽度: 768px (16块×48px)              │
│ 高度: 576px (12块×48px)              │
│                                     │
│ [帧1][帧2][帧3][静态]×4              │
│                                     │
│ 每块: 192×96 (自动瓦片模板)           │
└─────────────────────────────────────┘
```

### A2 (地面)

```
┌─────────────────────────────────────┐
│ 宽度: 768px                          │
│ 高度: 576px                          │
│                                     │
│ [地面1][地面2]...[地面8]              │
│                                     │
│ 每个: 96×192 (基础+边缘过渡)          │
└─────────────────────────────────────┘
```

### A3 (屋顶)

```
┌─────────────────────────────────────┐
│ 宽度: 768px                          │
│ 高度: 384px                          │
│                                     │
│ [屋顶1][屋顶2]...                     │
│                                     │
│ 每个: 96×96                          │
└─────────────────────────────────────┘
```

### A4 (墙壁)

```
┌─────────────────────────────────────┐
│ 宽度: 768px                          │
│ 高度: 720px                          │
│                                     │
│ [墙顶][墙侧][墙顶][墙侧]...           │
│                                     │
│ 墙顶: 96×96, 墙侧: 96×144            │
└─────────────────────────────────────┘
```

### A5 (静态下层)

```
┌─────────────────────────────────────┐
│ 宽度: 384px                          │
│ 高度: 768px                          │
│                                     │
│ [瓦片1][瓦片2]... (普通网格排列)       │
│                                     │
│ 每瓦片: 48×48                        │
└─────────────────────────────────────┘
```

### B-E (上层)

```
┌─────────────────────────────────────┐
│ 宽度: 768px                          │
│ 高度: 768px                          │
│                                     │
│ [瓦片1][瓦片2]...                     │
│                                     │
│ 每瓦片: 48×48                        │
│ 16列×16行 = 256瓦片/文件              │
└─────────────────────────────────────┘
```

## 在代码中使用

### 加载瓦片集

```javascript
// Spriteset_Map.js
Spriteset_Map.prototype.loadTileset = function() {
    this._tileset = $gameMap.tileset();
    if (this._tileset) {
        var tilesetNames = this._tileset.tilesetNames;
        for (var i = 0; i < tilesetNames.length; i++) {
            this._tilemap.bitmaps[i] = ImageManager.loadTileset(tilesetNames[i]);
        }
        var newTilesetFlags = $gameMap.tilesetFlags();
        this._tilemap.refreshTileset();
        if (!this._tilemap.flags.equals(newTilesetFlags)) {
            this._tilemap.refresh();
        }
        this._tilemap.flags = newTilesetFlags;
    }
};
```

### 获取标志

```javascript
// Game_Map.js
Game_Map.prototype.tilesetFlags = function() {
    var tileset = this.tileset();
    if (tileset) {
        return tileset.flags;
    } else {
        return [];
    }
};
```

### 切换瓦片集

```javascript
// 事件指令: 更改瓦片集
Game_Map.prototype.changeTileset = function(tilesetId) {
    this._tilesetId = tilesetId;
    this.refresh();
};
```

## 项目示例

本项目 `Tilesets.json` 分析：

- 包含多个瓦片集配置
- 每个 map 通过 `tilesetId` 引用对应瓦片集
- `Map001.json` 使用默认瓦片集 (id=1)
