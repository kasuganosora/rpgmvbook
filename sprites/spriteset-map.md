# Spriteset_Map - 地图精灵集

## 一、结构概览

```
Spriteset_Map (PIXI.Container)
├── _baseSprite (背景容器)
│   └── _backgroundBitmap (模糊截图)
├── _parallax (远景图精灵)
├── _tilemap (瓦片地图)
├── _characterContainer (角色精灵容器)
│   └── [Sprite_Character] × N
├── _shadowSprite (飞船阴影)
├── _destinationSprite (目标指示)
├── _weather (天气效果)
├── _pictureContainer (图片精灵)
└── _timerSprite (计时器)
```

## 二、createLowerLayer - 创建底层

```javascript
Spriteset_Map.prototype.createLowerLayer = function() {
    // 创建背景（用于菜单背景）
    this.createBackground();
    
    // 创建远景图
    this.createParallax();
    
    // 创建瓦片地图
    this.createTilemap();
    
    // 创建角色容器
    this.createCharacters();
    
    // 创建阴影
    this.createShadow();
    
    // 创建目标指示
    this.createDestination();
    
    // 创建天气
    this.createWeather();
};
```

### 2.1 createBackground 详解

```javascript
Spriteset_Map.prototype.createBackground = function() {
    this._backgroundSprite = new Sprite();
    
    // 当打开菜单时，会调用 snapForBackground
    // 创建模糊的地图截图作为菜单背景
    this._backgroundSprite.bitmap = new Bitmap(Graphics.width, Graphics.height);
    
    this._baseSprite = new Sprite();
    this._baseSprite.addChild(this._backgroundSprite);
    this.addChild(this._baseSprite);
};
```

### 2.2 createTilemap 详解

```javascript
Spriteset_Map.prototype.createTilemap = function() {
    // 检测是否支持 WebGL
    if (Graphics.isWebGL()) {
        // 使用 ShaderTilemap（GPU加速）
        this._tilemap = new ShaderTilemap();
    } else {
        // 使用普通 Tilemap（Canvas）
        this._tilemap = new Tilemap();
    }
    
    // 设置地图尺寸
    this._tilemap.tileWidth = $gameMap.tileWidth();   // 48
    this._tilemap.tileHeight = $gameMap.tileHeight(); // 48
    
    // 设置地图数据
    this._tilemap.setData($gameMap.width(), $gameMap.height(), $gameMap.data());
    
    // 加载瓦片集图像
    this.loadTileset();
    
    this.addChild(this._tilemap);
};
```

### 2.3 createCharacters 详解

```javascript
Spriteset_Map.prototype.createCharacters = function() {
    this._characterContainer = new Sprite();
    
    // 为每个事件创建精灵
    $gameMap.events().forEach(function(event) {
        this._characterContainer.addChild(new Sprite_Character(event));
    }, this);
    
    // 创建交通工具精灵
    $gameMap.vehicles().forEach(function(vehicle) {
        this._characterContainer.addChild(new Sprite_Character(vehicle));
    }, this);
    
    // 创建玩家精灵
    this._characterContainer.addChild(new Sprite_Character($gamePlayer));
    
    this.addChild(this._characterContainer);
};
```

## 三、update - 每帧更新

```javascript
Spriteset_Map.prototype.update = function() {
    this.updateTileset();       // 更新瓦片集
    this.updateParallax();      // 更新远景图
    this.updateTilemap();       // 更新地图
    this.updateCharacters();    // 更新角色
    this.updateShadow();        // 更新阴影
    this.updateWeather();       // 更新天气
    this.updatePictures();      // 更新图片
    this.updateTimer();         // 更新计时器
    this.updateScreenFilters(); // 更新滤镜
};
```

### 3.1 updateTileset - 瓦片集切换

```javascript
Spriteset_Map.prototype.updateTileset = function() {
    // 检测瓦片集是否改变
    if (this._tileset !== $gameMap.tileset()) {
        this._tileset = $gameMap.tileset();
        this.loadTileset();
    }
};

Spriteset_Map.prototype.loadTileset = function() {
    // 清除旧图像
    this._tileset.tilesetNames.forEach(function(name, i) {
        if (name) {
            var bitmap = ImageManager.loadTileset(name);
            this._tilemap.bitmaps[i] = bitmap;
        }
    }, this);
    
    // 标记需要重新绘制
    this._tilemap.flags = $gameMap.tilesetFlags();
    this._tilemap.refresh();
};
```

