# 第二章：游戏主循环

## 游戏是怎么"动"起来的？

### 电影 vs 游戏

**电影**：
- 每秒 24 张图片（24 FPS）
- 图片是提前拍好的
- 观众只能看，不能改

**游戏**：
- 每秒 60 张图片（60 FPS）
- 图片是实时计算出来的
- 玩家的输入会影响图片内容

### 游戏的本质

游戏就是：**不断地计算 → 画图 → 计算 → 画图 → ...**

```
帧 1: 计算玩家位置(100,100) → 画图
帧 2: 计算玩家位置(103,100) → 画图  （玩家按了右键）
帧 3: 计算玩家位置(106,100) → 画图
帧 4: 计算玩家位置(106,100) → 画图  （玩家松开了右键）
...
```

每一张图片叫做一**帧**（Frame）。

---

## 什么是游戏主循环？

游戏主循环（Game Loop）是游戏的**心脏**。

它不断重复三件事：
1. **处理输入**：玩家按了什么键？
2. **更新状态**：根据输入，更新游戏世界
3. **渲染画面**：把更新后的状态画出来

```
        ┌──────────────────────────────┐
        │                              │
        │     ┌──────────────────┐     │
        │     │   处理输入       │     │
        │     │  (Input)         │     │
        │     └────────┬─────────┘     │
        │              │               │
        │              ▼               │
        │     ┌──────────────────┐     │
        │     │   更新状态       │     │
        │     │  (Update)        │     │
        │     └────────┬─────────┘     │
        │              │               │
        │              ▼               │
        │     ┌──────────────────┐     │
        │     │   渲染画面       │     │
        │     │  (Render)        │     │
        │     └────────┬─────────┘     │
        │              │               │
        │              └───────────────┤
        │                    重复      │
        │                              │
        └──────────────────────────────┘
```

---

## 为什么要 60 FPS？

### 人眼的限制

- 低于 24 FPS：感觉卡顿
- 24-30 FPS：能接受，但不够流畅
- 60 FPS：流畅舒适
- 高于 60 FPS：大多数人不觉得有区别

### 为什么游戏要 60 FPS？

1. **流畅感**：角色移动、动画更顺滑
2. **响应快**：操作延迟更低
3. **习惯**：大多数显示器是 60Hz（每秒刷新 60 次）

### RPG 不需要 60 FPS？

其实 30 FPS 对回合制 RPG 也够用。

但 60 FPS 有好处：
- 战斗动画更流畅
- UI 交互更顺滑
- 统一标准，减少问题

---

## 实现第一个游戏循环

### 错误示范：while 循环

```javascript
// ❌ 错误！这会卡死浏览器
while (true) {
    update();
    render();
}
```

**为什么错了？**
- JavaScript 是单线程的
- `while(true)` 会一直执行，不让浏览器做其他事
- 浏览器无法响应用户操作，页面卡死

### 正确做法：requestAnimationFrame

```javascript
// ✅ 正确！使用 requestAnimationFrame
function gameLoop() {
    update();   // 更新游戏状态
    render();   // 渲染画面

    requestAnimationFrame(gameLoop);  // 下一帧继续
}

gameLoop();  // 启动游戏循环
```

**requestAnimationFrame 是什么？**
- 浏览器提供的 API
- 告诉浏览器："下一帧请调用这个函数"
- 通常每秒调用 60 次（与显示器刷新率同步）
- 浏览器标签页不可见时自动暂停，节省资源

---

## 完整示例：让方块移动起来

### HTML 文件

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>第一个游戏循环</title>
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
    <canvas id="gameCanvas" width="400" height="300"></canvas>
    <script src="game.js"></script>
</body>
</html>
```

### JavaScript 文件 (game.js)

```javascript
// ============================================
// 第一步：获取画布和上下文
// ============================================
var canvas = document.getElementById('gameCanvas');
var ctx = canvas.getContext('2d');

// 什么是上下文？
// - Canvas 有两种上下文：2d 和 webgl
// - 2d 上下文提供绑图方法（fillRect、drawImage 等）
// - 我们先用 2d，后面再说 WebGL

