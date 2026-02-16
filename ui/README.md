# UI 系统详解

## 一、UI 组件体系

RPG Maker MV 的 UI 系统基于 **Window** 类构建，所有窗口都继承自 PIXI.Container。

### 1.1 继承层次

```
PIXI.Container
└── Window (窗口容器)
    ├── Window_Base (基础窗口)
    │   ├── Window_Selectable (可选择窗口)
    │   │   ├── Window_Command (命令窗口)
    │   │   ├── Window_MenuCommand (菜单命令)
    │   │   ├── Window_BattleActor (战斗角色选择)
    │   │   ├── Window_BattleEnemy (战斗敌人选择)
    │   │   ├── Window_ItemList (物品列表)
    │   │   ├── Window_SkillList (技能列表)
    │   │   └── ...
    │   ├── Window_Help (帮助窗口)
    │   ├── Window_Gold (金币窗口)
    │   ├── Window_Status (状态窗口)
    │   ├── Window_Message (消息窗口)
    │   ├── Window_ChoiceList (选择窗口)
    │   ├── Window_NumberInput (数字输入)
    │   └── ...
    └── WindowLayer (窗口层)
```

### 1.2 核心组件列表

| 组件 | 用途 | 场景 |
|------|------|------|
| Window_Base | 所有窗口的基类 | 绘制文字、图标、计量条 |
| Window_Selectable | 可选择列表 | 菜单、物品、技能 |
| Window_Command | 命令按钮列表 | 主菜单、战斗指令 |
| Window_Help | 显示说明文字 | 技能/物品描述 |
| Window_Gold | 显示金币 | 菜单、商店 |
| Window_MenuStatus | 队伍状态 | 主菜单 |
| Window_ItemList | 物品列表 | 物品/装备画面 |
| Window_SkillList | 技能列表 | 技能画面 |
| Window_EquipSlot | 装备槽 | 装备画面 |
| Window_EquipStatus | 装备状态 | 装备画面 |
| Window_ShopBuy | 商店购买 | 商店画面 |
| Window_ShopSell | 商店出售 | 商店画面 |
| Window_Message | 对话框 | 事件消息 |
| Window_ChoiceList | 选项 | 事件选择 |
| Window_NumberInput | 数字输入 | 事件输入 |
| Window_BattleLog | 战斗日志 | 战斗场景 |
| Window_PartyCommand | 队伍指令 | 战斗场景 |
| Window_ActorCommand | 角色指令 | 战斗场景 |

---

## 二、Window 核心渲染原理

### 2.1 Window 类结构

```javascript
function Window() {
    this.initialize.apply(this, arguments);
}

Window.prototype = Object.create(PIXI.Container.prototype);
Window.prototype.constructor = Window;

Window.prototype.initialize = function() {
    PIXI.Container.prototype.initialize.call(this);
    
    // 窗口的三个核心层
    this._windowSpriteContainer = new PIXI.Container();  // 主容器
    this._windowBackSprite = new Sprite();               // 背景
    this._windowCursorSprite = new Sprite();             // 光标
    this._windowFrameSprite = new Sprite();              // 边框
    this._windowContentsSprite = new Sprite();           // 内容
    
    this._windowSpriteContainer.addChild(this._windowBackSprite);
    this._windowSpriteContainer.addChild(this._windowFrameSprite);
    this.addChild(this._windowSpriteContainer);
    this.addChild(this._windowCursorSprite);
    this.addChild(this._windowContentsSprite);
};
```

**渲染层次**：
```
Window (PIXI.Container)
├── _windowSpriteContainer
│   ├── _windowBackSprite (背景，包含色调)
│   └── _windowFrameSprite (边框，9宫格拉伸)
├── _windowCursorSprite (光标动画)
└── _windowContentsSprite (内容位图 Bitmap)
```

### 2.2 背景渲染

```javascript
Window.prototype._refreshBack = function() {
    var bitmap = new Bitmap(this.width, this.height);
    
    // 绘制渐变背景
    var c1 = this._color1;  // 顶部颜色
    var c2 = this._color2;  // 底部颜色
    var grad = bitmap._context.createLinearGradient(0, 0, 0, this.height);
    grad.addColorStop(0, c1);
    grad.addColorStop(1, c2);
    
    bitmap._context.fillStyle = grad;
    bitmap._context.fillRect(0, 0, this.width, this.height);
    
    // 应用透明度
    bitmap.baseTexture.update();
    
    this._windowBackSprite.bitmap = bitmap;
    this._windowBackSprite.setFrame(0, 0, this.width, this.height);
    this._windowBackSprite.alpha = this.backOpacity / 255;
};
```

