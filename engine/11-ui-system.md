# 第十一章：用户界面系统

## 前言

UI（用户界面）是玩家与游戏交互的桥梁。一个好的 UI 系统需要：

```
┌─────────────────────────────────────────────────┐
│ 1. 层次管理 - 窗口重叠、层级关系               │
│ 2. 输入处理 - 键盘、鼠标、触摸                 │
│ 3. 焦点管理 - 哪个窗口接收输入                 │
│ 4. 动画效果 - 开窗、关窗、过渡                 │
│ 5. 渲染优化 - 只渲染可见部分                   │
└─────────────────────────────────────────────────┘
```

---

## 11.1 UI 系统架构

### 整体结构

```
SceneManager
     │
     └─── Scene (场景)
              │
              ├─── WindowLayer (窗口层)
              │        │
              │        ├─── Window_Base (基础窗口)
              │        ├─── Window_Selectable (可选择窗口)
              │        └─── Window_Message (消息窗口)
              │
              └─── Spriteset (精灵集)
                       │
                       └─── 地图/角色精灵
```

### 窗口类型

| 类型 | 说明 | 例子 |
|------|------|------|
| **Window_Base** | 基础窗口，显示静态内容 | 状态栏、标题 |
| **Window_Selectable** | 可选择窗口，支持光标 | 物品列表、技能列表 |
| **Window_Command** | 命令窗口，垂直选项 | 主菜单、战斗菜单 |
| **Window_Message** | 消息窗口，显示对话 | NPC 对话、系统提示 |
| **Window_NumberInput** | 数字输入 | 购买数量、命名 |

---

## 11.2 窗口基类实现

### 为什么需要基类？

```
所有窗口都有共同的特征：
- 位置和大小
- 可见性
- 激活状态
- 背景和边框
- 更新和渲染

把共同代码放在基类，避免重复
```

### 完整的 Window 基类

