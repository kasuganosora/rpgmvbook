# 第十三章：物品与背包

## 物品数据结构

```javascript
// 物品定义
var ItemDatabase = {
    1: { id: 1, name: '回复药', type: 'consumable', effect: 'heal', value: 50, price: 10 },
    2: { id: 2, name: '魔法药', type: 'consumable', effect: 'mp', value: 30, price: 15 },
    3: { id: 3, name: '铁剑', type: 'weapon', attack: 5, price: 100 },
    4: { id: 4, name: '皮甲', type: 'armor', defense: 3, price: 80 }
};
```

---

## 背包系统

```javascript
function Inventory(maxItems) {
    this.maxItems = maxItems || 99;
    this.items = [];  // [{id: 1, count: 3}, ...]
}

// 添加物品
Inventory.prototype.addItem = function(itemId, count) {
    count = count || 1;

    // 查找是否已有
    for (var i = 0; i < this.items.length; i++) {
        if (this.items[i].id === itemId) {
            this.items[i].count += count;
            return true;
        }
    }

    // 新物品
    if (this.items.length < this.maxItems) {
        this.items.push({ id: itemId, count: count });
        return true;
    }

    return false;  // 背包满了
};

// 移除物品
Inventory.prototype.removeItem = function(itemId, count) {
    count = count || 1;

    for (var i = 0; i < this.items.length; i++) {
        if (this.items[i].id === itemId) {
            this.items[i].count -= count;
            if (this.items[i].count <= 0) {
                this.items.splice(i, 1);
            }
            return true;
        }
    }
    return false;
};

// 获取物品数量
Inventory.prototype.getItemCount = function(itemId) {
    for (var i = 0; i < this.items.length; i++) {
        if (this.items[i].id === itemId) {
            return this.items[i].count;
        }
    }
    return 0;
};

// 使用物品
Inventory.prototype.useItem = function(itemId, target) {
    var item = ItemDatabase[itemId];
    if (!item) return false;

    if (item.type === 'consumable') {
        if (this.getItemCount(itemId) <= 0) return false;

        // 应用效果
        switch (item.effect) {
            case 'heal':
                target.hp = Math.min(target.maxHp, target.hp + item.value);
                break;
            case 'mp':
                target.mp = Math.min(target.maxMp, target.mp + item.value);
                break;
        }

        this.removeItem(itemId, 1);
        return true;
    }

    return false;
};
```

---

## 物品窗口

```javascript
function ItemWindow(inventory) {
    Window.call(this, 50, 50, 300, 400);
    this.inventory = inventory;
    this.selectedIndex = 0;
}
ItemWindow.prototype = Object.create(Window.prototype);

ItemWindow.prototype.update = function() {
    if (Input.isPressed('up')) {
        this.selectedIndex = Math.max(0, this.selectedIndex - 1);
    }
    if (Input.isPressed('down')) {
        this.selectedIndex = Math.min(this.inventory.items.length - 1, this.selectedIndex + 1);
    }
};

ItemWindow.prototype.render = function(ctx) {
    Window.prototype.render.call(this, ctx);

    var items = this.inventory.items;
    for (var i = 0; i < items.length; i++) {
        var item = ItemDatabase[items[i].id];
        var y = this.y + 30 + i * 25;

        // 高亮选中项
        if (i === this.selectedIndex) {
            ctx.fillStyle = 'rgba(255,255,0,0.3)';
            ctx.fillRect(this.x + 5, y - 18, this.width - 10, 24);
        }

        // 物品名称
        ctx.fillStyle = '#fff';
        ctx.fillText(item.name, this.x + 20, y);

        // 数量
        ctx.fillStyle = '#aaa';
        ctx.fillText('×' + items[i].count, this.x + this.width - 50, y);
    }
};
```

---

## 本章小结

1. **物品数据库**定义所有物品
2. **背包**管理拥有的物品
3. **物品窗口**显示和选择物品

## 下一章

[第十四章：战斗系统](./14-battle.md) - 实现回合制战斗
