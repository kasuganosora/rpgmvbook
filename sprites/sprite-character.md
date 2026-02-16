# Sprite_Character - 角色精灵

## 一、职责

Sprite_Character 负责将 Game_Character 渲染到屏幕上：
- 加载角色行走图
- 处理动画帧
- 更新位置和方向
- 渲染瓦片角色（事件图块）

## 二、核心属性

```javascript
function Sprite_Character() {
    this.initialize.apply(this, arguments);
}

Sprite_Character.prototype.initialize = function(character) {
    Sprite_Base.prototype.initialize.call(this);
    
    this._character = character;  // 关联的游戏对象
    this._balloonSprite = null;   // 气球精灵
    this._tileId = 0;             // 瓦片ID（事件图块）
    this._characterName = '';     // 行走图文件名
    this._characterIndex = 0;     // 行走图索引
    this._pattern = 0;            // 动画图案
    this._motionType = null;      // 动作类型
    this._motionSpeed = 12;       // 动画速度
    
    this.updateBitmap();          // 初始化图像
    this.updatePosition();        // 初始化位置
};
```

## 三、update - 每帧更新

```javascript
Sprite_Character.prototype.update = function() {
    Sprite_Base.prototype.update.call(this);
    
    // 更新位图（如果行走图改变）
    this.updateBitmap();
    
    // 更新位置（像素坐标）
    this.updatePosition();
    
    // 更新动画
    this.updateAnimation();
    
    // 更新气球
    this.updateBalloon();
    
    // 更新其他属性
    this.updateOther();
};
```

### 3.1 updateBitmap 详解

```javascript
Sprite_Character.prototype.updateBitmap = function() {
    // 检测是瓦片角色还是行走图角色
    if (this._character.isTile()) {
        // 瓦片角色（事件设置中选择的图块）
        this.updateTileBitmap();
    } else {
        // 行走图角色
        this.updateCharacterBitmap();
    }
};

Sprite_Character.prototype.updateCharacterBitmap = function() {
    // 检测行走图是否改变
    var name = this._character.characterName();
    var index = this._character.characterIndex();
    
    if (name !== this._characterName || index !== this._characterIndex) {
        this._characterName = name;
        this._characterIndex = index;
        this.setCharacterBitmap();
    }
};

Sprite_Character.prototype.setCharacterBitmap = function() {
    // 加载行走图
    this.bitmap = ImageManager.loadCharacter(this._characterName);
    
    // 计算行走图在图像中的位置
    // 行走图格式：4行×2组（每组3列）
    // 每个角色占 3列×4行
    this._isBigCharacter = this._characterName.charAt(0) === '$';
    
    if (this._isBigCharacter) {
        // 大行走图（单角色，整张图）
        this._frameWidth = this.bitmap.width / 3;
        this._frameHeight = this.bitmap.height / 4;
    } else {
        // 普通行走图（8角色，每角色3×4）
        this._frameWidth = this.bitmap.width / 12;
        this._frameHeight = this.bitmap.height / 8;
    }
};
```

**行走图文件结构**：
```
普通行走图 (Actor1.png, 288×256):
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│角色0│     │     │角色1│     │     │角色2│     │     │角色3│     │     │
│ 左  │ 中  │ 右  │ 左  │ 中  │ 右  │ 左  │ 中  │ 右  │ 左  │ 中  │ 右  │
├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
│角色4│     │     │角色5│     │     │角色6│     │     │角色7│     │     │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
每列48px，每角色宽144px

大行走图 ($Actor1.png):
┌───────────┬───────────┬───────────┐
│    左     │    中     │    右     │
├───────────┼───────────┼───────────┤
│    下     │           │           │
├───────────┼───────────┼───────────┤
│    左     │           │           │
├───────────┼───────────┼───────────┤
│    上     │           │           │
└───────────┴───────────┴───────────┘
整张图是一个角色
```

### 3.2 updateFrame 详解