```javascript
// ============================================
// Window - 所有窗口的基类
// ============================================

function Window(x, y, width, height) {
    // ========== 位置和大小 ==========
    // 为什么用 x, y 而不是 left, top？
    // 因为 Canvas 使用 x, y 坐标系统
    this.x = x;
    this.y = y;
    this.width = width;
    this.height = height;
    
    // ========== 可见性和激活 ==========
    // visible: 是否渲染
    // active: 是否接收输入
    // 为什么分开？因为窗口可能可见但不活动
    this.visible = true;
    this.active = false;
    this.openness = 255;  // 开放度（0-255），用于开窗动画
    
    // ========== 内容区域 ==========
    // padding: 内边距，内容不贴边
    // margin: 外边距，窗口与其他元素的距离
    this.padding = 12;
    this.margin = 4;
    
    // ========== 背景设置 ==========
    // 为什么有 opacity？让窗口可以半透明
    this.backOpacity = 192;     // 背景透明度
    this.contentsOpacity = 255; // 内容透明度
    
    // ========== 窗口边框 ==========
    // 九宫格边框：4角 + 4边 + 中心
    // 这样可以任意拉伸而保持边框形状
    this._windowskin = null;    // 窗口皮肤图片
    
    // ========== 内容画布 ==========
    // 为什么单独一个画布？
    // 便于管理和刷新，不用每次都重绘边框
    this._windowContentsSprite = null;
    
    // ========== 动画状态 ==========
    this._opening = false;   // 正在打开
    this._closing = false;   // 正在关闭
    
    // ========== 初始化 ==========
    this.initialize();
}

// ============================================
// 初始化
// ============================================
Window.prototype.initialize = function() {
    // 创建内容画布
    // 为什么内容区域比窗口小？
    // 因为要留出边框和内边距的空间
    var contentsWidth = this.width - this.padding * 2;
    var contentsHeight = this.height - this.padding * 2;
    
    this._windowContentsSprite = new Sprite();
    this._windowContentsSprite.bitmap = new Bitmap(contentsWidth, contentsHeight);
    this._windowContentsSprite.x = this.padding;
    this._windowContentsSprite.y = this.padding;
    
    // 加载窗口皮肤
    this.loadWindowskin();
};

// ============================================
// 加载窗口皮肤
// ============================================
Window.prototype.loadWindowskin = function() {
    // 窗口皮肤是一张图片，包含：
    // - 背景图案
    // - 边框图案
    // - 光标图案
    // - 其他 UI 元素
    this._windowskin = ImageManager.loadSystem('Window');
};

// ============================================
// 更新
// ============================================
Window.prototype.update = function() {
    // 更新开放度动画
    this.updateOpenness();
    
    // 更新内容透明度
    this._windowContentsSprite.opacity = this.contentsOpacity * this.openness / 255;
    
    // 子类可以重写此方法添加自定义更新逻辑
    this.updateContents();
};

// ============================================
// 更新开放度（开窗/关窗动画）
// ============================================
Window.prototype.updateOpenness = function() {
    if (this._opening) {
        // 打开动画：开放度从 0 增加到 255
        this.openness += 32;  // 每帧增加 32
        if (this.openness >= 255) {
            this.openness = 255;
            this._opening = false;
        }
    }
    
    if (this._closing) {
        // 关闭动画：开放度从 255 减少到 0
        this.openness -= 32;
        if (this.openness <= 0) {
            this.openness = 0;
            this._closing = false;
            this.visible = false;  // 动画完成后隐藏
        }
    }
};

// ============================================
// 更新内容
// ============================================
Window.prototype.updateContents = function() {
    // 子类重写此方法
};

// ============================================
// 打开窗口
// ============================================
Window.prototype.open = function() {
    this.visible = true;
    this._opening = true;
    this._closing = false;
    this.openness = 0;  // 从 0 开始动画
};

// ============================================
// 关闭窗口
// ============================================
Window.prototype.close = function() {
    this._closing = true;
    this._opening = false;
    // 注意：不立即隐藏，等动画完成
};

// ============================================
// 判断是否打开完成
// ============================================
Window.prototype.isOpen = function() {
    return this.visible && this.openness === 255 && !this._opening;
};

// ============================================
// 判断是否关闭完成
// ============================================
Window.prototype.isClosed = function() {
    return !this.visible || this.openness === 0;
};

// ============================================
// 刷新内容
// ============================================
Window.prototype.refresh = function() {
    // 清除现有内容
    this.contents.clear();
    // 子类重写此方法绘制内容
};

// ============================================
// 渲染
// ============================================
Window.prototype.render = function(ctx) {
    if (!this.visible) return;
    
    // 计算透明度
    var opacity = this.openness / 255;
    
    ctx.save();
    ctx.globalAlpha = opacity;
    
    // 1. 渲染背景
    this.drawBackground(ctx);
    
    // 2. 渲染边框
    this.drawFrame(ctx);
    
    // 3. 渲染内容
    this.drawContents(ctx);
    
    ctx.restore();
};

// ============================================
// 绘制背景
// ============================================
Window.prototype.drawBackground = function(ctx) {
    ctx.fillStyle = 'rgba(0, 0, 0, ' + (this.backOpacity / 255) + ')';
    ctx.fillRect(this.x, this.y, this.width, this.height);
};

// ============================================
// 绘制边框
// ============================================
Window.prototype.drawFrame = function(ctx) {
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 2;
    ctx.strokeRect(this.x, this.y, this.width, this.height);
    
    // 内边框（增加层次感）
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.3)';
    ctx.lineWidth = 1;
    ctx.strokeRect(this.x + 2, this.y + 2, this.width - 4, this.height - 4);
};

// ============================================
// 绘制内容
// ============================================
Window.prototype.drawContents = function(ctx) {
    // 将内容画布绘制到窗口上
    if (this._windowContentsSprite && this._windowContentsSprite.bitmap) {
        ctx.drawImage(
            this._windowContentsSprite.bitmap._canvas,
            this.x + this.padding,
            this.y + this.padding
        );
    }
};

// ============================================
// 获取内容画布（便于子类绘制）
// ============================================
Object.defineProperty(Window.prototype, 'contents', {
    get: function() {
        return this._windowContentsSprite ? this._windowContentsSprite.bitmap : null;
    }
});

// ============================================
// 工具方法：在内容区域绘制文本
// ============================================
Window.prototype.drawText = function(text, x, y, maxWidth, align) {
    if (!this.contents) return;
    
    this.contents.fontSize = 16;
    this.contents.textColor = '#ffffff';
    this.contents.drawText(text, x, y, maxWidth || this.contents.width, align || 'left');
};

// ============================================
// 工具方法：绘制计量条
// ============================================
Window.prototype.drawGauge = function(x, y, width, rate, color1, color2) {
    if (!this.contents) return;
    
    var height = 12;
    
    // 背景
    this.contents.fillRect(x, y, width, height, '#333333');
    
    // 填充（渐变）
    var fillWidth = Math.floor(width * rate);
    if (fillWidth > 0) {
        this.contents.gradientFillRect(x, y, fillWidth, height, color1, color2);
    }
    
    // 边框
    this.contents.strokeRect(x, y, width, height, '#ffffff');
};
```

