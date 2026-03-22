# Chapter 4. TCP vs UDP

## 4.1 전송 계층(Transport Layer)의 역할

**전송 계층**은 애플리케이션 간 데이터 전송을 담당합니다.

### 비유: 택배 배송 방식

```
TCP = 등기우편
- 서명 확인
- 배송 추적
- 분실 시 재배송
- 느리지만 확실

UDP = 일반우편
- 그냥 우체통에 투척
- 추적 불가
- 분실 가능
- 빠르지만 불확실
```

---

## 4.2 TCP (Transmission Control Protocol)

### 정의

**신뢰성 있는 연결 지향 프로토콜**

### 핵심 특징

```
1. 연결 지향 (Connection-Oriented)
   → 데이터 전송 전 연결 수립 필수

2. 신뢰성 보장 (Reliable)
   → 패킷 손실 시 재전송
   → 순서 보장

3. 흐름 제어 (Flow Control)
   → 수신자 속도에 맞춤

4. 혼잡 제어 (Congestion Control)
   → 네트워크 상태에 따라 속도 조절
```

---

## 4.3 TCP 3-Way Handshake ⭐ 중요!

### 연결 수립 과정

```
[클라이언트]                        [서버]
     │                                │
     │                                │ (LISTEN 상태)
     │                                │
     │──────── SYN ─────────────────►│
     │    "연결하고 싶어요!"            │
     │    Seq = 1000                  │
     │                                │
     │                                │ (SYN_RECEIVED)
     │◄────── SYN + ACK ──────────────│
     │    "좋아요! 연결합시다!"          │
     │    Seq = 2000, Ack = 1001      │
     │                                │
     │──────── ACK ─────────────────►│
     │    "확인했어요!"                 │
     │    Ack = 2001                  │
     │                                │
     │         연결 수립 완료!           │
     │◄──────── 데이터 전송 ────────────►│
```

### 각 단계 상세

```
1단계: SYN (Synchronize)
클라이언트: "서버님, 통신 시작할까요? 제 시퀀스 번호는 1000입니다"

2단계: SYN + ACK (Synchronize + Acknowledge)
서버: "좋아요! 제 시퀀스는 2000이고, 당신 번호 1000 받았어요(1001 기다림)"

3단계: ACK (Acknowledge)
클라이언트: "당신 번호 2000 받았습니다(2001 기다림). 이제 데이터 보낼게요!"
```

### 웹개발 예시

```javascript
// HTTP 요청 시 자동으로 TCP 3-Way Handshake 발생
const response = await fetch('https://api.example.com/users');

// 내부 동작:
// 1. TCP 3-Way Handshake (연결 수립)
// 2. TLS Handshake (HTTPS인 경우)
// 3. HTTP 요청 전송
// 4. HTTP 응답 수신
// 5. TCP 연결 유지 또는 종료
```

### 실제 확인하기

```bash
# tcpdump로 3-Way Handshake 확인
sudo tcpdump -i any port 80

# 출력 예시:
# 12:00:00.000000 IP client.54321 > server.80: Flags [S], seq 1000
# 12:00:00.050000 IP server.80 > client.54321: Flags [S.], seq 2000, ack 1001
# 12:00:00.051000 IP client.54321 > server.80: Flags [.], ack 2001
```

---

## 4.4 TCP 4-Way Handshake (연결 종료)

### 연결 종료 과정

```
[클라이언트]                        [서버]
     │                                │
     │──────── FIN ─────────────────►│
     │    "전송 끝! 연결 끊을게요"       │
     │                                │
     │◄────── ACK ───────────────────│
     │    "확인! 잠시만요..."           │
     │                                │
     │◄────── FIN ───────────────────│
     │    "저도 전송 끝났어요"           │
     │                                │
     │──────── ACK ─────────────────►│
     │    "확인! 종료합니다"             │
     │                                │
     └────── 연결 종료 ─────────────────┘
```

### 왜 4단계나 필요할까?

