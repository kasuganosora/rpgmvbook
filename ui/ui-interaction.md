# 窗口交互机制

## 1. Window_Selectable 更新流程

```javascript
Window_Selectable.prototype.update = function() {
    Window_Base.prototype.update.call(this);

    this.processCursorMove();  // 1. 光标移动
    this.processHandling();    // 2. 确认/取消
    this.processTouch();       // 3. 触摸/鼠标
};
```

**顺序重要**：先移动光标，再处理确认（基于新位置）。

## 2. 光标移动

```javascript
Window_Selectable.prototype.processCursorMove = function() {
    if (this.isCursorMovable()) {
        var lastIndex = this.index();

        // 为什么用 isRepeated？支持长按快速移动
        if (Input.isRepeated('down')) {
            this.cursorDown(Input.isTriggered('down'));  // 传入是否首次（用于音效）
        }
        // ... 其他方向

        // 翻页用 isTriggered（一次一页）
        if (Input.isTriggered('pagedown')) {
            this.cursorPagedown();
        }

        // 光标移动时播放音效
        if (this.index() !== lastIndex) {
            SoundManager.playCursor();
        }
    }
};
```

## 3. 确认/取消

```javascript
Window_Selectable.prototype.processOk = function() {
    if (this.isCurrentItemEnabled()) {
        this.callOkHandler();  // 调用注册的回调
    } else {
        this.playBuzzerSound();  // 不可用时播放蜂鸣音
    }
};

Window_Selectable.prototype.callOkHandler = function() {
    if (this.isHandled('ok')) {
        this._handlers['ok'](this.index());  // 传入选中项索引
    }
};
```

## 4. 处理器注册

```javascript
var commandWindow = new Window_MenuCommand(0, 0);
commandWindow.setHandler('ok', function(index) {
    switch (index) {
        case 0: SceneManager.push(Scene_Item); break;
        case 1: SceneManager.push(Scene_Skill); break;
    }
});
commandWindow.setHandler('cancel', function() {
    SceneManager.pop();
});
```

**设计原因**：窗口只负责交互，具体操作由场景处理（解耦）。

## 5. 触摸/鼠标处理

```javascript
Window_Selectable.prototype.processTouch = function() {
    if (this.isActive()) {
        if (TouchInput.isTriggered()) {
            this._touching = true;
            this._touchSelect(TouchInput.x, TouchInput.y);  // 按下时选择
        }

        if (TouchInput.isPressed() && this._touching && TouchInput.moved) {
            this._touchSelect(TouchInput.x, TouchInput.y);  // 拖动时更新
        }

        if (TouchInput.isReleased() && this._touching) {
            this._touching = false;
            this.processOk();  // 抬起时确认
        }
    }
};

// 计算点击位置对应的项目
Window_Selectable.prototype.hitTest = function(x, y) {
    var localX = x - this.x - this.padding;  // 窗口内坐标
    var localY = y - this.y - this.padding;

    for (var i = 0; i < this.maxItems(); i++) {
        var rect = this.itemRect(i);
        if (localX >= rect.x && localX < rect.x + rect.width &&
            localY >= rect.y && localY < rect.y + rect.height) {
            return i;
        }
    }
    return -1;
};
```

**流程**：按下选择 → 拖动更新 → 抬起确认
