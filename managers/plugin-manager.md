# PluginManager - 插件管理

## 职责

- 加载插件脚本
- 解析插件参数
- 管理插件依赖

## 插件列表

```javascript
// plugins.js
var $plugins = [
    {
        name: 'Community_Basic',
        status: true,
        description: 'Basic configuration plugin',
        parameters: {
            "cacheLimit": "10",
            "screenWidth": "816",
            "screenHeight": "624"
        }
    },
    {
        name: 'CustomLogo',
        status: true,
        description: 'Custom logo display',
        parameters: {
            "Logo1": "",
            "Logo2": "",
            "Delay": "60"
        }
    }
];
```

## 初始化

```javascript
PluginManager.setup = function(plugins) {
    this._plugins = plugins;
    this._parameters = {};
    
    plugins.forEach(function(plugin) {
        if (plugin.status) {
            this.setParameters(plugin.name, plugin.parameters);
        }
    }, this);
};
```

## 参数获取

```javascript
PluginManager.parameters = function(name) {
    return this._parameters[name.toLowerCase()] || {};
};

// 使用示例
var params = PluginManager.parameters('Community_Basic');
var cacheLimit = Number(params['cacheLimit'] || 10);
```

## 加载脚本

```javascript
PluginManager.loadScript = function(name) {
    var url = 'js/plugins/' + name;
    var script = document.createElement('script');
    script.type = 'text/javascript';
    script.src = url;
    script.async = false;
    script.onerror = this.onError.bind(this);
    document.body.appendChild(script);
};

PluginManager.onError = function(e) {
    throw new Error('Failed to load plugin: ' + e.target.src);
};
```

## 插件开发模式

### Alias 模式

```javascript
// 保存原方法引用
var _Game_Player_update = Game_Player.prototype.update;

// 重写方法
Game_Player.prototype.update = function(sceneActive) {
    // 调用原方法
    _Game_Player_update.call(this, sceneActive);
    
    // 添加新功能
    this.updateCustomFeature();
};
```

### 命名空间模式

```javascript
// 创建插件命名空间
var MyPlugin = MyPlugin || {};

MyPlugin.someFunction = function() {
    // ...
};

MyPlugin.params = PluginManager.parameters('MyPlugin');
```

## 命令注册

```javascript
// 注册插件命令
var _Game_Interpreter_pluginCommand = Game_Interpreter.prototype.pluginCommand;

Game_Interpreter.prototype.pluginCommand = function(command, args) {
    _Game_Interpreter_pluginCommand.call(this, command, args);
    
    if (command === 'MyCommand') {
        MyPlugin.processCommand(args);
    }
};

// 事件中使用: 插件命令: MyCommand arg1 arg2
```

## 本项目插件

| 插件名 | 功能 |
|--------|------|
| Community_Basic | 基础配置 (缓存、分辨率) |
| CustomLogo | 自定义启动Logo |
| AltMenuScreen | 菜单界面增强 |
| EnemyBook | 敌人图鉴 |
| ItemBook | 物品图鉴 |
