# StorageManager - 存储管理器

## 一、职责

StorageManager 处理数据持久化：
- 存档/读档
- 全局信息管理
- 本地存储与云存储适配

## 二、存储模式

```javascript
// 检测存储模式
StorageManager.isLocalMode = function() {
    return Utils.isNwjs() && !Utils.isMobileDevice();
    // 本地NW.js环境使用文件系统
};

StorageManager.isWebMode = function() {
    return !this.isLocalMode();
    // Web环境使用localStorage
};
```

## 三、本地文件存储

### 3.1 文件路径

```javascript
// Windows: C:\Users\用户名\AppData\Local\游戏名\
// Mac: ~/Library/Application Support/游戏名/

StorageManager.localFileDirectoryPath = function() {
    var path = require('path');
    var base = require('nw.gui').App.dataPath;
    return path.join(base, this._localFilePrefix);
};

StorageManager.localFilePath = function(savefileId) {
    var name = this._localFilePrefix + 'Save' + savefileId + '.rpgsave';
    return this.localFileDirectoryPath() + name;
};
```

### 3.2 文件读写

```javascript
StorageManager.saveToLocalFile = function(savefileId, json) {
    var fs = require('fs');
    var path = this.localFilePath(savefileId);
    
    // 创建目录
    var dir = this.localFileDirectoryPath();
    if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir);
    }
    
    // 写入文件
    fs.writeFileSync(path, json);
};

StorageManager.loadFromLocalFile = function(savefileId) {
    var fs = require('fs');
    var path = this.localFilePath(savefileId);
    
    if (fs.existsSync(path)) {
        return fs.readFileSync(path, { encoding: 'utf8' });
    }
    return null;
};
```

## 四、Web存储（localStorage）

### 4.1 存储键名

```javascript
StorageManager.webStorageKey = function(savefileId) {
    return 'RPG Save' + savefileId;
};

// 存档0是全局信息
// 'RPG Save0' → 全局信息
// 'RPG Save1' → 存档1
// 'RPG Save2' → 存档2
```

### 4.2 localStorage读写

```javascript
StorageManager.saveToWebStorage = function(savefileId, json) {
    var key = this.webStorageKey(savefileId);
    localStorage.setItem(key, json);
};

StorageManager.loadFromWebStorage = function(savefileId) {
    var key = this.webStorageKey(savefileId);
    return localStorage.getItem(key);
};

StorageManager.removeWebStorage = function(savefileId) {
    var key = this.webStorageKey(savefileId);
    localStorage.removeItem(key);
};
```

## 五、统一接口

```javascript
// 保存
StorageManager.save = function(savefileId, json) {
    if (this.isLocalMode()) {
        this.saveToLocalFile(savefileId, json);
    } else {
        this.saveToWebStorage(savefileId, json);
    }
};

// 加载
StorageManager.load = function(savefileId) {
    if (this.isLocalMode()) {
        return this.loadFromLocalFile(savefileId);
    } else {
        return this.loadFromWebStorage(savefileId);
    }
};

// 删除
StorageManager.remove = function(savefileId) {
    if (this.isLocalMode()) {
        this.removeFromLocalFile(savefileId);
    } else {
        this.removeWebStorage(savefileId);
    }
};

// 检测存在
StorageManager.exists = function(savefileId) {
    if (this.isLocalMode()) {
        return this.localFileExists(savefileId);
    } else {
        return this.webStorageExists(savefileId);
    }
};
```

## 六、备份系统

```javascript
// 备份当前存档
StorageManager.backup = function(savefileId) {
    if (this.exists(savefileId)) {
        var json = this.load(savefileId);
        var backupId = savefileId + '_backup';
        this.save(backupId, json);
    }
};

// 恢复备份
StorageManager.restoreBackup = function(savefileId) {
    var backupId = savefileId + '_backup';
    if (this.exists(backupId)) {
        var json = this.load(backupId);
        this.save(savefileId, json);
        this.remove(backupId);
    }
};

// 删除备份
StorageManager.removeBackup = function(savefileId) {
    var backupId = savefileId + '_backup';
    if (this.exists(backupId)) {
        this.remove(backupId);
    }
};
```

## 七、LZString压缩

```javascript
// RPG Maker MV 使用 LZString 压缩存档数据
// 压缩后体积可减少约60%

StorageManager.compress = function(json) {
    return LZString.compressToBase64(json);
};

StorageManager.decompress = function(str) {
    return LZString.decompressFromBase64(str);
};

// 实际使用（存档时）
var json = JSON.stringify(DataManager.makeSaveContents());
var compressed = StorageManager.compress(json);
StorageManager.save(savefileId, compressed);

// 实际使用（读档时）
var compressed = StorageManager.load(savefileId);
var json = StorageManager.decompress(compressed);
var contents = JSON.parse(json);
```

## 八、调试案例

### 案例1：查看所有存档

```javascript
// Web模式
for (var i = 0; i <= 20; i++) {
    var key = 'RPG Save' + i;
    var data = localStorage.getItem(key);
    if (data) {
        console.log('存档' + i + ':', (data.length / 1024).toFixed(2) + 'KB');
    }
}
```

### 案例2：导出存档

```javascript
// 导出存档1为文本
var data = StorageManager.load(1);
console.log(data);
// 复制到文本文件保存
```

### 案例3：导入存档

```javascript
// 从文本导入
var savedData = "你的存档数据字符串...";
StorageManager.save(1, savedData);
```

### 案例4：清除所有存档

```javascript
// ⚠️ 危险操作！
for (var i = 0; i <= 20; i++) {
    StorageManager.remove(i);
}
console.log('所有存档已清除');
```

### 案例5：查看存档详情

```javascript
var compressed = StorageManager.load(1);
var json = StorageManager.decompress(compressed);
var data = JSON.parse(json);

console.log('队伍金币:', data.party._gold);
console.log('玩家位置:', data.player._x, data.player._y);
console.log('地图ID:', data.map._mapId);
console.log('开关1:', data.switches._data[1]);
console.log('变量1:', data.variables._data[1]);
```

### 案例6：修改存档数据

```javascript
// 读取
var compressed = StorageManager.load(1);
var json = StorageManager.decompress(compressed);
var data = JSON.parse(json);

// 修改
data.party._gold = 99999;
data.variables._data[1] = 100;

// 保存
var newJson = JSON.stringify(data);
var newCompressed = StorageManager.compress(newJson);
StorageManager.save(1, newCompressed);

console.log('存档已修改');
```