### 2.3 边框渲染（9宫格）

```javascript
Window.prototype._refreshFrame = function() {
    var bitmap = ImageManager.loadSystem('Window');  // 窗口皮肤
    
    // 9宫格边框参数
    var pw = this._width;   // 窗口宽度
    var ph = this._height;  // 窗口高度
    var mw = 48;  // 左右边框宽度
    var mh = 48;  // 上下边框高度
    var margin = 24;  // 边距
    
    // 4个角 + 4个边
    // 左上角
    this._windowFrameSprite.bitmap = bitmap;
    this._windowFrameSprite.setFrame(0, 0, pw, ph);
    
    // 实际是9个精灵组合
    // [左上][上边][右上]
    // [左边][中心][右边]
    // [左下][下边][右下]
};
```

**窗口皮肤结构** (Window.png)：
```
┌────────┬────────┬────────┐
│ 左上角 │ 上边框 │ 右上角 │  64x64
├────────┼────────┼────────┤
│ 左边框 │ 中心   │ 右边框 │  64x64
├────────┼────────┼────────┤
│ 左下角 │ 下边框 │ 右下角 │  64x64
└────────┴────────┴────────┘
每个区域 64x64 像素
```

### 2.4 光标渲染

```javascript
Window.prototype._refreshCursor = function() {
    var bitmap = ImageManager.loadSystem('Window');
    
    // 光标在窗口皮肤中的位置
    var sx = 96;   // X偏移
    var sy = 64;   // Y偏移
    var sw = 48;   // 光标宽度
    var sh = 48;   // 光标高度
    
    this._windowCursorSprite.bitmap = bitmap;
    this._windowCursorSprite.setFrame(sx, sy, sw, sh);
    
    // 设置光标位置（跟随选中项）
    var rect = this._cursorRect;
    this._windowCursorSprite.x = rect.x;
    this._windowCursorSprite.y = rect.y;
    this._windowCursorSprite.scale.x = rect.width / sw;
    this._windowCursorSprite.scale.y = rect.height / sh;
};

// 光标闪烁动画
Window.prototype._updateCursor = function() {
    this._windowCursorSprite.alpha = this._cursorAlpha;
    this._cursorAlpha += this._cursorBlinkSpeed;
    
    if (this._cursorAlpha >= 1) {
        this._cursorAlpha = 1;
        this._cursorBlinkSpeed = -0.04;
    } else if (this._cursorAlpha <= 0) {
        this._cursorAlpha = 0;
        this._cursorBlinkSpeed = 0.04;
    }
};
```

---

## 三、Bitmap 绘制系统

### 3.1 Bitmap 类核心

```javascript
function Bitmap(width, height) {
    this.initialize.apply(this, arguments);
}

Bitmap.prototype.initialize = function(width, height) {
    this._canvas = document.createElement('canvas');
    this._context = this._canvas.getContext('2d');
    this._canvas.width = width;
    this._canvas.height = height;
    
    this._baseTexture = new PIXI.BaseTexture(this._canvas);
    this._texture = new PIXI.Texture(this._baseTexture);
    
    this.width = width;
    this.height = height;
    this._loadingState = 'initialized';
};
```

### 3.2 文字绘制

```javascript
Bitmap.prototype.drawText = function(text, x, y, maxWidth, lineHeight, align) {
    var context = this._context;
    
    // 设置字体
    context.font = this._makeFontNameText();
    context.textAlign = align || 'left';
    context.textBaseline = 'top';
    
    // 设置颜色
    context.fillStyle = this.textColor;
    
    // 计算位置
    var tx = x;
    if (align === 'center') {
        tx = x + maxWidth / 2;
    } else if (align === 'right') {
        tx = x + maxWidth;
    }
    
    // 绘制
    context.fillText(text, tx, y + this.fontSize * 0.4);
    
    // 标记需要更新纹理
    this._setDirty();
};

// 字体设置
Bitmap.prototype._makeFontNameText = function() {
    return (this.fontItalic ? 'Italic ' : '') +
           this.fontSize + 'px ' + 
           this.fontFace;
};
```

