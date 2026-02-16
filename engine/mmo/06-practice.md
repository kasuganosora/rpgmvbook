# 实战案例：完整的聊天+移动 MMO

## 本章目标

实现一个可运行的最小 MMO Demo。

## 服务器代码

```javascript
// server.js
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

var players = new Map();

wss.on('connection', function(ws) {
    var id = Date.now().toString(36) + Math.random().toString(36).substr(2, 5);
    
    var player = {
        id: id,
        ws: ws,
        name: 'Player' + id.substr(0, 4),
        x: Math.random() * 800,
        y: Math.random() * 600,
        color: '#' + Math.floor(Math.random()*16777215).toString(16)
    };
    
    players.set(id, player);
    
    // 发送初始化数据
    send(ws, { type: 'init', id: id, x: player.x, y: player.y });
    
    // 广播加入
    broadcast({ type: 'join', id: id, name: player.name, x: player.x, y: player.y, color: player.color }, id);
    
    ws.on('message', function(msg) {
        var data = JSON.parse(msg);
        
        if (data.type === 'move') {
            player.x = Math.max(0, Math.min(800, data.x));
            player.y = Math.max(0, Math.min(600, data.y));
            broadcast({ type: 'move', id: id, x: player.x, y: player.y }, id);
        }
        
        if (data.type === 'chat') {
            broadcast({ type: 'chat', name: player.name, message: data.message });
        }
    });
    
    ws.on('close', function() {
        players.delete(id);
        broadcast({ type: 'leave', id: id });
    });
});

function send(ws, data) {
    if (ws.readyState === 1) ws.send(JSON.stringify(data));
}

function broadcast(data, excludeId) {
    players.forEach((p, id) => {
        if (id !== excludeId) send(p.ws, data);
    });
}

console.log('MMO 服务器启动在 ws://localhost:8080');
```

## 客户端代码

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>MMO Demo</title>
    <style>
        body { margin: 0; background: #1a1a2e; }
        canvas { display: block; margin: 20px auto; border: 2px solid #fff; }
        #chat { width: 804px; height: 100px; margin: 0 auto; background: rgba(0,0,0,0.8); color: #fff; overflow-y: auto; }
        #input { width: 800px; margin: 10px auto; display: block; padding: 5px; }
    </style>
</head>
<body>
    <canvas id="game" width="800" height="600"></canvas>
    <div id="chat"></div>
    <input id="input" placeholder="输入消息按回车发送">
    
    <script>
        var canvas = document.getElementById('game');
        var ctx = canvas.getContext('2d');
        var socket = new WebSocket('ws://localhost:8080');
        
        var myId, myPlayer = { x: 400, y: 300 }, players = {}, keys = {};
        
        document.addEventListener('keydown', e => keys[e.key] = true);
        document.addEventListener('keyup', e => keys[e.key] = false);
        
        document.getElementById('input').addEventListener('keypress', e => {
            if (e.key === 'Enter') {
                socket.send(JSON.stringify({ type: 'chat', message: e.target.value }));
                e.target.value = '';
            }
        });
        
        socket.onmessage = function(e) {
            var d = JSON.parse(e.data);
            if (d.type === 'init') { myId = d.id; myPlayer.x = d.x; myPlayer.y = d.y; }
            if (d.type === 'join') { players[d.id] = d; log(d.name + ' 加入'); }
            if (d.type === 'leave') { delete players[d.id]; log('有人离开'); }
            if (d.type === 'move' && players[d.id]) { players[d.id].x = d.x; players[d.id].y = d.y; }
            if (d.type === 'chat') log(d.name + ': ' + d.message);
        };
        
        function log(msg) {
            var div = document.getElementById('chat');
            div.innerHTML += '<div>' + msg + '</div>';
            div.scrollTop = div.scrollHeight;
        }
        
        function loop() {
            var speed = 5;
            if (keys['ArrowUp'] || keys['w']) myPlayer.y -= speed;
            if (keys['ArrowDown'] || keys['s']) myPlayer.y += speed;
            if (keys['ArrowLeft'] || keys['a']) myPlayer.x -= speed;
            if (keys['ArrowRight'] || keys['d']) myPlayer.x += speed;
            
            myPlayer.x = Math.max(0, Math.min(800, myPlayer.x));
            myPlayer.y = Math.max(0, Math.min(600, myPlayer.y));
            
            if (socket.readyState === 1) {
                socket.send(JSON.stringify({ type: 'move', x: myPlayer.x, y: myPlayer.y }));
            }
            
            ctx.fillStyle = '#1a1a2e';
            ctx.fillRect(0, 0, 800, 600);
            
            for (var id in players) {
                var p = players[id];
                ctx.fillStyle = p.color;
                ctx.fillRect(p.x - 16, p.y - 16, 32, 32);
                ctx.fillStyle = '#fff';
                ctx.fillText(p.name, p.x - 20, p.y - 20);
            }
            
            ctx.fillStyle = '#ff0';
            ctx.fillRect(myPlayer.x - 16, myPlayer.y - 16, 32, 32);
            
            requestAnimationFrame(loop);
        }
        
        loop();
    </script>
</body>
</html>
```

## 运行方法

```bash
# 1. 安装依赖
npm install ws

# 2. 启动服务器
node server.js

# 3. 用浏览器打开 client.html
# 4. 打开多个浏览器标签页测试多人
```

## 本章小结

这个 Demo 包含了 MMO 的核心要素：
- WebSocket 连接
- 玩家移动同步
- 聊天功能
- 玩家加入/离开广播

你可以在此基础上扩展更多功能！
