# Window_Selectable - 可选择窗口

## 一、职责

Window_Selectable 是所有菜单窗口的基类：
- 处理光标移动
- 管理选中项
- 处理滚动
- 响应输入

## 二、核心属性

```javascript
function Window_Selectable() {
    this.initialize.apply(this, arguments);
}

Window_Selectable.prototype.initialize = function(x, y, width, height) {
    Window_Base.prototype.initialize.call(this, x, y, width, height);
    
    // 光标相关
    this._index = -1;           // 当前选中索引
    this._cursorFixed = false;  // 光标是否固定
    this._cursorAll = false;    // 是否全选
    
    // 滚动相关
    this._topRow = 0;           // 顶部行号
    this._pageRows = 1;         // 每页行数
    
    // 列设置
    this._maxCols = 1;          // 最大列数
    this._spacing = 8;          // 列间距
    
    // 激活状态
    this._handlers = {};        // 按键处理器
};
```

## 三、光标系统

### 3.1 光标位置计算

```javascript
// 光标矩形（用于绘制光标框）
Window_Selectable.prototype.cursorRect = function() {
    var rect = new Rectangle();
    var index = this.index();
    
    if (index >= 0) {
        rect = this.itemRect(index);
    }
    
    return rect;
};

// 每个项目矩形
Window_Selectable.prototype.itemRect = function(index) {
    var rect = new Rectangle();
    var maxCols = this.maxCols();
    var itemWidth = this.itemWidth();
    var itemHeight = this.itemHeight();
    
    // 计算列和行
    var col = index % maxCols;
    var row = Math.floor(index / maxCols);
    
    rect.x = col * (itemWidth + this._spacing);
    rect.y = row * itemHeight - this._scrollY;
    rect.width = itemWidth;
    rect.height = itemHeight;
    
    return rect;
};
```

### 3.2 光标移动

```javascript
// 光标下移
Window_Selectable.prototype.cursorDown = function(wrap) {
    var index = this.index();
    var maxItems = this.maxItems();
    var maxCols = this.maxCols();
    
    if (index < maxItems - maxCols || wrap) {
        this.select((index + maxCols) % maxItems);
    }
};

// 光标上移
Window_Selectable.prototype.cursorUp = function(wrap) {
    var index = this.index();
    var maxItems = this.maxItems();
    var maxCols = this.maxCols();
    
    if (index >= maxCols || wrap) {
        this.select((index - maxCols + maxItems) % maxItems);
    }
};

// 光标右移
Window_Selectable.prototype.cursorRight = function(wrap) {
    var index = this.index();
    var maxItems = this.maxItems();
    var maxCols = this.maxCols();
    
    if (maxCols >= 2 && (index < maxItems - 1 || wrap)) {
        this.select((index + 1) % maxItems);
    }
};
```

## 四、滚动系统

### 4.1 滚动计算

```javascript
// 顶部行号
Window_Selectable.prototype.topRow = function() {
    return Math.floor(this._scrollY / this.itemHeight());
};

// 设置顶部行号（滚动）
Window_Selectable.prototype.setTopRow = function(row) {
    var maxTopRow = this.maxTopRow();
    this._scrollY = Math.min(row, maxTopRow) * this.itemHeight();
    this._refreshCursor();
};

// 最大顶部行号
Window_Selectable.prototype.maxTopRow = function() {
    return Math.max(0, this.maxRows() - this.pageRows());
};

// 每页行数
Window_Selectable.prototype.pageRows = function() {
    return Math.floor(this.height / this.itemHeight());
};
```

### 4.2 滚动到可见

```javascript
Window_Selectable.prototype.ensureCursorVisible = function() {
    var row = this.row();
    
    if (row < this.topRow()) {
        // 光标在可见区域上方
        this.setTopRow(row);
    } else if (row > this.bottomRow()) {
        // 光标在可见区域下方
        this.setTopRow(row - this.pageRows() + 1);
    }
};

Window_Selectable.prototype.bottomRow = function() {
    return this.topRow() + this.pageRows() - 1;
};
```

