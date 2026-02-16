# 瓦片与地图系统

## 概述

瓦片地图系统是 RPG Maker MV 的核心渲染组件，负责将地图数据渲染为游戏画面。本章将深入分析其实现原理。

## 核心概念

### 瓦片规格

```javascript
// rpg_core.js - Tilemap
Tilemap.prototype._tileWidth = 48;   // 瓦片宽度: 48像素
Tilemap.prototype._tileHeight = 48;  // 瓦片高度: 48像素
Tilemap.prototype._mapWidth = 0;     // 地图宽度 (瓦片数)
Tilemap.prototype._mapHeight = 0;    // 地图高度 (瓦片数)
```

### 瓦片ID系统

RPG Maker MV 使用统一的瓦片ID系统：

| ID范围 | 类型 | 说明 |
|--------|------|------|
| 0-255 | B-E瓦片 | 上层瓦片 (非自动瓦片) |
| 256-2047 | A1瓦片 | 动画水域瓦片 |
| 2048-2815 | A2瓦片 | 地面自动瓦片 |
| 2816-4351 | A3瓦片 | 屋顶瓦片 |
| 4352-5887 | A4瓦片 | 墙壁瓦片 |
| 5888-8191 | A5瓦片 | 静态下层瓦片 |

```javascript
// Tile ID 常量
Tilemap.TILE_ID_A1 = 2048;   // A1起始
Tilemap.TILE_ID_A2 = 2816;   // A2起始
Tilemap.TILE_ID_A3 = 4352;   // A3起始
Tilemap.TILE_ID_A4 = 5888;   // A4起始
Tilemap.TILE_ID_A5 = 8192;   // A5起始 (实际是 1536 在某些实现中)
```

## 图层系统

### 三层结构

```
┌─ 上层 (Level 4) ─────────────────────────────────┐
│  玩家站立其上的装饰物、树冠、屋顶等                  │
│  碰撞: 可穿透                                     │
├─ 角色层 (Level 1-3) ────────────────────────────┤
│  角色精灵、事件、交通工具                          │
│  Z坐标: 1=低优先级, 3=普通, 5=高优先级             │
├─ 下层 (Level 0) ────────────────────────────────┤
│  地面、墙壁、水域等基础地形                         │
│  碰撞: 根据瓦片属性决定                            │
└──────────────────────────────────────────────────┘
```

### 地图数据格式

每个地图位置最多存储4层瓦片：

```javascript
// Map001.json 中的 data 数组
// 格式: [x, y, layer] → data[x + y * width + layer * width * height]
"data": [
    0, 0, 0, 0, 0, 0, 0, 0,  // 位置 (0,0) 的4层
    2844, 2844, 2844, 2844,  // 位置 (0,1) 的4层 (A2自动瓦片)
    // ...
]
```

### 读取指定位置瓦片

```javascript
// rpg_objects.js - Game_Map
Game_Map.prototype.tileId = function(x, y, z) {
    var width = $dataMap.width;
    var height = $dataMap.height;
    return $dataMap.data[(z * height + y) * width + x];
};

// 获取某位置所有层瓦片
Game_Map.prototype.layeredTiles = function(x, y) {
    return [this.tileId(x, y, 0),
            this.tileId(x, y, 1),
            this.tileId(x, y, 2),
            this.tileId(x, y, 3)];
};
```

## Z坐标与渲染顺序

```javascript
// Z坐标定义
var z = {
    LOWER_TILES: 0,      // 下层瓦片
    LOWER_CHARACTERS: 1, // 低优先级角色
    NORMAL_CHARACTERS: 3,// 普通角色
    UPPER_TILES: 4,      // 上层瓦片
    UPPER_CHARACTERS: 5, // 高优先级角色
    AIRSHIP_SHADOW: 6,   // 飞船阴影
    BALLOON: 7,          // 气球图标
    ANIMATION: 8,        // 动画
    DESTINATION: 9       // 目的地指示
};
```

### 角色Z坐标计算

```javascript
Game_CharacterBase.prototype.screenZ = function() {
    // 基础值 + 优先级调整
    // _priorityType: 0=低, 1=普通, 2=高
    return this._priorityType * 2 + 3;
};

// 结合Y坐标实现伪深度排序
Sprite_Character.prototype.updatePosition = function() {
    this.x = this._character.screenX();
    this.y = this._character.screenY();
    this.z = this._character.screenZ();
    // PIXI.js 按 z 和 y 排序渲染
};
```

## 章节导航

1. **[Tilemap类详解](tilemap-class.md)** - 核心渲染类实现
2. **[自动瓦片系统](autotile.md)** - Autotile 原理与渲染
3. **[图层与深度](layers-depth.md)** - 图层管理和Z排序
4. **[瓦片集数据结构](tileset-data.md)** - Tilesets.json 格式
5. **[地图渲染流程](rendering-pipeline.md)** - 从数据到画面
6. **[ShaderTilemap加速](shader-tilemap.md)** - WebGL渲染优化
