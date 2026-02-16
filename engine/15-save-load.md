# 第十五章：存档系统

## 为什么需要存档？

- 玩家可以随时退出游戏
- 下次打开时继续之前的进度
- 记录角色等级、物品、位置等

---

## 存档数据结构

```javascript
function SaveData() {
    this.version = 1;           // 存档版本（用于兼容性）
    this.timestamp = 0;         // 保存时间戳

    // 玩家数据
    this.player = {
        name: '',
        level: 1,
        exp: 0,
        hp: 100,
        mp: 30,
        x: 0,
        y: 0,
        mapId: 1
    };

    // 背包数据
    this.inventory = [];

    // 游戏变量
    this.variables = {};
    this.switches = {};
    this.playtime = 0;  // 游戏时长（秒）
}
```

---

## 保存到本地存储

```javascript
var SaveManager = {
    SAVE_KEY: 'myrpg_save_',

    // 保存游戏
    save: function(slotId) {
        var saveData = new SaveData();

        // 收集数据
        saveData.timestamp = Date.now();
        saveData.player = {
            name: $gamePlayer.name,
            level: $gamePlayer.level,
            exp: $gamePlayer.exp,
            hp: $gamePlayer.hp,
            mp: $gamePlayer.mp,
            x: $gamePlayer.x,
            y: $gamePlayer.y,
            mapId: $gameMap.mapId
        };
        saveData.inventory = $gameParty.inventory.items;
        saveData.variables = $gameVariables._data;
        saveData.switches = $gameSwitches._data;
        saveData.playtime = $gameSystem.playtime;

        // 转换为 JSON 字符串
        var json = JSON.stringify(saveData);

        // 保存到 localStorage
        var key = this.SAVE_KEY + slotId;
        localStorage.setItem(key, json);

        console.log('游戏已保存到存档 ' + slotId);
        return true;
    },

    // 加载游戏
    load: function(slotId) {
        var key = this.SAVE_KEY + slotId;
        var json = localStorage.getItem(key);

        if (!json) {
            console.log('存档 ' + slotId + ' 不存在');
            return false;
        }

        try {
            var saveData = JSON.parse(json);

            // 检查版本兼容性
            if (saveData.version !== 1) {
                console.log('存档版本不兼容');
                return false;
            }

            // 恢复数据
            $gamePlayer.name = saveData.player.name;
            $gamePlayer.level = saveData.player.level;
            $gamePlayer.exp = saveData.player.exp;
            $gamePlayer.hp = saveData.player.hp;
            $gamePlayer.mp = saveData.player.mp;
            $gamePlayer.x = saveData.player.x;
            $gamePlayer.y = saveData.player.y;

            $gameParty.inventory.items = saveData.inventory;
            $gameVariables._data = saveData.variables;
            $gameSwitches._data = saveData.switches;
            $gameSystem.playtime = saveData.playtime;

            // 切换到对应地图
            $gameMap.loadMap(saveData.player.mapId);

            console.log('存档 ' + slotId + ' 已加载');
            return true;
        } catch (e) {
            console.log('加载存档失败：' + e.message);
            return false;
        }
    },

    // 检查存档是否存在
    exists: function(slotId) {
        var key = this.SAVE_KEY + slotId;
        return localStorage.getItem(key) !== null;
    },

    // 删除存档
    delete: function(slotId) {
        var key = this.SAVE_KEY + slotId;
        localStorage.removeItem(key);
        console.log('存档 ' + slotId + ' 已删除');
    },

    // 获取存档信息
    getSaveInfo: function(slotId) {
        var key = this.SAVE_KEY + slotId;
        var json = localStorage.getItem(key);

        if (!json) return null;

        try {
            var data = JSON.parse(json);
            return {
                level: data.player.level,
                mapName: getMapName(data.player.mapId),
                playtime: formatPlaytime(data.playtime),
                timestamp: new Date(data.timestamp).toLocaleString()
            };
        } catch (e) {
            return null;
        }
    }
};

// 格式化游戏时长
function formatPlaytime(seconds) {
    var hours = Math.floor(seconds / 3600);
    var minutes = Math.floor((seconds % 3600) / 60);
    var secs = seconds % 60;

    return hours.toString().padStart(2, '0') + ':' +
           minutes.toString().padStart(2, '0') + ':' +
           secs.toString().padStart(2, '0');
}
```

