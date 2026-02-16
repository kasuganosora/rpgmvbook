# 输入事件系统

## 1. Input 类设计

```javascript
// 静态类：全局只有一个键盘状态
Input._currentState = {};   // 当前帧按键状态
Input._previousState = {};  // 上一帧按键状态
Input._latestButton = null; // 最近按下的按钮
Input._pressedTime = 0;     // 按键持续时间（帧数）
```

**为什么用两个状态？** 区分"刚按下"和"持续按下"。

## 2. 按键映射

```javascript
Input.keyMapper = {
    13: 'ok',   // Enter
    32: 'ok',   // Space
    90: 'ok',   // Z（传统确认键）

    27: 'escape', // ESC
    88: 'escape', // X（传统取消键）

    37: 'left', 38: 'up', 39: 'right', 40: 'down',  // 方向键
};
```

**映射流程**：浏览器 keyCode → keyMapper → 逻辑键名 → `_currentState`

## 3. 检测方法

```javascript
// 持续按下
Input.isPressed = function(keyName) {
    return !!this._currentState[keyName];
};

// 刚按下（只触发一次）
Input.isTriggered = function(keyName) {
    return this._latestButton === keyName && this._pressedTime === 0;
};

// 重复触发（支持长按）
Input.isRepeated = function(keyName) {
    return (this._latestButton === keyName &&
            (this._pressedTime === 0 ||
             (this._pressedTime >= 24 && this._pressedTime % 6 === 0)));
};

// 长按检测
Input.isLongPressed = function(keyName) {
    return this._latestButton === keyName && this._pressedTime >= 24;
};
```

## 4. 时序图

```
帧:       0    1    2    3  ...  23   24   25   26   27   28
按键:     ─────────────────────────────────────────────────
Pressed:  T    T    T    T  ...  T    T    T    T    T    T
Triggered:T    F    F    F  ...  F    F    F    F    F    F
Repeated: T    F    F    F  ...  F    T    F    F    F    T
LongPress:F    F    F    F  ...  F    T    T    T    T    T
```

**参数含义**：
- 等待 24 帧（0.4秒）后开始重复
- 每 6 帧（0.1秒）触发一次

## 5. TouchInput 类

```javascript
TouchInput._x = 0;          // 游戏内 X 坐标
TouchInput._y = 0;          // 游戏内 Y 坐标
TouchInput._triggered = false;  // 刚按下
TouchInput._cancelled = false;  // 右键
TouchInput._moved = false;      // 移动了
TouchInput._released = false;   // 刚释放
```

**坐标转换**（处理 CSS 缩放）：

```javascript
var canvas = Graphics._canvas;
var rect = canvas.getBoundingClientRect();
var scaleX = canvas.width / rect.width;   // 如 816/1632 = 0.5
var scaleY = canvas.height / rect.height;

this._x = Math.floor((event.pageX - rect.left) * scaleX);
this._y = Math.floor((event.pageY - rect.top) * scaleY);
```

**为什么需要缩放？** 游戏画布可能被 CSS 放大/缩小，点击坐标需要转换回游戏内坐标。
