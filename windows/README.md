# 窗口系统

## 概述

窗口系统负责游戏UI的显示和交互，包括菜单、对话框、状态栏等。

## 窗口继承关系

```
PIXI.Container
└── Window
        ├── Window_Base (基础窗口)
        │       ├── Window_Selectable (可选择窗口)
        │       │       ├── Window_Command (命令窗口)
        │       │       ├── Window_MenuCommand
        │       │       ├── Window_ItemList
        │       │       ├── Window_SkillList
        │       │       ├── Window_EquipSlot
        │       │       └── Window_SavefileList
        │       ├── Window_Help (帮助窗口)
        │       ├── Window_Gold (金钱窗口)
        │       ├── Window_NameEdit (名称编辑)
        │       └── Window_MenuStatus
        ├── Window_Message (消息窗口)
        ├── Window_BattleLog (战斗日志)
        ├── Window_BattleStatus
        ├── Window_PartyCommand
        └── Window_ActorCommand
```

## 核心组件

### Window_Base

所有窗口的基类，提供：
- 背景绘制
- 文本绘制
- 图标绘制
- 状态绘制

### Window_Selectable

可选择窗口，提供：
- 光标移动
- 项目选择
- 滚动功能

### Window_Command

命令窗口，提供：
- 命令列表管理
- 启用/禁用状态

## 章节导航

- [Window_Base - 窗口基类](window-base.md)
- [Window_Selectable - 可选择窗口](window-selectable.md)
- [常用窗口类](common-windows.md)
