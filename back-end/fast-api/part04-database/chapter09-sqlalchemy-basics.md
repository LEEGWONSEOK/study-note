# Chapter 9. SQLAlchemy 기초

## 9.1 SQLAlchemy란?

SQLAlchemy는 Python의 가장 인기 있는 ORM(Object-Relational Mapping) 라이브러리입니다. Spring Boot의 JPA/Hibernate와 유사한 역할을 합니다.

### Spring Boot JPA vs SQLAlchemy

| 기능 | Spring Boot (JPA/Hibernate) | FastAPI (SQLAlchemy) |
|------|----------------------------|---------------------|
| **ORM** | JPA 표준 + Hibernate 구현 | SQLAlchemy ORM |
| **엔티티** | `@Entity` 클래스 | `declarative_base()` 클래스 |
| **기본키** | `@Id`, `@GeneratedValue` | `Column(primary_key=True)` |
| **컬럼** | `@Column` | `Column()` |
| **관계** | `@OneToMany`, `@ManyToOne` | `relationship()` |
| **쿼리** | JPQL, QueryDSL | SQLAlchemy Query API |
| **트랜잭션** | `@Transactional` | Session commit/rollback |
| **마이그레이션** | Flyway, Liquibase | Alembic |

---

## 9.2 설치 및 설정

### 패키지 설치

```bash
# SQLAlchemy 설치
pip install sqlalchemy

# 데이터베이스 드라이버 설치
pip install psycopg2-binary  # PostgreSQL
pip install pymysql          # MySQL
pip install asyncpg          # PostgreSQL (비동기)
```

### 데이터베이스 연결 설정

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

# 데이터베이스 URL
# PostgreSQL
SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"

# MySQL
# SQLALCHEMY_DATABASE_URL = "mysql+pymysql://user:password@localhost/dbname"

# SQLite (개발용)
# SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

# 엔진 생성
engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    # SQLite만 필요
    # connect_args={"check_same_thread": False}
)

# 세션 팩토리 생성
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Base 클래스 생성
Base = declarative_base()
```

**Spring Boot와 비교:**
```yaml
# Spring Boot - application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost/dbname
    username: user
    password: password
  jpa:
    hibernate:
      ddl-auto: update
```

---

## 9.3 모델(엔티티) 정의

### 기본 모델

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from datetime import datetime

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(100), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

**Spring Boot JPA와 비교:**
```java
// Spring Boot
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 50, unique = true, nullable = false)
    private String username;

    @Column(length = 100, unique = true, nullable = false)
    private String email;

    @Column(length = 255, nullable = false)
    private String hashedPassword;

    @Column(nullable = false)
    private Boolean isActive = true;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

### 컬럼 타입

```python
from sqlalchemy import (
    Integer, String, Text, Boolean,
    Float, Numeric, Date, DateTime, Time,
    JSON, ARRAY, Enum
)
from enum import Enum as PyEnum

class UserRole(str, PyEnum):
    ADMIN = "admin"
    USER = "user"
    GUEST = "guest"

class Product(Base):
    __tablename__ = "products"

    id = Column(Integer, primary_key=True)
    name = Column(String(200), nullable=False)           # VARCHAR(200)
    description = Column(Text)                           # TEXT
    price = Column(Numeric(10, 2), nullable=False)       # DECIMAL(10,2)
    stock = Column(Integer, default=0)                   # INTEGER
    is_available = Column(Boolean, default=True)         # BOOLEAN
    rating = Column(Float)                               # FLOAT
    tags = Column(ARRAY(String))                         # PostgreSQL ARRAY
    metadata = Column(JSON)                              # JSON
    role = Column(Enum(UserRole))                        # ENUM
    created_at = Column(DateTime, default=func.now())    # TIMESTAMP
    publish_date = Column(Date)                          # DATE
```

---

## 9.4 데이터베이스 초기화

### 테이블 생성

