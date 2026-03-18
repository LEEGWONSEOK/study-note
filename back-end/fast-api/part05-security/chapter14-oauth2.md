# Chapter 14. OAuth2 인증

## 14.1 OAuth2란?

OAuth2는 인증(Authentication)과 인가(Authorization)를 위한 산업 표준 프로토콜입니다. FastAPI는 OAuth2를 기본적으로 지원하며, Spring Security OAuth2와 유사한 방식으로 구현할 수 있습니다.

### OAuth2 Flow 종류

| Flow | 설명 | 사용 사례 |
|------|------|----------|
| **Password Flow** | 사용자명/비밀번호로 직접 인증 | 자체 앱 (신뢰할 수 있는 클라이언트) |
| **Authorization Code** | 인가 코드를 통한 인증 | 웹 애플리케이션 |
| **Implicit** | 토큰 직접 반환 (비권장) | SPA (Single Page Application) |
| **Client Credentials** | 클라이언트 자격증명 | 서버 간 통신 |

### Spring Security OAuth2 vs FastAPI

```java
// Spring Security OAuth2
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
            .withClient("client-id")
            .secret("client-secret")
            .authorizedGrantTypes("password", "refresh_token")
            .scopes("read", "write");
    }
}
```

```python
# FastAPI OAuth2
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # 인증 로직
    return {"access_token": token, "token_type": "bearer"}
```

---

## 14.2 OAuth2 Password Flow

FastAPI가 제공하는 가장 간단한 OAuth2 구현입니다.

### 기본 설정

```python
# app/core/oauth2.py
from fastapi.security import OAuth2PasswordBearer

# tokenUrl: 토큰을 얻기 위한 엔드포인트
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/token")
```

### 토큰 엔드포인트

```python
# app/api/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from datetime import timedelta

from app.database import get_db
from app.models.user import User
from app.core.security import verify_password
from app.core.jwt import create_access_token, create_refresh_token
from app.core.config import settings

router = APIRouter(prefix="/auth", tags=["authentication"])

@router.post("/token")
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """
    OAuth2 호환 토큰 로그인
    - username: 이메일 또는 사용자명
    - password: 비밀번호
    """
    # 이메일 또는 사용자명으로 조회
    user = (
        db.query(User)
        .filter(
            (User.email == form_data.username) |
            (User.username == form_data.username)
        )
        .first()
    )

    # 사용자 검증
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )

    # 토큰 생성
    access_token_expires = timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": str(user.id)},
        expires_delta=access_token_expires
    )

    refresh_token = create_refresh_token(data={"sub": str(user.id)})

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }
```

**Spring Security와 비교:**
```java
// Spring Security OAuth2
@PostMapping("/oauth/token")
public ResponseEntity<TokenResponse> token(
    @RequestParam("grant_type") String grantType,
    @RequestParam("username") String username,
    @RequestParam("password") String password
) {
    Authentication authentication = authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(username, password)
    );

    String accessToken = jwtTokenProvider.generateToken(authentication);
    return ResponseEntity.ok(new TokenResponse(accessToken, "bearer"));
}
```

### OAuth2 인증 의존성

```python
# app/api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from jose import JWTError, jwt

from app.database import get_db
from app.core.config import settings
from app.core.oauth2 import oauth2_scheme
from app.models.user import User

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """OAuth2 토큰으로 현재 사용자 가져오기"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = db.query(User).filter(User.id == int(user_id)).first()
    if user is None:
        raise credentials_exception

    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """활성 사용자만 허용"""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )
    return current_user
```

---

## 14.3 Scope를 이용한 권한 관리

OAuth2의 Scope를 사용하여 세밀한 권한 제어가 가능합니다.

### Scope 정의

```python
# app/core/oauth2.py
from fastapi.security import OAuth2PasswordBearer, SecurityScopes

# Scope 지원 추가
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="/api/auth/token",
    scopes={
        "read": "Read access",
        "write": "Write access",
        "admin": "Admin access"
    }
)
```

### 토큰에 Scope 포함

```python
# app/core/jwt.py
def create_access_token(
    data: dict,
    expires_delta: Optional[timedelta] = None,
    scopes: List[str] = None
) -> str:
    """Scope를 포함한 Access Token 생성"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({
        "exp": expire,
        "scopes": scopes or []
    })

    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

# 로그인 시 Scope 포함
@router.post("/token")
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    # ... 사용자 검증 ...

    # 사용자 역할에 따라 Scope 결정
    scopes = ["read", "write"]
    if user.is_superuser:
        scopes.append("admin")

    access_token = create_access_token(
        data={"sub": str(user.id)},
        scopes=scopes
    )

    return {"access_token": access_token, "token_type": "bearer"}
```

