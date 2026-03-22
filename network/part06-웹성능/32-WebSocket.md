# Chapter 32. WebSocket과 실시간 통신

## 32.1 WebSocket이란?

### 정의

```
WebSocket:
양방향 실시간 통신 프로토콜

비유: 전화 vs 문자
┌────────────────────────────────┐
│ HTTP: 문자 (요청/응답)         │
│ - 질문 보내기 → 답변 받기      │
│ - 매번 연결/종료               │
│                                │
│ WebSocket: 전화 (연결 유지)    │
│ - 통화 중 계속 대화            │
│ - 연결 한 번, 양방향 통신      │
└────────────────────────────────┘
```

### HTTP vs WebSocket

```
HTTP (폴링):
Client: 새 메시지 있어?
Server: 없어
(1초 후)
Client: 새 메시지 있어?
Server: 없어
(1초 후)
Client: 새 메시지 있어?
Server: 있어! "안녕"

→ 비효율적 (계속 요청)

WebSocket:
Client: (연결)
Server: (연결 수락)
[연결 유지]
Server: "새 메시지: 안녕"
Client: "읽음"
Server: "읽음 확인"

→ 효율적 (실시간)
```

---

## 32.2 WebSocket 동작 원리

### Handshake

```
1. HTTP Upgrade 요청
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

2. 서버 응답
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. WebSocket 연결 성립
[양방향 통신 시작]
```

### Frame 구조

```
WebSocket Frame:
┌────────────────────────────┐
│ FIN (1bit): 마지막 프레임  │
│ Opcode (4bit): 타입        │
│ - 0x1: 텍스트              │
│ - 0x2: 바이너리            │
│ - 0x8: 연결 종료           │
│ - 0x9: Ping                │
│ - 0xA: Pong                │
│ Mask (1bit): 마스킹 여부   │
│ Payload: 실제 데이터       │
└────────────────────────────┘
```

---

## 32.3 WebSocket 구현 (ws 라이브러리)

### 기본 서버

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('클라이언트 연결:', req.socket.remoteAddress);

  // 메시지 수신
  ws.on('message', (message) => {
    console.log('수신:', message.toString());

    // 에코 (받은 메시지 그대로 전송)
    ws.send(`Echo: ${message}`);
  });

  // 에러 처리
  ws.on('error', (error) => {
    console.error('WebSocket 에러:', error);
  });

  // 연결 종료
  ws.on('close', (code, reason) => {
    console.log('연결 종료:', code, reason.toString());
  });

  // 환영 메시지
  ws.send('Welcome to WebSocket server!');
});

console.log('WebSocket 서버 시작: ws://localhost:8080');
```

### 클라이언트

```javascript
// 브라우저
const ws = new WebSocket('ws://localhost:8080');

// 연결 성공
ws.onopen = () => {
  console.log('연결됨');
  ws.send('안녕하세요!');
};

// 메시지 수신
ws.onmessage = (event) => {
  console.log('수신:', event.data);
};

// 에러
ws.onerror = (error) => {
  console.error('에러:', error);
};

// 연결 종료
ws.onclose = (event) => {
  console.log('연결 종료:', event.code, event.reason);
};

// 메시지 전송
function sendMessage(msg) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(msg);
  } else {
    console.error('연결 안 됨');
  }
}
```

---

## 32.4 채팅 애플리케이션

### 서버

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

// 연결된 클라이언트 목록
const clients = new Map();

wss.on('connection', (ws, req) => {
  const clientId = Date.now();
  clients.set(clientId, {
    ws,
    username: null
  });

  console.log(`클라이언트 ${clientId} 연결`);

  ws.on('message', (message) => {
    try {
      const data = JSON.parse(message);

      switch (data.type) {
        case 'join':
          handleJoin(clientId, data.username);
          break;

        case 'message':
          handleMessage(clientId, data.text);
          break;

        case 'typing':
          handleTyping(clientId);
          break;
      }
    } catch (err) {
      console.error('메시지 파싱 에러:', err);
    }
  });

  ws.on('close', () => {
    handleLeave(clientId);
  });
});

function handleJoin(clientId, username) {
  const client = clients.get(clientId);
  client.username = username;

  // 입장 알림 (모든 클라이언트에게)
  broadcast({
    type: 'system',
    text: `${username}님이 입장했습니다.`
  });

  // 현재 사용자 목록 전송
  const userList = Array.from(clients.values())
    .map(c => c.username)
    .filter(Boolean);

  client.ws.send(JSON.stringify({
    type: 'userList',
    users: userList
  }));
}

function handleMessage(clientId, text) {
  const client = clients.get(clientId);

  broadcast({
    type: 'message',
    username: client.username,
    text,
    timestamp: new Date().toISOString()
  });
}

function handleTyping(clientId) {
  const client = clients.get(clientId);

  // 자신 제외하고 전송
  broadcast({
    type: 'typing',
    username: client.username
  }, clientId);
}

function handleLeave(clientId) {
  const client = clients.get(clientId);

  if (client.username) {
    broadcast({
      type: 'system',
      text: `${client.username}님이 퇴장했습니다.`
    });
  }

  clients.delete(clientId);
}

// 모든 클라이언트에게 전송
function broadcast(message, excludeId = null) {
  const data = JSON.stringify(message);

  clients.forEach((client, id) => {
    if (id !== excludeId && client.ws.readyState === WebSocket.OPEN) {
      client.ws.send(data);
    }
  });
}

console.log('채팅 서버 시작: ws://localhost:8080');
```

