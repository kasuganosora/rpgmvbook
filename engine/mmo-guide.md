# 从单机 RPG 到 MMO：网络多人游戏开发指南

## 前言

MMO（Massively Multiplayer Online）大型多人在线游戏，是单机游戏的进阶形态。

**单机 RPG vs MMO**：

```
单机 RPG：
┌─────────────┐
│   游戏客户端  │
│   ┌───────┐  │
│   │游戏逻辑│  │
│   │游戏数据│  │
│   │存档系统│  │
│   └───────┘  │
└─────────────┘

MMO：
┌─────────────┐          ┌─────────────┐
│  客户端 A   │◄────────►│             │
├─────────────┤          │   游戏服务器 │
│  客户端 B   │◄────────►│   ┌───────┐ │
├─────────────┤          │   │游戏逻辑│ │
│  客户端 C   │◄────────►│   │游戏数据│ │
├─────────────┤          │   │数据库  │ │
│    ...     │◄────────►│   └───────┘ │
└─────────────┘          └─────────────┘
```

**核心区别**：
1. **逻辑在哪**：单机在客户端，MMO 在服务器
2. **数据存储**：单机在本地，MMO 在数据库
3. **信任谁**：单机信任自己，MMO 不信任客户端

---

## 第一部分：网络基础

### 1.1 网络协议选择

| 协议 | 特点 | 适用场景 |
|------|------|---------|
| **TCP** | 可靠、有序、有连接 | 聊天、交易、登录 |
| **UDP** | 不可靠、快速、无连接 | 实时移动、战斗 |
| **WebSocket** | TCP、双向、Web支持 | 浏览器游戏 |
| **HTTP/REST** | 请求-响应 | 排行榜、商城 |

### 1.2 为什么游戏常用 UDP？

**TCP 的问题**：
```
发送数据：A B C D E
网络丢包：A B _ D E
TCP 重传：A B [等待C] C D E  ← 延迟！

游戏场景：
玩家位置：(100, 100) → (105, 100) → (110, 100)
如果中间丢包，玩家会"卡住"
```

**UDP 的优势**：
```
发送数据：位置1 位置2 位置3 位置4
网络丢包：位置1 位置2 ___ 位置4
UDP 处理：继续用位置2，直接跳到位置4  ← 流畅！

不需要重传，因为旧位置已经没用了
```

### 1.3 WebSocket 基础

对于 Web 游戏，WebSocket 是首选：

```javascript
// ============================================
// 客户端 WebSocket 连接
// ============================================

var socket = new WebSocket('ws://game-server.com:8080');

// 连接成功
socket.onopen = function() {
    console.log('连接服务器成功');
    socket.send(JSON.stringify({
        type: 'login',
        username: 'player1',
        password: 'xxx'
    }));
};

// 收到消息
socket.onmessage = function(event) {
    var data = JSON.parse(event.data);
    handleMessage(data);
};

// 连接关闭
socket.onclose = function() {
    console.log('连接断开');
    // 尝试重连
    setTimeout(reconnect, 3000);
};

// 发送消息
function sendMessage(type, payload) {
    socket.send(JSON.stringify({
        type: type,
        payload: payload,
        timestamp: Date.now()
    }));
}

// 处理消息
function handleMessage(data) {
    switch(data.type) {
        case 'playerMove':
            updatePlayerPosition(data.payload);
            break;
        case 'chat':
            displayChatMessage(data.payload);
            break;
        case 'playerList':
            updatePlayerList(data.payload);
            break;
    }
}
```

