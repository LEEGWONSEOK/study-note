# Chapter 16. 보안 Best Practices

## 16.1 CORS (Cross-Origin Resource Sharing)

CORS는 다른 도메인에서 API에 접근할 수 있도록 허용하는 메커니즘입니다.

### CORS 설정

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS 미들웨어 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",      # React 개발 서버
        "http://localhost:8080",      # Vue 개발 서버
        "https://yourdomain.com",     # 프로덕션 도메인
    ],
    allow_credentials=True,           # 쿠키 허용
    allow_methods=["*"],              # 모든 HTTP 메서드 허용
    allow_headers=["*"],              # 모든 헤더 허용
)

@app.get("/api/data")
def get_data():
    return {"message": "CORS enabled"}
```

**Spring Boot와 비교:**
```java
// Spring Boot
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",
                "https://yourdomain.com"
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### 환경별 CORS 설정

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    CORS_ORIGINS: List[str] = [
        "http://localhost:3000",
        "http://localhost:8080"
    ]

    class Config:
        env_file = ".env"

settings = Settings()

# main.py
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 프로덕션 CORS 설정

```python
# .env.production
CORS_ORIGINS=["https://app.example.com","https://admin.example.com"]

# 보안 강화
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,  # 특정 도메인만
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],  # 필요한 메서드만
    allow_headers=["Content-Type", "Authorization"],  # 필요한 헤더만
    max_age=600,  # Preflight 캐시 시간 (초)
)
```

---

## 16.2 CSRF 방지

### Double Submit Cookie 패턴

```python
# app/core/csrf.py
import secrets
from fastapi import Request, HTTPException, status
from fastapi.responses import Response

CSRF_TOKEN_KEY = "csrf_token"
CSRF_HEADER_NAME = "X-CSRF-Token"

def generate_csrf_token() -> str:
    """CSRF 토큰 생성"""
    return secrets.token_urlsafe(32)

def set_csrf_token(response: Response) -> str:
    """응답에 CSRF 토큰 쿠키 설정"""
    token = generate_csrf_token()
    response.set_cookie(
        key=CSRF_TOKEN_KEY,
        value=token,
        httponly=False,  # JavaScript에서 읽을 수 있어야 함
        secure=True,     # HTTPS only
        samesite="strict"
    )
    return token

async def verify_csrf_token(request: Request):
    """CSRF 토큰 검증"""
    # GET, HEAD, OPTIONS는 검증 불필요
    if request.method in ["GET", "HEAD", "OPTIONS"]:
        return

    # 쿠키에서 토큰 가져오기
    cookie_token = request.cookies.get(CSRF_TOKEN_KEY)

    # 헤더에서 토큰 가져오기
    header_token = request.headers.get(CSRF_HEADER_NAME)

    # 검증
    if not cookie_token or not header_token or cookie_token != header_token:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid CSRF token"
        )

# 사용
from fastapi import Depends

@app.post("/api/data")
async def post_data(
    data: dict,
    _: None = Depends(verify_csrf_token)
):
    return {"message": "Data saved"}
```

**Spring Security와 비교:**
```java
// Spring Security - 기본적으로 CSRF 활성화
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf()
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .and()
            // ...
    }
}
```

---

## 16.3 SQL Injection 방지

SQLAlchemy ORM을 사용하면 자동으로 SQL Injection이 방지됩니다.

### 안전한 방법 (ORM 사용)

```python
# ✅ 안전 - ORM 사용
from sqlalchemy.orm import Session

def get_user_by_email(db: Session, email: str):
    # 파라미터가 자동으로 이스케이프됨
    return db.query(User).filter(User.email == email).first()
```

### 위험한 방법 (Raw SQL)

```python
# ❌ 위험 - Raw SQL with string formatting
def get_user_by_email_unsafe(db: Session, email: str):
    # SQL Injection 취약!
    query = f"SELECT * FROM users WHERE email = '{email}'"
    return db.execute(query).fetchone()

# 공격 예: email = "admin@example.com' OR '1'='1"
```

### Raw SQL 사용 시 안전한 방법

```python
# ✅ 안전 - 파라미터 바인딩 사용
from sqlalchemy import text

def get_user_by_email_safe(db: Session, email: str):
    query = text("SELECT * FROM users WHERE email = :email")
    return db.execute(query, {"email": email}).fetchone()
```

**Spring Boot JPA와 비교:**
```java
// ✅ 안전 - JPA/JPQL
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(@Param("email") String email);

// ❌ 위험 - Native Query with concatenation
@Query(value = "SELECT * FROM users WHERE email = '" + email + "'", nativeQuery = true)
User findByEmailUnsafe(String email);

// ✅ 안전 - Native Query with parameter
@Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
User findByEmailSafe(String email);
```

---

## 16.4 XSS (Cross-Site Scripting) 방지

### HTML 이스케이프