### Scope 검증

```python
# app/api/deps.py
from fastapi import Security
from fastapi.security import SecurityScopes

async def get_current_user_with_scopes(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """Scope를 확인하며 현재 사용자 가져오기"""

    if security_scopes.scopes:
        authenticate_value = f'Bearer scope="{security_scopes.scope_str}"'
    else:
        authenticate_value = "Bearer"

    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": authenticate_value},
    )

    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception

        token_scopes = payload.get("scopes", [])
    except JWTError:
        raise credentials_exception

    # Scope 검증
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions",
                headers={"WWW-Authenticate": authenticate_value},
            )

    user = db.query(User).filter(User.id == int(user_id)).first()
    if user is None:
        raise credentials_exception

    return user
```

### Scope 사용 예제

```python
# app/api/routers/items.py
from fastapi import APIRouter, Security
from app.api.deps import get_current_user_with_scopes

router = APIRouter(prefix="/items", tags=["items"])

@router.get("/")
async def read_items(
    current_user: User = Security(get_current_user_with_scopes, scopes=["read"])
):
    """아이템 조회 - read 권한 필요"""
    return {"items": [...]}

@router.post("/")
async def create_item(
    item: ItemCreate,
    current_user: User = Security(get_current_user_with_scopes, scopes=["write"])
):
    """아이템 생성 - write 권한 필요"""
    return {"item": item}

@router.delete("/{item_id}")
async def delete_item(
    item_id: int,
    current_user: User = Security(get_current_user_with_scopes, scopes=["admin"])
):
    """아이템 삭제 - admin 권한 필요"""
    return {"deleted": item_id}
```

**Spring Security와 비교:**
```java
// Spring Security - @PreAuthorize
@GetMapping("/items")
@PreAuthorize("hasAuthority('SCOPE_read')")
public List<Item> getItems() {
    return itemService.findAll();
}

@PostMapping("/items")
@PreAuthorize("hasAuthority('SCOPE_write')")
public Item createItem(@RequestBody Item item) {
    return itemService.save(item);
}

@DeleteMapping("/items/{id}")
@PreAuthorize("hasAuthority('SCOPE_admin')")
public void deleteItem(@PathVariable Long id) {
    itemService.delete(id);
}
```

---

## 14.4 소셜 로그인 - Google OAuth2

### Google OAuth2 설정

```python
# app/core/config.py
class Settings(BaseSettings):
    # ... 기존 설정 ...

    # Google OAuth2
    GOOGLE_CLIENT_ID: str = ""
    GOOGLE_CLIENT_SECRET: str = ""
    GOOGLE_REDIRECT_URI: str = "http://localhost:8000/api/auth/google/callback"

    class Config:
        env_file = ".env"

settings = Settings()
```

### Google OAuth2 라우터

```python
# app/api/routers/oauth.py
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import RedirectResponse
from sqlalchemy.orm import Session
import requests

from app.database import get_db
from app.core.config import settings
from app.core.jwt import create_access_token, create_refresh_token
from app.models.user import User

router = APIRouter(prefix="/auth", tags=["oauth"])

@router.get("/google/login")
async def google_login():
    """Google 로그인 페이지로 리다이렉트"""
    google_auth_url = (
        "https://accounts.google.com/o/oauth2/v2/auth"
        f"?client_id={settings.GOOGLE_CLIENT_ID}"
        f"&redirect_uri={settings.GOOGLE_REDIRECT_URI}"
        "&response_type=code"
        "&scope=openid email profile"
    )
    return RedirectResponse(url=google_auth_url)

@router.get("/google/callback")
async def google_callback(code: str, db: Session = Depends(get_db)):
    """Google OAuth2 콜백"""
    # 1. 인가 코드로 액세스 토큰 요청
    token_url = "https://oauth2.googleapis.com/token"
    token_data = {
        "code": code,
        "client_id": settings.GOOGLE_CLIENT_ID,
        "client_secret": settings.GOOGLE_CLIENT_SECRET,
        "redirect_uri": settings.GOOGLE_REDIRECT_URI,
        "grant_type": "authorization_code"
    }

    token_response = requests.post(token_url, data=token_data)
    token_response.raise_for_status()
    tokens = token_response.json()
    access_token = tokens["access_token"]

    # 2. 액세스 토큰으로 사용자 정보 요청
    userinfo_url = "https://www.googleapis.com/oauth2/v2/userinfo"
    headers = {"Authorization": f"Bearer {access_token}"}
    userinfo_response = requests.get(userinfo_url, headers=headers)
    userinfo_response.raise_for_status()
    user_info = userinfo_response.json()

    # 3. 사용자 조회 또는 생성
    email = user_info["email"]
    user = db.query(User).filter(User.email == email).first()

    if not user:
        # 새 사용자 생성
        user = User(
            email=email,
            username=user_info.get("name", email.split("@")[0]),
            hashed_password="",  # 소셜 로그인은 비밀번호 없음
            is_active=True
        )
        db.add(user)
        db.commit()
        db.refresh(user)

    # 4. JWT 토큰 생성
    jwt_access_token = create_access_token(data={"sub": str(user.id)})
    jwt_refresh_token = create_refresh_token(data={"sub": str(user.id)})

    # 5. 프론트엔드로 리다이렉트 (토큰 포함)
    frontend_url = f"http://localhost:3000/auth/callback?token={jwt_access_token}"
    return RedirectResponse(url=frontend_url)
```

