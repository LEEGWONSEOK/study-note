# Chapter 2. OSI 7계층과 TCP/IP

## 2.1 왜 계층 구조가 필요할까?

### 비유: 택배 시스템

택배를 보낼 때 여러 단계를 거칩니다:

```
1. 물건 포장 (내용물 준비)
2. 송장 작성 (주소 기입)
3. 택배 기사 픽업 (운송)
4. 집하장 분류
5. 트럭 운송
6. 배송
7. 수령 확인
```

각 단계는 **독립적**입니다:
- 포장 담당자는 트럭 운전 방법을 몰라도 됨
- 트럭 운전사는 상자 안에 뭐가 들었는지 몰라도 됨

**네트워크도 마찬가지!**

---

## 2.2 OSI 7계층 모델

**OSI (Open Systems Interconnection)** 모델은 네트워크 통신을 **7개 계층**으로 나눈 **표준 모델**입니다.

### 전체 구조

```
┌─────────────────────────────────────────────────────┐
│  7. Application (응용 계층)                          │  ← 사용자와 직접 상호작용
│     HTTP, HTTPS, FTP, SMTP, DNS                     │
│     "웹 브라우저, 이메일 클라이언트"                   │
├─────────────────────────────────────────────────────┤
│  6. Presentation (표현 계층)                         │  ← 데이터 형식 변환
│     암호화, 압축, 인코딩                              │
│     "JPEG, MPEG, SSL/TLS"                           │
├─────────────────────────────────────────────────────┤
│  5. Session (세션 계층)                              │  ← 연결 관리
│     세션 생성, 유지, 종료                             │
│     "로그인 세션, API 세션"                           │
├─────────────────────────────────────────────────────┤
│  4. Transport (전송 계층)                            │  ← 데이터 전송 보장
│     TCP, UDP                                        │
│     "포트 번호, 신뢰성 보장"                           │
├─────────────────────────────────────────────────────┤
│  3. Network (네트워크 계층)                           │  ← 경로 설정
│     IP, ICMP, 라우팅                                 │
│     "IP 주소, 라우터"                                 │
├─────────────────────────────────────────────────────┤
│  2. Data Link (데이터 링크 계층)                      │  ← 물리적 연결
│     Ethernet, WiFi, MAC 주소                        │
│     "스위치, MAC 주소"                                │
├─────────────────────────────────────────────────────┤
│  1. Physical (물리 계층)                             │  ← 전기 신호
│     케이블, 전파, 전기 신호                            │
│     "랜선, WiFi 전파"                                 │
└─────────────────────────────────────────────────────┘
```

### 암기법: "아파서 전네 데피"

- **아**: 응용 (Application)
- **파**: 표현 (Presentation)
- **서**: 세션 (Session)
- **전**: 전송 (Transport)
- **네**: 네트워크 (Network)
- **데**: 데이터 링크 (Data Link)
- **피**: 물리 (Physical)

---

## 2.3 각 계층 상세 설명

### Layer 7: Application (응용 계층)

**역할**: 사용자와 직접 상호작용하는 애플리케이션 프로토콜

```
┌──────────────────────────────────────┐
│  웹 브라우저에서 URL 입력             │
│  https://www.example.com/api/users   │
└──────────────────────────────────────┘
           ↓
  Application Layer 프로토콜 사용
           ↓
      HTTP/HTTPS
```

**주요 프로토콜:**

| 프로토콜 | 용도 | 포트 | 예시 |
|---------|------|-----|------|
| **HTTP** | 웹 페이지 전송 | 80 | `http://example.com` |
| **HTTPS** | 암호화된 웹 | 443 | `https://example.com` |
| **DNS** | 도메인 → IP 변환 | 53 | `naver.com` → `223.130.195.95` |
| **FTP** | 파일 전송 | 21 | 서버에 파일 업로드 |
| **SMTP** | 이메일 발송 | 25 | Gmail 이메일 전송 |
| **SSH** | 원격 접속 | 22 | `ssh user@server` |

**웹개발 예시:**

```javascript
// HTTP 프로토콜 사용 (Application Layer)
fetch('https://api.example.com/users', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token123'
  }
});
```

---

### Layer 6: Presentation (표현 계층)

**역할**: 데이터 형식 변환, 암호화, 압축

