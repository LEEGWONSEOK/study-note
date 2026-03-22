# Chapter 6. HTTP 기초

## 6.1 HTTP란?

**HTTP (HyperText Transfer Protocol)**는 웹에서 데이터를 주고받기 위한 **프로토콜**입니다.

### 정의

```
HyperText Transfer Protocol

HyperText: 링크로 연결된 문서 (HTML)
Transfer: 전송
Protocol: 규칙, 약속

→ 웹 문서를 전송하는 규칙
```

### 비유: 식당 주문

```
손님(클라이언트)         웨이터(HTTP)         주방(서버)
    │                      │                    │
    │──── "파스타 주세요" ───►│                    │
    │      (HTTP 요청)       │──── 주문 전달 ────►│
    │                       │                    │
    │                       │◄─── 파스타 완성 ───│
    │◄──── 파스타 전달 ─────│                    │
    │      (HTTP 응답)       │                    │
```

---

## 6.2 HTTP의 특징

### 1. 클라이언트-서버 구조

```
[클라이언트]                [서버]
- 브라우저                  - 웹 서버
- 모바일 앱                 - API 서버
- Postman                  - 백엔드 애플리케이션
    │                          │
    │───── 요청 (Request) ─────►│
    │                          │
    │◄──── 응답 (Response) ────│
```

### 2. 무상태 (Stateless) ⭐ 중요!

```
각 요청은 독립적!
서버는 이전 요청을 기억하지 않음

요청 1: "로그인해주세요"
→ 서버: "OK, 로그인 완료"

요청 2: "내 프로필 보여주세요"
→ 서버: "누구세요?" ← 이전 로그인 기억 안 함!
```

**해결책: 쿠키/세션/토큰으로 상태 유지**

```javascript
// 매 요청마다 인증 정보 포함
fetch('/api/profile', {
  headers: {
    'Authorization': 'Bearer eyJhbGciOiJ...'  // JWT 토큰
  }
});
```

### 3. 비연결성 (Connectionless)

```
요청 → 응답 → 연결 종료

장점:
- 서버 리소스 절약
- 많은 클라이언트 동시 처리 가능

단점:
- 매번 연결 수립 비용 (TCP 3-Way Handshake)

해결:
- HTTP Keep-Alive (연결 재사용)
- HTTP/2 (단일 연결로 다중 요청)
```

### 4. 확장 가능

```
HTTP 헤더로 기능 확장 가능

Authorization: Bearer token
Content-Type: application/json
Cache-Control: max-age=3600
X-Custom-Header: custom-value
```

---

## 6.3 HTTP 요청 (Request) 구조

### 전체 구조

```
┌─────────────────────────────────────────┐
│  Request Line (요청 라인)                │
│  GET /users/123 HTTP/1.1                │
├─────────────────────────────────────────┤
│  Headers (헤더)                          │
│  Host: api.example.com                  │
│  User-Agent: Mozilla/5.0...             │
│  Accept: application/json               │
│  Authorization: Bearer token123         │
├─────────────────────────────────────────┤
│  (빈 줄)                                 │
├─────────────────────────────────────────┤
│  Body (본문) - 선택적                    │
│  {"name":"John","email":"john@x.com"}   │
└─────────────────────────────────────────┘
```

### 1. Request Line

```
메서드 + 경로 + HTTP 버전

GET /api/users HTTP/1.1
│   │          │
│   │          └─ HTTP 버전
│   └─ 요청 경로 (URI)
└─ HTTP 메서드
```

### 2. Request Headers

```
Host: api.example.com              # 필수! 도메인
User-Agent: Mozilla/5.0...         # 클라이언트 정보
Accept: application/json           # 받고 싶은 형식
Content-Type: application/json     # 보내는 데이터 형식
Content-Length: 56                 # 본문 길이
Authorization: Bearer token123     # 인증 정보
Cookie: session_id=abc123          # 쿠키
```

### 3. Request Body (선택적)

```json
// POST, PUT, PATCH 요청 시 데이터 전송

{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}
```

---

## 6.4 HTTP 응답 (Response) 구조

### 전체 구조