**Spring Security OAuth2와 비교:**
```java
// Spring Security OAuth2
@Configuration
public class OAuth2Config {
    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
        ClientRegistrationRepository clientRegistrationRepository,
        OAuth2AuthorizedClientRepository authorizedClientRepository
    ) {
        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()
                .refreshToken()
                .build();

        DefaultOAuth2AuthorizedClientManager authorizedClientManager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository,
                authorizedClientRepository
            );
        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);

        return authorizedClientManager;
    }
}

// application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,email,profile
```

---

## 14.5 GitHub OAuth2

### GitHub OAuth2 설정

```python
# app/core/config.py
class Settings(BaseSettings):
    # ... 기존 설정 ...

    # GitHub OAuth2
    GITHUB_CLIENT_ID: str = ""
    GITHUB_CLIENT_SECRET: str = ""
    GITHUB_REDIRECT_URI: str = "http://localhost:8000/api/auth/github/callback"

settings = Settings()
```

### GitHub OAuth2 라우터

```python
# app/api/routers/oauth.py
@router.get("/github/login")
async def github_login():
    """GitHub 로그인 페이지로 리다이렉트"""
    github_auth_url = (
        "https://github.com/login/oauth/authorize"
        f"?client_id={settings.GITHUB_CLIENT_ID}"
        f"&redirect_uri={settings.GITHUB_REDIRECT_URI}"
        "&scope=user:email"
    )
    return RedirectResponse(url=github_auth_url)

@router.get("/github/callback")
async def github_callback(code: str, db: Session = Depends(get_db)):
    """GitHub OAuth2 콜백"""
    # 1. 인가 코드로 액세스 토큰 요청
    token_url = "https://github.com/login/oauth/access_token"
    token_data = {
        "code": code,
        "client_id": settings.GITHUB_CLIENT_ID,
        "client_secret": settings.GITHUB_CLIENT_SECRET,
        "redirect_uri": settings.GITHUB_REDIRECT_URI
    }
    headers = {"Accept": "application/json"}

    token_response = requests.post(token_url, data=token_data, headers=headers)
    token_response.raise_for_status()
    tokens = token_response.json()
    access_token = tokens["access_token"]

    # 2. 액세스 토큰으로 사용자 정보 요청
    userinfo_url = "https://api.github.com/user"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Accept": "application/json"
    }
    userinfo_response = requests.get(userinfo_url, headers=headers)
    userinfo_response.raise_for_status()
    user_info = userinfo_response.json()

    # 이메일 조회 (별도 API)
    email_url = "https://api.github.com/user/emails"
    email_response = requests.get(email_url, headers=headers)
    email_response.raise_for_status()
    emails = email_response.json()

    # Primary 이메일 찾기
    primary_email = next(
        (email["email"] for email in emails if email["primary"]),
        None
    )

    if not primary_email:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="No email found for GitHub account"
        )

    # 3. 사용자 조회 또는 생성
    user = db.query(User).filter(User.email == primary_email).first()

    if not user:
        user = User(
            email=primary_email,
            username=user_info.get("login", primary_email.split("@")[0]),
            hashed_password="",
            is_active=True
        )
        db.add(user)
        db.commit()
        db.refresh(user)

    # 4. JWT 토큰 생성 및 리다이렉트
    jwt_access_token = create_access_token(data={"sub": str(user.id)})
    jwt_refresh_token = create_refresh_token(data={"sub": str(user.id)})

    frontend_url = f"http://localhost:3000/auth/callback?token={jwt_access_token}"
    return RedirectResponse(url=frontend_url)
```

---

## 14.6 통합 소셜 로그인 시스템

### Provider Enum

