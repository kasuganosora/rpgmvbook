# Scene_Base - 场景基类

## 职责

所有场景的基类，提供场景生命周期管理、淡入淡出效果、精灵和窗口容器。

## 核心属性

```javascript
Scene_Base.prototype.initialize = function() {
    PIXI.Container.call(this);
    this._active = false;
    this._fadeSign = 0;
    this._fadeDuration = 0;
    this._fadeSprite = null;
    this._imageReservationId = Utils.generateRuntimeId();
};
```

## 生命周期方法

### create - 创建子组件

```javascript
Scene_Base.prototype.create = function() {
    this.createDisplayObjects();
};

Scene_Base.prototype.createDisplayObjects = function() {
    this.createWindowLayer();  // 创建窗口层
    this.createSpriteset();     // 创建精灵集 (子类重写)
};
```

### start - 开始动画

```javascript
Scene_Base.prototype.start = function() {
    this._active = true;
};
```

### update - 每帧更新

```javascript
Scene_Base.prototype.update = function() {
    this.updateFade();      // 更新淡入淡出
    this.updateChildren();  // 更新子组件
};

Scene_Base.prototype.updateChildren = function() {
    this.children.forEach(function(child) {
        if (child.update) {
            child.update();
        }
    });
};
```

### stop - 停止动画

```javascript
Scene_Base.prototype.stop = function() {
    this._active = false;
};
```

### terminate - 销毁清理

```javascript
Scene_Base.prototype.terminate = function() {
    this.releaseReservation();
};
```

## 淡入淡出效果

### startFadeIn

```javascript
Scene_Base.prototype.startFadeIn = function(duration, white) {
    this._fadeSign = 1;  // 正数 = 淡入
    this._fadeDuration = duration || 30;
    this._fadeSprite = this.createFadeSprite(white);
    this.updateFade();
};
```

### startFadeOut

```javascript
Scene_Base.prototype.startFadeOut = function(duration, white) {
    this._fadeSign = -1;  // 负数 = 淡出
    this._fadeDuration = duration || 30;
    this._fadeSprite = this.createFadeSprite(white);
    this.updateFade();
};
```

### updateFade

```javascript
Scene_Base.prototype.updateFade = function() {
    if (this._fadeDuration > 0) {
        var d = this._fadeDuration;
        
        if (this._fadeSign > 0) {
            // 淡入: 从不透明到透明
            this._fadeSprite.opacity -= this._fadeSprite.opacity / d;
        } else {
            // 淡出: 从透明到不透明
            this._fadeSprite.opacity += (255 - this._fadeSprite.opacity) / d;
        }
        
        this._fadeDuration--;
        
        if (this._fadeDuration === 0) {
            this.onFadeComplete();
        }
    }
};
```

### onFadeComplete

```javascript
Scene_Base.prototype.onFadeComplete = function() {
    if (this._fadeSign < 0) {
        // 淡出完成后移除遮罩
        this.removeChild(this._fadeSprite);
        this._fadeSprite = null;
    }
};
```

## 状态检测

### isBusy

```javascript
Scene_Base.prototype.isBusy = function() {
    return this._fadeDuration > 0;
};

Scene_Base.prototype.isReady = function() {
    return this._imageReservationId && ImageManager.isReady();
};
```

## 窗口层管理

### createWindowLayer

```javascript
Scene_Base.prototype.createWindowLayer = function() {
    var width = Graphics.boxWidth;
    var height = Graphics.boxHeight;
    var x = (Graphics.width - width) / 2;
    var y = (Graphics.height - height) / 2;
    
    this._windowLayer = new WindowLayer();
    this._windowLayer.move(x, y, width, height);
    this.addChild(this._windowLayer);
};

Scene_Base.prototype.addWindow = function(window) {
    this._windowLayer.addChild(window);
};
```

## 资源预约

```javascript
Scene_Base.prototype.attachReservation = function() {
    ImageManager.setReservation(this._imageReservationId);
};

Scene_Base.prototype.detachReservation = function() {
    ImageManager.releaseReservation(this._imageReservationId);
};

Scene_Base.prototype.releaseReservation = function() {
    ImageManager.releaseReservation(this._imageReservationId);
};
```

## 快照功能

```javascript
Scene_Base.prototype.startFadeOutWithSnapshot = function(duration, white) {
    // 先截图再淡出，避免闪烁
    this._snapForFadeOut();
    this.startFadeOut(duration, white);
};

Scene_Base.prototype._snapForFadeOut = function() {
    this._fadeSprite = new Sprite();
    this._fadeSprite.bitmap = Bitmap.snap(this);
    this._fadeSprite.opacity = 0;
    this.addChild(this._fadeSprite);
};
```

## 子类扩展示例

```javascript
// Scene_Map 的 create 实现
Scene_Map.prototype.create = function() {
    Scene_Base.prototype.create.call(this);
    this._transfer = $gamePlayer.isTransferring();
    if (this._transfer) {
        DataManager.loadMapData($gamePlayer.newMapId());
    }
    this.createDisplayObjects();
};

Scene_Map.prototype.createDisplayObjects = function() {
    this.createSpriteset();        // 创建地图精灵
    this.createMapNameWindow();    // 创建地图名窗口
    this.createWindowLayer();      // 创建窗口层
    this.createAllWindows();       // 创建所有窗口
};
```
