# AudioManager - 音频管理器

## 一、音频类型

| 类型 | 循环 | 用途 |
|------|------|------|
| BGM | 是 | 背景音乐 |
| BGS | 是 | 背景声音（雨声、风声） |
| ME | 否 | 音乐效果（胜利、失败） |
| SE | 否 | 音效（选择、攻击） |

## 二、BGM 播放详解

```javascript
AudioManager.playBgm = function(bgm, pos) {
    // 如果是同一首BGM，只更新参数
    if (this.isCurrentBgm(bgm)) {
        this.updateBgmParameters(bgm);
        return;
    }
    
    // 停止当前BGM
    this.stopBgm();
    
    if (bgm.name) {
        // 创建音频缓冲
        this._bgmBuffer = this.createBuffer('bgm', bgm.name);
        this.updateBgmParameters(bgm);
        
        // 播放（true=循环，pos=起始位置）
        this._bgmBuffer.play(true, pos || 0);
        this._currentBgm = bgm;
    }
};

// 音频对象结构
var bgm = {
    name: 'Theme1',    // 文件名
    volume: 80,        // 音量 0-100
    pitch: 100,        // 音调 50-150
    pan: 0             // 声像 -100~100
};
```

## 三、ME 的特殊处理

```javascript
AudioManager.playMe = function(me) {
    this.stopMe();
    
    if (me.name) {
        // ME播放时暂停BGM（这是特殊设计）
        if (this._bgmBuffer && this._currentBgm) {
            this._currentBgm.pos = this._bgmBuffer.seek();  // 记录位置
            this._bgmBuffer.stop();                          // 暂停
        }
        
        this._meBuffer = this.createBuffer('me', me.name);
        this._meBuffer.play(false);  // 不循环
        this._currentMe = me;
    }
};

// ME 播放结束后恢复 BGM
AudioManager.meOnEnd = function() {
    // 检查是否还有 ME 在播放
    if (!this._meBuffer && this._currentBgm) {
        // 从暂停位置恢复
        this.playBgm(this._currentBgm, this._currentBgm.pos);
    }
};
```

## 四、音量计算

```javascript
// 最终音量 = 主音量 × 类型音量 × 音频音量
AudioManager.updateBufferParameters = function(buffer, configVolume, audio) {
    if (buffer && audio) {
        // 计算：1.0 × 100 × 80 / 10000 = 0.8
        var volume = this._masterVolume * configVolume * audio.volume / 10000;
        var pitch = audio.pitch / 100;  // 100 → 1.0（正常速度）
        var pan = audio.pan / 100;      // 0 → 中央
        
        buffer.volume = volume;
        buffer.pitch = pitch;
        buffer.pan = pan;
    }
};
```

## 五、淡入淡出

```javascript
// 2秒淡出BGM
AudioManager.fadeOutBgm = function(duration) {
    if (this._bgmBuffer && this._currentBgm) {
        this._bgmBuffer.fadeOut(duration);
        this._currentBgm = null;
    }
};

// WebAudio 的 fadeOut 实现
WebAudio.prototype.fadeOut = function(duration) {
    var startVolume = this._volume;
    var startTime = Date.now();
    
    var fadeInterval = setInterval(function() {
        var elapsed = (Date.now() - startTime) / 1000;
        var progress = elapsed / duration;
        
        if (progress >= 1) {
            this._volume = 0;
            this.stop();
            clearInterval(fadeInterval);
        } else {
            this._volume = startVolume * (1 - progress);
        }
    }.bind(this), 16);
};
```

## 六、Web Audio API 节点链

```
AudioContext
    │
    ▼
[BufferSource] ─→ [GainNode] ─→ [StereoPanner] ─→ [Destination]
   (音频数据)      (音量控制)     (左右声道)       (扬声器)
```

```javascript
WebAudioSound.prototype.play = function(loop, offset) {
    // 创建音频源
    this._sourceNode = context.createBufferSource();
    this._sourceNode.buffer = this._buffer;
    this._sourceNode.loop = loop;
    
    // 音量节点
    this._gainNode = context.createGain();
    this._gainNode.gain.value = this._volume;
    
    // 声像节点
    this._panNode = context.createStereoPanner();
    this._panNode.pan.value = this._pan;
    
    // 连接节点
    this._sourceNode.connect(this._gainNode);
    this._gainNode.connect(this._panNode);
    this._panNode.connect(context.destination);
    
    // 播放
    this._sourceNode.start(0, offset);
};
```

## 七、调试案例

```javascript
// 查看当前播放状态
console.log('BGM:', AudioManager._currentBgm);
console.log('BGS:', AudioManager._currentBgs);
console.log('音量:', {
    master: AudioManager._masterVolume,
    bgm: AudioManager._bgmVolume,
    se: AudioManager._seVolume
});

// 手动播放BGM
AudioManager.playBgm({
    name: 'Theme1',
    volume: 80,
    pitch: 100,
    pan: 0
});

// 手动停止所有音频
AudioManager.stopAll();

// 获取BGM当前播放位置
var pos = AudioManager._bgmBuffer ? AudioManager._bgmBuffer.seek() : 0;
console.log('当前播放位置:', pos, '秒');
```