```javascript
// ============================================
// 服务器端 WebSocket (Node.js + ws 库)
// ============================================

const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// 存储所有连接的玩家
var players = new Map();

wss.on('connection', function(ws) {
    var playerId = generateId();
    players.set(playerId, {
        socket: ws,
        x: 0,
        y: 0,
        name: ''
    });

    console.log('玩家连接:', playerId);

    // 发送欢迎消息
    ws.send(JSON.stringify({
        type: 'welcome',
        playerId: playerId
    }));

    // 收到消息
    ws.on('message', function(message) {
        var data = JSON.parse(message);
        handlePlayerMessage(playerId, data);
    });

    // 断开连接
    ws.on('close', function() {
        players.delete(playerId);
        broadcast({
            type: 'playerLeave',
            playerId: playerId
        });
        console.log('玩家断开:', playerId);
    });
});

// 处理玩家消息
function handlePlayerMessage(playerId, data) {
    var player = players.get(playerId);
    if (!player) return;

    switch(data.type) {
        case 'login':
            player.name = data.username;
            broadcast({
                type: 'playerJoin',
                playerId: playerId,
                name: player.name
            });
            break;

        case 'move':
            // 更新位置
            player.x = data.x;
            player.y = data.y;
            // 广播给其他玩家
            broadcast({
                type: 'playerMove',
                playerId: playerId,
                x: player.x,
                y: player.y
            }, playerId);  // 排除自己
            break;

        case 'chat':
            broadcast({
                type: 'chat',
                playerId: playerId,
                name: player.name,
                message: data.message
            });
            break;
    }
}

// 广播消息
function broadcast(message, excludeId) {
    var json = JSON.stringify(message);
    players.forEach(function(player, id) {
        if (id !== excludeId && player.socket.readyState === WebSocket.OPEN) {
            player.socket.send(json);
        }
    });
}

function generateId() {
    return Math.random().toString(36).substr(2, 9);
}
```

---

## 第二部分：服务器架构

### 2.1 架构演进

```
阶段 1：单服务器（适合小规模）
┌────────────────┐
│   游戏服务器    │
│  ┌──────────┐  │
│  │ 逻辑+数据 │  │
│  └──────────┘  │
└────────────────┘
支持：100-1000 人

阶段 2：分离数据库（适合中型）
┌────────────────┐     ┌────────────┐
│   游戏服务器    │◄───►│   数据库   │
└────────────────┘     └────────────┘
支持：1000-10000 人

阶段 3：多服务器 + 网关（适合大型）
┌────────┐  ┌────────┐  ┌────────┐
│ 游戏服1 │  │ 游戏服2 │  │ 游戏服3 │
└───┬────┘  └───┬────┘  └───┬────┘
    │           │           │
    └───────────┼───────────┘
                │
          ┌─────▼─────┐
          │   网关    │
          └─────┬─────┘
                │
          ┌─────▼─────┐
          │   数据库   │
          └───────────┘
支持：10000+ 人

阶段 4：微服务（适合超大型）
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│登录服│ │地图服│ │战斗服│ │聊天服│
└──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
   │        │        │        │
   └────────┴────────┴────────┘
                    │
              ┌─────▼─────┐
              │ 消息队列  │
              └─────┬─────┘
                    │
              ┌─────▼─────┐
              │  数据库   │
              └───────────┘
支持：10万+ 人
```

### 2.2 简单服务器架构示例

```javascript
// ============================================
// 游戏服务器核心
// ============================================

class GameServer {
    constructor() {
        this.players = new Map();      // 在线玩家
        this.maps = new Map();         // 地图实例
        this.database = new Database(); // 数据库连接
    }

    // 玩家登录
    async login(ws, username, password) {
        // 1. 验证账号
        var account = await this.database.getAccount(username);
        if (!account || account.password !== hashPassword(password)) {
            return { success: false, error: '用户名或密码错误' };
        }

        // 2. 检查是否已在线
        if (this.players.has(account.id)) {
            return { success: false, error: '账号已在其他地方登录' };
        }

        // 3. 加载角色数据
        var character = await this.database.getCharacter(account.id);

        // 4. 创建玩家实例
        var player = new Player(ws, character);
        this.players.set(player.id, player);

        // 5. 加入地图
        var map = this.getOrCreateMap(character.mapId);
        map.addPlayer(player);

        return {
            success: true,
            player: player.toClientData(),
            mapPlayers: map.getOtherPlayers(player.id)
        };
    }

    // 获取或创建地图
    getOrCreateMap(mapId) {
        if (!this.maps.has(mapId)) {
            this.maps.set(mapId, new GameMap(mapId));
        }
        return this.maps.get(mapId);
    }

    // 玩家移动
    movePlayer(playerId, x, y, direction) {
        var player = this.players.get(playerId);
        if (!player) return;

        // 验证移动是否合法
        if (!this.validateMove(player, x, y)) {
            // 作弊检测
            return;
        }

        player.x = x;
        player.y = y;
        player.direction = direction;

        // 广播给同地图玩家
        var map = this.maps.get(player.mapId);
        map.broadcast({
            type: 'playerMove',
            playerId: playerId,
            x: x,
            y: y,
            direction: direction
        }, playerId);
    }

    // 验证移动
    validateMove(player, newX, newY) {
        var speed = player.speed;
        var dx = Math.abs(newX - player.x);
        var dy = Math.abs(newY - player.y);
        var distance = Math.sqrt(dx * dx + dy * dy);

        // 检查移动速度是否合理
        if (distance > speed * 2) {  // 允许一定误差
            console.log('可疑移动:', player.id, distance);
            return false;
        }

        return true;
    }
}
```