```python
from database import engine, Base
from models import User, Product

# 모든 테이블 생성
Base.metadata.create_all(bind=engine)

# 특정 테이블만 생성
User.__table__.create(bind=engine, checkfirst=True)
```

**Spring Boot와 비교:**
```java
// Spring Boot는 자동으로 테이블 생성
// application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: create  # 또는 update, validate
```

### 프로젝트 구조

```
app/
├── main.py
├── database.py          # DB 설정
├── models/
│   ├── __init__.py
│   ├── user.py         # User 모델
│   └── product.py      # Product 모델
├── schemas/            # Pydantic 스키마
│   ├── __init__.py
│   ├── user.py
│   └── product.py
├── crud/               # CRUD 작업
│   ├── __init__.py
│   ├── user.py
│   └── product.py
└── routers/            # API 라우터
    ├── __init__.py
    ├── users.py
    └── products.py
```

---

## 9.5 세션 관리

### 의존성으로 세션 제공

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.ext.declarative import declarative_base

SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# 세션 의존성
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
# main.py
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session
from database import get_db

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    return user
```

**Spring Boot와 비교:**
```java
// Spring Boot - EntityManager가 자동 주입됨
@Service
public class UserService {
    @PersistenceContext  // 또는 @Autowired EntityManager
    private EntityManager em;

    public User findById(Long id) {
        return em.find(User.class, id);
    }
}
```

---

## 9.6 기본 CRUD 작업

### Create (생성)

```python
from sqlalchemy.orm import Session
from models import User

def create_user(db: Session, username: str, email: str, password: str):
    # 새 객체 생성
    db_user = User(
        username=username,
        email=email,
        hashed_password=hash_password(password)
    )

    # 세션에 추가
    db.add(db_user)

    # 커밋 (DB에 저장)
    db.commit()

    # 객체 새로고침 (ID 등 DB에서 생성된 값 가져오기)
    db.refresh(db_user)

    return db_user

# 사용
@app.post("/users")
def create_user_endpoint(
    username: str,
    email: str,
    password: str,
    db: Session = Depends(get_db)
):
    user = create_user(db, username, email, password)
    return user
```

**Spring Boot JPA와 비교:**
```java
// Spring Boot
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    @Transactional
    public User createUser(String username, String email, String password) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setHashedPassword(hashPassword(password));

        return userRepository.save(user);  // 저장 및 반환
    }
}
```

### Read (조회)

```python
from sqlalchemy.orm import Session

# 단일 조회 - Primary Key
def get_user_by_id(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

# 단일 조회 - 다른 컬럼
def get_user_by_email(db: Session, email: str):
    return db.query(User).filter(User.email == email).first()

# 여러 개 조회
def get_users(db: Session, skip: int = 0, limit: int = 10):
    return db.query(User).offset(skip).limit(limit).all()

# 개수 조회
def count_users(db: Session):
    return db.query(User).count()

# 존재 여부 확인
def user_exists(db: Session, email: str):
    return db.query(User).filter(User.email == email).first() is not None
```

**Spring Boot JPA와 비교:**
```java
// Spring Boot
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
    List<User> findAll(Pageable pageable);
    long count();
    boolean existsByEmail(String email);
}
```

### Update (수정)

```python
def update_user(db: Session, user_id: int, username: str = None, email: str = None):
    # 사용자 조회
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        return None

    # 값 수정
    if username:
        user.username = username
    if email:
        user.email = email

    # 커밋
    db.commit()
    db.refresh(user)

    return user

# 또는 update() 사용
def update_user_bulk(db: Session, user_id: int, update_data: dict):
    db.query(User).filter(User.id == user_id).update(update_data)
    db.commit()

    return db.query(User).filter(User.id == user_id).first()
