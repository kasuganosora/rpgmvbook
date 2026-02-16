# UI 开发案例

## 案例 1: 自定义状态窗口

### 完整代码

```javascript
function Window_CustomStatus() {
    this.initialize.apply(this, arguments);
}

Window_CustomStatus.prototype = Object.create(Window_Base.prototype);
Window_CustomStatus.prototype.constructor = Window_CustomStatus;

// 初始化
Window_CustomStatus.prototype.initialize = function(x, y) {
    // 调用父类构造，设置位置和大小
    // 400×200: 足够显示头像(144宽) + 状态信息(剩余空间)
    Window_Base.prototype.initialize.call(this, x, y, 400, 200);
    this._actor = null;  // 先不设置角色，等外部传入
};

// 设置角色
Window_CustomStatus.prototype.setActor = function(actor) {
    // 只有角色变化时才刷新（避免不必要的重绘）
    if (this._actor !== actor) {
        this._actor = actor;
        this.refresh();
    }
};

// 刷新内容
Window_CustomStatus.prototype.refresh = function() {
    // 【关键】必须先清除旧内容！
    // Bitmap 是画布，新内容会叠加在旧内容上
    this.contents.clear();

    if (this._actor) {
        // 绘制头像：左上角 (10, 10)
        // 头像默认 144×144，占窗口左边
        this.drawActorFace(this._actor, 10, 10);

        // 绘制名字：头像右侧
        // x=160: 头像宽度144 + 间距16
        this.drawActorName(this._actor, 160, 10);

        // 绘制等级：名字下方一行
        // y=40: 名字占一行(36px) + 间距
        this.drawActorLevel(this._actor, 160, 40);

        // 绘制HP：等级下方
        // y=70: 40 + 一行(36px) - 6(重叠一点紧凑)
        this.drawActorHp(this._actor, 160, 70, 200);

        // 绘制MP：HP下方
        this.drawActorMp(this._actor, 160, 100, 200);

        // 绘制状态图标：MP下方
        // 图标每行最多画多个，自动排列
        this.drawActorIcons(this._actor, 160, 130);
    }
};
```

### 逐行解析 refresh()

| 代码 | 作用 | 为什么这样写 |
|------|------|--------------|
| `this.contents.clear()` | 清除画布 | 不清除会看到重叠文字 |
| `this.drawActorFace(actor, 10, 10)` | 画头像 | 头像大，放左边合适 |
| `x=160` | 右侧信息起点 | 144(头像宽)+16(间距)=160 |
| `y=40, 70, 100` | 逐行下移 | 每行36px高度 |
| `width=200` | HP/MP条宽度 | 窗口400-160(左边)-40(边距)=200 |

### 使用方法

```javascript
// 在场景中创建
var statusWindow = new Window_CustomStatus(0, 0);
this.addWindow(statusWindow);

// 设置要显示的角色
statusWindow.setActor($gameParty.members()[0]);
```

---

## 案例 2: 自定义命令窗口

```javascript
function Window_BattleCommand() {
    this.initialize.apply(this, arguments);
}

Window_BattleCommand.prototype = Object.create(Window_Command.prototype);
Window_BattleCommand.prototype.constructor = Window_BattleCommand;

// 定义命令列表
Window_BattleCommand.prototype.makeCommandList = function() {
    // addCommand(显示名, 符号, 是否可用)
    this.addCommand('攻击', 'attack', true);
    this.addCommand('魔法', 'magic', this._canUseMagic());  // 根据条件
    this.addCommand('物品', 'item', true);
    this.addCommand('防御', 'guard', true);
    this.addCommand('逃跑', 'escape', BattleManager.canEscape());
};

// 自定义绘制
Window_BattleCommand.prototype.drawItem = function(index) {
    var rect = this.itemRectForText(index);
    var enabled = this.isCommandEnabled(index);

    // 不可用项显示灰色
    this.changePaintOpacity(enabled);
    this.drawText(this.commandName(index), rect.x, rect.y, rect.width, 'center');
    this.changePaintOpacity(true);
};

// 使用
var cmdWindow = new Window_BattleCommand(0, 0);
cmdWindow.setHandler('attack', function() {
    BattleManager.selectNextCommand();
});
cmdWindow.setHandler('magic', function() {
    SceneManager._scene.startSkillSelection();
});
cmdWindow.setHandler('cancel', function() {
    BattleManager.selectPreviousCommand();
});
```

---

## 案例 3: 自定义物品列表

```javascript
function Window_Inventory() {
    this.initialize.apply(this, arguments);
}

Window_Inventory.prototype = Object.create(Window_Selectable.prototype);
Window_Inventory.prototype.constructor = Window_Inventory;

Window_Inventory.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._data = [];  // 存储物品数据
    this.refresh();
};

// 【必须实现】列数
Window_Inventory.prototype.maxCols = function() {
    return 2;  // 两列布局
};

// 【必须实现】项目数
Window_Inventory.prototype.maxItems = function() {
    return this._data.length;
};

// 【必须实现】项目高度
Window_Inventory.prototype.itemHeight = function() {
    return this.lineHeight() * 2;  // 每项两行高（图标+名称/描述）
};

// 【必须实现】绘制单个项目
Window_Inventory.prototype.drawItem = function(index) {
    var item = this._data[index];
    if (!item) return;

    var rect = this.itemRect(index);

    // 图标：左上角
    this.drawIcon(item.iconIndex, rect.x + 4, rect.y + 4);

    // 名称：图标右侧
    this.drawText(item.name, rect.x + 40, rect.y + 4, rect.width - 44);

    // 数量：右对齐
    var count = $gameParty.numItems(item);
    this.drawText('×' + count, rect.x, rect.y + 4, rect.width - 4, 'right');

    // 描述：第二行
    this.changeTextColor(this.systemColor());
    this.drawText(item.description, rect.x + 40, rect.y + this.lineHeight() + 4,
                  rect.width - 44);
};

// 刷新数据
Window_Inventory.prototype.refresh = function() {
    this._data = $gameParty.items().filter(function(item) {
        return $gameParty.numItems(item) > 0;
    });
    this.createContents();  // 重建内容画布
    this.drawAllItems();    // 绘制所有项目
};
```

---

## 调试技巧

### 1. 查看窗口树

```javascript
function dumpWindows(scene) {
    scene._windowLayer.children.forEach(function(win, i) {
        console.log(i, win.constructor.name,
            win.x + ',' + win.y, win.width + '×' + win.height);
    });
}
dumpWindows(SceneManager._scene);
```

### 2. 监控输入

```javascript
var _Input_update = Input.update;
Input.update = function() {
    _Input_update.call(this);
    if (this._latestButton) {
        console.log('[Input]', this._latestButton,
            'time:', this._pressedTime);
    }
};
```

### 3. 可视化点击区域

```javascript
Window_Selectable.prototype.debugDrawHitAreas = function() {
    for (var i = 0; i < this.maxItems(); i++) {
        var rect = this.itemRect(i);
        this.contents.strokeStyle = 'rgba(255,0,0,0.5)';
        this.contents.strokeRect(rect.x, rect.y, rect.width, rect.height);
    }
};
```

---

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 内容不显示 | 没调用 refresh() | 初始化后调用 refresh() |
| 文字重叠 | 没调用 clear() | refresh() 开头清除画布 |
| 点击无响应 | active=false 或没注册处理器 | 设置 active=true，调用 setHandler() |
| 光标位置错 | itemRect 计算错误 | 检查 maxCols/maxItems/itemHeight |
