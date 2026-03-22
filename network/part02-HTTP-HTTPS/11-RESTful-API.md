# Chapter 11. RESTful API 설계

## 11.1 REST란?

### 정의

**REST** (Representational State Transfer)

```
아키텍처 스타일:
웹의 장점을 최대한 활용하는 API 설계 원칙

2000년 Roy Fielding의 박사논문에서 처음 소개
```

### 비유: 도서관 시스템

```
도서관 (서버):
- 책(리소스)을 관리
- 사서(API)가 요청 처리

이용자 (클라이언트):
"책 목록 보여주세요" → GET /books
"이 책 빌려주세요" → POST /books/123/borrow
"책 반납합니다" → DELETE /books/123/borrow
"책 정보 수정" → PUT /books/123

→ 명확하고 일관된 규칙!
```

---

## 11.2 REST의 6가지 원칙

### 1. Client-Server (클라이언트-서버 구조)

```
역할 분리:
┌─────────────┐         ┌─────────────┐
│ 클라이언트   │         │   서버      │
│ (UI 담당)   │ ←────→  │ (데이터 담당)│
└─────────────┘         └─────────────┘

장점:
- 독립적 개발 가능
- 각자 진화 가능
```

### 2. Stateless (무상태)

```
서버는 클라이언트 상태를 저장하지 않음

❌ Stateful:
요청 1: "로그인: 철수"
요청 2: "내 정보 보여줘" (서버가 '철수'를 기억)

✅ Stateless:
요청 1: "로그인: 철수" → 토큰 발급
요청 2: "내 정보 보여줘 + 토큰" (매번 인증 정보 포함)

장점:
- 확장성 (서버 추가 쉬움)
- 신뢰성 (서버 장애 시 영향 적음)
```

### 3. Cacheable (캐시 가능)

```
응답은 캐시 가능 여부를 명시

HTTP 헤더 사용:
Cache-Control: public, max-age=3600
ETag: "abc123"

GET /api/users/1
→ 1시간 동안 캐시 가능
→ 서버 부하 감소, 응답 속도 증가
```

### 4. Uniform Interface (일관된 인터페이스)

```
URI로 리소스 식별:
/users/123
/posts/456
/comments/789

HTTP 메서드로 행위 표현:
GET: 조회
POST: 생성
PUT: 수정
DELETE: 삭제

→ 예측 가능하고 학습하기 쉬움
```

### 5. Layered System (계층 구조)

```
클라이언트는 중간 서버를 모름

┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│클라이언트│→│   Proxy  │→│Load Balancer│→│   서버   │
└──────────┘   └──────────┘   └──────────┘   └──────────┘

장점:
- 보안 (방화벽)
- 캐싱 (CDN)
- 로드밸런싱
```

### 6. Code on Demand (선택)

```
서버가 실행 가능한 코드를 전송

예: JavaScript
서버 → 클라이언트: <script src="app.js"></script>

선택적 제약조건 (필수 아님)
```

---

## 11.3 리소스 설계 ⭐ 핵심!

### 리소스 = 명사

```
✅ 좋은 예:
/users
/posts
/comments
/products

❌ 나쁜 예:
/getUsers
/createPost
/deleteComment
→ 동사 사용 금지! (HTTP 메서드가 동사 역할)
```

### URI 네이밍 규칙

```
1. 소문자 사용
✅ /users
❌ /Users

2. 하이픈(-) 사용 (언더스코어 피하기)
✅ /user-profiles
❌ /user_profiles

3. 슬래시(/)로 계층 표현
✅ /users/123/posts
❌ /users-posts-123

4. 확장자 사용 안 함
✅ /users/123
❌ /users/123.json

5. 복수형 사용
✅ /users (컬렉션)
✅ /users/123 (단일 항목)
```

### 계층 구조

```
컬렉션 → 문서 → 컬렉션 → 문서

/users                     (사용자 컬렉션)
/users/123                 (사용자 문서)
/users/123/posts           (게시글 컬렉션)
/users/123/posts/456       (게시글 문서)
/users/123/posts/456/comments  (댓글 컬렉션)

최대 3단계 권장:
/resource/id/sub-resource
```

---

