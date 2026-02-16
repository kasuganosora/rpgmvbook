# 服务器架构：从单服到微服务

## 本章目标

理解 MMO 服务器架构的演进过程，学会根据规模选择合适的架构。

## 架构演进概览

```
阶段 1：单服务器（100-1000 人）
┌────────────────┐
│   游戏服务器    │
│  ┌──────────┐  │
│  │ 逻辑+数据 │  │  ← 所有功能在一台机器
│  └──────────┘  │
└────────────────┘

阶段 2：分离数据库（1000-10000 人）
┌────────────────┐     ┌────────────┐
│   游戏服务器    │◄───►│   数据库   │
└────────────────┘     └────────────┘

阶段 3：多服务器 + 网关（10000-100000 人）
┌────────┐  ┌────────┐  ┌────────┐
│ 游戏服1 │  │ 游戏服2 │  │ 游戏服3 │
└───┬────┘  └───┬────┘  └───┬────┘
    └───────────┼───────────┘
                │
          ┌─────▼─────┐
          │   网关    │
          └─────┬─────┘
                │
          ┌─────▼─────┐
          │   数据库   │
          └───────────┘

阶段 4：微服务（10万+ 人）
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│登录服│ │地图服│ │战斗服│ │聊天服│  ← 功能拆分
└──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
   └────────┴────────┴────────┘
                │
          ┌─────▼─────┐
          │ 消息队列  │
          └─────┬─────┘
                │
          ┌─────▼─────┐
          │  数据库   │
          └───────────┘
```

## 阶段 1：单服务器架构

### 最简单的架构

所有功能都在一个 Node.js 进程里：

