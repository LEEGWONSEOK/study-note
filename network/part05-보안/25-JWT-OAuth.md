# Chapter 25. JWT와 OAuth

## 25.1 JWT (JSON Web Token)

### 정의

```
JWT: JSON 형태의 토큰 기반 인증

세션 vs JWT:
┌──────────┬──────────┬──────────┐
│   항목    │  세션    │  JWT     │
├──────────┼──────────┼──────────┤
│ 저장 위치 │ 서버     │ 클라이언트│
│ 확장성    │ 어려움   │ 쉬움     │
│ 상태      │ Stateful │ Stateless│
│ 취소      │ 쉬움     │ 어려움   │
└──────────┴──────────┴──────────┘
```

### JWT 구조

```
JWT = Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VySWQiOjEyMywiZW1haWwiOiJ1c2VyQGV4YW1wbGUuY29tIn0.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

├─ Header (알고리즘)
├─ Payload (데이터)
└─ Signature (서명)
```

### JWT 생성 및 검증

```javascript
const jwt = require('jsonwebtoken');

const SECRET_KEY = process.env.JWT_SECRET || 'your-secret-key';

// JWT 생성
function generateToken(user) {
  const payload = {
    userId: user.id,
    email: user.email,
    role: user.role
  };

  const token = jwt.sign(payload, SECRET_KEY, {
    expiresIn: '1h',        // 1시간 유효
    issuer: 'my-app',       // 발급자
    audience: 'users'       // 대상
  });

  return token;
}

// JWT 검증
function verifyToken(token) {
  try {
    const decoded = jwt.verify(token, SECRET_KEY);
    return decoded;
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      throw new Error('토큰 만료');
    }
    throw new Error('유효하지 않은 토큰');
  }
}

// 로그인
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ where: { email } });
  const isValid = await bcrypt.compare(password, user.password);

  if (!isValid) {
    return res.status(401).json({ error: '인증 실패' });
  }

  const token = generateToken(user);

  res.json({
    token,
    user: {
      id: user.id,
      email: user.email
    }
  });
});

// 인증 미들웨어
function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: '토큰 없음' });
  }

  const token = authHeader.substring(7);  // "Bearer " 제거

  try {
    const decoded = verifyToken(token);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: error.message });
  }
}

// 보호된 엔드포인트
app.get('/api/profile', authenticate, async (req, res) => {
  const user = await User.findByPk(req.user.userId);
  res.json(user);
});
```

---

## 25.2 Refresh Token

### 개념

```
Access Token: 짧은 유효기간 (15분)
Refresh Token: 긴 유효기간 (7일)

흐름:
1. 로그인 → Access + Refresh 토큰 발급
2. Access 토큰으로 API 호출
3. Access 만료 → Refresh로 재발급
4. Refresh 만료 → 재로그인
```

### 구현

```javascript
// Refresh Token 생성
function generateRefreshToken(user) {
  const payload = { userId: user.id };

  return jwt.sign(payload, REFRESH_SECRET_KEY, {
    expiresIn: '7d'
  });
}

// 로그인: Access + Refresh 발급
app.post('/api/auth/login', async (req, res) => {
  // 인증...

  const accessToken = generateToken(user);
  const refreshToken = generateRefreshToken(user);

  // Refresh Token DB 저장
  await RefreshToken.create({
    userId: user.id,
    token: refreshToken,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  res.json({
    accessToken,
    refreshToken
  });
});

// Access Token 재발급
app.post('/api/auth/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  if (!refreshToken) {
    return res.status(401).json({ error: 'Refresh 토큰 없음' });
  }

  try {
    const decoded = jwt.verify(refreshToken, REFRESH_SECRET_KEY);

    // DB에서 확인
    const storedToken = await RefreshToken.findOne({
      where: { token: refreshToken, userId: decoded.userId }
    });

    if (!storedToken || storedToken.expiresAt < new Date()) {
      return res.status(401).json({ error: '유효하지 않은 토큰' });
    }

    // 새 Access Token 발급
    const user = await User.findByPk(decoded.userId);
    const newAccessToken = generateToken(user);

    res.json({ accessToken: newAccessToken });
  } catch (error) {
    res.status(401).json({ error: '토큰 검증 실패' });
  }
});

// 로그아웃: Refresh Token 삭제
app.post('/api/auth/logout', async (req, res) => {
  const { refreshToken } = req.body;

  await RefreshToken.destroy({
    where: { token: refreshToken }
  });

  res.json({ message: '로그아웃 성공' });
});
```

---

## 25.3 JWT 보안

### 민감 정보 저장 금지

