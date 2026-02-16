# 第三章：画布与坐标系统

## Canvas 是什么？

### 基本概念

Canvas（画布）是 HTML5 提供的一个元素，用于绑制图形。

```html
<canvas id="myCanvas" width="816" height="624"></canvas>
```

**Canvas 的特点**：
- 是一个矩形区域
- 用 JavaScript 控制绑图
- 支持 2D 和 WebGL 两种模式
- 像素级控制

### Canvas 就像一张画纸

想象你有一张画纸：
- 你可以用笔画线、画圆、画方块
- 画上去的东西就固定在那里了
- 想改变？只能擦掉重画

Canvas 也是一样：
- `ctx.fillRect()` = 画一个方块
- `ctx.drawImage()` = 贴一张图片
- 想让方块移动？清除画布，在新位置重画

---

## 2D 坐标系统

### 坐标原点在哪里？

```
(0,0) ──────────────────► X 轴
  │
  │    ┌──────────────┐
  │    │              │
  │    │   玩家位置    │
  │    │   (100, 50)  │
  │    │              │
  │    └──────────────┘
  │
  ▼
 Y 轴
```

**重要**：
- 原点 **(0, 0)** 在**左上角**
- **X 轴向右增加**
- **Y 轴向下增加**

这与数学课上的坐标系不同！数学课的原点在左下角，Y 轴向上。

### 为什么这样设计？

因为显示器是从上到下扫描的：
```
第 0 行像素
第 1 行像素
第 2 行像素
...
```

所以 Y=0 在最上面，Y 越大越靠下。

### 坐标单位

坐标单位是**像素**（Pixel）。

```javascript
// 在坐标 (100, 50) 画一个 32×32 的方块
ctx.fillRect(100, 50, 32, 32);

// 这个方块的四个角：
// 左上：(100, 50)
// 右上：(132, 50)  // 100 + 32
// 左下：(100, 82)  // 50 + 32
// 右下：(132, 82)
```

---

## 瓦片（Tile）概念

### 什么是瓦片？

想象你在铺地板：
- 地板砖是 30cm × 30cm 的正方形
- 你用很多块相同的地板砖，铺满整个房间
- 每块砖的位置可以计算：第几行、第几列

**游戏中的瓦片**：
- 瓦片是固定大小的图片块
- 常见大小：16×16、32×32、48×48 像素
- 用瓦片拼成大地图

### 为什么用瓦片？

**方法一：一整张大图**

假设地图是 100×100 个格子：
```
一张 3200×3200 像素的图片
（100 格 × 32 像素 = 3200）
```

问题：
- 文件太大（10+ MB）
- 修改一个小地方要重画整张图
- 内存占用高

**方法二：瓦片拼接**

```
草地瓦片（32×32）  草地瓦片         草地瓦片
草地瓦片           水面瓦片         草地瓦片
草地瓦片           草地瓦片         草地瓦片

只有几种瓦片，但可以拼成无限大的地图！
```

优点：
- 文件小（每种瓦片一个图）
- 修改方便（换一种瓦片，全部更新）
- 内存省（只需加载用到的瓦片）

### RPG Maker 的瓦片系统

RPG Maker MV 使用 **48×48** 像素的瓦片：

```
一个地图格子 = 48×48 像素

816 像素宽 ÷ 48 = 17 格
624 像素高 ÷ 48 = 13 格

默认屏幕显示 17×13 = 221 个瓦片
```

---

## 像素坐标 vs 瓦片坐标

### 两种坐标系统

| 坐标系统 | 单位 | 范围 | 用途 |
|---------|------|------|------|
| 像素坐标 | 像素 | 0~815, 0~623 | 渲染、碰撞检测 |
| 瓦片坐标 | 格子 | 0~16, 0~12 | 地图数据、寻路 |

### 相互转换

假设瓦片大小 = 48 像素：

```javascript
var TILE_SIZE = 48;

// 像素坐标 → 瓦片坐标
function pixelToTile(px, py) {
    return {
        x: Math.floor(px / TILE_SIZE),
        y: Math.floor(py / TILE_SIZE)
    };
}

// 瓦片坐标 → 像素坐标
function tileToPixel(tx, ty) {
    return {
        x: tx * TILE_SIZE,
        y: ty * TILE_SIZE
    };
}

// 示例
pixelToTile(100, 50);  // {x: 2, y: 1}  第2列第1行
tileToPixel(2, 1);     // {x: 96, y: 48}  像素位置
```

### 为什么需要两种坐标？

**像素坐标**：
- 渲染时用：`ctx.drawImage(img, 96, 48)`
- 精确碰撞：`if (player.x < enemy.x + 32)`
- 平滑移动：角色可以停在任意像素位置

**瓦片坐标**：
- 地图数据：`map[2][1] = 'grass'`
- 网格碰撞：`if (map[playerTileY+1][playerTileX] === 'wall')`
- 寻路算法：从格子 (2,1) 到 (10,8)

---

## 实战：画一个简单的地图

### 定义地图数据