```
양방향 통신이기 때문!

클라이언트 → 서버: 더 보낼 데이터 없음 (FIN)
서버 → 클라이언트: 아직 보낼 데이터 있을 수 있음
서버 → 클라이언트: 이제 다 보냈음 (FIN)
클라이언트 → 서버: 알겠음 (ACK)
```

---

## 4.5 TCP 신뢰성 보장 메커니즘

### 1. 순서 번호 (Sequence Number)

```
전송 순서:
패킷1 (Seq: 1000, 데이터: 500바이트)
패킷2 (Seq: 1500, 데이터: 500바이트)
패킷3 (Seq: 2000, 데이터: 500바이트)

수신 순서: 패킷1 → 패킷3 → 패킷2 (순서 뒤바뀜)
          ↓
TCP가 자동 재정렬: 패킷1 → 패킷2 → 패킷3
```

### 2. 확인 응답 (ACK)

```
[송신자]                [수신자]
   │──── 데이터(Seq:1000) ────►│
   │                           │
   │◄──── ACK(1500) ───────────│
   │    "1500까지 받았어요"      │
   │                           │
   │──── 데이터(Seq:1500) ────►│
   │                           │
   │◄──── ACK(2000) ───────────│
   │    "2000까지 받았어요"      │
```

### 3. 재전송 (Retransmission)

```
[송신자]                [수신자]
   │──── 패킷1 ────────────►│ ✅ 도착
   │◄──── ACK ─────────────│
   │                       │
   │──── 패킷2 ────────X    │ ❌ 손실!
   │                       │
   │ (타임아웃 대기...)     │
   │                       │
   │──── 패킷2 재전송 ─────►│ ✅ 도착
   │◄──── ACK ─────────────│
```

### 4. 체크섬 (Checksum)

```
데이터 무결성 검증

전송:
데이터 = "Hello"
체크섬 = 계산값(예: 0x1234)

수신:
데이터 = "Hello"
체크섬 재계산 = 0x1234 ✅ 일치!

만약 손상:
데이터 = "Hxllo" (손상됨)
체크섬 재계산 = 0x5678 ❌ 불일치!
→ 패킷 버리고 재전송 요청
```

---

## 4.6 TCP 흐름 제어 (Flow Control)

### Window Size

```
수신자가 한 번에 받을 수 있는 데이터 양 제한

[송신자]                    [수신자]
   │                          │ Window Size: 1000바이트
   │                          │
   │──── 500바이트 ──────────►│
   │──── 500바이트 ──────────►│
   │                          │ (버퍼 가득 참)
   │◄──── Window: 0 ─────────│
   │   "잠깐 멈춰주세요!"        │
   │                          │
   │   (대기...)              │ (버퍼 비움)
   │                          │
   │◄──── Window: 1000 ──────│
   │   "다시 보내도 돼요!"       │
   │                          │
   │──── 500바이트 ──────────►│
```

### 슬라이딩 윈도우 (Sliding Window)

```
효율적인 전송을 위해 ACK 받기 전에 여러 패킷 전송

Window Size = 4 패킷

[1] [2] [3] [4] | [5] [6] [7] [8] ...
 ↑              ↑
전송 시작        Window 끝

패킷1 ACK 받음:
    [2] [3] [4] [5] | [6] [7] [8] ...
     ↑              ↑
    전송             Window 끝 (이동)
```

---

## 4.7 TCP 혼잡 제어 (Congestion Control)

### 네트워크 혼잡 감지

```
상황:
라우터 버퍼 가득 참 → 패킷 드롭 → 재전송 증가
→ TCP가 감지하고 전송 속도 조절
```

### Slow Start

```
처음에는 천천히, 점점 빠르게

Window Size:
1   →  2   →  4   →  8   →  16  →  32 (지수 증가)
│      │      │      │      │      │
│      │      │      │      │      └─ 혼잡 감지!
│      │      │      │      │         (Window를 절반으로)
│      │      │      │      │
│      │      │      │      └─ 16으로 돌아감
│      │      │      │         이후 선형 증가: 17, 18, 19...
```

