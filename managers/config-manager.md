# ConfigManager - 配置管理器

## 一、职责

ConfigManager 管理用户设置：
- 音量控制
- 操作设置
- 自动存档
- 配置持久化

## 二、配置属性

```javascript
ConfigManager.alwaysDash = false;      // 总是冲刺
ConfigManager.commandRemember = false; // 记住指令
ConfigManager.augmentScale = 0;        // 画面缩放
ConfigManager.bgmVolume = 100;         // BGM音量
ConfigManager.bgsVolume = 100;         // BGS音量
ConfigManager.meVolume = 100;          // ME音量
ConfigManager.seVolume = 100;          // SE音量
```

## 三、配置加载与保存

### 3.1 加载配置

```javascript
ConfigManager.load = function() {
    var json = StorageManager.load(-1);  // -1是配置文件
    if (json) {
        var config = JSON.parse(json);
        this.applyData(config);
    }
};

ConfigManager.applyData = function(config) {
    this.alwaysDash = this.readFlag(config, 'alwaysDash', false);
    this.commandRemember = this.readFlag(config, 'commandRemember', false);
    this.augmentScale = this.readInt(config, 'augmentScale', 0);
    this.bgmVolume = this.readInt(config, 'bgmVolume', 100);
    this.bgsVolume = this.readInt(config, 'bgsVolume', 100);
    this.meVolume = this.readInt(config, 'meVolume', 100);
    this.seVolume = this.readInt(config, 'seVolume', 100);
};
```

### 3.2 保存配置

```javascript
ConfigManager.save = function() {
    StorageManager.save(-1, JSON.stringify(this.makeData()));
};

ConfigManager.makeData = function() {
    var config = {};
    config.alwaysDash = this.alwaysDash;
    config.commandRemember = this.commandRemember;
    config.augmentScale = this.augmentScale;
    config.bgmVolume = this.bgmVolume;
    config.bgsVolume = this.bgsVolume;
    config.meVolume = this.meVolume;
    config.seVolume = this.seVolume;
    return config;
};
```

## 四、音量控制

### 4.1 音量属性

```javascript
Object.defineProperty(ConfigManager, 'bgmVolume', {
    get: function() {
        return AudioManager.bgmVolume;
    },
    set: function(value) {
        AudioManager.bgmVolume = value;
        AudioManager.updateBgmParameters(AudioManager._currentBgm);
    },
    configurable: true
});

// 其他音量同理
```

### 4.2 音量应用

```javascript
// 当音量改变时，立即更新当前播放的音频
AudioManager.updateBgmParameters = function(bgm) {
    this.updateBufferParameters(this._bgmBuffer, this._bgmVolume, bgm);
};

AudioManager.updateBufferParameters = function(buffer, configVolume, audio) {
    if (buffer && audio) {
        var volume = configVolume * this._masterVolume * audio.volume / 10000;
        buffer.volume = volume;
    }
};
```

## 五、冲刺设置

```javascript
// 在 Game_Player 中应用
Game_Player.prototype.executeMove = function(direction) {
    // 根据设置决定冲刺行为
    if (this.isDashPressed() !== this.isDashing()) {
        if (ConfigManager.alwaysDash) {
            // 总是冲刺模式：默认冲刺，按住键走路
            if (!this.isDashPressed()) {
                this.startDash();
            } else {
                this.endDash();
            }
        } else {
            // 普通模式：按住键冲刺
            if (this.isDashPressed()) {
                this.startDash();
            } else {
                this.endDash();
            }
        }
    }
    this.moveStraight(direction);
};
```

## 六、指令记忆

```javascript
// 在 Window_MenuCommand 中应用
Window_MenuCommand.prototype.selectLast = function() {
    if (ConfigManager.commandRemember) {
        // 记住上次选择
        this.select(this._lastCommandSymbol);
    } else {
        // 默认选择第一个
        this.select(0);
    }
};

// 记录选择
Window_MenuCommand.prototype.processOk = function() {
    this._lastCommandSymbol = this.currentSymbol();
    Window_Command.prototype.processOk.call(this);
};
```

## 七、调试案例

### 案例1：读取当前配置

```javascript
console.log('冲刺:', ConfigManager.alwaysDash);
console.log('指令记忆:', ConfigManager.commandRemember);
console.log('BGM音量:', ConfigManager.bgmVolume);
console.log('SE音量:', ConfigManager.seVolume);
```

### 案例2：修改配置

```javascript
// 设置总是冲刺
ConfigManager.alwaysDash = true;

// 设置音量
ConfigManager.bgmVolume = 50;
ConfigManager.seVolume = 80;

// 保存配置
ConfigManager.save();
```

### 案例3：重置配置

```javascript
ConfigManager.alwaysDash = false;
ConfigManager.commandRemember = false;
ConfigManager.bgmVolume = 100;
ConfigManager.bgsVolume = 100;
ConfigManager.meVolume = 100;
ConfigManager.seVolume = 100;
ConfigManager.save();
```

### 案例4：查看配置文件

```javascript
// Web模式
var configKey = 'RPG Save-1';  // 注意：-1
var config = localStorage.getItem(configKey);
console.log(JSON.parse(config));
```

### 案例5：静音

```javascript
// 全部静音
ConfigManager.bgmVolume = 0;
ConfigManager.bgsVolume = 0;
ConfigManager.meVolume = 0;
ConfigManager.seVolume = 0;

// 不保存，仅临时静音
```
