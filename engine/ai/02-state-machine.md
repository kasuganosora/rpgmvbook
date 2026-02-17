# 状态机 AI

## 本章目标

学会使用有限状态机（FSM）实现复杂的 AI 行为。

## 什么是状态机？

```
状态机的核心思想：
┌─────────────────────────────────────┐
│ 角色在任何时刻只处于一种"状态"     │
│ 满足条件时，切换到另一个状态        │
│ 每个状态有特定的行为                │
└─────────────────────────────────────┘

例子：敌人的状态
┌─────────┐  发现玩家   ┌─────────┐
│  巡逻   │ ──────────► │  追踪   │
└─────────┘             └─────────┘
     ▲                       │
     │                       │ 进入攻击范围
     │                       ▼
     │                  ┌─────────┐
     └──────────────────│  攻击   │
       丢失目标          └─────────┘
```

## 为什么用状态机？

| 优点 | 说明 |
|------|------|
| **结构清晰** | 每个状态独立，易于理解 |
| **容易扩展** | 添加新状态很简单 |
| **便于调试** | 可以清楚看到当前状态 |
| **性能好** | 只执行当前状态的逻辑 |

## 状态机实现

### 基础版本

```javascript
// ============================================
// 简单状态机实现
// ============================================

class StateMachine {
    constructor() {
        this.currentState = null;    // 当前状态
        this.states = {};            // 所有状态
        this.transitions = {};       // 状态转换规则
    }
    
    // 添加状态
    addState(name, state) {
        this.states[name] = state;
    }
    
    // 添加转换规则
    addTransition(from, to, condition) {
        if (!this.transitions[from]) {
            this.transitions[from] = [];
        }
        this.transitions[from].push({
            to: to,
            condition: condition
        });
    }
    
    // 设置初始状态
    setInitialState(name) {
        this.currentState = this.states[name];
        this.currentStateName = name;
        if (this.currentState.enter) {
            this.currentState.enter();
        }
    }
    
    // 更新
    update(deltaTime) {
        // 1. 检查是否需要切换状态
        this.checkTransitions();
        
        // 2. 执行当前状态的更新
        if (this.currentState && this.currentState.update) {
            this.currentState.update(deltaTime);
        }
    }
    
    // 检查状态转换
    checkTransitions() {
        var transitions = this.transitions[this.currentStateName];
        if (!transitions) return;
        
        for (var transition of transitions) {
            if (transition.condition()) {
                this.changeState(transition.to);
                return;
            }
        }
    }
    
    // 切换状态
    changeState(newStateName) {
        // 退出当前状态
        if (this.currentState && this.currentState.exit) {
            this.currentState.exit();
        }
        
        // 进入新状态
        this.currentStateName = newStateName;
        this.currentState = this.states[newStateName];
        if (this.currentState.enter) {
            this.currentState.enter();
        }
    }
}
```

### 敌人 AI 示例

