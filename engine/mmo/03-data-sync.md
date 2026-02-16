# 数据同步：让所有玩家看到一致的世界

## 本章目标

理解网络游戏的核心难题——如何让不同客户端看到一致的游戏状态。

## 同步问题是什么？

```
时间点 T：
玩家A 在位置 (100, 100)，玩家B 在看A

理想情况：
T+0ms   A 移动，B 立即看到
        A: (105, 100)  B: 看到A在 (105, 100)

现实情况（网络延迟 50ms）：
T+0ms   A 移动
T+0ms   A 发送位置给服务器
T+50ms  服务器收到，转发给 B
T+100ms B 收到，更新 A 的位置

结果：A 已经走到 (110, 100)，B 才看到 A 在 (105, 100)
```

## 三种同步策略

### 策略 1：状态同步
服务器权威，所有状态由服务器计算，适合大多数 RPG。

### 策略 2：帧同步
所有客户端执行相同逻辑，适合 RTS 游戏（星际争霸）。

### 策略 3：预测+校正
客户端预测结果，服务器确认后校正，适合 FPS/动作游戏。

## 位置同步实现

```javascript
// 客户端：发送移动请求
function onPlayerMove(dx, dy) {
    // 发送给服务器
    socket.send(JSON.stringify({
        type: 'move',
        dx: dx,
        dy: dy,
        timestamp: Date.now()
    }));
}

// 服务器：验证并广播
function handleMove(playerId, data) {
    var player = players.get(playerId);
    
    // 验证移动速度
    var distance = Math.sqrt(data.dx * data.dx + data.dy * data.dy);
    if (distance > player.speed) {
        return;  // 作弊，忽略
    }
    
    // 更新位置
    player.x += data.dx;
    player.y += data.dy;
    
    // 广播给其他玩家
    broadcast({
        type: 'playerMove',
        playerId: playerId,
        x: player.x,
        y: player.y
    }, playerId);
}
```

## 插值平滑

其他玩家的移动需要插值，否则会"瞬移"：

```javascript
class RemotePlayer {
    constructor() {
        this.x = 0;
        this.y = 0;
        this.targetX = 0;
        this.targetY = 0;
    }
    
    // 收到新位置
    updatePosition(x, y) {
        this.targetX = x;
        this.targetY = y;
    }
    
    // 每帧插值
    interpolate() {
        // 线性插值，100ms 内平滑过渡
        this.x += (this.targetX - this.x) * 0.1;
        this.y += (this.targetY - this.y) * 0.1;
    }
}
```

## 本章小结

| 策略 | 适用场景 | 复杂度 |
|------|---------|--------|
| 状态同步 | RPG、回合制 | ★★ |
| 帧同步 | RTS、MOBA | ★★★ |
| 预测校正 | FPS、动作 | ★★★★ |
