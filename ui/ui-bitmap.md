# Bitmap 绘图原理

## 1. Bitmap 本质

```javascript
function Bitmap(width, height) {
    // 封装 HTML Canvas
    this._canvas = document.createElement('canvas');
    this._context = this._canvas.getContext('2d');

    // 创建 PIXI 纹理（WebGL 使用）
    this._baseTexture = new PIXI.BaseTexture(this._canvas);
}
```

**为什么封装？** 提供 drawText/drawIcon 等高级方法，自动处理纹理更新。

## 2. 绘制文字

```javascript
Bitmap.prototype.drawText = function(text, x, y, maxWidth, lineHeight, align) {
    var context = this._context;

    // 设置字体
    context.font = this.fontSize + 'px ' + this.fontFace;

    // 对齐方式：'left'/'center'/'right'
    context.textAlign = align || 'left';

    // 计算 Y 坐标（垂直居中）
    // y 是行顶部，需要加上偏移让文字在行中间
    var textY = y + lineHeight * 0.4;

    // 描边（让文字在复杂背景上更清晰）
    if (this.outlineWidth > 0) {
        context.strokeStyle = this.outlineColor;
        context.strokeText(text, x, textY, maxWidth);
    }

    // 填充
    context.fillStyle = this.textColor;
    context.fillText(text, x, textY, maxWidth);

    this._setDirty();  // 标记纹理需要更新
};
```

## 3. 绘制图标

```javascript
Window_Base.prototype.drawIcon = function(iconIndex, x, y) {
    var bitmap = ImageManager.loadSystem('IconSet');

    // IconSet 是 16 列网格，每格 32×32
    var sx = iconIndex % 16 * 32;              // 列位置
    var sy = Math.floor(iconIndex / 16) * 32;  // 行位置

    // 复制图片区域
    this.contents.blt(bitmap, sx, sy, 32, 32, x, y);
};
```

**图标 25 的位置计算**：
- sx = 25 % 16 × 32 = 288
- sy = ⌊25/16⌋ × 32 = 32

## 4. 绘制计量条

```javascript
Window_Base.prototype.drawGauge = function(x, y, width, rate, color1, color2) {
    var gaugeHeight = 6;
    var gaugeY = y + this.lineHeight() - gaugeHeight - 2;  // 底部对齐

    // 1. 背景（深灰色，代表"空"）
    this.contents.fillRect(x, gaugeY, width, gaugeHeight, this.textColor(19));

    // 2. 填充（渐变，代表"当前值"）
    if (rate > 0) {
        var fillWidth = Math.floor(width * rate);
        this.contents.gradientFillRect(x, gaugeY, fillWidth, gaugeHeight,
                                       color1, color2, false);  // false=水平渐变
    }
};
```

**为什么用渐变？** 更有立体感，HP 用红色、MP 用蓝色，视觉直观。

## 5. 绘制角色 HP

```javascript
Window_Base.prototype.drawActorHp = function(actor, x, y, width) {
    // 1. 标签 "HP"
    this.changeTextColor(this.systemColor());
    this.drawText(TextManager.hpA, x, y, 44);

    // 2. 根据比例选颜色
    var rate = actor.hp / actor.mhp;
    var color1 = rate > 0.5 ? this.powerUpColor() :  // 绿色
                 rate > 0.25 ? this.crisisColor() :   // 黄色
                 this.deathColor();                    // 红色

    // 3. 计量条
    this.drawGauge(x + 44, y, width - 44, rate, color1, color1);

    // 4. 数值
    this.changeTextColor(this.normalColor());
    this.drawText(actor.hp + '/' + actor.mhp, x + 44, y, width - 44, 'right');
};
```

**颜色变化原因**：让玩家一眼看出血量状态（健康/受伤/危急）。
