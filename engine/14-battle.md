# 第十四章：战斗系统

## 回合制战斗流程

```
1. 战斗开始
   ↓
2. 选择行动（攻击/技能/物品/逃跑）
   ↓
3. 执行行动（计算伤害、播放动画）
   ↓
4. 检查胜负
   ↓
5. 敌人行动
   ↓
6. 检查胜负
   ↓
7. 回到步骤 2（循环直到战斗结束）
```

---

## 战斗数据

```javascript
function BattleManager() {
    this.phase = 'none';       // none, select, action, enemy, end
    this.party = [];           // 玩家队伍
    this.enemies = [];         // 敌人列表
    this.currentActor = 0;     // 当前行动者
    this.turn = 0;             // 回合数
}

// 开始战斗
BattleManager.prototype.startBattle = function(enemies) {
    this.enemies = enemies;
    this.party = [$gamePlayer];  // 简化：只有一个玩家
    this.phase = 'select';
    this.currentActor = 0;
    this.turn = 1;

    console.log('战斗开始！');
};

// 选择行动
BattleManager.prototype.selectAction = function(action, target) {
    var actor = this.party[this.currentActor];
    actor.battleAction = action;
    actor.battleTarget = target;

    this.currentActor++;
    if (this.currentActor >= this.party.length) {
        this.phase = 'action';
        this.executeActions();
    }
};

// 执行行动
BattleManager.prototype.executeActions = function() {
    var self = this;

    // 玩家攻击
    var player = this.party[0];
    var target = this.enemies[0];

    var damage = this.calculateDamage(player, target);
    target.hp -= damage;

    console.log(player.name + ' 攻击 ' + target.name + '，造成 ' + damage + ' 伤害！');

    // 检查敌人是否死亡
    if (target.hp <= 0) {
        console.log(target.name + ' 被击败了！');
        this.endBattle(true);
        return;
    }

    // 敌人反击
    setTimeout(function() {
        self.enemyTurn();
    }, 1000);
};

// 敌人回合
BattleManager.prototype.enemyTurn = function() {
    var enemy = this.enemies[0];
    var target = this.party[0];

    var damage = this.calculateDamage(enemy, target);
    target.hp -= damage;

    console.log(enemy.name + ' 攻击 ' + target.name + '，造成 ' + damage + ' 伤害！');

    // 检查玩家是否死亡
    if (target.hp <= 0) {
        console.log(target.name + ' 被击败了...');
        this.endBattle(false);
        return;
    }

    // 下一回合
    this.turn++;
    this.currentActor = 0;
    this.phase = 'select';
};

// 计算伤害
BattleManager.prototype.calculateDamage = function(attacker, defender) {
    var baseDamage = attacker.attack * 2 - defender.defense;
    var variance = Math.floor(Math.random() * 5) - 2;  // -2 到 +2
    return Math.max(1, baseDamage + variance);
};

// 结束战斗
BattleManager.prototype.endBattle = function(victory) {
    this.phase = 'end';

    if (victory) {
        var exp = 0;
        var gold = 0;

        for (var i = 0; i < this.enemies.length; i++) {
            exp += this.enemies[i].exp;
            gold += this.enemies[i].gold;
        }

        console.log('战斗胜利！');
        console.log('获得 ' + exp + ' 经验值，' + gold + ' 金币');

        $gamePlayer.gainExp(exp);
        $gameParty.gold += gold;
    } else {
        console.log('战斗失败...');
    }
};
```

---

## 敌人数据

```javascript
var EnemyDatabase = {
    1: { name: '史莱姆', hp: 30, attack: 5, defense: 2, exp: 10, gold: 5 },
    2: { name: '哥布林', hp: 50, attack: 8, defense: 3, exp: 20, gold: 15 },
    3: { name: '骷髅', hp: 40, attack: 10, defense: 5, exp: 25, gold: 20 }
};

function createEnemy(enemyId) {
    var data = EnemyDatabase[enemyId];
    return {
        name: data.name,
        hp: data.hp,
        maxHp: data.hp,
        attack: data.attack,
        defense: data.defense,
        exp: data.exp,
        gold: data.gold
    };
}
```

---

## 本章小结

1. **回合制**：玩家先行动，敌人后行动
2. **伤害公式**：攻击力 × 2 - 防御力
3. **胜负判定**：HP ≤ 0 时失败

## 下一章

[第十五章：存档系统](./15-save-load.md) - 实现存档功能