---

## 第三部分：数据同步

### 3.1 同步策略

**策略 1：状态同步（State Synchronization）**
```
客户端发送：我的位置是 (100, 100)
服务器验证：合理，广播给其他客户端
其他客户端：更新玩家位置到 (100, 100)

特点：
- 服务器权威
- 客户端只发送操作
- 所有逻辑在服务器
- 安全但延迟高
```

**策略 2：帧同步（Lockstep）**
```
所有客户端：同时执行第 N 帧
客户端 A：我按了右键
客户端 B：我按了攻击
服务器：收集所有操作，广播
所有客户端：用相同输入，执行相同逻辑

特点：
- 带宽低
- 需要确定性
- 一个卡住，全部等待
- 适合 RTS 游戏
```

**策略 3：预测+校正（Client-Side Prediction）**
```
客户端按右键：
1. 立即显示移动（预测）
2. 同时发送给服务器
3. 服务器确认后校正位置

如果预测正确：流畅
如果预测错误：校正（可能会"瞬移"）

特点：
- 低延迟体验
- 实现复杂
- 需要平滑校正
```

### 3.2 位置同步实现

```javascript
// ============================================
// 客户端：预测 + 发送
// ============================================

class PlayerMovement {
    constructor() {
        this.x = 100;
        this.y = 100;
        this.pendingMoves = [];  // 等待确认的移动
        this.sequenceNumber = 0;
    }

    // 玩家按下方向键
    move(dx, dy) {
        // 1. 立即预测移动
        this.x += dx;
        this.y += dy;

        // 2. 记录这次移动
        var move = {
            seq: this.sequenceNumber++,
            x: this.x,
            y: this.y,
            dx: dx,
            dy: dy,
            time: Date.now()
        };
        this.pendingMoves.push(move);

        // 3. 发送给服务器
        socket.send(JSON.stringify({
            type: 'move',
            seq: move.seq,
            dx: dx,
            dy: dy
        }));
    }

    // 收到服务器确认
    onServerAck(ackSeq, serverX, serverY) {
        // 1. 移除已确认的移动
        this.pendingMoves = this.pendingMoves.filter(m => m.seq > ackSeq);

        // 2. 检查位置是否一致
        var dx = Math.abs(this.x - serverX);
        var dy = Math.abs(this.y - serverY);

        if (dx > 1 || dy > 1) {
            // 位置不一致，需要校正
            this.x = serverX;
            this.y = serverY;

            // 重放未确认的移动
            for (var move of this.pendingMoves) {
                this.x += move.dx;
                this.y += move.dy;
            }
        }
    }
}
```

```javascript
// ============================================
// 服务器：验证 + 广播
// ============================================

class ServerMovement {
    constructor() {
        this.lastAck = new Map();  // 每个玩家的最后确认序列号
    }

    handleMove(playerId, seq, dx, dy) {
        var player = players.get(playerId);

        // 检查序列号是否正确（防重放）
        var lastSeq = this.lastAck.get(playerId) || 0;
        if (seq <= lastSeq) {
            console.log('重复或乱序的移动包');
            return;
        }

        // 检查移动速度
        var maxSpeed = player.speed;
        if (Math.abs(dx) > maxSpeed || Math.abs(dy) > maxSpeed) {
            console.log('移动速度异常');
            return;
        }

        // 更新位置
        player.x += dx;
        player.y += dy;

        // 边界检查
        player.x = Math.max(0, Math.min(player.x, mapWidth));
        player.y = Math.max(0, Math.min(player.y, mapHeight));

        // 记录确认
        this.lastAck.set(playerId, seq);

        // 发送确认
        player.socket.send(JSON.stringify({
            type: 'moveAck',
            seq: seq,
            x: player.x,
            y: player.y
        }));

        // 广播给其他玩家
        broadcast({
            type: 'playerMove',
            playerId: playerId,
            x: player.x,
            y: player.y
        }, playerId);
    }
}
```

