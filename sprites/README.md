# 精灵系统

## 概述

精灵是游戏对象的可视化表示，负责将数据模型渲染到屏幕。

## 精灵继承关系

```
PIXI.Container
└── Sprite
        ├── Sprite_Base (基础精灵)
        ├── Sprite_Character (角色精灵)
        ├── Sprite_Battler (战斗者精灵)
        │       ├── Sprite_Actor (角色)
        │       └── Sprite_Enemy (敌人)
        ├── Sprite_Picture (图片)
        ├── Sprite_StateIcon (状态图标)
        ├── Sprite_Balloon (气球图标)
        └── Sprite_Animation (动画)

PIXI.Container
└── Spriteset_Base
        └── Spriteset_Map (地图精灵集)
        └── Spriteset_Battle (战斗精灵集)
```

## 精灵集 (Spriteset)

精灵集管理一组相关精灵：

| 精灵集 | 包含内容 |
|--------|----------|
| Spriteset_Map | Tilemap, 角色精灵, 远景图, 天气, 图片 |
| Spriteset_Battle | 背景图, 敌人精灵, 角色精灵, 特效 |

## 章节导航

- [Sprite_Base - 精灵基类](sprite-base.md)
- [Sprite_Character - 角色精灵](sprite-character.md)
- [Spriteset_Map - 地图精灵集](spriteset-map.md)
- [Sprite_Battler - 战斗精灵](sprite-battler.md)