```
┌─────────────────────────────────────────┐
│  Status Line (상태 라인)                 │
│  HTTP/1.1 200 OK                        │
├─────────────────────────────────────────┤
│  Headers (헤더)                          │
│  Content-Type: application/json         │
│  Content-Length: 125                    │
│  Cache-Control: max-age=3600            │
│  Set-Cookie: session_id=xyz789          │
├─────────────────────────────────────────┤
│  (빈 줄)                                 │
├─────────────────────────────────────────┤
│  Body (본문)                             │
│  {"id":123,"name":"John","email":"..."}│
└─────────────────────────────────────────┘
```

### 1. Status Line

```
HTTP/1.1 200 OK
│        │   │
│        │   └─ 상태 메시지
│        └─ 상태 코드
└─ HTTP 버전
```

### 2. Response Headers

```
Content-Type: application/json     # 응답 데이터 형식
Content-Length: 1234               # 응답 본문 길이
Cache-Control: max-age=3600        # 캐시 설정
Set-Cookie: session_id=xyz789      # 쿠키 설정
Location: /users/124               # 리다이렉트 위치
ETag: "abc123"                     # 캐시 검증
```

### 3. Response Body

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-01T00:00:00Z"
}
```

---

## 6.5 HTTP 메서드 ⭐ 필수 암기!

### 주요 메서드

```
┌────────┬─────────────┬──────────┬─────────┐
│ 메서드  │   의미       │ 안전성   │ 멱등성  │
├────────┼─────────────┼──────────┼─────────┤
│  GET   │ 조회         │   ✅     │   ✅    │
│  POST  │ 생성         │   ❌     │   ❌    │
│  PUT   │ 전체 수정    │   ❌     │   ✅    │
│  PATCH │ 부분 수정    │   ❌     │   ❌    │
│ DELETE │ 삭제         │   ❌     │   ✅    │
│  HEAD  │ 헤더만 조회  │   ✅     │   ✅    │
│OPTIONS │ 지원 메서드  │   ✅     │   ✅    │
└────────┴─────────────┴──────────┴─────────┘

안전성: 서버 상태를 변경하지 않음
멱등성: 여러 번 실행해도 결과가 같음
```

### GET - 조회

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com

특징:
- 데이터 조회만
- 서버 상태 변경 ❌
- Body 없음 (쿼리 파라미터 사용)
- 캐시 가능
- 브라우저 히스토리에 남음
```

```javascript
// JavaScript 예시
const response = await fetch('/api/users/123');
const user = await response.json();

// 쿼리 파라미터
fetch('/api/users?page=1&limit=10');
```

### POST - 생성

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}

특징:
- 새 리소스 생성
- 서버 상태 변경 ✅
- Body에 데이터 전송
- 캐시 불가
- 멱등성 ❌ (매번 새 리소스 생성)
```

```javascript
const response = await fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com'
  })
});
```

### PUT - 전체 수정

```http
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Updated",
  "email": "john.new@example.com",
  "age": 31
}

특징:
- 리소스 전체 교체
- 멱등성 ✅ (같은 요청 반복해도 결과 동일)
- 없으면 생성 가능 (upsert)
```

```javascript
// 전체 필드 전송 (일부만 보내면 나머지는 null/삭제)
await fetch('/api/users/123', {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Updated',
    email: 'john.new@example.com',
    age: 31
  })
});
```

### PATCH - 부분 수정

```http
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Updated"
}

특징:
- 리소스 일부만 수정
- 보낸 필드만 업데이트
- PUT보다 효율적
```

```javascript
// 이름만 수정 (나머지 필드는 그대로)
await fetch('/api/users/123', {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Updated'
  })
});
```

### DELETE - 삭제

```http
DELETE /api/users/123 HTTP/1.1
Host: api.example.com

특징:
- 리소스 삭제
- 멱등성 ✅ (이미 삭제된 리소스 삭제해도 결과 동일)
- Body 보통 없음
```

```javascript
await fetch('/api/users/123', {
  method: 'DELETE'
});
```

---

## 6.6 HTTP 상태 코드 ⭐ 필수 암기!

### 분류

```
1xx: 정보성 (거의 안 씀)
2xx: 성공 ✅
3xx: 리다이렉션 🔄
4xx: 클라이언트 에러 ❌
5xx: 서버 에러 💥
```

### 2xx - 성공

```
200 OK
- 요청 성공
- GET, PUT, PATCH, DELETE 성공 시

