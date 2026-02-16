# Window_Base - 窗口基类

## 一、继承关系

```
PIXI.Container
└── Window (PIXI渲染窗口)
    └── Window_Base (基础功能)
        ├── Window_Selectable (可选择)
        ├── Window_Command (命令列表)
        ├── Window_Help (帮助文字)
        └── ...
```

## 二、核心属性

```javascript
function Window_Base() {
    this.initialize.apply(this, arguments);
}

Window_Base.prototype.initialize = function(x, y, width, height) {
    Window.prototype.initialize.call(this);
    
    // 位置和尺寸
    this.x = x;
    this.y = y;
    this.width = width;
    this.height = height;
    
    // 内边距
    this.padding = 18;  // 内容与边框的间距
    
    // 透明度
    this.opacity = 255;          // 窗口背景
    this.backOpacity = 192;      // 背景不透明度
    this.contentsOpacity = 255;  // 内容不透明度
    
    // 创建内容位图
    this.createContents();
    
    // 初始化色调
    this.updateTone();
};
```

## 三、内容位图

```javascript
Window_Base.prototype.createContents = function() {
    // 内容区域尺寸 = 窗口尺寸 - 边距×2
    var width = this.width - this.padding * 2;
    var height = this.height - this.padding * 2;
    
    this.contents = new Bitmap(width, height);
    // Bitmap 是绘制文字和图像的画布
};

// 绘制后需要重置内容
Window_Base.prototype.resetContents = function() {
    this.contents.clear();
};
```

## 四、绘制文字

### 4.1 基础绘制

```javascript
Window_Base.prototype.drawText = function(text, x, y, maxWidth, align) {
    // text: 文字内容
    // x, y: 绘制位置
    // maxWidth: 最大宽度（用于对齐）
    // align: 'left', 'center', 'right'
    
    this.contents.fontSize = this.standardFontSize();
    this.contents.textColor = this.normalColor();
    
    if (align === 'center') {
        x += (maxWidth - this.textWidth(text)) / 2;
    } else if (align === 'right') {
        x += maxWidth - this.textWidth(text);
    }
    
    this.contents.drawText(text, x, y, maxWidth, this.lineHeight());
};
```

### 4.2 文字颜色

```javascript
Window_Base.prototype.normalColor = function() {
    return '#ffffff';  // 白色
};

Window_Base.prototype.systemColor = function() {
    return '#00b0c0';  // 青色（标签文字）
};

Window_Base.prototype.crisisColor = function() {
    return '#ff6040';  // 红色（危机状态）
};

Window_Base.prototype.deathColor = function() {
    return '#ff0000';  // 深红（死亡）
};

Window_Base.prototype.powerUpColor = function() {
    return '#00ff00';  // 绿色（强化）
};

Window_Base.prototype.powerDownColor = function() {
    return '#ff8080';  // 粉色（弱化）
};

// 使用颜色代码
Window_Base.prototype.textColor = function(n) {
    var colors = [
        '#ffffff', '#00b0c0', '#ff6040', '#ff8080',
        '#00ff00', '#ffff00', '#8080ff', '#ff80ff'
    ];
    return colors[n] || '#ffffff';
};
```

### 4.3 文字绘制案例

```javascript
// 简单文字
this.drawText('Hello', 0, 0, 200, 'left');

// 居中文字
this.drawText('标题', 0, 0, this.contents.width, 'center');

// 系统颜色标签
this.changeTextColor(this.systemColor());
this.drawText('HP', 0, 0, 60);
this.changeTextColor(this.normalColor());
this.drawText('100/100', 70, 0, 100);

// 文字换行
var text = '这是一段很长的文字，会自动换行';
this.drawTextEx(text, 0, 0);
```

## 五、绘制图形

### 5.1 绘制计量条

```javascript
Window_Base.prototype.drawGauge = function(x, y, width, rate, color1, color2) {
    // rate: 0.0 ~ 1.0 的比例
    // color1, color2: 渐变色
    
    var gaugeHeight = 18;
    var fillWidth = Math.floor(width * rate);
    
    // 背景
    this.contents.fillRect(x, y + this.lineHeight() - gaugeHeight - 2, width, gaugeHeight, '#000');
    
    // 渐变填充
    this.contents.gradientFillRect(x, y + this.lineHeight() - gaugeHeight - 2, fillWidth, gaugeHeight, color1, color2);
};
```

### 5.2 绘制图标