```
Application Layer의 데이터를
전송 가능한 형식으로 변환

예:
- JSON → 바이트 스트림
- 암호화: 평문 → 암호문
- 압축: 큰 파일 → 작은 파일
```

**실제 동작:**

```javascript
// 웹개발에서 Presentation Layer 역할

// 1. 데이터 직렬화 (JSON → 문자열)
const user = { id: 1, name: 'John' };
const jsonString = JSON.stringify(user);
// → '{"id":1,"name":"John"}'

// 2. 인코딩
const encoded = new TextEncoder().encode(jsonString);
// → 바이트 배열 [123, 34, 105, 100, ...]

// 3. HTTPS의 경우 SSL/TLS로 암호화
// 평문 → 암호화된 데이터
```

**압축 예시:**

```http
# 서버 응답 헤더 (Presentation Layer 기능)
Content-Encoding: gzip
Content-Type: application/json

# 원본 크기: 10MB
# 압축 후: 2MB  ← 80% 절감!
```

---

### Layer 5: Session (세션 계층)

**역할**: 통신 세션 생성, 유지, 종료

```
세션 = 통신하는 동안의 "대화"

비유:
전화 통화 세션
1. 전화 걸기 (세션 시작)
2. 대화하기 (세션 유지)
3. 전화 끊기 (세션 종료)
```

**웹개발 예시:**

```javascript
// 로그인 세션
// 1. 세션 시작
app.post('/login', (req, res) => {
  const user = authenticateUser(req.body);

  // 세션 생성
  req.session.userId = user.id;
  req.session.loginTime = Date.now();

  res.json({ message: '로그인 성공' });
});

// 2. 세션 유지
app.get('/profile', (req, res) => {
  // 세션이 있는지 확인
  if (req.session.userId) {
    const user = getUserById(req.session.userId);
    res.json(user);
  } else {
    res.status(401).json({ error: '로그인 필요' });
  }
});

// 3. 세션 종료
app.post('/logout', (req, res) => {
  req.session.destroy();
  res.json({ message: '로그아웃 완료' });
});
```

**실시간 통신 세션:**

```javascript
// WebSocket 세션
const socket = new WebSocket('wss://example.com/chat');

// 세션 시작
socket.onopen = () => {
  console.log('WebSocket 세션 시작');
};

// 세션 유지 (메시지 주고받기)
socket.onmessage = (event) => {
  console.log('메시지 수신:', event.data);
};

// 세션 종료
socket.onclose = () => {
  console.log('WebSocket 세션 종료');
};
```

---

### Layer 4: Transport (전송 계층) ⭐ 중요!

**역할**: 데이터 전송 보장, 포트 번호로 애플리케이션 구분

```
┌─────────────────────────────────────┐
│  Transport Layer의 역할              │
│                                     │
│  1. 포트 번호로 애플리케이션 구분      │
│  2. 데이터 전송 보장 (TCP)           │
│  3. 빠른 전송 (UDP)                  │
└─────────────────────────────────────┘
```

**TCP vs UDP:**

| 특징 | TCP | UDP |
|------|-----|-----|
| 신뢰성 | 보장 (패킷 손실 시 재전송) | 보장 안 함 |
| 속도 | 느림 | 빠름 |
| 연결 | 연결 지향 (3-way handshake) | 비연결 |
| 순서 | 순서 보장 | 순서 보장 안 함 |
| 용도 | 웹, 이메일, 파일 전송 | 스트리밍, 게임, DNS |

**TCP 3-way Handshake:**

```
[클라이언트]                [서버]
     │                        │
     │───── SYN ─────────────►│  "연결 요청!"
     │                        │
     │◄──── SYN + ACK ────────│  "알겠어, 연결하자!"
     │                        │
     │───── ACK ─────────────►│  "확인!"
     │                        │
   연결 수립 완료!
     │                        │
     │──── 데이터 전송 ───────►│
     │◄──── 데이터 전송 ──────│
```

**포트 번호:**

```
같은 IP에서 여러 서비스 구분

192.168.0.10:80    ← 웹 서버
192.168.0.10:443   ← HTTPS 서버
192.168.0.10:22    ← SSH 서버
192.168.0.10:3306  ← MySQL
```