// ============================================
// 第二步：定义游戏状态
// ============================================
var gameState = {
    player: {
        x: 200,      // 玩家 X 坐标
        y: 150,      // 玩家 Y 坐标
        width: 32,   // 宽度
        height: 32,  // 高度
        speed: 5,    // 移动速度（每帧移动的像素）
        color: '#e94560'  // 颜色
    },
    input: {
        up: false,
        down: false,
        left: false,
        right: false
    }
};

// 为什么要用对象存储状态？
// - 方便管理：所有数据都在一个地方
// - 方便扩展：随时可以添加新属性
// - 方便调试：console.log(gameState) 看到所有状态

// ============================================
// 第三步：处理输入
// ============================================
function setupInput() {
    // 按键按下
    document.addEventListener('keydown', function(event) {
        switch(event.key) {
            case 'ArrowUp':
            case 'w':
            case 'W':
                gameState.input.up = true;
                break;
            case 'ArrowDown':
            case 's':
            case 'S':
                gameState.input.down = true;
                break;
            case 'ArrowLeft':
            case 'a':
            case 'A':
                gameState.input.left = true;
                break;
            case 'ArrowRight':
            case 'd':
            case 'D':
                gameState.input.right = true;
                break;
        }
    });

    // 按键释放
    document.addEventListener('keyup', function(event) {
        switch(event.key) {
            case 'ArrowUp':
            case 'w':
            case 'W':
                gameState.input.up = false;
                break;
            case 'ArrowDown':
            case 's':
            case 'S':
                gameState.input.down = false;
                break;
            case 'ArrowLeft':
            case 'a':
            case 'A':
                gameState.input.left = false;
                break;
            case 'ArrowRight':
            case 'd':
            case 'D':
                gameState.input.right = false;
                break;
        }
    });
}

// 为什么同时支持方向键和 WASD？
// - 方向键：方便右手操作
// - WASD：方便左手操作（游戏玩家习惯）
// - 多一种选择，体验更好

// ============================================
// 第四步：更新游戏状态
// ============================================
function update() {
    var player = gameState.player;
    var input = gameState.input;

    // 根据输入更新玩家位置
    if (input.up) {
        player.y -= player.speed;
    }
    if (input.down) {
        player.y += player.speed;
    }
    if (input.left) {
        player.x -= player.speed;
    }
    if (input.right) {
        player.x += player.speed;
    }

    // 边界检测（防止玩家跑出屏幕）
    if (player.x < 0) {
        player.x = 0;
    }
    if (player.x > canvas.width - player.width) {
        player.x = canvas.width - player.width;
    }
    if (player.y < 0) {
        player.y = 0;
    }
    if (player.y > canvas.height - player.height) {
        player.y = canvas.height - player.height;
    }

    // 为什么用 if 而不是 else if？
    // - 用 if，可以同时按两个键（斜向移动）
    // - 用 else if，一次只能响应一个方向
}

// ============================================
// 第五步：渲染画面
// ============================================
function render() {
    var player = gameState.player;

    // 1. 清除整个画布
    ctx.fillStyle = '#16213e';  // 背景色
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // 为什么要清除？
    // - Canvas 是画布，新内容会叠加在旧内容上
    // - 不清除的话，会看到"残影"
    // - 每帧都重新画，就像翻书动画

    // 2. 绘制玩家
    ctx.fillStyle = player.color;
    ctx.fillRect(player.x, player.y, player.width, player.height);

    // fillRect 参数：
    // - x, y: 左上角坐标
    // - width, height: 宽度和高度
}

// ============================================
// 第六步：游戏主循环
// ============================================
function gameLoop() {
    update();   // 更新
    render();   // 渲染

    requestAnimationFrame(gameLoop);  // 下一帧
}

// ============================================
// 第七步：启动游戏
// ============================================
setupInput();  // 先设置输入监听
gameLoop();    // 启动游戏循环

console.log('游戏启动！用方向键或 WASD 移动方块。');
```

---

## 理解代码执行流程

```
时间线（假设每帧 16.67ms，即 60 FPS）：

