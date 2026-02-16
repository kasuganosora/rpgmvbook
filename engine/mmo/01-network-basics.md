# 网络基础：数据如何在网络中传输

## 本章目标

理解网络游戏的数据传输原理，学会选择合适的网络协议。

## 为什么单机游戏不需要学网络？

单机游戏的所有数据都在你自己的电脑上：

```
┌─────────────────────────────────────┐
│            你的电脑                  │
│  ┌─────────┐      ┌─────────┐       │
│  │ 游戏逻辑 │ ───► │ 显示屏幕 │       │
│  └─────────┘      └─────────┘       │
│       │                             │
│       ▼                             │
│  ┌─────────┐                        │
│  │ 存档文件 │  ← 数据直接存硬盘       │
│  └─────────┘                        │
└─────────────────────────────────────┘
        所有数据都在这台电脑里
```

多人游戏需要把数据传给其他玩家：

```
┌─────────────┐         网络          ┌─────────────┐
│  你的电脑   │ ◄───────────────────► │  对面电脑   │
│  (客户端A)  │      数据传输         │  (客户端B)  │
└─────────────┘                       └─────────────┘
        ↑                                   ↑
        │                                   │
        └─────────── 中间经过很多设备 ───────┘
                   路由器、交换机、光缆...
```

## 网络协议：数据传输的规则

想象你要寄一封信：

```
信的内容 = 游戏数据（比如：我移动到了坐标 100, 100）
信封格式 = 网络协议（规定数据怎么打包）
邮寄方式 = TCP 或 UDP（可靠但慢，或快速但可能丢）
```

### TCP vs UDP

这是网络游戏最重要的选择：

#### TCP（Transmission Control Protocol）

```
特点：可靠、有序、有连接

发送方                     接收方
  │                          │
  │─────── 数据包 1 ────────►│
  │◄────── 确认收到 1 ───────│
  │─────── 数据包 2 ────────►│
  │     (网络拥堵，丢了)      │
  │◄────── 重发包 2 ─────────│  ← TCP 自动重传
  │─────── 数据包 2 ────────►│
  │◄────── 确认收到 2 ───────│
  │─────── 数据包 3 ────────►│
  │◄────── 确认收到 3 ───────│

优点：数据一定完整到达
缺点：慢（每次都要确认）
```

**适用场景**：
- 聊天消息（不能丢字）
- 账号登录（必须可靠）
- 物品交易（不能出错）
- 商城购买（金额必须准确）

#### UDP（User Datagram Protocol）

```
特点：不可靠、无序、无连接

发送方                     接收方
  │                          │
  │─────── 数据包 1 ────────►│
  │─────── 数据包 2 ────────►│
  │     (丢了，不管)          │  ← UDP 不重传
  │─────── 数据包 3 ────────►│
  │                          │

优点：快（不用等待确认）
缺点：可能丢包、乱序
```

**适用场景**：
- 玩家位置（丢了就丢了，反正马上又有新位置）
- 实时战斗（延迟比准确更重要）
- 语音通话（偶尔卡一下没关系）

### 为什么游戏常用 UDP？

看这个例子：

```
玩家位置每秒更新 20 次：
帧1: 位置 (100, 100)
帧2: 位置 (105, 100)
帧3: 位置 (110, 100)  ← 这个包丢了
帧4: 位置 (115, 100)
帧5: 位置 (120, 100)

如果用 TCP：
帧1 到达 → 帧2 到达 → 帧3 丢了 → 等待重传... → 玩家卡住
帧3 重传到达 → 帧4 才能处理 → 帧5 处理
结果：玩家感觉"卡顿"

如果用 UDP：
帧1 到达 → 帧2 到达 → 帧3 丢了 → 直接处理帧4 → 帧5
结果：玩家从 (105, 100) 跳到 (115, 100)，稍微不连贯但流畅
```

**核心原因**：旧的位置数据已经没用了，不需要重传。

### WebSocket：Web 游戏的选择

如果你做的是浏览器游戏（比如 RPG Maker MV 导出的网页版），TCP 和 UDP 都不能直接用。浏览器提供了 **WebSocket**：

```
特点：
- 基于 TCP（可靠）
- 全双工（双向通信）
- 浏览器原生支持
- 适合 Web 游戏

客户端（浏览器）                服务器
     │                           │
     │────── HTTP 握手 ─────────►│  ← 先用 HTTP 建立连接
     │◄───── 升级为 WebSocket ───│
     │                           │
     │◄═════════ 双向通道 ═══════►│  ← 之后可以双向发消息
     │                           │
```

## 协议选择指南

| 场景 | 推荐协议 | 原因 |
|------|---------|------|
| **Web 网页游戏** | WebSocket | 浏览器唯一选择 |
| **PC 客户端游戏** | UDP + TCP 混合 | 位置用 UDP，交易用 TCP |
| **回合制游戏** | TCP / WebSocket | 不需要实时，可靠性重要 |
| **动作/射击游戏** | UDP | 延迟敏感 |
| **聊天系统** | TCP / WebSocket | 消息不能丢 |

## WebSocket 实战：第一个连接

### 服务器端代码

