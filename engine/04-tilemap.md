# 第四章：瓦片地图系统

## 本章目标

上一章我们用颜色方块画了一个简单地图。这一章，我们要：
1. 使用真实的图片瓦片
2. 支持多层地图（地面层、装饰层）
3. 实现地图滚动（地图比屏幕大）

---

## 瓦片集（Tileset）

### 什么是瓦片集？

瓦片集是把多个瓦片放在一张大图里：

```
┌────────┬────────┬────────┬────────┐
│ 草地 1 │ 草地 2 │ 花朵   │ 树桩   │  第0行
├────────┼────────┼────────┼────────┤
│ 水面 1 │ 水面 2 │ 水岸   │ 水岸   │  第1行
├────────┼────────┼────────┼────────┤
│ 墙壁   │ 地板   │ 楼梯   │ 门     │  第2行
├────────┼────────┼────────┼────────┤
│ 空白   │ 空白   │ 空白   │ 空白   │  第3行
└────────┴────────┴────────┴────────┘

每个瓦片 32×32 像素
整张图 128×128 像素（4×4 瓦片）
```

### 为什么用瓦片集？

1. **减少文件**：一个文件包含所有瓦片，不用加载很多小图
2. **减少请求**：浏览器一次加载完成
3. **方便管理**：美术只需维护一张图

### 瓦片索引

每个瓦片有一个索引号：

```
┌────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │
├────┼────┼────┼────┤
│ 4  │ 5  │ 6  │ 7  │
├────┼────┼────┼────┤
│ 8  │ 9  │ 10 │ 11 │
├────┼────┼────┼────┤
│ 12 │ 13 │ 14 │ 15 │
└────┴────┴────┴────┘

索引 0 = 左上角的草地
索引 5 = 水面
索引 8 = 墙壁
```

### 从索引计算位置

```javascript
var TILE_SIZE = 32;     // 每个瓦片 32×32
var TILESET_COLS = 4;   // 瓦片集每行 4 个

function getTilePosition(index) {
    // 列 = 索引 % 列数
    var col = index % TILESET_COLS;

    // 行 = Math.floor(索引 / 列数)
    var row = Math.floor(index / TILESET_COLS);

    // 像素位置
    return {
        x: col * TILE_SIZE,
        y: row * TILE_SIZE
    };
}

// 示例：获取索引 5 的位置
// col = 5 % 4 = 1
// row = Math.floor(5 / 4) = 1
// 位置 = (32, 32)，即第1列第1行
```

---

## 多层地图

### 为什么需要多层？

考虑这个场景：
- 地面是草地
- 草地上有花朵（装饰）
- 角色站在花朵旁边

如果只有一层：
- 角色在花朵"上面"走过，花朵被遮挡
- 或者角色在花朵"下面"走过，角色被遮挡

**多层方案**：
```
第2层（顶层）：角色
第1层（装饰）：花朵、树木
第0层（地面）：草地、水面

渲染顺序：从下到上
地面 → 装饰 → 角色
```

### 地图数据结构

```javascript
var mapData = {
    width: 20,      // 地图宽度（格）
    height: 15,     // 地图高度（格）
    layers: [
        // 第0层：地面
        {
            name: 'ground',
            data: [
                [8, 8, 8, 8, 8, ...],  // 墙壁
                [8, 9, 9, 9, 9, ...],  // 墙壁、地板...
                ...
            ]
        },
        // 第1层：装饰
        {
            name: 'decoration',
            data: [
                [-1, -1, -1, -1, ...],  // -1 表示空
                [-1, 2, -1, -1, ...],   // 位置 (1,1) 有花朵
                ...
            ]
        }
    ]
};
```

### 渲染多层地图

```javascript
function renderMap() {
    // 遍历所有层
    for (var l = 0; l < mapData.layers.length; l++) {
        var layer = mapData.layers[l];

        // 遍历所有格子
        for (var y = 0; y < layer.data.length; y++) {
            for (var x = 0; x < layer.data[y].length; x++) {
                var tileIndex = layer.data[y][x];

                // -1 表示空，跳过
                if (tileIndex < 0) continue;

                // 计算瓦片在瓦片集中的位置
                var srcPos = getTilePosition(tileIndex);

                // 计算画布上的位置
                var dstX = x * TILE_SIZE;
                var dstY = y * TILE_SIZE;

                // 从瓦片集绘制到画布
                ctx.drawImage(
                    tilesetImage,      // 瓦片集图片
                    srcPos.x, srcPos.y, // 源位置
                    TILE_SIZE, TILE_SIZE, // 源大小
                    dstX, dstY,        // 目标位置
                    TILE_SIZE, TILE_SIZE  // 目标大小
                );
            }
        }
    }
}
```

---

## 地图滚动

### 问题：地图比屏幕大

假设：
- 屏幕大小：816×624（17×13 格）
- 地图大小：1920×1440（40×30 格）

玩家走到地图边缘时，地图需要滚动。

### 视口（Viewport）

视口是"摄像机能看到的区域"：

```
┌─────────────────────────────────────┐
│             整个地图                 │
│    1920×1440 像素                   │
│                                     │
│      ┌──────────────┐               │
│      │    视口      │               │
│      │  816×624    │               │
│      │  (玩家位置)  │               │
│      └──────────────┘               │
│                                     │
└─────────────────────────────────────┘
```

### 视口坐标

