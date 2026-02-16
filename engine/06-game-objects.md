# 第六章：游戏对象系统

## 什么是游戏对象？

在 RPG 游戏中，几乎所有的"东西"都是游戏对象：

- **玩家**：可以控制的角色
- **NPC**：可以对话的角色
- **物品**：可以捡起的东西
- **门**：可以打开的
- **事件点**：触发剧情的

它们的共同点：
1. 都有位置（x, y）
2. 都需要更新（每帧做什么）
3. 都需要渲染（怎么画出来）

---

## 基础实体类

```javascript
// ============================================
// Entity - 所有游戏对象的基类
// ============================================

function Entity(x, y) {
    this.x = x;
    this.y = y;
    this.width = 32;
    this.height = 32;
    this.active = true;  // 是否活跃
}

// 更新（子类重写）
Entity.prototype.update = function() {
    // 基类不做任何事
};

// 渲染（子类重写）
Entity.prototype.render = function(ctx, cameraX, cameraY) {
    // 基类不做任何事
};

// 获取边界矩形（用于碰撞检测）
Entity.prototype.getBounds = function() {
    return {
        x: this.x,
        y: this.y,
        width: this.width,
        height: this.height
    };
};
```

---

## 具体对象实现

### 玩家

```javascript
function Player(x, y) {
    Entity.call(this, x, y);  // 调用父类构造函数
    this.speed = 3;
    this.direction = 0;
}
Player.prototype = Object.create(Entity.prototype);

Player.prototype.update = function(input, map) {
    var dx = 0, dy = 0;

    if (input.up) dy = -this.speed;
    if (input.down) dy = this.speed;
    if (input.left) dx = -this.speed;
    if (input.right) dx = this.speed;

    // 尝试移动
    var newX = this.x + dx;
    var newY = this.y + dy;

    // 碰撞检测
    if (!this.checkCollision(newX, this.y, map)) {
        this.x = newX;
    }
    if (!this.checkCollision(this.x, newY, map)) {
        this.y = newY;
    }
};

Player.prototype.checkCollision = function(x, y, map) {
    // 检查四个角是否碰到墙壁
    var corners = [
        {x: x, y: y},
        {x: x + this.width, y: y},
        {x: x, y: y + this.height},
        {x: x + this.width, y: y + this.height}
    ];

    for (var i = 0; i < corners.length; i++) {
        var tile = map.getTileAt(corners[i].x, corners[i].y);
        if (tile === 1) return true;  // 1 = 墙壁
    }
    return false;
};
```

### NPC

```javascript
function NPC(x, y, name, dialog) {
    Entity.call(this, x, y);
    this.name = name;
    this.dialog = dialog;  // 对话内容
}
NPC.prototype = Object.create(Entity.prototype);

NPC.prototype.render = function(ctx, cameraX, cameraY) {
    // 画 NPC
    ctx.fillStyle = '#4CAF50';
    ctx.fillRect(
        this.x - cameraX,
        this.y - cameraY,
        this.width,
        this.height
    );

    // 画名字
    ctx.fillStyle = '#fff';
    ctx.font = '12px Arial';
    ctx.textAlign = 'center';
    ctx.fillText(this.name, this.x - cameraX + this.width/2, this.y - cameraY - 5);
};
```

---

## 对象管理器

```javascript
// ============================================
// EntityManager - 管理所有游戏对象
// ============================================

var EntityManager = {
    entities: [],

    // 添加对象
    add: function(entity) {
        this.entities.push(entity);
    },

    // 移除对象
    remove: function(entity) {
        var index = this.entities.indexOf(entity);
        if (index > -1) {
            this.entities.splice(index, 1);
        }
    },

    // 更新所有对象
    update: function() {
        for (var i = 0; i < this.entities.length; i++) {
            if (this.entities[i].active) {
                this.entities[i].update();
            }
        }
    },

    // 渲染所有对象
    render: function(ctx, cameraX, cameraY) {
        // 按 Y 坐标排序（后面的先画）
        this.entities.sort(function(a, b) {
            return a.y - b.y;
        });

        for (var i = 0; i < this.entities.length; i++) {
            if (this.entities[i].active) {
                this.entities[i].render(ctx, cameraX, cameraY);
            }
        }
    }
};
```

---

## 本章小结

1. **Entity** 是所有游戏对象的基类
2. **继承**让代码复用更简单
3. **EntityManager** 统一管理所有对象
4. **Y 坐标排序**实现正确的遮挡关系

## 下一章

[第七章：碰撞检测](./07-collision.md) - 让角色不能穿墙
