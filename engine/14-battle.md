# 第十四章：战斗系统

## 前言

战斗系统是 RPG 游戏的核心玩法之一，主要有两种设计：

| 类型 | 代表游戏 | 特点 | 适合人群 |
|------|---------|------|---------|
| **回合制** | 宝可梦、最终幻想1-10、RPG Maker | 策略性强、节奏慢 | 喜欢思考的玩家 |
| **即时制** | 塞尔达、黑魂、原神、最终幻想15+ | 反应要求高、节奏快 | 喜欢动作的玩家 |

---

# 第一部分：回合制战斗系统

## 1.1 什么是回合制？

```
核心思想：时间被分割成一个个"回合"，每个回合双方轮流行动

传统回合制：
回合 1: 玩家行动 → 敌人行动
回合 2: 玩家行动 → 敌人行动
...

速度回合制（RPG Maker 采用）：
回合 1: 按速度排序后依次行动
回合 2: ...
```

## 1.2 为什么选择回合制？

**优点**：逻辑简单、容易平衡、存档简单、策略深度高、门砍低
**缺点**：节奏慢、沉浸感差、刷怪容易无聊

## 1.3 战斗流程

```
战斗开始 → 回合开始 → 选择行动 → 执行行动 → 胜负判定
              ↑                              ↓
              ←←←←←← 继续战斗 ←←←←←←←←←←←←←←←
```

## 1.4 核心代码

```javascript
// ============================================
// BattleManager - 回合制战斗管理器
// ============================================

function BattleManager() {
    this._phase = 'none';      // 当前阶段
    this._turn = 0;            // 回合数
    this._party = [];          // 玩家队伍
    this._enemies = [];        // 敌人列表
    this._actionOrder = [];    // 行动顺序
}

// 开始战斗
BattleManager.prototype.startBattle = function(enemyIds) {
    this._phase = 'init';
    this._turn = 0;
    
    // 创建玩家队伍
    this._party = $gameParty.members().map(function(actor) {
        return {
            type: 'actor',
            unit: actor,
            hp: actor.hp,
            mp: actor.mp,
            guarding: false,
            selectedAction: null,
            selectedTarget: null
        };
    });
    
    // 创建敌人
    this._enemies = enemyIds.map(function(enemyId) {
        var data = $dataEnemies[enemyId];
        return {
            type: 'enemy',
            name: data.name,
            hp: data.params[0],
            attack: data.params[2],
            defense: data.params[3],
            agi: data.params[6],
            exp: data.exp,
            gold: data.gold
        };
    });
    
    this.startTurn();
};

// 回合开始
BattleManager.prototype.startTurn = function() {
    this._turn++;
    this._actionOrder = this.calculateActionOrder();
    this._phase = 'selectAction';
    
    // 让所有单位选择行动
    this._actionOrder.forEach(function(battler) {
        if (battler.type === 'enemy') {
            // 敌人 AI 随机选择
            battler.selectedAction = 'attack';
            var aliveParty = this._party.filter(p => p.hp > 0);
            battler.selectedTarget = aliveParty[Math.floor(Math.random() * aliveParty.length)];
        }
    }.bind(this));
};

// 计算行动顺序（按速度排序）
BattleManager.prototype.calculateActionOrder = function() {
    var allUnits = this._party.concat(this._enemies);
    allUnits.sort(function(a, b) {
        var agiA = a.agi || a.unit.agi;
        var agiB = b.agi || b.unit.agi;
        return agiB - agiA;
    });
    return allUnits;
};

// 执行攻击
BattleManager.prototype.executeAttack = function(attacker, target) {
    // 计算伤害
    var atk = attacker.attack || attacker.unit.atk;
    var def = target.defense || target.unit.def;
    var damage = Math.max(1, atk * 2 - def);
    
    // 防御减半
    if (target.guarding) {
        damage = Math.floor(damage * 0.5);
    }
    
    target.hp -= damage;
    console.log(attacker.name || attacker.unit.name, '攻击', target.name, '造成', damage, '伤害');
    
    // 检查死亡
    if (target.hp <= 0) {
        this.checkBattleEnd();
    }
};

// 检查战斗结束
BattleManager.prototype.checkBattleEnd = function() {
    var aliveParty = this._party.filter(p => p.hp > 0);
    var aliveEnemies = this._enemies.filter(e => e.hp > 0);
    
    if (aliveParty.length === 0) return 'defeat';
    if (aliveEnemies.length === 0) return 'victory';
    return 'continue';
};
```