```python
# app/utils/sanitize.py
import html
from typing import Optional

def sanitize_html(text: Optional[str]) -> Optional[str]:
    """HTML 특수 문자 이스케이프"""
    if not text:
        return text
    return html.escape(text)

# 사용
from app.utils.sanitize import sanitize_html

@app.post("/posts")
def create_post(title: str, content: str, db: Session = Depends(get_db)):
    # XSS 방지를 위해 HTML 이스케이프
    safe_title = sanitize_html(title)
    safe_content = sanitize_html(content)

    post = Post(title=safe_title, content=safe_content)
    db.add(post)
    db.commit()

    return post
```

### bleach 라이브러리 사용

```python
# 설치: pip install bleach
import bleach

def sanitize_rich_text(text: str) -> str:
    """허용된 HTML 태그만 남기고 나머지 제거"""
    allowed_tags = ['p', 'br', 'strong', 'em', 'u', 'a', 'ul', 'ol', 'li']
    allowed_attributes = {'a': ['href', 'title']}

    return bleach.clean(
        text,
        tags=allowed_tags,
        attributes=allowed_attributes,
        strip=True
    )

# 사용
@app.post("/posts")
def create_post(content: str, db: Session = Depends(get_db)):
    safe_content = sanitize_rich_text(content)
    # ...
```

---

## 16.5 Rate Limiting

API 남용을 방지하기 위한 요청 제한입니다.

### slowapi 사용

```bash
pip install slowapi
```

```python
# app/core/limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# main.py
from fastapi import FastAPI, Request
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from app.core.limiter import limiter

app = FastAPI()

# Rate limiter 설정
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# 사용
@app.get("/api/data")
@limiter.limit("5/minute")  # 분당 5회 제한
async def get_data(request: Request):
    return {"message": "Rate limited endpoint"}

@app.post("/api/login")
@limiter.limit("3/minute")  # 로그인은 더 엄격하게
async def login(request: Request, credentials: LoginCredentials):
    # 로그인 로직
    pass
```

### Redis 기반 Rate Limiting

```python
# 설치: pip install redis
import redis
from datetime import timedelta
from fastapi import HTTPException, status

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def check_rate_limit(key: str, limit: int, window: int) -> bool:
    """
    Rate limit 체크
    key: 식별자 (IP, user_id 등)
    limit: 허용 횟수
    window: 시간 윈도우 (초)
    """
    current = redis_client.get(key)

    if current and int(current) >= limit:
        return False

    pipe = redis_client.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    pipe.execute()

    return True

# 사용
@app.post("/api/expensive-operation")
async def expensive_operation(request: Request):
    client_ip = request.client.host
    key = f"rate_limit:{client_ip}"

    if not check_rate_limit(key, limit=10, window=60):
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Too many requests. Please try again later."
        )

    # 비싼 작업 수행
    return {"message": "Operation completed"}
```

**Spring Boot와 비교:**
```java
// Spring Boot - Bucket4j
@RestController
public class ApiController {
    private final Bucket bucket;

    public ApiController() {
        Bandwidth limit = Bandwidth.classic(10, Refill.intervally(10, Duration.ofMinutes(1)));
        this.bucket = Bucket4j.builder()
            .addLimit(limit)
            .build();
    }

    @GetMapping("/api/data")
    public ResponseEntity<String> getData() {
        if (bucket.tryConsume(1)) {
            return ResponseEntity.ok("Data");
        }
        return ResponseEntity.status(429).body("Too many requests");
    }
}
```

---

## 16.6 환경 변수 관리

### .env 파일 사용

```python
# .env
SECRET_KEY=very-long-random-secret-key-change-in-production
DATABASE_URL=postgresql://user:password@localhost/dbname
REDIS_URL=redis://localhost:6379/0
CORS_ORIGINS=["http://localhost:3000"]

# JWT
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# app/core/config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # 앱 설정
    APP_NAME: str = "FastAPI App"
    DEBUG: bool = False

    # 보안
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # 데이터베이스
    DATABASE_URL: str

    # CORS
    CORS_ORIGINS: List[str]

    # Redis
    REDIS_URL: str

    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

### 환경별 설정 파일

```python
# app/core/config.py
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ENVIRONMENT: str = os.getenv("ENVIRONMENT", "development")

    class Config:
        env_file = f".env.{os.getenv('ENVIRONMENT', 'development')}"

# .env.development
DEBUG=True
DATABASE_URL=postgresql://user:password@localhost/dev_db

# .env.production
DEBUG=False
DATABASE_URL=postgresql://user:password@prod-host/prod_db

# .env.test
DEBUG=True
DATABASE_URL=sqlite:///./test.db
```

---

## 16.7 보안 헤더

### 보안 헤더 추가

```python
# app/middleware/security_headers.py
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # 보안 헤더 추가
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"

        return response

# main.py
from app.middleware.security_headers import SecurityHeadersMiddleware

app.add_middleware(SecurityHeadersMiddleware)
```

**Spring Security와 비교:**
```java
// Spring Security
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.headers()
            .contentTypeOptions()
            .and()
            .xssProtection()
            .and()
            .frameOptions().deny()
            .and()
            .httpStrictTransportSecurity()
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true);
    }
}
```

---

## 16.8 로깅 및 모니터링

### 보안 이벤트 로깅

```python
# app/core/security_logger.py
import logging
from datetime import datetime

