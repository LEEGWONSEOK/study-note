# Chapter 22. CORS 완전정복

## 22.1 CORS란?

**CORS (Cross-Origin Resource Sharing)**는 **다른 출처(Origin)의 리소스에 접근할 수 있는 권한을 부여**하는 메커니즘입니다.

### 비유: 아파트 경비

```
┌──────────────────────────────────────┐
│  https://example.com (내 아파트)      │
│                                      │
│  "저희 아파트 주민만 출입 가능합니다" │
│                                      │
│  ❌ https://hacker.com 차단          │
│  ✅ https://example.com 허용         │
└──────────────────────────────────────┘
```

---

## 22.2 출처(Origin)란?

**출처 = 프로토콜 + 도메인 + 포트**

```
https://www.example.com:443/api/users
│     │  │              │   │
│     │  │              │   └─ 경로 (출처 X)
│     │  │              └─ 포트 (출처 O)
│     │  └─ 도메인 (출처 O)
│     └─ 서브도메인 (출처 O)
└─ 프로토콜 (출처 O)

출처: https://www.example.com:443
```

### 같은 출처 vs 다른 출처

```
기준: https://www.example.com

┌────────────────────────────────┬──────────┐
│  URL                           │  결과     │
├────────────────────────────────┼──────────┤
│ https://www.example.com/page   │ ✅ 같음   │
│ https://www.example.com:443/   │ ✅ 같음   │
│                                │          │
│ http://www.example.com         │ ❌ 다름   │
│ (프로토콜 다름)                 │          │
│                                │          │
│ https://api.example.com        │ ❌ 다름   │
│ (서브도메인 다름)               │          │
│                                │          │
│ https://www.example.com:8080   │ ❌ 다름   │
│ (포트 다름)                     │          │
│                                │          │
│ https://example.com            │ ❌ 다름   │
│ (www 유무)                      │          │
└────────────────────────────────┴──────────┘
```

---

## 22.3 왜 CORS가 필요한가?

### SOP (Same-Origin Policy) - 동일 출처 정책

브라우저의 기본 보안 정책: **다른 출처의 리소스 접근 차단**

```
┌─────────────────────────────────────────┐
│  https://mybank.com                     │
│  (은행 사이트)                           │
│                                         │
│  로그인됨: 쿠키에 인증 정보 있음         │
└─────────────────────────────────────────┘

만약 SOP가 없다면?

┌─────────────────────────────────────────┐
│  https://evil.com                       │
│  (악성 사이트)                           │
│                                         │
│  <script>                               │
│    // mybank.com의 쿠키로 요청!        │
│    fetch('https://mybank.com/transfer', {│
│      method: 'POST',                    │
│      body: { to: 'hacker', amount: 1000 }│
│    })                                   │
│  </script>                              │
└─────────────────────────────────────────┘

→ 사용자 돈이 해커에게! ❌
```

**SOP가 막아줌:**

```
https://evil.com 에서
https://mybank.com 으로 요청
→ 브라우저가 차단! ✅
```

---

## 22.4 CORS 에러 발생 시나리오

### 시나리오 1: 로컬 개발

```javascript
// 프론트엔드: http://localhost:3000
fetch('http://localhost:8000/api/users')

// ❌ CORS 에러 발생!
// Access to fetch at 'http://localhost:8000/api/users'
// from origin 'http://localhost:3000' has been blocked by CORS policy
```

**원인:**
- 프론트: `http://localhost:3000`
- 백엔드: `http://localhost:8000`
- 포트가 다름 → 다른 출처!

### 시나리오 2: 배포 환경

```javascript
// 프론트엔드: https://example.com
fetch('https://api.example.com/users')

// ❌ CORS 에러!
```

**원인:**
- 프론트: `https://example.com`
- API: `https://api.example.com`
- 서브도메인이 다름 → 다른 출처!

---

## 22.5 CORS 동작 방식

### 단순 요청 (Simple Request)

```
조건:
1. 메서드: GET, HEAD, POST
2. 헤더: Accept, Accept-Language, Content-Language,
        Content-Type (특정 값만)
3. Content-Type:
   - application/x-www-form-urlencoded
   - multipart/form-data
   - text/plain
```

```
┌──────────────┐                   ┌──────────────┐
│ 프론트엔드    │                   │  백엔드       │
│ example.com  │                   │ api.exam.com │
└──────┬───────┘                   └──────┬───────┘
       │                                  │
       │ 1. GET /api/users                │
       │    Origin: https://example.com   │
       ├─────────────────────────────────►│
       │                                  │
       │ 2. 200 OK                        │
       │    Access-Control-Allow-Origin:  │
       │    https://example.com           │
       │◄─────────────────────────────────┤
       │                                  │
    ✅ 성공
```

