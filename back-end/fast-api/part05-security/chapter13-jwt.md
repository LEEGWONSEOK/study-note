# Chapter 13. JWT 인증

## 13.1 JWT란?

JWT(JSON Web Token)는 사용자 인증을 위한 토큰 기반 인증 방식입니다. Spring Security의 JWT 구현과 유사한 방식으로 FastAPI에서도 사용할 수 있습니다.

### JWT 구조

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**구성 요소:**
1. **Header**: 토큰 타입, 알고리즘
2. **Payload**: 사용자 정보 (Claims)
3. **Signature**: 검증 서명

### Spring Boot vs FastAPI

| 기능 | Spring Security | FastAPI |
|------|----------------|---------|
| **라이브러리** | Spring Security + JWT | python-jose[cryptography] |
| **설정** | SecurityConfig 클래스 | 함수형 의존성 |
| **토큰 생성** | JwtTokenProvider | jose.jwt.encode() |
| **토큰 검증** | JwtAuthenticationFilter | Depends() |
| **비밀번호 해싱** | BCryptPasswordEncoder | passlib |

---

## 13.2 필요한 패키지 설치

```bash
# JWT 관련
pip install python-jose[cryptography]

# 비밀번호 해싱
pip install passlib[bcrypt]

# 환경 변수 관리
pip install python-dotenv
```

---

## 13.3 비밀번호 해싱

### 패스워드 유틸리티

```python
# app/core/security.py
from passlib.context import CryptContext

# Bcrypt 설정
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """비밀번호 검증"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """비밀번호 해싱"""
    return pwd_context.hash(password)
```

**Spring Boot와 비교:**
```java
// Spring Security
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

// 사용
String hashedPassword = passwordEncoder.encode(rawPassword);
boolean matches = passwordEncoder.matches(rawPassword, hashedPassword);
```

---

## 13.4 JWT 토큰 생성 및 검증

### 설정

```python
# app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    SECRET_KEY: str = "your-secret-key-here-change-in-production"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    class Config:
        env_file = ".env"

settings = Settings()
```

### JWT 유틸리티

```python
# app/core/jwt.py
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from app.core.config import settings

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Access Token 생성"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire, "type": "access"})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

    return encoded_jwt

def create_refresh_token(data: dict) -> str:
    """Refresh Token 생성"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)

    to_encode.update({"exp": expire, "type": "refresh"})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

    return encoded_jwt

def verify_token(token: str) -> Optional[dict]:
    """토큰 검증 및 페이로드 반환"""
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
        return payload
    except JWTError:
        return None
```

**Spring Security와 비교:**
```java
// Spring Security
@Component
public class JwtTokenProvider {
    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration}")
    private Long expiration;

    public String generateToken(Authentication authentication) {
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();

        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, secretKey)
            .compact();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }
}
```

---

## 13.5 사용자 인증

### User 모델

```python
# app/models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from app.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(100), unique=True, nullable=False, index=True)
    username = Column(String(50), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime, server_default=func.now())
```

### 스키마

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class UserRegister(BaseModel):
    email: EmailStr
    username: str
    password: str

class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class TokenPayload(BaseModel):
    sub: int  # user_id
    exp: int
    type: str

class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    is_active: bool

    class Config:
        from_attributes = True
```

---

## 13.6 인증 의존성

### 현재 사용자 가져오기

```python
# app/api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.orm import Session
from app.database import get_db
from app.core.jwt import verify_token
from app.models.user import User

security = HTTPBearer()

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
) -> User:
    """현재 인증된 사용자 가져오기"""
    token = credentials.credentials

    # 토큰 검증
    payload = verify_token(token)
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )

    # 토큰 타입 확인
    if payload.get("type") != "access":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token type"
        )

    # 사용자 조회
    user_id = payload.get("sub")
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )

    return user

def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """활성 사용자만 허용"""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )
    return current_user

def get_current_superuser(
    current_user: User = Depends(get_current_user)
) -> User:
    """관리자만 허용"""
    if not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    return current_user
