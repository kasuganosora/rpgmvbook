# 数据库文件详解

## 一、文件列表

| 文件 | 全局变量 | 说明 |
|------|----------|------|
| Actors.json | $dataActors | 角色数据 |
| Classes.json | $dataClasses | 职业数据 |
| Skills.json | $dataSkills | 技能数据 |
| Items.json | $dataItems | 物品数据 |
| Weapons.json | $dataWeapons | 武器数据 |
| Armors.json | $dataArmors | 防具数据 |
| Enemies.json | $dataEnemies | 敌人数据 |
| Troops.json | $dataTroops | 敌群数据 |
| States.json | $dataStates | 状态数据 |
| Animations.json | $dataAnimations | 动画数据 |
| Tilesets.json | $dataTilesets | 瓦片集数据 |
| System.json | $dataSystem | 系统设置 |

## 二、角色数据结构

```javascript
// Actors.json[1]
{
    "id": 1,
    "name": "哈罗德",
    "nickname": "",
    "classId": 1,
    "initialLevel": 1,
    "maxLevel": 99,
    "characterName": "Actor1",
    "characterIndex": 0,
    "faceName": "Actor1",
    "faceIndex": 0,
    "battlerName": "Actor1_1",
    "expParams": [30, 20, 10, 10],  // 经验曲线
    "params": [                     // 基础参数
        [100, 150, 200, ...],       // HP
        [50, 75, 100, ...],         // MP
        // ... 8个参数
    ],
    "traits": [...],                // 特性列表
    "note": ""                      // 备注
}
```

## 三、技能数据结构

```javascript
// Skills.json[1]
{
    "id": 1,
    "name": "攻击",
    "description": "",
    "iconIndex": 0,
    "mpCost": 0,
    "tpCost": 0,
    "scope": 1,         // 目标范围
    "occasion": 1,      // 使用场合
    "speed": 0,
    "successRate": 100,
    "repeats": 1,
    "hitType": 1,       // 命中类型
    "damage": {
        "type": 1,      // 伤害类型
        "formula": "a.atk * 4 - b.def * 2",
        "variance": 20,
        "critical": true
    },
    "effects": [...],
    "note": ""
}
```

## 四、调试案例

```javascript
// 获取角色1的HP基础值
var hp = $dataActors[1].params[0][1];  // 等级1的HP

// 获取技能的伤害公式
var formula = $dataSkills[1].damage.formula;

// 动态修改数据（临时）
$dataActors[1].name = "新名字";
$gameActors.actor(1).refresh();

// 遍历所有物品
$dataItems.forEach(function(item) {
    if (item) {
        console.log(item.name, item.price);
    }
});
```