```

**Spring Boot와 비교:**
```java
// Spring Boot
@Transactional
public User updateUser(Long userId, String username, String email) {
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new NotFoundException());

    if (username != null) user.setUsername(username);
    if (email != null) user.setEmail(email);

    return userRepository.save(user);  // 자동 업데이트
}
```

### Delete (삭제)

```python
def delete_user(db: Session, user_id: int):
    user = db.query(User).filter(User.id == user_id).first()

    if not user:
        return False

    db.delete(user)
    db.commit()

    return True

# 또는 쿼리로 직접 삭제
def delete_user_query(db: Session, user_id: int):
    deleted_count = db.query(User).filter(User.id == user_id).delete()
    db.commit()

    return deleted_count > 0
```

**Spring Boot와 비교:**
```java
// Spring Boot
@Transactional
public void deleteUser(Long userId) {
    userRepository.deleteById(userId);
}

// 또는
public void deleteUser(Long userId) {
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new NotFoundException());
    userRepository.delete(user);
}
```

---

## 9.7 쿼리 작성

### 필터링

```python
from sqlalchemy import and_, or_, not_

# WHERE 조건
db.query(User).filter(User.username == "john").all()
db.query(User).filter(User.id > 10).all()
db.query(User).filter(User.email.like("%@gmail.com")).all()
db.query(User).filter(User.username.in_(["john", "jane"])).all()

# AND 조건
db.query(User).filter(
    User.is_active == True,
    User.created_at > datetime(2024, 1, 1)
).all()

# 또는 and_() 사용
db.query(User).filter(
    and_(
        User.is_active == True,
        User.created_at > datetime(2024, 1, 1)
    )
).all()

# OR 조건
db.query(User).filter(
    or_(
        User.username == "john",
        User.email == "john@example.com"
    )
).all()

# NOT 조건
db.query(User).filter(not_(User.is_active)).all()
```

**Spring Boot JPQL과 비교:**
```java
// Spring Boot
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);

@Query("SELECT u FROM User u WHERE u.id > :id")
List<User> findByIdGreaterThan(@Param("id") Long id);

@Query("SELECT u FROM User u WHERE u.email LIKE %:domain")
List<User> findByEmailDomain(@Param("domain") String domain);

@Query("SELECT u FROM User u WHERE u.isActive = true AND u.createdAt > :date")
List<User> findActiveUsersAfter(@Param("date") LocalDateTime date);
```

### 정렬

```python
# 오름차순
db.query(User).order_by(User.username).all()

# 내림차순
db.query(User).order_by(User.created_at.desc()).all()

# 여러 컬럼 정렬
db.query(User).order_by(User.is_active.desc(), User.username).all()
```

### 페이지네이션

```python
def get_users_paginated(db: Session, page: int = 1, page_size: int = 10):
    skip = (page - 1) * page_size
    users = db.query(User).offset(skip).limit(page_size).all()
    total = db.query(User).count()

    return {
        "items": users,
        "total": total,
        "page": page,
        "page_size": page_size,
        "total_pages": (total + page_size - 1) // page_size
    }
```

**Spring Boot Pageable과 비교:**
```java
// Spring Boot
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findAll(Pageable pageable);
}

// 사용
Pageable pageable = PageRequest.of(0, 10, Sort.by("username"));
Page<User> users = userRepository.findAll(pageable);
```

---

## 9.8 집계 함수

```python
from sqlalchemy import func

# COUNT
user_count = db.query(func.count(User.id)).scalar()

# MAX, MIN
max_id = db.query(func.max(User.id)).scalar()
min_created = db.query(func.min(User.created_at)).scalar()

# AVG, SUM (Product 모델 가정)
avg_price = db.query(func.avg(Product.price)).scalar()
total_stock = db.query(func.sum(Product.stock)).scalar()

# GROUP BY
from sqlalchemy import func

user_counts_by_date = db.query(
    func.date(User.created_at).label("date"),
    func.count(User.id).label("count")
).group_by(func.date(User.created_at)).all()
```

---

## 9.9 완전한 CRUD 예제

```python
# models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(100), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())