## 11.4 HTTP 메서드 활용

### CRUD 매핑

```
┌────────┬─────────┬─────────────────────┐
│ CRUD   │ HTTP    │  예시                │
├────────┼─────────┼─────────────────────┤
│ Create │ POST    │ POST /users         │
│ Read   │ GET     │ GET /users/123      │
│ Update │ PUT     │ PUT /users/123      │
│        │ PATCH   │ PATCH /users/123    │
│ Delete │ DELETE  │ DELETE /users/123   │
└────────┴─────────┴─────────────────────┘
```

### GET (조회)

```javascript
// 컬렉션 조회
GET /api/users
→ 모든 사용자 목록

// 단일 항목 조회
GET /api/users/123
→ ID 123인 사용자

// 필터링
GET /api/users?role=admin&status=active
→ 관리자이면서 활성 상태인 사용자

// 페이지네이션
GET /api/users?page=2&limit=20
→ 2페이지, 20개씩

// 정렬
GET /api/users?sort=created_at&order=desc
→ 생성일 내림차순

// 응답 예시
{
  "data": [
    { "id": 1, "name": "철수" },
    { "id": 2, "name": "영희" }
  ],
  "total": 100,
  "page": 1,
  "limit": 20
}
```

### POST (생성)

```javascript
// 새 리소스 생성
POST /api/users
Content-Type: application/json

{
  "name": "철수",
  "email": "chulsu@example.com"
}

// 응답 (201 Created)
HTTP/1.1 201 Created
Location: /api/users/123

{
  "id": 123,
  "name": "철수",
  "email": "chulsu@example.com",
  "created_at": "2024-01-01T00:00:00Z"
}

// 중요: Location 헤더로 생성된 리소스 위치 전달
```

### PUT (전체 수정)

```javascript
// 전체 교체
PUT /api/users/123
Content-Type: application/json

{
  "name": "김철수",
  "email": "new@example.com",
  "age": 30,
  "address": "서울"
}

// 모든 필드를 보내야 함!
// 누락된 필드는 삭제됨
```

### PATCH (부분 수정)

```javascript
// 일부만 수정
PATCH /api/users/123
Content-Type: application/json

{
  "email": "new@example.com"
}

// name, age, address는 그대로 유지
// email만 변경

// 응답 (200 OK)
{
  "id": 123,
  "name": "철수",
  "email": "new@example.com",  // 변경됨
  "age": 25,
  "address": "서울"
}
```

### DELETE (삭제)

```javascript
// 리소스 삭제
DELETE /api/users/123

// 응답 옵션 1: 204 No Content (본문 없음)
HTTP/1.1 204 No Content

// 응답 옵션 2: 200 OK (삭제된 정보 반환)
HTTP/1.1 200 OK
{
  "id": 123,
  "name": "철수",
  "deleted_at": "2024-01-01T00:00:00Z"
}
```

---

## 11.5 상태 코드 활용

### 2xx: 성공

```
200 OK: 성공 (GET, PUT, PATCH, DELETE)
201 Created: 생성 성공 (POST)
204 No Content: 성공, 본문 없음 (DELETE)

예:
GET /api/users/123 → 200
POST /api/users → 201
DELETE /api/users/123 → 204
```

### 4xx: 클라이언트 오류

```
400 Bad Request: 잘못된 요청
401 Unauthorized: 인증 필요
403 Forbidden: 권한 없음
404 Not Found: 리소스 없음
409 Conflict: 충돌 (이미 존재)
422 Unprocessable Entity: 검증 실패

예:
POST /api/users (이메일 형식 오류) → 400
GET /api/admin (비로그인) → 401
DELETE /api/users/1 (관리자만 가능) → 403
GET /api/users/999 (존재 안 함) → 404
```

### 5xx: 서버 오류

```
500 Internal Server Error: 서버 오류
502 Bad Gateway: 게이트웨이 오류
503 Service Unavailable: 서비스 불가

예:
데이터베이스 연결 실패 → 500
업스트림 서버 오류 → 502
점검 중 → 503
```

---

## 11.6 에러 응답 설계

### 일관된 에러 포맷

