# 地图数据格式详解

## 一、MapXXX.json 结构

```javascript
{
    "data": [0, 0, 0, 2816, 2816, ...],  // 瓦片ID数组（核心！）
    "events": [null, {...}, {...}],       // 事件列表
    "width": 17,                          // 地图宽度（瓦片数）
    "height": 13,                         // 地图高度（瓦片数）
    "scrollType": 0,                      // 滚动类型
    "tilesetId": 1,                       // 瓦片集ID
    "parallaxName": "",                   // 远景图名称
    "parallaxLoopX": false,               // 远景X循环
    "parallaxLoopY": false,               // 远景Y循环
    "parallaxSx": 0,                      // 远景X速度
    "parallaxSy": 0,                      // 远景Y速度
    "parallaxShow": false,                // 远景显示
    "note": ""                            // 备注
}
```

## 二、瓦片数据数组详解

### 2.1 数组长度计算

```javascript
// data 数组长度 = width × height × 4
// 每个位置有4个图层的数据

// Map001.json (17×13)
// data.length = 17 × 13 × 4 = 884

// 示例：5×3 地图的 data 数组
// [层0(5×3), 层1(5×3), 层2(5×3), 层3(5×3)]
// = 5×3 + 5×3 + 5×3 + 5×3 = 60个元素
```

### 2.2 瓦片索引计算

```javascript
// 获取 (x, y) 位置第 z 层的瓦片ID
function getTileId(mapData, x, y, z, mapWidth) {
    var layerSize = mapWidth * $gameMap.height();
    var index = (z * layerSize) + (y * mapWidth) + x;
    return mapData[index];
}

// 实际代码（rpg_objects.js）
Game_Map.prototype.tileId = function(x, y, z) {
    var width = $dataMap.width;
    var height = $dataMap.height;
    return $dataMap.data[(z * height + y) * width + x];
};

// 示例：17×13 地图，获取 (5, 3) 位置第 0 层
// index = (0 * 13 + 3) * 17 + 5 = 3 * 17 + 5 = 56
// tileId = $dataMap.data[56]
```

### 2.3 瓦片ID编码

```javascript
// 瓦片ID范围：
// 0: 空瓦片
// 1-4095: A层瓦片（自动瓦片+普通瓦片）
// 4096+: B/C/D层瓦片

// A层瓦片细分：
// 1-2047: 自动瓦片（Auto-tiles A1-A5）
// 2048-4095: 普通A层瓦片（A5）

// 计算瓦片来源
function getTileSource(tileId) {
    if (tileId === 0) return { layer: 'none', tileset: null };
    if (tileId < 2048) {
        // 自动瓦片 A1-A4
        var autoTileIndex = Math.floor((tileId - 1) / 48);
        return { 
            layer: 'A1-A4', 
            autotile: autoTileIndex,
            shape: (tileId - 1) % 48 
        };
    }
    if (tileId < 4096) {
        // A5 普通瓦片
        return { 
            layer: 'A5', 
            index: tileId - 2048 
        };
    }
    // B/C/D 层瓦片
    var bcdIndex = tileId - 4096;
    var tilesetIndex = Math.floor(bcdIndex / 256);
    var localIndex = bcdIndex % 256;
    return { 
        layer: ['B', 'C', 'D'][tilesetIndex],
        index: localIndex 
    };
}
```

## 三、实际案例分析

### 3.1 Map001.json 数据

```javascript
// 读取 Map001.json
var data = $dataMap.data;
var width = $dataMap.width;   // 17
var height = $dataMap.height; // 13

// 查看某个位置的瓦片
function showTile(x, y) {
    for (var z = 0; z < 4; z++) {
        var id = data[(z * height + y) * width + x];
        console.log('层' + z + ': ID=' + id);
    }
}

showTile(0, 0);
// 输出示例：
// 层0: ID=2816  (A5瓦片，草地)
// 层1: ID=0     (空)
// 层2: ID=0     (空)
// 层3: ID=0     (空)
```

### 3.2 瓦片ID反推位置

```javascript
// 找出所有使用特定瓦片的位置
function findTiles(tileId) {
    var results = [];
    var data = $dataMap.data;
    var width = $dataMap.width;
    var height = $dataMap.height;
    
    for (var z = 0; z < 4; z++) {
        for (var y = 0; y < height; y++) {
            for (var x = 0; x < width; x++) {
                var id = data[(z * height + y) * width + x];
                if (id === tileId) {
                    results.push({x: x, y: y, z: z});
                }
            }
        }
    }
    return results;
}

// 找所有草地瓦片
var grassTiles = findTiles(2816);
console.log('草地位置:', grassTiles);
```

## 四、事件数据结构