### 3.3 图形绘制

```javascript
// 绘制矩形
Bitmap.prototype.fillRect = function(x, y, width, height, color) {
    var context = this._context;
    context.fillStyle = color;
    context.fillRect(x, y, width, height);
    this._setDirty();
};

// 绘制渐变矩形
Bitmap.prototype.gradientFillRect = function(x, y, width, height, color1, color2, vertical) {
    var context = this._context;
    var grad;
    
    if (vertical) {
        grad = context.createLinearGradient(x, y, x, y + height);
    } else {
        grad = context.createLinearGradient(x, y, x + width, y);
    }
    
    grad.addColorStop(0, color1);
    grad.addColorStop(1, color2);
    
    context.fillStyle = grad;
    context.fillRect(x, y, width, height);
    this._setDirty();
};

// 绘制圆角矩形
Bitmap.prototype.drawRoundRect = function(x, y, width, height, radius, color) {
    var context = this._context;
    context.fillStyle = color;
    context.beginPath();
    context.roundRect(x, y, width, height, radius);
    context.fill();
    this._setDirty();
};

// 绘制位图（blt = bit block transfer）
Bitmap.prototype.blt = function(source, sx, sy, sw, sh, dx, dy, dw, dh) {
    dw = dw || sw;
    dh = dh || sh;
    
    if (sw > 0 && sh > 0 && dw > 0 && dh > 0) {
        this._context.drawImage(source._canvas, sx, sy, sw, sh, dx, dy, dw, dh);
        this._setDirty();
    }
};
```

### 3.4 计量条绘制详解

```javascript
Window_Base.prototype.drawGauge = function(x, y, width, rate, color1, color2) {
    // rate: 0.0 ~ 1.0
    var gaugeHeight = 18;
    var gaugeY = y + this.lineHeight() - gaugeHeight - 4;
    
    // 1. 绘制背景（深色）
    var backgroundColor = '#000000';
    this.contents.fillRect(x, gaugeY, width, gaugeHeight, backgroundColor);
    
    // 2. 绘制填充（渐变）
    if (rate > 0) {
        var fillWidth = Math.floor(width * rate);
        this.contents.gradientFillRect(x, gaugeY, fillWidth, gaugeHeight, color1, color2);
    }
    
    // 3. 绘制边框
    var borderColor = 'rgba(255,255,255,0.3)';
    this.contents.strokeStyle = borderColor;
    this.contents.lineWidth = 1;
    this.contents.strokeRect(x, gaugeY, width, gaugeHeight);
};

// 使用示例
this.drawGauge(20, 50, 200, 0.7, '#ff4040', '#ff8080');  // HP条
this.drawGauge(20, 80, 200, 0.5, '#4040ff', '#8080ff');  // MP条
this.drawGauge(20, 110, 200, 0.3, '#40ff40', '#80ff80'); // TP条
```

---

## 四、输入事件系统

### 4.1 Input 类 - 键盘输入

```javascript
function Input() {
    throw new Error('This is a static class');
}

Input.initialize = function() {
    this.clear();
    this._setupEventHandlers();
};

Input.clear = function() {
    this._currentState = {};
    this._previousState = {};
    this._latestButton = null;
    this._pressedTime = 0;
};

// 按键映射
Input.keyMapper = {
    9: 'tab',       // Tab
    13: 'ok',       // Enter
    16: 'shift',    // Shift
    17: 'control',  // Ctrl
    18: 'control',  // Alt
    27: 'escape',   // ESC
    32: 'ok',       // Space
    33: 'pageup',   // PageUp
    34: 'pagedown', // PageDown
    37: 'left',     // ←
    38: 'up',       // ↑
    39: 'right',    // →
    40: 'down',     // ↓
    45: 'escape',   // Insert
    81: 'pageup',   // Q
    87: 'pagedown', // W
    88: 'escape',   // X
    90: 'ok',       // Z
    96: 'escape',   // Num0
    98: 'down',     // Num2
    100: 'left',    // Num4
    102: 'right',   // Num6
    104: 'up'       // Num8
};
```

### 4.2 输入检测方法

