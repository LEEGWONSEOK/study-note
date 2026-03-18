# Chapter 8. 고급 의존성 패턴

## 8.1 yield를 사용한 컨텍스트 관리

`yield`를 사용하면 의존성의 setup과 teardown을 관리할 수 있습니다. Spring Boot의 `@PostConstruct`와 `@PreDestroy`와 유사합니다.

### 기본 yield 패턴

```python
from fastapi import Depends

def get_db():
    # Setup: 리소스 생성
    db = "database_connection"
    print("Database connection opened")

    try:
        yield db  # 의존성을 사용하는 코드에 전달
    finally:
        # Teardown: 리소스 정리
        print("Database connection closed")

@app.get("/items")
def read_items(db: str = Depends(get_db)):
    # db를 사용하여 작업
    return {"db": db, "items": [...]}

# 실행 순서:
# 1. "Database connection opened" 출력
# 2. read_items 함수 실행
# 3. "Database connection closed" 출력
```

### Spring Boot와 비교

```java
// Spring Boot
@Component
public class DatabaseConnection {
    private Connection connection;

    @PostConstruct  // 생성 후 실행
    public void init() {
        connection = createConnection();
        System.out.println("Database connection opened");
    }

    @PreDestroy  // 소멸 전 실행
    public void cleanup() {
        connection.close();
        System.out.println("Database connection closed");
    }

    public Connection getConnection() {
        return connection;
    }
}
```

```python
# FastAPI - 더 간결
def get_db():
    db = create_connection()
    print("Database connection opened")
    try:
        yield db
    finally:
        db.close()
        print("Database connection closed")
```

---

## 8.2 실전 예제: 데이터베이스 세션

### SQLAlchemy 세션 관리

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from fastapi import Depends

# 데이터베이스 엔진 생성
SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 세션 의존성
def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 사용 예제
@app.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users")
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

**Spring Boot JPA와 비교:**
```java
// Spring Boot - EntityManager가 자동으로 관리됨
@Service
public class UserService {
    @Autowired
    private EntityManager entityManager;  // Spring이 자동으로 생성/소멸 관리

    @Transactional
    public User getUser(Long id) {
        return entityManager.find(User.class, id);
    }
}
```

---

## 8.3 예외 처리와 yield

yield를 사용할 때 예외를 처리하는 방법입니다.

### 기본 예외 처리

```python
from fastapi import Depends, HTTPException

def get_db():
    db = create_connection()
    try:
        yield db
    except Exception as e:
        # 예외 발생 시 롤백
        db.rollback()
        print(f"Error occurred: {e}")
        raise  # 예외를 다시 던짐
    finally:
        # 항상 실행됨
        db.close()

@app.post("/users")
def create_user(user: UserCreate, db = Depends(get_db)):
    # 에러 발생 시 자동으로 롤백됨
    db.execute("INSERT INTO users ...")
    db.commit()
    return {"status": "success"}
```

### 트랜잭션 관리

```python
from contextlib import contextmanager
from sqlalchemy.orm import Session

def get_db():
    db = SessionLocal()
    try:
        yield db
        # 정상 종료 시 커밋 (선택사항)
        # db.commit()
    except Exception:
        # 에러 발생 시 롤백
        db.rollback()
        raise
    finally:
        db.close()

# 명시적 트랜잭션
@app.post("/transfer")
def transfer_money(
    from_account: int,
    to_account: int,
    amount: float,
    db: Session = Depends(get_db)
):
    try:
        # 출금
        account1 = db.query(Account).filter(Account.id == from_account).first()
        account1.balance -= amount

        # 입금
        account2 = db.query(Account).filter(Account.id == to_account).first()
        account2.balance += amount

        # 커밋
        db.commit()
        return {"status": "success"}
    except Exception as e:
        db.rollback()  # 롤백
        raise HTTPException(status_code=400, detail=str(e))
```

---

## 8.4 의존성 오버라이드 (테스트용)