```javascript
// ============================================
// 单服务器架构 - 适合 100-1000 人
// ============================================

const WebSocket = require('ws');
const fs = require('fs');

class SimpleGameServer {
    constructor() {
        // 为什么用 Map 而不是 Object？
        // Map 有 size 属性，键可以是任意类型，性能更好
        this.players = new Map();       // 在线玩家
        this.maps = new Map();          // 地图数据
        this.items = new Map();         // 物品数据
        
        // 加载静态数据
        this.loadData();
    }
    
    // 加载游戏数据
    loadData() {
        // 为什么启动时加载？
        // 避免每次请求都读文件，提高性能
        this.itemData = JSON.parse(fs.readFileSync('data/items.json'));
        this.mapData = JSON.parse(fs.readFileSync('data/maps.json'));
    }
    
    // 玩家连接
    onConnect(ws) {
        var playerId = this.generateId();
        
        var player = {
            id: playerId,
            ws: ws,                    // WebSocket 连接
            name: '',
            x: 100,
            y: 100,
            mapId: 1,
            hp: 100,
            maxHp: 100,
            mp: 30,
            maxMp: 30,
            level: 1,
            exp: 0,
            gold: 100,
            inventory: [],             // 背包
            lastActiveTime: Date.now()
        };
        
        this.players.set(playerId, player);
        
        // 发送初始化数据
        this.send(ws, {
            type: 'init',
            playerId: playerId,
            player: this.getPlayerData(player)
        });
        
        console.log('玩家连接:', playerId, '当前在线:', this.players.size);
        return playerId;
    }
    
    // 处理消息
    onMessage(playerId, data) {
        var player = this.players.get(playerId);
        if (!player) return;
        
        player.lastActiveTime = Date.now();
        
        switch(data.type) {
            case 'login':
                this.handleLogin(player, data);
                break;
            case 'move':
                this.handleMove(player, data);
                break;
            case 'chat':
                this.handleChat(player, data);
                break;
            case 'attack':
                this.handleAttack(player, data);
                break;
            case 'useItem':
                this.handleUseItem(player, data);
                break;
        }
    }
    
    // 处理登录
    handleLogin(player, data) {
        player.name = data.name || ('Player' + player.id.substr(0, 4));
        
        // 广播玩家加入
        this.broadcast({
            type: 'playerJoin',
            player: this.getPlayerData(player)
        }, player.id);
        
        // 发送在线玩家列表
        var onlinePlayers = [];
        this.players.forEach((p, id) => {
            if (id !== player.id && p.mapId === player.mapId) {
                onlinePlayers.push(this.getPlayerData(p));
            }
        });
        
        this.send(player.ws, {
            type: 'playerList',
            players: onlinePlayers
        });
    }
    
    // 处理移动
    handleMove(player, data) {
        // 验证移动是否合法
        if (!this.validateMove(player, data.x, data.y)) {
            // 作弊检测
            this.send(player.ws, {
                type: 'error',
                message: '移动无效'
            });
            return;
        }
        
        player.x = data.x;
        player.y = data.y;
        
        // 广播给同地图玩家
        this.broadcastInMap(player.mapId, {
            type: 'playerMove',
            playerId: player.id,
            x: player.x,
            y: player.y,
            direction: data.direction
        }, player.id);
    }
    
    // 验证移动
    validateMove(player, newX, newY) {
        // 1. 边界检查
        if (newX < 0 || newX > 1000 || newY < 0 || newY > 1000) {
            return false;
        }
        
        // 2. 速度检查（防加速）
        var distance = Math.sqrt(
            Math.pow(newX - player.x, 2) + 
            Math.pow(newY - player.y, 2)
        );
        if (distance > 10) {  // 单次移动不能超过 10 单位
            return false;
        }
        
        return true;
    }
    
    // 处理聊天
    handleChat(player, data) {
        // 过滤敏感词（实际项目需要更完善的过滤）
        var message = data.message.replace(/傻逼|操你/g, '**');
        
        this.broadcast({
            type: 'chat',
            playerId: player.id,
            playerName: player.name,
            message: message
        });
    }
    
    // 处理攻击
    handleAttack(player, data) {
        var targetId = data.targetId;
        var target = this.players.get(targetId);
        
        if (!target) return;
        
        // 验证攻击距离
        var distance = Math.sqrt(
            Math.pow(player.x - target.x, 2) + 
            Math.pow(player.y - target.y, 2)
        );
        
        if (distance > 50) {
            this.send(player.ws, {
                type: 'error',
                message: '目标太远'
            });
            return;
        }
        
        // 计算伤害
        var damage = Math.floor(Math.random() * 10) + 5;
        target.hp -= damage;
        
        // 广播攻击事件
        this.broadcastInMap(player.mapId, {
            type: 'attack',
            attackerId: player.id,
            targetId: target.id,
            damage: damage,
            targetHp: target.hp
        });
        
        // 检查死亡
        if (target.hp <= 0) {
            this.handleDeath(target, player);
        }
    }
    
    // 处理死亡
    handleDeath(deadPlayer, killer) {
        deadPlayer.hp = deadPlayer.maxHp;
        deadPlayer.x = 100;
        deadPlayer.y = 100;
        
        // 广播死亡
        this.broadcast({
            type: 'playerDeath',
            deadId: deadPlayer.id,
            killerId: killer.id,
            killerName: killer.name
        });
        
        // 复活
        this.send(deadPlayer.ws, {
            type: 'revive',
            x: deadPlayer.x,
            y: deadPlayer.y,
            hp: deadPlayer.hp
        });
    }
    
    // 处理使用物品
    handleUseItem(player, data) {
        var slot = data.slot;
        var item = player.inventory[slot];
        
        if (!item) return;
        
        var itemData = this.itemData[item.id];
        if (!itemData) return;
        
        // 应用效果
        if (itemData.type === 'potion') {
            if (itemData.effect === 'hp') {
                player.hp = Math.min(player.maxHp, player.hp + itemData.value);
            } else if (itemData.effect === 'mp') {
                player.mp = Math.min(player.maxMp, player.mp + itemData.value);
            }
        }
        
        // 扣除数量
        item.count--;
        if (item.count <= 0) {
            player.inventory.splice(slot, 1);
        }
        
        // 通知
        this.send(player.ws, {
            type: 'useItem',
            slot: slot,
            item: item,
            hp: player.hp,
            mp: player.mp
        });
    }
    
    // 玩家断开
    onDisconnect(playerId) {
        var player = this.players.get(playerId);
        if (player) {
            this.broadcast({
                type: 'playerLeave',
                playerId: playerId
            });
            this.players.delete(playerId);
            console.log('玩家断开:', playerId, '当前在线:', this.players.size);
        }
    }
    
    // ========== 工具方法 ==========
    
    send(ws, data) {
        if (ws && ws.readyState === 1) {  // WebSocket.OPEN = 1
            ws.send(JSON.stringify(data));
        }
    }
    
    // 广播给所有人
    broadcast(data, excludeId) {
        var json = JSON.stringify(data);
        this.players.forEach((player, id) => {
            if (id !== excludeId) {
                this.send(player.ws, data);
            }
        });
    }
    
    // 广播给同地图玩家
    broadcastInMap(mapId, data, excludeId) {
        this.players.forEach((player, id) => {
            if (player.mapId === mapId && id !== excludeId) {
                this.send(player.ws, data);
            }
        });
    }
    
    getPlayerData(player) {
        return {
            id: player.id,
            name: player.name,
            x: player.x,
            y: player.y,
            mapId: player.mapId,
            hp: player.hp,
            maxHp: player.maxHp,
            level: player.level
        };
    }
    
    generateId() {
        return Date.now().toString(36) + Math.random().toString(36).substr(2, 5);
    }
}

// 启动服务器
const wss = new WebSocket.Server({ port: 8080 });
const game = new SimpleGameServer();

wss.on('connection', function(ws) {
    var playerId = game.onConnect(ws);
    
    ws.on('message', function(message) {
        try {
            var data = JSON.parse(message);
            game.onMessage(playerId, data);
        } catch(e) {
            console.error('消息解析错误:', e);
        }
    });
    
    ws.on('close', function() {
        game.onDisconnect(playerId);
    });
});

console.log('游戏服务器启动在 ws://localhost:8080');
```