### AIMD (Additive Increase Multiplicative Decrease)

```
증가: 천천히 (+1)
감소: 빠르게 (×0.5)

정상 → 1, 2, 3, 4, 5, 6, 7, 8 (선형 증가)
혼잡! → 4 (절반으로)
정상 → 5, 6, 7, 8 (다시 증가)
혼잡! → 4 (절반으로)
```

---

## 4.8 UDP (User Datagram Protocol)

### 정의

**비연결성, 신뢰성 없는 빠른 프로토콜**

### 핵심 특징

```
1. 비연결성 (Connectionless)
   → 연결 수립 없이 바로 전송

2. 신뢰성 없음 (Unreliable)
   → 패킷 손실 시 재전송 안 함
   → 순서 보장 안 함

3. 빠름 (Fast)
   → 오버헤드 최소화

4. 단순함 (Simple)
   → 헤더가 매우 간단 (8바이트)
```

### UDP 동작

```
[송신자]                [수신자]
   │                       │
   │──── 데이터 ─────────►│  (연결 수립 없음)
   │──── 데이터 ─────────►│
   │──── 데이터 ───X       │  (손실되어도 무시)
   │──── 데이터 ─────────►│
   │                       │
   └─ 계속 전송, ACK 없음 ──┘
```

---

## 4.9 TCP vs UDP 비교 ⭐ 중요!

### 상세 비교표

```
┌──────────────┬─────────────────────┬─────────────────────┐
│    항목       │        TCP          │        UDP          │
├──────────────┼─────────────────────┼─────────────────────┤
│ 연결         │ 연결 지향            │ 비연결              │
│              │ (3-Way Handshake)   │ (바로 전송)         │
├──────────────┼─────────────────────┼─────────────────────┤
│ 신뢰성       │ 보장                │ 보장 안 함          │
│              │ (재전송, 순서 보장)  │ (손실 가능)         │
├──────────────┼─────────────────────┼─────────────────────┤
│ 속도         │ 느림                │ 빠름                │
│              │ (오버헤드 많음)      │ (오버헤드 적음)     │
├──────────────┼─────────────────────┼─────────────────────┤
│ 헤더 크기    │ 20~60바이트         │ 8바이트             │
├──────────────┼─────────────────────┼─────────────────────┤
│ 순서 보장    │ ✅ 보장              │ ❌ 보장 안 함        │
├──────────────┼─────────────────────┼─────────────────────┤
│ 흐름 제어    │ ✅ 있음              │ ❌ 없음              │
├──────────────┼─────────────────────┼─────────────────────┤
│ 혼잡 제어    │ ✅ 있음              │ ❌ 없음              │
├──────────────┼─────────────────────┼─────────────────────┤
│ 브로드캐스트 │ ❌ 불가능            │ ✅ 가능              │
├──────────────┼─────────────────────┼─────────────────────┤
│ 일대다 통신  │ ❌ 불가능            │ ✅ 가능              │
├──────────────┼─────────────────────┼─────────────────────┤
│ 용도         │ 웹(HTTP)            │ 스트리밍            │
│              │ 이메일(SMTP)        │ 게임                │
│              │ 파일전송(FTP)       │ DNS                 │
│              │ SSH                 │ VoIP                │
│              │ 데이터베이스        │ 실시간 통신         │
└──────────────┴─────────────────────┴─────────────────────┘
```

### 선택 기준

```
TCP 사용:
- 데이터 손실이 치명적인 경우
- 순서가 중요한 경우
- 예: 파일 다운로드, 이메일, 웹 페이지, API

UDP 사용:
- 속도가 중요한 경우
- 약간의 손실은 괜찮은 경우
- 예: 라이브 스트리밍, 온라인 게임, VoIP
```

---

## 4.10 웹개발에서 TCP/UDP 사용 예시

### TCP 사용 (대부분의 웹 통신)

