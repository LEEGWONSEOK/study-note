# Chapter 8. HTTP 헤더 완전정복

## 8.1 HTTP 헤더란?

**HTTP 헤더**는 요청/응답에 대한 **메타데이터(부가 정보)**를 전달합니다.

### 비유: 편지 봉투

```
┌──────────────────────────────────┐
│  받는 사람: 서울시 강남구...          │  ← 헤더
│  보내는 사람: 부산시...              │  ← 헤더
│  우편번호: 06000                   │  ← 헤더
├──────────────────────────────────┤
│                                  │
│  안녕하세요,                        │  ← 본문(Body)
│  편지 내용...                       │
│                                  │
└──────────────────────────────────┘
```

### HTTP 헤더 구조

```http
GET /api/users HTTP/1.1
Host: api.example.com                    ← 헤더
User-Agent: Mozilla/5.0                  ← 헤더
Accept: application/json                 ← 헤더
Authorization: Bearer token123           ← 헤더
                                         ← 빈 줄
{"name": "John"}                         ← 본문
```

---

## 8.2 헤더 분류

### 요청 헤더 (Request Headers)

클라이언트 → 서버로 보내는 정보

```
Host: api.example.com
User-Agent: Mozilla/5.0...
Accept: application/json
Authorization: Bearer token
Cookie: session_id=abc123
```

### 응답 헤더 (Response Headers)

서버 → 클라이언트로 보내는 정보

```
Content-Type: application/json
Content-Length: 1234
Set-Cookie: session_id=xyz789
Cache-Control: max-age=3600
```

### 일반 헤더 (General Headers)

요청/응답 모두 사용

```
Date: Mon, 15 Jan 2024 10:00:00 GMT
Connection: keep-alive
```

### 엔티티 헤더 (Entity Headers)

본문(Body)에 대한 정보

```
Content-Type: application/json
Content-Length: 1234
Content-Encoding: gzip
```

---

## 8.3 필수 요청 헤더 ⭐

### Host (필수!)

```http
Host: api.example.com

역할: 요청 대상 서버의 도메인
필수: HTTP/1.1부터 필수!
```

**왜 필수일까?**

```
하나의 IP에 여러 도메인 호스팅 (Virtual Host)

13.209.84.49
├─ api.example.com
├─ www.example.com
└─ admin.example.com

Host 헤더로 구분!
```

```javascript
// 잘못된 요청 (Host 없음)
fetch('http://13.209.84.49/api/users')
// 서버: "어느 도메인 요청인지 모르겠음"

// 올바른 요청
fetch('http://api.example.com/api/users')
// Host: api.example.com 자동 추가됨
```

### User-Agent

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
            AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/120.0.0.0 Safari/537.36

