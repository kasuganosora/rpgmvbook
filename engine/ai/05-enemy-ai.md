# 敌人 AI

## 本章目标

综合前面学到的知识，实现完整的敌人 AI 系统。

## 敌人 AI 组成

```
完整的敌人 AI 需要：
┌─────────────────────────────────────┐
│ 1. 感知系统：发现玩家               │
│ 2. 决策系统：选择行为               │
│ 3. 移动系统：走路、追踪             │
│ 4. 战斗系统：攻击、技能             │
└─────────────────────────────────────┘
```

## 完整实现

```javascript
// ============================================
// EnemyAI - 完整的敌人 AI
// ============================================

class EnemyAI {
    constructor(enemy) {
        this.enemy = enemy;
        
        // 感知参数
        this.sightRange = 200;      // 视野范围
        this.sightAngle = 90;       // 视野角度
        this.hearingRange = 100;    // 听觉范围
        
        // 战斗参数
        this.attackRange = 50;      // 攻击范围
        this.attackCooldown = 0;    // 攻击冷却
        this.attackCooldownTime = 1000;
        
        // AI 状态
        this.state = 'idle';        // idle, patrol, chase, attack, flee
        this.target = null;         // 当前目标
        
        // 巡逻
        this.patrolPoints = [];
        this.currentPatrolIndex = 0;
        
        // 寻路
        this.path = [];
        this.pathUpdateTimer = 0;
        this.pathUpdateInterval = 500;
    }
    
    // 更新
    update(deltaTime, player) {
        // 更新冷却
        this.attackCooldown -= deltaTime;
        
        // 更新寻路
        this.pathUpdateTimer -= deltaTime;
        
        // 感知
        if (this.canSense(player)) {
            this.target = player;
        }
        
        // 根据状态执行行为
        switch(this.state) {
            case 'idle':
                this.updateIdle(deltaTime);
                break;
            case 'patrol':
                this.updatePatrol(deltaTime);
                break;
            case 'chase':
                this.updateChase(deltaTime, player);
                break;
            case 'attack':
                this.updateAttack(deltaTime, player);
                break;
            case 'flee':
                this.updateFlee(deltaTime, player);
                break;
        }
    }
    
    // ========== 感知系统 ==========
    
    canSense(player) {
        // 视觉检测
        if (this.canSee(player)) return true;
        
        // 听觉检测（玩家在跑步时）
        if (player.isRunning && this.canHear(player)) return true;
        
        return false;
    }
    
    canSee(player) {
        var dist = this.getDistanceTo(player);
        if (dist > this.sightRange) return false;
        
        // 检查角度
        var angle = this.getAngleTo(player);
        var angleDiff = Math.abs(angle - this.enemy.direction);
        if (angleDiff > this.sightAngle / 2) return false;
        
        // 检查视线遮挡（简化）
        return !this.hasLineOfSightBlocked(player);
    }
    
    canHear(player) {
        var dist = this.getDistanceTo(player);
        return dist < this.hearingRange;
    }
    
    // ========== 状态更新 ==========
    
    updateIdle(deltaTime) {
        // 空闲时开始巡逻
        this.state = 'patrol';
        this.patrolPoints = this.generatePatrolPoints();
    }
    
    updatePatrol(deltaTime) {
        if (this.target) {
            this.state = 'chase';
            return;
        }
        
        // 沿巡逻点移动
        var target = this.patrolPoints[this.currentPatrolIndex];
        this.moveToward(target.x, target.y, deltaTime);
        
        if (this.getDistanceToPoint(target) < 5) {
            this.currentPatrolIndex++;
            if (this.currentPatrolIndex >= this.patrolPoints.length) {
                this.currentPatrolIndex = 0;
            }
        }
    }
    
    updateChase(deltaTime, player) {
        if (!this.target) {
            this.state = 'patrol';
            return;
        }
        
        var dist = this.getDistanceTo(player);
        
        // 进入攻击范围
        if (dist < this.attackRange) {
            this.state = 'attack';
            return;
        }
        
        // 丢失目标
        if (dist > this.sightRange * 2) {
            this.target = null;
            this.state = 'patrol';
            return;
        }
        
        // 使用寻路追踪
        this.updatePath(player);
        this.followPath(deltaTime);
    }
    
    updateAttack(deltaTime, player) {
        var dist = this.getDistanceTo(player);
        
        // 目标离开攻击范围
        if (dist > this.attackRange * 1.5) {
            this.state = 'chase';
            return;
        }
        
        // 面向目标
        this.faceToward(player);
        
        // 攻击
        if (this.attackCooldown <= 0) {
            this.performAttack(player);
            this.attackCooldown = this.attackCooldownTime;
        }
    }
    
    updateFlee(deltaTime, player) {
        // 向远离玩家的方向移动
        var dx = this.enemy.x - player.x;
        var dy = this.enemy.y - player.y;
        var dist = Math.sqrt(dx * dx + dy * dy);
        
        this.enemy.x += (dx / dist) * this.enemy.speed * deltaTime / 100;
        this.enemy.y += (dy / dist) * this.enemy.speed * deltaTime / 100;
        
        // 逃远后恢复巡逻
        if (dist > this.sightRange * 2) {
            this.state = 'patrol';
            this.target = null;
        }
    }
    
    // ========== 辅助方法 ==========
    
    getDistanceTo(target) {
        var dx = this.enemy.x - target.x;
        var dy = this.enemy.y - target.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    moveToward(x, y, deltaTime) {
        var dx = x - this.enemy.x;
        var dy = y - this.enemy.y;
        var dist = Math.sqrt(dx * dx + dy * dy);
        
        if (dist > 1) {
            this.enemy.x += (dx / dist) * this.enemy.speed * deltaTime / 100;
            this.enemy.y += (dy / dist) * this.enemy.speed * deltaTime / 100;
            this.enemy.direction = dx >= 0 ? 'right' : 'left';
        }
    }
    
    updatePath(target) {
        if (this.pathUpdateTimer <= 0) {
            this.path = pathFinder.findPath(
                this.enemy.x, this.enemy.y,
                target.x, target.y
            );
            this.pathUpdateTimer = this.pathUpdateInterval;
        }
    }
    
    followPath(deltaTime) {
        if (!this.path || this.path.length === 0) return;
        
        var next = this.path[0];
        this.moveToward(next.x, next.y, deltaTime);
        
        if (this.getDistanceToPoint(next) < 5) {
            this.path.shift();
        }
    }
    
    generatePatrolPoints() {
        return [
            { x: this.enemy.x - 50, y: this.enemy.y },
            { x: this.enemy.x + 50, y: this.enemy.y }
        ];
    }
}
```

## 本章小结

1. **感知**：视野、听觉检测
2. **状态**：空闲、巡逻、追踪、攻击、逃跑
3. **寻路**：使用 A* 算法追踪玩家

## 下一章

[NPC AI](06-npc-ai.md) - 实现友方 NPC 的行为。
