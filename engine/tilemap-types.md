# 瓦片地图类型详解

## 概述

瓦片地图有多种不同的**投影方式**和**排列模式**，每种适合不同类型的游戏。

---

## 一、俯视角网格瓦片（Top-down Grid）

### 1.1 标准网格（Square Grid）

最常见的瓦片类型，RPG Maker 使用的就是这种。

```
俯视图：
┌────┬────┬────┐
│    │    │    │
├────┼────┼────┤
│    │ ★  │    │  ← 玩家
├────┼────┼────┤
│    │    │    │
└────┴────┴────┘

每个瓦片是正方形
```

**特点**：
- 瓦片是正方形（如 32×32、48×48）
- X/Y 方向移动距离相同
- 适合回合制 RPG、2D 像素游戏

**代表游戏**：
- RPG Maker 系列
- 《宝可梦》红绿蓝黄
- 《勇者斗恶龙》1-6

**RPG Maker 的特殊设计**：

RPG Maker 把瓦片分成多层：
```
A层（地面层）：
  - A1: 动画瓦片（水面、瀑布）
  - A2: 地面+边缘（草地+路径）
  - A3: 建筑（墙壁）
  - A4: 墙壁+天花板
  - A5: 静态地面

B-E层（装饰层）：
  - 放在地面上的物体
  - 树木、花草、家具等
```

### 1.2 自动瓦片（Auto-tile）

RPG Maker 的自动瓦片系统：

```
单个自动瓦片图片：
┌────────┬────────┐
│ 角落   │ 边缘   │  16×16 像素的子瓦片
├────────┼────────┤
│ 内角   │ 中心   │
└────────┴────────┘

系统会根据相邻瓦片自动选择正确的边缘
```

**原理**：检查 8 个相邻格子，计算 48 种可能的组合。

---

## 二、横版瓦片（Side-scrolling）

### 2.1 平台游戏瓦片

```
侧视图：
       ▓▓▓▓▓▓▓▓
       ▓       ▓
   ★   ▓       ▓    ← 玩家
███▓▓▓▓▓████████████ ← 地面
```

**特点**：
- 重力向下
- 只关注 X（水平）和 Y（垂直）两个方向
- 地面是"实心"的，不能穿透
- 常用于平台跳跃游戏

**代表游戏**：
- 《超级马里奥》系列
- 《空洞骑士》
- 《蔚蓝 Celeste》
- 《泰拉瑞亚》

**与 RPG 瓦片的区别**：

| RPG 瓦片 | 横版瓦片 |
|---------|---------|
| 四方向移动 | 左右移动 + 跳跃 |
| 碰撞基于格子 | 碰撞基于像素 |
| 地面可以穿透 | 地面是实心的 |
| 层次简单 | 需要前景/背景层 |

### 2.2 横版地图的数据结构

```javascript
// 与 RPG 类似，但需要额外处理"实心"属性
var mapData = [
    [0, 0, 0, 0, 0, 0, 0, 0],
    [0, 0, 0, 0, 0, 0, 0, 0],
    [0, 0, 0, 0, 0, 0, 0, 0],
    [1, 1, 1, 1, 1, 1, 1, 1],  // 1 = 实心地面
];

// 瓦片属性
var tileProperties = {
    0: { solid: false },  // 空气
    1: { solid: true },   // 地面
    2: { solid: true, breakable: true },  // 可破坏的砖块
    3: { solid: false, hazard: true }     // 尖刺（伤害）
};
```

---

## 三、等距瓦片（Isometric）

### 3.1 什么是等距投影？

等距投影是一种**伪 3D** 效果，用 2D 图形表现 3D 透视：

```
正交视图（RPG Maker）    等距视图
      ┌───┐               ◇
      │   │              ╱ ╲
      └───┘             ◇   ◇
                       ╱ ╲ ╱ ╲

瓦片形状：
      ╱ ╲
     ╱   ╲
    ╲     ╱
     ╲   ╱
      ╲ ╱
```

