# 插件系统

## 概述

插件系统允许通过 JavaScript 扩展游戏功能，无需修改核心代码。

## 插件加载

### plugins.js

```javascript
var $plugins = [
    {
        name: 'Community_Basic',
        status: true,
        description: '基础配置插件',
        parameters: {
            "cacheLimit": "10",
            "screenWidth": "816",
            "screenHeight": "624"
        }
    }
];
```

### 加载流程

```javascript
// main.js
PluginManager.setup($plugins);

// 按顺序加载启用的插件
plugins.forEach(function(plugin) {
    if (plugin.status) {
        PluginManager.loadScript(plugin.name + '.js');
    }
});
```

## 插件开发

### Alias 模式

通过保存原方法引用来扩展功能：

```javascript
// 保存原方法
var _Game_Player_update = Game_Player.prototype.update;

// 重写方法
Game_Player.prototype.update = function(sceneActive) {
    // 调用原方法
    _Game_Player_update.call(this, sceneActive);
    
    // 添加新功能
    this.updateCustomFeature();
};
```

### 参数获取

```javascript
// 获取插件参数
var parameters = PluginManager.parameters('MyPlugin');
var value = Number(parameters['myParam'] || 0);
```

### 插件命令

注册自定义事件命令：

```javascript
var _Game_Interpreter_pluginCommand = 
    Game_Interpreter.prototype.pluginCommand;

Game_Interpreter.prototype.pluginCommand = function(command, args) {
    _Game_Interpreter_pluginCommand.call(this, command, args);
    
    if (command === 'MyCommand') {
        switch (args[0]) {
            case 'option1':
                // 处理选项1
                break;
            case 'option2':
                // 处理选项2
                break;
        }
    }
};
```

在事件中使用：
```
插件命令: MyCommand option1 参数1 参数2
```

## 章节导航

- [插件开发指南](development.md)
- [内置插件分析](builtin-plugins.md)
