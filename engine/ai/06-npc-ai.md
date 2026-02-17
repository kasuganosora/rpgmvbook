# NPC AI

## 本章目标

实现友方 NPC 的行为，包括对话、日程系统等。

## NPC vs 敌人 AI

| 维度 | 敌人 AI | NPC AI |
|------|---------|--------|
| **目标** | 攻击玩家 | 与玩家互动 |
| **行为** | 追踪、攻击 | 走路、对话、工作 |
| **状态** | 敌对 | 友好 |

## NPC 行为类型

### 1. 静止 NPC

```javascript
// 站在固定位置的 NPC
class StaticNPC {
    constructor(x, y, dialog) {
        this.x = x;
        this.y = y;
        this.dialog = dialog;
        this.direction = 'down';
    }
    
    // 玩家接近时转向
    onPlayerApproach(player) {
        var dx = player.x - this.x;
        var dy = player.y - this.y;
        
        if (Math.abs(dx) > Math.abs(dy)) {
            this.direction = dx > 0 ? 'right' : 'left';
        } else {
            this.direction = dy > 0 ? 'down' : 'up';
        }
    }
}
```

### 2. 移动 NPC

```javascript
// 在固定路线移动的 NPC
class PatrolNPC {
    constructor(waypoints, dialog) {
        this.waypoints = waypoints;
        this.currentWaypoint = 0;
        this.dialog = dialog;
        this.speed = 1;
    }
    
    update(deltaTime) {
        var target = this.waypoints[this.currentWaypoint];
        
        // 移动到下一个点
        var dx = target.x - this.x;
        var dy = target.y - this.y;
        var dist = Math.sqrt(dx * dx + dy * dy);
        
        if (dist < 2) {
            // 到达，切换到下一个点
            this.currentWaypoint++;
            if (this.currentWaypoint >= this.waypoints.length) {
                this.currentWaypoint = 0;
            }
        } else {
            this.x += (dx / dist) * this.speed;
            this.y += (dy / dist) * this.speed;
        }
    }
}
```

### 3. 日程 NPC

```javascript
// 按时间表行动的 NPC
class ScheduleNPC {
    constructor(schedule) {
        this.schedule = schedule;  // 日程安排
        this.currentAction = null;
    }
    
    update(gameTime) {
        // 查找当前时间应该做什么
        var action = this.findActionForTime(gameTime);
        
        if (action !== this.currentAction) {
            this.currentAction = action;
            this.executeAction(action);
        }
    }
    
    findActionForTime(gameTime) {
        var hour = gameTime.hour;
        
        for (var item of this.schedule) {
            if (hour >= item.startHour && hour < item.endHour) {
                return item;
            }
        }
        
        return null;
    }
    
    executeAction(action) {
        switch(action.type) {
            case 'move':
                this.moveTo(action.location);
                break;
            case 'work':
                this.work(action.task);
                break;
            case 'sleep':
                this.sleep();
                break;
        }
    }
}

// 日程示例
var npcSchedule = [
    { startHour: 6, endHour: 8, type: 'move', location: { x: 100, y: 100 } },
    { startHour: 8, endHour: 12, type: 'work', task: 'shop' },
    { startHour: 12, endHour: 13, type: 'move', location: { x: 200, y: 150 } },
    { startHour: 13, endHour: 18, type: 'work', task: 'shop' },
    { startHour: 18, endHour: 22, type: 'move', location: { x: 300, y: 200 } },
    { startHour: 22, endHour: 6, type: 'sleep' }
];
```

## 对话系统

```javascript
// ============================================
// 对话系统
// ============================================

class DialogSystem {
    constructor() {
        this.currentDialog = null;
        this.currentIndex = 0;
    }
    
    // 开始对话
    startDialog(npc) {
        this.currentDialog = npc.dialog;
        this.currentIndex = 0;
        this.showCurrentLine();
    }
    
    // 下一句
    next() {
        this.currentIndex++;
        
        if (this.currentIndex >= this.currentDialog.length) {
            this.endDialog();
        } else {
            this.showCurrentLine();
        }
    }
    
    // 显示当前句
    showCurrentLine() {
        var line = this.currentDialog[this.currentIndex];
        
        // 检查是否有选项
        if (line.options) {
            this.showOptions(line.options);
        } else {
            this.showText(line.speaker, line.text);
        }
    }
    
    // 结束对话
    endDialog() {
        this.currentDialog = null;
        this.currentIndex = 0;
        this.hideDialog();
    }
}

// 对话数据示例
var sampleDialog = [
    { speaker: '村长', text: '你好，年轻人！' },
    { speaker: '村长', text: '有什么可以帮你的吗？' },
    { 
        speaker: '村长', 
        options: [
            { text: '请问商店在哪里？', next: 'shop' },
            { text: '最近有什么任务吗？', next: 'quest' },
            { text: '没有，谢谢。', next: 'end' }
        ]
    }
];
```

## 本章小结

1. **静止 NPC**：站在固定位置
2. **移动 NPC**：沿固定路线巡逻
3. **日程 NPC**：按时间表行动
4. **对话系统**：与玩家互动

## 系列总结

通过这 6 章，你学会了：
- AI 基础概念
- 状态机实现
- 行为树结构
- A* 寻路算法
- 敌人 AI 系统
- NPC AI 系统

现在你可以为自己的游戏设计各种 AI 行为了！