```javascript
// 持续按下
Input.isPressed = function(keyName) {
    return !!this._currentState[keyName];
};

// 刚按下（单次触发）
Input.isTriggered = function(keyName) {
    return this._latestButton === keyName && this._pressedTime === 0;
};

// 重复按下（支持连按）
Input.isRepeated = function(keyName) {
    return (this._latestButton === keyName && 
            (this._pressedTime === 0 || 
             (this._pressedTime >= this.keyRepeatWait && 
              this._pressedTime % this.keyRepeatInterval === 0)));
};

// 长按
Input.isLongPressed = function(keyName) {
    return (this._latestButton === keyName && 
            this._pressedTime >= this.keyRepeatWait);
};
```

**检测时机图**：
```
按键: [按下──────────────────────────释放]
        │    │                    │
        ▼    ▼                    ▼
Triggered  Repeated            Repeated
 (1次)    (延迟后重复)          (持续重复)
 
isPressed: [──────────────────────────────] 整个按下期间返回true
```

### 4.3 TouchInput 类 - 触摸/鼠标

```javascript
function TouchInput() {
    throw new Error('This is a static class');
}

TouchInput.initialize = function() {
    this.clear();
    this._setupEventHandlers();
};

// 核心属性
TouchInput._mouseX = 0;           // 鼠标X
TouchInput._mouseY = 0;           // 鼠标Y
TouchInput._screenX = 0;          // 屏幕X
TouchInput._screenY = 0;          // 屏幕Y
TouchInput._clicked = false;      // 点击
TouchInput._cancelled = false;    // 取消（右键）
TouchInput._moved = false;        // 移动
TouchInput._pressed = false;      // 按下
TouchInput._released = false;     // 释放
TouchInput._wheelX = 0;           // 滚轮X
TouchInput._wheelY = 0;           // 滚轮Y
```

### 4.4 触摸事件处理

```javascript
TouchInput._setupEventHandlers = function() {
    var target = document;
    
    // 鼠标事件
    target.addEventListener('mousedown', this._onMouseDown.bind(this));
    target.addEventListener('mousemove', this._onMouseMove.bind(this));
    target.addEventListener('mouseup', this._onMouseUp.bind(this));
    target.addEventListener('wheel', this._onWheel.bind(this));
    
    // 触摸事件
    target.addEventListener('touchstart', this._onTouchStart.bind(this));
    target.addEventListener('touchmove', this._onTouchMove.bind(this));
    target.addEventListener('touchend', this._onTouchEnd.bind(this));
    
    // 取消（右键）
    target.addEventListener('contextmenu', this._onContextMenu.bind(this));
};

// 坐标转换
TouchInput._onMouseMove = function(event) {
    var x = Graphics.pageToCanvasX(event.pageX);
    var y = Graphics.pageToCanvasY(event.pageY);
    
    this._mouseX = x;
    this._mouseY = y;
    
    if (this._pressed) {
        this._moved = true;
    }
};

// 屏幕坐标到游戏坐标
Graphics.pageToCanvasX = function(x) {
    var rect = this._canvas.getBoundingClientRect();
    return Math.floor((x - rect.left) * this._canvas.width / rect.width);
};

Graphics.pageToCanvasY = function(y) {
    var rect = this._canvas.getBoundingClientRect();
    return Math.floor((y - rect.top) * this._canvas.height / rect.height);
};
```

---

## 五、窗口交互系统

### 5.1 Window_Selectable 输入处理

```javascript
Window_Selectable.prototype.processCursorMove = function() {
    if (this.isCursorMovable()) {
        var lastIndex = this.index();
        
        // 方向键
        if (Input.isRepeated('down')) {
            this.cursorDown(Input.isTriggered('down'));
        }
        if (Input.isRepeated('up')) {
            this.cursorUp(Input.isTriggered('up'));
        }
        if (Input.isRepeated('right')) {
            this.cursorRight(Input.isTriggered('right'));
        }
        if (Input.isRepeated('left')) {
            this.cursorLeft(Input.isTriggered('left'));
        }
        
        // 翻页
        if (Input.isTriggered('pagedown')) {
            this.cursorPagedown();
        }
        if (Input.isTriggered('pageup')) {
            this.cursorPageup();
        }
        
        // 播放光标音效
        if (this.index() !== lastIndex) {
            SoundManager.playCursor();
        }
    }
};

// 确认/取消处理
Window_Selectable.prototype.processHandling = function() {
    if (this.isActive()) {
        if (Input.isTriggered('ok')) {
            this.processOk();
        }
        if (Input.isTriggered('cancel')) {
            this.processCancel();
        }
    }
};

Window_Selectable.prototype.processOk = function() {
    if (this.isCurrentItemEnabled()) {
        this._index = this.index();
        this.callOkHandler();
    } else {
        this.playBuzzerSound();
    }
};

Window_Selectable.prototype.callOkHandler = function() {
    if (this.isHandled('ok')) {
        this._handlers['ok'](this.index());
    }
};
```

