# 第十章：事件系统

## 什么是游戏事件？

游戏中的"事件"是**触发特定条件后发生的事情**：

- 走到某个位置 → 显示对话
- 按下确认键 → 打开宝箱
- 打败 BOSS → 触发剧情

---

## 事件触发器

```javascript
// 事件触发器
function Trigger(x, y, width, height, action) {
    this.x = x;
    this.y = y;
    this.width = width;
    this.height = height;
    this.action = action;    // 触发时执行的函数
    this.oneShot = false;    // 是否只触发一次
    this.triggered = false;  // 是否已触发过
}

Trigger.prototype.check = function(entity) {
    if (this.oneShot && this.triggered) return false;

    // 检测玩家是否在触发区域内
    if (checkAABB(entity, this)) {
        this.action();
        this.triggered = true;
        return true;
    }
    return false;
};
```

---

## NPC 对话事件

```javascript
function DialogEvent(x, y, dialogs) {
    this.x = x;
    this.y = y;
    this.dialogs = dialogs;  // 对话内容数组
    this.currentIndex = 0;
    this.isActive = false;
}

DialogEvent.prototype.trigger = function() {
    if (!this.isActive) {
        this.isActive = true;
        this.currentIndex = 0;
        showDialog(this.dialogs[this.currentIndex]);
    }
};

DialogEvent.prototype.next = function() {
    this.currentIndex++;
    if (this.currentIndex < this.dialogs.length) {
        showDialog(this.dialogs[this.currentIndex]);
    } else {
        this.isActive = false;
        hideDialog();
    }
};

// 使用示例
var npcEvent = new DialogEvent(200, 150, [
    '你好，欢迎来到村庄！',
    '这里是我出生长大的地方。',
    '有什么需要帮忙的吗？'
]);
```

---

## 事件管理器

```javascript
var EventManager = {
    events: [],

    add: function(event) {
        this.events.push(event);
    },

    update: function(player) {
        // 检查玩家是否触发任何事件
        for (var i = 0; i < this.events.length; i++) {
            var event = this.events[i];

            // 玩家按下确认键且在事件旁边
            if (Input.isPressed('confirm')) {
                var distance = Math.sqrt(
                    Math.pow(player.x - event.x, 2) +
                    Math.pow(player.y - event.y, 2)
                );
                if (distance < 50) {
                    event.trigger();
                }
            }
        }
    }
};
```

---

## 本章小结

1. **事件**由触发条件和执行动作组成
2. **Trigger**检测玩家位置
3. **DialogEvent**实现对话系统

## 下一章

[第十一章：用户界面](./11-ui-system.md) - 实现对话框和菜单
