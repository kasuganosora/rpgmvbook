# AI 基础概念

## 本章目标

理解游戏 AI 的基本原理，学会设计简单的 AI 行为。

## 最简单的 AI：随机行动

```javascript
// ============================================
// 最简单的 AI：随机选择行动
// ============================================
// 适用场景：史莱姆等低级怪物
// 特点：完全随机，没有策略

function simpleAI(enemy, player) {
    var actions = ['attack', 'defend', 'doNothing'];
    var action = actions[Math.floor(Math.random() * actions.length)];
    
    switch(action) {
        case 'attack':
            enemy.attack(player);
            break;
        case 'defend':
            enemy.defend();
            break;
        case 'doNothing':
            // 什么都不做
            break;
    }
}
```

**为什么这样做？**
- 实现简单，不会出错
- 低级怪物不需要聪明
- 给新手玩家练习机会

## 条件 AI：根据情况行动

```javascript
// ============================================
// 条件 AI：根据条件选择行动
// ============================================
// 适用场景：中级怪物
// 特点：有一定策略，但不复杂

function conditionalAI(enemy, player) {
    // 条件1：HP 低于 20%，逃跑或治疗
    if (enemy.hp / enemy.maxHp < 0.2) {
        if (enemy.hasPotion()) {
            return enemy.usePotion();
        } else {
            return enemy.flee();
        }
    }
    
    // 条件2：玩家 HP 低于 30%，强力攻击
    if (player.hp / player.maxHp < 0.3) {
        return enemy.powerAttack(player);
    }
    
    // 条件3：随机攻击或防御
    if (Math.random() < 0.7) {
        return enemy.attack(player);
    } else {
        return enemy.defend();
    }
}
```

**为什么用条件？**
- 让 AI 看起来"聪明"
- 行为可预测，玩家能学习
- 增加战斗策略性

## 权重 AI：根据权重选择

```javascript
// ============================================
// 权重 AI：每个行动有不同权重
// ============================================
// 适用场景：高级怪物、BOSS
// 特点：灵活，可调节难度

function weightedAI(enemy, player) {
    // 定义行动及其权重
    var actions = [
        { action: 'attack', weight: 50 },
        { action: 'skill1', weight: 30 },
        { action: 'skill2', weight: 15 },
        { action: 'defend', weight: 5 }
    ];
    
    // 根据情况调整权重
    if (enemy.hp / enemy.maxHp < 0.3) {
        // HP 低时更倾向于防御
        actions.find(a => a.action === 'defend').weight = 30;
        actions.find(a => a.action === 'attack').weight = 30;
    }
    
    // 计算总权重
    var totalWeight = actions.reduce((sum, a) => sum + a.weight, 0);
    
    // 随机选择
    var random = Math.random() * totalWeight;
    var current = 0;
    
    for (var a of actions) {
        current += a.weight;
        if (random < current) {
            return enemy[a.action](player);
        }
    }
    
    // 默认攻击
    return enemy.attack(player);
}
```

**为什么用权重？**
- 可以精确控制行为概率
- 容易调节难度
- 支持动态调整

## AI 设计模式对比

| 模式 | 实现难度 | 灵活性 | 适用场景 |
|------|---------|--------|---------|
| **随机** | ★ | ★ | 低级怪物 |
| **条件** | ★★ | ★★ | 中级怪物 |
| **权重** | ★★★ | ★★★ | 高级怪物、BOSS |
| **状态机** | ★★★ | ★★★★ | 复杂行为 |
| **行为树** | ★★★★ | ★★★★★ | BOSS、复杂 AI |

## 常见 AI 行为

### 1. 巡逻

```
敌人沿着固定路线移动
     ┌───┐
     │ A │
     └─┬─┘
       │
     ┌─▼─┐
     │ B │
     └─┬─┘
       │
     ┌─▼─┐
     │ C │
     └─┬─┘
       │
     ┌─▼─┐
     │ A │ ← 回到起点
     └───┘
```

### 2. 追踪

```
发现玩家后，向玩家移动

       玩家
         ★
        ↗
      ↗
敌人 ●→→→
```

### 3. 攻击

```
在攻击范围内时，执行攻击

攻击范围:
  ┌─────────┐
  │    ★    │ 玩家
  │  ┌───┐  │
  │  │敌 │  │
  │  └───┘  │
  └─────────┘
```

### 4. 逃跑

```
HP 低时，远离玩家

敌人 ●←←←←←←
         ★ 玩家
```

## AI 性能优化

```javascript
// ============================================
// AI 性能优化：不要每帧都计算
// ============================================

class EnemyAI {
    constructor() {
        this.decisionInterval = 500;  // 500ms 做一次决策
        this.decisionTimer = 0;
        this.currentAction = null;
    }
    
    update(deltaTime) {
        this.decisionTimer += deltaTime;
        
        // 只在需要时做决策
        if (this.decisionTimer >= this.decisionInterval) {
            this.decisionTimer = 0;
            this.currentAction = this.makeDecision();
        }
        
        // 执行当前决策
        this.executeAction(this.currentAction);
    }
    
    makeDecision() {
        // 复杂的计算放在这里
        // 因为只定期执行，不会太卡
    }
    
    executeAction(action) {
        // 简单的执行逻辑
        // 每帧都执行
    }
}
```

## 本章小结

1. **随机 AI**：最简单，适合低级怪物
2. **条件 AI**：根据情况选择，适合中级怪物
3. **权重 AI**：可调节概率，适合高级怪物
4. **性能优化**：不要每帧都计算

## 下一章

[状态机 AI](02-state-machine.md) - 学习使用有限状态机实现复杂行为。