```javascript
Sprite_Character.prototype.updateFrame = function() {
    var pattern = this._character.pattern();
    var direction = this._character.direction();
    
    // 计算源矩形
    var pw = this._frameWidth;
    var ph = this._frameHeight;
    var sx = 0;
    var sy = 0;
    
    if (this._isBigCharacter) {
        // 大行走图
        sx = pattern * pw;
        sy = (direction - 2) / 2 * ph;  // 2→0, 4→1, 6→2, 8→3
    } else {
        // 普通行走图
        var row = Math.floor(this._characterIndex / 4);
        var col = this._characterIndex % 4;
        sx = (col * 3 + pattern) * pw;
        sy = (row * 4 + (direction - 2) / 2) * ph;
    }
    
    this.setFrame(sx, sy, pw, ph);
};

// 方向到行索引的转换
// direction=2(下) → row=0
// direction=4(左) → row=1
// direction=6(右) → row=2
// direction=8(上) → row=3
```

### 3.3 updatePosition 详解

```javascript
Sprite_Character.prototype.updatePosition = function() {
    // 将瓦片坐标转为屏幕像素坐标
    this.x = $gameMap.adjustX(this._character._realX) * $gameMap.tileWidth();
    this.y = $gameMap.adjustY(this._character._realY) * $gameMap.tileHeight();
    
    // Y偏移（使角色底部对齐瓦片底部）
    this.y -= this._frameHeight - $gameMap.tileHeight();
    
    // Z坐标（用于深度排序）
    this.z = this._character.screenZ();
};

// adjustX/Y 处理地图滚动和循环
Game_Map.prototype.adjustX = function(x) {
    // 计算相对于屏幕的X坐标
    var screenX = x - this._displayX;
    
    // 循环地图处理
    if (this.isLoopHorizontal() && screenX < -this.width() / 2) {
        screenX += this.width();
    }
    
    return screenX;
};
```

## 四、瓦片角色渲染

```javascript
Sprite_Character.prototype.updateTileBitmap = function() {
    var tileId = this._character.tileId();
    
    if (tileId !== this._tileId) {
        this._tileId = tileId;
        this.bitmap = ImageManager.loadSystem('TileA');  // 或使用瓦片集
        
        // 计算瓦片在图像中的位置
        var sx = (tileId % 8) * 48;
        var sy = Math.floor(tileId / 8) * 48;
        this.setFrame(sx, sy, 48, 48);
    }
};
```

## 五、气球动画

```javascript
Sprite_Character.prototype.setupBalloon = function() {
    if (this._character.balloonId() > 0) {
        this._balloonSprite = new Sprite_Balloon();
        this._balloonSprite.x = this._frameWidth / 2;
        this._balloonSprite.y = -this._frameHeight;
        this._balloonSprite.setup(this._character.balloonId());
        this.addChild(this._balloonSprite);
    }
};

// 气球类型
// 1: 感叹号  2: 问号  3: 音乐符
// 4: 心形    5: 愤怒   6: 汗水
// 7: 挠头    8: 沉默   9: 灯泡
// 10: ZZZ
```

## 六、调试案例

### 案例1：查看角色精灵状态

```javascript
var sprite = SceneManager._scene._spriteset._characterContainer.children[0];
console.log('行走图:', sprite._characterName, '索引:', sprite._characterIndex);
console.log('帧尺寸:', sprite._frameWidth, 'x', sprite._frameHeight);
console.log('屏幕位置:', sprite.x, sprite.y);
console.log('Z坐标:', sprite.z);
```

### 案例2：动态更换行走图

```javascript
// 修改玩家行走图
$gamePlayer.setImage('Actor2', 0);
// 参数：文件名, 索引

// 修改事件行走图
var event = $gameMap.event(1);
event.setImage('Actor3', 2);
```

### 案例3：显示气球

```javascript
// 在玩家头顶显示感叹号
$gamePlayer.requestBalloon(1);

// 气球类型
// 1: !!!   2: ???   3: ♪
// 4: ♥     5: 愤怒   6: 汗
// 7: 挠头  8: ...    9: 💡
```

### 案例4：强制更新精灵

```javascript
// 获取所有角色精灵并强制更新
var sprites = SceneManager._scene._spriteset._characterContainer.children;
sprites.forEach(function(sprite) {
    sprite.updateBitmap();
    sprite.updateFrame();
});
```

### 案例5：修改动画速度

```javascript
// 加快动画（每6帧切换一次图案）
$gamePlayer._animationCount = 0;
$gamePlayer._stepAnime = true;
// 然后修改 Sprite_Character.prototype._motionSpeed = 6;
```