### 5.2 触摸/鼠标点击

```javascript
Window_Selectable.prototype.processTouch = function() {
    if (this.isActive()) {
        if (TouchInput.isTriggered()) {
            this._touching = true;
            this._touchSelect(TouchInput.x, TouchInput.y);
        }
        
        if (TouchInput.isPressed() && this._touching) {
            if (TouchInput.moved) {
                this._touchSelect(TouchInput.x, TouchInput.y);
            }
        } else {
            this._touching = false;
        }
        
        if (TouchInput.isReleased() && this._touching) {
            this._touching = false;
            if (this.index() >= 0) {
                this.processOk();
            }
        }
    }
};

Window_Selectable.prototype._touchSelect = function(x, y) {
    var hitIndex = this.hitTest(x, y);
    if (hitIndex >= 0 && hitIndex < this.maxItems()) {
        this.select(hitIndex);
    }
};

// 点击测试
Window_Selectable.prototype.hitTest = function(x, y) {
    // 转换为窗口内坐标
    var localX = x - this.x - this.padding;
    var localY = y - this.y - this.padding;
    
    // 遍历所有项目矩形
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

### 5.3 处理器注册系统

```javascript
// 注册处理器
Window_Selectable.prototype.setHandler = function(symbol, method) {
    this._handlers[symbol] = method;
};

// 调用处理器
Window_Selectable.prototype.callHandler = function(symbol) {
    if (this.isHandled(symbol)) {
        this._handlers[symbol]();
    }
};

// 检查是否已注册
Window_Selectable.prototype.isHandled = function(symbol) {
    return !!this._handlers[symbol];
};

// 使用示例
var win = new Window_MenuCommand(0, 0);
win.setHandler('item', function() {
    SceneManager.push(Scene_Item);
});
win.setHandler('skill', function() {
    SceneManager.push(Scene_Skill);
});
win.setHandler('cancel', function() {
    SceneManager.pop();
});
```

---

## 六、实际组件案例

### 6.1 自定义状态窗口

```javascript
function Window_CustomStatus() {
    this.initialize.apply(this, arguments);
}

Window_CustomStatus.prototype = Object.create(Window_Base.prototype);
Window_CustomStatus.prototype.constructor = Window_CustomStatus;

Window_CustomStatus.prototype.initialize = function(x, y, width, height) {
    Window_Base.prototype.initialize.call(this, x, y, width, height);
    this._actor = null;
    this.refresh();
};

Window_CustomStatus.prototype.setActor = function(actor) {
    if (this._actor !== actor) {
        this._actor = actor;
        this.refresh();
    }
};

Window_CustomStatus.prototype.refresh = function() {
    this.contents.clear();
    if (this._actor) {
        this.drawActorFace(this._actor, 10, 10);
        this.drawActorName(this._actor, 160, 10);
        this.drawActorLevel(this._actor, 160, 40);
        this.drawActorHp(this._actor, 160, 70, 200);
        this.drawActorMp(this._actor, 160, 100, 200);
        this.drawActorIcons(this._actor, 160, 130);
    }
};

Window_CustomStatus.prototype.drawActorHp = function(actor, x, y, width) {
    var color1 = '#ff4040';
    var color2 = '#ff8080';
    
    // 绘制标签
    this.changeTextColor(this.systemColor());
    this.drawText('HP', x, y, 40);
    
    // 绘制计量条
    var rate = actor.hp / actor.mhp;
    this.drawGauge(x + 44, y, width - 44, rate, color1, color2);
    
    // 绘制数值
    this.changeTextColor(this.normalColor());
    this.drawText(actor.hp + '/' + actor.mhp, x + 44, y, width - 44, 'right');
};