```javascript
var camera = {
    x: 0,  // 视口左上角在地图中的 X 坐标
    y: 0   // 视口左上角在地图中的 Y 坐标
};

// 更新相机位置（跟随玩家）
function updateCamera() {
    // 让玩家始终在屏幕中心
    camera.x = player.x - canvas.width / 2;
    camera.y = player.y - canvas.height / 2;

    // 限制视口不超出地图边界
    if (camera.x < 0) camera.x = 0;
    if (camera.y < 0) camera.y = 0;
    if (camera.x > mapWidth - canvas.width) {
        camera.x = mapWidth - canvas.width;
    }
    if (camera.y > mapHeight - canvas.height) {
        camera.y = mapHeight - canvas.height;
    }
}
```

### 偏移绘制

```javascript
function renderMap() {
    for (var l = 0; l < mapData.layers.length; l++) {
        var layer = mapData.layers[l];

        for (var y = 0; y < layer.data.length; y++) {
            for (var x = 0; x < layer.data[y].length; x++) {
                var tileIndex = layer.data[y][x];
                if (tileIndex < 0) continue;

                var srcPos = getTilePosition(tileIndex);

                // 减去相机偏移
                var dstX = x * TILE_SIZE - camera.x;
                var dstY = y * TILE_SIZE - camera.y;

                // 跳过视口外的瓦片（优化性能）
                if (dstX < -TILE_SIZE || dstX > canvas.width ||
                    dstY < -TILE_SIZE || dstY > canvas.height) {
                    continue;
                }

                ctx.drawImage(
                    tilesetImage,
                    srcPos.x, srcPos.y,
                    TILE_SIZE, TILE_SIZE,
                    dstX, dstY,
                    TILE_SIZE, TILE_SIZE
                );
            }
        }
    }
}
```

---

## 完整示例

```javascript
// ============================================
// 瓦片地图系统
// ============================================

var Tilemap = {
    tileset: null,     // 瓦片集图片
    tileSize: 32,      // 瓦片大小
    tilesetCols: 8,    // 瓦片集列数

    mapData: null,     // 地图数据
    mapWidth: 0,       // 地图像素宽度
    mapHeight: 0,      // 地图像素高度

    cameraX: 0,        // 相机 X
    cameraY: 0,        // 相机 Y

    // 初始化
    init: function(tilesetSrc, mapData) {
        this.mapData = mapData;
        this.mapWidth = mapData.width * this.tileSize;
        this.mapHeight = mapData.height * this.tileSize;

        // 加载瓦片集
        this.tileset = new Image();
        this.tileset.src = tilesetSrc;

        return new Promise((resolve) => {
            this.tileset.onload = resolve;
        });
    },

    // 更新相机
    updateCamera: function(targetX, targetY, canvasWidth, canvasHeight) {
        this.cameraX = targetX - canvasWidth / 2;
        this.cameraY = targetY - canvasHeight / 2;

        // 边界限制
        this.cameraX = Math.max(0, Math.min(this.cameraX, this.mapWidth - canvasWidth));
        this.cameraY = Math.max(0, Math.min(this.cameraY, this.mapHeight - canvasHeight));
    },

    // 渲染
    render: function(ctx, canvasWidth, canvasHeight) {
        // 计算可见范围
        var startX = Math.floor(this.cameraX / this.tileSize);
        var startY = Math.floor(this.cameraY / this.tileSize);
        var endX = startX + Math.ceil(canvasWidth / this.tileSize) + 1;
        var endY = startY + Math.ceil(canvasHeight / this.tileSize) + 1;

        // 遍历所有层
        for (var l = 0; l < this.mapData.layers.length; l++) {
            var layer = this.mapData.layers[l];

            // 只绘制可见区域
            for (var y = startY; y < endY && y < layer.data.length; y++) {
                for (var x = startX; x < endX && x < layer.data[y].length; x++) {
                    if (y < 0 || x < 0) continue;

                    var tileIndex = layer.data[y][x];
                    if (tileIndex < 0) continue;

                    // 瓦片在瓦片集中的位置
                    var srcX = (tileIndex % this.tilesetCols) * this.tileSize;
                    var srcY = Math.floor(tileIndex / this.tilesetCols) * this.tileSize;

                    // 画布上的位置（减去相机偏移）
                    var dstX = x * this.tileSize - this.cameraX;
                    var dstY = y * this.tileSize - this.cameraY;

                    ctx.drawImage(
                        this.tileset,
                        srcX, srcY, this.tileSize, this.tileSize,
                        dstX, dstY, this.tileSize, this.tileSize
                    );
                }
            }
        }
    },

    // 获取某个位置的瓦片
    getTile: function(x, y, layerIndex) {
        var tileX = Math.floor(x / this.tileSize);
        var tileY = Math.floor(y / this.tileSize);

        if (tileY < 0 || tileY >= this.mapData.height ||
            tileX < 0 || tileX >= this.mapData.width) {
            return -1;
        }

        return this.mapData.layers[layerIndex].data[tileY][tileX];
    }
};
```

---

## 本章小结

1. **瓦片集**把多个瓦片放在一张图里，用索引访问
2. **多层地图**让装饰和地面分开，方便遮挡处理
3. **视口**控制显示地图的哪个区域
4. **只渲染可见区域**是重要的性能优化

## 练习

1. **创建自定义瓦片集**：用画图软件画一个简单的瓦片集
2. **添加第三层**：实现"飞行层"（云朵、鸟）
3. **相机平滑跟随**：让相机不是瞬间跟随，而是慢慢移动

## 下一章

[第五章：精灵与动画](./05-sprite-animation.md) - 让角色动起来
