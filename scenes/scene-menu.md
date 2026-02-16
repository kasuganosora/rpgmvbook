# Scene_Menu - 菜单场景

## 职责

显示游戏主菜单，提供物品、技能、装备、状态、存档等子功能入口。

## 核心组件

```javascript
Scene_Menu.prototype.create = function() {
    Scene_MenuBase.prototype.create.call(this);
    this.createCommandWindow();    // 命令窗口
    this.createGoldWindow();        // 金钱窗口
    this.createStatusWindow();      // 状态窗口
};
```

## 布局结构

```
┌─────────────────────────────────────┐
│                                     │
│  ┌─────────┐  ┌─────────────────┐  │
│  │ 物品    │  │                 │  │
│  │ 技能    │  │   角色状态      │  │
│  │ 装备    │  │   头像          │  │
│  │ 状态    │  │   HP/MP/EXP     │  │
│  │ 队形    │  │                 │  │
│  │ 选项    │  │                 │  │
│  │ 存档    │  └─────────────────┘  │
│  │ 结束游戏│                        │
│  └─────────┘  ┌─────────────────┐  │
│               │ G: 10000        │  │
│               └─────────────────┘  │
└─────────────────────────────────────┘
```

## 命令窗口

```javascript
Scene_Menu.prototype.createCommandWindow = function() {
    this._commandWindow = new Window_MenuCommand(0, 0);
    
    this._commandWindow.setHandler('item', this.commandItem.bind(this));
    this._commandWindow.setHandler('skill', this.commandPersonal.bind(this));
    this._commandWindow.setHandler('equip', this.commandPersonal.bind(this));
    this._commandWindow.setHandler('status', this.commandPersonal.bind(this));
    this._commandWindow.setHandler('formation', this.commandFormation.bind(this));
    this._commandWindow.setHandler('options', this.commandOptions.bind(this));
    this._commandWindow.setHandler('save', this.commandSave.bind(this));
    this._commandWindow.setHandler('gameEnd', this.commandGameEnd.bind(this));
    this._commandWindow.setHandler('cancel', this.popScene.bind(this));
    
    this.addWindow(this._commandWindow);
};
```

### Window_MenuCommand

```javascript
Window_MenuCommand.prototype.makeCommandList = function() {
    this.addMainCommands();
    this.addFormationCommand();
    this.addOriginalCommands();
    this.addOptionsCommand();
    this.addSaveCommand();
    this.addGameEndCommand();
};

Window_MenuCommand.prototype.addMainCommands = function() {
    this.addCommand(TextManager.item, 'item', true);
    this.addCommand(TextManager.skill, 'skill', this.isSkillEnabled());
    this.addCommand(TextManager.equip, 'equip', this.isEquipEnabled());
    this.addCommand(TextManager.status, 'status', this.isStatusEnabled());
};
```

## 金钱窗口

```javascript
Scene_Menu.prototype.createGoldWindow = function() {
    this._goldWindow = new Window_Gold(0, 0);
    this._goldWindow.y = Graphics.boxHeight - this._goldWindow.height;
    this.addWindow(this._goldWindow);
};
```

## 状态窗口

```javascript
Scene_Menu.prototype.createStatusWindow = function() {
    var wx = this._commandWindow.width;
    var wy = 0;
    var ww = Graphics.boxWidth - wx;
    var wh = Graphics.boxHeight;
    
    this._statusWindow = new Window_MenuStatus(wx, wy, ww, wh);
    this._statusWindow.setHandler('ok', this.onPersonalOk.bind(this));
    this._statusWindow.setHandler('cancel', this.onPersonalCancel.bind(this));
    this.addWindow(this._statusWindow);
};
```

## 命令处理

### 物品

```javascript
Scene_Menu.prototype.commandItem = function() {
    SceneManager.push(Scene_Item);
};
```

### 个人命令 (技能/装备/状态)

```javascript
Scene_Menu.prototype.commandPersonal = function() {
    this._statusWindow.select(0);
    this._statusWindow.activate();
    this._statusWindow.setHandler('ok', this.onPersonalOk.bind(this));
    this._statusWindow.setHandler('cancel', this.onPersonalCancel.bind(this));
};

Scene_Menu.prototype.onPersonalOk = function() {
    switch (this._commandWindow.currentSymbol()) {
        case 'skill':
            SceneManager.push(Scene_Skill);
            break;
        case 'equip':
            SceneManager.push(Scene_Equip);
            break;
        case 'status':
            SceneManager.push(Scene_Status);
            break;
    }
};

Scene_Menu.prototype.onPersonalCancel = function() {
    this._statusWindow.deselect();
    this._commandWindow.activate();
};
```

### 队形

```javascript
Scene_Menu.prototype.commandFormation = function() {
    this._statusWindow.select(0);
    this._statusWindow.activate();
    this._statusWindow.setFormationMode(true);
};
```

### 选项/存档/结束

```javascript
Scene_Menu.prototype.commandOptions = function() {
    SceneManager.push(Scene_Options);
};

Scene_Menu.prototype.commandSave = function() {
    SceneManager.push(Scene_Save);
};

Scene_Menu.prototype.commandGameEnd = function() {
    SceneManager.push(Scene_GameEnd);
};
```

## Scene_MenuBase

菜单场景的基类，提供：
- 角色选择
- 背景图像
- 帮助窗口

```javascript
Scene_MenuBase.prototype.create = function() {
    Scene_Base.prototype.create.call(this);
    this.createBackground();
    this.updateActor();
    this.createWindowLayer();
};

Scene_MenuBase.prototype.actor = function() {
    return $gameParty.menuActor();
};

Scene_MenuBase.prototype.updateActor = function() {
    this._actor = $gameParty.menuActor();
};

Scene_MenuBase.prototype.onActorChange = function() {
    this.updateActor();
    this._statusWindow.refresh();
};
```

## 快捷键

```javascript
// 通过 Window_MenuCommand 类方法
Window_MenuCommand.initCommandPosition = function() {
    this._lastCommandSymbol = null;
};

Window_MenuCommand.lastCommandSymbol = function() {
    return this._lastCommandSymbol;
};
```
