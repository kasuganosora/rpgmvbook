# 第八章：地图与场景

## 场景的概念

**场景**是游戏中的一个独立状态或画面：

- 标题场景：显示游戏标题
- 地图场景：探索世界
- 战斗场景：回合制战斗
- 菜单场景：物品、装备管理

**同一时间只有一个场景处于活动状态**。

---

## 场景基类

```javascript
// ============================================
// Scene - 场景基类
// ============================================

function Scene() {
    this.objects = [];     // 场景中的对象
    this.isActive = false; // 是否激活
}

// 进入场景时调用
Scene.prototype.enter = function() {
    this.isActive = true;
};

// 离开场景时调用
Scene.prototype.leave = function() {
    this.isActive = false;
};

// 更新
Scene.prototype.update = function() {
    for (var i = 0; i < this.objects.length; i++) {
        this.objects[i].update();
    }
};

// 渲染
Scene.prototype.render = function(ctx) {
    for (var i = 0; i < this.objects.length; i++) {
        this.objects[i].render(ctx);
    }
};
```

---

## 场景管理器

```javascript
// ============================================
// SceneManager - 场景管理器
// ============================================

var SceneManager = {
    currentScene: null,
    nextScene: null,

    // 切换场景
    changeScene: function(newScene) {
        this.nextScene = newScene;
    },

    // 更新（处理场景切换）
    update: function() {
        // 处理场景切换
        if (this.nextScene) {
            if (this.currentScene) {
                this.currentScene.leave();
            }
            this.currentScene = this.nextScene;
            this.currentScene.enter();
            this.nextScene = null;
        }

        // 更新当前场景
        if (this.currentScene) {
            this.currentScene.update();
        }
    },

    // 渲染
    render: function(ctx) {
        if (this.currentScene) {
            this.currentScene.render(ctx);
        }
    }
};
```

---

## 地图场景示例

```javascript
// ============================================
// Scene_Map - 地图场景
// ============================================

function Scene_Map(mapId) {
    Scene.call(this);
    this.mapId = mapId;
    this.map = null;
    this.player = null;
    this.npcs = [];
    this.cameraX = 0;
    this.cameraY = 0;
}

Scene_Map.prototype = Object.create(Scene.prototype);

Scene_Map.prototype.enter = function() {
    Scene.prototype.enter.call(this);

    // 加载地图
    this.map = loadMap(this.mapId);

    // 创建玩家
    this.player = new Player(100, 100);
    this.objects.push(this.player);

    // 创建 NPC
    this.npcs.push(new NPC(200, 150, '村民', '你好！'));
    this.objects.push(this.npcs[0]);
};

Scene_Map.prototype.update = function() {
    Scene.prototype.update.call(this);

    // 更新相机
    this.updateCamera();
};

Scene_Map.prototype.updateCamera = function() {
    var targetX = this.player.x - canvas.width / 2;
    var targetY = this.player.y - canvas.height / 2;

    // 平滑跟随
    this.cameraX += (targetX - this.cameraX) * 0.1;
    this.cameraY += (targetY - this.cameraY) * 0.1;

    // 边界限制
    this.cameraX = Math.max(0, Math.min(this.cameraX, this.map.width - canvas.width));
    this.cameraY = Math.max(0, Math.min(this.cameraY, this.map.height - canvas.height));
};

Scene_Map.prototype.render = function(ctx) {
    // 清屏
    ctx.fillStyle = '#000';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // 渲染地图
    this.map.render(ctx, this.cameraX, this.cameraY);

    // 渲染对象
    for (var i = 0; i < this.objects.length; i++) {
        this.objects[i].render(ctx, this.cameraX, this.cameraY);
    }
};
```

---

## 本章小结

1. **场景**是游戏的独立状态
2. **SceneManager** 管理场景切换
3. **地图场景**包含地图、玩家、NPC

## 下一章

[第九章：输入处理](./09-input.md) - 完善输入系统