**瓦片尺寸比例**：宽:高 = 2:1（如 64×32）

### 3.2 坐标转换

```javascript
// 屏幕坐标 → 等距坐标
function screenToIso(screenX, screenY) {
    var tileWidth = 64;
    var tileHeight = 32;

    var isoX = (screenX / tileWidth + screenY / tileHeight) / 2;
    var isoY = (screenY / tileHeight - screenX / tileWidth) / 2;

    return { x: isoX, y: isoY };
}

// 等距坐标 → 屏幕坐标
function isoToScreen(isoX, isoY) {
    var tileWidth = 64;
    var tileHeight = 32;

    var screenX = (isoX - isoY) * (tileWidth / 2);
    var screenY = (isoX + isoY) * (tileHeight / 2);

    return { x: screenX, y: screenY };
}
```

### 3.3 渲染顺序

等距地图必须**从后到前、从上到下**渲染：

```
渲染顺序：
    1
   2 3
  4 5 6
 7 8 9 10

先画远处的，再画近处的，这样近处的会遮挡远处的
```

### 3.4 代表游戏

- 《模拟城市》早期版本
- 《暗黑破坏神 2》
- 《星际争霸 1》
- 《牧场物语》部分版本
- 《足球经理》系列

### 3.5 与 RPG Maker 的对比

| 特点 | RPG Maker（正交） | 等距 |
|------|-----------------|------|
| 瓦片形状 | 正方形 | 菱形 |
| 3D 感 | 无 | 有 |
| 实现难度 | 简单 | 中等 |
| 移动方向 | 4 方向 | 8 方向 |
| 遮挡处理 | 简单 | 需要深度排序 |

---

## 四、六边形瓦片（Hexagonal）

### 4.1 为什么用六边形？

六边形有 **6 个相邻格子**（正方形只有 4 个），移动更灵活：

```
正方形网格：           六边形网格：
  ┌─┐                  ◇
┌─┼─┐                ╱ ╲
│ │ │              ◇───◇
└─┼─┘                ╲ ╱
  └─┘                  ◇

4 个邻居              6 个邻居
```

**六边形的优势**：
1. 所有邻居距离相等（正方形对角线距离是 √2 倍）
2. 更适合策略游戏的移动范围计算
3. 看起来更"自然"

### 4.1 六边形的两种朝向

```
尖顶朝上（Pointy-top）    平顶朝上（Flat-top）
      ◇                     ┌───┐
     ╱ ╲                   ╱     ╲
    ◇   ◇                 ◇───────◇
     ╲ ╱                   ╲     ╱
      ◇                     └───┘
```

### 4.2 坐标系统

六边形有多种坐标系统：

```javascript
// 偏移坐标（Offset）- 最直观
var map = [
    [0, 1, 2, 3],     // 偶数行
      [4, 5, 6, 7],   // 奇数行（偏移半格）
    [8, 9, 10, 11],
];

// 立方坐标（Cube）- 计算最方便
// x + y + z = 0
function cubeDistance(a, b) {
    return Math.max(
        Math.abs(a.x - b.x),
        Math.abs(a.y - b.y),
        Math.abs(a.z - b.z)
    );
}

// 轴坐标（Axial）- 存储最省空间
// 只存储 q 和 r，z = -q - r
```

### 4.3 代表游戏

- 《文明》系列
- 《火焰纹章》部分版本
- 《陷阵之志》（Into the Breach）
- 《环形帝国》

---

## 五、其他特殊类型

### 5.1 体素瓦片（Voxel）

类似《我的世界》，但本质上是 3D 瓦片：

```
    ┌───┐
   ┌┴───┴┐
   │     │
   └─────┘

每个"瓦片"是一个 3D 立方体
```

**代表游戏**：
- 《我的世界》
- 《泰拉瑞亚》（2D 体素）
- 《星露谷物语》的某些元素

### 5.2 斜视角瓦片（Oblique）

比等距更简单的伪 3D：