201 Created
- 리소스 생성 성공
- POST 성공 시
- Location 헤더에 새 리소스 URL

202 Accepted
- 요청 접수, 처리 중
- 비동기 작업

204 No Content
- 성공했지만 응답 본문 없음
- DELETE 성공 시 자주 사용
```

```javascript
// 201 Created 예시
app.post('/api/users', (req, res) => {
  const user = createUser(req.body);
  res.status(201)
     .location(`/api/users/${user.id}`)
     .json(user);
});

// 204 No Content 예시
app.delete('/api/users/:id', (req, res) => {
  deleteUser(req.params.id);
  res.status(204).send();
});
```

### 3xx - 리다이렉션

```
301 Moved Permanently
- 영구 이동
- SEO에 영향

302 Found
- 임시 이동

304 Not Modified
- 캐시된 버전 사용
- ETag/Last-Modified 활용
```

```javascript
// 301 리다이렉션
app.get('/old-url', (req, res) => {
  res.redirect(301, '/new-url');
});

// 304 캐시 활용
app.get('/api/users', (req, res) => {
  const etag = generateETag(users);

  if (req.headers['if-none-match'] === etag) {
    return res.status(304).send();  // 캐시 사용
  }

  res.set('ETag', etag).json(users);
});
```

### 4xx - 클라이언트 에러

```
400 Bad Request
- 잘못된 요청 (문법 오류)
- 유효성 검증 실패

401 Unauthorized
- 인증 필요 (로그인 필요)
- Authorization 헤더 없음

403 Forbidden
- 권한 없음 (인증은 되었지만 접근 금지)
- 예: 관리자만 접근 가능

404 Not Found
- 리소스 없음
- 잘못된 URL

405 Method Not Allowed
- 허용되지 않은 HTTP 메서드
- 예: POST만 허용하는데 GET 요청

409 Conflict
- 리소스 충돌
- 예: 이미 존재하는 이메일로 가입 시도

422 Unprocessable Entity
- 문법은 맞지만 의미상 오류
- 예: 유효성 검증 실패

429 Too Many Requests
- 요청 횟수 초과
- Rate Limiting
```

```javascript
// 400 Bad Request
if (!req.body.email) {
  return res.status(400).json({
    error: 'Email is required'
  });
}

// 401 Unauthorized
if (!req.headers.authorization) {
  return res.status(401).json({
    error: 'Authentication required'
  });
}

// 403 Forbidden
if (user.role !== 'admin') {
  return res.status(403).json({
    error: 'Admin access required'
  });
}

// 404 Not Found
const user = findUser(id);
if (!user) {
  return res.status(404).json({
    error: 'User not found'
  });
}

// 409 Conflict
const existingUser = findUserByEmail(email);
if (existingUser) {
  return res.status(409).json({
    error: 'Email already exists'
  });
}

// 429 Too Many Requests
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15분
  max: 100,  // 최대 100회
  statusCode: 429,
  message: 'Too many requests'
});
app.use('/api/', limiter);
```

### 5xx - 서버 에러

```
500 Internal Server Error
- 서버 내부 오류
- 예외 처리되지 않은 에러

502 Bad Gateway
- 게이트웨이/프록시 에러
- 업스트림 서버 응답 없음

503 Service Unavailable
- 서버 일시적 사용 불가
- 과부하, 유지보수

504 Gateway Timeout
- 게이트웨이 타임아웃
- 업스트림 서버 응답 지연
```

```javascript
// 500 Internal Server Error
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    error: 'Internal Server Error'
  });
});

