# 内置插件分析

## Community_Basic

基础配置插件，提供常用系统设置。

```javascript
/*:
 * @plugindesc 基础配置插件
 * @author Community
 *
 * @param cacheLimit
 * @desc 图像缓存大小 (MB)
 * @default 10
 *
 * @param screenWidth
 * @desc 屏幕宽度
 * @default 816
 *
 * @param screenHeight
 * @desc 屏幕高度
 * @default 624
 */

(function() {
    var parameters = PluginManager.parameters('Community_Basic');
    var cacheLimit = Number(parameters['cacheLimit']) || 10;
    
    // 设置缓存限制
    ImageCache.limit = cacheLimit * 1000000; // MB to bytes
})();
```

## CustomLogo

自定义启动Logo显示。

```javascript
// 主要功能
- 显示自定义启动图像
- 支持多个Logo顺序显示
- 可配置延迟时间

/*:
 * @param Logo1
 * @desc Logo图片1 (放在 img/pictures/ 目录)
 * @default
 *
 * @param Logo2
 * @desc Logo图片2
 * @default
 *
 * @param Delay
 * @desc 每个Logo显示时间 (帧)
 * @default 60
 */
```

## AltMenuScreen

菜单界面增强。

```javascript
// 功能
- 自定义菜单布局
- 增强的角色状态显示
- 可配置的命令图标
```

## EnemyBook

敌人图鉴系统。

```javascript
// 功能
- 记录遇到的敌人
- 显示敌人详细信息
- 掉落物统计
- 完成度追踪
```

## ItemBook

物品图鉴系统。

```javascript
// 功能
- 记录获得的物品
- 物品分类显示
- 稀有度标记
```

## 项目插件列表

本项目 `js/plugins/` 目录包含：

| 插件 | 功能 |
|------|------|
| Community_Basic.js | 基础配置 |
| CustomLogo.js | 启动Logo |
| AltMenuScreen.js | 菜单增强 |
| EnemyBook.js | 敌人图鉴 |
| ItemBook.js | 物品图鉴 |
| AltSaveScreen.js | 存档界面增强 |
| AltBattleScreen.js | 战斗界面增强 |
| SimpleMsgSideView.js | 简化侧视战斗消息 |
| WindowConsole.js | 调试控制台 |

## 插件配置示例

### plugins.js 结构

```javascript
var $plugins = [
    {
        "name": "Community_Basic",
        "status": true,
        "description": "Basic configuration plugin.",
        "parameters": {
            "cacheLimit": "10",
            "screenWidth": "816",
            "screenHeight": "624",
            "changeWindowWidthTo": "",
            "changeWindowHeightTo": ""
        }
    },
    {
        "name": "CustomLogo",
        "status": true,
        "description": "Custom logo display on startup.",
        "parameters": {
            "Logo1": "",
            "Logo2": "",
            "Delay": "60",
            "FadeSpeed": "8"
        }
    }
];
```

## 禁用插件

设置 `status: false` 可禁用插件：

```javascript
{
    "name": "EnemyBook",
    "status": false,  // 禁用
    "parameters": {...}
}
```
