# BattleManager - 战斗管理器

## 一、战斗流程状态机

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
    init → start → input → turn → action → turnEnd ──────┘
                      │                    │
                      │                    ▼
                      │              abort/end
                      │                    │
                      ▼                    ▼
                  [战斗结束]          [胜利/失败/逃跑]
```

## 二、核心属性

```javascript
BattleManager._phase = 'init';     // 当前阶段
BattleManager._canEscape = false;  // 能否逃跑
BattleManager._canLose = false;    // 能否失败
BattleManager._turn = 0;           // 回合数
BattleManager._actorIndex = 0;     // 当前角色索引
BattleManager._subject = null;     // 当前行动者
BattleManager._actionBattlers = []; // 行动顺序列表
```

## 三、setup - 初始化战斗

```javascript
BattleManager.setup = function(troopId, canEscape, canLose) {
    // 初始化
    this.initMembers();
    
    // 设置敌群
    $gameTroop.setup(troopId);
    
    // 设置逃跑/失败选项
    this._canEscape = canEscape;
    this._canLose = canLose;
    
    // 设置阶段
    this._phase = 'init';
};

BattleManager.initMembers = function() {
    this._phase = '';
    this._canEscape = false;
    this._canLose = false;
    this._battleTV = false;
    this._preemptive = false;  // 先制攻击
    this._surprise = false;    // 偷袭
    this._actorIndex = -1;
    this._actionBattlers = [];
    this._subject = null;
    this._action = null;
    this._targets = [];
    this._logWindow = null;
    this._spriteset = null;
    this._escapeRatio = 0;
    this._troopId = 0;
    this._turn = 0;
};
```

## 四、startBattle - 战斗开始

```javascript
BattleManager.startBattle = function() {
    this._phase = 'start';
    
    // 检测先制/偷袭
    this._preemptive = (Math.random() < 0.05);
    this._surprise = (Math.random() < 0.05 && !this._preemptive);
    
    // 播放BGM
    $gameSystem.battleMeStop();
    $gameSystem.battleBgmPlay();
    
    // 初始逃跑率
    this._escapeRatio = 0.5;
};
```

## 五、行动顺序排序

```javascript
BattleManager.makeActionOrders = function() {
    var battlers = [];
    
    // 添加敌人
    $gameTroop.members().forEach(function(enemy) {
        if (enemy.isAlive()) {
            battlers.push(enemy);
        }
    });
    
    // 添加角色
    $gameParty.members().forEach(function(actor) {
        if (actor.isAlive()) {
            battlers.push(actor);
        }
    });
    
    // 按速度排序
    battlers.sort(function(a, b) {
        return b.speed() - a.speed();
    });
    
    this._actionBattlers = battlers;
};

// 速度计算
Game_Battler.prototype.speed = function() {
    return this._speed + Math.randomInt(5);
};
```

## 六、输入阶段详解

```javascript
BattleManager.startInput = function() {
    this._phase = 'input';
    this._actorIndex = -1;
    
    // 清除角色的行动
    $gameParty.members().forEach(function(actor) {
        actor.clearActions();
    });
    
    // 选择第一个角色
    this.selectNextCommand();
};

BattleManager.selectNextCommand = function() {
    do {
        this._actorIndex++;
        if (this._actorIndex >= $gameParty.size()) {
            // 所有角色输入完成，开始回合
            this.startTurn();
            return;
        }
    } while (!this.actor().canInput());
    
    // 等待角色输入
};

BattleManager.selectPreviousCommand = function() {
    do {
        this._actorIndex--;
        if (this._actorIndex < 0) {
            return;
        }
    } while (!this.actor().canInput());
};
```

## 七、回合执行

```javascript
BattleManager.startTurn = function() {
    this._phase = 'turn';
    this._turn++;
    
    // 生成行动顺序
    this.makeActionOrders();
    
    // 触发回合开始事件
    $gameTroop.increaseTurn();
    this._performedBattlers = [];
};

BattleManager.updateTurn = function() {
    if (this._actionBattlers.length > 0) {
        // 获取下一个行动者
        var battler = this._actionBattlers.shift();
        
        if (battler.isAlive()) {
            this._subject = battler;
            this.startAction();
        } else {
            this.endAction();
        }
    } else {
        // 所有行动完成
        this._phase = 'turnEnd';
    }
};
```

## 八、行动执行

```javascript
BattleManager.startAction = function() {
    var subject = this._subject;
    var action = subject.currentAction();
    
    if (action) {
        // 设置行动对象
        this._action = action;
        
        // 获取目标
        this._targets = action.makeTargets();
        
        // 显示行动
        this._logWindow.startAction(subject, action, this._targets);
        
        this._phase = 'action';
    } else {
        // 没有行动，结束
        subject.removeCurrentAction();
        this.endAction();
    }
};

