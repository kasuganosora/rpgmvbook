# 第十一章：用户界面

## UI 的作用

- **对话框**：显示 NPC 对话
- **菜单**：物品、装备、存档
- **HUD**：HP/MP 显示、小地图

---

## 窗口基类

```javascript
function Window(x, y, width, height) {
    this.x = x;
    this.y = y;
    this.width = width;
    this.height = height;
    this.visible = true;
    this.active = false;
}

Window.prototype.update = function() {};

Window.prototype.render = function(ctx) {
    if (!this.visible) return;

    // 背景
    ctx.fillStyle = 'rgba(0, 0, 0, 0.8)';
    ctx.fillRect(this.x, this.y, this.width, this.height);

    // 边框
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 2;
    ctx.strokeRect(this.x, this.y, this.width, this.height);
};
```

---

## 对话框

```javascript
function DialogWindow() {
    Window.call(this, 50, 400, 716, 150);
    this.text = '';
    this.speaker = '';
}
DialogWindow.prototype = Object.create(Window.prototype);

DialogWindow.prototype.show = function(speaker, text) {
    this.speaker = speaker;
    this.text = text;
    this.visible = true;
};

DialogWindow.prototype.render = function(ctx) {
    Window.prototype.render.call(this, ctx);

    // 说话者名字
    ctx.fillStyle = '#ffff00';
    ctx.font = 'bold 16px Arial';
    ctx.fillText(this.speaker, this.x + 20, this.y + 25);

    // 对话内容
    ctx.fillStyle = '#fff';
    ctx.font = '14px Arial';
    this.wrapText(ctx, this.text, this.x + 20, this.y + 50, this.width - 40, 20);
};

Window.prototype.wrapText = function(ctx, text, x, y, maxWidth, lineHeight) {
    var words = text.split('');
    var line = '';

    for (var i = 0; i < words.length; i++) {
        var testLine = line + words[i];
        var metrics = ctx.measureText(testLine);

        if (metrics.width > maxWidth && i > 0) {
            ctx.fillText(line, x, y);
            line = words[i];
            y += lineHeight;
        } else {
            line = testLine;
        }
    }
    ctx.fillText(line, x, y);
};
```

---

## 状态窗口

```javascript
function StatusWindow() {
    Window.call(this, 10, 10, 200, 80);
    this.player = null;
}
StatusWindow.prototype = Object.create(Window.prototype);

StatusWindow.prototype.render = function(ctx) {
    if (!this.player) return;
    Window.prototype.render.call(this, ctx);

    var p = this.player;

    // HP
    ctx.fillStyle = '#fff';
    ctx.font = '14px Arial';
    ctx.fillText('HP: ' + p.hp + '/' + p.maxHp, this.x + 10, this.y + 25);

    // HP 条
    this.drawBar(ctx, this.x + 10, this.y + 35, 180, 10,
                 p.hp / p.maxHp, '#ff4444', '#ff8888');

    // MP
    ctx.fillText('MP: ' + p.mp + '/' + p.maxMp, this.x + 10, this.y + 65);

    // MP 条
    this.drawBar(ctx, this.x + 10, this.y + 75, 180, 10,
                 p.mp / p.maxMp, '#4444ff', '#8888ff');
};

StatusWindow.prototype.drawBar = function(ctx, x, y, w, h, rate, color1, color2) {
    // 背景
    ctx.fillStyle = '#333';
    ctx.fillRect(x, y, w, h);

    // 填充
    ctx.fillStyle = color1;
    ctx.fillRect(x, y, w * rate, h);
};
```

---

## 本章小结

1. **Window** 是所有 UI 的基类
2. **DialogWindow** 显示对话
3. **StatusWindow** 显示 HP/MP

## 下一章

[第十二章：角色数据系统](./12-character-data.md) - 实现角色属性
