# 第九章：输入处理

## 输入的三种状态

1. **Pressed（按下）**：这一帧刚按下
2. **Held（持续）**：一直按着
3. **Released（释放）**：这一帧刚松开

```javascript
var Input = {
    keys: {},           // 当前状态
    previousKeys: {},   // 上一帧状态

    // 按键映射
    keyMap: {
        'ArrowUp': 'up',
        'ArrowDown': 'down',
        'ArrowLeft': 'left',
        'ArrowRight': 'right',
        'z': 'confirm',
        'x': 'cancel',
        'Enter': 'confirm',
        'Escape': 'cancel'
    },

    // 初始化
    init: function() {
        var self = this;

        document.addEventListener('keydown', function(e) {
            var action = self.keyMap[e.key];
            if (action) {
                self.keys[action] = true;
                e.preventDefault();
            }
        });

        document.addEventListener('keyup', function(e) {
            var action = self.keyMap[e.key];
            if (action) {
                self.keys[action] = false;
            }
        });
    },

    // 每帧调用，保存上一帧状态
    update: function() {
        this.previousKeys = Object.assign({}, this.keys);
    },

    // 刚按下
    isPressed: function(action) {
        return this.keys[action] && !this.previousKeys[action];
    },

    // 持续按着
    isHeld: function(action) {
        return this.keys[action];
    },

    // 刚释放
    isReleased: function(action) {
        return !this.keys[action] && this.previousKeys[action];
    }
};
```

---

## 使用示例

```javascript
function update() {
    Input.update();

    // 移动：持续按着
    if (Input.isHeld('up')) player.y -= speed;
    if (Input.isHeld('down')) player.y += speed;

    // 确认：只触发一次
    if (Input.isPressed('confirm')) {
        talkToNPC();
    }

    // 取消：只触发一次
    if (Input.isPressed('cancel')) {
        openMenu();
    }
}
```

---

## 本章小结

1. **三种状态**：Pressed、Held、Released
2. **按键映射**让代码更易读
3. **防止默认行为**避免浏览器快捷键冲突

## 下一章

[第十章：事件系统](./10-event-system.md) - 实现 NPC 对话