```javascript
// $dataMap.events 数组
// 索引0是null，索引从1开始

// 事件结构
{
    "id": 1,
    "name": "EV001",
    "x": 8,
    "y": 12,
    "width": 1,
    "height": 1,
    "note": "",
    "pages": [
        {
            "conditions": {
                "actorId": 1,
                "actorValid": false,
                "itemId": 1,
                "itemValid": false,
                "selfSwitchCh": "A",
                "selfSwitchValid": false,
                "switch1Id": 1,
                "switch1Valid": false,
                "switch2Id": 1,
                "switch2Valid": false,
                "variableId": 1,
                "variableValid": false,
                "variableValue": 0
            },
            "directionFix": false,
            "image": {
                "tileId": 0,
                "characterName": "Actor1",
                "characterIndex": 0,
                "direction": 2,
                "pattern": 1,
                "priorityType": 1,  // 0=下层 1=普通 2=上层
                "tileId": 0,
                "transparency": 0
            },
            "list": [
                {
                    "code": 201,  // 传送指令
                    "indent": 0,
                    "parameters": [0, 2, 8, 6, 2, 0]
                    // [直接指定, 地图ID, X, Y, 方向, 淡入淡出]
                },
                {
                    "code": 0,  // 结束
                    "indent": 0,
                    "parameters": []
                }
            ],
            "moveFrequency": 3,
            "moveRoute": {
                "list": [{"code": 0}],
                "repeat": true,
                "skippable": false,
                "wait": false
            },
            "moveSpeed": 3,
            "moveType": 0,  // 0=固定 1=随机 2=靠近
            "priorityType": 1,
            "stepAnime": false,
            "through": false,
            "trigger": 1,  // 0=动作按钮 1=接触 2=自动 3=并行
            "walkAnime": true
        }
    ]
}
```

## 五、事件指令代码

| 代码 | 指令 |
|------|------|
| 0 | 结束 |
| 101 | 显示文字 |
| 102 | 显示选择 |
| 103 | 输入数字 |
| 104 | 选择物品 |
| 105 | 显示滚动文字 |
| 111 | 条件分支 |
| 112 | 循环 |
| 113 | 中断循环 |
| 115 | 中断事件处理 |
| 117 | 调用公共事件 |
| 118 | 标签 |
| 119 | 跳转到标签 |
| 121 | 控制开关 |
| 122 | 控制变量 |
| 123 | 控制独立开关 |
| 124 | 控制计时器 |
| 125 | 增减金币 |
| 126 | 增减物品 |
| 127 | 增减武器 |
| 128 | 增减防具 |
| 129 | 更换队伍 |
| 201 | 场所移动 |
| 202 | 设置交通工具位置 |
| 203 | 设置事件位置 |
| 204 | 获取位置信息 |
| 205 | 设置移动路线 |
| 206 | 乘坐/下船 |
| 211 | 透明设置 |
| 212 | 显示动画 |
| 213 | 显示气球 |
| 214 | 暂时消除事件 |
| 216 | 更改跟随状态 |
| 217 | 集合跟随角色 |
| 221 | 淡出画面 |
| 222 | 淡入画面 |
| 223 | 更改画面色调 |
| 224 | 画面闪烁 |
| 225 | 画面震动 |
| 230 | 等待 |
| 231 | 显示图片 |
| 232 | 移动图片 |
| 233 | 旋转图片 |
| 234 | 着色图片 |
| 235 | 消除图片 |
| 236 | 设置天气 |
| 241 | 播放BGM |
| 242 | 淡出BGM |
| 243 | 保存BGM |
| 244 | 恢复BGM |
| 245 | 播放BGS |
| 246 | 淡出BGS |
| 249 | 播放ME |
| 250 | 播放SE |
| 251 | 停止SE |
| 301 | 战斗处理 |
| 302 | 商店处理 |
| 303 | 名字输入处理 |
| 311 | 增减HP |
| 312 | 增减MP |
| 313 | 增减状态 |
| 314 | 全回复 |
| 315 | 增减经验 |
| 316 | 增减等级 |
| 317 | 增减能力值 |
| 318 | 增减技能 |
| 319 | 更换装备 |
| 320 | 更改名称 |
| 321 | 更改职业 |
| 322 | 更改图像 |
| 331 | 增减敌人HP |
| 332 | 增减敌人MP |
| 333 | 更改敌人状态 |
| 334 | 敌人出现 |
| 335 | 敌人变身 |
| 336 | 显示战斗动画 |
| 337 | 强制行动 |
| 338 | 中止战斗 |
| 339 | 调用菜单 |
| 340 | 游戏结束 |
| 341 | 返回标题 |
| 351 | 返回标题画面 |
| 352 | 打开存档画面 |
| 353 | 打开读取画面 |
| 354 | 游戏结束 |
| 355 | 脚本 |

## 六、调试案例

### 案例1：遍历地图瓦片

```javascript
var data = $dataMap.data;
var width = $dataMap.width;
var height = $dataMap.height;

for (var y = 0; y < height; y++) {
    for (var x = 0; x < width; x++) {
        var tileId = data[(0 * height + y) * width + x];
        if (tileId > 0) {
            console.log('(' + x + ',' + y + '):', tileId);
        }
    }
}
```

### 案例2：修改地图瓦片

```javascript
// 注意：这只是临时修改，不会保存
var x = 5, y = 3;
var index = (0 * $dataMap.height + y) * $dataMap.width + x;
$dataMap.data[index] = 2816;  // 改为草地

// 刷新地图显示
$gameMap.refresh();
```

### 案例3：查看事件指令

```javascript
var event = $dataMap.events[1];
var page = event.pages[0];
page.list.forEach(function(cmd, i) {
    console.log('指令' + i + ': 代码=' + cmd.code, cmd.parameters);
});
```

### 案例4：获取地图所有事件

```javascript
$dataMap.events.forEach(function(event, id) {
    if (event) {
        console.log('事件' + id + ':', event.name, 
                    '位置:', event.x + ',' + event.y);
    }
});
```