### 사전 요청 (Preflight Request) ⭐ 중요!

```
복잡한 요청 (PUT, DELETE, 커스텀 헤더 등)은
사전 요청을 먼저 보냄
```

```
┌──────────────┐                   ┌──────────────┐
│ 프론트엔드    │                   │  백엔드       │
└──────┬───────┘                   └──────┬───────┘
       │                                  │
       │ 1. OPTIONS /api/users (Preflight)│
       │    Origin: https://example.com   │
       │    Access-Control-Request-Method:│
       │    POST                          │
       │    Access-Control-Request-Headers:│
       │    Content-Type, Authorization   │
       ├─────────────────────────────────►│
       │                                  │
       │ 2. 200 OK (Preflight 응답)       │
       │    Access-Control-Allow-Origin:  │
       │    https://example.com           │
       │    Access-Control-Allow-Methods: │
       │    GET, POST, PUT, DELETE        │
       │    Access-Control-Allow-Headers: │
       │    Content-Type, Authorization   │
       │◄─────────────────────────────────┤
       │                                  │
       │ 3. POST /api/users (실제 요청)   │
       │    Origin: https://example.com   │
       │    Content-Type: application/json│
       │    Authorization: Bearer token   │
       ├─────────────────────────────────►│
       │                                  │
       │ 4. 201 Created                   │
       │◄─────────────────────────────────┤
       │                                  │
    ✅ 성공
```

---

## 22.6 CORS 해결 방법

### 방법 1: 백엔드에서 CORS 허용 (권장) ⭐

#### Node.js/Express

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// 1. 모든 출처 허용 (개발 환경)
app.use(cors());

// 2. 특정 출처만 허용 (운영 환경)
app.use(cors({
  origin: 'https://example.com'
}));

// 3. 여러 출처 허용
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com']
}));

// 4. 동적 출처 검증
app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://example.com',
      'https://app.example.com'
    ];

    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true  // 쿠키 허용
}));

// 5. 특정 라우트만
app.get('/api/public', cors(), (req, res) => {
  res.json({ message: 'Public data' });
});

app.listen(8000);
```

#### 수동 설정

```javascript
app.use((req, res, next) => {
  // 모든 출처 허용
  res.header('Access-Control-Allow-Origin', '*');

  // 특정 출처만
  res.header('Access-Control-Allow-Origin', 'https://example.com');

  // 허용 메서드
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');

  // 허용 헤더
  res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization');

  // 쿠키 허용
  res.header('Access-Control-Allow-Credentials', 'true');

  // Preflight 캐싱 (초)
  res.header('Access-Control-Max-Age', '86400');

  // OPTIONS 요청 처리
  if (req.method === 'OPTIONS') {
    return res.sendStatus(200);
  }

  next();
});
```

### 방법 2: 프록시 사용 (개발 환경)

#### React (package.json)

```json
{
  "proxy": "http://localhost:8000"
}
```

```javascript
// 이제 상대 경로로 요청 가능
fetch('/api/users')  // → http://localhost:8000/api/users
```

#### Vite (vite.config.js)

```javascript
export default {
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true
      }
    }
  }
}
```

#### Next.js (next.config.js)

```javascript
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'http://localhost:8000/api/:path*'
      }
    ];
  }
};
```

### 방법 3: NGINX 리버스 프록시

```nginx
server {
    listen 80;
    server_name example.com;

    # 프론트엔드
    location / {
        proxy_pass http://localhost:3000;
    }

    # API (같은 도메인으로!)
    location /api {
        proxy_pass http://localhost:8000;

        # CORS 헤더 추가
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "Content-Type, Authorization";

        # OPTIONS 요청 처리
        if ($request_method = OPTIONS) {
            return 204;
        }
    }
}
```

---

## 22.7 쿠키와 CORS (credentials)

### 쿠키 전송 설정

```javascript
// 프론트엔드
fetch('https://api.example.com/profile', {
  credentials: 'include'  // 쿠키 포함
})

// 또는 axios
axios.get('https://api.example.com/profile', {
  withCredentials: true
})
```

### 백엔드 설정

```javascript
app.use(cors({
  origin: 'https://example.com',  // * 불가! 특정 출처 필수
  credentials: true
}));

app.post('/login', (req, res) => {
  // 쿠키 설정
  res.cookie('session_id', 'abc123', {
    httpOnly: true,
    secure: true,
    sameSite: 'none'  // CORS 환경에서는 'none' 필수
  });

  res.json({ message: 'Login successful' });
});
```

### 주의사항

```
credentials: true 사용 시:

❌ Access-Control-Allow-Origin: *  (불가!)
✅ Access-Control-Allow-Origin: https://example.com

