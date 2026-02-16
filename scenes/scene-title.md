# Scene_Title - 标题场景

## 职责

显示游戏标题画面，提供开始游戏、继续游戏、选项等入口。

## 核心组件

```javascript
Scene_Title.prototype.create = function() {
    Scene_Base.prototype.create.call(this);
    this.createBackground();       // 创建背景
    this.createForeground();       // 创建前景(标题)
    this.createWindowLayer();      // 创建窗口层
    this.createCommandWindow();    // 创建命令窗口
};
```

## 背景图像

```javascript
Scene_Title.prototype.createBackground = function() {
    this._backSprite1 = new Sprite(
        ImageManager.loadTitle1($dataSystem.title1Name)
    );
    this._backSprite2 = new Sprite(
        ImageManager.loadTitle2($dataSystem.title2Name)
    );
    this.addChild(this._backSprite1);
    this.addChild(this._backSprite2);
};
```

## 前景图像 (游戏标题)

```javascript
Scene_Title.prototype.createForeground = function() {
    this._gameTitleSprite = new Sprite(
        new Bitmap(Graphics.width, Graphics.height)
    );
    
    this._gameTitleSprite.bitmap.outlineColor = 'black';
    this._gameTitleSprite.bitmap.outlineWidth = 8;
    this._gameTitleSprite.bitmap.fontSize = 72;
    this._gameTitleSprite.bitmap.drawText(
        $dataSystem.gameTitle,
        0, Graphics.height / 4, Graphics.width, 80, 'center'
    );
    
    this.addChild(this._gameTitleSprite);
};
```

## 命令窗口

```javascript
Scene_Title.prototype.createCommandWindow = function() {
    this._commandWindow = new Window_TitleCommand();
    this._commandWindow.setHandler('newGame', this.commandNewGame.bind(this));
    this._commandWindow.setHandler('continue', this.commandContinue.bind(this));
    this._commandWindow.setHandler('options', this.commandOptions.bind(this));
    this.addWindow(this._commandWindow);
};
```

### Window_TitleCommand

```javascript
function Window_TitleCommand() {
    this.initialize.apply(this, arguments);
}

Window_TitleCommand.prototype.makeCommandList = function() {
    this.addCommand(TextManager.newGame, 'newGame');
    this.addCommand(TextManager.continue_, 'continue', this.isContinueEnabled());
    this.addCommand(TextManager.options, 'options');
};
```

## 命令处理

### 新游戏

```javascript
Scene_Title.prototype.commandNewGame = function() {
    DataManager.setupNewGame();
    this._commandWindow.close();
    this.fadeOutAll();
    SceneManager.goto(Scene_Map);
};
```

### 继续游戏

```javascript
Scene_Title.prototype.commandContinue = function() {
    this._commandWindow.close();
    SceneManager.push(Scene_Load);
};
```

### 选项

```javascript
Scene_Title.prototype.commandOptions = function() {
    this._commandWindow.close();
    SceneManager.push(Scene_Options);
};
```

## 检测存档

```javascript
Scene_Title.prototype.isContinueEnabled = function() {
    return DataManager.isAnySavefileExists();
};

Scene_Title.prototype.start = function() {
    Scene_Base.prototype.start.call(this);
    this._commandWindow.open();
    
    // 更新继续按钮状态
    if (!this.isContinueEnabled()) {
        this._commandWindow.drawItem(1, false);
    }
};
```

## 背景音乐

```javascript
Scene_Title.prototype.start = function() {
    Scene_Base.prototype.start.call(this);
    SceneManager.snapForBackground();
    this.playTitleMusic();
};

Scene_Title.prototype.playTitleMusic = function() {
    var bgm = $dataSystem.titleBgm;
    if (bgm.name) {
        AudioManager.playBgm(bgm);
    }
};
```

## 绘制流程

```
Scene_Title
│
├── _backSprite1 (背景层1)
│
├── _backSprite2 (背景层2)
│
├── _gameTitleSprite (游戏标题)
│
└── _windowLayer
        │
        └── _commandWindow
                ├── 新游戏
                ├── 继续 (灰色无存档)
                └── 选项
```