---

## 存档菜单

```javascript
function SaveMenu(mode) {
    Window.call(this, 200, 100, 400, 350);
    this.mode = mode;  // 'save' 或 'load'
    this.selectedIndex = 0;
    this.maxSlots = 3;
}
SaveMenu.prototype = Object.create(Window.prototype);

SaveMenu.prototype.update = function() {
    if (Input.isPressed('up')) {
        this.selectedIndex = Math.max(0, this.selectedIndex - 1);
    }
    if (Input.isPressed('down')) {
        this.selectedIndex = Math.min(this.maxSlots - 1, this.selectedIndex + 1);
    }

    if (Input.isPressed('confirm')) {
        if (this.mode === 'save') {
            SaveManager.save(this.selectedIndex + 1);
        } else {
            if (SaveManager.exists(this.selectedIndex + 1)) {
                SaveManager.load(this.selectedIndex + 1);
                SceneManager.changeScene(new Scene_Map());
            }
        }
    }

    if (Input.isPressed('cancel')) {
        SceneManager.changeScene(new Scene_Menu());
    }
};

SaveMenu.prototype.render = function(ctx) {
    Window.prototype.render.call(this, ctx);

    ctx.fillStyle = '#fff';
    ctx.font = 'bold 18px Arial';
    ctx.textAlign = 'center';
    ctx.fillText(this.mode === 'save' ? '保存游戏' : '读取游戏',
                 this.x + this.width / 2, this.y + 30);

    for (var i = 0; i < this.maxSlots; i++) {
        var y = this.y + 60 + i * 80;
        var info = SaveManager.getSaveInfo(i + 1);

        // 高亮
        if (i === this.selectedIndex) {
            ctx.fillStyle = 'rgba(255,255,0,0.3)';
            ctx.fillRect(this.x + 10, y - 15, this.width - 20, 70);
        }

        ctx.fillStyle = '#fff';
        ctx.font = '14px Arial';
        ctx.textAlign = 'left';

        if (info) {
            ctx.fillText('存档 ' + (i + 1), this.x + 20, y);
            ctx.fillText('等级: ' + info.level, this.x + 20, y + 20);
            ctx.fillText('时间: ' + info.playtime, this.x + 150, y + 20);
            ctx.fillText(info.timestamp, this.x + 20, y + 40);
        } else {
            ctx.fillStyle = '#666';
            ctx.fillText('存档 ' + (i + 1) + ' - 空', this.x + 20, y + 20);
        }
    }
};
```

---

## 自动保存

```javascript
var AutoSave = {
    interval: 300000,  // 5 分钟（毫秒）
    enabled: true,

    start: function() {
        if (!this.enabled) return;

        setInterval(function() {
            SaveManager.save(0);  // 使用槽位 0 作为自动存档
            console.log('自动保存完成');
        }, this.interval);
    }
};
```

---

## 本章小结

1. **localStorage** 是浏览器提供的本地存储
2. **JSON** 用于序列化和反序列化
3. **版本号** 用于处理存档兼容性

---

# 系列结语

恭喜你完成了 RPG 游戏引擎的学习！

**你学到了什么**：
- 游戏循环和帧动画
- 瓦片地图和坐标系统
- 精灵、碰撞检测、场景管理
- 输入处理和事件系统
- UI、数据、战斗、存档

**下一步**：
1. 完善这个引擎，添加更多功能
2. 用这个引擎做一个小游戏
3. 学习 Unity、Godot 等商业引擎
4. 阅读 RPG Maker MV 的源代码

**推荐资源**：
- MDN Canvas 教程
- 《游戏编程模式》
- RPG Maker MV 官方文档

祝你在游戏开发的道路上越走越远！