0ms:     gameLoop() 开始
         ├─ update() 执行（玩家位置：200, 150）
         │   └─ 没有按键，位置不变
         ├─ render() 执行
         │   └─ 画布清除，画一个红色方块在 (200, 150)
         └─ requestAnimationFrame(gameLoop)

16ms:    gameLoop() 再次执行
         ├─ update() 执行
         │   └─ 假设玩家按了右键，位置变成 (205, 150)
         ├─ render() 执行
         │   └─ 画布清除，画方块在 (205, 150)
         └─ requestAnimationFrame(gameLoop)

33ms:    gameLoop() 再次执行
         ...
```

**每 16.67ms（约）执行一次完整循环**。

---

## 添加帧率显示

为了验证我们的游戏确实是 60 FPS，可以添加帧率显示：

```javascript
// 在 gameState 中添加帧率统计
gameState.fps = {
    frames: 0,        // 帧计数
    lastTime: 0,      // 上次记录时间
    value: 0          // 当前 FPS 值
};

// 修改 render 函数
function render() {
    // ... 原有绑图代码 ...

    // 绘制 FPS
    ctx.fillStyle = '#ffffff';
    ctx.font = '14px Arial';
    ctx.fillText('FPS: ' + gameState.fps.value, 10, 20);
}

// 修改 gameLoop 函数
function gameLoop(timestamp) {
    // 计算 FPS
    gameState.fps.frames++;
    if (timestamp - gameState.fps.lastTime >= 1000) {
        gameState.fps.value = gameState.fps.frames;
        gameState.fps.frames = 0;
        gameState.fps.lastTime = timestamp;
    }

    update();
    render();

    requestAnimationFrame(gameLoop);
}

// 启动时传入时间戳
requestAnimationFrame(gameLoop);
```

---

## 时间增量（Delta Time）

### 问题：不同电脑速度不同

假设你的电脑是 60 FPS，朋友的电脑是 30 FPS：

| 你的电脑 | 朋友的电脑 |
|---------|-----------|
| 每秒执行 60 次 update | 每秒执行 30 次 update |
| 移动速度 = 5×60 = 300 像素/秒 | 移动速度 = 5×30 = 150 像素/秒 |

**问题**：朋友看到的游戏慢了一半！

### 解决方案：基于时间的移动

不写"每帧移动 5 像素"，而是写"每秒移动 300 像素"：

```javascript
function update(deltaTime) {
    // deltaTime = 距离上一帧的时间（秒）
    // 如果是 60 FPS，deltaTime ≈ 0.0167 秒

    var player = gameState.player;
    var input = gameState.input;
    var speed = 300;  // 每秒移动 300 像素

    // 移动距离 = 速度 × 时间
    if (input.up) {
        player.y -= speed * deltaTime;
    }
    if (input.down) {
        player.y += speed * deltaTime;
    }
    // ...

    // 这样，无论多少 FPS，每秒移动的距离都一样
}
```

### 完整的 deltaTime 实现

```javascript
var lastTime = 0;

function gameLoop(timestamp) {
    // 计算时间增量
    var deltaTime = (timestamp - lastTime) / 1000;  // 转换为秒
    lastTime = timestamp;

    // 防止 deltaTime 过大（比如切换标签页后回来）
    if (deltaTime > 0.1) {
        deltaTime = 0.1;
    }

    update(deltaTime);
    render();

    requestAnimationFrame(gameLoop);
}
```

---

## 本章小结

1. **游戏循环**是游戏的心脏：不断执行"输入→更新→渲染"
2. **60 FPS** 是标准帧率，使用 `requestAnimationFrame` 实现
3. **状态分离**：输入状态和游戏状态要分开
4. **deltaTime** 让游戏在不同帧率下表现一致

## 练习

1. **修改颜色**：让方块变成你喜欢的颜色
2. **调整速度**：让方块移动得更快或更慢
3. **添加第二个方块**：在屏幕上再画一个方块，它不移动
4. **边界反弹**：当方块碰到边界时，不是停止，而是反弹

## 下一章

[第三章：画布与坐标系统](./03-canvas-coordinates.md) - 理解 2D 游戏的坐标系统和瓦片概念