```javascript
// 표준 에러 응답
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "이메일 형식이 올바르지 않습니다",
    "details": [
      {
        "field": "email",
        "message": "유효한 이메일 주소를 입력하세요"
      }
    ]
  }
}

// Express 구현
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({
    error: {
      code: err.code || 'INTERNAL_ERROR',
      message: err.message,
      details: err.details || []
    }
  });
});
```

### 에러 코드 체계

```javascript
// 커스텀 에러 코드
const ErrorCodes = {
  // 인증/인가
  UNAUTHORIZED: 'UNAUTHORIZED',
  FORBIDDEN: 'FORBIDDEN',

  // 검증
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  INVALID_INPUT: 'INVALID_INPUT',

  // 리소스
  NOT_FOUND: 'NOT_FOUND',
  ALREADY_EXISTS: 'ALREADY_EXISTS',

  // 비즈니스 로직
  INSUFFICIENT_BALANCE: 'INSUFFICIENT_BALANCE',
  OUT_OF_STOCK: 'OUT_OF_STOCK'
};

// 사용 예시
if (!user) {
  throw new ApiError(404, ErrorCodes.NOT_FOUND, '사용자를 찾을 수 없습니다');
}
```

---

## 11.7 버저닝 (Versioning)

### 방법 1: URL에 포함 (권장)

```
✅ 가장 명확하고 직관적

/api/v1/users
/api/v2/users
/api/v3/users

장점:
- 명확함
- 브라우저에서 바로 테스트 가능
- 캐싱 쉬움
```

### 방법 2: 헤더 사용

```
GET /api/users
Accept: application/vnd.myapi.v2+json

또는:
API-Version: 2

장점:
- URL 깔끔
단점:
- 테스트 어려움
- 캐싱 복잡
```

### 방법 3: 쿼리 파라미터

```
GET /api/users?version=2

단점:
- 덜 RESTful
- 비추천
```

---

## 11.8 페이지네이션

### Offset 기반

```javascript
// 요청
GET /api/posts?page=2&limit=20

// 응답
{
  "data": [...],
  "pagination": {
    "total": 1000,
    "page": 2,
    "limit": 20,
    "total_pages": 50,
    "has_next": true,
    "has_prev": true
  }
}

// 구현 (Express + Sequelize)
router.get('/posts', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const offset = (page - 1) * limit;

  const { count, rows } = await Post.findAndCountAll({
    limit,
    offset,
    order: [['created_at', 'DESC']]
  });

  res.json({
    data: rows,
    pagination: {
      total: count,
      page,
      limit,
      total_pages: Math.ceil(count / limit),
      has_next: page * limit < count,
      has_prev: page > 1
    }
  });
});
```

### Cursor 기반 (무한 스크롤)

```javascript
// 요청
GET /api/posts?cursor=eyJpZCI6MTIzfQ==&limit=20

// 응답
{
  "data": [...],
  "next_cursor": "eyJpZCI6MTQzfQ==",
  "has_more": true
}

// 구현
router.get('/posts', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const cursor = req.query.cursor
    ? JSON.parse(Buffer.from(req.query.cursor, 'base64').toString())
    : null;

  const where = cursor ? { id: { $lt: cursor.id } } : {};

  const posts = await Post.findAll({
    where,
    limit: limit + 1,  // +1로 다음 페이지 존재 여부 확인
    order: [['id', 'DESC']]
  });

  const hasMore = posts.length > limit;
  const data = hasMore ? posts.slice(0, -1) : posts;

  const nextCursor = hasMore
    ? Buffer.from(JSON.stringify({ id: data[data.length - 1].id })).toString('base64')
    : null;

  res.json({
    data,
    next_cursor: nextCursor,
    has_more: hasMore
  });
});
```

---

## 11.9 필터링, 정렬, 검색

### 필터링

```javascript
// 단일 필터
GET /api/products?category=electronics

// 다중 필터
GET /api/products?category=electronics&status=active&price_min=1000

// 구현
router.get('/products', async (req, res) => {
  const { category, status, price_min, price_max } = req.query;

  const where = {};
  if (category) where.category = category;
  if (status) where.status = status;
  if (price_min || price_max) {
    where.price = {};
    if (price_min) where.price.$gte = price_min;
    if (price_max) where.price.$lte = price_max;
  }

  const products = await Product.findAll({ where });
  res.json({ data: products });
});
```

