# 寻路算法

## 本章目标

理解 A* 算法原理，实现游戏中的寻路功能。

## 为什么需要寻路？

```
没有寻路：
敌人 ●─────────────► 墙 ─────────────► 撞墙！
                                      ★ 玩家

有寻路：
敌人 ●──►──►──►──►──►──►──►──►──►──► ★ 玩家
         绕过障碍物
```

## A* 算法原理

```
A* = Dijkstra + 启发式估计

公式：F = G + H

G = 从起点到当前点的实际距离
H = 从当前点到终点的估计距离（启发函数）
F = 总估计代价

选择 F 值最小的点继续探索
```

## A* 算法实现

```javascript
// ============================================
// A* 寻路算法
// ============================================

class PathFinder {
    constructor(map) {
        this.map = map;  // 地图数据
    }
    
    // 寻找路径
    findPath(startX, startY, endX, endY) {
        var openList = [];    // 待检查的节点
        var closedList = [];  // 已检查的节点
        
        // 起点节点
        var startNode = {
            x: startX,
            y: startY,
            g: 0,
            h: this.heuristic(startX, startY, endX, endY),
            f: 0,
            parent: null
        };
        startNode.f = startNode.g + startNode.h;
        
        openList.push(startNode);
        
        while (openList.length > 0) {
            // 找 F 值最小的节点
            openList.sort((a, b) => a.f - b.f);
            var current = openList.shift();
            
            // 到达终点
            if (current.x === endX && current.y === endY) {
                return this.reconstructPath(current);
            }
            
            // 加入已检查列表
            closedList.push(current);
            
            // 检查相邻节点
            var neighbors = this.getNeighbors(current.x, current.y);
            
            for (var neighbor of neighbors) {
                // 跳过已检查的
                if (closedList.find(n => n.x === neighbor.x && n.y === neighbor.y)) {
                    continue;
                }
                
                // 计算新的 G 值
                var newG = current.g + neighbor.cost;
                
                // 查找是否在 openList 中
                var existing = openList.find(n => n.x === neighbor.x && n.y === neighbor.y);
                
                if (existing) {
                    // 如果新路径更短，更新
                    if (newG < existing.g) {
                        existing.g = newG;
                        existing.f = existing.g + existing.h;
                        existing.parent = current;
                    }
                } else {
                    // 新节点
                    var newNode = {
                        x: neighbor.x,
                        y: neighbor.y,
                        g: newG,
                        h: this.heuristic(neighbor.x, neighbor.y, endX, endY),
                        f: 0,
                        parent: current
                    };
                    newNode.f = newNode.g + newNode.h;
                    openList.push(newNode);
                }
            }
        }
        
        // 没有找到路径
        return null;
    }
    
    // 启发函数（曼哈顿距离）
    heuristic(x1, y1, x2, y2) {
        return Math.abs(x1 - x2) + Math.abs(y1 - y2);
    }
    
    // 获取相邻节点
    getNeighbors(x, y) {
        var neighbors = [];
        var directions = [
            { dx: 0, dy: -1, cost: 1 },  // 上
            { dx: 0, dy: 1, cost: 1 },   // 下
            { dx: -1, dy: 0, cost: 1 },  // 左
            { dx: 1, dy: 0, cost: 1 }    // 右
        ];
        
        for (var dir of directions) {
            var nx = x + dir.dx;
            var ny = y + dir.dy;
            
            // 检查边界和障碍
            if (this.isWalkable(nx, ny)) {
                neighbors.push({ x: nx, y: ny, cost: dir.cost });
            }
        }
        
        return neighbors;
    }
    
    // 检查是否可通行
    isWalkable(x, y) {
        // 检查边界
        if (x < 0 || y < 0 || x >= this.map.width || y >= this.map.height) {
            return false;
        }
        
        // 检查障碍物
        return !this.map.isBlocked(x, y);
    }
    
    // 重建路径
    reconstructPath(endNode) {
        var path = [];
        var current = endNode;
        
        while (current) {
            path.unshift({ x: current.x, y: current.y });
            current = current.parent;
        }
        
        return path;
    }
}
```

## 使用示例

```javascript
// 创建寻路器
var pathFinder = new PathFinder(gameMap);

// 寻找路径
var path = pathFinder.findPath(enemy.x, enemy.y, player.x, player.y);

if (path) {
    // 沿路径移动
    enemy.followPath(path);
} else {
    // 无法到达
    console.log('无法到达目标');
}
```

## 简化路径

```javascript
// 减少路径点，让移动更平滑
function simplifyPath(path) {
    if (path.length <= 2) return path;
    
    var simplified = [path[0]];
    
    for (var i = 1; i < path.length - 1; i++) {
        var prev = simplified[simplified.length - 1];
        var curr = path[i];
        var next = path[i + 1];
        
        // 如果方向改变，保留这个点
        var dir1 = { x: curr.x - prev.x, y: curr.y - prev.y };
        var dir2 = { x: next.x - curr.x, y: next.y - curr.y };
        
        if (dir1.x !== dir2.x || dir1.y !== dir2.y) {
            simplified.push(curr);
        }
    }
    
    simplified.push(path[path.length - 1]);
    return simplified;
}
```

## 本章小结

1. **A\*** = Dijkstra + 启发式估计
2. **F = G + H**：总代价 = 已走距离 + 估计距离
3. **优化**：简化路径、缓存结果

## 下一章

[敌人 AI](05-enemy-ai.md) - 综合应用，实现完整的敌人 AI。