**웹개발 예시:**

```javascript
// Node.js 서버 - Transport Layer (TCP) 사용
const express = require('express');
const app = express();

// 포트 3000에서 TCP 연결 대기
app.listen(3000, () => {
  console.log('서버가 포트 3000에서 실행 중');
});

// 클라이언트 요청 시 TCP 연결 수립됨
```

---

### Layer 3: Network (네트워크 계층) ⭐ 중요!

**역할**: IP 주소를 사용해 패킷을 목적지까지 라우팅

```
┌──────────────────────────────────────┐
│  출발지: 192.168.0.10                │
│  목적지: 13.209.84.49                │
│                                      │
│  라우터들이 최적 경로 찾기             │
└──────────────────────────────────────┘

[내 PC] → [라우터1] → [라우터2] → [라우터3] → [서버]
192.168.0.10                              13.209.84.49
```

**IP 패킷 구조:**

```
┌─────────────────────────────────────┐
│  IP Header                          │
│  - 출발지 IP: 192.168.0.10          │
│  - 목적지 IP: 13.209.84.49          │
│  - TTL: 64 (Time To Live)           │
│  - Protocol: TCP (6) 또는 UDP (17)   │
├─────────────────────────────────────┤
│  Data (실제 데이터)                  │
│  - HTTP 요청, 파일 데이터 등          │
└─────────────────────────────────────┘
```

**라우팅 예시:**

```bash
# traceroute로 경로 확인
traceroute google.com

# 결과:
1  192.168.0.1       1 ms   (내 공유기)
2  10.1.1.1          5 ms   (ISP 라우터)
3  172.16.0.1       15 ms   (ISP 백본)
4  142.250.207.46   35 ms   (Google 서버)
```

**웹개발에서 중요한 IP 개념:**

```javascript
// Express에서 클라이언트 IP 확인
app.get('/api/check-ip', (req, res) => {
  const clientIp = req.ip; // Network Layer 정보
  console.log('클라이언트 IP:', clientIp);

  res.json({ your_ip: clientIp });
});

// IP 기반 접근 제어
const allowedIPs = ['192.168.0.10', '10.0.0.5'];

app.use((req, res, next) => {
  if (allowedIPs.includes(req.ip)) {
    next();
  } else {
    res.status(403).json({ error: '접근 거부' });
  }
});
```

---

### Layer 2: Data Link (데이터 링크 계층)

**역할**: 같은 네트워크 내에서 MAC 주소로 통신

```
MAC 주소 = 네트워크 카드의 고유 주소 (변경 불가)

예: AA:BB:CC:DD:EE:FF
```

**비유:**
- **IP 주소** = 우편 주소 (이사하면 변경됨)
- **MAC 주소** = 주민등록번호 (평생 고정)

**동작 방식:**

```
같은 WiFi에 연결된 기기들:

[내 노트북]           [공유기]           [스마트폰]
MAC: AA:BB:CC        MAC: 11:22:33      MAC: DD:EE:FF
IP: 192.168.0.10     IP: 192.168.0.1    IP: 192.168.0.20

Data Link Layer에서는 MAC 주소로 통신
Network Layer에서는 IP 주소로 통신
```

**ARP (Address Resolution Protocol):**

```
IP 주소 → MAC 주소 변환

예:
"192.168.0.20의 MAC 주소가 뭐죠?"
→ "DD:EE:FF 입니다"
```

**웹개발에서는:**
- 대부분 신경 쓸 필요 없음
- 하지만 네트워크 디버깅 시 알아두면 유용

---

### Layer 1: Physical (물리 계층)

**역할**: 실제 전기 신호, 전파로 데이터 전송

```
디지털 데이터 → 전기 신호/전파

0과 1의 비트 → 전압 높음/낮음
              → 전파의 진폭 변화
```

**매체:**
- **유선**: 랜선 (Ethernet 케이블), 광케이블
- **무선**: WiFi, Bluetooth, 5G

**웹개발에서는:**
- 거의 신경 쓸 필요 없음
- "랜선이 연결되었는지", "WiFi가 켜져 있는지" 정도

---

## 2.4 TCP/IP 4계층 모델 ⭐ 실무에서 더 많이 사용