```javascript
// ❌ 나쁜 예
const payload = {
  userId: 123,
  email: 'user@example.com',
  password: 'hashed_password',  // 절대 금지!
  creditCard: '1234-5678-...'   // 절대 금지!
};

// ✅ 좋은 예
const payload = {
  userId: 123,
  email: 'user@example.com',
  role: 'user'
};

// JWT는 디코딩 가능 (암호화 아님!)
// Base64로 인코딩되어 있을 뿐
```

### 강력한 Secret Key

```javascript
// ❌ 나쁜 예
const SECRET_KEY = 'secret';

// ✅ 좋은 예
const SECRET_KEY = crypto.randomBytes(64).toString('hex');
// 또는 환경 변수
const SECRET_KEY = process.env.JWT_SECRET;

// .env
JWT_SECRET=a1b2c3d4e5f6...  (64+ 문자)
```

### XSS 방어

```javascript
// LocalStorage 저장 (XSS 취약)
// ❌ localStorage.setItem('token', token)

// HttpOnly 쿠키 저장 (권장)
// ✅
res.cookie('accessToken', token, {
  httpOnly: true,   // JS 접근 차단
  secure: true,     // HTTPS only
  sameSite: 'strict'
});

// 또는 메모리 저장 (새로고침 시 소실)
```

---

## 25.4 OAuth 2.0

### 정의

```
OAuth: 인증 위임 프로토콜
"다른 서비스의 계정으로 로그인"

예:
- Google 계정으로 로그인
- Facebook 계정으로 로그인
- GitHub 계정으로 로그인
```

### OAuth 흐름

```
1. 사용자가 "Google로 로그인" 클릭

2. Google 로그인 페이지로 리다이렉트
   https://accounts.google.com/o/oauth2/auth?
     client_id=...
     &redirect_uri=...
     &scope=email profile

3. 사용자가 Google에 로그인 및 권한 승인

4. 앱으로 리다이렉트 (Authorization Code 포함)
   https://myapp.com/callback?code=ABC123

5. 앱이 Code로 Access Token 요청
   POST https://oauth2.googleapis.com/token
   {
     code: 'ABC123',
     client_id: '...',
     client_secret: '...',
     grant_type: 'authorization_code'
   }

6. Google이 Access Token 반환

7. Access Token으로 사용자 정보 요청
   GET https://www.googleapis.com/oauth2/v1/userinfo?
     access_token=XYZ789
```

---

## 25.5 Google OAuth 구현

### Google Cloud Console 설정

```
1. https://console.cloud.google.com 접속
2. 프로젝트 생성
3. OAuth 동의 화면 구성
4. 사용자 인증 정보 → OAuth 클라이언트 ID 생성
   - 웹 애플리케이션
   - 승인된 리디렉션 URI: http://localhost:3000/auth/google/callback

5. Client ID, Client Secret 발급
```

### Passport.js 구현

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

// Passport 설정
passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      // 사용자 찾기 또는 생성
      let user = await User.findOne({
        where: { googleId: profile.id }
      });

      if (!user) {
        user = await User.create({
          googleId: profile.id,
          email: profile.emails[0].value,
          name: profile.displayName,
          avatar: profile.photos[0].value
        });
      }

      return done(null, user);
    } catch (error) {
      return done(error, null);
    }
  }
));

// 세션 직렬화
passport.serializeUser((user, done) => {
  done(null, user.id);
});

passport.deserializeUser(async (id, done) => {
  const user = await User.findByPk(id);
  done(null, user);
});

// 라우트
app.use(passport.initialize());
app.use(passport.session());

// Google 로그인 시작
app.get('/auth/google',
  passport.authenticate('google', {
    scope: ['profile', 'email']
  })
);

// Google 콜백
app.get('/auth/google/callback',
  passport.authenticate('google', {
    failureRedirect: '/login'
  }),
  (req, res) => {
    // 성공 시
    res.redirect('/dashboard');
  }
);
```

---

## 25.6 JWT vs Session vs OAuth

### 비교

```
┌──────────┬─────────┬─────────┬─────────┐
│   항목    │ Session │  JWT    │  OAuth  │
├──────────┼─────────┼─────────┼─────────┤
│ 저장소    │ 서버    │ 클라이언트│ 서버    │
│ 확장성    │ 낮음    │ 높음    │ 보통    │
│ 취소      │ 쉬움    │ 어려움  │ 쉬움    │
│ 용도      │ 웹앱    │ API     │ 소셜로그인│
│ 보안      │ 높음    │ 보통    │ 높음    │
└──────────┴─────────┴─────────┴─────────┘
```

### 선택 가이드

```
Session:
✅ 전통적인 웹 애플리케이션
✅ 서버가 1대 또는 Redis 사용 가능
✅ 즉시 로그아웃 필요