❌ SameSite: strict 또는 lax
✅ SameSite: none; Secure
```

---

## 22.8 CORS 에러 디버깅

### Chrome DevTools

```
1. F12 → Console 탭
   → CORS 에러 메시지 확인

2. Network 탭
   → 실패한 요청 클릭
   → Headers 탭 확인
     - Request Headers: Origin
     - Response Headers: Access-Control-*

3. Preflight 요청 확인
   → OPTIONS 메서드 요청 찾기
```

### 일반적인 에러 메시지

```
1. "No 'Access-Control-Allow-Origin' header"
   → 서버에서 CORS 헤더를 안 보냄

2. "The 'Access-Control-Allow-Origin' header contains
    the invalid value ''"
   → 빈 값 전송됨

3. "Wildcard '*' cannot be used when credentials flag is true"
   → credentials: true인데 origin: '*' 사용

4. "Request header field Authorization is not allowed"
   → Access-Control-Allow-Headers에 Authorization 없음

5. "Method POST is not allowed"
   → Access-Control-Allow-Methods에 POST 없음
```

---

## 22.9 실전 시나리오

### 시나리오 1: 마이크로서비스

```
프론트엔드: https://app.example.com
User API:   https://user-api.example.com
Order API:  https://order-api.example.com
Product API: https://product-api.example.com

각 API마다 CORS 설정 필요!
```

```javascript
// 각 마이크로서비스
app.use(cors({
  origin: [
    'https://app.example.com',
    'https://admin.example.com'
  ],
  credentials: true
}));
```

### 시나리오 2: 모바일 앱 + 웹

```javascript
app.use(cors({
  origin: (origin, callback) => {
    // 웹
    if (origin === 'https://example.com') {
      return callback(null, true);
    }

    // 모바일 앱 (origin 없음)
    if (!origin) {
      return callback(null, true);
    }

    callback(new Error('Not allowed'));
  }
}));
```

### 시나리오 3: CDN 사용

```javascript
// CDN에서 프론트엔드 제공
// https://cdn.cloudflare.com/app.js

// API 서버에서 CDN 출처 허용
app.use(cors({
  origin: 'https://cdn.cloudflare.com'
}));
```

---

## 22.10 보안 고려사항

### 1. origin: '*' 사용 주의

```javascript
// ❌ 나쁜 예: 모든 출처 허용
app.use(cors({ origin: '*' }))

// ✅ 좋은 예: 특정 출처만
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS.split(',')
}))
```

### 2. credentials 신중하게

```javascript
// 인증이 필요한 엔드포인트만
app.get('/api/public', cors({ origin: '*' }), handler);

app.get('/api/private', cors({
  origin: 'https://example.com',
  credentials: true
}), authMiddleware, handler);
```

### 3. 환경별 설정

```javascript
const corsOptions = {
  origin: process.env.NODE_ENV === 'production'
    ? 'https://example.com'
    : 'http://localhost:3000',
  credentials: true
};

app.use(cors(corsOptions));
```

---

## 핵심 요약

### CORS란?
```
다른 출처의 리소스 접근을 제어하는 메커니즘
브라우저의 보안 정책 (SOP)을 완화
```

### 출처 (Origin)
```
프로토콜 + 도메인 + 포트
하나라도 다르면 다른 출처!
```

### CORS 해결
```
백엔드:
- Access-Control-Allow-Origin 헤더 설정
- cors 미들웨어 사용

프론트엔드:
- 프록시 사용 (개발 환경)
- credentials: 'include' (쿠키 전송 시)
```

### Preflight 요청
```
복잡한 요청 전에 OPTIONS 요청 먼저 전송
서버가 허용하면 실제 요청 전송
```

---

## 다음 단계

CORS를 완전히 이해했으니, 이제 **인증과 인가**를 배워봅시다!

👉 [Chapter 23. 인증과 인가](23-인증과-인가.md)

---

## 실습 과제

### 과제 1: CORS 에러 재현 및 해결

```javascript
// 1. 프론트엔드 (http://localhost:3000)
fetch('http://localhost:8000/api/users')
  .then(res => res.json())
  .catch(err => console.error(err));

// 2. 백엔드 (http://localhost:8000)
// CORS 설정 없이 실행 → 에러 확인

// 3. CORS 미들웨어 추가 → 해결 확인
app.use(cors({ origin: 'http://localhost:3000' }));
```

### 과제 2: Preflight 요청 확인

```javascript
// 복잡한 요청으로 Preflight 발생시키기
fetch('http://localhost:8000/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer token'
  },
  body: JSON.stringify({ name: 'John' })
});

// Chrome DevTools → Network 탭에서
// OPTIONS 요청 확인
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 2-3시간
**대상**: 모든 웹개발자
**중요도**: ⭐⭐⭐ 매우 중요!