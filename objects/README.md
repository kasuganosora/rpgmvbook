# 游戏对象系统

## 概述

游戏对象是纯数据模型，不包含渲染逻辑。它们存储游戏运行时的状态，由精灵类负责可视化。

## 对象继承关系

```
Game_BattlerBase
└── Game_Battler
        ├── Game_Actor (玩家角色)
        └── Game_Enemy (敌人)

Game_CharacterBase
└── Game_Character
        ├── Game_Player (玩家)
        ├── Game_Event (事件)
        ├── Game_Vehicle (交通工具)
        └── Game_Follower (跟随者)

Game_BaseAction
└── Game_Action (战斗动作)
```

## 全局对象

| 对象 | 类型 | 说明 |
|------|------|------|
| $gameTemp | Game_Temp | 临时数据 |
| $gameSystem | Game_System | 系统状态 |
| $gameScreen | Game_Screen | 屏幕效果 |
| $gameTimer | Game_Timer | 计时器 |
| $gameMessage | Game_Message | 消息框状态 |
| $gameSwitches | Game_Switches | 开关 |
| $gameVariables | Game_Variables | 变量 |
| $gameSelfSwitches | Game_SelfSwitches | 独立开关 |
| $gameActors | Game_Actors | 角色集合 |
| $gameParty | Game_Party | 队伍 |
| $gameTroop | Game_Troop | 敌群 |
| $gameMap | Game_Map | 地图 |
| $gamePlayer | Game_Player | 玩家 |

## 章节导航

- [Game_Temp - 临时数据](game-temp.md)
- [Game_System - 系统数据](game-system.md)
- [Game_Switches & Variables - 开关与变量](game-switches-variables.md)
- [Game_Screen - 屏幕效果](game-screen.md)
- [Game_Map - 地图对象](game-map.md)
- [Game_Player - 玩家对象](game-player.md)
- [Game_Event - 事件对象](game-event.md)
- [Game_Character - 角色基类](game-character.md)
- [Game_Actor & Enemy - 战斗角色](game-actor-enemy.md)