```javascript
// HTTP/HTTPS - TCP 기반
const response = await fetch('https://api.example.com/users');

// WebSocket - TCP 기반
const socket = new WebSocket('wss://example.com/chat');
socket.onmessage = (event) => {
  console.log('메시지:', event.data);
  // TCP가 순서와 신뢰성 보장
};

// Server-Sent Events - TCP 기반
const eventSource = new EventSource('/events');
eventSource.onmessage = (event) => {
  console.log('이벤트:', event.data);
};

// GraphQL - TCP 기반 (HTTP 위에서 동작)
const result = await client.query({ query: GET_USERS });
```

### UDP 사용 (특수한 경우)

```javascript
// WebRTC (P2P 영상통화) - UDP 사용 가능
const peerConnection = new RTCPeerConnection();

// Data Channel로 게임 데이터 전송 (UDP 모드 가능)
const dataChannel = peerConnection.createDataChannel('game', {
  ordered: false,        // 순서 보장 안 함 (UDP처럼)
  maxRetransmits: 0      // 재전송 안 함 (UDP처럼)
});

// DNS 조회 (내부적으로 UDP)
// 브라우저가 자동 처리
```

### Node.js UDP 서버 예시

```javascript
const dgram = require('dgram');

// UDP 서버 생성
const server = dgram.createSocket('udp4');

server.on('message', (msg, rinfo) => {
  console.log(`받은 메시지: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  const address = server.address();
  console.log(`UDP 서버 시작: ${address.address}:${address.port}`);
});

server.bind(41234);

// UDP 클라이언트
const client = dgram.createSocket('udp4');
const message = Buffer.from('Hello UDP!');

client.send(message, 41234, 'localhost', (err) => {
  if (err) console.error(err);
  client.close();
});
```

---

## 4.11 실시간 통신: TCP vs UDP

### 시나리오 1: 채팅 애플리케이션

```javascript
// TCP 사용 (WebSocket)
// 장점: 메시지 순서 보장, 손실 없음
// 단점: 약간의 지연 가능

const socket = new WebSocket('wss://chat.example.com');

socket.send(JSON.stringify({
  type: 'message',
  text: 'Hello!'
}));

// 모든 메시지가 순서대로 도착 보장
```

### 시나리오 2: 온라인 게임

```javascript
// UDP 선호 (WebRTC Data Channel)
// 장점: 낮은 지연, 빠른 속도
// 단점: 일부 패킷 손실 가능 (괜찮음)

const dataChannel = peerConnection.createDataChannel('game', {
  ordered: false,         // 순서 중요하지 않음
  maxRetransmits: 0       // 재전송 안 함 (지연 방지)
});

// 플레이어 위치 업데이트 (초당 60회)
setInterval(() => {
  dataChannel.send(JSON.stringify({
    x: player.x,
    y: player.y
  }));
  // 일부 손실되어도 다음 업데이트가 곧 옴
}, 16); // 60 FPS
```

### 시나리오 3: 라이브 스트리밍

```
전통적 방식 (TCP):
- HLS (HTTP Live Streaming)
- DASH
→ 안정적이지만 지연 시간 김 (3-30초)

실시간 방식 (UDP):
- WebRTC
- RTMP over UDP
→ 낮은 지연 (0.5초 이하), 화질 다소 불안정
```

---

## 4.12 TCP 최적화 기법

### 1. Keep-Alive

```javascript
// HTTP Keep-Alive로 TCP 연결 재사용
const http = require('http');
const agent = new http.Agent({
  keepAlive: true,
  keepAliveMsecs: 1000,
  maxSockets: 10
});

// 같은 서버로 여러 요청 시 연결 재사용
fetch('http://api.example.com/users', { agent });
fetch('http://api.example.com/posts', { agent });
// → 3-Way Handshake 한 번만!
```

### 2. TCP Fast Open (TFO)

```
일반 TCP:
1. SYN (연결)
2. SYN+ACK
3. ACK
4. 데이터 전송 ← 여기서야 시작

