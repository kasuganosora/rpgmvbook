# 从零实现自定义控件教程

## 目标：实现一个"技能冷却按钮"

```
┌─────────────────────┐
│ [图标] 技能名称     │
│ ████████░░░ CD: 3s  │  ← 冷却进度条 + 倒计时
│ 消耗: 20 MP         │
└─────────────────────┘
```

**功能需求**：
- 显示技能图标、名称、消耗
- 冷却时显示进度条和倒计时
- 点击时触发回调
- 冷却完成前不可用

---

## 第 1 步：选择基类

**问题**：我的控件应该继承哪个类？

```
需求分析：
✓ 需要显示内容（文字、图标、进度条）→ 需要 Window_Base 的绘制方法
✓ 需要点击交互 → 需要 Window_Selectable 的交互方法
✓ 只有一个选项 → 可以用 Window_Selectable (maxItems = 1)

选择：继承 Window_Selectable
```

**为什么不用 Window_Command？**
- Window_Command 用于多个命令的列表
- 我们只有一个按钮，不需要命令列表

**为什么不用 Window_Base？**
- Window_Base 没有交互功能
- 需要自己处理点击检测

---

## 第 2 步：创建基础结构

```javascript
//=============================================================================
// Window_SkillButton.js - 技能冷却按钮控件
//=============================================================================

/*:
 * @plugindesc 技能冷却按钮控件
 * @author YourName
 *
 * @help
 * 使用方法：
 *   var btn = new Window_SkillButton(100, 100, 200, 80);
 *   btn.setSkill($dataSkills[1]);
 *   btn.setHandler('ok', function() { ... });
 */

// 1. 构造函数（RPG Maker 的标准写法）
function Window_SkillButton() {
    this.initialize.apply(this, arguments);
}

// 2. 继承 Window_Selectable
Window_SkillButton.prototype = Object.create(Window_Selectable.prototype);

// 3. 修正 constructor 指向（重要！否则 instanceof 会出错）
Window_SkillButton.prototype.constructor = Window_SkillButton;

// 4. 初始化方法
Window_SkillButton.prototype.initialize = function(x, y, width, height) {
    // 【必须】调用父类初始化
    // 为什么必须？因为父类会设置 _handlers、_index 等重要属性
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);

    // 初始化成员变量
    this._skill = null;       // 当前技能
    this._cooldown = 0;       // 冷却剩余时间（帧数）
    this._maxCooldown = 0;    // 最大冷却时间

    // 初始刷新
    this.refresh();
};
```

**为什么要写 `this.initialize.apply(this, arguments)`？**
- RPG Maker 的惯例，允许子类用 `new Window_SkillButton(x, y, w, h)` 创建
- `apply` 把参数数组传给 initialize

---

## 第 3 步：实现必须的方法

Window_Selectable 要求子类实现以下方法：

```javascript
// 【必须】最大项目数
Window_SkillButton.prototype.maxItems = function() {
    return 1;  // 我们只有一个按钮
};

// 【必须】最大列数
Window_SkillButton.prototype.maxCols = function() {
    return 1;  // 单列
};

// 【可选】项目高度（默认使用 lineHeight）
Window_SkillButton.prototype.itemHeight = function() {
    return this.lineHeight() * 3;  // 3行高：图标+名称 / 进度条 / 消耗
};

// 【必须】绘制单个项目
Window_SkillButton.prototype.drawItem = function(index) {
    if (!this._skill) return;

    var rect = this.itemRect(index);  // 获取项目区域

    // 清除这个区域（避免重叠）
    this.contents.clearRect(rect.x, rect.y, rect.width, rect.height);

    // 第1行：图标 + 名称
    this.drawIcon(this._skill.iconIndex, rect.x + 4, rect.y + 4);
    this.drawText(this._skill.name, rect.x + 40, rect.y + 4, rect.width - 44);

    // 第2行：冷却进度条
    var barY = rect.y + this.lineHeight() + 4;
    this.drawCooldownBar(rect.x + 4, barY, rect.width - 8);

    // 第3行：消耗信息
    var costY = rect.y + this.lineHeight() * 2 + 4;
    this.drawSkillCost(rect.x + 4, costY, rect.width - 8);
};
```

---

## 第 4 步：添加自定义绘制方法