```javascript
// ============================================
// 敌人 AI 状态机
// ============================================

class EnemyAI {
    constructor(enemy, player) {
        this.enemy = enemy;
        this.player = player;
        this.fsm = new StateMachine();
        
        // 配置状态机
        this.setupStates();
        this.setupTransitions();
        
        // 初始状态为巡逻
        this.fsm.setInitialState('patrol');
    }
    
    // 设置状态
    setupStates() {
        var self = this;
        
        // ========== 巡逻状态 ==========
        this.fsm.addState('patrol', {
            enter: function() {
                console.log('进入巡逻状态');
                // 选择巡逻路径
                self.patrolPoints = self.getPatrolPoints();
                self.currentPatrolIndex = 0;
            },
            
            update: function(deltaTime) {
                // 沿着巡逻路径移动
                var target = self.patrolPoints[self.currentPatrolIndex];
                self.moveToward(target.x, target.y, deltaTime);
                
                // 检查是否到达巡逻点
                if (self.distanceTo(target) < 10) {
                    self.currentPatrolIndex++;
                    if (self.currentPatrolIndex >= self.patrolPoints.length) {
                        self.currentPatrolIndex = 0;
                    }
                }
            },
            
            exit: function() {
                console.log('离开巡逻状态');
            }
        });
        
        // ========== 追踪状态 ==========
        this.fsm.addState('chase', {
            enter: function() {
                console.log('进入追踪状态');
                // 播放发现玩家的动画/音效
            },
            
            update: function(deltaTime) {
                // 向玩家移动
                self.moveToward(self.player.x, self.player.y, deltaTime);
            },
            
            exit: function() {
                console.log('离开追踪状态');
            }
        });
        
        // ========== 攻击状态 ==========
        this.fsm.addState('attack', {
            enter: function() {
                console.log('进入攻击状态');
                self.attackCooldown = 0;
            },
            
            update: function(deltaTime) {
                self.attackCooldown -= deltaTime;
                
                // 面向玩家
                self.faceToward(self.player.x, self.player.y);
                
                // 攻击冷却好了就攻击
                if (self.attackCooldown <= 0) {
                    self.performAttack();
                    self.attackCooldown = 1000;  // 1秒冷却
                }
            },
            
            exit: function() {
                console.log('离开攻击状态');
            }
        });
        
        // ========== 逃跑状态 ==========
        this.fsm.addState('flee', {
            enter: function() {
                console.log('进入逃跑状态');
            },
            
            update: function(deltaTime) {
                // 远离玩家
                var dx = self.enemy.x - self.player.x;
                var dy = self.enemy.y - self.player.y;
                var dist = Math.sqrt(dx * dx + dy * dy);
                
                // 向反方向移动
                self.enemy.x += (dx / dist) * self.enemy.speed * deltaTime / 100;
                self.enemy.y += (dy / dist) * self.enemy.speed * deltaTime / 100;
            },
            
            exit: function() {
                console.log('离开逃跑状态');
            }
        });
    }
    
    // 设置状态转换
    setupTransitions() {
        var self = this;
        
        // 巡逻 → 追踪：发现玩家
        this.fsm.addTransition('patrol', 'chase', function() {
            return self.canSeePlayer() && self.getDistanceToPlayer() < 200;
        });
        
        // 追踪 → 攻击：进入攻击范围
        this.fsm.addTransition('chase', 'attack', function() {
            return self.getDistanceToPlayer() < 50;
        });
        
        // 追踪 → 巡逻：丢失玩家
        this.fsm.addTransition('chase', 'patrol', function() {
            return !self.canSeePlayer() || self.getDistanceToPlayer() > 300;
        });
        
        // 攻击 → 追踪：玩家离开攻击范围
        this.fsm.addTransition('attack', 'chase', function() {
            return self.getDistanceToPlayer() > 60;
        });
        
        // 攻击 → 逃跑：HP 低于 20%
        this.fsm.addTransition('attack', 'flee', function() {
            return self.enemy.hp / self.enemy.maxHp < 0.2;
        });
        
        // 逃跑 → 巡逻：成功逃离
        this.fsm.addTransition('flee', 'patrol', function() {
            return self.getDistanceToPlayer() > 400;
        });
    }
    
    // 更新
    update(deltaTime) {
        this.fsm.update(deltaTime);
    }
    
    // ========== 辅助方法 ==========
    
    canSeePlayer() {
        // 简化：视线检测
        return true;
    }
    
    getDistanceToPlayer() {
        var dx = this.enemy.x - this.player.x;
        var dy = this.enemy.y - this.player.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    distanceTo(point) {
        var dx = this.enemy.x - point.x;
        var dy = this.enemy.y - point.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    moveToward(x, y, deltaTime) {
        var dx = x - this.enemy.x;
        var dy = y - this.enemy.y;
        var dist = Math.sqrt(dx * dx + dy * dy);
        
        if (dist > 5) {
            this.enemy.x += (dx / dist) * this.enemy.speed * deltaTime / 100;
            this.enemy.y += (dy / dist) * this.enemy.speed * deltaTime / 100;
        }
    }
    
    faceToward(x, y) {
        var dx = x - this.enemy.x;
        this.enemy.direction = dx >= 0 ? 'right' : 'left';
    }
    
    getPatrolPoints() {
        return [
            { x: this.enemy.x - 100, y: this.enemy.y },
            { x: this.enemy.x + 100, y: this.enemy.y },
            { x: this.enemy.x + 100, y: this.enemy.y + 50 },
            { x: this.enemy.x - 100, y: this.enemy.y + 50 }
        ];
    }
    
    performAttack() {
        console.log(this.enemy.name, '攻击玩家！');
        // 实际攻击逻辑
    }
}
```

## 状态图

```
                    发现玩家
    ┌─────────┐ ──────────────► ┌─────────┐
    │  巡逻   │                 │  追踪   │
    └─────────┘ ◄────────────── └─────────┘
         ▲        丢失目标           │
         │                           │
         │                      进入攻击范围
         │                           │
         │                           ▼
         │                      ┌─────────┐
         │                      │  攻击   │
         │                      └─────────┘
         │                           │
         │      成功逃离             │ HP < 20%
         │                           ▼
         └───────────── ◄────── ┌─────────┐
                              │  逃跑   │
                              └─────────┘
```

## 本章小结

1. **状态机**：角色在任何时刻只处于一种状态
2. **状态转换**：满足条件时切换状态
3. **优点**：结构清晰、容易扩展、便于调试

## 下一章

[行为树 AI](03-behavior-tree.md) - 学习更灵活的 AI 结构。