// 使用
var statusWindow = new Window_CustomStatus(0, 0, 400, 200);
SceneManager._scene.addWindow(statusWindow);
statusWindow.setActor($gameActors.actor(1));
```

### 6.2 自定义命令窗口

```javascript
function Window_CustomCommand() {
    this.initialize.apply(this, arguments);
}

Window_CustomCommand.prototype = Object.create(Window_Command.prototype);
Window_CustomCommand.prototype.constructor = Window_CustomCommand;

Window_CustomCommand.prototype.initialize = function(x, y) {
    Window_Command.prototype.initialize.call(this, x, y);
};

// 定义命令列表
Window_CustomCommand.prototype.makeCommandList = function() {
    this.addCommand('攻击', 'attack', true);
    this.addCommand('魔法', 'magic', this._actor && this._actor.canUseMagic());
    this.addCommand('物品', 'item', true);
    this.addCommand('防御', 'guard', true);
    this.addCommand('逃跑', 'escape', BattleManager.canEscape());
};

// 绘制命令
Window_CustomCommand.prototype.drawItem = function(index) {
    var rect = this.itemRectForText(index);
    var status = this.commandStatus(index);
    var enabled = this.isCommandEnabled(index);
    
    // 根据状态改变颜色
    if (!enabled) {
        this.changeTextColor(this.deathColor());
    } else if (status === 'magic') {
        this.changeTextColor(this.powerUpColor());
    } else {
        this.changeTextColor(this.normalColor());
    }
    
    this.drawText(this.commandName(index), rect.x, rect.y, rect.width, 'center');
};

// 使用
var cmdWindow = new Window_CustomCommand(100, 200);
cmdWindow.setHandler('attack', function() {
    BattleManager.selectNextCommand();
});
cmdWindow.setHandler('magic', function() {
    SceneManager._scene.startActorSkillSelection();
});
```

### 6.3 自定义列表窗口

```javascript
function Window_Inventory() {
    this.initialize.apply(this, arguments);
}

Window_Inventory.prototype = Object.create(Window_Selectable.prototype);
Window_Inventory.prototype.constructor = Window_Inventory;

Window_Inventory.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._category = 'all';
    this._data = [];
    this.refresh();
};

// 列数
Window_Inventory.prototype.maxCols = function() {
    return 2;
};

// 项目数
Window_Inventory.prototype.maxItems = function() {
    return this._data.length;
};

// 项目高度
Window_Inventory.prototype.itemHeight = function() {
    return this.lineHeight() * 2 + 10;  // 2行高度 + 间距
};

// 绘制项目
Window_Inventory.prototype.drawItem = function(index) {
    var item = this._data[index];
    if (!item) return;
    
    var rect = this.itemRect(index);
    
    // 绘制图标
    this.drawIcon(item.iconIndex, rect.x + 5, rect.y + 5);
    
    // 绘制名称
    this.changeTextColor(this.normalColor());
    this.drawText(item.name, rect.x + 40, rect.y + 5, rect.width - 45);
    
    // 绘制数量
    var count = $gameParty.numItems(item);
    this.drawText('×' + count, rect.x, rect.y + 5, rect.width - 5, 'right');
    
    // 绘制简短描述
    this.changeTextColor(this.systemColor());
    var desc = item.description.substring(0, 20) + '...';
    this.drawText(desc, rect.x + 40, rect.y + this.lineHeight() + 5, rect.width - 45);
};

// 刷新
Window_Inventory.prototype.refresh = function() {
    this._data = this._getItems();
    this.createContents();
    this.drawAllItems();
};

Window_Inventory.prototype._getItems = function() {
    var items = [];
    
    // 添加物品
    if (this._category === 'all' || this._category === 'items') {
        items = items.concat($dataItems.filter(function(item) {
            return item && item.name && $gameParty.numItems(item) > 0;
        }));
    }
    
    // 添加武器
    if (this._category === 'all' || this._category === 'weapons') {
        items = items.concat($dataWeapons.filter(function(item) {
            return item && item.name && $gameParty.numItems(item) > 0;
        }));
    }
    
    return items;
};
```

---

## 七、UI 开发最佳实践

### 7.1 窗口生命周期

```javascript
// 创建
Scene_Menu.prototype.create = function() {
    this.createCommandWindow();
    this.createStatusWindow();
};