### 单服务器架构的优缺点

| 优点 | 缺点 |
|------|------|
| 实现简单 | 只能单核运行（Node.js 单线程） |
| 部署方便 | 内存有限 |
| 调试容易 | 无法水平扩展 |
| 成本低 | 单点故障 |

## 阶段 2：分离数据库

当玩家数据需要持久化时，加入数据库：

### 架构图

```
┌──────────────────┐
│   游戏服务器      │
│                  │
│  ┌────────────┐  │
│  │ 游戏逻辑   │  │
│  │ 网络处理   │  │
│  └─────┬──────┘  │
│        │         │
│  ┌─────▼──────┐  │
│  │ 数据访问层 │  │  ← 抽象数据库操作
│  └─────┬──────┘  │
└────────┼─────────┘
         │
         │ MySQL 协议
         │
┌────────▼─────────┐
│     MySQL        │
│   ┌──────────┐   │
│   │ 账号表   │   │
│   │ 角色表   │   │
│   │ 物品表   │   │
│   └──────────┘   │
└──────────────────┘
```

### 数据访问层

```javascript
// ============================================
// 数据库访问层 - 抽象数据库操作
// ============================================

const mysql = require('mysql2/promise');

class Database {
    constructor(config) {
        // 为什么用连接池？
        // 避免每次请求都创建新连接，提高性能
        this.pool = mysql.createPool({
            host: config.host || 'localhost',
            user: config.user || 'root',
            password: config.password || '',
            database: config.database || 'game',
            waitForConnections: true,
            connectionLimit: 10,  // 最大连接数
            queueLimit: 0
        });
    }
    
    // ========== 账号相关 ==========
    
    // 创建账号
    async createAccount(username, passwordHash, email) {
        try {
            var [result] = await this.pool.execute(
                'INSERT INTO accounts (username, password_hash, email) VALUES (?, ?, ?)',
                [username, passwordHash, email]
            );
            return { success: true, accountId: result.insertId };
        } catch(e) {
            if (e.code === 'ER_DUP_ENTRY') {
                return { success: false, error: '用户名已存在' };
            }
            return { success: false, error: '数据库错误' };
        }
    }
    
    // 验证登录
    async verifyLogin(username, passwordHash) {
        var [rows] = await this.pool.execute(
            'SELECT id FROM accounts WHERE username = ? AND password_hash = ?',
            [username, passwordHash]
        );
        return rows.length > 0 ? rows[0].id : null;
    }
    
    // ========== 角色相关 ==========
    
    // 获取角色
    async getCharacter(accountId) {
        var [rows] = await this.pool.execute(
            'SELECT * FROM characters WHERE account_id = ?',
            [accountId]
        );
        return rows.length > 0 ? rows[0] : null;
    }
    
    // 创建角色
    async createCharacter(accountId, name) {
        try {
            var [result] = await this.pool.execute(
                `INSERT INTO characters 
                (account_id, name, level, exp, hp, mp, x, y, map_id) 
                VALUES (?, ?, 1, 0, 100, 30, 100, 100, 1)`,
                [accountId, name]
            );
            return { success: true, characterId: result.insertId };
        } catch(e) {
            if (e.code === 'ER_DUP_ENTRY') {
                return { success: false, error: '角色名已存在' };
            }
            return { success: false, error: '数据库错误' };
        }
    }
    
    // 保存角色
    async saveCharacter(characterId, data) {
        await this.pool.execute(
            `UPDATE characters SET 
            level = ?, exp = ?, hp = ?, mp = ?, 
            x = ?, y = ?, map_id = ?, gold = ?
            WHERE id = ?`,
            [data.level, data.exp, data.hp, data.mp, 
             data.x, data.y, data.mapId, data.gold, characterId]
        );
    }
    
    // ========== 物品相关 ==========
    
    // 获取背包
    async getInventory(characterId) {
        var [rows] = await this.pool.execute(
            'SELECT * FROM items WHERE character_id = ? ORDER BY slot',
            [characterId]
        );
        return rows;
    }
    
    // 添加物品
    async addItem(characterId, itemId, count, slot) {
        await this.pool.execute(
            'INSERT INTO items (character_id, item_id, count, slot) VALUES (?, ?, ?, ?)',
            [characterId, itemId, count, slot]
        );
    }
    
    // 更新物品数量
    async updateItemCount(itemId, count) {
        if (count <= 0) {
            await this.pool.execute('DELETE FROM items WHERE id = ?', [itemId]);
        } else {
            await this.pool.execute(
                'UPDATE items SET count = ? WHERE id = ?',
                [count, itemId]
            );
        }
    }
}

// 使用示例
const db = new Database({
    host: 'localhost',
    user: 'game',
    password: 'password',
    database: 'rpg_game'
});

// 登录流程
async function handleLogin(ws, username, password) {
    var passwordHash = hashPassword(password);
    var accountId = await db.verifyLogin(username, passwordHash);
    
    if (!accountId) {
        return { success: false, error: '用户名或密码错误' };
    }
    
    var character = await db.getCharacter(accountId);
    if (!character) {
        return { success: false, error: '请先创建角色' };
    }
    
    var inventory = await db.getInventory(character.id);
    
    return {
        success: true,
        character: character,
        inventory: inventory
    };
}
```