### 정렬

```javascript
// 단일 정렬
GET /api/products?sort=price

// 내림차순
GET /api/products?sort=-price

// 다중 정렬
GET /api/products?sort=category,-price

// 구현
router.get('/products', async (req, res) => {
  const sortParam = req.query.sort || 'created_at';
  const sortFields = sortParam.split(',');

  const order = sortFields.map(field => {
    if (field.startsWith('-')) {
      return [field.substring(1), 'DESC'];
    }
    return [field, 'ASC'];
  });

  const products = await Product.findAll({ order });
  res.json({ data: products });
});
```

### 검색

```javascript
// 단순 검색
GET /api/products?q=laptop

// 필드별 검색
GET /api/products?name=laptop&description=gaming

// 구현 (Sequelize)
router.get('/products', async (req, res) => {
  const { q } = req.query;

  const where = q ? {
    [Op.or]: [
      { name: { [Op.like]: `%${q}%` } },
      { description: { [Op.like]: `%${q}%` } }
    ]
  } : {};

  const products = await Product.findAll({ where });
  res.json({ data: products });
});
```

---

## 11.10 HATEOAS (선택)

### 하이퍼미디어로 API 상태 전이

```javascript
// HATEOAS 응답 예시
GET /api/users/123

{
  "id": 123,
  "name": "철수",
  "email": "chulsu@example.com",
  "_links": {
    "self": { "href": "/api/users/123" },
    "posts": { "href": "/api/users/123/posts" },
    "followers": { "href": "/api/users/123/followers" },
    "update": {
      "href": "/api/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/api/users/123",
      "method": "DELETE"
    }
  }
}

// 클라이언트는 하드코딩 불필요
// _links를 따라가면 됨
```

---

## 11.11 실전 예제: 블로그 API

### 리소스 설계

```
사용자 (Users)
├─ 게시글 (Posts)
│  └─ 댓글 (Comments)
└─ 팔로워 (Followers)

카테고리 (Categories)
└─ 게시글 (Posts)
```

### 엔드포인트

```javascript
// 사용자
GET    /api/users                  // 사용자 목록
GET    /api/users/:id              // 사용자 조회
POST   /api/users                  // 사용자 생성
PUT    /api/users/:id              // 사용자 수정
DELETE /api/users/:id              // 사용자 삭제

// 게시글
GET    /api/posts                  // 게시글 목록
GET    /api/posts/:id              // 게시글 조회
POST   /api/posts                  // 게시글 생성
PUT    /api/posts/:id              // 게시글 수정
DELETE /api/posts/:id              // 게시글 삭제

// 사용자별 게시글
GET    /api/users/:id/posts        // 특정 사용자의 게시글 목록
POST   /api/users/:id/posts        // 특정 사용자의 게시글 생성

// 댓글
GET    /api/posts/:id/comments     // 게시글의 댓글 목록
POST   /api/posts/:id/comments     // 댓글 생성
PUT    /api/comments/:id           // 댓글 수정
DELETE /api/comments/:id           // 댓글 삭제

// 팔로우
GET    /api/users/:id/followers    // 팔로워 목록
GET    /api/users/:id/following    // 팔로잉 목록
POST   /api/users/:id/follow       // 팔로우
DELETE /api/users/:id/follow       // 언팔로우

// 카테고리
GET    /api/categories             // 카테고리 목록
GET    /api/categories/:id/posts   // 카테고리별 게시글
```

### Express 구현 예시

