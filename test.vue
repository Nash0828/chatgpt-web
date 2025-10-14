// server.js
const express = require('express');
const http = require('http');
const WebSocket = require('ws');

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// 静态文件服务（可选）
app.use(express.static('public'));

// WebSocket 连接处理
wss.on('connection', (ws, request) => {
  console.log('新的 WebSocket 连接已建立');
  
  // 获取客户端 IP
  const clientIP = request.socket.remoteAddress;
  console.log(`客户端 IP: ${clientIP}`);

  // 向客户端发送欢迎消息
  ws.send(JSON.stringify({
    type: 'welcome',
    message: '连接 WebSocket 服务器成功！',
    timestamp: new Date().toISOString()
  }));

  // 接收客户端消息
  ws.on('message', (data) => {
    try {
      const message = data.toString();
      console.log('收到客户端消息:', message);

      // 解析 JSON 消息
      let parsedMessage;
      try {
        parsedMessage = JSON.parse(message);
      } catch {
        parsedMessage = { type: 'text', content: message };
      }

      // 处理不同类型的消息
      switch (parsedMessage.type) {
        case 'chat':
          // 广播聊天消息给所有客户端
          wss.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
              client.send(JSON.stringify({
                type: 'chat',
                user: parsedMessage.user || '匿名用户',
                message: parsedMessage.message,
                timestamp: new Date().toISOString()
              }));
            }
          });
          break;

        case 'ping':
          // 响应 ping 消息
          ws.send(JSON.stringify({
            type: 'pong',
            timestamp: new Date().toISOString()
          }));
          break;

        default:
          // 回声测试
          ws.send(JSON.stringify({
            type: 'echo',
            original: parsedMessage,
            timestamp: new Date().toISOString()
          }));
      }
    } catch (error) {
      console.error('处理消息时出错:', error);
    }
  });

  // 定时发送消息（可选）
  const interval = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        type: 'server_time',
        timestamp: new Date().toISOString(),
        message: '这是来自服务器的定时消息'
      }));
    }
  }, 30000); // 每30秒发送一次

  // 连接关闭处理
  ws.on('close', () => {
    console.log('WebSocket 连接已关闭');
    clearInterval(interval);
    
    // 广播用户离开消息
    wss.clients.forEach((client) => {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({
          type: 'user_left',
          message: '有用户离开了聊天室',
          timestamp: new Date().toISOString()
        }));
      }
    });
  });

  // 错误处理
  ws.on('error', (error) => {
    console.error('WebSocket 错误:', error);
  });
});

// 广播消息给所有客户端的方法
function broadcast(message) {
  wss.clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(message));
    }
  });
}

// Express 路由
app.get('/', (req, res) => {
  res.send(`
    <html>
      <head><title>WebSocket 服务器</title></head>
      <body>
        <h1>WebSocket 服务器运行中</h1>
        <p>当前连接数: ${wss.clients.size}</p>
        <p>使用 WebSocket 客户端连接 ws://localhost:3000</p>
      </body>
    </html>
  `);
});

// 获取连接信息的 API
app.get('/api/connections', (req, res) => {
  res.json({
    connectionCount: wss.clients.size,
    status: 'running',
    timestamp: new Date().toISOString()
  });
});

// 广播消息的 API
app.post('/api/broadcast', express.json(), (req, res) => {
  const { message } = req.body;
  if (message) {
    broadcast({
      type: 'broadcast',
      message: message,
      from: 'server',
      timestamp: new Date().toISOString()
    });
    res.json({ success: true, message: '广播发送成功' });
  } else {
    res.status(400).json({ success: false, message: '消息内容不能为空' });
  }
});

// 启动服务器
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`服务器运行在 http://localhost:${PORT}`);
  console.log(`WebSocket 服务运行在 ws://localhost:${PORT}`);
});