### 定时保存

```javascript
// ============================================
// 定时保存玩家数据
// ============================================

class AutoSave {
    constructor(gameServer, db) {
        this.game = gameServer;
        this.db = db;
        
        // 每 5 分钟自动保存所有在线玩家
        // 为什么定时保存而不是每次操作都保存？
        // 减少数据库压力，提高性能
        setInterval(() => this.saveAll(), 5 * 60 * 1000);
    }
    
    async saveAll() {
        console.log('开始自动保存...');
        var count = 0;
        
        for (var [playerId, player] of this.game.players) {
            try {
                await this.db.saveCharacter(player.characterId, {
                    level: player.level,
                    exp: player.exp,
                    hp: player.hp,
                    mp: player.mp,
                    x: player.x,
                    y: player.y,
                    mapId: player.mapId,
                    gold: player.gold
                });
                count++;
            } catch(e) {
                console.error('保存失败:', playerId, e);
            }
        }
        
        console.log('自动保存完成，保存了', count, '个玩家');
    }
    
    // 玩家下线时立即保存
    async savePlayer(playerId) {
        var player = this.game.players.get(playerId);
        if (!player) return;
        
        await this.db.saveCharacter(player.characterId, {
            level: player.level,
            exp: player.exp,
            hp: player.hp,
            mp: player.mp,
            x: player.x,
            y: player.y,
            mapId: player.mapId,
            gold: player.gold
        });
    }
}
```