---

## 11.3 可选择窗口

### 为什么需要 Selectable？

```
Window_Base 只能显示内容
Window_Selectable 可以让玩家选择

选择功能需要：
- 光标显示
- 上下左右移动
- 确认和取消
- 触摸/点击支持
```

### 完整实现

```javascript
// ============================================
// Window_Selectable - 可选择窗口
// ============================================
// 继承自 Window_Base，添加选择功能

function Window_Selectable(x, y, width, height) {
    Window.call(this, x, y, width, height);
    
    // ========== 选择相关 ==========
    this.index = -1;           // 当前选中的索引（-1 表示无选择）
    this.maxItems = 0;         // 总项目数
    this.maxCols = 1;          // 每行列数
    this.itemHeight = 32;      // 每项高度
    this.itemWidth = this.contents.width;  // 每项宽度
    
    // ========== 光标设置 ==========
    this.cursorFixed = false;  // 光标是否固定
    this.cursorAll = false;    // 是否全选
    
    // ========== 滚动设置 ==========
    this._scrollY = 0;         // 滚动偏移
    this.topRow = 0;           // 顶部显示的行
    this.maxPageRows = Math.floor(this.height / this.itemHeight);  // 每页行数
    
    // ========== 处理器 ==========
    // 用于回调，当选择改变或确认时通知父场景
    this._handlers = {};
    
    // ========== 光标动画 ==========
    this._cursorRect = new Rectangle(0, 0, 0, 0);
    this._cursorOpacity = 255;
    this._cursorBlinkCount = 0;
}

Window_Selectable.prototype = Object.create(Window.prototype);
Window_Selectable.prototype.constructor = Window_Selectable;

// ============================================
// 更新
// ============================================
Window_Selectable.prototype.update = function() {
    Window.prototype.update.call(this);
    
    // 只有激活时才处理输入
    if (this.active) {
        this.processInput();
    }
    
    // 更新光标动画
    this.updateCursor();
    
    // 确保光标在可见区域
    this.ensureCursorVisible();
};

// ============================================
// 处理输入
// ============================================
Window_Selectable.prototype.processInput = function() {
    // 处理方向键
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
    
    // 处理确认和取消
    if (Input.isTriggered('ok')) {
        this.processOk();
    }
    if (Input.isTriggered('cancel')) {
        this.processCancel();
    }
    
    // 处理触摸/点击
    this.processTouch();
};

// ============================================
// 光标移动
// ============================================
Window_Selectable.prototype.cursorDown = function(wrap) {
    var maxRows = Math.ceil(this.maxItems / this.maxCols);
    var currentRow = Math.floor(this.index / this.maxCols);
    var currentCol = this.index % this.maxCols;
    
    if (currentRow < maxRows - 1) {
        // 移到下一行
        this.select(this.index + this.maxCols);
    } else if (wrap) {
        // 循环到第一行
        this.select(currentCol);
    }
};

Window_Selectable.prototype.cursorUp = function(wrap) {
    var currentRow = Math.floor(this.index / this.maxCols);
    var currentCol = this.index % this.maxCols;
    
    if (currentRow > 0) {
        // 移到上一行
        this.select(this.index - this.maxCols);
    } else if (wrap) {
        // 循环到最后一行
        var maxRows = Math.ceil(this.maxItems / this.maxCols);
        this.select((maxRows - 1) * this.maxCols + currentCol);
    }
};

Window_Selectable.prototype.cursorRight = function(wrap) {
    if (this.maxCols > 1) {
        if (this.index < this.maxItems - 1) {
            this.select(this.index + 1);
        } else if (wrap) {
            this.select(0);
        }
    }
};

Window_Selectable.prototype.cursorLeft = function(wrap) {
    if (this.maxCols > 1) {
        if (this.index > 0) {
            this.select(this.index - 1);
        } else if (wrap) {
            this.select(this.maxItems - 1);
        }
    }
};

// ============================================
// 选择项目
// ============================================
Window_Selectable.prototype.select = function(index) {
    // 检查边界
    if (index < 0) index = -1;
    if (index >= this.maxItems) index = this.maxItems - 1;
    
    var lastIndex = this.index;
    this.index = index;
    
    // 如果选择改变，调用回调
    if (this.index !== lastIndex) {
        this.callHandler('change');
    }
    
    // 更新光标
    this.updateCursor();
};

// ============================================
// 更新光标
// ============================================
Window_Selectable.prototype.updateCursor = function() {
    if (this.index < 0) {
        // 无选择，隐藏光标
        this._cursorRect.set(0, 0, 0, 0);
    } else {
        // 计算光标位置
        var row = Math.floor(this.index / this.maxCols);
        var col = this.index % this.maxCols;
        var x = col * this.itemWidth;
        var y = row * this.itemHeight - this._scrollY;
        
        this._cursorRect.set(x, y, this.itemWidth, this.itemHeight);
    }
    
    // 光标闪烁动画
    this._cursorBlinkCount++;
    if (this._cursorBlinkCount % 30 < 15) {
        this._cursorOpacity = 255;
    } else {
        this._cursorOpacity = 128;
    }
};

// ============================================
// 确保光标可见
// ============================================
Window_Selectable.prototype.ensureCursorVisible = function() {
    if (this.index < 0) return;
    
    var row = Math.floor(this.index / this.maxCols);
    var topRow = this.topRow;
    var bottomRow = topRow + this.maxPageRows - 1;
    
    if (row < topRow) {
        // 光标在上方，向上滚动
        this.topRow = row;
    } else if (row > bottomRow) {
        // 光标在下方，向下滚动
        this.topRow = row - this.maxPageRows + 1;
    }
    
    this._scrollY = this.topRow * this.itemHeight;
};

// ============================================
// 处理确认
// ============================================
Window_Selectable.prototype.processOk = function() {
    if (this.index >= 0 && this.isCurrentItemEnabled()) {
        this.callHandler('ok');
    } else {
        this.playBuzzerSound();  // 播放错误音效
    }
};

// ============================================
// 处理取消
// ============================================
Window_Selectable.prototype.processCancel = function() {
    this.callHandler('cancel');
};

// ============================================
// 注册处理器
// ============================================
Window_Selectable.prototype.setHandler = function(symbol, method) {
    this._handlers[symbol] = method;
};

// ============================================
// 调用处理器
// ============================================
Window_Selectable.prototype.callHandler = function(symbol) {
    if (this._handlers[symbol]) {
        this._handlers[symbol].call(this);
    }
};

// ============================================
// 当前项是否可用
// ============================================
Window_Selectable.prototype.isCurrentItemEnabled = function() {
    return true;  // 子类重写
};

// ============================================
// 刷新
// ============================================
Window_Selectable.prototype.refresh = function() {
    Window.prototype.refresh.call(this);
    this.drawAllItems();
};

// ============================================
// 绘制所有项目
// ============================================
Window_Selectable.prototype.drawAllItems = function() {
    var topIndex = this.topRow * this.maxCols;
    var maxVisibleItems = this.maxPageRows * this.maxCols;
    
    for (var i = 0; i < maxVisibleItems; i++) {
        var index = topIndex + i;
        if (index < this.maxItems) {
            this.drawItem(index);
        }
    }
};

// ============================================
// 绘制单个项目（子类重写）
// ============================================
Window_Selectable.prototype.drawItem = function(index) {
    // 子类必须重写此方法
};

// ============================================
// 渲染（添加光标）
// ============================================
Window_Selectable.prototype.render = function(ctx) {
    Window.prototype.render.call(this, ctx);
    
    // 绘制光标
    if (this.index >= 0) {
        this.drawCursor(ctx);
    }
};

// ============================================
// 绘制光标
// ============================================
Window_Selectable.prototype.drawCursor = function(ctx) {
    var rect = this._cursorRect;
    if (rect.width <= 0 || rect.height <= 0) return;
    
    ctx.save();
    ctx.globalAlpha = this._cursorOpacity / 255;
    ctx.strokeStyle = '#ffffff';
    ctx.lineWidth = 2;
    ctx.strokeRect(
        this.x + this.padding + rect.x,
        this.y + this.padding + rect.y,
        rect.width,
        rect.height
    );
    ctx.restore();
};
```