OSI 7계층은 **이론적 모델**이고, 실제로는 **TCP/IP 4계층**을 더 많이 사용합니다.

### TCP/IP vs OSI 비교

```
┌─────────────────┐  ┌─────────────────────────┐
│  TCP/IP 4계층    │  │  OSI 7계층               │
├─────────────────┤  ├─────────────────────────┤
│  4. Application │  │  7. Application         │
│     (응용)       │  │  6. Presentation        │
│                 │  │  5. Session             │
├─────────────────┤  ├─────────────────────────┤
│  3. Transport   │  │  4. Transport           │
│     (전송)       │  │                         │
├─────────────────┤  ├─────────────────────────┤
│  2. Internet    │  │  3. Network             │
│     (인터넷)     │  │                         │
├─────────────────┤  ├─────────────────────────┤
│  1. Network     │  │  2. Data Link           │
│     Access      │  │  1. Physical            │
│   (네트워크접근) │  │                         │
└─────────────────┘  └─────────────────────────┘
```

### TCP/IP 4계층 상세

#### Layer 4: Application (응용 계층)

OSI의 Application + Presentation + Session 통합

```
HTTP, HTTPS, FTP, SMTP, DNS, SSH 등
```

#### Layer 3: Transport (전송 계층)

TCP, UDP

#### Layer 2: Internet (인터넷 계층)

IP, ICMP, ARP

#### Layer 1: Network Access (네트워크 접근 계층)

Ethernet, WiFi 등 물리적 전송

---

## 2.5 데이터 캡슐화 (Encapsulation)

데이터가 각 계층을 지날 때마다 **헤더가 추가**됩니다.

### 비유: 러시아 인형 (마트료시카)

```
큰 인형 안에 중간 인형
중간 인형 안에 작은 인형
작은 인형 안에 가장 작은 인형
```

### 실제 캡슐화 과정

```
┌─────────────────────────────────────────────────────┐
│  Application Layer                                  │
│  HTTP Request: GET /users HTTP/1.1                  │
└─────────────────────────────────────────────────────┘
                      ↓ 전송 계층으로 전달
┌─────────────────────────────────────────────────────┐
│  Transport Layer                                    │
│  ┌──────────┬────────────────────────────┐          │
│  │TCP Header│ HTTP Request               │          │
│  │Port:443  │ GET /users HTTP/1.1        │          │
│  └──────────┴────────────────────────────┘          │
└─────────────────────────────────────────────────────┘
                      ↓ 네트워크 계층으로 전달
┌─────────────────────────────────────────────────────┐
│  Network Layer                                      │
│  ┌──────────┬──────────┬─────────────────┐          │
│  │IP Header │TCP Header│ HTTP Request    │          │
│  │Src:..10  │Port:443  │ GET /users      │          │
│  │Dst:..49  │          │                 │          │
│  └──────────┴──────────┴─────────────────┘          │
└─────────────────────────────────────────────────────┘
                      ↓ 데이터 링크 계층으로 전달
┌─────────────────────────────────────────────────────┐
│  Data Link Layer                                    │
│  ┌─────────┬─────────┬─────────┬──────────┬───────┐ │
│  │Ethernet │IP Header│TCP Head │HTTP Req  │Trailer│ │
│  │Header   │         │         │          │       │ │
│  │MAC Addr │         │         │          │       │ │
│  └─────────┴─────────┴─────────┴──────────┴───────┘ │
└─────────────────────────────────────────────────────┘
                      ↓ 물리 계층
              전기 신호/전파로 전송
```

### 역캡슐화 (서버에서 수신 시)

```
전기 신호 수신
      ↓
Ethernet Header 제거
      ↓
IP Header 확인 및 제거
      ↓
TCP Header 확인 및 제거
      ↓
HTTP Request 추출
```

---

## 2.6 실제 웹 요청에서 계층별 동작

### 예시: `https://api.example.com/users` 요청

```javascript
fetch('https://api.example.com/users');
```

#### Layer 7 (Application): HTTP

```http
GET /users HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Cookie: session_id=abc123
```

#### Layer 6 (Presentation): SSL/TLS 암호화

```
HTTP 요청 → TLS로 암호화
평문 → 암호문
```

#### Layer 5 (Session): 세션 유지

```
기존 HTTPS 연결이 있으면 재사용
없으면 새로 수립
```