---

# 第二部分：即时制战斗系统

## 2.1 什么是即时制？

```
核心思想：战斗和探索在同一场景，时间持续流动

时间线：
0s   ──────────────────────────────────────►
玩家: [攻击]────[闪避]──[攻击]────[技能]
敌人: ──[攻击]──────[攻击]──[攻击]─────
```

## 2.2 关键概念

| 概念 | 说明 | 作用 |
|------|------|------|
| **攻击冷却** | 攻击后需要等待才能再次攻击 | 防止疯狂点击 |
| **硬直** | 被打中后短暂无法行动 | 让攻击有意义 |
| **无敌帧** | 闪避时短暂不受伤害 | 让闪避有用 |
| **攻击判定框** | 攻击的有效范围 | 精确的碰撞检测 |

## 2.3 核心代码

```javascript
// ============================================
// CombatUnit - 即时制战斗单位
// ============================================

class CombatUnit {
    constructor(data, isPlayer) {
        this.isPlayer = isPlayer;
        this.name = data.name;
        this.hp = data.hp;
        this.maxHp = data.hp;
        this.attack = data.attack;
        this.defense = data.defense;
        this.speed = data.speed || 3;
        
        // 位置
        this.x = 0;
        this.y = 0;
        this.direction = 'right';
        
        // 状态
        this.state = 'idle';  // idle, walk, attack, dodge, hurt, dead
        
        // 战斗参数
        this.attackCooldown = 0;
        this.attackCooldownTime = 500;  // 500ms 冷却
        this.hitstunTime = 0;
        this.invincibleTime = 0;
        this.isAttacking = false;
    }
    
    // 每帧更新
    update(deltaTime) {
        if (this.hp <= 0) {
            this.state = 'dead';
            return;
        }
        
        // 更新冷却
        if (this.attackCooldown > 0) this.attackCooldown -= deltaTime;
        if (this.invincibleTime > 0) this.invincibleTime -= deltaTime;
        if (this.hitstunTime > 0) {
            this.hitstunTime -= deltaTime;
            return;  // 硬直中不更新
        }
    }
    
    // 能否攻击
    canAttack() {
        return this.attackCooldown <= 0 && this.hitstunTime <= 0 && !this.isAttacking;
    }
    
    // 执行攻击
    performAttack(target) {
        if (!this.canAttack()) return false;
        
        this.attackCooldown = this.attackCooldownTime;
        this.isAttacking = true;
        
        // 计算伤害
        var damage = Math.max(1, this.attack - target.defense * 0.5);
        
        // 检查目标是否无敌
        if (!target.isInvincible()) {
            target.takeDamage(damage);
            target.knockback(this.direction, 20);
        }
        
        // 攻击动画结束后重置
        setTimeout(() => {
            this.isAttacking = false;
        }, 200);
        
        return true;
    }
    
    // 受到伤害
    takeDamage(damage) {
        this.hp -= damage;
        this.hitstunTime = 200;  // 200ms 硬直
        
        if (this.hp <= 0) {
            this.hp = 0;
            this.state = 'dead';
        }
    }
    
    // 是否无敌
    isInvincible() {
        return this.invincibleTime > 0;
    }
    
    // 闪避
    dodge() {
        if (this.hitstunTime > 0 || this.invincibleTime > 0) return;
        
        this.invincibleTime = 300;  // 300ms 无敌
        this.state = 'dodge';
        
        // 闪避位移
        var dx = this.direction === 'right' ? 50 : -50;
        this.x += dx;
        
        setTimeout(() => {
            this.state = 'idle';
        }, 300);
    }
    
    // 击退
    knockback(direction, distance) {
        if (this.isInvincible()) return;
        
        var dx = direction === 'right' ? distance : -distance;
        this.x += dx;
    }
}
```