### 클라이언트

```html
<!DOCTYPE html>
<html>
<head>
  <title>WebSocket Chat</title>
</head>
<body>
  <div id="chat">
    <div id="messages"></div>
    <div id="typing"></div>
    <input type="text" id="username" placeholder="이름">
    <button onclick="join()">입장</button>
    <input type="text" id="message" placeholder="메시지" disabled>
    <button onclick="send()" disabled>전송</button>
  </div>

  <script>
    let ws;
    let username;

    function join() {
      username = document.getElementById('username').value;
      if (!username) return;

      ws = new WebSocket('ws://localhost:8080');

      ws.onopen = () => {
        ws.send(JSON.stringify({
          type: 'join',
          username
        }));

        document.getElementById('message').disabled = false;
        document.querySelector('button[onclick="send()"]').disabled = false;
      };

      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);

        switch (data.type) {
          case 'message':
            addMessage(`${data.username}: ${data.text}`);
            break;

          case 'system':
            addMessage(`[시스템] ${data.text}`, 'system');
            break;

          case 'typing':
            showTyping(`${data.username}님이 입력 중...`);
            break;

          case 'userList':
            console.log('현재 사용자:', data.users);
            break;
        }
      };
    }

    function send() {
      const input = document.getElementById('message');
      const text = input.value;

      if (!text) return;

      ws.send(JSON.stringify({
        type: 'message',
        text
      }));

      input.value = '';
    }

    function addMessage(text, className = '') {
      const div = document.createElement('div');
      div.textContent = text;
      div.className = className;
      document.getElementById('messages').appendChild(div);
    }

    function showTyping(text) {
      const div = document.getElementById('typing');
      div.textContent = text;
      setTimeout(() => div.textContent = '', 2000);
    }

    // 입력 중 알림
    let typingTimer;
    document.getElementById('message').addEventListener('input', () => {
      clearTimeout(typingTimer);
      typingTimer = setTimeout(() => {
        if (ws && ws.readyState === WebSocket.OPEN) {
          ws.send(JSON.stringify({ type: 'typing' }));
        }
      }, 300);
    });
  </script>
</body>
</html>
```

---

## 32.5 Socket.IO

### Socket.IO vs ws

```
Socket.IO:
✅ 자동 재연결
✅ Room/Namespace 지원
✅ Fallback (Long Polling)
✅ Binary 지원
✅ 브로드캐스팅 간편

ws:
✅ 가벼움
✅ WebSocket 표준
❌ 직접 구현 필요
```

### Socket.IO 서버

```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.static('public'));

// 연결
io.on('connection', (socket) => {
  console.log('클라이언트 연결:', socket.id);

  // 입장
  socket.on('join', (username) => {
    socket.username = username;

    // 전체 브로드캐스트
    io.emit('system', `${username}님이 입장했습니다.`);

    // 현재 사용자 수
    io.emit('userCount', io.engine.clientsCount);
  });

  // 메시지
  socket.on('message', (text) => {
    io.emit('message', {
      username: socket.username,
      text,
      timestamp: new Date()
    });
  });

  // 입력 중
  socket.on('typing', () => {
    socket.broadcast.emit('typing', socket.username);
  });

  // 연결 종료
  socket.on('disconnect', () => {
    if (socket.username) {
      io.emit('system', `${socket.username}님이 퇴장했습니다.`);
      io.emit('userCount', io.engine.clientsCount);
    }
  });
});

server.listen(3000, () => {
  console.log('서버 시작: http://localhost:3000');
});
```