TCP Fast Open:
1. SYN + 데이터 (동시에!)
2. SYN+ACK + 데이터
→ 왕복 시간 절약
```

### 3. Nagle 알고리즘 비활성화

```javascript
// Node.js에서 Nagle 알고리즘 끄기
socket.setNoDelay(true);

// Nagle 알고리즘:
// - 작은 패킷들을 모아서 한 번에 전송 (효율적)
// - 하지만 실시간 통신에는 지연 발생

// 실시간 게임, 채팅 → Nagle OFF
// 파일 전송, 일반 웹 → Nagle ON (기본)
```

---

## 4.13 패킷 손실 시뮬레이션

### TCP의 재전송 확인

```bash
# Linux에서 패킷 손실 시뮬레이션
sudo tc qdisc add dev eth0 root netem loss 10%
# → 10% 패킷 손실

# 테스트
curl http://example.com
# → TCP가 자동으로 재전송하여 정상 동작

# 복구
sudo tc qdisc del dev eth0 root
```

### UDP의 손실 허용

```javascript
// UDP로 100개 패킷 전송
for (let i = 0; i < 100; i++) {
  client.send(`Packet ${i}`, 41234, 'localhost');
}

// 10% 손실 시:
// 받은 패킷: 90개 (순서 뒤바뀔 수 있음)
// TCP였다면: 100개 모두 (순서 보장, 느림)
```

---

## 4.14 실무 시나리오

### 시나리오 1: API 서버 응답 느림

```
문제:
API 응답 시간이 3초 → 너무 느림

분석:
1. TCP 3-Way Handshake: 50ms
2. TLS Handshake: 100ms
3. HTTP 요청/응답: 2850ms ← 진짜 문제
4. TCP 4-Way Handshake: 0ms (Keep-Alive로 생략)

해결:
- Keep-Alive 활성화 → Handshake 재사용
- HTTP/2 사용 → 멀티플렉싱
- CDN 사용 → 지연 시간 감소
```

### 시나리오 2: 실시간 알림 구현

```javascript
// 방법 1: Long Polling (TCP)
// 단점: 연결 유지 비용 높음
async function longPoll() {
  while (true) {
    const response = await fetch('/notifications?timeout=30');
    const data = await response.json();
    if (data.notifications.length > 0) {
      showNotifications(data.notifications);
    }
  }
}

// 방법 2: Server-Sent Events (TCP)
// 장점: 단방향 실시간, 자동 재연결
const eventSource = new EventSource('/notifications');
eventSource.onmessage = (event) => {
  const notification = JSON.parse(event.data);
  showNotification(notification);
};

// 방법 3: WebSocket (TCP)
// 장점: 양방향 실시간
const socket = new WebSocket('wss://api.example.com/notifications');
socket.onmessage = (event) => {
  const notification = JSON.parse(event.data);
  showNotification(notification);
};
```

### 시나리오 3: 대용량 파일 업로드

```javascript
// TCP의 흐름 제어가 자동으로 처리

const file = document.getElementById('fileInput').files[0];
const formData = new FormData();
formData.append('file', file);

// TCP가 자동으로:
// 1. 파일을 작은 패킷으로 분할
// 2. 순서대로 전송
// 3. 손실된 패킷 재전송
// 4. 수신자 속도에 맞춰 조절
const response = await fetch('/upload', {
  method: 'POST',
  body: formData
});

// 개발자는 TCP의 복잡한 메커니즘을 신경 쓸 필요 없음!
```

---

## 4.15 TCP/UDP 헤더 구조

### TCP 헤더 (20-60바이트)

```
 0                   16                  31
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       Source Port     |   Dest Port     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Sequence Number               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Acknowledgment Number            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Offset |Res|Flags|      Window Size      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Checksum         |  Urgent Pointer |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Options (가변)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