// 503 Service Unavailable
app.use((req, res, next) => {
  if (isMaintenanceMode) {
    return res.status(503).json({
      error: 'Service temporarily unavailable'
    });
  }
  next();
});
```

---

## 6.7 URL 구조

### 전체 구조

```
https://www.example.com:443/api/users/123?page=1&limit=10#section1
│    │  │               │   │            │              │
│    │  │               │   │            │              └─ Fragment (프래그먼트)
│    │  │               │   │            └─ Query String (쿼리 문자열)
│    │  │               │   └─ Path (경로)
│    │  │               └─ Port (포트)
│    │  └─ Domain (도메인)
│    └─ Subdomain (서브도메인)
└─ Scheme/Protocol (프로토콜)
```

### 각 부분 설명

```
1. Scheme (프로토콜)
   http:// 또는 https://

2. Subdomain (서브도메인)
   www, api, admin 등

3. Domain (도메인)
   example.com

4. Port (포트)
   :80 (HTTP 기본), :443 (HTTPS 기본)
   생략 가능

5. Path (경로)
   /api/users/123

6. Query String (쿼리 문자열)
   ?key=value&key2=value2
   - 필터링, 정렬, 페이지네이션

7. Fragment (프래그먼트)
   #section1
   - 페이지 내 특정 위치
   - 서버로 전송 안 됨 (브라우저에서만 사용)
```

### JavaScript에서 URL 다루기

```javascript
// URL 생성
const url = new URL('https://api.example.com/users');
url.searchParams.append('page', '1');
url.searchParams.append('limit', '10');

console.log(url.toString());
// https://api.example.com/users?page=1&limit=10

// URL 파싱
const parsed = new URL('https://api.example.com:443/users/123?page=1#top');
console.log(parsed.protocol);  // https:
console.log(parsed.hostname);  // api.example.com
console.log(parsed.port);      // 443
console.log(parsed.pathname);  // /users/123
console.log(parsed.search);    // ?page=1
console.log(parsed.hash);      // #top

// Query String 다루기
const params = new URLSearchParams(parsed.search);
console.log(params.get('page'));  // 1
```

---

## 6.8 실전 HTTP 요청/응답 예시

### REST API 전체 흐름

```javascript
// 1. 사용자 목록 조회
GET /api/users?page=1&limit=10
Response: 200 OK
[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]

// 2. 사용자 생성
POST /api/users
{
  "name": "Charlie",
  "email": "charlie@example.com"
}
Response: 201 Created
Location: /api/users/3
{
  "id": 3,
  "name": "Charlie",
  "email": "charlie@example.com"
}

// 3. 사용자 조회
GET /api/users/3
Response: 200 OK
{
  "id": 3,
  "name": "Charlie",
  "email": "charlie@example.com"
}

// 4. 사용자 수정 (전체)
PUT /api/users/3
{
  "name": "Charlie Updated",
  "email": "charlie.new@example.com"
}
Response: 200 OK

// 5. 사용자 수정 (부분)
PATCH /api/users/3
{
  "name": "Charlie Final"
}
Response: 200 OK

// 6. 사용자 삭제
DELETE /api/users/3
Response: 204 No Content
```

### Express.js 서버 구현

```javascript
const express = require('express');
const app = express();

app.use(express.json());

let users = [
  { id: 1, name: 'Alice', email: 'alice@example.com' },
  { id: 2, name: 'Bob', email: 'bob@example.com' }
];

// GET /api/users - 목록 조회
app.get('/api/users', (req, res) => {
  const { page = 1, limit = 10 } = req.query;
  const start = (page - 1) * limit;
  const end = start + parseInt(limit);

  res.json({
    data: users.slice(start, end),
    page: parseInt(page),
    total: users.length
  });
});

// GET /api/users/:id - 단일 조회
app.get('/api/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));

  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }

  res.json(user);
});

// POST /api/users - 생성
app.post('/api/users', (req, res) => {
  const { name, email } = req.body;

  // 유효성 검증
  if (!name || !email) {
    return res.status(400).json({
      error: 'Name and email are required'
    });
  }

  // 중복 확인
  const exists = users.find(u => u.email === email);
  if (exists) {
    return res.status(409).json({
      error: 'Email already exists'
    });
  }

  const newUser = {
    id: users.length + 1,
    name,
    email
  };

  users.push(newUser);

  res.status(201)
     .location(`/api/users/${newUser.id}`)
     .json(newUser);
});

