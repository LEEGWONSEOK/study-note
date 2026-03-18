# Chapter 24: 프로젝트 구조 (Project Structure)

## 학습 목표
- 확장 가능한 FastAPI 프로젝트 구조 설계
- 레이어드 아키텍처 구현
- 모듈화와 패키지 구성
- 설정 관리 베스트 프랙티스
- Spring Boot의 패키지 구조와 비교

## 1. 프로젝트 구조 개요

### 1.1 작은 프로젝트 vs 대규모 프로젝트

**작은 프로젝트 (단일 파일):**
```
my_app/
├── main.py          # 모든 코드
└── requirements.txt
```

**중간 프로젝트:**
```
my_app/
├── app/
│   ├── main.py
│   ├── models.py
│   ├── schemas.py
│   └── database.py
├── tests/
└── requirements.txt
```

**대규모 프로젝트 (권장):**
```
my_fastapi_project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── core/              # 핵심 설정
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── security.py
│   │   └── database.py
│   ├── api/               # API 라우터
│   │   ├── __init__.py
│   │   ├── deps.py        # 의존성
│   │   └── v1/            # API 버전
│   │       ├── __init__.py
│   │       ├── endpoints/
│   │       │   ├── users.py
│   │       │   ├── items.py
│   │       │   └── auth.py
│   │       └── router.py
│   ├── models/            # SQLAlchemy 모델
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   ├── schemas/           # Pydantic 스키마
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   ├── crud/              # CRUD 작업
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── user.py
│   │   └── item.py
│   ├── services/          # 비즈니스 로직
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   └── auth_service.py
│   └── utils/             # 유틸리티
│       ├── __init__.py
│       └── helpers.py
├── alembic/               # DB 마이그레이션
│   ├── versions/
│   └── env.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── api/
│   └── services/
├── .env                   # 환경 변수
├── .env.example
├── alembic.ini
├── pyproject.toml
├── pytest.ini
└── README.md
```

### 1.2 Spring Boot와 비교

| 구분 | Spring Boot | FastAPI |
|------|-------------|---------|
| **엔트리 포인트** | `Application.java` (@SpringBootApplication) | `main.py` |
| **설정** | `application.yml`, `application.properties` | `.env`, `config.py` |
| **컨트롤러** | `@RestController` | `APIRouter` |
| **서비스** | `@Service` | `services/` 디렉토리 |
| **리포지토리** | `@Repository` (JPA) | `crud/` 디렉토리 |
| **모델** | `@Entity` | `models/` (SQLAlchemy) |
| **DTO** | Java Class | `schemas/` (Pydantic) |
| **의존성 주입** | `@Autowired` | `Depends()` |

**Spring Boot 구조:**
```
src/main/java/com/example/myapp/
├── MyAppApplication.java
├── config/
│   └── SecurityConfig.java
├── controller/
│   └── UserController.java
├── service/
│   ├── UserService.java
│   └── impl/
│       └── UserServiceImpl.java
├── repository/
│   └── UserRepository.java
├── domain/
│   └── User.java
└── dto/
    ├── UserRequestDto.java
    └── UserResponseDto.java
```

## 2. 핵심 구조 구현

### 2.1 설정 관리 (core/config.py)

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    """애플리케이션 설정"""

    # API 설정
    API_V1_STR: str = "/api/v1"
    PROJECT_NAME: str = "My FastAPI Project"
    VERSION: str = "1.0.0"

    # 데이터베이스
    DATABASE_URL: str

    # 보안
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # CORS
    BACKEND_CORS_ORIGINS: list[str] = ["http://localhost:3000"]

    # 환경
    ENVIRONMENT: str = "development"
    DEBUG: bool = True

    # 로깅
    LOG_LEVEL: str = "INFO"

    # Redis (선택적)
    REDIS_URL: Optional[str] = None

    # 이메일 (선택적)
    SMTP_HOST: Optional[str] = None
    SMTP_PORT: Optional[int] = None
    SMTP_USER: Optional[str] = None
    SMTP_PASSWORD: Optional[str] = None

    class Config:
        env_file = ".env"
        case_sensitive = True

