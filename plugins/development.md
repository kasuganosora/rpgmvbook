# 插件开发指南

## 一、插件基础结构

```javascript
//=============================================================================
// MyPlugin.js
//=============================================================================

/*:
 * @plugindesc 插件简短描述
 * @author 作者名
 *
 * @param 参数名
 * @desc 参数描述
 * @type string
 * @default 默认值
 *
 * @help
 * 详细帮助文本
 * 
 * 使用方法说明...
 */

(function() {
    'use strict';
    
    // 获取插件参数
    var parameters = PluginManager.parameters('MyPlugin');
    var myParam = String(parameters['参数名'] || '默认值');
    
    // 覆盖或扩展方法
    var _Game_Player_update = Game_Player.prototype.update;
    Game_Player.prototype.update = function() {
        _Game_Player_update.call(this);
        // 自定义代码
    };
    
})();
```

## 二、插件参数定义

### 2.1 基本类型

```javascript
/*:
 * @param textParam
 * @desc 文本参数
 * @type string
 * @default Hello
 *
 * @param numberParam
 * @desc 数字参数
 * @type number
 * @min 0
 * @max 100
 * @default 50
 *
 * @param booleanParam
 * @desc 布尔参数
 * @type boolean
 * @default true
 *
 * @param selectParam
 * @desc 下拉选择
 * @type select
 * @option 选项A
 * @option 选项B
 * @option 选项C
 * @default 选项A
 *
 * @param colorParam
 * @desc 颜色参数
 * @type struct<Color>
 * @default {"r":"255","g":"0","b":"0"}
 */
```

### 2.2 结构体参数

```javascript
/*:
 * @param enemySettings
 * @desc 敌人设置
 * @type struct<Enemy>[]
 * @default []
 */

/*~struct~Enemy:
 * @param name
 * @desc 敌人名称
 * @type string
 *
 * @param hp
 * @desc HP
 * @type number
 * @default 100
 */

// 使用
var enemySettings = JSON.parse(parameters['enemySettings'] || '[]');
enemySettings.forEach(function(enemy) {
    var e = JSON.parse(enemy);
    console.log(e.name, e.hp);
});
```

## 三、方法覆盖技巧

### 3.1 完全覆盖

```javascript
// 完全替换原方法
Game_Player.prototype.canMove = function() {
    return false;  // 禁止玩家移动
};
```

### 3.2 扩展（保留原逻辑）

```javascript
// 保存原方法引用
var _Game_Player_update = Game_Player.prototype.update;

Game_Player.prototype.update = function() {
    // 先调用原方法
    _Game_Player_update.call(this);
    
    // 添加新逻辑
    if (this.isMoving()) {
        // 自定义代码
    }
};
```

### 3.3 前置扩展

```javascript
var _Scene_Map_start = Scene_Map.prototype.start;

Scene_Map.prototype.start = function() {
    // 先执行自定义逻辑
    console.log('地图开始');
    
    // 再调用原方法
    _Scene_Map_start.call(this);
};
```

## 四、自定义命令

### 4.1 添加插件命令

```javascript
(function() {
    var parameters = PluginManager.parameters('MyPlugin');
    
    // 保存原命令处理器
    var _Game_Interpreter_pluginCommand = Game_Interpreter.prototype.pluginCommand;
    
    Game_Interpreter.prototype.pluginCommand = function(command, args) {
        _Game_Interpreter_pluginCommand.call(this, command, args);
        
        // 处理自定义命令
        if (command === 'myCommand') {
            switch (args[0]) {
                case 'hello':
                    console.log('Hello, ' + args[1]);
                    break;
                case 'addItem':
                    var itemId = Number(args[1]);
                    var count = Number(args[2] || 1);
                    $gameParty.gainItem($dataItems[itemId], count);
                    break;
            }
        }
    };
})();

// 事件中使用：
// 插件命令：myCommand hello World
// 插件命令：myCommand addItem 1 5
```

### 4.2 带参数的命令