security_logger = logging.getLogger("security")
security_logger.setLevel(logging.INFO)

handler = logging.FileHandler("logs/security.log")
formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
handler.setFormatter(formatter)
security_logger.addHandler(handler)

def log_login_attempt(email: str, success: bool, ip: str):
    """로그인 시도 로깅"""
    status = "SUCCESS" if success else "FAILED"
    security_logger.info(f"Login {status} - Email: {email}, IP: {ip}")

def log_unauthorized_access(user_id: int, resource: str, ip: str):
    """권한 없는 접근 시도 로깅"""
    security_logger.warning(
        f"Unauthorized access attempt - User: {user_id}, "
        f"Resource: {resource}, IP: {ip}"
    )

# 사용
@app.post("/auth/login")
async def login(
    credentials: LoginCredentials,
    request: Request,
    db: Session = Depends(get_db)
):
    client_ip = request.client.host
    user = authenticate_user(db, credentials.email, credentials.password)

    if not user:
        log_login_attempt(credentials.email, False, client_ip)
        raise HTTPException(status_code=401, detail="Invalid credentials")

    log_login_attempt(credentials.email, True, client_ip)
    # 토큰 발급...
```

---

## 16.9 실전 보안 체크리스트

### 필수 보안 조치

```python
# ✅ 1. HTTPS 사용 (프로덕션)
# Nginx 등에서 SSL 설정

# ✅ 2. 강력한 SECRET_KEY
import secrets
SECRET_KEY = secrets.token_urlsafe(32)

# ✅ 3. 비밀번호 해싱
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# ✅ 4. SQL Injection 방지
# ORM 사용 또는 파라미터 바인딩

# ✅ 5. XSS 방지
# HTML 이스케이프 또는 bleach 사용

# ✅ 6. CORS 설정
# 특정 도메인만 허용

# ✅ 7. Rate Limiting
# slowapi 또는 Redis 사용

# ✅ 8. 보안 헤더
# SecurityHeadersMiddleware 추가

# ✅ 9. 환경 변수
# .env 파일 사용, .gitignore에 추가

# ✅ 10. 로깅
# 보안 이벤트 로깅
```

### .gitignore

```gitignore
# 환경 변수
.env
.env.*

# 데이터베이스
*.db
*.sqlite

# 로그
logs/
*.log

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python

# 가상환경
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

---

## 16.10 핵심 요약

### 보안 Best Practices

| 보안 영역 | 조치 | 도구/방법 |
|----------|------|----------|
| **CORS** | 특정 도메인만 허용 | CORSMiddleware |
| **CSRF** | 토큰 검증 | Double Submit Cookie |
| **SQL Injection** | ORM 사용 | SQLAlchemy |
| **XSS** | HTML 이스케이프 | html.escape(), bleach |
| **Rate Limiting** | 요청 제한 | slowapi, Redis |
| **환경 변수** | .env 파일 사용 | pydantic-settings |
| **보안 헤더** | 헤더 추가 | Middleware |
| **로깅** | 보안 이벤트 기록 | logging |

### Spring Security vs FastAPI

| 기능 | Spring Security | FastAPI |
|------|----------------|---------|
| **CORS** | WebMvcConfigurer | CORSMiddleware |
| **CSRF** | 기본 활성화 | 수동 구현 |
| **헤더** | http.headers() | Middleware |
| **Rate Limit** | Bucket4j | slowapi |
| **환경 변수** | application.yml | .env |

### 기억할 핵심

1. ✅ **HTTPS**: 프로덕션에서 필수
2. ✅ **SECRET_KEY**: 강력한 키 사용
3. ✅ **ORM**: SQL Injection 자동 방지
4. ✅ **이스케이프**: XSS 방지
5. ✅ **Rate Limit**: API 남용 방지
6. ✅ **.env**: 민감 정보 관리
7. ✅ **로깅**: 보안 이벤트 추적

---

## 16.11 Part 05 완료!

Part 05에서 배운 내용:
- ✅ **Chapter 13**: JWT 인증 - 토큰 생성/검증
- ✅ **Chapter 14**: OAuth2 - Password Flow, 소셜 로그인
- ✅ **Chapter 15**: 권한 관리 - RBAC, Permission
- ✅ **Chapter 16**: 보안 Best Practices - CORS, XSS, Rate Limit

---

## 16.12 다음 챕터 예고

다음 Part에서는 **미들웨어와 예외 처리**를 다룹니다:
- 커스텀 미들웨어 작성
- 요청/응답 로깅
- 전역 예외 처리
- 에러 응답 표준화

**Spring의 Filter/Interceptor와 비교하며 FastAPI 미들웨어를 마스터합니다!**

---

[← 이전: Chapter 15. 권한과 역할 관리](chapter15-authorization.md) | [목차로](../README.md) | [다음: Part 06 →](../part06-middleware-exceptions/chapter17-middleware.md)