---

## 11.4 具体窗口示例

### 对话框窗口

```javascript
// ============================================
// Window_Message - 对话框窗口
// ============================================

function Window_Message() {
    Window_Base.call(this, 0, 288, Graphics.width, 160);
    
    this._text = '';           // 要显示的文本
    this._speakerName = '';    // 说话者名字
    this._choices = null;      // 选项列表
    this._textIndex = 0;       // 当前显示到第几个字
    this._textSpeed = 1;       // 打字速度
    this._waitCount = 0;       // 等待计数
    this._pause = false;       // 是否等待玩家按键
}

Window_Message.prototype = Object.create(Window_Base.prototype);

// ============================================
// 设置文本
// ============================================
Window_Message.prototype.setText = function(text) {
    this._text = text;
    this._textIndex = 0;
    this._pause = false;
    this.refresh();
};

// ============================================
// 更新
// ============================================
Window_Message.prototype.update = function() {
    Window_Base.prototype.update.call(this);
    
    if (this._pause) {
        // 等待玩家按键
        if (Input.isTriggered('ok') || TouchInput.isTriggered()) {
            this._pause = false;
            this.onTextFinish();
        }
    } else {
        // 打字机效果
        if (this._textIndex < this._text.length) {
            this._waitCount++;
            if (this._waitCount >= this._textSpeed) {
                this._waitCount = 0;
                this._textIndex++;
                this.refresh();
            }
        } else {
            // 文本显示完毕，等待玩家
            this._pause = true;
        }
    }
};

// ============================================
// 刷新
// ============================================
Window_Message.prototype.refresh = function() {
    this.contents.clear();
    
    // 显示说话者名字
    if (this._speakerName) {
        this.changeTextColor(this.systemColor());
        this.drawText(this._speakerName, 0, 0, 200);
        this.changeTextColor(this.normalColor());
    }
    
    // 显示文本（打字机效果）
    var displayText = this._text.substring(0, this._textIndex);
    this.drawTextEx(displayText, 0, 32);
    
    // 显示继续提示
    if (this._pause) {
        this.drawContinueArrow();
    }
};

// ============================================
// 绘制继续箭头
// ============================================
Window_Message.prototype.drawContinueArrow = function() {
    // 动画闪烁的向下箭头
    var x = this.contents.width - 20;
    var y = this.contents.height - 20;
    
    if (Math.floor(Graphics.frameCount / 15) % 2 === 0) {
        this.drawText('▼', x, y, 20);
    }
};
```