## 五、输入处理

### 5.1 update - 每帧检测

```javascript
Window_Selectable.prototype.update = function() {
    Window_Base.prototype.update.call(this);
    this.processCursorMove();  // 光标移动
    this.processHandling();    // 按键处理
    this.processWheel();       // 滚轮
    this.processTouch();       // 触摸/鼠标
};

Window_Selectable.prototype.processCursorMove = function() {
    if (this.isCursorMovable()) {
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
        
        // 快速翻页
        if (Input.isRepeated('pagedown')) {
            this.cursorPagedown();
        }
        if (Input.isRepeated('pageup')) {
            this.cursorPageup();
        }
    }
};
```

### 5.2 按键处理器

```javascript
// 设置处理器
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

// 处理按键
Window_Selectable.prototype.processHandling = function() {
    if (this.isActive()) {
        if (Input.isTriggered('ok')) {
            this.callOkHandler();
        }
        if (Input.isTriggered('cancel')) {
            this.callCancelHandler();
        }
    }
};

Window_Selectable.prototype.callOkHandler = function() {
    if (this.isHandled('ok')) {
        this.callHandler('ok');
    } else {
        this.activate();
    }
};

Window_Selectable.prototype.callCancelHandler = function() {
    if (this.isHandled('cancel')) {
        this.callHandler('cancel');
    }
};
```

## 六、实际案例：物品窗口

```javascript
function Window_ItemList() {
    this.initialize.apply(this, arguments);
}

Window_ItemList.prototype = Object.create(Window_Selectable.prototype);
Window_ItemList.prototype.constructor = Window_ItemList;

// 设置列数
Window_ItemList.prototype.maxCols = function() {
    return 2;
};

// 项目数量 = 物品数量
Window_ItemList.prototype.maxItems = function() {
    return this._data ? this._data.length : 0;
};

// 绘制每个项目
Window_ItemList.prototype.drawItem = function(index) {
    var item = this._data[index];
    if (item) {
        var rect = this.itemRect(index);
        
        // 绘制图标
        this.drawIcon(item.iconIndex, rect.x, rect.y);
        
        // 绘制名称
        this.drawText(item.name, rect.x + 36, rect.y, rect.width - 36);
        
        // 绘制数量（右对齐）
        var count = $gameParty.numItems(item);
        this.drawText('×' + count, rect.x, rect.y, rect.width, 'right');
    }
};

// 刷新内容
Window_ItemList.prototype.refresh = function() {
    this._data = $gameParty.items();  // 获取物品列表
    this.createContents();            // 重建位图
    this.drawAllItems();              // 绘制所有
};

Window_ItemList.prototype.drawAllItems = function() {
    for (var i = 0; i < this.maxItems(); i++) {
        this.drawItem(i);
    }
};
```

## 七、调试案例

### 案例1：查看窗口状态

```javascript
var win = SceneManager._scene._itemWindow;
console.log('索引:', win.index());
console.log('顶行:', win.topRow());
console.log('最大项:', win.maxItems());
console.log('可见行:', win.pageRows());
```

### 案例2：程序化选择

```javascript
// 选择第一个
win.select(0);

// 选择最后一个
win.select(win.maxItems() - 1);

// 取消选择
win.select(-1);

// 滚动到顶部
win.setTopRow(0);

// 滚动到底部
win.setTopRow(win.maxTopRow());
```

### 案例3：添加处理器

```javascript
win.setHandler('ok', function() {
    console.log('选择了:', win.index());
    // 处理选择
});

win.setHandler('cancel', function() {
    console.log('取消了');
    win.deactivate();
});
```

### 案例4：强制刷新

```javascript
// 物品数量改变后刷新
$gameParty.gainItem($dataItems[0], 1);
win.refresh();
```

### 案例5：查看选中项目

```javascript
// 在物品窗口中
var item = win.item();
console.log('选中物品:', item.name);
console.log('图标:', item.iconIndex);
console.log('价格:', item.price);
```