```javascript
// ============================================
// server.js - 最简单的 WebSocket 服务器
// ============================================
// 为什么用 ws 库？因为 Node.js 原生不支持 WebSocket
// 安装：npm install ws

const WebSocket = require('ws');

// 创建 WebSocket 服务器，监听 8080 端口
// 为什么选 8080？这是常用的 Web 服务端口，不冲突
const wss = new WebSocket.Server({ port: 8080 });

console.log('服务器启动在 ws://localhost:8080');

// 当有客户端连接时
wss.on('connection', function(ws, req) {
    // ws: 这个客户端的连接对象
    // req: HTTP 请求对象（包含客户端 IP 等）
    
    console.log('有客户端连接！来自:', req.socket.remoteAddress);
    
    // 发送欢迎消息给这个客户端
    // 为什么用 JSON？因为网络只能传字符串，JSON 是标准格式
    ws.send(JSON.stringify({
        type: 'welcome',
        message: '欢迎连接到游戏服务器！',
        time: Date.now()
    }));
    
    // 当收到客户端消息时
    ws.on('message', function(data) {
        // data 是 Buffer 或字符串，需要转换
        console.log('收到消息:', data.toString());
        
        try {
            // 尝试解析 JSON
            var msg = JSON.parse(data);
            
            // 根据消息类型处理
            switch(msg.type) {
                case 'chat':
                    // 聊天消息，原样返回
                    ws.send(JSON.stringify({
                        type: 'chat',
                        message: '服务器收到: ' + msg.message
                    }));
                    break;
                    
                case 'ping':
                    // 心跳检测，用于检测连接是否正常
                    ws.send(JSON.stringify({
                        type: 'pong',
                        time: Date.now()
                    }));
                    break;
            }
        } catch(e) {
            // JSON 解析失败
            ws.send(JSON.stringify({
                type: 'error',
                message: '消息格式错误'
            }));
        }
    });
    
    // 当连接关闭时
    ws.on('close', function() {
        console.log('客户端断开连接');
    });
    
    // 当发生错误时
    ws.on('error', function(error) {
        console.log('连接错误:', error.message);
    });
});
```

### 客户端代码（浏览器）

```html
<!-- ============================================
index.html - 最简单的 WebSocket 客户端
============================================ -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>WebSocket 测试</title>
</head>
<body>
    <h1>WebSocket 连接测试</h1>
    <div id="log" style="border: 1px solid #ccc; padding: 10px; height: 300px; overflow-y: auto;"></div>
    <input id="input" type="text" placeholder="输入消息" style="width: 300px;">
    <button onclick="send()">发送</button>
    
    <script>
        // 日志显示函数
        function log(msg) {
            var div = document.getElementById('log');
            div.innerHTML += '<div>' + msg + '</div>';
            div.scrollTop = div.scrollHeight;
        }
        
        // 创建 WebSocket 连接
        // ws:// 是 WebSocket 协议，wss:// 是加密版本
        var socket = new WebSocket('ws://localhost:8080');
        
        // ========== 连接状态事件 ==========
        
        // 连接成功
        socket.onopen = function() {
            log('✅ 连接成功！');
        };
        
        // 连接关闭
        socket.onclose = function(event) {
            // event.code: 关闭代码
            // event.reason: 关闭原因
            log('❌ 连接关闭: ' + event.code + ' ' + event.reason);
        };
        
        // 连接错误
        socket.onerror = function(error) {
            log('⚠️ 连接错误');
        };
        
        // ========== 数据事件 ==========
        
        // 收到消息
        socket.onmessage = function(event) {
            // event.data 是收到的数据（字符串）
            try {
                var msg = JSON.parse(event.data);
                log('📥 收到: ' + JSON.stringify(msg));
            } catch(e) {
                log('📥 收到: ' + event.data);
            }
        };
        
        // ========== 发送消息 ==========
        
        function send() {
            var input = document.getElementById('input');
            var message = input.value;
            
            if (!message) return;
            
            // 检查连接状态
            // readyState: 0=连接中, 1=已连接, 2=关闭中, 3=已关闭
            if (socket.readyState !== WebSocket.OPEN) {
                log('⚠️ 连接未打开，无法发送');
                return;
            }
            
            // 发送消息
            socket.send(JSON.stringify({
                type: 'chat',
                message: message
            }));
            
            log('📤 发送: ' + message);
            input.value = '';
        }
        
        // 回车发送
        document.getElementById('input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                send();
            }
        });
    </script>
</body>
</html>
```

## 消息格式设计

网络传输的数据需要统一格式：

### 为什么用 JSON？

```javascript
// 原始数据
var position = { x: 100, y: 200 };

// 转成 JSON 字符串
var json = JSON.stringify(position);  // '{"x":100,"y":200}'

// 网络传输这个字符串
socket.send(json);

// 接收方解析
var data = JSON.parse(json);  // { x: 100, y: 200 }
```

优点：
- 人类可读，调试方便
- JavaScript 原生支持
- 跨语言通用

缺点：
- 体积大（每个键名都传输）
- 解析慢（大量数据时）

### 消息类型设计

