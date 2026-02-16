# 安全性：防作弊与数据保护

## 本章目标

理解 MMO 安全的核心原则，学会防御常见攻击。

## 核心原则

> **永远不要信任客户端！**
> 
> 这是最重要的规则。任何来自客户端的数据都必须经过服务器验证。

## 常见作弊方式

| 作弊类型 | 原理 | 防御方法 |
|---------|------|---------|
| 修改客户端 | 修改游戏代码/内存 | 服务器验证所有操作 |
| 加速器 | 加快游戏速度 | 时间戳检查 |
| 封包修改 | 修改网络数据包 | 加密 + 签名 |
| 重放攻击 | 重复发送有效数据包 | 序列号 + 时间戳 |

## 服务器验证示例

```javascript
// 验证移动
function validateMove(player, newX, newY) {
    // 1. 速度检查
    var distance = Math.sqrt(
        Math.pow(newX - player.x, 2) + 
        Math.pow(newY - player.y, 2)
    );
    if (distance > player.speed * 2) {
        console.log('可疑移动:', player.id);
        return false;
    }
    
    // 2. 边界检查
    if (newX < 0 || newX > mapWidth || newY < 0 || newY > mapHeight) {
        return false;
    }
    
    // 3. 碰撞检查
    if (map.isBlocked(newX, newY)) {
        return false;
    }
    
    return true;
}

// 验证交易
function validateTrade(player, itemId, price) {
    // 价格必须从服务器数据库读取，不能信任客户端
    var realPrice = itemDatabase[itemId].price;
    if (price !== realPrice) {
        console.log('交易价格作弊');
        return false;
    }
    
    if (player.gold < realPrice) {
        return false;
    }
    
    return true;
}
```

## 本章小结

- 所有逻辑在服务器执行
- 验证所有客户端数据
- 关键数据（金币、物品）服务器计算