### 状态栏窗口

```javascript
// ============================================
// Window_Status - 状态栏
// ============================================

function Window_Status(x, y) {
    Window_Base.call(this, x, y, 240, 120);
    this._actor = null;
}

Window_Status.prototype = Object.create(Window_Base.prototype);

// ============================================
// 设置角色
// ============================================
Window_Status.prototype.setActor = function(actor) {
    if (this._actor !== actor) {
        this._actor = actor;
        this.refresh();
    }
};

// ============================================
// 刷新
// ============================================
Window_Status.prototype.refresh = function() {
    this.contents.clear();
    
    if (!this._actor) return;
    
    var x = 0;
    var y = 0;
    
    // 绘制头像
    this.drawActorFace(this._actor, x, y, 96);
    
    // 绘制名字和职业
    x = 106;
    this.drawActorName(this._actor, x, y);
    this.drawActorClass(this._actor, x, y + 20);
    
    // 绘制等级
    this.drawActorLevel(this._actor, x, y + 40);
    
    // 绘制 HP/MP 条
    y = 70;
    this.drawActorHp(this._actor, x, y, 120);
    this.drawActorMp(this._actor, x, y + 20, 120);
};

// ============================================
// 绘制角色头像
// ============================================
Window_Status.prototype.drawActorFace = function(actor, x, y, width) {
    // 加载头像图片并绘制
    var faceName = actor.faceName();
    var faceIndex = actor.faceIndex();
    
    var bitmap = ImageManager.loadFace(faceName);
    var pw = width / 4;  // 头像宽度
    var ph = width / 2;  // 头像高度
    var sx = (faceIndex % 4) * pw;
    var sy = Math.floor(faceIndex / 4) * ph;
    
    this.contents.blt(bitmap, sx, sy, pw, ph, x, y, width, width);
};

// ============================================
// 绘制 HP 条
// ============================================
Window_Status.prototype.drawActorHp = function(actor, x, y, width) {
    var rate = actor.hp / actor.mhp;
    
    // 绘制标签
    this.changeTextColor(this.systemColor());
    this.drawText('HP', x, y, 30);
    
    // 绘制计量条
    this.drawGauge(x + 34, y, width - 34, rate, '#ff4444', '#ff8888');
    
    // 绘制数值
    this.changeTextColor(this.normalColor());
    this.drawText(actor.hp + '/' + actor.mhp, x + 34, y, width - 34, 'right');
};

// ============================================
// 绘制 MP 条
// ============================================
Window_Status.prototype.drawActorMp = function(actor, x, y, width) {
    var rate = actor.mp / actor.mmp;
    
    // 绘制标签
    this.changeTextColor(this.systemColor());
    this.drawText('MP', x, y, 30);
    
    // 绘制计量条
    this.drawGauge(x + 34, y, width - 34, rate, '#4444ff', '#8888ff');
    
    // 绘制数值
    this.changeTextColor(this.normalColor());
    this.drawText(actor.mp + '/' + actor.mmp, x + 34, y, width - 34, 'right');
};
```