### 3.3 插值与平滑

其他玩家的移动需要插值，否则会"瞬移"：

```javascript
// ============================================
// 其他玩家的位置插值
// ============================================

class RemotePlayer {
    constructor() {
        this.x = 0;
        this.y = 0;
        this.targetX = 0;  // 目标位置
        this.targetY = 0;
        this.lastUpdateTime = 0;
    }

    // 收到新位置
    updatePosition(x, y) {
        this.targetX = x;
        this.targetY = y;
        this.lastUpdateTime = Date.now();
    }

    // 每帧调用
    interpolate() {
        var elapsed = Date.now() - this.lastUpdateTime;
        var t = Math.min(1, elapsed / 100);  // 100ms 内平滑

        // 线性插值
        this.x += (this.targetX - this.x) * t;
        this.y += (this.targetY - this.y) * t;
    }
}
```

---

## 第四部分：数据库设计

### 4.1 数据库选型

| 数据库 | 特点 | 适用场景 |
|--------|------|---------|
| **MySQL** | 关系型、ACID | 账号、物品、交易 |
| **Redis** | 内存、快速 | 缓存、会话、排行榜 |
| **MongoDB** | 文档型、灵活 | 玩家数据、日志 |
| **PostgreSQL** | 关系型、强大 | 复杂查询、地理 |

### 4.2 表结构设计

```sql
-- ============================================
-- 账号表
-- ============================================
CREATE TABLE accounts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(32) UNIQUE NOT NULL,
    password_hash VARCHAR(64) NOT NULL,
    salt VARCHAR(32) NOT NULL,
    email VARCHAR(128),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    banned BOOLEAN DEFAULT FALSE
);

-- ============================================
-- 角色表
-- ============================================
CREATE TABLE characters (
    id INT PRIMARY KEY AUTO_INCREMENT,
    account_id INT NOT NULL,
    name VARCHAR(16) UNIQUE NOT NULL,
    level INT DEFAULT 1,
    exp INT DEFAULT 0,
    hp INT DEFAULT 100,
    mp INT DEFAULT 30,
    strength INT DEFAULT 10,
    agility INT DEFAULT 10,
    intelligence INT DEFAULT 10,
    map_id INT DEFAULT 1,
    x INT DEFAULT 100,
    y INT DEFAULT 100,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

-- ============================================
-- 物品表
-- ============================================
CREATE TABLE items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    character_id INT NOT NULL,
    item_id INT NOT NULL,  -- 物品类型ID
    count INT DEFAULT 1,
    slot INT,  -- 背包位置
    FOREIGN KEY (character_id) REFERENCES characters(id)
);

-- ============================================
-- 装备表
-- ============================================
CREATE TABLE equipment (
    character_id INT PRIMARY KEY,
    weapon_id INT,
    armor_id INT,
    helmet_id INT,
    accessory_id INT,
    FOREIGN KEY (character_id) REFERENCES characters(id)
);
```

### 4.3 使用 Redis 缓存

```javascript
// ============================================
// Redis 缓存层
// ============================================

const redis = require('redis');
const client = redis.createClient();

class CacheManager {
    // 获取玩家数据（先查缓存，再查数据库）
    async getPlayer(playerId) {
        // 1. 查缓存
        var cached = await client.get('player:' + playerId);
        if (cached) {
            return JSON.parse(cached);
        }

        // 2. 查数据库
        var player = await db.query(
            'SELECT * FROM characters WHERE id = ?',
            [playerId]
        );

        // 3. 写入缓存（10分钟过期）
        await client.setex(
            'player:' + playerId,
            600,
            JSON.stringify(player)
        );

        return player;
    }

    // 更新玩家数据（同时更新缓存和数据库）
    async updatePlayer(playerId, data) {
        // 1. 更新数据库
        await db.query(
            'UPDATE characters SET x = ?, y = ?, hp = ?, mp = ? WHERE id = ?',
            [data.x, data.y, data.hp, data.mp, playerId]
        );

        // 2. 更新缓存
        await client.setex(
            'player:' + playerId,
            600,
            JSON.stringify(data)
        );
    }

    // 玩家下线时保存
    async saveAndRemove(playerId) {
        var data = await client.get('player:' + playerId);
        if (data) {
            var player = JSON.parse(data);
            await this.updatePlayer(playerId, player);
            await client.del('player:' + playerId);
        }
    }
}
```

