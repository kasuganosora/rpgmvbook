# 数据库设计：存储游戏数据

## 本章目标

学会设计和使用数据库存储玩家数据。

## 数据库选型

| 数据库 | 特点 | 适用场景 |
|--------|------|---------|
| **MySQL** | 关系型、ACID 事务 | 账号、物品、交易 |
| **Redis** | 内存、快速 | 缓存、会话、排行榜 |
| **MongoDB** | 文档型、灵活 | 玩家数据、日志 |

## 表结构设计

```sql
-- 账号表
CREATE TABLE accounts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(32) UNIQUE NOT NULL,
    password_hash VARCHAR(64) NOT NULL,
    email VARCHAR(128),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 角色表
CREATE TABLE characters (
    id INT PRIMARY KEY AUTO_INCREMENT,
    account_id INT NOT NULL,
    name VARCHAR(16) UNIQUE NOT NULL,
    level INT DEFAULT 1,
    exp INT DEFAULT 0,
    hp INT DEFAULT 100,
    mp INT DEFAULT 30,
    x INT DEFAULT 100,
    y INT DEFAULT 100,
    map_id INT DEFAULT 1,
    gold INT DEFAULT 0
);

-- 物品表
CREATE TABLE items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    character_id INT NOT NULL,
    item_id INT NOT NULL,
    count INT DEFAULT 1,
    slot INT
);
```

## Redis 缓存

```javascript
// 获取玩家数据（先查缓存，再查数据库）
async getPlayer(playerId) {
    // 1. 查 Redis 缓存
    var cached = await redis.get('player:' + playerId);
    if (cached) {
        return JSON.parse(cached);
    }
    
    // 2. 查 MySQL 数据库
    var player = await db.query(
        'SELECT * FROM characters WHERE id = ?',
        [playerId]
    );
    
    // 3. 写入缓存（10 分钟过期）
    await redis.setex('player:' + playerId, 600, JSON.stringify(player));
    
    return player;
}
```

## 本章小结

- MySQL 存持久化数据（账号、角色、物品）
- Redis 做缓存层，加速读取
- 定时保存，避免频繁写数据库