## 阶段 3：多服务器 + 网关

当单服务器扛不住时，需要多台服务器：

### 架构图

```
                    ┌─────────────┐
                    │   客户端    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   网关服    │  ← 负载均衡、路由
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │  游戏服 1   │  │  游戏服 2   │  │  游戏服 3   │
   │  (地图1-5)  │  │  (地图6-10) │  │  (地图11-15)│
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                    ┌──────▼──────┐
                    │   数据库    │
                    └─────────────┘
```

### 网关服务器

```javascript
// ============================================
// 网关服务器 - 负载均衡和路由
// ============================================

const WebSocket = require('ws');

class GatewayServer {
    constructor() {
        // 游戏服务器列表
        // 为什么用数组？方便轮询负载均衡
        this.gameServers = [
            { id: 'game1', url: 'ws://game1:8081', maps: [1, 2, 3, 4, 5], load: 0 },
            { id: 'game2', url: 'ws://game2:8081', maps: [6, 7, 8, 9, 10], load: 0 },
            { id: 'game3', url: 'ws://game3:8081', maps: [11, 12, 13, 14, 15], load: 0 }
        ];
        
        // 玩家 -> 游戏服映射
        this.playerServerMap = new Map();
        
        // 连接所有游戏服
        this.connectToGameServers();
    }
    
    // 连接游戏服
    connectToGameServers() {
        this.gameServers.forEach(server => {
            server.ws = new WebSocket(server.url);
            
            server.ws.on('open', () => {
                console.log('连接到游戏服:', server.id);
            });
            
            server.ws.on('message', (data) => {
                // 收到游戏服消息，转发给对应客户端
                try {
                    var msg = JSON.parse(data);
                    this.forwardToClient(msg);
                } catch(e) {
                    console.error('消息解析错误');
                }
            });
            
            server.ws.on('close', () => {
                console.log('游戏服断开:', server.id);
                // 尝试重连
                setTimeout(() => this.connectToGameServer(server), 5000);
            });
        });
    }
    
    // 客户端连接
    onClientConnect(ws) {
        var playerId = this.generateId();
        ws.playerId = playerId;
        
        // 暂存连接，等登录后分配游戏服
        ws.send(JSON.stringify({
            type: 'connected',
            playerId: playerId
        }));
    }
    
    // 处理客户端消息
    onClientMessage(ws, data) {
        try {
            var msg = JSON.parse(data);
            
            // 登录时分配游戏服
            if (msg.type === 'login') {
                this.handleLogin(ws, msg);
            } else {
                // 转发到对应游戏服
                var serverId = this.playerServerMap.get(ws.playerId);
                if (serverId) {
                    var server = this.gameServers.find(s => s.id === serverId);
                    if (server && server.ws.readyState === WebSocket.OPEN) {
                        msg.playerId = ws.playerId;
                        server.ws.send(JSON.stringify(msg));
                    }
                }
            }
        } catch(e) {
            console.error('处理消息错误:', e);
        }
    }
    
    // 处理登录
    async handleLogin(ws, msg) {
        // 验证账号（可以单独的登录服）
        var account = await this.verifyAccount(msg.username, msg.password);
        if (!account) {
            ws.send(JSON.stringify({ type: 'loginFailed', error: '账号或密码错误' }));
            return;
        }
        
        // 获取角色信息
        var character = await this.getCharacter(account.id);
        
        // 根据地图选择游戏服
        var server = this.selectServer(character.mapId);
        
        // 记录映射
        this.playerServerMap.set(ws.playerId, server.id);
        server.load++;
        
        // 转发登录请求到游戏服
        server.ws.send(JSON.stringify({
            type: 'playerLogin',
            playerId: ws.playerId,
            wsClientId: ws.playerId,  // 用于网关回传
            account: account,
            character: character
        }));
    }
    
    // 选择游戏服
    selectServer(mapId) {
        // 先按地图找
        for (var server of this.gameServers) {
            if (server.maps.includes(mapId)) {
                return server;
            }
        }
        
        // 地图不匹配，选负载最低的
        return this.gameServers.reduce((min, s) => s.load < min.load ? s : min);
    }
    
    // 转发给客户端
    forwardToClient(msg) {
        var ws = this.getClientConnection(msg.wsClientId);
        if (ws && ws.readyState === WebSocket.OPEN) {
            delete msg.wsClientId;
            ws.send(JSON.stringify(msg));
        }
    }
}
```