# schemas/user.py
from pydantic import BaseModel, EmailStr
from datetime import datetime

class UserBase(BaseModel):
    username: str
    email: EmailStr

class UserCreate(UserBase):
    password: str

class UserUpdate(BaseModel):
    username: str | None = None
    email: EmailStr | None = None
    is_active: bool | None = None

class UserResponse(UserBase):
    id: int
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True

# crud/user.py
from sqlalchemy.orm import Session
from models.user import User
from schemas.user import UserCreate, UserUpdate

def get_user(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(User).filter(User.email == email).first()

def get_users(db: Session, skip: int = 0, limit: int = 10):
    return db.query(User).offset(skip).limit(limit).all()

def create_user(db: Session, user: UserCreate):
    hashed_password = hash_password(user.password)  # 비밀번호 해싱
    db_user = User(
        username=user.username,
        email=user.email,
        hashed_password=hashed_password
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def update_user(db: Session, user_id: int, user: UserUpdate):
    db_user = get_user(db, user_id)
    if not db_user:
        return None

    update_data = user.dict(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_user, key, value)

    db.commit()
    db.refresh(db_user)
    return db_user

def delete_user(db: Session, user_id: int):
    db_user = get_user(db, user_id)
    if not db_user:
        return False

    db.delete(db_user)
    db.commit()
    return True

# routers/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from database import get_db
from schemas.user import UserCreate, UserUpdate, UserResponse
from crud import user as crud_user

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/", response_model=List[UserResponse])
def list_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    return crud_user.get_users(db, skip=skip, limit=limit)

@router.get("/{user_id}", response_model=UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = crud_user.get_user(db, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.post("/", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    # 이메일 중복 체크
    db_user = crud_user.get_user_by_email(db, user.email)
    if db_user:
        raise HTTPException(status_code=400, detail="Email already registered")

    return crud_user.create_user(db, user)

@router.put("/{user_id}", response_model=UserResponse)
def update_user(user_id: int, user: UserUpdate, db: Session = Depends(get_db)):
    db_user = crud_user.update_user(db, user_id, user)
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

@router.delete("/{user_id}", status_code=204)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    if not crud_user.delete_user(db, user_id):
        raise HTTPException(status_code=404, detail="User not found")
```

---

## 9.10 핵심 요약

### SQLAlchemy vs JPA 비교

| 작업 | Spring Boot JPA | SQLAlchemy |
|------|----------------|------------|
| **모델 정의** | `@Entity` | `Base` 상속 |
| **기본키** | `@Id` | `primary_key=True` |
| **생성** | `repository.save()` | `db.add()` + `db.commit()` |
| **조회** | `repository.findById()` | `db.query().filter().first()` |
| **수정** | `repository.save()` | 속성 변경 + `db.commit()` |
| **삭제** | `repository.delete()` | `db.delete()` + `db.commit()` |
| **쿼리** | JPQL, QueryDSL | Query API |

### 기억할 핵심

1. ✅ **Base 상속**: 모든 모델은 `Base`를 상속
2. ✅ **Session**: 트랜잭션 단위, 사용 후 반드시 close
3. ✅ **commit**: 변경사항을 DB에 반영
4. ✅ **refresh**: DB에서 객체 새로고침
5. ✅ **Query API**: 메서드 체이닝으로 쿼리 작성

---

## 9.11 다음 챕터 예고

다음 챕터에서는 **데이터베이스 CRUD 구현**을 다룹니다:
- Repository 패턴
- 복잡한 쿼리 작성
- 트랜잭션 관리
- 에러 처리

---

[← 이전: Chapter 8. 고급 의존성 패턴](../part03-dependency-injection/chapter08-advanced-di.md) | [목차로](../README.md) | [다음: Chapter 10. 데이터베이스 CRUD 구현 →](chapter10-crud.md)