---

## 第五部分：安全性

### 5.1 常见作弊方式

| 作弊类型 | 原理 | 防御 |
|---------|------|------|
| **修改客户端** | 修改游戏代码/内存 | 服务器验证所有操作 |
| **加速器** | 加快游戏速度 | 服务器时间戳检查 |
| **封包修改** | 修改网络数据包 | 加密 + 签名 |
| **重放攻击** | 重复发送有效数据包 | 序列号 + 时间戳 |
| **内存修改** | 修改内存中的数值 | 关键数据存服务器 |

### 5.2 服务器验证原则

```
核心原则：永远不要信任客户端！

❌ 错误做法：
客户端：我要购买物品ID=5，价格=10金币
服务器：好的，扣除10金币

✅ 正确做法：
客户端：我要购买物品ID=5
服务器：让我查一下...物品5价格是100金币
        你有200金币，扣除100，购买成功
```

### 5.3 验证示例

```javascript
// ============================================
// 服务器端验证
// ============================================

class ValidationManager {
    // 验证移动
    validateMove(player, newX, newY, timestamp) {
        // 1. 时间戳检查（防重放）
        if (timestamp < player.lastMoveTime) {
            return false;
        }

        // 2. 速度检查
        var elapsed = (Date.now() - player.lastMoveTime) / 1000;
        var maxDistance = player.speed * elapsed * 1.5;  // 允许50%误差
        var actualDistance = Math.sqrt(
            Math.pow(newX - player.x, 2) +
            Math.pow(newY - player.y, 2)
        );
        if (actualDistance > maxDistance) {
            console.log('速度作弊:', player.id, actualDistance, maxDistance);
            return false;
        }

        // 3. 地图边界检查
        if (newX < 0 || newX > mapWidth || newY < 0 || newY > mapHeight) {
            return false;
        }

        // 4. 碰撞检查
        if (map.isBlocked(newX, newY)) {
            return false;
        }

        return true;
    }

    // 验证战斗
    validateAttack(attacker, target, damage, timestamp) {
        // 1. 距离检查
        var distance = Math.sqrt(
            Math.pow(attacker.x - target.x, 2) +
            Math.pow(attacker.y - target.y, 2)
        );
        if (distance > attacker.attackRange) {
            console.log('攻击距离作弊');
            return false;
        }

        // 2. 伤害检查
        var maxDamage = attacker.getTotalAttack() * 1.2;  // 允许20%浮动
        if (damage > maxDamage) {
            console.log('伤害作弊', damage, maxDamage);
            return false;
        }

        // 3. 冷却检查
        if (timestamp - attacker.lastAttackTime < attacker.attackCooldown) {
            console.log('攻击速度作弊');
            return false;
        }

        return true;
    }

    // 验证交易
    validateTrade(player, itemId, price) {
        // 价格必须从服务器数据库读取
        var itemData = itemDatabase[itemId];
        if (itemData.price !== price) {
            console.log('交易价格作弊');
            return false;
        }

        // 检查金币是否足够
        if (player.gold < price) {
            return false;
        }

        return true;
    }
}
```

### 5.4 数据加密

```javascript
// ============================================
// 简单的消息签名
// ============================================

const crypto = require('crypto');
const SECRET_KEY = 'your-secret-key-here';

function signMessage(message) {
    var data = JSON.stringify({
        type: message.type,
        payload: message.payload,
        timestamp: message.timestamp,
        sequence: message.sequence
    });
    var signature = crypto
        .createHmac('sha256', SECRET_KEY)
        .update(data)
        .digest('hex');
    return signature;
}

function verifyMessage(message) {
    var expectedSignature = signMessage(message);
    return message.signature === expectedSignature;
}

// 客户端发送
function sendSignedMessage(type, payload) {
    var message = {
        type: type,
        payload: payload,
        timestamp: Date.now(),
        sequence: nextSequence++
    };
    message.signature = signMessage(message);
    socket.send(JSON.stringify(message));
}

// 服务器验证
function handleSignedMessage(message) {
    if (!verifyMessage(message)) {
        console.log('消息签名验证失败');
        return;
    }
    // 处理消息...
}
```