```

**Spring Security와 비교:**
```java
// Spring Security
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .addFilter(new JwtAuthenticationFilter(authenticationManager()))
            .authorizeRequests()
            .antMatchers("/api/auth/**").permitAll()
            .antMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated();
    }
}

// 현재 사용자 가져오기
@GetMapping("/me")
public User getCurrentUser(@AuthenticationPrincipal UserDetails userDetails) {
    return userService.findByUsername(userDetails.getUsername());
}
```

---

## 13.7 인증 라우터

### 회원가입 및 로그인

```python
# app/api/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from app.database import get_db
from app.schemas.auth import UserRegister, UserLogin, Token, UserResponse
from app.models.user import User
from app.core.security import verify_password, get_password_hash
from app.core.jwt import create_access_token, create_refresh_token

router = APIRouter(prefix="/auth", tags=["authentication"])

@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def register(user_data: UserRegister, db: Session = Depends(get_db)):
    """회원가입"""
    # 이메일 중복 체크
    existing_user = db.query(User).filter(User.email == user_data.email).first()
    if existing_user:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Email already registered"
        )

    # 사용자명 중복 체크
    existing_username = db.query(User).filter(User.username == user_data.username).first()
    if existing_username:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Username already taken"
        )

    # 비밀번호 해싱
    hashed_password = get_password_hash(user_data.password)

    # 사용자 생성
    user = User(
        email=user_data.email,
        username=user_data.username,
        hashed_password=hashed_password
    )

    db.add(user)
    db.commit()
    db.refresh(user)

    return user

@router.post("/login", response_model=Token)
def login(login_data: UserLogin, db: Session = Depends(get_db)):
    """로그인"""
    # 사용자 조회
    user = db.query(User).filter(User.email == login_data.email).first()

    if not user or not verify_password(login_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password"
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )

    # 토큰 생성
    access_token = create_access_token(data={"sub": user.id})
    refresh_token = create_refresh_token(data={"sub": user.id})

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }

@router.post("/refresh", response_model=Token)
def refresh_token(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
):
    """Refresh Token으로 새 Access Token 발급"""
    token = credentials.credentials

    # 토큰 검증
    payload = verify_token(token)
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token"
        )

    # Refresh Token 타입 확인
    if payload.get("type") != "refresh":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token type"
        )

    # 사용자 조회
    user_id = payload.get("sub")
    user = db.query(User).filter(User.id == user_id).first()

    if not user or not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found or inactive"
        )

    # 새 토큰 생성
    access_token = create_access_token(data={"sub": user.id})
    refresh_token = create_refresh_token(data={"sub": user.id})

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }
```

---

## 13.8 보호된 엔드포인트

### 인증이 필요한 API

```python
# app/api/routers/users.py
from fastapi import APIRouter, Depends
from app.api.deps import get_current_user, get_current_superuser
from app.models.user import User
from app.schemas.auth import UserResponse

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/me", response_model=UserResponse)
def get_current_user_info(current_user: User = Depends(get_current_user)):
    """현재 사용자 정보 조회"""
    return current_user

@router.put("/me", response_model=UserResponse)
def update_current_user(
    username: str,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """현재 사용자 정보 수정"""
    current_user.username = username
    db.commit()
    db.refresh(current_user)
    return current_user

@router.get("/all", response_model=List[UserResponse])
def list_all_users(
    current_user: User = Depends(get_current_superuser),  # 관리자만
    db: Session = Depends(get_db)
):
    """모든 사용자 조회 (관리자 전용)"""
    users = db.query(User).all()
    return users
```

---

## 13.9 클라이언트 사용 예제

### 회원가입

```bash
curl -X POST "http://localhost:8000/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "username": "john",
    "password": "secretpassword"
  }'
```

### 로그인

```bash
curl -X POST "http://localhost:8000/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "secretpassword"
  }'

# 응답:
# {
#   "access_token": "eyJhbGci...",
#   "refresh_token": "eyJhbGci...",
#   "token_type": "bearer"
# }
```

### 인증된 요청

```bash
curl -X GET "http://localhost:8000/users/me" \
  -H "Authorization: Bearer eyJhbGci..."
