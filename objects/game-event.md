# Game_Event - 事件对象

## 一、事件触发类型

| 类型 | 常量 | 说明 |
|------|------|------|
| 动作按钮 | 0 | 玩家按确定键 |
| 玩家接触 | 1 | 玩家移动到事件位置 |
| 事件接触 | 2 | 事件移动到玩家位置 |
| 自动执行 | 3 | 条件满足时立即执行（阻塞） |
| 并行处理 | 4 | 条件满足时并行执行（不阻塞） |

## 二、页面切换

```javascript
// 从后往前检查条件，找到第一个符合条件的页面
Game_Event.prototype.findProperPageIndex = function() {
    var pages = this.event().pages;
    for (var i = pages.length - 1; i >= 0; i--) {
        if (this.meetsConditions(pages[i])) {
            return i;
        }
    }
    return -1;
};

// 条件检查
Game_Event.prototype.meetsConditions = function(page) {
    var c = page.conditions;
    if (c.switch1Valid && !$gameSwitches.value(c.switch1Id)) return false;
    if (c.switch2Valid && !$gameSwitches.value(c.switch2Id)) return false;
    if (c.selfSwitchValid) {
        var key = [this._mapId, this._eventId, c.selfSwitchCh];
        if (!$gameSelfSwitches.value(key)) return false;
    }
    return true;
};
```

## 三、解释器核心

```javascript
// 事件指令执行
Game_Interpreter.prototype.executeCommand = function() {
    var cmd = this.currentCommand();
    switch (cmd.code) {
        case 121: return this.command121();  // 控制开关
        case 122: return this.command122();  // 控制变量
        case 201: return this.command201();  // 场所移动
        // ...100+指令
    }
};
```

## 四、调试案例

```javascript
// 查看事件
var e = $gameMap.event(1);
console.log('位置:', e.x, e.y, '触发:', e._trigger);

// 修改独立开关
$gameSelfSwitches.setValue([1, 1, 'A'], true);
$gameMap.event(1).refresh();

// 强制启动事件
$gameMap.event(1).start();

// 监控事件指令
var orig = Game_Interpreter.prototype.executeCommand;
Game_Interpreter.prototype.executeCommand = function() {
    console.log('指令:', this.currentCommand());
    return orig.apply(this, arguments);
};
```
