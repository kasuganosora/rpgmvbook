# 常用窗口类

## Window_Command - 命令窗口

用于显示命令列表。

```javascript
Window_Command.prototype.initialize = function(x, y) {
    this.clearCommandList();
    this.makeCommandList();
    var width = this.windowWidth();
    var height = this.windowHeight();
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this.refresh();
    this.select(0);
    this.activate();
};

Window_Command.prototype.makeCommandList = function() {
    // 子类重写
};

Window_Command.prototype.addCommand = function(name, symbol, enabled, ext) {
    this._list.push({
        name: name,
        symbol: symbol,
        enabled: enabled !== false,
        ext: ext || null
    });
};

Window_Command.prototype.drawItem = function(index) {
    var rect = this.itemRectForText(index);
    var align = this.itemTextAlign();
    this.changeTextColor(this.commandColor(this.isCommandEnabled(index)));
    this.drawText(this.commandName(index), rect.x, rect.y, rect.width, align);
};
```

## Window_Help - 帮助窗口

显示帮助文本的单行窗口。

```javascript
Window_Help.prototype.initialize = function(numLines) {
    var width = Graphics.boxWidth;
    var height = this.fittingHeight(numLines || 2);
    Window_Base.prototype.initialize.call(this, 0, 0, width, height);
    this._text = '';
};

Window_Help.prototype.setText = function(text) {
    if (this._text !== text) {
        this._text = text;
        this.refresh();
    }
};

Window_Help.prototype.refresh = function() {
    this.contents.clear();
    this.drawTextEx(this._text, this.textPadding(), 0);
};

Window_Help.prototype.clear = function() {
    this.setText('');
};
```

## Window_Gold - 金钱窗口

显示当前金钱。

```javascript
Window_Gold.prototype.initialize = function(x, y) {
    var width = this.windowWidth();
    var height = this.windowHeight();
    Window_Base.prototype.initialize.call(this, x, y, width, height);
    this.refresh();
};

Window_Gold.prototype.refresh = function() {
    var x = this.textPadding();
    var width = this.contents.width - this.textPadding() * 2;
    this.contents.clear();
    this.drawCurrencyValue(this.value(), this.currencyUnit(), x, 0, width);
};

Window_Gold.prototype.value = function() {
    return $gameParty.gold();
};

Window_Gold.prototype.currencyUnit = function() {
    return TextManager.currencyUnit;
};
```

## Window_Message - 消息窗口

显示对话和消息的窗口。

```javascript
Window_Message.prototype.initialize = function() {
    var width = this.windowWidth();
    var height = this.windowHeight();
    var x = (Graphics.boxWidth - width) / 2;
    var y = Graphics.boxHeight - height;
    Window_Base.prototype.initialize.call(this, x, y, width, height);
    
    this._textState = null;
    this._goldWindow = null;
    this._choiceWindow = null;
    this._numberWindow = null;
    this._itemWindow = null;
    this.openness = 0;
    this.deactivate();
};
```

### 消息处理流程

```javascript
Window_Message.prototype.update = function() {
    this.checkToNotClose();
    Window_Base.prototype.update.call(this);
    
    while (!this.isOpening() && !this.isClosing()) {
        if (this.updateWait()) {
            return;
        } else if (this.updateLoading()) {
            return;
        } else if (this.updateInput()) {
            return;
        } else if (this.updateMessage()) {
            return;
        } else if (this.canStart()) {
            this.startMessage();
        } else {
            this.startInput();
            return;
        }
    }
};

Window_Message.prototype.updateMessage = function() {
    if (this._textState) {
        while (!this.isEndOfText(this._textState)) {
            if (this.needsNewPage(this._textState)) {
                this.newPage(this._textState);
            }
            
            this.updateShowFast();
            this.processCharacter(this._textState);
            
            if (!this._showFast && !this._lineShowFast) {
                break;
            }
            
            if (this.pause || this._waitCount > 0) {
                break;
            }
        }
        
        if (this.isEndOfText(this._textState)) {
            this.onEndOfText();
        }
        
        return true;
    } else {
        return false;
    }
};
```

## Window_BattleLog - 战斗日志

显示战斗信息的窗口。

```javascript
Window_BattleLog.prototype.initialize = function() {
    var width = this.windowWidth();
    var height = this.windowHeight();
    Window_Base.prototype.initialize.call(this, 0, 0, width, height);
    this._methods = [];
    this._waitCount = 0;
    this._waitMode = '';
    this._baseLineStack = [];
    this._spriteset = null;
    this.opacity = 0;  // 透明背景
};
```

### 日志方法