```

### Python 클라이언트

```python
import requests

# 로그인
response = requests.post(
    "http://localhost:8000/auth/login",
    json={"email": "user@example.com", "password": "secretpassword"}
)
tokens = response.json()
access_token = tokens["access_token"]

# 인증된 요청
headers = {"Authorization": f"Bearer {access_token}"}
response = requests.get("http://localhost:8000/users/me", headers=headers)
user = response.json()
print(user)
```

---

## 13.10 실전 예제: 완전한 인증 시스템

```python
# main.py
from fastapi import FastAPI
from app.api.routers import auth, users
from app.database import engine, Base

# 테이블 생성
Base.metadata.create_all(bind=engine)

app = FastAPI(title="JWT Authentication Example")

# 라우터 등록
app.include_router(auth.router, prefix="/api")
app.include_router(users.router, prefix="/api")

@app.get("/")
def root():
    return {"message": "JWT Authentication API"}

@app.get("/public")
def public_endpoint():
    """인증 불필요한 공개 엔드포인트"""
    return {"message": "This is a public endpoint"}
```

---

## 13.11 보안 Best Practices

### 1. SECRET_KEY 관리

```python
# .env 파일 사용
SECRET_KEY=very-long-random-secret-key-change-in-production-please

# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    SECRET_KEY: str

    class Config:
        env_file = ".env"

# SECRET_KEY 생성 방법
import secrets
print(secrets.token_urlsafe(32))
```

### 2. 토큰 만료 시간 설정

```python
# 짧은 Access Token (15-30분)
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# 긴 Refresh Token (7-30일)
REFRESH_TOKEN_EXPIRE_DAYS = 7
```

### 3. HTTPS 사용

```python
# 프로덕션에서는 반드시 HTTPS 사용
# Secure 쿠키 설정
from fastapi import Response

@app.post("/login")
def login(response: Response, ...):
    # ...
    response.set_cookie(
        key="refresh_token",
        value=refresh_token,
        httponly=True,
        secure=True,  # HTTPS only
        samesite="lax"
    )
```

### 4. 비밀번호 정책

```python
import re

def validate_password(password: str) -> bool:
    """
    비밀번호 정책:
    - 최소 8자
    - 대문자 1개 이상
    - 소문자 1개 이상
    - 숫자 1개 이상
    - 특수문자 1개 이상
    """
    if len(password) < 8:
        return False

    if not re.search(r"[A-Z]", password):
        return False

    if not re.search(r"[a-z]", password):
        return False

    if not re.search(r"\d", password):
        return False

    if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", password):
        return False

    return True
```

---

## 13.12 핵심 요약

### JWT 인증 흐름

1. **회원가입**: 비밀번호 해싱 후 저장
2. **로그인**: 비밀번호 검증 → Access Token & Refresh Token 발급
3. **인증 요청**: Authorization 헤더에 Bearer Token
4. **토큰 검증**: JWT 검증 → 사용자 조회
5. **토큰 갱신**: Refresh Token으로 새 Access Token 발급

### Spring Security vs FastAPI

| 기능 | Spring Security | FastAPI |
|------|----------------|---------|
| **설정** | Java Config 클래스 | Python 함수 |
| **필터** | OncePerRequestFilter | Depends() |
| **토큰 생성** | JwtTokenProvider | create_access_token() |
| **인증 확인** | @AuthenticationPrincipal | Depends(get_current_user) |
| **권한 체크** | @PreAuthorize | Depends(get_current_superuser) |

---

## 13.13 다음 챕터 예고

다음 챕터에서는 **OAuth2 인증**을 다룹니다:
- OAuth2 Password Flow
- OAuth2 Authorization Code Flow
- 소셜 로그인 (Google, GitHub)
- Scope와 권한 관리

---

[← 이전: Chapter 12. 고급 데이터베이스 기능](../part04-database/chapter12-advanced-db.md) | [목차로](../README.md) | [다음: Chapter 14. OAuth2 인증 →](chapter14-oauth2.md)