BattleManager.updateAction = function() {
    var subject = this._subject;
    var action = this._action;
    var targets = this._targets;
    
    if (targets.length > 0) {
        // 对每个目标执行行动
        this.invokeAction(subject, targets.shift(), action);
    } else {
        // 所有目标处理完毕
        subject.removeCurrentAction();
        this.endAction();
    }
};

BattleManager.invokeAction = function(subject, target, action) {
    // 应用行动
    action.apply(target);
    
    // 显示结果
    this._logWindow.push('popup', target, action.result());
    
    // 触发反击
    if (action.isPhysical() && target.isAlive()) {
        this.invokeCounterAttack(subject, target);
    }
};
```

## 九、伤害计算详解

```javascript
// Game_Action.prototype.apply
Game_Action.prototype.apply = function(target) {
    var result = target.result();
    result.clear();
    
    // 判断是否命中
    result.used = this.testApply(target);
    result.missed = result.used && Math.random() >= this.itemHit(target);
    result.evaded = !result.missed && Math.random() < this.itemEva(target);
    result.physical = this.isPhysical();
    result.drain = this.isDrain();
    
    if (result.isHit()) {
        // 计算伤害
        var value = this.makeDamageValue(target, result.critical);
        this.executeDamage(target, value);
        
        // 应用效果
        this.item().effects.forEach(function(effect) {
            this.applyItemEffect(target, effect);
        }, this);
    }
};

// 伤害公式计算
Game_Action.prototype.makeDamageValue = function(target, critical) {
    var item = this.item();
    var a = this.subject();  // 攻击者
    var b = target;          // 防御者
    var v = $gameVariables._data;
    
    // 执行公式
    var value = eval(item.damage.formula);
    
    // 元素修正
    value *= this.calcElementRate(target);
    
    // 暴击
    if (critical) {
        value *= 3;
    }
    
    return Math.round(value);
};
```

**伤害公式示例**：
```javascript
// 物理攻击
"a.atk * 4 - b.def * 2"

// 魔法攻击
"a.mat * 4 - b.mdf * 2"

// 固定伤害
"100"

// 基于HP
"b.mhp / 4"

// 随机
"a.atk * (1 + Math.random())"
```

## 十、战斗结束

```javascript
BattleManager.checkBattleEnd = function() {
    if (this._phase !== 'end') {
        // 全灭敌人 = 胜利
        if ($gameTroop.isAllDead()) {
            this._victory = true;
            this._phase = 'end';
            return true;
        }
        
        // 全灭角色 = 失败
        if ($gameParty.isAllDead()) {
            this._defeat = true;
            this._phase = 'end';
            return true;
        }
    }
    return false;
};

BattleManager.processVictory = function() {
    $gameParty.removeBattleStates();
    $gameParty.performVictory();
    
    // 播放胜利ME
    $gameSystem.battleMePlay();
    
    // 获取奖励
    var exp = $gameTroop.experienceTotal();
    var gold = $gameTroop.goldTotal();
    var drops = $gameTroop.makeDropItems();
    
    // 给予奖励
    $gameParty.gainExp(exp);
    $gameParty.gainGold(gold);
    drops.forEach(function(item) {
        $gameParty.gainItem(item, 1);
    });
    
    // 显示结果
    this.displayVictory(exp, gold, drops);
};

BattleManager.processDefeat = function() {
    $gameParty.removeBattleStates();
    
    // 失败是否允许
    if (this._canLose) {
        // 允许失败，回到地图
        $gameParty.reviveBattleMembers();
        SceneManager.pop();
    } else {
        // 游戏结束
        SceneManager.goto(Scene_Gameover);
    }
};
```

## 十一、调试案例

### 案例1：查看战斗状态

```javascript
console.log('阶段:', BattleManager._phase);
console.log('回合:', BattleManager._turn);
console.log('当前行动者:', BattleManager._subject ? BattleManager._subject.name() : 'none');
console.log('行动顺序:', BattleManager._actionBattlers.map(b => b.name()));
```

### 案例2：强制胜利

```javascript
// 设置胜利标志
BattleManager._victory = true;
BattleManager._phase = 'end';
```

### 案例3：秒杀所有敌人

```javascript
$gameTroop.members().forEach(function(enemy) {
    enemy._hp = 0;
    enemy.refresh();
});
```

### 案例4：无限HP

```javascript
$gameParty.members().forEach(function(actor) {
    actor._hp = actor.mhp;
});
```

### 案例5：查看敌人掉落

```javascript
$gameTroop.members().forEach(function(enemy) {
    console.log(enemy.enemy().name);
    console.log('  经验:', enemy.exp());
    console.log('  金币:', enemy.gold());
    console.log('  掉落:', enemy.enemy().dropItems);
});
```

### 案例6：强制添加行动

```javascript
var action = new Game_Action($gameActors.actor(1));
action.setAttack();
$gameActors.actor(1).setAction(0, action);
```