## 2.4 即时制战斗管理器

```javascript
// ============================================
// RealTimeBattleSystem - 即时制战斗系统
// ============================================

class RealTimeBattleSystem {
    constructor() {
        this.player = null;
        this.enemies = [];
        this.isRunning = false;
    }
    
    // 开始战斗
    startBattle(playerData, enemyDataList) {
        this.player = new CombatUnit(playerData, true);
        this.player.x = 100;
        this.player.y = 300;
        
        this.enemies = enemyDataList.map((data, i) => {
            var enemy = new CombatUnit(data, false);
            enemy.x = 500 + i * 80;
            enemy.y = 300;
            return enemy;
        });
        
        this.isRunning = true;
        this.lastTime = performance.now();
        this.gameLoop();
    }
    
    // 游戏循环
    gameLoop() {
        if (!this.isRunning) return;
        
        var currentTime = performance.now();
        var deltaTime = currentTime - this.lastTime;
        this.lastTime = currentTime;
        
        this.update(deltaTime);
        this.render();
        
        if (!this.checkBattleEnd()) {
            requestAnimationFrame(() => this.gameLoop());
        }
    }
    
    // 更新
    update(deltaTime) {
        this.player.update(deltaTime);
        
        // 更新敌人并执行 AI
        this.enemies.forEach(enemy => {
            if (enemy.hp > 0) {
                enemy.update(deltaTime);
                this.updateEnemyAI(enemy);
            }
        });
    }
    
    // 敌人 AI
    updateEnemyAI(enemy) {
        var dx = this.player.x - enemy.x;
        var dy = this.player.y - enemy.y;
        var distance = Math.sqrt(dx * dx + dy * dy);
        
        if (distance > 50) {
            // 移动向玩家
            enemy.x += (dx / distance) * enemy.speed * 0.5;
            enemy.y += (dy / distance) * enemy.speed * 0.5;
        } else if (enemy.canAttack()) {
            // 攻击
            enemy.performAttack(this.player);
        }
    }
    
    // 检查战斗结束
    checkBattleEnd() {
        if (this.player.hp <= 0) {
            this.onDefeat();
            return true;
        }
        
        if (this.enemies.every(e => e.hp <= 0)) {
            this.onVictory();
            return true;
        }
        
        return false;
    }
    
    // 处理玩家输入
    handleInput(input) {
        if (this.player.hp <= 0) return;
        
        // 移动
        if (input.up) this.player.move(0, -1);
        if (input.down) this.player.move(0, 1);
        if (input.left) this.player.move(-1, 0);
        if (input.right) this.player.move(1, 0);
        
        // 攻击
        if (input.attack) this.player.performAttack(this.enemies[0]);
        
        // 闪避
        if (input.dodge) this.player.dodge();
    }
}
```

---

# 第三部分：两种系统对比

## 3.1 技术对比

| 维度 | 回合制 | 即时制 |
|------|--------|--------|
| **实现难度** | 简单 | 复杂 |
| **状态管理** | 离散（回合为单位） | 连续（帧为单位） |
| **碰撞检测** | 不需要 | 必须要 |
| **动画系统** | 简单 | 复杂（需要融合动画） |
| **网络同步** | 容易 | 困难 |

## 3.2 选择指南

```
选择回合制如果：
✅ 团队规模小（1-3人）
✅ 想强调策略性
✅ 目标玩家喜欢思考
✅ 开发时间有限

选择即时制如果：
✅ 有动作游戏开发经验
✅ 想强调操作感
✅ 目标玩家喜欢挑战
✅ 有足够的开发时间
```

---

## 本章小结

1. **回合制**：适合策略游戏，实现简单，RPG Maker 默认采用
2. **即时制**：适合动作游戏，实现复杂，现代游戏常用
3. **选择依据**：团队规模、目标玩家、开发时间

## 下一章

[第十五章：存档系统](./15-save-load.md) - 实现存档功能