// PUT /api/users/:id - 전체 수정
app.put('/api/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = users.findIndex(u => u.id === id);

  if (index === -1) {
    return res.status(404).json({ error: 'User not found' });
  }

  users[index] = {
    id,
    name: req.body.name,
    email: req.body.email
  };

  res.json(users[index]);
});

// PATCH /api/users/:id - 부분 수정
app.patch('/api/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const user = users.find(u => u.id === id);

  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }

  // 보낸 필드만 업데이트
  if (req.body.name) user.name = req.body.name;
  if (req.body.email) user.email = req.body.email;

  res.json(user);
});

// DELETE /api/users/:id - 삭제
app.delete('/api/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = users.findIndex(u => u.id === id);

  if (index === -1) {
    return res.status(404).json({ error: 'User not found' });
  }

  users.splice(index, 1);
  res.status(204).send();
});

app.listen(3000, () => {
  console.log('서버 시작: http://localhost:3000');
});
```

---

## 6.9 HTTP vs HTTPS

### 차이점

```
┌──────────────┬─────────────────┬─────────────────┐
│    항목       │      HTTP       │     HTTPS       │
├──────────────┼─────────────────┼─────────────────┤
│  포트         │       80        │      443        │
│  보안         │  평문 전송      │  암호화 전송    │
│  속도         │  빠름           │  약간 느림      │
│  인증서       │  불필요         │  SSL/TLS 필요   │
│  SEO          │  불리           │  유리           │
│  신뢰         │  낮음           │  높음           │
└──────────────┴─────────────────┴─────────────────┘
```

### HTTP 보안 문제

```
[클라이언트] ─── 평문 전송 ───► [서버]
                ↓
            중간에서 도청 가능!
                ↓
        비밀번호, 카드번호 노출
```

### HTTPS 암호화

```
[클라이언트] ─── 암호화 ───► [서버]
                ↓
        해독 불가능한 암호문
                ↓
            안전!
```

---

## 핵심 요약

### HTTP 특징
```
- 클라이언트-서버 구조
- 무상태 (Stateless)
- 비연결성 (Connectionless)
- 확장 가능 (헤더로 기능 추가)
```

### HTTP 메서드
```
GET: 조회
POST: 생성
PUT: 전체 수정
PATCH: 부분 수정
DELETE: 삭제
```

### HTTP 상태 코드
```
2xx: 성공 (200, 201, 204)
3xx: 리다이렉션 (301, 302, 304)
4xx: 클라이언트 에러 (400, 401, 403, 404)
5xx: 서버 에러 (500, 502, 503)
```

### HTTP vs HTTPS
```
HTTP: 평문, 빠름, 불안전
HTTPS: 암호화, 안전, 현대 표준
```

---

## 다음 단계

HTTP 기초를 이해했으니, 이제 **HTTP 메서드와 상태코드**를 더 깊이 배워봅시다!

👉 [Chapter 7. HTTP 메서드와 상태코드](07-메서드와-상태코드.md)

---

## 실습 과제

### 과제 1: Chrome DevTools로 HTTP 분석

```
1. Chrome DevTools (F12) → Network 탭
2. https://www.github.com 접속
3. 첫 번째 요청 클릭

확인 사항:
- Request Method: GET
- Status Code: 200
- Request Headers에서 User-Agent 찾기
- Response Headers에서 Content-Type 찾기
```

### 과제 2: curl로 HTTP 요청

```bash
# GET 요청
curl -v https://api.github.com/users/github

# POST 요청
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}'

# 응답에서 어떤 정보를 확인할 수 있나요?
```

### 과제 3: REST API 서버 만들기

위의 Express.js 예시 코드를 실행하고:

```bash
# 1. 사용자 목록 조회
curl http://localhost:3000/api/users

# 2. 사용자 생성
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com"}'

# 3. 사용자 조회
curl http://localhost:3000/api/users/3

# 4. 사용자 수정
curl -X PATCH http://localhost:3000/api/users/3 \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie Updated"}'

# 5. 사용자 삭제
curl -X DELETE http://localhost:3000/api/users/3
```

---

**작성일**: 2024년
**난이도**: ⭐⭐ 초급
**예상 학습 시간**: 2-3시간
**대상**: 모든 웹개발자