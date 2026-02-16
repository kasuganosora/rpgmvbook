# ImageManager - 图像管理器

## 一、核心加载方法

```javascript
ImageManager.loadBitmap = function(folder, filename, hue, smooth) {
    if (filename) {
        var path = folder + encodeURIComponent(filename) + '.png';
        var bitmap = this.loadNormalBitmap(path, hue || 0);
        bitmap.smooth = smooth;
        return bitmap;
    } else {
        return this.loadEmptyBitmap();
    }
};
```

## 二、缓存机制详解

```javascript
ImageManager._cache = {};  // 缓存对象

ImageManager.loadNormalBitmap = function(path, hue) {
    // 生成缓存键：路径 + 色调
    var key = this._generateCacheKey(path, hue);
    var bitmap = this._cache[key];
    
    if (!bitmap) {
        // 首次加载：创建 Bitmap 并缓存
        bitmap = Bitmap.load(path);
        
        // 色调变化需要在加载完成后处理
        if (hue !== 0) {
            bitmap.addLoadListener(function() {
                bitmap.rotateHue(hue);
            });
        }
        
        this._cache[key] = bitmap;
    }
    
    return bitmap;  // 后续直接返回缓存的 Bitmap
};
```

**实际案例**：
```javascript
// 第一次加载：从网络请求
var bitmap1 = ImageManager.loadCharacter('Actor1', 0);
console.log(bitmap1._loadingState);  // "loading"

// 第二次加载：直接从缓存返回
var bitmap2 = ImageManager.loadCharacter('Actor1', 0);
console.log(bitmap1 === bitmap2);    // true

// 不同色调会创建新缓存
var bitmap3 = ImageManager.loadCharacter('Actor1', 180);
console.log(bitmap1 === bitmap3);    // false
```

## 三、各类型加载方法

| 方法 | 路径 | 用途 |
|------|------|------|
| `loadCharacter` | img/characters/ | 角色行走图 |
| `loadFace` | img/faces/ | 角色脸图 |
| `loadTileset` | img/tilesets/ | 瓦片集 |
| `loadSystem` | img/system/ | 系统图标 |
| `loadPicture` | img/pictures/ | 显示图片 |
| `loadParallax` | img/parallaxes/ | 远景图 |
| `loadBattleback1/2` | img/battlebacks1/2/ | 战斗背景 |
| `loadSvActor/Enemy` | img/sv_actors/enemies/ | 侧视战斗图 |

## 四、资源预约系统

```javascript
// 预约资源（场景切换前预加载）
ImageManager.reserveCharacter = function(filename, hue, reservationId) {
    var bitmap = this.loadCharacter(filename, hue);
    this._reserveBitmap(bitmap, reservationId);
    return bitmap;
};

// 释放预约（场景终止时）
ImageManager.releaseReservation = function(reservationId) {
    if (this._reserved[reservationId]) {
        this._reserved[reservationId].forEach(function(bitmap) {
            bitmap._referenceCount--;
        });
        delete this._reserved[reservationId];
    }
};
```

**使用场景**：
```javascript
// Scene_Map.create 中预约地图资源
Scene_Map.prototype.create = function() {
    this._reservationId = 'scene_map';
    ImageManager.reserveTileset($gameMap.tilesetName(), 0, this._reservationId);
};

// Scene_Map.terminate 中释放
Scene_Map.prototype.terminate = function() {
    ImageManager.releaseReservation(this._reservationId);
};
```

## 五、色调变化原理

```javascript
Bitmap.prototype.rotateHue = function(offset) {
    var imageData = this._context.getImageData(0, 0, this.width, this.height);
    var pixels = imageData.data;
    
    for (var i = 0; i < pixels.length; i += 4) {
        var r = pixels[i];
        var g = pixels[i + 1];
        var b = pixels[i + 2];
        
        // RGB → HSL → 旋转H → RGB
        var hsl = this.rgbToHsl(r, g, b);
        hsl[0] = (hsl[0] + offset) % 360;
        var rgb = this.hslToRgb(hsl[0], hsl[1], hsl[2]);
        
        pixels[i] = rgb[0];
        pixels[i + 1] = rgb[1];
        pixels[i + 2] = rgb[2];
    }
    
    this._context.putImageData(imageData, 0, 0);
};
```

## 六、异步加载队列

```javascript
// 逐个解码图像，避免内存峰值
ImageManager._requestQueue = new RequestQueue();

RequestQueue.prototype.update = function() {
    if (this._queue.length > 0) {
        var item = this._queue[0];
        item.bitmap.decode();  // 解码图像数据
        
        if (item.bitmap.isReady()) {
            this.dequeue();    // 完成后移除
            this.update();     // 处理下一个
        }
    }
};
```

## 七、调试案例

```javascript
// 查看缓存内容
console.log('缓存数量:', Object.keys(ImageManager._cache).length);
Object.keys(ImageManager._cache).forEach(function(key) {
    var bitmap = ImageManager._cache[key];
    console.log(key, bitmap.width + 'x' + bitmap.height);
});

// 检测加载状态
var bitmap = ImageManager.loadCharacter('Actor1', 0);
bitmap.addLoadListener(function() {
    console.log('图像加载完成:', bitmap.width, bitmap.height);
});

// 计算缓存内存占用
var totalSize = 0;
for (var key in ImageManager._cache) {
    var b = ImageManager._cache[key];
    totalSize += b.width * b.height * 4;  // RGBA
}
console.log('缓存大小:', (totalSize / 1024 / 1024).toFixed(2), 'MB');
```