```javascript
// 用数字代表不同的地形
// 0 = 草地, 1 = 墙壁, 2 = 水

var mapData = [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 0, 0, 1, 1, 1, 1, 0, 0, 1],
    [1, 0, 0, 1, 2, 2, 1, 0, 0, 1],
    [1, 0, 0, 1, 2, 2, 1, 0, 0, 1],
    [1, 0, 0, 1, 1, 1, 1, 0, 0, 1],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
];

// 这是一个 10×9 的地图
// 外围是墙壁，中间有个水池
```

### 定义颜色

```javascript
var tileColors = {
    0: '#7ec850',  // 草地 - 绿色
    1: '#8b7355',  // 墙壁 - 棕色
    2: '#4a90d9'   // 水 - 蓝色
};
```

### 渲染地图

```javascript
var TILE_SIZE = 48;  // 每个瓦片 48×48 像素

function renderMap() {
    for (var y = 0; y < mapData.length; y++) {
        for (var x = 0; x < mapData[y].length; x++) {
            // 获取这个位置的瓦片类型
            var tileType = mapData[y][x];

            // 计算像素坐标
            var px = x * TILE_SIZE;
            var py = y * TILE_SIZE;

            // 设置颜色
            ctx.fillStyle = tileColors[tileType];

            // 绑制方块
            ctx.fillRect(px, py, TILE_SIZE, TILE_SIZE);

            // 画边框（让格子更清晰）
            ctx.strokeStyle = 'rgba(0,0,0,0.2)';
            ctx.strokeRect(px, py, TILE_SIZE, TILE_SIZE);
        }
    }
}
```

### 完整代码

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>瓦片地图示例</title>
    <style>
        body {
            margin: 0;
            background: #1a1a2e;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        canvas {
            border: 2px solid #4a4a6a;
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas" width="480" height="432"></canvas>
    <script>
        var canvas = document.getElementById('gameCanvas');
        var ctx = canvas.getContext('2d');
        var TILE_SIZE = 48;

        // 地图数据
        var mapData = [
            [1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
            [1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            [1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            [1, 0, 0, 1, 1, 1, 1, 0, 0, 1],
            [1, 0, 0, 1, 2, 2, 1, 0, 0, 1],
            [1, 0, 0, 1, 2, 2, 1, 0, 0, 1],
            [1, 0, 0, 1, 1, 1, 1, 0, 0, 1],
            [1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
        ];

        // 颜色映射
        var tileColors = {
            0: '#7ec850',  // 草地
            1: '#8b7355',  // 墙壁
            2: '#4a90d9'   // 水
        };

        // 渲染地图
        function renderMap() {
            for (var y = 0; y < mapData.length; y++) {
                for (var x = 0; x < mapData[y].length; x++) {
                    var tileType = mapData[y][x];
                    var px = x * TILE_SIZE;
                    var py = y * TILE_SIZE;

                    ctx.fillStyle = tileColors[tileType];
                    ctx.fillRect(px, py, TILE_SIZE, TILE_SIZE);

                    ctx.strokeStyle = 'rgba(0,0,0,0.2)';
                    ctx.strokeRect(px, py, TILE_SIZE, TILE_SIZE);
                }
            }
        }

        // 玩家
        var player = {
            x: 2,  // 瓦片坐标
            y: 1,
            color: '#e94560'
        };

        // 渲染玩家
        function renderPlayer() {
            var px = player.x * TILE_SIZE;
            var py = player.y * TILE_SIZE;

            ctx.fillStyle = player.color;
            ctx.fillRect(px + 8, py + 8, TILE_SIZE - 16, TILE_SIZE - 16);
        }

        // 输入处理
        document.addEventListener('keydown', function(e) {
            var newX = player.x;
            var newY = player.y;

            switch(e.key) {
                case 'ArrowUp':
                case 'w':
                    newY--;
                    break;
                case 'ArrowDown':
                case 's':
                    newY++;
                    break;
                case 'ArrowLeft':
                case 'a':
                    newX--;
                    break;
                case 'ArrowRight':
                case 'd':
                    newX++;
                    break;
                default:
                    return;  // 不是方向键，不处理
            }

            // 检查能否移动
            if (newY >= 0 && newY < mapData.length &&
                newX >= 0 && newX < mapData[0].length) {

                var tileType = mapData[newY][newX];

                // 草地可以走，墙壁和水不能走
                if (tileType === 0) {
                    player.x = newX;
                    player.y = newY;
                }
            }
        });

        // 游戏循环
        function gameLoop() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            renderMap();
            renderPlayer();
            requestAnimationFrame(gameLoop);
        }

        gameLoop();
    </script>
</body>
</html>
```

---

## 本章小结

1. **Canvas** 是 HTML5 的绘图画布，用 JavaScript 控制
2. **坐标原点**在左上角，X 向右增加，Y 向下增加
3. **瓦片**是固定大小的图片块，用于拼接大地图
4. **像素坐标**用于渲染，**瓦片坐标**用于逻辑

## 练习

1. **扩大地图**：把地图改成 15×15
2. **添加新地形**：添加"沙地"（黄色），可以走但走得慢
3. **添加终点**：在地图某处添加一个"出口"，到达时弹出提示

## 下一章

[第四章：瓦片地图系统](./04-tilemap.md) - 实现更复杂的地图渲染