```javascript
Window_BattleLog.prototype.addText = function(text) {
    this._methods.push({ name: 'addText', params: [text] });
};

Window_BattleLog.prototype.push = function(methodName) {
    this._methods.push({ name: 'push', params: [] });
    var args = Array.prototype.slice.call(arguments, 1);
    this._methods.push({ name: methodName, params: args });
};

Window_BattleLog.prototype.popBaseLine = function() {
    this._methods.push({ name: 'popBaseLine', params: [] });
};

Window_BattleLog.prototype.pushBaseLine = function() {
    this._methods.push({ name: 'pushBaseLine', params: [] });
};
```

## Window_ItemList - 物品列表

```javascript
Window_ItemList.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._category = 'none';
    this._data = [];
};

Window_ItemList.prototype.setCategory = function(category) {
    if (this._category !== category) {
        this._category = category;
        this.refresh();
        this.resetScroll();
    }
};

Window_ItemList.prototype.includes = function(item) {
    switch (this._category) {
        case 'item':
            return DataManager.isItem(item) && item.itypeId === 1;
        case 'weapon':
            return DataManager.isWeapon(item);
        case 'armor':
            return DataManager.isArmor(item);
        case 'keyItem':
            return DataManager.isItem(item) && item.itypeId === 2;
        default:
            return false;
    }
};

Window_ItemList.prototype.makeItemList = function() {
    this._data = $gameParty.allItems().filter(function(item) {
        return this.includes(item);
    }, this);
    
    if (this.includes(null)) {
        this._data.push(null);
    }
};
```

## Window_SkillList - 技能列表

```javascript
Window_SkillList.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._actor = null;
    this._stypeId = 0;
    this._data = [];
};

Window_SkillList.prototype.setActor = function(actor) {
    if (this._actor !== actor) {
        this._actor = actor;
        this.refresh();
        this.resetScroll();
    }
};

Window_SkillList.prototype.setStypeId = function(stypeId) {
    if (this._stypeId !== stypeId) {
        this._stypeId = stypeId;
        this.refresh();
        this.resetScroll();
    }
};

Window_SkillList.prototype.includes = function(item) {
    return item && item.stypeId === this._stypeId;
};
```

## Window_EquipSlot - 装备槽

```javascript
Window_EquipSlot.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._actor = null;
    this.refresh();
};

Window_EquipSlot.prototype.setActor = function(actor) {
    if (this._actor !== actor) {
        this._actor = actor;
        this.refresh();
    }
};

Window_EquipSlot.prototype.maxItems = function() {
    return this._actor ? this._actor.equipSlots().length : 0;
};

Window_EquipSlot.prototype.item = function() {
    return this._actor ? this._actor.equips()[this.index()] : null;
};

Window_EquipSlot.prototype.drawItem = function(index) {
    if (this._actor) {
        var rect = this.itemRectForText(index);
        var slotName = this._actor.slotName(index);
        var item = this._actor.equips()[index];
        
        this.changeTextColor(this.systemColor());
        this.changePaintOpacity(this.isEnabled(index));
        this.drawText(slotName, rect.x, rect.y, 138, this.lineHeight());
        this.drawItemName(item, rect.x + 138, rect.y, rect.width - 138);
    }
};
```

## Window_SavefileList - 存档列表

```javascript
Window_SavefileList.prototype.initialize = function(x, y, width, height) {
    Window_Selectable.prototype.initialize.call(this, x, y, width, height);
    this._mode = null;
    this._data = [];
};

Window_SavefileList.prototype.setMode = function(mode) {
    this._mode = mode;
};

Window_SavefileList.prototype.maxItems = function() {
    return DataManager.maxSavefiles();
};

Window_SavefileList.prototype.drawItem = function(index) {
    var id = index + 1;
    var valid = DataManager.isThisGameFile(id);
    var info = DataManager.loadSavefileInfo(id);
    var rect = this.itemRectForText(index);
    
    this.resetTextColor();
    
    if (this._mode === 'load') {
        this.changePaintOpacity(valid);
    }
    
    this.drawFileId(id, rect.x, rect.y);
    
    if (info) {
        this.changePaintOpacity(valid);
        this.drawContents(info, rect, valid);
        this.changePaintOpacity(true);
    }
};

Window_SavefileList.prototype.drawFileId = function(id, x, y) {
    this.drawText(TextManager.file + ' ' + id, x, y, 180);
};

Window_SavefileList.prototype.drawContents = function(info, rect, valid) {
    var bottom = rect.y + rect.height;
    var y = bottom - this.lineHeight();
    
    if (valid) {
        if (info.title) {
            this.drawText(info.title, rect.x + 192, rect.y, rect.width - 192);
        }
        if (info.characters) {
            this.drawPartyCharacters(info.characters, rect.x + 220, bottom - 4);
        }
    }
};
```