테스트 시 실제 의존성을 Mock으로 교체할 수 있습니다.

### 기본 오버라이드

```python
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient

app = FastAPI()

# 실제 데이터베이스
def get_real_db():
    return "real_database"

@app.get("/items")
def read_items(db: str = Depends(get_real_db)):
    return {"db": db}

# 테스트
def get_fake_db():
    return "fake_database"

client = TestClient(app)

# 의존성 오버라이드
app.dependency_overrides[get_real_db] = get_fake_db

response = client.get("/items")
assert response.json() == {"db": "fake_database"}

# 오버라이드 제거
app.dependency_overrides.clear()
```

### Spring Boot의 @MockBean과 비교

```java
// Spring Boot 테스트
@SpringBootTest
public class UserServiceTest {
    @MockBean  // 실제 빈을 Mock으로 교체
    private UserRepository userRepository;

    @Autowired
    private UserService userService;

    @Test
    public void testGetUser() {
        when(userRepository.findById(1L))
            .thenReturn(Optional.of(new User()));

        User user = userService.getUser(1L);
        assertNotNull(user);
    }
}
```

```python
# FastAPI 테스트
def test_get_user():
    def fake_get_db():
        return FakeDatabase()

    app.dependency_overrides[get_db] = fake_get_db

    response = client.get("/users/1")
    assert response.status_code == 200

    app.dependency_overrides.clear()
```

### 복잡한 오버라이드 예제

```python
# 실제 서비스
class RealEmailService:
    def send_email(self, to: str, subject: str, body: str):
        # 실제로 이메일 전송
        print(f"Sending email to {to}")

def get_email_service():
    return RealEmailService()

# 테스트용 Mock 서비스
class FakeEmailService:
    def __init__(self):
        self.emails_sent = []

    def send_email(self, to: str, subject: str, body: str):
        self.emails_sent.append({"to": to, "subject": subject, "body": body})

# 엔드포인트
@app.post("/send-welcome-email")
def send_welcome(email: str, service = Depends(get_email_service)):
    service.send_email(email, "Welcome!", "Welcome to our service")
    return {"status": "sent"}

# 테스트
def test_send_welcome_email():
    fake_service = FakeEmailService()

    def get_fake_email_service():
        return fake_service

    app.dependency_overrides[get_email_service] = get_fake_email_service

    response = client.post("/send-welcome-email?email=test@example.com")

    assert response.status_code == 200
    assert len(fake_service.emails_sent) == 1
    assert fake_service.emails_sent[0]["to"] == "test@example.com"

    app.dependency_overrides.clear()
```

---

## 8.5 의존성 캐싱

FastAPI는 같은 요청 내에서 같은 의존성을 여러 번 호출해도 한 번만 실행합니다.

### 기본 캐싱

```python
from fastapi import Depends

def expensive_operation():
    print("This is expensive!")
    return "result"

def dependency_a(result: str = Depends(expensive_operation)):
    return f"A: {result}"

def dependency_b(result: str = Depends(expensive_operation)):
    return f"B: {result}"

@app.get("/test")
def test_caching(
    a: str = Depends(dependency_a),
    b: str = Depends(dependency_b)
):
    return {"a": a, "b": b}

# "This is expensive!" 는 한 번만 출력됨
# expensive_operation은 한 번만 실행됨
```

### 캐싱 비활성화

```python
from fastapi import Depends

@app.get("/test")
def test_no_cache(
    a: str = Depends(expensive_operation, use_cache=False),
    b: str = Depends(expensive_operation, use_cache=False)
):
    return {"a": a, "b": b}

# "This is expensive!" 가 두 번 출력됨
```

---

## 8.6 의존성으로 공통 로직 추출

반복되는 로직을 의존성으로 추출하여 재사용합니다.

### 페이지네이션 의존성