### 物品列表窗口

```javascript
// ============================================
// Window_ItemList - 物品列表
// ============================================

function Window_ItemList(x, y, width, height) {
    Window_Selectable.call(this, x, y, width, height);
    this._data = [];       // 物品列表
    this._category = 'all'; // 当前分类
}

Window_ItemList.prototype = Object.create(Window_Selectable.prototype);

// ============================================
// 设置分类
// ============================================
Window_ItemList.prototype.setCategory = function(category) {
    if (this._category !== category) {
        this._category = category;
        this.refresh();
    }
};

// ============================================
// 获取物品数量
// ============================================
Object.defineProperty(Window_ItemList.prototype, 'maxItems', {
    get: function() {
        return this._data ? this._data.length : 0;
    }
});

// ============================================
// 获取物品
// ============================================
Window_ItemList.prototype.item = function() {
    return this._data[this.index];
};

// ============================================
// 刷新
// ============================================
Window_ItemList.prototype.refresh = function() {
    this._data = this.makeItemList();
    Window_Selectable.prototype.refresh.call(this);
};

// ============================================
// 生成物品列表
// ============================================
Window_ItemList.prototype.makeItemList = function() {
    var items = $gameParty.items();
    
    // 按分类过滤
    if (this._category !== 'all') {
        items = items.filter(function(item) {
            return this.includesCategory(item);
        }.bind(this));
    }
    
    return items;
};

// ============================================
// 绘制物品
// ============================================
Window_ItemList.prototype.drawItem = function(index) {
    var item = this._data[index];
    if (!item) return;
    
    var rect = this.itemRect(index);
    var x = rect.x;
    var y = rect.y;
    var width = rect.width;
    
    // 绘制图标
    this.drawIcon(item.iconIndex, x, y);
    
    // 绘制名称
    this.changeTextColor(this.normalColor());
    this.drawText(item.name, x + 32, y, width - 32);
    
    // 绘制数量
    var count = $gameParty.numItems(item);
    this.drawText('×' + count, x, y, width, 'right');
};

// ============================================
// 计算项目矩形
// ============================================
Window_ItemList.prototype.itemRect = function(index) {
    var rect = new Rectangle();
    var maxCols = this.maxCols;
    var itemWidth = this.contents.width / maxCols;
    
    rect.width = itemWidth - this.padding;
    rect.height = this.itemHeight;
    rect.x = (index % maxCols) * itemWidth;
    rect.y = Math.floor(index / maxCols) * this.itemHeight - this._scrollY;
    
    return rect;
};
```