#### Layer 4 (Transport): TCP

```
TCP Header 추가:
- 출발지 포트: 54321 (랜덤)
- 목적지 포트: 443 (HTTPS)
- 시퀀스 번호
- 체크섬 (데이터 무결성 검증)
```

#### Layer 3 (Network): IP

```
IP Header 추가:
- 출발지 IP: 192.168.0.10 (내 PC)
- 목적지 IP: 93.184.216.34 (api.example.com)
- TTL: 64
```

#### Layer 2 (Data Link): Ethernet

```
Ethernet Header 추가:
- 출발지 MAC: AA:BB:CC:DD:EE:FF
- 목적지 MAC: 11:22:33:44:55:66 (공유기)
```

#### Layer 1 (Physical): 전기 신호

```
WiFi 전파 또는 랜선 전기 신호로 전송
```

---

## 2.7 왜 계층 구조가 중요한가?

### 1. 독립성과 모듈화

```
Application Layer 개발자는 TCP/IP를 몰라도 됨

// HTTP만 알면 됨
fetch('https://example.com/api');

// TCP 연결, IP 라우팅은 하위 계층이 알아서 처리
```

### 2. 문제 격리

```
문제 발생 시 어느 계층인지 파악

"API 호출이 안 돼요!"

Layer 7 문제: HTTP 상태 코드 404 → URL 오류
Layer 4 문제: Connection timeout → 방화벽/포트 막힘
Layer 3 문제: No route to host → IP 라우팅 실패
Layer 2 문제: 네트워크 unreachable → 케이블 끊김
```

### 3. 표준화

```
HTTP는 어떤 네트워크에서든 동작
- WiFi든, 유선이든, 5G든 상관없음
- 하위 계층이 알아서 처리
```

---

## 2.8 웹개발자가 주로 다루는 계층

### Layer 7 (Application): 90% ⭐⭐⭐

```javascript
// HTTP 요청/응답
fetch('/api/users');

// WebSocket
const ws = new WebSocket('wss://example.com');

// GraphQL
client.query({ query: GET_USERS });
```

### Layer 4 (Transport): 5%

```javascript
// 포트 번호 설정
app.listen(3000);  // TCP 포트 3000

// 타임아웃 설정 (TCP)
fetch('/api/users', { signal: AbortSignal.timeout(5000) });
```

### Layer 3 (Network): 3%

```javascript
// IP 기반 접근 제어
if (req.ip.startsWith('192.168.')) {
  // 내부 네트워크만 허용
}
```

### Layer 2, 1: 거의 안 다룸

DevOps나 인프라 엔지니어 영역

---

## 2.9 디버깅 시 계층별 접근법

### 문제: "API 호출이 안 됩니다"

#### 1단계: Application Layer 확인

```bash
# curl로 HTTP 요청 테스트
curl -v https://api.example.com/users

# 결과 확인:
# - 200 OK: 성공
# - 404: URL 오류 (Application Layer 문제)
# - 401: 인증 오류 (Application Layer 문제)
# - 500: 서버 에러 (Application Layer 문제)
```

#### 2단계: Transport Layer 확인

```bash
# 포트가 열려 있는지 확인
telnet api.example.com 443

# 결과:
# - Connected: TCP 연결 성공
# - Connection refused: 포트 막힘 (Transport Layer 문제)
```

#### 3단계: Network Layer 확인

```bash
# IP 라우팅 확인
ping api.example.com

# 결과:
# - 응답 있음: IP 통신 OK
# - Request timeout: IP 라우팅 실패 (Network Layer 문제)
```

#### 4단계: DNS 확인

```bash
# 도메인 → IP 변환 확인
nslookup api.example.com

# 결과:
# - IP 반환: DNS OK
# - No answer: DNS 문제
```

---

## 2.10 실무 적용 예시

### 예시 1: HTTPS 배포

```
Layer 7 (Application): Node.js Express 앱
Layer 6 (Presentation): SSL/TLS 인증서 (Let's Encrypt)
Layer 4 (Transport): TCP 포트 443
Layer 3 (Network): 서버 IP 주소
```

