# 第七章：碰撞检测

## 为什么需要碰撞检测？

- 角色不能穿墙
- 角色走到水边会停下
- 角色撞到敌人会受伤
- 角色踩到陷阱会触发

**没有碰撞检测**：游戏就像幽灵世界，一切都能穿透。

---

## 矩形碰撞（AABB）

AABB = Axis-Aligned Bounding Box（轴对齐包围盒）

```javascript
function checkAABB(rect1, rect2) {
    // 两个矩形相交的条件：
    // 1. rect1 的右边 > rect2 的左边
    // 2. rect1 的左边 < rect2 的右边
    // 3. rect1 的下边 > rect2 的上边
    // 4. rect1 的上边 < rect2 的下边

    return rect1.x < rect2.x + rect2.width &&
           rect1.x + rect1.width > rect2.x &&
           rect1.y < rect2.y + rect2.height &&
           rect1.y + rect1.height > rect2.y;
}
```

**图解**：
```
碰撞情况：
┌────┐
│ A  │
└──┬─┘
   │  ┌────┐
   └──┤ B  │
      └────┘

不碰撞情况：
┌────┐      ┌────┐
│ A  │      │ B  │
└────┘      └────┘
（有间隙）
```

---

## 与瓦片地图的碰撞

```javascript
function checkTileCollision(x, y, width, height, map) {
    // 检查矩形四个角
    var tileSize = map.tileSize;

    // 四个角的位置
    var corners = [
        { x: x, y: y },                           // 左上
        { x: x + width - 1, y: y },               // 右上
        { x: x, y: y + height - 1 },              // 左下
        { x: x + width - 1, y: y + height - 1 }   // 右下
    ];

    for (var i = 0; i < corners.length; i++) {
        var tileX = Math.floor(corners[i].x / tileSize);
        var tileY = Math.floor(corners[i].y / tileSize);

        var tile = map.getTile(tileX, tileY);

        // 假设瓦片 1 是墙壁
        if (tile === 1) {
            return true;  // 发生碰撞
        }
    }

    return false;  // 没有碰撞
}
```

---

## 移动与碰撞

**问题**：如果每帧移动 5 像素，可能直接"穿进"墙壁。

**解决方案**：分别检测 X 和 Y 方向的移动

```javascript
function moveWithCollision(entity, dx, dy, map) {
    // 先尝试 X 方向移动
    if (dx !== 0) {
        var newX = entity.x + dx;
        if (!checkTileCollision(newX, entity.y, entity.width, entity.height, map)) {
            entity.x = newX;
        } else {
            // X 方向有碰撞，贴墙
            entity.x = dx > 0 ?
                Math.floor((entity.x + entity.width) / map.tileSize) * map.tileSize - entity.width - 1 :
                Math.ceil(entity.x / map.tileSize) * map.tileSize + 1;
        }
    }

    // 再尝试 Y 方向移动
    if (dy !== 0) {
        var newY = entity.y + dy;
        if (!checkTileCollision(entity.x, newY, entity.width, entity.height, map)) {
            entity.y = newY;
        } else {
            // Y 方向有碰撞，贴墙
            entity.y = dy > 0 ?
                Math.floor((entity.y + entity.height) / map.tileSize) * map.tileSize - entity.height - 1 :
                Math.ceil(entity.y / map.tileSize) * map.tileSize + 1;
        }
    }
}
```

---

## 本章小结

1. **AABB** 是最简单的碰撞检测方法
2. **四角检测**适合瓦片地图
3. **分离轴**可以处理斜向移动

## 下一章

[第八章：地图与场景](./08-map-scene.md) - 实现场景管理