```javascript
Game_Interpreter.prototype.pluginCommand = function(command, args) {
    if (command === 'showMessage') {
        var text = args.join(' ');
        $gameMessage.add(text);
    }
    
    if (command === 'setVariable') {
        var varId = Number(args[0]);
        var value = Number(args[1]);
        $gameVariables.setValue(varId, value);
    }
};

// 事件中使用：
// 插件命令：showMessage This is a test message
// 插件命令：setVariable 1 100
```

## 五、元数据（Meta）

### 5.1 从备注读取

```javascript
// 在编辑器备注栏写入：<myParam:100>

// 读取角色元数据
var actor = $dataActors[1];
if (actor._meta && actor._meta.myParam) {
    console.log('参数值:', actor._meta.myParam);
}

// 读取物品元数据
var item = $dataItems[1];
if (item._meta && item._meta.special) {
    // 物品有 <special> 标签
}
```

### 5.2 复杂元数据

```javascript
// 备注栏：<customData:{"atk":50,"def":30}>

var meta = $dataWeapons[1]._meta;
if (meta.customData) {
    var data = JSON.parse(meta.customData);
    console.log('攻击:', data.atk, '防御:', data.def);
}

// 备注栏：<skills:[1,2,3]>

if (meta.skills) {
    var skills = JSON.parse(meta.skills);
    skills.forEach(function(skillId) {
        actor.learnSkill(skillId);
    });
}
```

## 六、调试技巧

### 6.1 开发模式检测

```javascript
var isDebug = Utils.isOptionValid('test');

if (isDebug) {
    console.log('开发模式');
}
```

### 6.2 控制台输出

```javascript
// 使用前缀便于过滤
var PLUGIN_NAME = 'MyPlugin';

function log(message) {
    console.log('[' + PLUGIN_NAME + ']', message);
}

function warn(message) {
    console.warn('[' + PLUGIN_NAME + ']', message);
}

function error(message) {
    console.error('[' + PLUGIN_NAME + ']', message);
}
```

### 6.3 参数验证

```javascript
var parameters = PluginManager.parameters('MyPlugin');

function getNumber(name, defaultValue, min, max) {
    var value = Number(parameters[name] || defaultValue);
    if (min !== undefined && value < min) value = min;
    if (max !== undefined && value > max) value = max;
    return value;
}

var speed = getNumber('speed', 5, 1, 10);
```

## 七、完整示例：自动存档插件

```javascript
//=============================================================================
// AutoSave.js
//=============================================================================

/*:
 * @plugindesc 定时自动存档
 * @author YourName
 *
 * @param interval
 * @desc 自动存档间隔（秒）
 * @type number
 * @min 30
 * @max 600
 * @default 120
 *
 * @param showNotification
 * @desc 显示存档提示
 * @type boolean
 * @default true
 *
 * @help
 * 定时自动存档插件
 */

(function() {
    'use strict';
    
    var parameters = PluginManager.parameters('AutoSave');
    var interval = Number(parameters['interval'] || 120) * 60;  // 转帧数
    var showNotification = parameters['showNotification'] === 'true';
    
    var frameCount = 0;
    
    var _Scene_Map_update = Scene_Map.prototype.update;
    Scene_Map.prototype.update = function() {
        _Scene_Map_update.call(this);
        
        frameCount++;
        
        if (frameCount >= interval) {
            frameCount = 0;
            autoSave();
        }
    };
    
    function autoSave() {
        if ($gameSystem.isSaveEnabled()) {
            $gameSystem.onBeforeSave();
            if (DataManager.saveGame(1)) {
                if (showNotification) {
                    $gameMessage.add('\\c[2]已自动存档\\c[0]');
                }
                console.log('Auto-saved');
            }
        }
    }
    
})();
```

## 八、发布检查清单

1. ✅ 参数都有默认值
2. ✅ 使用严格模式 `'use strict'`
3. ✅ 变量作用域正确（IIFE包裹）
4. ✅ 兼容其他插件（正确保存/调用原方法）
5. ✅ 帮助文本完整
6. ✅ 错误处理（try-catch）
7. ✅ 注释清晰