JWT:
✅ REST API, 모바일 앱
✅ Stateless 필요 (마이크로서비스)
✅ 다중 서버, 서비스 간 인증

OAuth:
✅ 소셜 로그인
✅ 제3자 앱 권한 부여
```

---

## 25.7 실전 예제: JWT + Refresh Token

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

const app = express();
app.use(express.json());

const ACCESS_SECRET = process.env.ACCESS_SECRET;
const REFRESH_SECRET = process.env.REFRESH_SECRET;

// 토큰 생성
function generateAccessToken(user) {
  return jwt.sign(
    { userId: user.id, email: user.email, role: user.role },
    ACCESS_SECRET,
    { expiresIn: '15m' }
  );
}

function generateRefreshToken(user) {
  return jwt.sign(
    { userId: user.id },
    REFRESH_SECRET,
    { expiresIn: '7d' }
  );
}

// 로그인
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ where: { email } });
  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(401).json({ error: '인증 실패' });
  }

  const accessToken = generateAccessToken(user);
  const refreshToken = generateRefreshToken(user);

  // Refresh Token 저장
  await RefreshToken.create({
    userId: user.id,
    token: refreshToken,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  // Access Token: HttpOnly 쿠키
  res.cookie('accessToken', accessToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 15 * 60 * 1000
  });

  // Refresh Token: HttpOnly 쿠키
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000
  });

  res.json({ message: '로그인 성공' });
});

// 인증 미들웨어
function authenticate(req, res, next) {
  const token = req.cookies.accessToken;

  if (!token) {
    return res.status(401).json({ error: '토큰 없음' });
  }

  try {
    const decoded = jwt.verify(token, ACCESS_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: '토큰 만료', code: 'TOKEN_EXPIRED' });
    }
    res.status(401).json({ error: '유효하지 않은 토큰' });
  }
}

// Refresh
app.post('/api/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;

  if (!refreshToken) {
    return res.status(401).json({ error: 'Refresh 토큰 없음' });
  }

  try {
    const decoded = jwt.verify(refreshToken, REFRESH_SECRET);

    const storedToken = await RefreshToken.findOne({
      where: { token: refreshToken, userId: decoded.userId }
    });

    if (!storedToken) {
      return res.status(401).json({ error: '유효하지 않은 토큰' });
    }

    const user = await User.findByPk(decoded.userId);
    const newAccessToken = generateAccessToken(user);

    res.cookie('accessToken', newAccessToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 15 * 60 * 1000
    });

    res.json({ message: '토큰 갱신 성공' });
  } catch (error) {
    res.status(401).json({ error: '토큰 검증 실패' });
  }
});

// 로그아웃
app.post('/api/auth/logout', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;

  if (refreshToken) {
    await RefreshToken.destroy({ where: { token: refreshToken } });
  }

  res.clearCookie('accessToken');
  res.clearCookie('refreshToken');

  res.json({ message: '로그아웃 성공' });
});

// 보호된 엔드포인트
app.get('/api/profile', authenticate, async (req, res) => {
  const user = await User.findByPk(req.user.userId);
  res.json(user);
});
```

---

## 핵심 요약

### JWT
```
JSON Web Token
Stateless 인증
Access (15분) + Refresh (7일)
HttpOnly 쿠키 저장
```

### OAuth
```
소셜 로그인
인증 위임
Google, Facebook, GitHub 등
Passport.js 사용
```

### 보안
```
민감 정보 저장 금지
강력한 Secret Key
HttpOnly 쿠키
XSS/CSRF 방어
```

### 선택
```
웹앱: Session
API: JWT
소셜: OAuth
```

---

## 다음 단계

JWT와 OAuth를 이해했으니, 이제 **웹 보안 공격과 방어**를 배워봅시다!

👉 [Chapter 26. 웹 보안 공격과 방어](26-웹보안.md)

---

## 실습 과제

### 과제 1: JWT 인증

```javascript
// 1. Access + Refresh 토큰 발급
// 2. 인증 미들웨어
// 3. 토큰 갱신 API
// 4. HttpOnly 쿠키 저장
```

### 과제 2: Google OAuth

```javascript
// 1. Google Cloud Console 설정
// 2. Passport.js 설정
// 3. Google 로그인 구현
// 4. 사용자 정보 저장
```

### 과제 3: 통합 인증 시스템

```javascript
// 1. 이메일/비밀번호 로그인 (JWT)
// 2. Google OAuth
// 3. GitHub OAuth
// 4. 통합 사용자 관리
```

---

**작성일**: 2024년
**난이도**: ⭐⭐⭐⭐ 고급
**예상 학습 시간**: 3-4시간
**대상**: 백엔드 개발자
**중요도**: ⭐⭐⭐ 매우 중요!