// 更新
Scene_Menu.prototype.update = function() {
    Scene_Base.prototype.update.call(this);
    this._statusWindow.setActor($gameParty.menuActor());
};

// 销毁
Scene_Menu.prototype.terminate = function() {
    // 自动由Scene系统处理
};
```

### 7.2 性能优化

```javascript
// ❌ 不好的做法 - 每帧刷新
Window_Status.prototype.update = function() {
    Window_Base.prototype.update.call(this);
    this.refresh();  // 每帧都重绘
};

// ✅ 好的做法 - 只在需要时刷新
Window_Status.prototype.setActor = function(actor) {
    if (this._actor !== actor) {
        this._actor = actor;
        this.refresh();  // 只在actor变化时重绘
    }
};

// ✅ 使用脏标记
Window_Status.prototype.setHp = function(hp) {
    if (this._hp !== hp) {
        this._hp = hp;
        this._needsRefresh = true;
    }
};

Window_Status.prototype.update = function() {
    Window_Base.prototype.update.call(this);
    if (this._needsRefresh) {
        this.refresh();
        this._needsRefresh = false;
    }
};
```

### 7.3 响应式布局

```javascript
Window_MenuStatus.prototype.windowWidth = function() {
    return Graphics.boxWidth - 240;
};

Window_MenuStatus.prototype.windowHeight = function() {
    return Graphics.boxHeight;
};

Window_MenuStatus.prototype.numVisibleRows = function() {
    return Math.floor(this.windowHeight() / this.itemHeight());
};
```

---

## 八、调试技巧

### 8.1 可视化调试

```javascript
// 显示窗口边框
Window_Base.prototype.drawDebugBorder = function() {
    this.contents.strokeStyle = '#ff0000';
    this.contents.lineWidth = 2;
    this.contents.strokeRect(0, 0, this.contents.width, this.contents.height);
};

// 显示点击区域
Window_Selectable.prototype.drawDebugHitAreas = function() {
    for (var i = 0; i < this.maxItems(); i++) {
        var rect = this.itemRect(i);
        this.contents.strokeStyle = 'rgba(0,255,0,0.5)';
        this.contents.strokeRect(rect.x, rect.y, rect.width, rect.height);
    }
};
```

### 8.2 输入监控

```javascript
// 监控所有按键
var _Input_update = Input.update;
Input.update = function() {
    _Input_update.call(this);
    
    for (var key in this._currentState) {
        if (this._currentState[key]) {
            console.log('[Input]', key, 
                'Pressed:', this.isPressed(key),
                'Triggered:', this.isTriggered(key),
                'Repeated:', this.isRepeated(key));
        }
    }
};
```

### 8.3 窗口树查看

```javascript
function dumpWindowTree(window, indent) {
    indent = indent || '';
    console.log(indent + window.constructor.name, 
        'pos:', window.x + ',' + window.y,
        'size:', window.width + 'x' + window.height);
    
    if (window.children) {
        window.children.forEach(function(child) {
            dumpWindowTree(child, indent + '  ');
        });
    }
}

dumpWindowTree(SceneManager._scene._windowLayer);
```

---

## 九、常见问题

### Q1: 窗口不显示内容？
```javascript
// 确保调用了 refresh()
MyWindow.prototype.initialize = function() {
    Window_Base.prototype.initialize.call(this, 0, 0, 400, 300);
    this.refresh();  // 必须调用
};
```

### Q2: 文字超出窗口？
```javascript
// 使用 textWidth 检测
if (this.textWidth(text) > rect.width) {
    text = text.substring(0, 20) + '...';
}
this.drawText(text, x, y, rect.width);
```

### Q3: 点击无响应？
```javascript
// 确保 active = true
window.active = true;

// 确保处理器已注册
window.setHandler('ok', myHandler);
```

### Q4: 光标位置错误？
```javascript
// 检查 itemRect 计算
MyWindow.prototype.itemRect = function(index) {
    var rect = new Rectangle();
    rect.x = (index % this.maxCols()) * this.itemWidth();
    rect.y = Math.floor(index / this.maxCols()) * this.itemHeight();
    rect.width = this.itemWidth();
    rect.height = this.itemHeight();
    return rect;
};
```
