# 存档数据结构

## 一、存档内容

```javascript
// DataManager.makeSaveContents()
{
    system: $gameSystem,        // 系统设置
    screen: $gameScreen,        // 画面效果
    timer: $gameTimer,          // 计时器
    switches: $gameSwitches,    // 开关
    variables: $gameVariables,  // 变量
    selfSwitches: $gameSelfSwitches,  // 独立开关
    actors: $gameActors,        // 角色数据
    party: $gameParty,          // 队伍
    map: $gameMap,              // 地图状态
    player: $gamePlayer         // 玩家位置
}
```

## 二、关键数据详解

### 2.1 Party数据

```javascript
$gameParty = {
    _gold: 1000,              // 金币
    _steps: 500,              // 步数
    _lastItem: null,          // 最后使用的物品
    _menuActorId: 1,          // 菜单选中角色
    _targetActorId: 0,        // 目标角色
    _actors: [1, 2, 3, 4],    // 队伍成员ID
    _items: {1: 5, 3: 2},     // 物品 {ID: 数量}
    _weapons: {1: 2},         // 武器
    _armors: {1: 3}           // 防具
};
```

### 2.2 Player数据

```javascript
$gamePlayer = {
    _x: 8,                    // X坐标
    _y: 6,                    // Y坐标
    _direction: 2,            // 方向
    _mapId: 1,                // 地图ID
    _vehicleType: 'walk',     // 交通工具
    _moveSpeed: 4,            // 移动速度
    _transparent: false       // 透明
};
```

## 三、调试案例

```javascript
// 读取存档
var data = JSON.parse(LZString.decompressFromBase64(
    localStorage.getItem('RPG Save1')
));

console.log('金币:', data.party._gold);
console.log('位置:', data.player._x, data.player._y);

// 修改存档
data.party._gold = 99999;
localStorage.setItem('RPG Save1', 
    LZString.compressToBase64(JSON.stringify(data)));
```
