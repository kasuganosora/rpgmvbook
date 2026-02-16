# Scene_Boot - 启动场景

## 职责

游戏的第一个场景，负责：
- 加载所有数据库文件
- 加载系统图像
- 检测是否需要跳过标题

## 流程

```
SceneManager.run(Scene_Boot)
        │
        ▼
Scene_Boot.create()
    ├── loadSystemImages()
    └── 检测数据库是否加载完成
        │
        ▼
Scene_Boot.isReady()
    ├── 数据库加载完成?
    └── 系统图像加载完成?
        │
        ▼
Scene_Boot.start()
    ├── 检测跳过标题
    ├── 跳过? → goto(Scene_Map)
    └── 否则 → goto(Scene_Title)
```

## 核心方法

### create

```javascript
Scene_Boot.prototype.create = function() {
    Scene_Base.prototype.create.call(this);
    DataManager.loadDatabase();
    ConfigManager.load();
    this.loadSystemImages();
};
```

### loadSystemImages

```javascript
Scene_Boot.prototype.loadSystemImages = function() {
    ImageManager.loadSystem('Window');
    ImageManager.loadSystem('IconSet');
    ImageManager.loadSystem('Balloon');
    ImageManager.loadSystem('Shadow1');
    ImageManager.loadSystem('Shadow2');
    ImageManager.loadSystem('Damage');
    ImageManager.loadSystem('States');
    ImageManager.loadSystem('Weapons1');
    ImageManager.loadSystem('Weapons2');
    ImageManager.loadSystem('Weapons3');
    ImageManager.loadSystem('ButtonSet');
};
```

### isReady

```javascript
Scene_Boot.prototype.isReady = function() {
    if (Scene_Base.prototype.isReady.call(this)) {
        return DataManager.isDatabaseLoaded() && this.isGameFontLoaded();
    }
    return false;
};

Scene_Boot.prototype.isGameFontLoaded = function() {
    if (Graphics.isFontLoaded('GameFont')) {
        return true;
    }
    // 等待字体加载
    var loaded = false;
    Graphics._fontLoader('GameFont', function() {
        loaded = true;
    });
    return loaded;
};
```

### start

```javascript
Scene_Boot.prototype.start = function() {
    Scene_Base.prototype.start.call(this);
    SoundManager.preloadImportantSounds();
    
    if (DataManager.isBattleTest()) {
        DataManager.setupBattleTest();
        SceneManager.goto(Scene_Battle);
    } else if (DataManager.isEventTest()) {
        DataManager.setupEventTest();
        SceneManager.goto(Scene_Map);
    } else {
        this.checkPlayerLocation();
        SceneManager.goto(Scene_Title);
    }
};
```

### checkPlayerLocation

```javascript
Scene_Boot.prototype.checkPlayerLocation = function() {
    // 检测玩家初始位置是否有效
    if ($dataSystem.startMapId === 0) {
        throw new Error("Player's starting position is not set");
    }
};
```

## 检测数据库加载

```javascript
DataManager.isDatabaseLoaded = function() {
    for (var i = 0; i < this._databaseFiles.length; i++) {
        if (!window[this._databaseFiles[i].name]) {
            return false;
        }
    }
    return true;
};
```

## 跳过标题

```javascript
// 通过 $dataSystem 或插件实现
Scene_Boot.prototype.start = function() {
    // ...
    if ($dataSystem.optSkipTitle) {
        DataManager.setupNewGame();
        SceneManager.goto(Scene_Map);
    } else {
        SceneManager.goto(Scene_Title);
    }
};
```
