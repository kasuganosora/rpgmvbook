# ShaderTilemap 加速

## 概述

`ShaderTilemap` 是 Tilemap 的 WebGL 优化版本，利用 GPU 加速地图渲染。

## 类继承

```javascript
function ShaderTilemap() {
    this.initialize.apply(this, arguments);
}

ShaderTilemap.prototype = Object.create(Tilemap.prototype);
ShaderTilemap.prototype.constructor = ShaderTilemap;
```

## 核心区别

### Tilemap (Canvas 2D)

```
地图数据 → CPU绘制到位图 → Canvas渲染
         (慢，每帧计算量大)
```

### ShaderTilemap (WebGL)

```
地图数据 → GPU纹理 → Shader渲染
         (快，利用GPU并行计算)
```

## 实现原理

### 使用 PIXI.Tilemap

```javascript
ShaderTilemap.prototype._createSprites = function() {
    // 使用 pixi-tilemap.js 提供的 TileRenderer
    this._lowerZLayer = new PIXI.tilemap.ZLayer(this, 0);
    this._upperZLayer = new PIXI.tilemap.ZLayer(this, 4);
    
    this.addChild(this._lowerZLayer);
    this.addChild(this._upperZLayer);
};
```

### 图层管理

```javascript
ShaderTilemap.prototype._createLayers = function() {
    // 每个图层使用独立的 CompositeRectTileLayer
    for (var i = 0; i < 4; i++) {
        var layer = new PIXI.tilemap.CompositeRectTileLayer(0);
        if (i < 2) {
            this._lowerZLayer.addChild(layer);
        } else {
            this._upperZLayer.addChild(layer);
        }
        this._layers.push(layer);
    }
};
```

### 瓦片绘制

```javascript
ShaderTilemap.prototype._drawTile = function(layer, tileId, dx, dy) {
    if (tileId <= 0) return;
    
    if (Tilemap.isAutotile(tileId)) {
        this._drawAutotile(layer, tileId, dx, dy);
    } else {
        this._drawNormalTile(layer, tileId, dx, dy);
    }
};

ShaderTilemap.prototype._drawNormalTile = function(layer, tileId, dx, dy) {
    var setNumber = 4 + Math.floor(tileId / 256);
    var w = this._tileWidth;
    var h = this._tileHeight;
    var sx = (Math.floor(tileId / 128) % 2) * w;
    var sy = (tileId % 256) * h;
    var source = this.bitmaps[setNumber];
    
    if (source) {
        // 直接添加到 GPU 图层
        layer.addRectSet(sx, sy, dx, dy, w, h, source);
    }
};
```

## WebGL 渲染流程

```
1. 创建瓦片纹理
   ImageManager.loadTileset() 
       ↓
   Bitmap._baseTexture (PIXI.BaseTexture)
       ↓
   上传到 GPU

2. 构建渲染指令
   layer.addRectSet()
       ↓
   记录瓦片位置和纹理坐标

3. GPU 渲染
   WebGLRenderer.flush()
       ↓
   一次性绘制所有瓦片
```

## 性能对比

### Canvas 2D (Tilemap)

```javascript
// 每个瓦片需要:
// 1. CPU 计算源位置
// 2. CPU 执行位图复制 (blt)
// 3. 更新 Canvas 像素数据

// 1000个瓦片 = 1000次 CPU 操作
```

### WebGL (ShaderTilemap)

```javascript
// 所有瓦片:
// 1. CPU 只计算位置 (极少)
// 2. 批量提交到 GPU
// 3. GPU 一次绘制所有

// 1000个瓦片 = 1次 GPU draw call
```

## Shader 实现

### 顶点着色器

```glsl
// pixi-tilemap.js 内置
attribute vec2 aVertexPosition;
attribute vec2 aTextureCoord;

uniform mat3 translationMatrix;
uniform mat3 projectionMatrix;

varying vec2 vTextureCoord;

void main() {
    vTextureCoord = aTextureCoord;
    gl_Position = vec4((projectionMatrix * translationMatrix * vec3(aVertexPosition, 1.0)).xy, 0.0, 1.0);
}
```

### 片段着色器

```glsl
precision mediump float;

varying vec2 vTextureCoord;
uniform sampler2D uSampler;

void main() {
    gl_FragColor = texture2D(uSampler, vTextureCoord);
}
```

## 刷新机制

```javascript
ShaderTilemap.prototype.refreshTileset = function() {
    // 确保所有位图都已创建 PIXI 纹理
    var bitmaps = this.bitmaps.map(function(x) { 
        return x._baseTexture ? new PIXI.Texture(x._baseTexture) : x; 
    });
    
    // 通知图层更新纹理
    this._layers.forEach(function(layer) {
        layer.setBitmaps(bitmaps);
    });
};

ShaderTilemap.prototype.refresh = function() {
    // 完全重建所有图层
    Tilemap.prototype.refresh.call(this);
    this._layers.forEach(function(layer) {
        layer.clear();
    });
    this._paintAllTiles(this._lastStartX, this._lastStartY);
};
```

## 自动检测

```javascript
// rpg_core.js - Graphics
Graphics.isWebGL = function() {
    return this._renderer.type === PIXI.RENDERER_TYPE.WEBGL;
};

// 使用条件
if (Graphics.isWebGL()) {
    tilemap = new ShaderTilemap();
} else {
    tilemap = new Tilemap();
}
```

## 兼容性

| 特性 | Tilemap | ShaderTilemap |
|------|---------|---------------|
| Canvas 2D | ✅ | ❌ |
| WebGL | ✅ (慢) | ✅ (快) |
| 移动端 | ✅ | 视设备而定 |
| 自动回退 | N/A | ✅ 自动切换 |

## 调试

```javascript
// 强制使用 Canvas 模式 (调试用)
Graphics._renderer = new PIXI.CanvasRenderer(width, height);

// 检查当前渲染模式
console.log(Graphics._renderer.type);  // 0=UNKNOWN, 1=WEBGL, 2=CANVAS
```

## 注意事项

1. **纹理限制**: WebGL 有纹理大小限制，超大的瓦片集可能无法加载
2. **内存占用**: WebGL 需要将所有纹理保持在显存中
3. **兼容性**: 某些旧设备不支持 WebGL，会自动回退

## 项目配置

本项目使用的 pixi-tilemap 版本位于：

```
Project1/js/libs/pixi-tilemap.js
```

通过 `rpg_core.js` 中的 `ShaderTilemap` 类实现 WebGL 加速渲染。
