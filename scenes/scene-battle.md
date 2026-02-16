# Scene_Battle - 战斗场景

## 一、战斗流程状态机

```
init → start → input → turn → action → phase.abort → phase.end → [循环或胜利/失败]
```

| 阶段 | 说明 |
|------|------|
| init | 初始化战斗 |
| start | 战斗开始处理 |
| input | 玩家选择指令 |
| turn | 回合开始 |
| action | 执行行动 |
| abort | 中止战斗 |
| end | 战斗结束 |

## 二、create - 创建战斗界面

```javascript
Scene_Battle.prototype.create = function() {
    Scene_Base.prototype.create.call(this);
    this.createDisplayObjects();
};

Scene_Battle.prototype.createDisplayObjects = function() {
    // 战斗精灵集
    this.createSpriteset();
    
    // 窗口层
    this.createWindowLayer();
    
    // 所有窗口
    this.createAllWindows();
};

Scene_Battle.prototype.createAllWindows = function() {
    this.createLogWindow();       // 战斗日志
    this.createStatusWindow();    // 状态窗口
    this.createPartyCommandWindow(); // 队伍指令（战斗/逃跑）
    this.createActorCommandWindow(); // 角色指令（攻击/技能/防御/物品）
    this.createHelpWindow();      // 帮助窗口
    this.createSkillWindow();     // 技能选择
    this.createItemWindow();      // 物品选择
    this.createActorWindow();     // 角色选择
    this.createEnemyWindow();     // 敌人选择
};
```

## 三、start - 战斗开始

```javascript
Scene_Battle.prototype.start = function() {
    Scene_Base.prototype.start.call(this);
    
    // 初始化战斗状态
    BattleManager.startBattle();
    
    // 淡入
    this.startFadeIn(this.fadeSpeed(), false);
};
```

## 四、update - 主更新循环

```javascript
Scene_Battle.prototype.update = function() {
    var active = this.isActive();
    
    // 更新战斗管理器
    BattleManager.update(active);
    
    // 更新输入处理
    this.updateInput();
    
    Scene_Base.prototype.update.call(this);
};

// BattleManager.update 根据阶段分发
BattleManager.update = function(active) {
    if (!this.isBusy() && !this._eventBusy) {
        switch (this._phase) {
            case 'init':
                this.updateInit();      break;
            case 'start':
                this.updateStart();     break;
            case 'input':
                this.updateInput();     break;
            case 'turn':
                this.updateTurn();      break;
            case 'action':
                this.updateAction();    break;
            case 'turnEnd':
                this.updateTurnEnd();   break;
            case 'abort':
                this.updateAbort();     break;
            case 'end':
                this.updateEnd();       break;
        }
    }
};
```

## 五、输入阶段详解

```javascript
BattleManager.updateInput = function() {
    // 获取当前行动角色
    var actor = this._actorIndex >= 0 ? $gameParty.members()[this._actorIndex] : null;
    
    if (actor) {
        if (actor.isInputting()) {
            // 等待角色输入指令
        } else if (actor.isActing()) {
            // 角色正在行动，处理下一个
            this._actorIndex++;
            this.startInput();
        }
    } else {
        // 所有角色输入完成，开始回合
        this.startTurn();
    }
};

// 玩家选择指令后调用
Scene_Battle.prototype.selectNextCommand = function() {
    BattleManager.selectNextCommand();
};

BattleManager.selectNextCommand = function() {
    this._actorIndex++;
    this.startInput();
};
```

## 六、行动执行