### Socket.IO 클라이언트

```html
<!DOCTYPE html>
<html>
<head>
  <title>Socket.IO Chat</title>
  <script src="/socket.io/socket.io.js"></script>
</head>
<body>
  <div id="chat">
    <div id="messages"></div>
    <div id="userCount"></div>
    <input type="text" id="message" placeholder="메시지">
    <button onclick="send()">전송</button>
  </div>

  <script>
    const socket = io();

    // 입장
    const username = prompt('이름을 입력하세요:');
    socket.emit('join', username);

    // 메시지 수신
    socket.on('message', (data) => {
      addMessage(`${data.username}: ${data.text}`);
    });

    // 시스템 메시지
    socket.on('system', (text) => {
      addMessage(`[${text}]`, 'system');
    });

    // 사용자 수
    socket.on('userCount', (count) => {
      document.getElementById('userCount').textContent = `접속자: ${count}명`;
    });

    // 메시지 전송
    function send() {
      const input = document.getElementById('message');
      const text = input.value;

      if (!text) return;

      socket.emit('message', text);
      input.value = '';
    }

    function addMessage(text, className = '') {
      const div = document.createElement('div');
      div.textContent = text;
      div.className = className;
      document.getElementById('messages').appendChild(div);
    }
  </script>
</body>
</html>
```

---

## 32.6 Room과 Namespace

### Room (그룹 채팅)

```javascript
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  // 방 입장
  socket.on('joinRoom', (roomName) => {
    socket.join(roomName);
    socket.room = roomName;

    // 방 인원에게만 알림
    io.to(roomName).emit('system', `${socket.username}님이 입장했습니다.`);

    // 방 목록
    const rooms = Array.from(socket.rooms);
    socket.emit('rooms', rooms);
  });

  // 방 퇴장
  socket.on('leaveRoom', (roomName) => {
    socket.leave(roomName);
    io.to(roomName).emit('system', `${socket.username}님이 퇴장했습니다.`);
  });

  // 방에만 메시지 전송
  socket.on('message', (text) => {
    io.to(socket.room).emit('message', {
      username: socket.username,
      text
    });
  });

  // 자신 제외하고 전송
  socket.on('typing', () => {
    socket.to(socket.room).emit('typing', socket.username);
  });
});
```

### Namespace (앱 분리)

```javascript
// 기본 namespace
const io = require('socket.io')(server);

// 채팅 namespace
const chat = io.of('/chat');
chat.on('connection', (socket) => {
  socket.on('message', (text) => {
    chat.emit('message', { text });
  });
});

// 알림 namespace
const notifications = io.of('/notifications');
notifications.on('connection', (socket) => {
  socket.on('subscribe', (userId) => {
    socket.join(`user:${userId}`);
  });
});

// 알림 전송
function sendNotification(userId, message) {
  notifications.to(`user:${userId}`).emit('notification', message);
}
```

```javascript
// 클라이언트
const chatSocket = io('/chat');
const notificationSocket = io('/notifications');

chatSocket.on('message', (data) => {
  console.log('채팅:', data);
});

notificationSocket.on('notification', (data) => {
  console.log('알림:', data);
});
```

---

## 32.7 인증 및 보안

### JWT 인증

```javascript
const io = require('socket.io')(server);
const jwt = require('jsonwebtoken');

// 미들웨어
io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error('인증 필요'));
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded.userId;
    socket.username = decoded.username;
    next();
  } catch (err) {
    next(new Error('Invalid token'));
  }
});

io.on('connection', (socket) => {
  console.log(`${socket.username} 연결됨`);

  // 인증된 사용자만 접근 가능
  socket.on('message', (text) => {
    io.emit('message', {
      userId: socket.userId,
      username: socket.username,
      text
    });
  });
});
```

```javascript
// 클라이언트
const token = localStorage.getItem('token');

const socket = io({
  auth: {
    token
  }
});

socket.on('connect_error', (err) => {
  console.error('연결 실패:', err.message);
  // 재로그인 유도
});
```

### CORS 설정

```javascript
const io = require('socket.io')(server, {
  cors: {
    origin: 'https://example.com',
    methods: ['GET', 'POST'],
    credentials: true
  }
});
```

---

## 32.8 확장성 (Scaling)

### Redis Adapter

