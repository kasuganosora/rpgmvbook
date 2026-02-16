# 第五章：精灵与动画

## 什么是精灵（Sprite）？

### 定义

**精灵**是游戏中可移动、可交互的图像元素。

- 玩家角色是精灵
- NPC 是精灵
- 怪物是精灵
- 飘落的树叶也可以是精灵

### 精灵 vs 静态图片

| 静态图片 | 精灵 |
|---------|------|
| 固定位置 | 可以移动 |
| 单帧 | 可以播放动画 |
| 无状态 | 有状态（站立/行走/攻击） |

---

## 精灵图集（Sprite Sheet）

### 结构

把角色的所有动画帧放在一张图里：

```
┌─────────┬─────────┬─────────┬─────────┐
│ 向下站 │ 向下走1 │ 向下走2 │ 向下走3 │  行0
├─────────┼─────────┼─────────┼─────────┤
│ 向左站 │ 向左走1 │ 向左走2 │ 向左走3 │  行1
├─────────┼─────────┼─────────┼─────────┤
│ 向右站 │ 向右走1 │ 向右走2 │ 向右走3 │  行2
├─────────┼─────────┼─────────┼─────────┤
│ 向上站 │ 向上走1 │ 向上走2 │ 向上走3 │  行3
└─────────┴─────────┴─────────┴─────────┘

每帧 32×48 像素
行代表方向，列代表动画帧
```

### 方向常量

```javascript
var Direction = {
    DOWN:  0,
    LEFT:  1,
    RIGHT: 2,
    UP:    3
};
```

---

## 帧动画原理

### 什么是帧动画？

帧动画就是快速切换不同的图片：

```
时间 0: 显示帧0（站立）
时间 1: 显示帧1（左脚前）
时间 2: 显示帧2（右脚前）
时间 3: 显示帧1（左脚前）
时间 4: 显示帧0（站立）
...

循环播放 → 看起来在走路
```

### 动画速度

```javascript
var animation = {
    frames: [0, 1, 2, 1],  // 帧序列
    speed: 10,             // 每帧持续 10 个游戏帧
    currentFrame: 0,       // 当前帧索引
    counter: 0             // 计数器
};

function updateAnimation() {
    animation.counter++;
    if (animation.counter >= animation.speed) {
        animation.counter = 0;
        animation.currentFrame = (animation.currentFrame + 1) % animation.frames.length;
    }
}
```

---

## 实现角色精灵

```javascript
// ============================================
// 角色精灵类
// ============================================

function Character(spriteSheet, x, y) {
    this.spriteSheet = spriteSheet;  // 精灵图集
    this.x = x;                       // 位置 X
    this.y = y;                       // 位置 Y
    this.width = 32;                  // 宽度
    this.height = 48;                 // 高度
    this.speed = 3;                   // 移动速度

    this.direction = Direction.DOWN;  // 朝向
    this.isMoving = false;            // 是否在移动

    // 动画相关
    this.animFrame = 0;       // 当前动画帧
    this.animCounter = 0;     // 动画计数器
    this.animSpeed = 8;       // 动画速度（数字越小越快）
}

// 更新
Character.prototype.update = function(input) {
    var dx = 0, dy = 0;

    // 根据输入计算移动方向
    if (input.up) { dy = -1; this.direction = Direction.UP; }
    if (input.down) { dy = 1; this.direction = Direction.DOWN; }
    if (input.left) { dx = -1; this.direction = Direction.LEFT; }
    if (input.right) { dx = 1; this.direction = Direction.RIGHT; }

    // 移动
    this.isMoving = (dx !== 0 || dy !== 0);
    if (this.isMoving) {
        this.x += dx * this.speed;
        this.y += dy * this.speed;

        // 更新动画
        this.animCounter++;
        if (this.animCounter >= this.animSpeed) {
            this.animCounter = 0;
            this.animFrame = (this.animFrame + 1) % 4;  // 4帧循环
        }
    } else {
        this.animFrame = 0;  // 站立时显示第一帧
    }
};

// 渲染
Character.prototype.render = function(ctx, cameraX, cameraY) {
    // 计算精灵在图集中的位置
    var srcX = this.animFrame * this.width;
    var srcY = this.direction * this.height;

    // 计算画布上的位置（减去相机偏移）
    var dstX = this.x - cameraX - this.width / 2;
    var dstY = this.y - cameraY - this.height / 2;

    // 绘制
    ctx.drawImage(
        this.spriteSheet,
        srcX, srcY, this.width, this.height,
        dstX, dstY, this.width, this.height
    );
};
```

---

## 本章小结

1. **精灵**是可移动、可动画的图像元素
2. **精灵图集**把所有动画帧放在一张图里
3. **帧动画**通过快速切换图片实现

## 下一章

[第六章：游戏对象系统](./06-game-objects.md) - 统一管理所有游戏对象
