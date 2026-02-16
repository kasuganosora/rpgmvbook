# UI 系统详解

## 概述

RPG Maker MV 的 UI 基于 **Window** 类构建，所有窗口继承自 PIXI.Container。

## 继承体系

```
PIXI.Container
└── Window (背景/边框/光标)
    └── Window_Base (绘制工具)
        ├── Window_Selectable (选择交互)
        │   ├── Window_Command
        │   ├── Window_ItemList
        │   └── Window_SkillList
        └── Window_Help / Window_Gold / Window_Message
```

**设计原因**：每层只关注一个职责，方便复用扩展。

## 详细章节

- [窗口渲染原理](./ui-window.md) - Window 类、皮肤、九宫格、光标动画
- [Bitmap 绘图](./ui-bitmap.md) - 文字、图标、计量条绘制
- [输入系统](./ui-input.md) - Input/TouchInput 按键检测
- [窗口交互](./ui-interaction.md) - 选择、确认、触摸处理
- [开发案例](./ui-examples.md) - 自定义窗口完整示例

## 核心组件

| 组件 | 用途 |
|------|------|
| Window_Base | 基础绘制（文字/图标/计量条） |
| Window_Selectable | 可选择列表（菜单/物品/技能） |
| Window_Command | 命令按钮列表 |
| Window_Message | 对话框 |
| Window_Help | 说明文字 |