```javascript
// 好的设计：统一的消息格式
{
    "type": "playerMove",     // 消息类型，必须有
    "playerId": "abc123",     // 谁发的
    "payload": {              // 具体数据
        "x": 100,
        "y": 200,
        "direction": "right"
    },
    "timestamp": 1700000000000  // 什么时候发的
}

// 处理时
switch(message.type) {
    case 'playerMove':
        handleMove(message.playerId, message.payload);
        break;
    case 'playerAttack':
        handleAttack(message.playerId, message.payload);
        break;
    // ...
}
```

### 二进制协议（进阶）

对于性能要求高的游戏，可以用二进制：

```javascript
// JSON 格式：约 50 字节
{"type":"move","x":100,"y":200}

// 二进制格式：约 10 字节
// [类型1字节][x坐标4字节][y坐标4字节]
// Buffer: [0x01, 0x00, 0x00, 0x00, 0x64, 0x00, 0x00, 0x00, 0xC8]

// 编码函数
function encodePosition(x, y) {
    var buffer = Buffer.alloc(9);
    buffer.writeUInt8(0x01, 0);      // 类型：移动
    buffer.writeInt32LE(x, 1);       // X 坐标
    buffer.writeInt32LE(y, 5);       // Y 坐标
    return buffer;
}

// 解码函数
function decodePosition(buffer) {
    return {
        type: buffer.readUInt8(0),
        x: buffer.readInt32LE(1),
        y: buffer.readInt32LE(5)
    };
}
```

**什么时候用二进制？**
- 玩家数量 > 1000
- 每秒消息 > 100 条/人
- 带宽成本敏感

## 心跳机制

网络连接可能静默断开（比如拔网线），需要心跳检测：

```javascript
// ============================================
// 心跳检测：确保连接存活
// ============================================

// 客户端
var lastPongTime = Date.now();
var heartbeatInterval;

function startHeartbeat() {
    // 每 30 秒发一次 ping
    heartbeatInterval = setInterval(function() {
        if (socket.readyState === WebSocket.OPEN) {
            socket.send(JSON.stringify({ type: 'ping' }));
        }
    }, 30000);
    
    // 检查 pong 响应
    setInterval(function() {
        if (Date.now() - lastPongTime > 60000) {
            // 60 秒没收到 pong，认为断线
            console.log('心跳超时，重连...');
            reconnect();
        }
    }, 10000);
}

// 收到 pong 时更新时间
socket.onmessage = function(event) {
    var msg = JSON.parse(event.data);
    if (msg.type === 'pong') {
        lastPongTime = Date.now();
    }
};

// 服务器端
wss.on('connection', function(ws) {
    ws.isAlive = true;
    
    ws.on('message', function(data) {
        var msg = JSON.parse(data);
        if (msg.type === 'ping') {
            ws.isAlive = true;
            ws.send(JSON.stringify({ type: 'pong', time: Date.now() }));
        }
    });
});

// 定期检查所有连接
setInterval(function() {
    wss.clients.forEach(function(ws) {
        if (ws.isAlive === false) {
            return ws.terminate();  // 断开死连接
        }
        ws.isAlive = false;  // 标记为待检测
    });
}, 30000);
```

## 常见问题

### Q1: 为什么连接不上？

```
检查清单：
□ 服务器是否启动？→ node server.js
□ 端口是否被占用？→ netstat -an | findstr 8080
□ 防火墙是否开放？→ 关闭防火墙测试
□ 地址是否正确？→ ws://localhost:8080 或 ws://服务器IP:8080
```

### Q2: 为什么消息发不出去？

```javascript
// 检查连接状态
console.log('readyState:', socket.readyState);
// 0: CONNECTING - 正在连接
// 1: OPEN - 已连接，可以发送
// 2: CLOSING - 正在关闭
// 3: CLOSED - 已关闭

// 正确的发送方式
if (socket.readyState === WebSocket.OPEN) {
    socket.send(message);
} else {
    console.log('连接未准备好');
}
```

### Q3: 断线怎么自动重连？

```javascript
var socket;
var reconnectAttempts = 0;
var maxReconnectAttempts = 5;

function connect() {
    socket = new WebSocket('ws://localhost:8080');
    
    socket.onopen = function() {
        console.log('连接成功');
        reconnectAttempts = 0;  // 重置重连计数
    };
    
    socket.onclose = function() {
        console.log('连接断开');
        
        // 尝试重连
        if (reconnectAttempts < maxReconnectAttempts) {
            reconnectAttempts++;
            var delay = reconnectAttempts * 1000;  // 递增延迟
            console.log(delay/1000 + '秒后重连...');
            setTimeout(connect, delay);
        }
    };
}

connect();
```

## 本章小结

| 概念 | 要点 |
|------|------|
| **TCP** | 可靠但慢，适合聊天、交易 |
| **UDP** | 快但可能丢包，适合位置同步 |
| **WebSocket** | Web 游戏首选 |
| **消息格式** | 统一用 type + payload 结构 |
| **心跳** | 检测连接是否存活 |

## 下一章

[服务器架构](02-server-architecture.md) - 学习如何设计 MMO 的服务器结构。
