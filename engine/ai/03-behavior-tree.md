# 行为树 AI

## 本章目标

理解行为树的原理，学会实现灵活的 AI 决策系统。

## 什么是行为树？

```
行为树是一种树状结构，从根节点开始执行：
┌────────────────────────────────────────┐
│ 每个节点返回：成功 / 失败 / 运行中     │
│ 根据节点类型决定如何处理结果           │
└────────────────────────────────────────┘

结构示例：
              [选择器]  ← 根节点
              /    \
        [序列]      [序列]
        /   \        /   \
    [条件] [动作] [条件] [动作]
```

## 节点类型

### 1. 动作节点（Action）

```javascript
// 执行具体行为，返回成功/失败
class ActionNode {
    constructor(action) {
        this.action = action;
    }
    
    tick() {
        return this.action() ? 'success' : 'failure';
    }
}
```

### 2. 条件节点（Condition）

```javascript
// 检查条件，返回成功/失败
class ConditionNode {
    constructor(condition) {
        this.condition = condition;
    }
    
    tick() {
        return this.condition() ? 'success' : 'failure';
    }
}
```

### 3. 序列节点（Sequence）

```javascript
// 按顺序执行子节点，全部成功才成功
// 任何一个失败就立即返回失败
class SequenceNode {
    constructor(children) {
        this.children = children;
        this.currentIndex = 0;
    }
    
    tick() {
        while (this.currentIndex < this.children.length) {
            var result = this.children[this.currentIndex].tick();
            
            if (result === 'success') {
                this.currentIndex++;
            } else {
                // 失败或运行中，重置并返回
                this.currentIndex = 0;
                return result;
            }
        }
        
        this.currentIndex = 0;
        return 'success';
    }
}
```

### 4. 选择器节点（Selector）

```javascript
// 按顺序执行子节点，找到成功的就返回
// 全部失败才返回失败
class SelectorNode {
    constructor(children) {
        this.children = children;
        this.currentIndex = 0;
    }
    
    tick() {
        while (this.currentIndex < this.children.length) {
            var result = this.children[this.currentIndex].tick();
            
            if (result === 'success') {
                this.currentIndex = 0;
                return 'success';
            } else if (result === 'failure') {
                this.currentIndex++;
            } else {
                // 运行中
                return 'running';
            }
        }
        
        this.currentIndex = 0;
        return 'failure';
    }
}
```

## 完整行为树实现

```javascript
// ============================================
// 行为树实现
// ============================================

class BehaviorTree {
    constructor(root, context) {
        this.root = root;      // 根节点
        this.context = context; // 上下文（敌人、玩家等）
    }
    
    update() {
        return this.root.tick(this.context);
    }
}

// 工厂函数
function action(fn) {
    return new ActionNode(fn);
}

function condition(fn) {
    return new ConditionNode(fn);
}

function sequence(...children) {
    return new SequenceNode(children);
}

function selector(...children) {
    return new SelectorNode(children);
}
```

## 敌人 AI 示例

```javascript
// ============================================
// 用行为树实现敌人 AI
// ============================================

function createEnemyBehaviorTree(enemy, player) {
    // 根节点：选择器
    return selector(
        // 优先级1：逃跑（HP < 20%）
        sequence(
            condition(() => enemy.hp / enemy.maxHp < 0.2),
            action(() => {
                // 向远离玩家的方向移动
                var dx = enemy.x - player.x;
                var dy = enemy.y - player.y;
                var dist = Math.sqrt(dx * dx + dy * dy);
                enemy.x += (dx / dist) * enemy.speed * 0.5;
                enemy.y += (dy / dist) * enemy.speed * 0.5;
                return true;
            })
        ),
        
        // 优先级2：攻击（玩家在攻击范围内）
        sequence(
            condition(() => getDistance(enemy, player) < 50),
            action(() => {
                enemy.attack(player);
                return true;
            })
        ),
        
        // 优先级3：追踪（发现玩家）
        sequence(
            condition(() => canSee(enemy, player)),
            action(() => {
                var dx = player.x - enemy.x;
                var dy = player.y - enemy.y;
                var dist = Math.sqrt(dx * dx + dy * dy);
                enemy.x += (dx / dist) * enemy.speed * 0.3;
                enemy.y += (dy / dist) * enemy.speed * 0.3;
                return true;
            })
        ),
        
        // 优先级4：巡逻
        action(() => {
            enemy.patrol();
            return true;
        })
    );
}

// 辅助函数
function getDistance(a, b) {
    var dx = a.x - b.x;
    var dy = a.y - b.y;
    return Math.sqrt(dx * dx + dy * dy);
}

function canSee(enemy, player) {
    return getDistance(enemy, player) < 200;
}
```

## 行为树 vs 状态机

| 维度 | 状态机 | 行为树 |
|------|--------|--------|
| **结构** | 扁平，状态之间有转换 | 树状，层次分明 |
| **复杂度** | 状态多时难以管理 | 节点可复用，容易管理 |
| **灵活性** | 较低，需要预设转换 | 较高，动态组合节点 |
| **调试** | 容易看到当前状态 | 需要遍历树 |
| **适用** | 简单 AI | 复杂 AI、BOSS |

## 本章小结

1. **行为树**：树状结构的 AI 决策系统
2. **节点类型**：动作、条件、序列、选择器
3. **优点**：灵活、可复用、适合复杂 AI

## 下一章

[寻路算法](04-pathfinding.md) - 学习 A* 算法让角色找到路径。