```javascript
const express = require('express');
const router = express.Router();

// 미들웨어
const auth = require('../middleware/auth');
const validate = require('../middleware/validate');
const { postSchema } = require('../schemas');

// 게시글 목록
router.get('/posts', async (req, res) => {
  try {
    const { page = 1, limit = 20, sort = '-created_at', category } = req.query;

    const query = {};
    if (category) query.category = category;

    const posts = await Post.find(query)
      .sort(sort)
      .limit(limit * 1)
      .skip((page - 1) * limit)
      .populate('author', 'name email')
      .exec();

    const total = await Post.countDocuments(query);

    res.json({
      data: posts,
      pagination: {
        total,
        page: parseInt(page),
        limit: parseInt(limit),
        total_pages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    res.status(500).json({ error: { message: error.message } });
  }
});

// 게시글 조회
router.get('/posts/:id', async (req, res) => {
  try {
    const post = await Post.findById(req.params.id)
      .populate('author', 'name email')
      .populate('comments');

    if (!post) {
      return res.status(404).json({
        error: { code: 'NOT_FOUND', message: '게시글을 찾을 수 없습니다' }
      });
    }

    res.json({ data: post });
  } catch (error) {
    res.status(500).json({ error: { message: error.message } });
  }
});

// 게시글 생성 (인증 필요)
router.post('/posts', auth, validate(postSchema), async (req, res) => {
  try {
    const post = new Post({
      ...req.body,
      author: req.user.id
    });

    await post.save();

    res.status(201)
      .location(`/api/posts/${post.id}`)
      .json({ data: post });
  } catch (error) {
    res.status(500).json({ error: { message: error.message } });
  }
});

// 게시글 수정 (작성자만)
router.put('/posts/:id', auth, validate(postSchema), async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);

    if (!post) {
      return res.status(404).json({
        error: { code: 'NOT_FOUND', message: '게시글을 찾을 수 없습니다' }
      });
    }

    if (post.author.toString() !== req.user.id) {
      return res.status(403).json({
        error: { code: 'FORBIDDEN', message: '권한이 없습니다' }
      });
    }

    Object.assign(post, req.body);
    await post.save();

    res.json({ data: post });
  } catch (error) {
    res.status(500).json({ error: { message: error.message } });
  }
});

// 게시글 삭제 (작성자만)
router.delete('/posts/:id', auth, async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);

    if (!post) {
      return res.status(404).json({
        error: { code: 'NOT_FOUND', message: '게시글을 찾을 수 없습니다' }
      });
    }

    if (post.author.toString() !== req.user.id) {
      return res.status(403).json({
        error: { code: 'FORBIDDEN', message: '권한이 없습니다' }
      });
    }

    await post.remove();

    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: { message: error.message } });
  }
});

module.exports = router;
```

---

## 핵심 요약

### REST 6원칙
```
1. Client-Server
2. Stateless
3. Cacheable
4. Uniform Interface
5. Layered System
6. Code on Demand (선택)
```

### 설계 핵심
```
리소스: 명사 사용
메서드: HTTP 메서드 활용
상태: HTTP 상태 코드
계층: /resource/id/sub-resource
```

### Best Practices
```
✅ 명사 사용, 복수형
✅ 소문자, 하이픈
✅ 버저닝 (/api/v1)
✅ 필터링/정렬/페이지네이션
✅ 일관된 에러 포맷
```

---

## 다음 단계

HTTP와 REST API를 마스터했으니, 이제 **DNS 레코드 타입**을 배워봅시다!

👉 [Chapter 13. DNS 레코드 타입](../part03-DNS-도메인/13-DNS-레코드.md)

---

## 실습 과제

### 과제 1: TODO API 설계

```
요구사항:
- 할 일 CRUD
- 사용자별 할 일
- 카테고리별 할 일
- 완료/미완료 필터링
- 페이지네이션

엔드포인트 설계해보기:
GET    /api/todos
POST   /api/todos
GET    /api/todos/:id
PUT    /api/todos/:id
DELETE /api/todos/:id
GET    /api/todos?status=completed&page=1
```

### 과제 2: Express REST API 구현

```javascript
// Express + MongoDB로 블로그 API 구현
// - 사용자 인증 (JWT)
// - 게시글 CRUD
// - 댓글 기능
// - 페이지네이션
// - 에러 처리
```

### 과제 3: API 문서화

```
Swagger/OpenAPI로 API 문서화

swagger-jsdoc 사용:
npm install swagger-jsdoc swagger-ui-express

/**
 * @swagger
 * /api/posts:
 *   get:
 *     summary: 게시글 목록 조회
 *     parameters:
 *       - name: page
 *         in: query
 *         schema:
 *           type: integer
 *     responses:
 *       200:
 *         description: 성공
 */
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐ 중급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!