```python
from fastapi import Query, Depends

class PaginationParams:
    def __init__(
        self,
        skip: int = Query(0, ge=0, description="건너뛸 개수"),
        limit: int = Query(10, ge=1, le=100, description="가져올 개수"),
        page: int = Query(1, ge=1, description="페이지 번호")
    ):
        self.skip = skip
        self.limit = limit
        self.page = page
        self.offset = (page - 1) * limit

@app.get("/users")
def list_users(pagination: PaginationParams = Depends()):
    return {
        "skip": pagination.skip,
        "limit": pagination.limit,
        "page": pagination.page,
        "offset": pagination.offset
    }

@app.get("/posts")
def list_posts(pagination: PaginationParams = Depends()):
    return {
        "page": pagination.page,
        "offset": pagination.offset
    }
```

### 정렬 의존성

```python
from enum import Enum

class SortOrder(str, Enum):
    ASC = "asc"
    DESC = "desc"

class SortParams:
    def __init__(
        self,
        sort_by: str = Query("created_at", description="정렬 기준 필드"),
        order: SortOrder = Query(SortOrder.DESC, description="정렬 순서")
    ):
        self.sort_by = sort_by
        self.order = order

@app.get("/products")
def list_products(
    pagination: PaginationParams = Depends(),
    sort: SortParams = Depends()
):
    return {
        "pagination": {
            "page": pagination.page,
            "limit": pagination.limit
        },
        "sort": {
            "sort_by": sort.sort_by,
            "order": sort.order
        }
    }
```

---

## 8.7 의존성으로 권한 체크

인증과 권한을 체크하는 의존성을 만들 수 있습니다.

### 기본 인증

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    token = credentials.credentials
    # 토큰 검증 로직
    if token != "valid-token":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials"
        )

    return {"username": "john", "role": "user"}

@app.get("/me")
def read_current_user(current_user: dict = Depends(get_current_user)):
    return current_user
```

### 역할 기반 권한

```python
from typing import List

def require_roles(allowed_roles: List[str]):
    def role_checker(current_user: dict = Depends(get_current_user)):
        user_role = current_user.get("role")
        if user_role not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{user_role}' is not allowed"
            )
        return current_user
    return role_checker

# 관리자만 접근 가능
@app.get("/admin/users")
def list_all_users(
    current_user: dict = Depends(require_roles(["admin"]))
):
    return {"users": [...]}

# 관리자 또는 매니저 접근 가능
@app.get("/reports")
def get_reports(
    current_user: dict = Depends(require_roles(["admin", "manager"]))
):
    return {"reports": [...]}
```

**Spring Boot Security와 비교:**
```java
// Spring Boot
@RestController
public class AdminController {

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/admin/users")
    public List<User> listAllUsers() {
        return userService.findAll();
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    @GetMapping("/reports")
    public List<Report> getReports() {
        return reportService.findAll();
    }
}
```

---

## 8.8 클래스 기반 의존성 고급 패턴

### Singleton 패턴

```python
from fastapi import Depends

class SingletonService:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.initialized = False
        return cls._instance

