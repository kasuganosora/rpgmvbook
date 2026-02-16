# SceneManager - 场景管理器

## 一、启动流程

```javascript
// main.js
SceneManager.run(Scene_Boot);

// run 做了三件事
SceneManager.run = function(sceneClass) {
    this.initialize();      // 初始化子系统
    this.goto(sceneClass);  // 设置第一个场景
    this.requestUpdate();   // 启动游戏循环
};
```

## 二、初始化

```javascript
SceneManager.initialize = function() {
    this.initGraphics();  // Canvas/WebGL 渲染器 (816×624)
    this.initAudio();     // AudioContext
    this.initInput();     // 键盘/鼠标/触摸
};
```

## 三、主循环

```javascript
SceneManager.update = function() {
    this.tickStart();
    this.updateMain();     // 核心逻辑
    this.tickEnd();
    this.requestUpdate();  // 请求下一帧 (60FPS)
};

SceneManager.updateMain = function() {
    this.changeScene();    // 切换场景
    this.updateScene();    // 更新场景
    this.renderScene();    // 渲染画面
};
```

## 四、场景切换

```javascript
// 替换当前场景
SceneManager.goto = function(sceneClass) {
    this._nextScene = new sceneClass();
    if (this._scene) this._scene.stop();
};

// 压入栈（保留当前）
SceneManager.push = function(sceneClass) {
    this._stack.push(this._scene.constructor);
    this.goto(sceneClass);
};

// 弹出栈（返回）
SceneManager.pop = function() {
    if (this._stack.length > 0) {
        this.goto(this._stack.pop());
    }
};
```

## 五、场景生命周期

```
create() → isReady()? → start() → update()每帧 → stop() → terminate()
```

```javascript
SceneManager.changeScene = function() {
    if (this._scene) {
        this._scene.terminate();      // 1. 终止旧场景
    }
    this._scene = this._nextScene;
    this._nextScene = null;
    if (this._scene) {
        this._scene.create();         // 2. 创建新场景
    }
};

SceneManager.updateScene = function() {
    if (!this._sceneStarted && this._scene.isReady()) {
        this._scene.start();          // 3. 启动场景
        this._sceneStarted = true;
    }
    if (this._sceneStarted) {
        this._scene.update();         // 4. 每帧更新
    }
};
```

## 六、调试案例

```javascript
// 监控场景切换
var originalGoto = SceneManager.goto;
SceneManager.goto = function(sceneClass) {
    console.log('场景切换:', sceneClass.name);
    return originalGoto.apply(this, arguments);
};

// 查看场景栈
console.log('栈:', SceneManager._stack.map(c => c.name));
console.log('当前:', SceneManager._scene.constructor.name);
```