```javascript
// 绘制冷却进度条
Window_SkillButton.prototype.drawCooldownBar = function(x, y, width) {
    var height = 12;
    var rate = this._cooldown > 0 ?
               (this._maxCooldown - this._cooldown) / this._maxCooldown : 1;

    // 背景（深灰色）
    this.contents.fillRect(x, y, width, height, this.textColor(19));

    // 填充（渐变：蓝色到绿色）
    if (rate > 0) {
        var fillWidth = Math.floor(width * rate);
        var color1 = this.textColor(4);  // 蓝色
        var color2 = this.textColor(3);  // 绿色
        this.contents.gradientFillRect(x, y, fillWidth, height, color1, color2);
    }

    // 倒计时文字
    if (this._cooldown > 0) {
        var seconds = Math.ceil(this._cooldown / 60);  // 帧数转秒
        var text = 'CD: ' + seconds + 's';
        this.changeTextColor(this.systemColor());
        this.drawText(text, x, y - 2, width, 'center');
    }
};

// 绘制消耗信息
Window_SkillButton.prototype.drawSkillCost = function(x, y, width) {
    if (!this._skill) return;

    var cost = this._skill.mpCost;
    if (cost > 0) {
        this.changeTextColor(this.systemColor());
        this.drawText('消耗: ', x, y, 60);

        this.changeTextColor(this.mpCostColor());
        this.drawText(cost + ' MP', x + 60, y, width - 60);
    }
};
```

---

## 第 5 步：添加设置方法

```javascript
// 设置技能
Window_SkillButton.prototype.setSkill = function(skill) {
    if (this._skill !== skill) {
        this._skill = skill;
        this._cooldown = 0;
        this._maxCooldown = skill ? skill.meta.cooldown || 300 : 0;  // 默认5秒
        this.refresh();
    }
};

// 开始冷却
Window_SkillButton.prototype.startCooldown = function() {
    if (this._maxCooldown > 0) {
        this._cooldown = this._maxCooldown;
    }
};

// 是否可用
Window_SkillButton.prototype.isReady = function() {
    return this._skill && this._cooldown <= 0;
};
```

---

## 第 6 步：更新逻辑

```javascript
// 重写 update 方法
Window_SkillButton.prototype.update = function() {
    Window_Selectable.prototype.update.call(this);  // 【必须】调用父类

    this.updateCooldown();
};

// 更新冷却
Window_SkillButton.prototype.updateCooldown = function() {
    if (this._cooldown > 0) {
        this._cooldown--;
        this.refresh();  // 每帧刷新（生产环境应优化）

        // 冷却完成时回调
        if (this._cooldown <= 0 && this.isHandled('cooldownEnd')) {
            this.callHandler('cooldownEnd');
        }
    }
};

// 重写 isCurrentItemEnabled（影响是否可以点击）
Window_SkillButton.prototype.isCurrentItemEnabled = function() {
    return this.isReady();
};
```

**注意**：每帧 refresh() 性能不好，实际应该只在冷却开始/结束时刷新。

---

## 第 7 步：完整的 refresh 方法

```javascript
Window_SkillButton.prototype.refresh = function() {
    this.contents.clear();  // 【必须】清除旧内容

    if (this._skill) {
        this.drawItem(0);  // 绘制唯一的项

        // 更新光标状态
        if (this.isReady()) {
            this.showCursor();   // 可用：显示光标
        } else {
            this.hideCursor();   // 冷却中：隐藏光标
        }
    }
};

Window_SkillButton.prototype.showCursor = function() {
    this.setCursorRect(0, 0, this.contents.width, this.itemHeight());
};

Window_SkillButton.prototype.hideCursor = function() {
    this.setCursorRect(0, 0, 0, 0);
};
```

---

## 第 8 步：在场景中使用

```javascript
// 假设在 Scene_Map 中添加技能按钮
Scene_Map.prototype.createSkillButtons = function() {
    this._skillButtonWindow = new Window_SkillButton(10, 10, 200, 80);
    this._skillButtonWindow.setSkill($dataSkills[1]);  // 设置技能

    // 注册点击处理器
    this._skillButtonWindow.setHandler('ok', this.onSkillButtonOk.bind(this));

    // 注册冷却结束处理器
    this._skillButtonWindow.setHandler('cooldownEnd', this.onSkillCooldownEnd.bind(this));

    this.addWindow(this._skillButtonWindow);
};

Scene_Map.prototype.onSkillButtonOk = function() {
    var skill = this._skillButtonWindow._skill;
    var actor = $gameParty.leader();

    // 检查是否可以使用
    if (actor.canUse(skill)) {
        // 使用技能
        actor.useItem(skill);
        BattleManager.invokeAction(actor, actor.currentAction());

        // 开始冷却
        this._skillButtonWindow.startCooldown();

        // 播放音效
        SoundManager.playUseItem();
    } else {
        // 不能使用（MP不足等）
        SoundManager.playBuzzer();
    }
};

Scene_Map.prototype.onSkillCooldownEnd = function() {
    // 冷却完成，可以播放提示音
    SoundManager.playCursor();
};
```

---

## 第 9 步：完整代码总结