### 3.2 updateParallax - 远景图滚动

```javascript
Spriteset_Map.prototype.updateParallax = function() {
    if (this._parallaxName !== $gameMap.parallaxName()) {
        this._parallaxName = $gameMap.parallaxName();
        this._parallax.bitmap = ImageManager.loadParallax(this._parallaxName);
    }
    
    if (this._parallax.bitmap) {
        // 计算滚动位置
        var sx = $gameMap.parallaxOx();
        var sy = $gameMap.parallaxOy();
        
        this._parallax.origin.x = sx;
        this._parallax.origin.y = sy;
    }
};
```

### 3.3 updateCharacters - 更新角色精灵

```javascript
Spriteset_Map.prototype.updateCharacters = function() {
    // 遍历所有角色精灵
    this._characterContainer.children.forEach(function(sprite) {
        sprite.update();
    });
    
    // Z排序（按Y坐标）
    this._characterContainer.children.sort(function(a, b) {
        return a.z - b.z;
    });
};
```

## 四、Z坐标系统

```javascript
// Tilemap 中定义的 Z 层
Tilemap.Z_LOWER_TILE    = 0;  // 下层瓦片
Tilemap.Z_LOWER_CHAR    = 1;  // 下层角色
Tilemap.Z_NORMAL_CHAR   = 3;  // 普通角色
Tilemap.Z_UPPER_TILE    = 4;  // 上层瓦片
Tilemap.Z_UPPER_CHAR    = 5;  // 上层角色
Tilemap.Z_AIRSHIP       = 6;  // 飞船
Tilemap.Z_SHADOW        = 7;  // 阴影
Tilemap.Z_BALLOON       = 8;  // 气球
Tilemap.Z_ANIMATION     = 9;  // 动画
Tilemap.Z_DESTINATION   = 9;  // 目标指示

// Sprite_Character 中计算 Z
Sprite_Character.prototype.z = function() {
    return this._character.screenZ();
};

Game_CharacterBase.prototype.screenZ = function() {
    return $gameMap.adjustZ(this._realY, this._priority);
};

// Y坐标转Z值
Game_Map.prototype.adjustZ = function(y, priority) {
    var z = y + priority * 0.1;
    return z;
};
```

**渲染顺序示例**：
```
角色A在Y=5: z = 5 + 0.1 = 5.1
角色B在Y=6: z = 6 + 0.1 = 6.1
上层瓦片在Y=5: z = 5 + 4 = 9

渲染顺序: 角色A → 角色B → 上层瓦片
（z小的先渲染，会被后面的覆盖）
```

## 五、天气系统

```javascript
Spriteset_Map.prototype.createWeather = function() {
    this._weather = new Weather();
    this.addChild(this._weather);
};

// Weather 类
function Weather() {
    this.initialize.apply(this, arguments);
}

Weather.prototype.update = function() {
    var type = $gameScreen.weatherType();
    var power = $gameScreen.weatherPower();
    
    this._type = type;
    this._power = power;
    
    // 根据类型创建粒子
    switch (type) {
        case 'rain':
            this.updateRain();
            break;
        case 'storm':
            this.updateStorm();
            break;
        case 'snow':
            this.updateSnow();
            break;
    }
};
```

## 六、调试案例

### 案例1：查看精灵数量

```javascript
console.log('角色精灵数:', $gameMap.events().length + 3); // 事件+3交通工具+玩家
console.log('图片精灵数:', $gameScreen._pictures.length);
console.log('瓦片集:', $gameMap.tileset().name);
```

### 案例2：访问特定精灵

```javascript
// 获取玩家精灵
var playerSprite = SceneManager._scene._spriteset._characterContainer.children
    .find(s => s._character === $gamePlayer);
console.log('玩家精灵:', playerSprite);

// 修改透明度
playerSprite.opacity = 128;
```

### 案例3：强制刷新地图

```javascript
// 切换瓦片集后刷新
SceneManager._scene._spriteset.loadTileset();
```

### 案例4：获取所有事件精灵

```javascript
var sprites = SceneManager._scene._spriteset._characterContainer.children
    .filter(s => s._character && s._character.eventId);
sprites.forEach(s => {
    console.log(s._character.event().name, 'z=', s.z);
});
```