역할: 클라이언트 정보 (브라우저, OS 등)
용도:
- 통계 수집
- 모바일/데스크톱 구분
- 브라우저별 대응
```

```javascript
// Express.js에서 User-Agent 확인
app.get('/api/users', (req, res) => {
  const userAgent = req.headers['user-agent'];

  if (userAgent.includes('Mobile')) {
    res.json({ version: 'mobile' });
  } else {
    res.json({ version: 'desktop' });
  }
});
```

### Accept

```http
Accept: application/json
Accept: text/html
Accept: application/json, text/html, */*

역할: 클라이언트가 받고 싶은 콘텐츠 타입
```

```javascript
// JSON 요청
fetch('/api/users', {
  headers: {
    'Accept': 'application/json'
  }
});

// HTML 요청 (브라우저 기본)
fetch('/page', {
  headers: {
    'Accept': 'text/html'
  }
});
```

```javascript
// 서버: Accept 헤더에 따라 응답 형식 변경
app.get('/api/users', (req, res) => {
  const users = getAllUsers();

  if (req.accepts('json')) {
    res.json(users);
  } else if (req.accepts('xml')) {
    res.type('xml').send(convertToXML(users));
  } else {
    res.status(406).send('Not Acceptable');
  }
});
```

### Accept-Language

```http
Accept-Language: ko-KR, ko;q=0.9, en-US;q=0.8, en;q=0.7

역할: 선호 언어
q: 우선순위 (0-1)
```

```javascript
app.get('/api/greeting', (req, res) => {
  const lang = req.acceptsLanguages('ko', 'en', 'ja');

  const greetings = {
    ko: '안녕하세요',
    en: 'Hello',
    ja: 'こんにちは'
  };

  res.json({ message: greetings[lang] || greetings.en });
});
```

### Authorization ⭐ 매우 중요!

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

역할: 인증 토큰
형식:
- Basic: Basic base64(username:password)
- Bearer: Bearer <token> (JWT 등)
- Digest, OAuth, ...
```

```javascript
// JWT 토큰 전송
const token = localStorage.getItem('token');

fetch('/api/profile', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

```javascript
// 서버: 토큰 검증
app.get('/api/profile', (req, res) => {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const token = authHeader.substring(7);

  try {
    const decoded = jwt.verify(token, SECRET_KEY);
    const user = findUser(decoded.userId);
    res.json(user);
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
});
```

### Content-Type (요청 본문)

```http
Content-Type: application/json
Content-Type: application/x-www-form-urlencoded
Content-Type: multipart/form-data

역할: 요청 본문의 타입
```

```javascript
// JSON 전송
fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'John' })
});

// 폼 데이터
const formData = new FormData();
formData.append('name', 'John');
formData.append('file', fileInput.files[0]);

fetch('/api/upload', {
  method: 'POST',
  body: formData
  // Content-Type은 자동 설정됨 (multipart/form-data)
});
```

### Cookie

```http
Cookie: session_id=abc123; user_id=456; theme=dark

역할: 쿠키 전송
형식: key=value; key2=value2
```

```javascript
// 자동으로 쿠키 전송됨
fetch('/api/profile', {
  credentials: 'include'  // 쿠키 포함
});
```

---

## 8.4 필수 응답 헤더 ⭐

### Content-Type (응답 본문)

```http
Content-Type: application/json; charset=utf-8
Content-Type: text/html; charset=utf-8
Content-Type: image/png

역할: 응답 본문의 타입
```

```javascript
// Express.js
app.get('/api/users', (req, res) => {
  res.type('json');  // 또는 res.json()이 자동 설정
  res.json({ name: 'John' });
});

app.get('/page', (req, res) => {
  res.type('html');
  res.send('<h1>Hello</h1>');
});

app.get('/image', (req, res) => {
  res.type('png');
  res.sendFile('/path/to/image.png');
});
```

### Content-Length

```http
Content-Length: 1234

역할: 응답 본문의 길이 (바이트)
```

### Set-Cookie ⭐ 매우 중요!

```http
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Strict

속성:
- HttpOnly: JavaScript 접근 차단 (XSS 방지)
- Secure: HTTPS에서만 전송
- SameSite: CSRF 방지
- Max-Age: 유효 기간 (초)
- Expires: 만료 날짜
- Domain: 쿠키 유효 도메인
- Path: 쿠키 유효 경로
```

```javascript
// Express.js
app.post('/login', (req, res) => {
  const user = authenticateUser(req.body);

  // 세션 쿠키 설정
  res.cookie('session_id', generateSessionId(), {
    httpOnly: true,    // XSS 방지
    secure: true,      // HTTPS만
    sameSite: 'strict', // CSRF 방지
    maxAge: 24 * 60 * 60 * 1000  // 24시간
  });

  res.json({ message: 'Login successful' });
});
```

### Cache-Control ⭐ 성능 최적화!

```http
Cache-Control: public, max-age=3600
Cache-Control: private, no-cache
Cache-Control: no-store

지시자:
- public: 모든 캐시 가능 (CDN 포함)
- private: 브라우저만 캐시 (CDN X)
- no-cache: 캐시하지만 재검증 필요
- no-store: 절대 캐시 안 함
- max-age=초: 캐시 유효 시간
- must-revalidate: 만료 후 재검증 필수
```

```javascript
// 정적 파일: 장기 캐싱
app.get('/static/logo.png', (req, res) => {
  res.set('Cache-Control', 'public, max-age=31536000'); // 1년
  res.sendFile('/path/to/logo.png');
});

// API 응답: 짧은 캐싱
app.get('/api/users', (req, res) => {
  res.set('Cache-Control', 'public, max-age=60'); // 1분
  res.json(users);
});

// 민감한 데이터: 캐싱 안 함
app.get('/api/private', (req, res) => {
  res.set('Cache-Control', 'private, no-store');
  res.json(sensitiveData);
});
```

### ETag

```http
ETag: "abc123"

역할: 리소스의 고유 식별자 (버전)
용도: 캐시 검증
```

```javascript
// 서버: ETag 생성
const crypto = require('crypto');

app.get('/api/users', (req, res) => {
  const users = getAllUsers();
  const etag = crypto
    .createHash('md5')
    .update(JSON.stringify(users))
    .digest('hex');

  // 클라이언트의 ETag와 비교
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).send(); // Not Modified
  }

  res.set('ETag', etag);
  res.json(users);
});
```

```javascript
// 클라이언트: 조건부 요청
const cachedETag = localStorage.getItem('users-etag');

fetch('/api/users', {
  headers: {
    'If-None-Match': cachedETag
  }
})
  .then(res => {
    if (res.status === 304) {
      // 캐시 사용
      return JSON.parse(localStorage.getItem('users-data'));
    }

    // 새 데이터
    const newETag = res.headers.get('ETag');
    localStorage.setItem('users-etag', newETag);

    return res.json().then(data => {
      localStorage.setItem('users-data', JSON.stringify(data));
      return data;
    });
  });
```

### Location

```http
Location: /api/users/123

역할: 리다이렉트 또는 생성된 리소스 위치
사용: 201 Created, 3xx 리다이렉트
```

```javascript
// 리소스 생성 후
app.post('/api/users', (req, res) => {
  const user = createUser(req.body);

  res.status(201)
     .location(`/api/users/${user.id}`)
     .json(user);
});

// 리다이렉트
app.get('/old-url', (req, res) => {
  res.redirect(301, '/new-url');
  // Location: /new-url 자동 설정
});
```

---

## 8.5 CORS 헤더 ⭐ 매우 중요!

### Access-Control-Allow-Origin

```http
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Origin: *

역할: 어느 출처(Origin)의 요청을 허용할지
```

```javascript
// Express.js CORS 설정
const cors = require('cors');

// 모든 출처 허용 (개발 환경)
app.use(cors());

// 특정 출처만 허용 (운영 환경)
app.use(cors({
  origin: 'https://example.com'
}));

// 여러 출처 허용
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com']
}));

// 동적 출처 검증
app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = ['https://example.com', 'https://app.example.com'];

    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  }
}));
```

### Access-Control-Allow-Methods

```http
Access-Control-Allow-Methods: GET, POST, PUT, DELETE

역할: 허용할 HTTP 메서드
```

### Access-Control-Allow-Headers

```http
Access-Control-Allow-Headers: Content-Type, Authorization

역할: 허용할 요청 헤더
```

### Access-Control-Allow-Credentials

```http
Access-Control-Allow-Credentials: true

역할: 쿠키 전송 허용 여부
```

```javascript
// 쿠키 포함 CORS
app.use(cors({
  origin: 'https://example.com',
  credentials: true  // 쿠키 허용
}));

// 클라이언트
fetch('https://api.example.com/data', {
  credentials: 'include'  // 쿠키 포함
});
```

### Preflight 요청

```
복잡한 요청 (POST + JSON 등) 전에 OPTIONS 요청 먼저 전송

1. Preflight (OPTIONS)
OPTIONS /api/users
Origin: https://example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type

2. Preflight 응답
200 OK
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type

3. 실제 요청 (POST)
POST /api/users
Origin: https://example.com
Content-Type: application/json
```

```javascript
// Express.js Preflight 처리
app.options('*', cors());  // 모든 OPTIONS 요청 허용

// 또는 수동 처리
app.options('/api/users', (req, res) => {
  res.set('Access-Control-Allow-Origin', 'https://example.com');
  res.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  res.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.status(200).send();
});
```

---

## 8.6 보안 헤더 ⭐ 보안 필수!

### X-Content-Type-Options

```http
X-Content-Type-Options: nosniff

역할: MIME 타입 스니핑 방지
설명: 브라우저가 Content-Type 무시하고
     파일 내용으로 타입 추측하는 것 방지
```

### X-Frame-Options

```http
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN

역할: 클릭재킹(Clickjacking) 방지
DENY: iframe 사용 불가
SAMEORIGIN: 같은 출처만 iframe 가능
```

### X-XSS-Protection

```http
X-XSS-Protection: 1; mode=block

역할: XSS 공격 감지 시 페이지 차단
참고: 구형 브라우저용 (현대 브라우저는 CSP 사용)
```

### Strict-Transport-Security (HSTS)

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains

역할: HTTPS 강제
설명: 브라우저가 항상 HTTPS로만 접속하도록
```

### Content-Security-Policy (CSP)

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com

역할: XSS 방지
설명: 어디서 리소스를 로드할 수 있는지 제한
```

```javascript
// Express.js 보안 헤더 설정
const helmet = require('helmet');

app.use(helmet());  // 보안 헤더 자동 설정

// 또는 수동 설정
app.use((req, res, next) => {
  res.set('X-Content-Type-Options', 'nosniff');
  res.set('X-Frame-Options', 'DENY');
  res.set('X-XSS-Protection', '1; mode=block');
  res.set('Strict-Transport-Security', 'max-age=31536000');
  res.set('Content-Security-Policy', "default-src 'self'");
  next();
});
```

---

## 8.7 커스텀 헤더

### X- 접두사 (비표준)

```http
X-Request-ID: abc-123-def-456
X-API-Key: secret-key-123
X-User-ID: 789

역할: 애플리케이션별 커스텀 헤더
참고: X- 접두사는 더 이상 권장되지 않음
```

```javascript
// 요청 ID 추적
app.use((req, res, next) => {
  const requestId = uuid.v4();
  req.requestId = requestId;
  res.set('X-Request-ID', requestId);
  next();
});

// API 키 인증
app.use((req, res, next) => {
  const apiKey = req.headers['x-api-key'];

  if (!apiKey || !isValidApiKey(apiKey)) {
    return res.status(401).json({ error: 'Invalid API key' });
  }

  next();
});
```

---

## 8.8 실전 헤더 설정

### 완전한 Express.js 예시

```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');

const app = express();

// 1. 보안 헤더
app.use(helmet());

// 2. CORS
app.use(cors({
  origin: process.env.ALLOWED_ORIGIN,
  credentials: true
}));

// 3. 압축
app.use(compression());

// 4. JSON 파싱
app.use(express.json());

// 5. 커스텀 헤더
app.use((req, res, next) => {
  res.set('X-Powered-By', 'MyApp');
  res.set('X-Request-ID', uuid.v4());
  next();
});

// 6. API 엔드포인트
app.get('/api/users', (req, res) => {
  const users = getAllUsers();

  res.set('Cache-Control', 'public, max-age=60');
  res.set('ETag', generateETag(users));
  res.json(users);
});

app.post('/api/users', (req, res) => {
  const user = createUser(req.body);

  res.status(201)
     .location(`/api/users/${user.id}`)
     .json(user);
});

app.listen(3000);
```

---

## 8.9 헤더 디버깅

### Chrome DevTools

```
1. F12 → Network 탭
2. 요청 클릭
3. Headers 탭 확인
   - Request Headers (요청 헤더)
   - Response Headers (응답 헤더)
```

### curl

```bash
# 헤더 포함 출력
curl -v https://api.example.com/users

# 특정 헤더 전송
curl -H "Authorization: Bearer token123" \
     -H "Content-Type: application/json" \
     https://api.example.com/users

# 응답 헤더만
curl -I https://api.example.com/users
```

### Node.js 로깅

```javascript
// 모든 헤더 로깅
app.use((req, res, next) => {
  console.log('Request Headers:', req.headers);

  const originalSend = res.send;
  res.send = function(data) {
    console.log('Response Headers:', res.getHeaders());
    originalSend.call(this, data);
  };

  next();
});
```

---

## 핵심 요약

### 필수 요청 헤더
```
Host: api.example.com (필수!)
Authorization: Bearer token
Content-Type: application/json
Accept: application/json
Cookie: session_id=abc123
```

### 필수 응답 헤더
```
Content-Type: application/json
Set-Cookie: session_id=xyz; HttpOnly; Secure
Cache-Control: public, max-age=3600
ETag: "abc123"
```

### CORS 헤더
```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```

### 보안 헤더
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000
Content-Security-Policy: default-src 'self'
```

---

## 다음 단계

HTTP 헤더를 마스터했으니, 이제 **HTTPS와 SSL/TLS**를 배워봅시다!

👉 [Chapter 9. HTTPS와 SSL/TLS](09-HTTPS-SSL-TLS.md)

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 3-4시간
**대상**: 모든 웹개발자
**중요도**: ⭐⭐⭐ 매우 중요!