```javascript
Window_Base.prototype.drawIcon = function(iconIndex, x, y) {
    // 加载系统图标图像
    var bitmap = ImageManager.loadSystem('IconSet');
    
    // 计算图标在图像中的位置
    var pw = 32;  // 图标宽度
    var ph = 32;  // 图标高度
    var sx = (iconIndex % 16) * pw;  // 每行16个图标
    var sy = Math.floor(iconIndex / 16) * ph;
    
    // 绘制到位图
    this.contents.blt(bitmap, sx, sy, pw, ph, x, y);
};
```

### 5.3 绘制脸图

```javascript
Window_Base.prototype.drawFace = function(faceName, faceIndex, x, y, width, height) {
    // 加载脸图
    var bitmap = ImageManager.loadFace(faceName);
    
    // 脸图格式：4行×2列
    var pw = width || 144;   // 脸图宽度
    var ph = height || 144;  // 脸图高度
    var sx = (faceIndex % 4) * pw;
    var sy = Math.floor(faceIndex / 4) * ph;
    
    this.contents.blt(bitmap, sx, sy, pw, ph, x, y);
};
```

### 5.4 绘制角色行走图

```javascript
Window_Base.prototype.drawCharacter = function(charName, charIndex, x, y) {
    var bitmap = ImageManager.loadCharacter(charName);
    
    var isBig = charName.charAt(0) === '$';
    var pw = isBig ? bitmap.width / 3 : bitmap.width / 12;
    var ph = isBig ? bitmap.height / 4 : bitmap.height / 8;
    
    var row = Math.floor(charIndex / 4);
    var col = charIndex % 4;
    var sx = col * 3 * pw + pw;  // 中间帧
    var sy = row * 4 * ph + ph;  // 下方向
    
    this.contents.blt(bitmap, sx, sy, pw, ph, x - pw / 2, y - ph);
};
```

## 六、行高和字体

```javascript
Window_Base.prototype.lineHeight = function() {
    return 36;  // 默认行高
};

Window_Base.prototype.standardFontSize = function() {
    return 28;  // 默认字号
};

Window_Base.prototype.standardFontFace = function() {
    // 根据语言返回字体
    if (navigator.language === 'ja') {
        return 'GameFont, Yu Gothic, sans-serif';
    }
    return 'GameFont, sans-serif';
};
```

## 七、更新和刷新

```javascript
Window_Base.prototype.update = function() {
    Window.prototype.update.call(this);
    this.updateTone();  // 更新色调
    this.updateOpen();  // 更新打开动画
    this.updateClose(); // 更新关闭动画
};

// 打开/关闭动画
Window_Base.prototype.updateOpen = function() {
    if (this._opening) {
        this.openness += 32;
        if (this.openness >= 255) {
            this.openness = 255;
            this._opening = false;
        }
    }
};

Window_Base.prototype.updateClose = function() {
    if (this._closing) {
        this.openness -= 32;
        if (this.openness <= 0) {
            this.openness = 0;
            this._closing = false;
        }
    }
};

// 调用打开/关闭
Window_Base.prototype.open = function() {
    this._opening = true;
    this._closing = false;
};

Window_Base.prototype.close = function() {
    this._closing = true;
    this._opening = false;
};
```

## 八、调试案例

### 案例1：创建自定义窗口

```javascript
var win = new Window_Base(100, 100, 400, 200);
SceneManager._scene.addChild(win);

// 绘制内容
win.contents.clear();
win.drawText('自定义窗口', 0, 0, 400, 'center');

// 绘制HP条
win.drawGauge(20, 50, 200, 0.7, '#ff4040', '#ff8080');
win.drawText('HP', 20, 50, 50);
win.drawText('70/100', 80, 50, 140, 'right');

// 绘制图标
win.drawIcon(1, 20, 100);
win.drawText('药草', 60, 100, 100);
```

### 案例2：修改窗口样式

```javascript
// 透明窗口
win.opacity = 0;
win.backOpacity = 0;

// 半透明
win.opacity = 128;

// 完全不透明
win.backOpacity = 255;
```

### 案例3：动态更新内容

```javascript
// 每60帧更新一次
var counter = 0;
var updateWindow = function() {
    win.contents.clear();
    win.drawText('计数: ' + counter, 0, 0, 200);
    counter++;
    requestAnimationFrame(updateWindow);
};
updateWindow();
```

### 案例4：获取文字宽度

```javascript
var text = '测试文字';
var width = win.textWidth(text);
console.log('文字宽度:', width);
```

### 案例5：绘制富文本

```javascript
var text = '\\C[2]红色文字\\C[0] 普通文字 \\I[1]带图标';
win.drawTextEx(text, 0, 0);

// 控制代码：
// \C[n] - 颜色
// \I[n] - 图标
// \{ - 放大
// \} - 缩小
// \G - 金币符号
```