    def __init__(self):
        if not self.initialized:
            print("Initializing singleton service")
            self.data = "singleton_data"
            self.initialized = True

def get_singleton_service():
    return SingletonService()

@app.get("/test1")
def test1(service: SingletonService = Depends(get_singleton_service)):
    return {"data": service.data}

@app.get("/test2")
def test2(service: SingletonService = Depends(get_singleton_service)):
    return {"data": service.data}

# "Initializing singleton service"는 첫 요청 때만 출력됨
```

**Spring Boot의 @Scope와 비교:**
```java
// Spring Boot - 기본이 Singleton
@Service
@Scope("singleton")  // 기본값
public class SingletonService {
    public SingletonService() {
        System.out.println("Initializing singleton service");
    }
}
```

### Factory 패턴

```python
from enum import Enum

class DatabaseType(str, Enum):
    POSTGRES = "postgres"
    MYSQL = "mysql"
    SQLITE = "sqlite"

class DatabaseFactory:
    @staticmethod
    def create(db_type: DatabaseType):
        if db_type == DatabaseType.POSTGRES:
            return PostgresDatabase()
        elif db_type == DatabaseType.MYSQL:
            return MySQLDatabase()
        elif db_type == DatabaseType.SQLITE:
            return SQLiteDatabase()

def get_database(db_type: DatabaseType = Query(DatabaseType.POSTGRES)):
    return DatabaseFactory.create(db_type)

@app.get("/query")
def execute_query(
    query: str,
    db = Depends(get_database)
):
    result = db.execute(query)
    return {"result": result}

# /query?db_type=postgres&query=SELECT * FROM users
```

---

## 8.9 의존성 체인 최적화

복잡한 의존성 체인을 효율적으로 관리하는 방법입니다.

### 의존성 조합

```python
from fastapi import Depends

# 개별 의존성들
def get_db():
    return "database"

def get_cache():
    return "redis"

def get_logger():
    return "logger"

# 조합 의존성
class ServiceDependencies:
    def __init__(
        self,
        db: str = Depends(get_db),
        cache: str = Depends(get_cache),
        logger: str = Depends(get_logger)
    ):
        self.db = db
        self.cache = cache
        self.logger = logger

# 사용
@app.get("/users")
def list_users(deps: ServiceDependencies = Depends()):
    return {
        "db": deps.db,
        "cache": deps.cache,
        "logger": deps.logger
    }
```

### 의존성 컨테이너 패턴

```python
from dataclasses import dataclass

@dataclass
class AppContainer:
    db: str
    cache: str
    config: dict

def get_app_container() -> AppContainer:
    return AppContainer(
        db="database_connection",
        cache="redis_connection",
        config={"debug": True}
    )

class UserService:
    def __init__(self, container: AppContainer = Depends(get_app_container)):
        self.db = container.db
        self.cache = container.cache
        self.config = container.config