```javascript
//=============================================================================
// Window_SkillButton
//=============================================================================
function Window_SkillButton() {
    this.initialize.apply(this, arguments);
}

Window_SkillButton.prototype = Object.create(Window_Selectable.prototype);
Window_SkillButton.prototype.constructor = Window_SkillButton;

Window_SkillButton.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._skill = null;
    this._cooldown = 0;
    this._maxCooldown = 0;
    this.refresh();
};

Window_SkillButton.prototype.maxItems = function() { return 1; };
Window_SkillButton.prototype.maxCols = function() { return 1; };
Window_SkillButton.prototype.itemHeight = function() { return this.lineHeight() * 3; };

Window_SkillButton.prototype.setSkill = function(skill) {
    if (this._skill !== skill) {
        this._skill = skill;
        this._cooldown = 0;
        this._maxCooldown = skill ? Number(skill.meta.cooldown) || 300 : 0;
        this.refresh();
    }
};

Window_SkillButton.prototype.startCooldown = function() {
    if (this._maxCooldown > 0) {
        this._cooldown = this._maxCooldown;
    }
};

Window_SkillButton.prototype.isReady = function() {
    return this._skill && this._cooldown <= 0;
};

Window_SkillButton.prototype.isCurrentItemEnabled = function() {
    return this.isReady();
};

Window_SkillButton.prototype.update = function() {
    Window_Selectable.prototype.update.call(this);
    this.updateCooldown();
};

Window_SkillButton.prototype.updateCooldown = function() {
    if (this._cooldown > 0) {
        this._cooldown--;
        if (this._cooldown === 0 || this._cooldown % 10 === 0) {  // 每10帧刷新一次
            this.refresh();
        }
        if (this._cooldown <= 0 && this.isHandled('cooldownEnd')) {
            this.callHandler('cooldownEnd');
        }
    }
};

Window_SkillButton.prototype.refresh = function() {
    this.contents.clear();
    if (this._skill) {
        this.drawItem(0);
        this.setCursorRect(0, 0,
            this.isReady() ? this.contents.width : 0,
            this.isReady() ? this.itemHeight() : 0);
    }
};

Window_SkillButton.prototype.drawItem = function(index) {
    if (!this._skill) return;
    var rect = this.itemRect(index);

    // 第1行：图标 + 名称
    this.drawIcon(this._skill.iconIndex, rect.x + 4, rect.y + 4);
    this.drawText(this._skill.name, rect.x + 40, rect.y + 4, rect.width - 44);

    // 第2行：冷却进度条
    this.drawCooldownBar(rect.x + 4, rect.y + this.lineHeight() + 4, rect.width - 8);

    // 第3行：消耗
    this.drawSkillCost(rect.x + 4, rect.y + this.lineHeight() * 2 + 4, rect.width - 8);
};

Window_SkillButton.prototype.drawCooldownBar = function(x, y, width) {
    var height = 12;
    var rate = this._cooldown > 0 ?
               (this._maxCooldown - this._cooldown) / this._maxCooldown : 1;

    this.contents.fillRect(x, y, width, height, this.textColor(19));
    if (rate > 0) {
        this.contents.gradientFillRect(x, y, Math.floor(width * rate), height,
                                       this.textColor(4), this.textColor(3));
    }
    if (this._cooldown > 0) {
        this.changeTextColor(this.systemColor());
        this.drawText('CD: ' + Math.ceil(this._cooldown / 60) + 's', x, y - 2, width, 'center');
    }
};

Window_SkillButton.prototype.drawSkillCost = function(x, y, width) {
    if (!this._skill || this._skill.mpCost <= 0) return;
    this.changeTextColor(this.systemColor());
    this.drawText('消耗: ', x, y, 60);
    this.changeTextColor(this.mpCostColor());
    this.drawText(this._skill.mpCost + ' MP', x + 60, y, width - 60);
};
```

---

## 开发控件检查清单

| 检查项 | 说明 | 是否完成 |
|--------|------|----------|
| 选择正确基类 | 根据功能选 Window_Base/Selectable/Command | ☐ |
| 调用父类 initialize | `ParentClass.prototype.initialize.call(this, ...)` | ☐ |
| 实现 maxItems | 返回项目数量 | ☐ |
| 实现 maxCols | 返回列数 | ☐ |
| 实现 drawItem | 绘制单个项目 | ☐ |
| 实现 refresh | 清除 + 重绘 | ☐ |
| 注册处理器 | setHandler('ok', callback) | ☐ |
| 处理启用状态 | isCurrentItemEnabled | ☐ |
| 在场景中创建 | new Window + addWindow | ☐ |

---

## 调试步骤

```javascript
// 1. 在浏览器控制台创建测试
var testBtn = new Window_SkillButton(100, 100, 200, 80);
testBtn.setSkill($dataSkills[1]);
SceneManager._scene.addWindow(testBtn);

// 2. 检查属性
console.log('skill:', testBtn._skill);
console.log('cooldown:', testBtn._cooldown);
console.log('isReady:', testBtn.isReady());

// 3. 手动触发冷却
testBtn.startCooldown();

// 4. 检查光标
console.log('cursorRect:', testBtn._cursorRect);
```