# 싱글톤 인스턴스
settings = Settings()
```

**.env:**
```env
# API
API_V1_STR=/api/v1
PROJECT_NAME=My FastAPI Project

# Database
DATABASE_URL=postgresql://user:password@localhost/dbname

# Security
SECRET_KEY=your-secret-key-here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# CORS
BACKEND_CORS_ORIGINS=["http://localhost:3000","http://localhost:8080"]

# Environment
ENVIRONMENT=development
DEBUG=True

# Logging
LOG_LEVEL=INFO
```

### 2.2 데이터베이스 설정 (core/database.py)

```python
# app/core/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from app.core.config import settings

# 엔진 생성
engine = create_engine(
    settings.DATABASE_URL,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20
)

# 세션 팩토리
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Base 클래스
Base = declarative_base()

# 의존성
def get_db():
    """DB 세션 의존성"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 2.3 모델 정의 (models/)

```python
# app/models/user.py
from sqlalchemy import Boolean, Column, Integer, String, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from app.core.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True, nullable=False)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # 관계
    items = relationship("Item", back_populates="owner")

# app/models/item.py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from app.core.database import Base

class Item(Base):
    __tablename__ = "items"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False)
    description = Column(String)
    owner_id = Column(Integer, ForeignKey("users.id"))

    # 관계
    owner = relationship("User", back_populates="items")

# app/models/__init__.py
from app.models.user import User
from app.models.item import Item

__all__ = ["User", "Item"]
```

### 2.4 스키마 정의 (schemas/)

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime
from typing import Optional

# 기본 속성
class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr

# 생성 시 사용
class UserCreate(UserBase):
    password: str = Field(..., min_length=8)

# 업데이트 시 사용 (모든 필드 선택적)
class UserUpdate(BaseModel):
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    email: Optional[EmailStr] = None
    password: Optional[str] = Field(None, min_length=8)

# DB에서 읽을 때 사용
class UserInDB(UserBase):
    id: int
    is_active: bool
    is_superuser: bool
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True

# API 응답
class User(UserInDB):
    pass

# 로그인 응답
class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"

class TokenPayload(BaseModel):
    sub: Optional[int] = None

# app/schemas/__init__.py
from app.schemas.user import (
    User, UserCreate, UserUpdate, UserInDB,
    Token, TokenPayload
)
from app.schemas.item import Item, ItemCreate, ItemUpdate

__all__ = [
    "User", "UserCreate", "UserUpdate", "UserInDB",
    "Token", "TokenPayload",
    "Item", "ItemCreate", "ItemUpdate"
]
```

### 2.5 CRUD 레이어 (crud/)

```python
# app/crud/base.py
from typing import Any, Dict, Generic, List, Optional, Type, TypeVar, Union
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel
from sqlalchemy.orm import Session
from app.core.database import Base

ModelType = TypeVar("ModelType", bound=Base)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)

class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    """기본 CRUD 작업"""

    def __init__(self, model: Type[ModelType]):
        self.model = model

    def get(self, db: Session, id: Any) -> Optional[ModelType]:
        """ID로 조회"""
        return db.query(self.model).filter(self.model.id == id).first()

    def get_multi(
        self, db: Session, *, skip: int = 0, limit: int = 100
    ) -> List[ModelType]:
        """목록 조회"""
        return db.query(self.model).offset(skip).limit(limit).all()

    def create(self, db: Session, *, obj_in: CreateSchemaType) -> ModelType:
        """생성"""
        obj_in_data = jsonable_encoder(obj_in)
        db_obj = self.model(**obj_in_data)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def update(
        self,
        db: Session,
        *,
        db_obj: ModelType,
        obj_in: Union[UpdateSchemaType, Dict[str, Any]]
    ) -> ModelType:
        """업데이트"""
        obj_data = jsonable_encoder(db_obj)
        if isinstance(obj_in, dict):
            update_data = obj_in
        else:
            update_data = obj_in.dict(exclude_unset=True)

        for field in obj_data:
            if field in update_data:
                setattr(db_obj, field, update_data[field])

        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def remove(self, db: Session, *, id: int) -> ModelType:
        """삭제"""
        obj = db.query(self.model).get(id)
        db.delete(obj)
        db.commit()
        return obj

# app/crud/user.py
from typing import Optional
from sqlalchemy.orm import Session
from app.crud.base import CRUDBase
from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate
from app.core.security import get_password_hash, verify_password

class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
    """사용자 CRUD 작업"""

    def get_by_email(self, db: Session, *, email: str) -> Optional[User]:
        """이메일로 조회"""
        return db.query(User).filter(User.email == email).first()

    def get_by_username(self, db: Session, *, username: str) -> Optional[User]:
        """사용자명으로 조회"""
        return db.query(User).filter(User.username == username).first()

    def create(self, db: Session, *, obj_in: UserCreate) -> User:
        """사용자 생성 (비밀번호 해싱)"""
        db_obj = User(
            username=obj_in.username,
            email=obj_in.email,
            hashed_password=get_password_hash(obj_in.password),
        )
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj

    def authenticate(
        self, db: Session, *, username: str, password: str
    ) -> Optional[User]:
        """인증"""
        user = self.get_by_username(db, username=username)
        if not user:
            return None
        if not verify_password(password, user.hashed_password):
            return None
        return user

    def is_active(self, user: User) -> bool:
        """활성 사용자 확인"""
        return user.is_active

    def is_superuser(self, user: User) -> bool:
        """슈퍼유저 확인"""
        return user.is_superuser

# 인스턴스 생성
user = CRUDUser(User)

# app/crud/__init__.py
from app.crud.user import user
from app.crud.item import item

__all__ = ["user", "item"]
```

### 2.6 서비스 레이어 (services/)

```python
# app/services/user_service.py
from typing import Optional
from sqlalchemy.orm import Session
from fastapi import HTTPException, status
from app import crud, schemas
from app.models.user import User

class UserService:
    """사용자 비즈니스 로직"""

    @staticmethod
    def create_user(db: Session, user_in: schemas.UserCreate) -> User:
        """사용자 생성"""
        # 중복 확인
        user = crud.user.get_by_email(db, email=user_in.email)
        if user:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="Email already registered"
            )

        user = crud.user.get_by_username(db, username=user_in.username)
        if user:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="Username already taken"
            )

        # 생성
        return crud.user.create(db, obj_in=user_in)

    @staticmethod
    def get_user(db: Session, user_id: int) -> User:
        """사용자 조회"""
        user = crud.user.get(db, id=user_id)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found"
            )
        return user

    @staticmethod
    def update_user(
        db: Session,
        user_id: int,
        user_update: schemas.UserUpdate,
        current_user: User
    ) -> User:
        """사용자 업데이트"""
        user = crud.user.get(db, id=user_id)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found"
            )

        # 권한 확인
        if user.id != current_user.id and not current_user.is_superuser:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions"
            )

        return crud.user.update(db, db_obj=user, obj_in=user_update)

    @staticmethod
    def delete_user(db: Session, user_id: int, current_user: User) -> User:
        """사용자 삭제"""
        user = crud.user.get(db, id=user_id)
        if not user:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="User not found"
            )

        # 권한 확인
        if not current_user.is_superuser:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions"
            )

        return crud.user.remove(db, id=user_id)

# app/services/__init__.py
from app.services.user_service import UserService
from app.services.auth_service import AuthService

__all__ = ["UserService", "AuthService"]
```

### 2.7 API 엔드포인트 (api/v1/endpoints/)

```python
# app/api/deps.py
from typing import Generator
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
from sqlalchemy.orm import Session
from app import crud, models, schemas
from app.core import security
from app.core.config import settings
from app.core.database import get_db

oauth2_scheme = OAuth2PasswordBearer(tokenUrl=f"{settings.API_V1_STR}/auth/login")

def get_current_user(
    db: Session = Depends(get_db),
    token: str = Depends(oauth2_scheme)
) -> models.User:
    """현재 사용자 가져오기"""
    try:
        payload = jwt.decode(
            token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM]
        )
        token_data = schemas.TokenPayload(**payload)
    except JWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials"
        )

    user = crud.user.get(db, id=token_data.sub)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user

def get_current_active_user(
    current_user: models.User = Depends(get_current_user),
) -> models.User:
    """활성 사용자 확인"""
    if not crud.user.is_active(current_user):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Inactive user"
        )
    return current_user

def get_current_superuser(
    current_user: models.User = Depends(get_current_user),
) -> models.User:
    """슈퍼유저 확인"""
    if not crud.user.is_superuser(current_user):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    return current_user

# app/api/v1/endpoints/users.py
from typing import List
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session
from app import schemas
from app.api import deps
from app.services import UserService
from app.models.user import User

router = APIRouter()

@router.post("/", response_model=schemas.User, status_code=status.HTTP_201_CREATED)
def create_user(
    *,
    db: Session = Depends(deps.get_db),
    user_in: schemas.UserCreate,
):
    """사용자 생성"""
    return UserService.create_user(db, user_in)

@router.get("/me", response_model=schemas.User)
def read_user_me(
    current_user: User = Depends(deps.get_current_active_user),
):
    """현재 사용자 정보"""
    return current_user

@router.get("/{user_id}", response_model=schemas.User)
def read_user(
    user_id: int,
    db: Session = Depends(deps.get_db),
):
    """사용자 조회"""
    return UserService.get_user(db, user_id)

@router.put("/{user_id}", response_model=schemas.User)
def update_user(
    user_id: int,
    user_in: schemas.UserUpdate,
    db: Session = Depends(deps.get_db),
    current_user: User = Depends(deps.get_current_active_user),
):
    """사용자 업데이트"""
    return UserService.update_user(db, user_id, user_in, current_user)

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(
    user_id: int,
    db: Session = Depends(deps.get_db),
    current_user: User = Depends(deps.get_current_superuser),
):
    """사용자 삭제"""
    UserService.delete_user(db, user_id, current_user)

# app/api/v1/router.py
from fastapi import APIRouter
from app.api.v1.endpoints import users, items, auth

api_router = APIRouter()
api_router.include_router(auth.router, prefix="/auth", tags=["auth"])
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(items.router, prefix="/items", tags=["items"])
```

### 2.8 메인 애플리케이션 (main.py)

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings
from app.api.v1.router import api_router

def create_application() -> FastAPI:
    """애플리케이션 생성 팩토리"""
    app = FastAPI(
        title=settings.PROJECT_NAME,
        version=settings.VERSION,
        openapi_url=f"{settings.API_V1_STR}/openapi.json",
        docs_url=f"{settings.API_V1_STR}/docs",
        redoc_url=f"{settings.API_V1_STR}/redoc",
    )

    # CORS 설정
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.BACKEND_CORS_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # 라우터 등록
    app.include_router(api_router, prefix=settings.API_V1_STR)

    @app.get("/health")
    def health_check():
        """헬스 체크"""
        return {"status": "healthy"}

    return app

app = create_application()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 3. 고급 구조 패턴

### 3.1 도메인 주도 설계 (DDD) 구조

```
app/
├── domain/                 # 도메인 레이어
│   ├── users/
│   │   ├── __init__.py
│   │   ├── models.py      # 도메인 모델
│   │   ├── repository.py  # 리포지토리 인터페이스
│   │   ├── service.py     # 도메인 서비스
│   │   └── exceptions.py  # 도메인 예외
│   └── items/
│       └── ...
├── infrastructure/         # 인프라 레이어
│   ├── database/
│   │   ├── repositories/  # 리포지토리 구현
│   │   └── models.py      # ORM 모델
│   ├── external/
│   │   └── api_client.py
│   └── cache/
│       └── redis_cache.py
├── application/            # 애플리케이션 레이어
│   ├── users/
│   │   ├── commands.py    # 커맨드 (쓰기)
│   │   ├── queries.py     # 쿼리 (읽기)
│   │   └── dtos.py        # DTO
│   └── items/
│       └── ...
└── presentation/           # 프레젠테이션 레이어
    ├── api/
    │   └── v1/
    │       └── endpoints/
    └── schemas/
```

### 3.2 기능별 구조 (Feature-based)

```
app/
├── features/
│   ├── users/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── crud.py
│   │   ├── service.py
│   │   ├── router.py
│   │   └── dependencies.py
│   ├── items/
│   │   └── ...
│   └── auth/
│       └── ...
├── core/
│   ├── config.py
│   ├── database.py
│   └── security.py
└── main.py
```

## 4. 설정 관리 패턴

### 4.1 환경별 설정

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from typing import Literal

class BaseConfig(BaseSettings):
    """기본 설정"""
    PROJECT_NAME: str = "My FastAPI Project"
    API_V1_STR: str = "/api/v1"

    class Config:
        case_sensitive = True

class DevelopmentConfig(BaseConfig):
    """개발 환경"""
    DEBUG: bool = True
    DATABASE_URL: str = "sqlite:///./dev.db"
    LOG_LEVEL: str = "DEBUG"

class ProductionConfig(BaseConfig):
    """프로덕션 환경"""
    DEBUG: bool = False
    DATABASE_URL: str  # 환경 변수에서 읽음
    LOG_LEVEL: str = "WARNING"

class TestConfig(BaseConfig):
    """테스트 환경"""
    TESTING: bool = True
    DATABASE_URL: str = "sqlite:///./test.db"

# 환경에 따라 설정 선택
def get_settings(env: Literal["development", "production", "test"] = "development"):
    config_map = {
        "development": DevelopmentConfig,
        "production": ProductionConfig,
        "test": TestConfig,
    }
    return config_map[env]()

# 사용
import os
ENVIRONMENT = os.getenv("ENVIRONMENT", "development")
settings = get_settings(ENVIRONMENT)
```

## 5. 유틸리티와 헬퍼

### 5.1 공통 유틸리티

```python
# app/utils/pagination.py
from typing import Generic, List, TypeVar
from pydantic import BaseModel
from sqlalchemy.orm import Query

T = TypeVar("T")

class Page(BaseModel, Generic[T]):
    """페이지네이션 응답"""
    items: List[T]
    total: int
    page: int
    size: int
    pages: int

def paginate(query: Query, page: int = 1, size: int = 50) -> Page:
    """쿼리 페이지네이션"""
    total = query.count()
    items = query.offset((page - 1) * size).limit(size).all()
    pages = (total + size - 1) // size

    return Page(
        items=items,
        total=total,
        page=page,
        size=size,
        pages=pages
    )

# app/utils/logger.py
import logging
from app.core.config import settings

def get_logger(name: str) -> logging.Logger:
    """로거 생성"""
    logger = logging.getLogger(name)
    logger.setLevel(settings.LOG_LEVEL)

    handler = logging.StreamHandler()
    formatter = logging.Formatter(
        "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger
```

## 6. 요약

### 핵심 포인트

1. **모듈화된 구조**
   - 기능별 또는 레이어별 분리
   - 명확한 책임 분리

2. **레이어드 아키텍처**
   - API → Service → CRUD → Model
   - 각 레이어의 독립성

3. **설정 관리**
   - pydantic-settings 사용
   - 환경별 설정 분리

4. **Spring Boot와 유사성**
   - Controller → Router
   - Service → Service
   - Repository → CRUD

5. **확장성**
   - 새 기능 추가 용이
   - 테스트 가능한 구조

### 다음 챕터 예고

**Chapter 25: 디자인 패턴 (Design Patterns)**

다음 챕터에서는 FastAPI에서 활용할 수 있는 디자인 패턴을 다룹니다:
- Repository 패턴
- Service 패턴
- Factory 패턴
- Singleton 패턴
- Strategy 패턴
- Dependency Injection 패턴

잘 구조화된 프로젝트는 유지보수와 확장을 쉽게 만듭니다!