```javascript
// 다중 서버 환경에서 Socket.IO 동기화
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ host: 'localhost', port: 6379 });
const subClient = pubClient.duplicate();

Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
  io.adapter(createAdapter(pubClient, subClient));

  io.on('connection', (socket) => {
    // 모든 서버에 브로드캐스트됨
    io.emit('message', 'Hello from any server');
  });
});

// 서버 1에서 emit → 서버 2, 3도 수신
// → 모든 클라이언트에게 전달
```

### Sticky Session

```nginx
# nginx.conf
upstream socket_nodes {
    ip_hash;  # 같은 IP는 같은 서버로

    server 192.168.1.101:3000;
    server 192.168.1.102:3000;
    server 192.168.1.103:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://socket_nodes;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

---

## 32.9 실전 예제

### 실시간 알림

```javascript
// 서버
const io = require('socket.io')(server);

io.use(authenticate);  // JWT 인증

io.on('connection', (socket) => {
  // 사용자별 Room 입장
  socket.join(`user:${socket.userId}`);

  console.log(`User ${socket.userId} 연결`);
});

// 알림 전송 함수
async function sendNotification(userId, notification) {
  // DB 저장
  await Notification.create({
    userId,
    type: notification.type,
    message: notification.message
  });

  // Socket.IO로 실시간 전송
  io.to(`user:${userId}`).emit('notification', notification);
}

// 사용 예
app.post('/api/posts/:id/like', authenticate, async (req, res) => {
  const post = await Post.findByPk(req.params.id);
  await Like.create({ userId: req.user.id, postId: post.id });

  // 게시글 작성자에게 알림
  await sendNotification(post.userId, {
    type: 'like',
    message: `${req.user.username}님이 좋아요를 눌렀습니다.`,
    postId: post.id
  });

  res.json({ success: true });
});
```

```javascript
// 클라이언트
const socket = io({
  auth: { token: localStorage.getItem('token') }
});

socket.on('notification', (notification) => {
  // 알림 UI 표시
  showNotification(notification.message);

  // 뱃지 업데이트
  updateBadgeCount();
});

function showNotification(message) {
  const div = document.createElement('div');
  div.className = 'notification';
  div.textContent = message;
  document.body.appendChild(div);

  setTimeout(() => div.remove(), 3000);
}
```

### 실시간 협업 (Google Docs 스타일)

```javascript
// 서버
io.on('connection', (socket) => {
  socket.on('joinDocument', (docId) => {
    socket.join(`doc:${docId}`);

    // 현재 편집자 목록
    const editors = getEditorsInDocument(docId);
    socket.emit('editors', editors);
  });

  socket.on('edit', (data) => {
    const { docId, position, text, action } = data;

    // DB 저장
    saveEdit(docId, data);

    // 다른 사용자에게 전송 (자신 제외)
    socket.to(`doc:${docId}`).emit('edit', {
      userId: socket.userId,
      username: socket.username,
      position,
      text,
      action
    });
  });

  socket.on('cursor', (data) => {
    const { docId, position } = data;

    // 커서 위치 공유
    socket.to(`doc:${docId}`).emit('cursor', {
      userId: socket.userId,
      username: socket.username,
      position
    });
  });
});
```

---

## 핵심 요약

### WebSocket
```
양방향 실시간 통신
Handshake (HTTP Upgrade)
Frame 기반 프로토콜
```

### Socket.IO
```
자동 재연결
Room/Namespace
브로드캐스팅
Binary 지원
```

### 인증
```
JWT 미들웨어
CORS 설정
```

### 확장성
```
Redis Adapter
Sticky Session
다중 서버 동기화
```

---

## 다음 단계

Part 6 (웹 성능 최적화) 완료! 이제 **실전 운영** 파트로 이동합니다!

👉 [Chapter 33. 웹앱 배포 아키텍처](../part07-실전/33-배포-아키텍처.md)

---

## 실습 과제

### 과제 1: 채팅 구현

```javascript
// 1. ws로 기본 채팅
// 2. Socket.IO로 고급 채팅
// 3. Room 기능
// 4. 입력 중 표시
```

### 과제 2: 실시간 알림

```javascript
// 1. Socket.IO + JWT 인증
// 2. 사용자별 Room
// 3. 알림 전송
// 4. 읽음 확인
```

### 과제 3: Redis Adapter

```javascript
// 1. Socket.IO + Redis
// 2. 서버 2대 실행
// 3. 메시지 동기화 확인
// 4. nginx 로드밸런싱
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 4시간
**대상**: Full-stack 개발자
**중요도**: ⭐⭐⭐ 매우 중요!