---

## 11.5 窗口层管理

```javascript
// ============================================
// WindowLayer - 窗口层，管理多个窗口
// ============================================

function WindowLayer() {
    this._windows = [];      // 所有窗口
    this._zIndex = 0;        // 当前 z-index
}

// ============================================
// 添加窗口
// ============================================
WindowLayer.prototype.addWindow = function(window) {
    this._windows.push(window);
};

// ============================================
// 移除窗口
// ============================================
WindowLayer.prototype.removeWindow = function(window) {
    var index = this._windows.indexOf(window);
    if (index >= 0) {
        this._windows.splice(index, 1);
    }
};

// ============================================
// 更新所有窗口
// ============================================
WindowLayer.prototype.update = function() {
    this._windows.forEach(function(window) {
        window.update();
    });
};

// ============================================
// 渲染所有窗口
// ============================================
WindowLayer.prototype.render = function(ctx) {
    // 按 z-index 排序后渲染
    var sortedWindows = this._windows.slice().sort(function(a, b) {
        return (a.z || 0) - (b.z || 0);
    });
    
    sortedWindows.forEach(function(window) {
        window.render(ctx);
    });
};

// ============================================
// 获取最上层的活动窗口
// ============================================
WindowLayer.prototype.getTopWindow = function() {
    for (var i = this._windows.length - 1; i >= 0; i--) {
        if (this._windows[i].active) {
            return this._windows[i];
        }
    }
    return null;
};
```

---

## 本章小结

| 内容 | 要点 |
|------|------|
| **Window 基类** | 位置、可见性、开放度、刷新 |
| **Selectable** | 光标、选择、滚动、处理器 |
| **消息窗口** | 打字机效果、继续提示 |
| **状态窗口** | 头像、HP/MP 条 |
| **物品列表** | 分类、图标、数量 |
| **窗口层** | 层级管理、排序渲染 |

## 下一章

[第十二章：角色数据系统](./12-character-data.md) - 实现角色属性。