```python
# app/models/user.py
from enum import Enum as PyEnum

class AuthProvider(str, PyEnum):
    LOCAL = "local"
    GOOGLE = "google"
    GITHUB = "github"
    FACEBOOK = "facebook"

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(100), unique=True, nullable=False, index=True)
    username = Column(String(50), unique=True, nullable=False)
    hashed_password = Column(String(255))  # nullable for social login
    is_active = Column(Boolean, default=True)
    auth_provider = Column(String(20), default=AuthProvider.LOCAL)  # 추가
    provider_user_id = Column(String(100))  # 소셜 플랫폼의 사용자 ID
    created_at = Column(DateTime, server_default=func.now())
```

### 통합 OAuth2 핸들러

```python
# app/services/oauth.py
from typing import Dict, Any
from sqlalchemy.orm import Session
from app.models.user import User, AuthProvider

class OAuthService:
    @staticmethod
    def get_or_create_user(
        db: Session,
        email: str,
        username: str,
        provider: AuthProvider,
        provider_user_id: str
    ) -> User:
        """OAuth 제공자로부터 사용자 조회 또는 생성"""

        # 이메일로 기존 사용자 조회
        user = db.query(User).filter(User.email == email).first()

        if user:
            # 기존 사용자의 provider 정보 업데이트
            if not user.auth_provider:
                user.auth_provider = provider
                user.provider_user_id = provider_user_id
                db.commit()
                db.refresh(user)
            return user

        # 새 사용자 생성
        user = User(
            email=email,
            username=username,
            hashed_password="",  # 소셜 로그인은 비밀번호 없음
            is_active=True,
            auth_provider=provider,
            provider_user_id=provider_user_id
        )
        db.add(user)
        db.commit()
        db.refresh(user)

        return user
```

---

## 14.7 프론트엔드 통합 예제

### React 예제

```javascript
// Login.jsx
import React from 'react';

function Login() {
  const handleGoogleLogin = () => {
    window.location.href = 'http://localhost:8000/api/auth/google/login';
  };

  const handleGithubLogin = () => {
    window.location.href = 'http://localhost:8000/api/auth/github/login';
  };

  return (
    <div>
      <h1>로그인</h1>
      <button onClick={handleGoogleLogin}>Google로 로그인</button>
      <button onClick={handleGithubLogin}>GitHub로 로그인</button>
    </div>
  );
}

// AuthCallback.jsx
import { useEffect } from 'react';
import { useLocation, useNavigate } from 'react-router-dom';

function AuthCallback() {
  const location = useLocation();
  const navigate = useNavigate();

  useEffect(() => {
    const params = new URLSearchParams(location.search);
    const token = params.get('token');

    if (token) {
      // 토큰 저장
      localStorage.setItem('access_token', token);
      // 홈으로 이동
      navigate('/');
    }
  }, [location, navigate]);

  return <div>로그인 처리 중...</div>;
}
```

---

## 14.8 핵심 요약

### OAuth2 Flow 비교

| Flow | 장점 | 단점 | 사용 사례 |
|------|------|------|----------|
| **Password** | 간단함 | 보안 취약 | 자체 앱 |
| **Authorization Code** | 안전함 | 복잡함 | 웹 앱 |
| **Client Credentials** | 서버 간 통신 | 사용자 컨텍스트 없음 | API to API |

### Spring Security OAuth2 vs FastAPI

| 기능 | Spring Security | FastAPI |
|------|----------------|---------|
| **설정** | application.yml + Java Config | Python 코드 |
| **토큰 발급** | OAuth2AuthorizationServer | 직접 구현 |
| **소셜 로그인** | spring-security-oauth2-client | 직접 구현 |
| **Scope** | @PreAuthorize | Security(scopes=[]) |

### 기억할 핵심

1. ✅ **OAuth2PasswordBearer**: 토큰 기반 인증
2. ✅ **SecurityScopes**: 세밀한 권한 제어
3. ✅ **소셜 로그인**: 인가 코드 → 액세스 토큰 → 사용자 정보
4. ✅ **Provider**: 여러 인증 방식 통합 관리

---

## 14.9 실습 예제

### 실습 1: Scope 기반 API

**요구사항:**
- read scope: 게시글 조회
- write scope: 게시글 작성
- admin scope: 게시글 삭제

### 실습 2: Google OAuth2 통합

**요구사항:**
- Google 로그인 버튼 구현
- 콜백 처리
- 사용자 정보 저장

---

## 14.10 다음 챕터 예고

다음 챕터에서는 **권한과 역할 관리**를 다룹니다:
- Role-Based Access Control (RBAC)
- Permission 시스템
- 동적 권한 체크
- Spring Security의 @PreAuthorize와 비교

---

[← 이전: Chapter 13. JWT 인증](chapter13-jwt.md) | [목차로](../README.md) | [다음: Chapter 15. 권한과 역할 관리 →](chapter15-authorization.md)