```javascript
BattleManager.updateAction = function() {
    var subject = this._subject;
    var action = subject.currentAction();
    
    if (action) {
        // 执行行动
        action.applyGlobal();  // 应用全局效果
        
        if (action.isForFriend()) {
            this.invokeAction(subject, action.getTargets(), action);
        } else {
            this.invokeAction(subject, action.getTargets(), action);
        }
        
        // 移动到下一个行动
        subject.removeCurrentAction();
        
        if (subject.isAlive()) {
            this._phase = 'action';
        } else {
            this.endAction(subject);
        }
    } else {
        // 行动结束
        this.endAction(subject);
    }
};

// 行动执行细节
Game_Action.prototype.apply = function(target) {
    var result = target.result();
    
    this.subject()->clearResult();
    result.clear();
    result.used = this.testApply(target);
    result.missed = result.used && Math.random() >= this.itemHit(target);
    result.evaded = !result.missed && Math.random() < this.itemEva(target);
    
    if (result.isHit()) {
        // 计算伤害
        var value = this.makeDamageValue(target, result.critical);
        value = this.calcDamageRate(target, value);
        this.executeDamage(target, value);
        
        // 应用状态
        this.item.effects.forEach(function(effect) {
            this.applyItemEffect(target, effect);
        }, this);
    }
};
```

## 七、伤害计算详解

```javascript
Game_Action.prototype.makeDamageValue = function(target, critical) {
    var item = this.item();
    var baseValue = this.evalDamageFormula(target);
    var value = baseValue * this.calcElementRate(target);
    
    // 暴击加成
    if (critical) {
        value *= 3;
    }
    
    // 物理防御
    if (this.isPhysical()) {
        value *= target.pdr;
    }
    
    // 魔法防御
    if (this.isMagical()) {
        value *= target.mdr;
    }
    
    return Math.round(value);
};

// 伤害公式解析
// 例如：a.atk * 4 - b.def * 2
Game_Action.prototype.evalDamageFormula = function(target) {
    var a = this.subject();
    var b = target;
    var v = $gameVariables._data;
    
    try {
        return eval(this.item().damage.formula);
    } catch (e) {
        return 0;
    }
};
```

## 八、战斗结束

```javascript
BattleManager.updateEnd = function() {
    if (this._phase === 'end') {
        if (this.checkBattleEnd()) {
            return;  // 战斗继续
        }
        
        if (this._canEscape) {
            // 逃跑成功
            this.processEscape();
        } else if (this._victory) {
            // 胜利处理
            this.processVictory();
        } else {
            // 失败处理
            this.processDefeat();
        }
        
        // 返回地图
        SceneManager.pop();
    }
};

BattleManager.processVictory = function() {
    // 播放胜利ME
    $gameSystem.battleMeReplay();
    
    // 获取经验和金币
    var exp = $gameTroop.experienceTotal();
    var gold = $gameTroop.goldTotal();
    var items = $gameTroop.makeDropItems();
    
    // 给予队伍
    $gameParty.gainExp(exp);
    $gameParty.gainGold(gold);
    items.forEach(function(item) {
        $gameParty.gainItem(item, 1);
    });
    
    // 显示结果
    this.displayVictory(exp, gold, items);
};
```

## 九、调试案例

### 案例1：查看战斗状态

```javascript
console.log('当前阶段:', BattleManager._phase);
console.log('回合数:', BattleManager._turn);
console.log('当前行动者:', BattleManager._subject ? BattleManager._subject.name() : 'none');
console.log('行动顺序:', BattleManager._actionBattlers.map(b => b.name()));
```

### 案例2：强制胜利

```javascript
BattleManager._victory = true;
BattleManager._phase = 'end';
```

### 案例3：修改敌人属性

```javascript
// 第一个敌人 HP 设为 1
$gameTroop.members()[0]._hp = 1;
$gameTroop.members()[0].refresh();
```

### 案例4：添加敌人

```javascript
// 在战斗中添加敌人
var enemyId = 1;
$gameTroop._enemies.push(new Game_Enemy(enemyId, $gameTroop._enemies.length, 0));
```

### 案例5：跳过战斗动画

```javascript
// 加速战斗
Scene_Battle.prototype.spritesetUpdate = function() {
    this._spriteset.update();
    // 跳过动画等待
    BattleManager._waitCount = 0;
};
```

### 案例6：查看敌人掉落

```javascript
$gameTroop.members().forEach(function(enemy) {
    var drops = enemy.enemy().dropItems;
    console.log(enemy.enemy().name, '掉落:', drops);
});
```