```
每个瓦片是正方形，但物体有"侧面"
┌─────────┐
│  顶面   │
├─────────┤
│  侧面   │
└─────────┘
```

**代表游戏**：
- 早期《模拟城市》
- 《帝国时代 1》

### 5.3 混合视角

现代游戏经常混合多种视角：

- 《八方旅人》：2D 角色 + 3D 场景
- 《暗黑破坏神 3》：3D 渲染 + 等距视角
- 《歧路旅人》：HD-2D 技术

---

## 六、对比总结

| 类型 | 瓦片形状 | 邻居数 | 难度 | 适合游戏 |
|------|---------|--------|------|---------|
| 标准网格 | 正方形 | 4 | ★ | 回合制 RPG、宝可梦 |
| 横版 | 正方形 | - | ★★ | 平台跳跃 |
| 等距 | 菱形 | 8 | ★★★ | 模拟经营、ARPG |
| 六边形 | 六边形 | 6 | ★★★ | 策略战棋 |
| 体素 | 立方体 | 26 | ★★★★ | 沙盒建造 |

---

## 七、如何选择？

### 根据游戏类型

```
回合制 RPG → 标准网格（RPG Maker 风格）
平台跳跃   → 横版瓦片
策略战棋   → 六边形或标准网格
模拟经营   → 等距瓦片
沙盒建造   → 体素
```

### 根据团队规模

```
个人/小团队 → 标准网格（最简单）
中型团队   → 等距或六边形
大型团队   → 3D 或混合
```

### 根据美术资源

```
像素美术     → 标准网格/横版
手绘美术     → 等距
3D 模型     → 3D 场景 + 2D 瓦片逻辑
```

---

## 八、代码实现对比

### 8.1 标准网格（RPG Maker 风格）

```javascript
// 渲染
for (var y = 0; y < mapHeight; y++) {
    for (var x = 0; x < mapWidth; x++) {
        var screenX = x * tileSize;
        var screenY = y * tileSize;
        drawTile(map[y][x], screenX, screenY);
    }
}

// 移动检查
function canMove(tileX, tileY) {
    return map[tileY][tileX] !== WALL;
}
```

### 8.2 等距瓦片

```javascript
// 渲染（必须按深度顺序）
var tiles = [];
for (var y = 0; y < mapHeight; y++) {
    for (var x = 0; x < mapWidth; x++) {
        tiles.push({
            x: x, y: y,
            depth: x + y  // 深度用于排序
        });
    }
}
tiles.sort((a, b) => a.depth - b.depth);

for (var tile of tiles) {
    var screenX = (tile.x - tile.y) * (tileWidth / 2);
    var screenY = (tile.x + tile.y) * (tileHeight / 2);
    drawTile(map[tile.y][tile.x], screenX, screenY);
}

// 移动检查
function canMove(isoX, isoY) {
    return map[isoY][isoX] !== WALL;
}
```

### 8.3 六边形瓦片

```javascript
// 尖顶朝上的渲染
for (var r = 0; r < mapHeight; r++) {
    for (var q = 0; q < mapWidth; q++) {
        var screenX = tileSize * (3/2 * q);
        var screenY = tileSize * (Math.sqrt(3)/2 * q + Math.sqrt(3) * r);
        if (q % 2 === 1) screenY += tileSize * Math.sqrt(3) / 2;
        drawHexTile(map[r][q], screenX, screenY);
    }
}

// 获取邻居
function getNeighbors(q, r) {
    var directions = [
        [1, 0], [1, -1], [0, -1],
        [-1, 0], [-1, 1], [0, 1]
    ];
    var neighbors = [];
    for (var d of directions) {
        neighbors.push({ q: q + d[0], r: r + d[1] });
    }
    return neighbors;
}
```

---

## 参考资料

- [Red Blob Games - Hexagonal Grids](https://www.redblobgames.com/grids/hexagons/)
- [Isometric Tiles Tutorial](https://clintbellanger.net/articles/isometric_math/)
- [Tiled Map Editor Documentation](https://doc.mapeditor.org/)
