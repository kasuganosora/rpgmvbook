# 窗口渲染原理

## 1. Window 核心组成

```javascript
Window.prototype.initialize = function() {
    PIXI.Container.call(this);  // 继承 PIXI 容器

    // 为什么需要这些层？每层负责不同视觉效果
    this._windowBackSprite = new Sprite();     // 背景层
    this._windowFrameSprite = new Sprite();    // 边框层
    this._windowContentsSprite = new Sprite(); // 内容层
    this._windowCursorSprite = new Sprite();   // 光标层

    // 渲染顺序（底→顶）：背景 → 边框 → 内容 → 光标
};
```

**分层原因**：
- 光标闪烁不影响内容
- 内容刷新不重绘边框
- 每层独立管理，性能更好

## 2. 窗口皮肤结构

`img/system/Window.png` 是一张 512×512 的图片：

```
┌──────────────┬──────────────┐
│ 边框区       │ 光标/按钮    │  Y: 0-192
├──────────────┼──────────────┤
│ 系统颜色     │ 装饰元素     │  Y: 320+
└──────────────┴──────────────┘
```

**为什么用图片？**
1. 美术可自定义外观
2. 图片复制比代码绘制快
3. 所有窗口自动统一风格

## 3. 九宫格边框

```
┌────┬──────────┬────┐
│ A  │    B     │ C  │   A,C,G,I = 角落（不拉伸）
├────┼──────────┼────┤   B,D,F,H = 边（单向拉伸）
│ D  │    E     │ F  │   E = 中心
├────┼──────────┼────┤
│ G  │    H     │ I  │
└────┴──────────┴────┘
```

**设计原因**：窗口可任意大小，角落永不变形。

## 4. 光标动画

```javascript
Window.prototype._updateCursor = function() {
    var blinkCount = this._animationCount % 40;  // 40帧≈0.67秒周期
    var alpha;

    if (blinkCount < 20) {
        alpha = 0.4 + (blinkCount / 20) * 0.6;  // 0.4 → 1.0
    } else {
        alpha = 1.0 - ((blinkCount - 20) / 20) * 0.6;  // 1.0 → 0.4
    }

    this._cursorSprite.alpha = alpha;
};
```

**为什么最小透明度是 0.4？** 保证光标始终可见，即使在"暗"的半周期。

## 5. 开窗动画

```javascript
Window.prototype.updateOpen = function() {
    this._openness += 8;  // 每帧+8，约0.5秒完成
    // openness 0→255 对应 scaleY 0→1（从中心展开）
};
```

**为什么只缩放 Y？** "从中间展开"比"从小变大"更自然，符合传统 RPG 风格。