```javascript
// HTTPS 서버 설정
const https = require('https');
const fs = require('fs');
const express = require('express');

const app = express();

const options = {
  key: fs.readFileSync('privkey.pem'),    // Presentation Layer
  cert: fs.readFileSync('cert.pem')
};

https.createServer(options, app).listen(443);  // Transport Layer
```

### 예시 2: 로드밸런서 설정

```
     Client
       ↓
Layer 7: HTTPS 요청
       ↓
Layer 4: 로드밸런서 (TCP 443)
       ↓
   ┌───┴───┬───────┐
   ↓       ↓       ↓
서버1   서버2   서버3
```

### 예시 3: 방화벽 설정 (AWS 보안 그룹)

```
Layer 4 (Transport):
- 인바운드 규칙:
  - TCP 포트 80 (HTTP) - 모든 IP 허용
  - TCP 포트 443 (HTTPS) - 모든 IP 허용
  - TCP 포트 22 (SSH) - 특정 IP만 허용

Layer 3 (Network):
- IP 기반 필터링:
  - 192.168.0.0/16 → 허용
  - 다른 IP → 거부
```

---

## 핵심 요약

### OSI 7계층

1. **Application**: HTTP, HTTPS, DNS (웹개발자가 주로 다룸)
2. **Presentation**: 암호화, 압축 (SSL/TLS)
3. **Session**: 세션 관리 (로그인 세션)
4. **Transport**: TCP/UDP, 포트 번호 (중요!)
5. **Network**: IP 주소, 라우팅 (중요!)
6. **Data Link**: MAC 주소
7. **Physical**: 전기 신호

### TCP/IP 4계층 (실무에서 더 많이 사용)

1. **Application**: HTTP, DNS 등
2. **Transport**: TCP, UDP
3. **Internet**: IP
4. **Network Access**: Ethernet, WiFi

### 계층별 중요도 (웹개발자 기준)

- **Layer 7 (Application)**: ⭐⭐⭐ 매우 중요
- **Layer 4 (Transport)**: ⭐⭐ 중요
- **Layer 3 (Network)**: ⭐⭐ 중요
- **Layer 2, 1**: ⭐ 알아두면 좋음

### 캡슐화

각 계층을 지날 때마다 헤더 추가
```
Data → TCP Header + Data → IP Header + TCP Header + Data → ...
```

---

## 다음 단계

계층 구조를 이해했으니, 이제 **IP 주소와 서브넷**을 배워봅시다!

👉 [Chapter 3. IP 주소와 서브넷](03-IP주소와-서브넷.md)

---

## 실습 과제

### 과제 1: Chrome DevTools로 계층 확인

1. Chrome DevTools → Network 탭
2. 아무 웹사이트 접속
3. 요청 클릭 → Headers 탭 확인
4. 다음 정보 찾기:
   - **Application Layer**: HTTP 메서드, 상태 코드
   - **Transport Layer**: Remote Address (IP:Port)
   - **Network Layer**: IP 주소

### 과제 2: curl로 계층 확인

```bash
# HTTP 요청 상세 정보 보기
curl -v https://www.google.com

# 출력에서 찾아보기:
# - Application: HTTP/1.1, 상태 코드
# - Transport: Connected to ... (IP:Port)
# - Presentation: SSL/TLS 핸드셰이크
```

### 과제 3: ping과 traceroute

```bash
# Network Layer 테스트
ping google.com

# 라우팅 경로 확인
traceroute google.com
# (Windows: tracert google.com)

# 몇 개의 라우터를 거쳐가나요?
```

### 과제 4: 포트 확인

```bash
# Transport Layer 테스트
# 여러 포트 확인

telnet google.com 80   # HTTP
telnet google.com 443  # HTTPS
telnet google.com 22   # SSH (막혀 있을 것)

# 어떤 포트가 열려 있나요?
```

### 과제 5: 실제 패킷 분석 (선택)

Wireshark 설치 후:
1. 웹사이트 접속하면서 패킷 캡처
2. HTTP 패킷 선택
3. 각 계층의 헤더 확인:
   - Ethernet Header (Layer 2)
   - IP Header (Layer 3)
   - TCP Header (Layer 4)
   - HTTP Data (Layer 7)

---

**작성일**: 2024년
**난이도**: ⭐⭐ 초급
**예상 학습 시간**: 2시간
**대상**: 모든 웹개발자