주요 필드:
- Source/Dest Port: 출발지/목적지 포트
- Sequence Number: 순서 번호
- ACK Number: 확인 번호
- Flags: SYN, ACK, FIN 등
- Window Size: 흐름 제어
- Checksum: 데이터 무결성
```

### UDP 헤더 (8바이트)

```
 0                   16                  31
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       Source Port     |   Dest Port     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Length        |    Checksum     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

주요 필드:
- Source/Dest Port: 포트 번호
- Length: 전체 길이
- Checksum: 데이터 무결성 (선택)

매우 단순! → 빠름
```

---

## 핵심 요약

### TCP (Transmission Control Protocol)

```
✅ 연결 지향 (3-Way Handshake)
✅ 신뢰성 보장 (재전송, 순서 보장)
✅ 흐름 제어, 혼잡 제어
❌ 느림 (오버헤드)

용도: HTTP, HTTPS, SSH, FTP, 이메일, DB
```

### UDP (User Datagram Protocol)

```
✅ 빠름 (오버헤드 최소)
✅ 단순함 (8바이트 헤더)
✅ 브로드캐스트/멀티캐스트 가능
❌ 신뢰성 없음 (손실, 순서 보장 안 함)

용도: DNS, 스트리밍, 게임, VoIP
```

### 선택 기준

```
데이터 손실 불가 → TCP
실시간 속도 중요 → UDP
웹 개발 대부분 → TCP (HTTP/HTTPS)
```

---

## 다음 단계

TCP와 UDP를 이해했으니, 이제 **포트와 소켓**을 배워봅시다!

👉 [Chapter 5. 포트와 소켓](05-포트와-소켓.md)

---

## 실습 과제

### 과제 1: Wireshark로 3-Way Handshake 확인

```
1. Wireshark 설치
2. 캡처 시작
3. 웹 브라우저로 http://example.com 접속
4. Wireshark에서 TCP 필터: tcp.flags.syn == 1
5. 3-Way Handshake 패킷 찾기:
   - SYN
   - SYN+ACK
   - ACK
```

### 과제 2: TCP Keep-Alive 테스트

```javascript
// keep-alive.js
const http = require('http');

// Keep-Alive 활성화
const agent = new http.Agent({
  keepAlive: true,
  maxSockets: 1
});

// 같은 서버로 10번 요청
for (let i = 0; i < 10; i++) {
  http.get('http://example.com', { agent }, (res) => {
    console.log(`요청 ${i + 1} 완료`);
  });
}

// Wireshark로 확인:
// → 3-Way Handshake가 한 번만 발생하는지 확인
```

### 과제 3: UDP 에코 서버 만들기

```javascript
// udp-server.js
const dgram = require('dgram');
const server = dgram.createSocket('udp4');

server.on('message', (msg, rinfo) => {
  console.log(`받음: ${msg} from ${rinfo.address}:${rinfo.port}`);

  // 에코 (그대로 돌려보내기)
  server.send(msg, rinfo.port, rinfo.address);
});

server.bind(41234, () => {
  console.log('UDP 서버 시작: 포트 41234');
});

// udp-client.js
const dgram = require('dgram');
const client = dgram.createSocket('udp4');

client.on('message', (msg) => {
  console.log(`서버 응답: ${msg}`);
  client.close();
});

const message = Buffer.from('Hello UDP!');
client.send(message, 41234, 'localhost');
```

### 과제 4: Chrome DevTools로 TCP 연결 분석

```
1. Chrome DevTools 열기 (F12)
2. Network 탭
3. 웹사이트 접속
4. 요청 선택 → Timing 탭 확인
   - Queueing
   - Stalled
   - DNS Lookup
   - Initial connection (TCP 3-Way Handshake)
   - SSL (TLS Handshake)
   - Request sent
   - Waiting (TTFB)
   - Content Download

어느 단계가 가장 오래 걸리나요?
```

---

**작성일**: 2024년
**난이도**: ⭐⭐ 초급
**예상 학습 시간**: 2시간
**대상**: 모든 웹개발자