---

## 第六部分：实际案例

### 6.1 简单聊天室 MMO

```javascript
// ============================================
// server.js - 完整的聊天+移动服务器
// ============================================

const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

class SimpleMMO {
    constructor() {
        this.players = new Map();
    }

    onConnect(ws) {
        var id = this.generateId();
        var player = {
            id: id,
            ws: ws,
            name: 'Player' + id.substr(0, 4),
            x: Math.random() * 800,
            y: Math.random() * 600,
            color: '#' + Math.floor(Math.random()*16777215).toString(16)
        };

        this.players.set(id, player);

        // 发送初始化数据
        this.send(ws, {
            type: 'init',
            id: id,
            x: player.x,
            y: player.y,
            players: this.getAllPlayers(id)
        });

        // 广播新玩家加入
        this.broadcast({
            type: 'join',
            id: id,
            name: player.name,
            x: player.x,
            y: player.y,
            color: player.color
        }, id);

        console.log('玩家连接:', id, '当前人数:', this.players.size);
    }

    onMessage(id, data) {
        var player = this.players.get(id);
        if (!player) return;

        switch(data.type) {
            case 'move':
                player.x = Math.max(0, Math.min(800, data.x));
                player.y = Math.max(0, Math.min(600, data.y));
                this.broadcast({
                    type: 'move',
                    id: id,
                    x: player.x,
                    y: player.y
                }, id);
                break;

            case 'chat':
                this.broadcast({
                    type: 'chat',
                    id: id,
                    name: player.name,
                    message: data.message
                });
                break;
        }
    }

    onDisconnect(id) {
        this.players.delete(id);
        this.broadcast({
            type: 'leave',
            id: id
        });
        console.log('玩家断开:', id, '当前人数:', this.players.size);
    }

    send(ws, data) {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(data));
        }
    }

    broadcast(data, excludeId) {
        var json = JSON.stringify(data);
        this.players.forEach((player, id) => {
            if (id !== excludeId) {
                this.send(player.ws, data);
            }
        });
    }

    getAllPlayers(excludeId) {
        var result = [];
        this.players.forEach((player, id) => {
            if (id !== excludeId) {
                result.push({
                    id: id,
                    name: player.name,
                    x: player.x,
                    y: player.y,
                    color: player.color
                });
            }
        });
        return result;
    }

    generateId() {
        return Math.random().toString(36).substr(2, 9);
    }
}

var game = new SimpleMMO();

wss.on('connection', function(ws) {
    var playerId = null;

    ws.on('message', function(message) {
        var data = JSON.parse(message);

        if (data.type === 'init') {
            playerId = game.onConnect(ws);
        } else if (playerId) {
            game.onMessage(playerId, data);
        }
    });

    ws.on('close', function() {
        if (playerId) {
            game.onDisconnect(playerId);
        }
    });
});

console.log('MMO 服务器启动在 ws://localhost:8080');
```