## 阶段 4：微服务架构

对于大型 MMO，需要把功能拆分：

### 服务拆分

```
┌────────────────────────────────────────────────────┐
│                    客户端                          │
└───────────────────────┬────────────────────────────┘
                        │
                 ┌──────▼──────┐
                 │   API 网关   │
                 └──────┬──────┘
                        │
    ┌───────────────────┼───────────────────┐
    │                   │                   │
┌───▼───┐          ┌───▼───┐          ┌───▼───┐
│登录服  │          │游戏服  │          │聊天服  │
│       │          │       │          │       │
│认证    │          │地图    │          │私聊    │
│注册    │          │战斗    │          │频道    │
└───┬───┘          │物品    │          └───┬───┘
    │              └───┬───┘              │
    │                  │                  │
    └──────────────────┼──────────────────┘
                       │
                ┌──────▼──────┐
                │  消息队列    │  ← 服务间通信
                │  (Redis)    │
                └──────┬──────┘
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
┌───▼───┐          ┌───▼───┐          ┌───▼───┐
│ MySQL │          │ Redis │          │MongoDB│
│账号    │          │缓存    │          │日志    │
│角色    │          │会话    │          │分析    │
└───────┘          └───────┘          └───────┘
```

### 服务间通信

```javascript
// ============================================
// 使用 Redis 发布订阅进行服务间通信
// ============================================

const redis = require('redis');

class ServiceBus {
    constructor(serviceName) {
        this.serviceName = serviceName;
        this.publisher = redis.createClient();
        this.subscriber = redis.createClient();
        
        // 订阅自己的频道
        this.subscriber.subscribe(serviceName);
        
        this.handlers = new Map();
        
        this.subscriber.on('message', (channel, message) => {
            try {
                var msg = JSON.parse(message);
                var handler = this.handlers.get(msg.type);
                if (handler) {
                    handler(msg);
                }
            } catch(e) {
                console.error('消息处理错误:', e);
            }
        });
    }
    
    // 发送消息给其他服务
    send(targetService, type, payload) {
        this.publisher.publish(targetService, JSON.stringify({
            from: this.serviceName,
            type: type,
            payload: payload,
            timestamp: Date.now()
        }));
    }
    
    // 注册消息处理器
    on(type, handler) {
        this.handlers.set(type, handler);
    }
}

// 使用示例
// 登录服
var loginBus = new ServiceBus('login-service');

loginBus.on('validateToken', (msg) => {
    // 验证 token
    var valid = validateToken(msg.payload.token);
    
    // 回复
    loginBus.send(msg.from, 'tokenResult', {
        requestId: msg.payload.requestId,
        valid: valid
    });
});

// 游戏服
var gameBus = new ServiceBus('game-service');

async function onPlayerJoin(token) {
    // 请求登录服验证 token
    gameBus.send('login-service', 'validateToken', {
        requestId: generateId(),
        token: token
    });
}
```

## 架构选择指南

| 在线人数 | 推荐架构 | 成本 | 开发难度 |
|---------|---------|------|---------|
| < 100 | 单服务器（无数据库） | 很低 | ★ |
| 100-1000 | 单服务器 + 数据库 | 低 | ★★ |
| 1000-10000 | 多服务器 + 网关 | 中 | ★★★ |
| 10000-100000 | 微服务 | 高 | ★★★★ |
| > 100000 | 分布式微服务 + 定制 | 很高 | ★★★★★ |

## 本章小结

| 阶段 | 核心变化 | 解决的问题 |
|------|---------|-----------|
| 单服务器 | 所有功能在一起 | 能跑起来 |
| 分离数据库 | 数据持久化 | 数据不丢失 |
| 多服务器 | 负载分散 | 承载更多人 |
| 微服务 | 功能拆分 | 独立扩展、容错 |

## 下一章

[数据同步](03-data-sync.md) - 学习如何让多个玩家看到一致的游戏世界。