    def get_user(self, user_id: int):
        # self.db, self.cache 사용
        return {"id": user_id, "username": "john"}

@app.get("/users/{user_id}")
def get_user(user_id: int, service: UserService = Depends()):
    return service.get_user(user_id)
```

---

## 8.10 실전 예제: 완전한 레이어드 아키텍처

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

app = FastAPI()

# ==================== Database Layer ====================
def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ==================== Repository Layer ====================
class UserRepository:
    def __init__(self, db: Session = Depends(get_db)):
        self.db = db

    def find_by_id(self, user_id: int):
        return self.db.query(User).filter(User.id == user_id).first()

    def find_all(self, skip: int = 0, limit: int = 10):
        return self.db.query(User).offset(skip).limit(limit).all()

    def create(self, user_data: dict):
        user = User(**user_data)
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user

    def update(self, user_id: int, user_data: dict):
        user = self.find_by_id(user_id)
        if not user:
            return None
        for key, value in user_data.items():
            setattr(user, key, value)
        self.db.commit()
        self.db.refresh(user)
        return user

    def delete(self, user_id: int):
        user = self.find_by_id(user_id)
        if user:
            self.db.delete(user)
            self.db.commit()
            return True
        return False

# ==================== Service Layer ====================
class UserService:
    def __init__(self, repo: UserRepository = Depends()):
        self.repo = repo

    def get_user(self, user_id: int):
        user = self.repo.find_by_id(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")
        return user

    def list_users(self, skip: int = 0, limit: int = 10):
        return self.repo.find_all(skip, limit)

    def create_user(self, user_create: UserCreate):
        # 비즈니스 로직: 중복 확인 등
        user_data = user_create.dict()
        return self.repo.create(user_data)

    def update_user(self, user_id: int, user_update: UserUpdate):
        user_data = user_update.dict(exclude_unset=True)
        user = self.repo.update(user_id, user_data)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")
        return user

    def delete_user(self, user_id: int):
        if not self.repo.delete(user_id):
            raise HTTPException(status_code=404, detail="User not found")

# ==================== Controller Layer ====================
@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int,
    service: UserService = Depends()
):
    return service.get_user(user_id)

@app.get("/users", response_model=List[UserResponse])
def list_users(
    skip: int = 0,
    limit: int = 10,
    service: UserService = Depends()
):
    return service.list_users(skip, limit)

@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(
    user: UserCreate,
    service: UserService = Depends()
):
    return service.create_user(user)

@app.put("/users/{user_id}", response_model=UserResponse)
def update_user(
    user_id: int,
    user: UserUpdate,
    service: UserService = Depends()
):
    return service.update_user(user_id, user)

@app.delete("/users/{user_id}", status_code=204)
def delete_user(
    user_id: int,
    service: UserService = Depends()
):
    service.delete_user(user_id)
```

**Spring Boot와 완전히 동일한 구조!**

---

## 8.11 핵심 요약

### 고급 의존성 패턴

| 패턴 | 설명 | 사용 사례 |
|------|------|----------|
| **yield** | Setup/Teardown | DB 연결, 파일 핸들 |
| **오버라이드** | 테스트용 Mock | 단위 테스트 |
| **캐싱** | 중복 실행 방지 | 성능 최적화 |
| **체인** | 의존성 조합 | 복잡한 로직 |
| **권한 체크** | 인증/인가 | 보안 |

### Spring Boot와 비교

| 기능 | Spring Boot | FastAPI |
|------|-------------|---------|
| **리소스 관리** | `@PostConstruct`, `@PreDestroy` | `yield` |
| **테스트 Mock** | `@MockBean` | `dependency_overrides` |
| **권한 체크** | `@PreAuthorize` | `Depends(require_roles)` |
| **Singleton** | `@Scope("singleton")` | 직접 구현 |
| **레이어드 아키텍처** | Controller-Service-Repository | 동일하게 구현 가능 |

---

## 8.12 실습 예제

### 실습 1: 트랜잭션 관리

**요구사항:**
- 데이터베이스 세션 관리
- 에러 발생 시 자동 롤백
- 정상 종료 시 자동 커밋

**정답:**
```python
from sqlalchemy.orm import Session

def get_db_with_transaction():
    db = SessionLocal()
    try:
        yield db
        db.commit()  # 정상 종료 시 커밋
    except Exception:
        db.rollback()  # 에러 시 롤백
        raise
    finally:
        db.close()

@app.post("/transfer")
def transfer_money(
    from_id: int,
    to_id: int,
    amount: float,
    db: Session = Depends(get_db_with_transaction)
):
    # 출금
    from_account = db.query(Account).get(from_id)
    from_account.balance -= amount

    # 입금
    to_account = db.query(Account).get(to_id)
    to_account.balance += amount

    return {"status": "success"}
    # 자동으로 커밋됨 (에러 발생 시 자동 롤백)
```

---

## 8.13 연습 문제

### 문제 1
`yield`를 사용하는 의존성과 일반 함수 의존성의 차이점을 설명하세요.

### 문제 2
테스트에서 실제 데이터베이스를 SQLite 메모리 DB로 교체하는 코드를 작성하세요.

### 문제 3
다음 요구사항을 만족하는 의존성을 작성하세요:
- 사용자 인증 확인
- 사용자가 활성화 상태인지 확인
- 사용자가 프리미엄 회원인지 확인

---

## 8.14 다음 챕터 예고

다음 챕터에서는 **데이터베이스 연동**을 다룹니다:
- SQLAlchemy ORM 기초
- 모델 정의 (Spring JPA Entity와 비교)
- CRUD 작업
- Relationship (OneToMany, ManyToMany)

**Spring Boot JPA와 비교하며 SQLAlchemy를 완전히 마스터합니다!**

---

[← 이전: Chapter 7. 의존성 주입 기초](chapter07-di-basics.md) | [목차로](../README.md) | [다음: Part 04 →](../part04-database/chapter09-sqlalchemy-basics.md)