### 6.2 客户端

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>简单 MMO</title>
    <style>
        body { margin: 0; background: #1a1a2e; display: flex; flex-direction: column; align-items: center; }
        canvas { border: 2px solid #fff; margin: 20px; }
        #chat { width: 812px; height: 150px; background: rgba(0,0,0,0.8); color: #fff; overflow-y: auto; padding: 4px; }
        #input { width: 800px; padding: 5px; margin-top: 10px; }
    </style>
</head>
<body>
    <canvas id="game" width="800" height="600"></canvas>
    <div id="chat"></div>
    <input id="input" placeholder="输入消息，按回车发送">

    <script>
        var canvas = document.getElementById('game');
        var ctx = canvas.getContext('2d');
        var chatDiv = document.getElementById('chat');
        var input = document.getElementById('input');

        var socket = new WebSocket('ws://localhost:8080');
        var myId = null;
        var players = {};
        var myPlayer = { x: 0, y: 0, color: '#fff' };
        var keys = {};

        // 键盘输入
        document.addEventListener('keydown', e => keys[e.key] = true);
        document.addEventListener('keyup', e => keys[e.key] = false);

        // 发送消息
        input.addEventListener('keypress', e => {
            if (e.key === 'Enter' && input.value) {
                socket.send(JSON.stringify({
                    type: 'chat',
                    message: input.value
                }));
                input.value = '';
            }
        });

        // WebSocket 消息
        socket.onmessage = function(e) {
            var data = JSON.parse(e.data);

            switch(data.type) {
                case 'init':
                    myId = data.id;
                    myPlayer.x = data.x;
                    myPlayer.y = data.y;
                    players = {};
                    data.players.forEach(p => players[p.id] = p);
                    break;

                case 'join':
                    players[data.id] = data;
                    addChat(data.name + ' 加入了游戏');
                    break;

                case 'leave':
                    delete players[data.id];
                    addChat(players[data.id]?.name + ' 离开了游戏');
                    break;

                case 'move':
                    if (players[data.id]) {
                        players[data.id].x = data.x;
                        players[data.id].y = data.y;
                    }
                    break;

                case 'chat':
                    addChat(data.name + ': ' + data.message);
                    break;
            }
        };

        function addChat(msg) {
            chatDiv.innerHTML += '<div>' + msg + '</div>';
            chatDiv.scrollTop = chatDiv.scrollHeight;
        }

        // 游戏循环
        function gameLoop() {
            // 移动
            var speed = 5;
            if (keys['ArrowUp'] || keys['w']) myPlayer.y -= speed;
            if (keys['ArrowDown'] || keys['s']) myPlayer.y += speed;
            if (keys['ArrowLeft'] || keys['a']) myPlayer.x -= speed;
            if (keys['ArrowRight'] || keys['d']) myPlayer.x += speed;

            // 边界
            myPlayer.x = Math.max(0, Math.min(800, myPlayer.x));
            myPlayer.y = Math.max(0, Math.min(600, myPlayer.y));

            // 发送位置
            if (socket.readyState === WebSocket.OPEN) {
                socket.send(JSON.stringify({
                    type: 'move',
                    x: myPlayer.x,
                    y: myPlayer.y
                }));
            }

            // 渲染
            ctx.fillStyle = '#1a1a2e';
            ctx.fillRect(0, 0, 800, 600);

            // 画其他玩家
            for (var id in players) {
                var p = players[id];
                ctx.fillStyle = p.color;
                ctx.fillRect(p.x - 16, p.y - 16, 32, 32);
                ctx.fillStyle = '#fff';
                ctx.font = '12px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(p.name, p.x, p.y - 25);
            }

            // 画自己
            ctx.fillStyle = myPlayer.color;
            ctx.fillRect(myPlayer.x - 16, myPlayer.y - 16, 32, 32);
            ctx.fillStyle = '#ff0';
            ctx.font = '12px Arial';
            ctx.fillText('(你)', myPlayer.x, myPlayer.y - 25);

            requestAnimationFrame(gameLoop);
        }

        gameLoop();
    </script>
</body>
</html>
```

---

## 第七部分：扩展阅读

### 7.1 推荐技术栈

**小型 MMO（<1000人）**：
- 后端：Node.js + Socket.io
- 数据库：MongoDB + Redis
- 部署：单服务器 + PM2

**中型 MMO（1000-10000人）**：
- 后端：Node.js / Go / Java
- 数据库：MySQL + Redis
- 部署：多服务器 + Nginx 负载均衡

**大型 MMO（10000+人）**：
- 后端：Go / C++ / Rust
- 数据库：MySQL 分片 + Redis 集群
- 部署：Kubernetes + 微服务

### 7.2 学习资源

- [Gaffer On Games](https://gafferongames.com/) - 网络游戏开发经典文章
- 《网络游戏核心技术与实战》
- 《大规模多人在线游戏开发》
- [WebRTC for Games](https://webrtc.org/)

---

## 总结

从单机 RPG 到 MMO，需要掌握：

1. **网络编程**：WebSocket、TCP/UDP
2. **服务器架构**：从小到大逐步扩展
3. **数据同步**：状态同步、插值、预测
4. **数据库**：MySQL、Redis
5. **安全性**：永远不信任客户端

**最重要的一点**：先做小的，能跑起来，再逐步扩展！
