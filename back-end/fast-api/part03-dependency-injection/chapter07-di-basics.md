# Chapter 7. 의존성 주입 기초

## 7.1 의존성 주입이란?

의존성 주입(Dependency Injection)은 객체가 필요로 하는 의존성을 외부에서 주입받는 디자인 패턴입니다. FastAPI의 `Depends`는 Spring Boot의 `@Autowired`와 유사한 역할을 합니다.

### Spring Boot vs FastAPI DI

```java
// Spring Boot
@Service
public class UserService {
    // 비즈니스 로직
}

@RestController
public class UserController {
    @Autowired  // Spring이 자동으로 주입
    private UserService userService;

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

```python
# FastAPI
from fastapi import FastAPI, Depends

app = FastAPI()

# 의존성 함수
def get_user_service():
    return UserService()

@app.get("/users/{id}")
def get_user(
    id: int,
    user_service: UserService = Depends(get_user_service)  # FastAPI가 자동으로 주입
):
    return user_service.find_by_id(id)
```

---

## 7.2 Depends 기본 사용법

### 함수형 의존성

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# 의존성 함수 정의
def common_parameters(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items")
def read_items(commons: dict = Depends(common_parameters)):
    return commons

@app.get("/users")
def read_users(commons: dict = Depends(common_parameters)):
    return commons

# 요청: /items?skip=0&limit=20
# 결과: {"skip": 0, "limit": 20}
```

**Spring Boot와 비교:**
```java
// Spring Boot - @ModelAttribute 사용
public class CommonParameters {
    private int skip = 0;
    private int limit = 10;
    // Getters, Setters
}

@GetMapping("/items")
public ResponseEntity<?> readItems(@ModelAttribute CommonParameters params) {
    return ResponseEntity.ok(params);
}
```

### 의존성 재사용

```python
from typing import Optional

def pagination(skip: int = 0, limit: int = 100):
    return {"skip": skip, "limit": limit}

def search_params(q: Optional[str] = None):
    return {"q": q}

@app.get("/items")
def list_items(
    page: dict = Depends(pagination),
    search: dict = Depends(search_params)
):
    return {**page, **search}

# /items?skip=10&limit=20&q=laptop
# {"skip": 10, "limit": 20, "q": "laptop"}
```

---

## 7.3 클래스형 의존성

클래스를 의존성으로 사용할 수 있습니다.

### 기본 클래스 의존성

```python
from fastapi import Depends

class CommonQueryParams:
    def __init__(self, skip: int = 0, limit: int = 10, q: str | None = None):
        self.skip = skip
        self.limit = limit
        self.q = q

@app.get("/items")
def read_items(commons: CommonQueryParams = Depends(CommonQueryParams)):
    response = {"skip": commons.skip, "limit": commons.limit}
    if commons.q:
        response["q"] = commons.q
    return response

# 축약형
@app.get("/items")
def read_items(commons: CommonQueryParams = Depends()):
    # Depends()만 쓰면 타입 힌트에서 자동으로 추론
    return {"skip": commons.skip, "limit": commons.limit}
```

### Spring Boot Service 패턴

```python
# 서비스 클래스 정의
class DatabaseService:
    def __init__(self):
        self.db = "PostgreSQL"

    def get_connection(self):
        return f"Connected to {self.db}"

class UserService:
    def __init__(self, db: DatabaseService = Depends()):
        self.db = db

    def get_user(self, user_id: int):
        connection = self.db.get_connection()
        return {"user_id": user_id, "connection": connection}

# 사용
@app.get("/users/{user_id}")
def read_user(
    user_id: int,
    user_service: UserService = Depends()
):
    return user_service.get_user(user_id)
```

**Spring Boot와 비교:**
```java
// Spring Boot
@Service
public class DatabaseService {
    public String getConnection() {
        return "Connected to PostgreSQL";
    }
}

@Service
public class UserService {
    @Autowired
    private DatabaseService databaseService;

    public User getUser(Long userId) {
        String connection = databaseService.getConnection();
        // ...
    }
}

@RestController
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/users/{userId}")
    public User getUser(@PathVariable Long userId) {
        return userService.getUser(userId);
    }
}
```

---

## 7.4 의존성 체인

의존성이 다른 의존성을 가질 수 있습니다.

### 다단계 의존성

```python
from fastapi import Depends, HTTPException

# 1단계: 데이터베이스 연결
def get_db():
    db = "database_connection"
    try:
        yield db
    finally:
        print("Database connection closed")

# 2단계: 현재 사용자 조회
def get_current_user(db: str = Depends(get_db)):
    # DB에서 사용자 조회
    user = {"username": "john", "email": "john@example.com"}
    if not user:
        raise HTTPException(status_code=401, detail="Not authenticated")
    return user

# 3단계: 관리자 권한 확인
def get_current_admin_user(current_user: dict = Depends(get_current_user)):
    if current_user.get("username") != "admin":
        raise HTTPException(status_code=403, detail="Not enough permissions")
    return current_user

# 사용
@app.get("/admin/users")
def list_users(admin: dict = Depends(get_current_admin_user)):
    return {"admin": admin, "users": [...]}
```

**Spring Boot와 비교:**
```java
// Spring Boot
@Service
public class AuthService {
    @Autowired
    private DatabaseService databaseService;

    public User getCurrentUser() {
        // DB에서 사용자 조회
    }

    public User getCurrentAdminUser() {
        User user = getCurrentUser();
        if (!user.isAdmin()) {
            throw new ForbiddenException();
        }
        return user;
    }
}
```

---

## 7.5 서브 의존성 (Sub-dependencies)

의존성 함수가 다른 의존성을 사용할 수 있습니다.

```python
from fastapi import Depends, Header, HTTPException

def verify_token(x_token: str = Header(...)):
    if x_token != "secret-token":
        raise HTTPException(status_code=400, detail="Invalid X-Token header")
    return x_token

def verify_key(x_key: str = Header(...)):
    if x_key != "secret-key":
        raise HTTPException(status_code=400, detail="Invalid X-Key header")
    return x_key

# 두 의존성을 모두 사용하는 서브 의존성
def verify_auth(
    token: str = Depends(verify_token),
    key: str = Depends(verify_key)
):
    return {"token": token, "key": key}

@app.get("/items")
def read_items(auth: dict = Depends(verify_auth)):
    return {"auth": auth, "items": [...]}
```

---

## 7.6 전역 의존성

앱 전체에 적용되는 의존성을 설정할 수 있습니다.

```python
from fastapi import FastAPI, Depends, HTTPException

async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != "my-api-key":
        raise HTTPException(status_code=403, detail="Invalid API Key")

# 전역 의존성 설정
app = FastAPI(dependencies=[Depends(verify_api_key)])

@app.get("/items")
def read_items():
    # verify_api_key가 자동으로 실행됨
    return {"items": [...]}

@app.get("/users")
def read_users():
    # verify_api_key가 자동으로 실행됨
    return {"users": [...]}
```

### 라우터별 의존성

```python
from fastapi import APIRouter, Depends

async def verify_admin(x_token: str = Header(...)):
    if x_token != "admin-token":
        raise HTTPException(status_code=403)

# 관리자 라우터에만 의존성 적용
admin_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_admin)]
)

@admin_router.get("/users")
def admin_list_users():
    return {"users": [...]}

@admin_router.get("/settings")
def admin_settings():
    return {"settings": {...}}

app.include_router(admin_router)
```

---

## 7.7 Callable 클래스 의존성

`__call__` 메서드를 사용하여 클래스를 callable하게 만들 수 있습니다.

```python
from fastapi import Depends

class Paginator:
    def __init__(self, max_limit: int = 100):
        self.max_limit = max_limit

    def __call__(self, skip: int = 0, limit: int = 10):
        if limit > self.max_limit:
            limit = self.max_limit
        return {"skip": skip, "limit": limit}

# 인스턴스 생성
pagination = Paginator(max_limit=50)

@app.get("/items")
def list_items(page: dict = Depends(pagination)):
    return page

# /items?skip=0&limit=100
# 결과: {"skip": 0, "limit": 50}  # max_limit에 의해 제한됨
```

---

## 7.8 실전 예제: 데이터베이스 세션 관리

```python
from fastapi import FastAPI, Depends
from typing import Generator

app = FastAPI()

# 가상의 데이터베이스 클래스
class Database:
    def __init__(self):
        self.connection = "DB Connection"

    def close(self):
        print("Database connection closed")

# 데이터베이스 세션 의존성
def get_db() -> Generator:
    db = Database()
    try:
        yield db
    finally:
        db.close()

# Repository 패턴
class UserRepository:
    def __init__(self, db: Database = Depends(get_db)):
        self.db = db

    def find_by_id(self, user_id: int):
        # self.db.connection을 사용하여 쿼리 실행
        return {"id": user_id, "username": "john"}

    def find_all(self, skip: int = 0, limit: int = 10):
        # 쿼리 실행
        return [{"id": 1, "username": "john"}, {"id": 2, "username": "jane"}]

# Service 패턴
class UserService:
    def __init__(self, repo: UserRepository = Depends()):
        self.repo = repo

    def get_user(self, user_id: int):
        return self.repo.find_by_id(user_id)

    def list_users(self, skip: int = 0, limit: int = 10):
        return self.repo.find_all(skip, limit)

# Controller (엔드포인트)
@app.get("/users/{user_id}")
def get_user(
    user_id: int,
    service: UserService = Depends()
):
    return service.get_user(user_id)

@app.get("/users")
def list_users(
    skip: int = 0,
    limit: int = 10,
    service: UserService = Depends()
):
    return service.list_users(skip, limit)
```

**Spring Boot와 비교:**
```java
// Spring Boot - 거의 동일한 구조!
@Repository
public class UserRepository {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    public User findById(Long id) { ... }
    public List<User> findAll(int skip, int limit) { ... }
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        return userRepository.findById(id);
    }

    public List<User> listUsers(int skip, int limit) {
        return userRepository.findAll(skip, limit);
    }
}

@RestController
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

---

## 7.9 핵심 요약

### Spring Boot vs FastAPI DI 비교

| 기능 | Spring Boot | FastAPI |
|------|-------------|---------|
| **의존성 주입** | `@Autowired` | `Depends()` |
| **스코프** | Singleton, Prototype, Request | 함수 호출마다 |
| **설정** | `@Configuration`, `@Bean` | 함수 정의 |
| **라이프사이클** | `@PostConstruct`, `@PreDestroy` | `try-finally` (yield) |
| **조건부 빈** | `@Conditional` | 함수 로직 |
| **전역 설정** | 자동 컴포넌트 스캔 | `dependencies=[]` |

### 기억할 핵심

1. ✅ **Depends()**: Spring의 `@Autowired`와 동일한 역할
2. ✅ **함수형**: 함수만으로도 의존성 정의 가능
3. ✅ **클래스형**: 클래스도 의존성으로 사용 가능
4. ✅ **체인**: 의존성이 다른 의존성을 가질 수 있음
5. ✅ **재사용**: 같은 의존성을 여러 곳에서 사용 가능
6. ✅ **전역**: 앱 전체 또는 라우터별로 의존성 설정

---

## 7.10 실습 예제

### 실습 1: 인증 의존성

**요구사항:**
- API 키 검증 의존성 작성
- 관리자 권한 검증 의존성 작성
- 두 의존성을 체인으로 연결

**정답:**
```python
from fastapi import Depends, Header, HTTPException

def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != "secret-api-key":
        raise HTTPException(status_code=403, detail="Invalid API Key")
    return x_api_key

def verify_admin(
    api_key: str = Depends(verify_api_key),
    x_admin_token: str = Header(...)
):
    if x_admin_token != "admin-token":
        raise HTTPException(status_code=403, detail="Not an admin")
    return {"api_key": api_key, "is_admin": True}

@app.get("/admin/dashboard")
def admin_dashboard(admin: dict = Depends(verify_admin)):
    return {"message": "Welcome Admin", "admin": admin}
```

---

## 7.11 연습 문제

### 문제 1
FastAPI의 `Depends()`와 Spring Boot의 `@Autowired`의 차이점 5가지를 설명하세요.

### 문제 2
다음 Spring Boot 코드를 FastAPI로 변환하세요:

```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;

    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }
}

@RestController
public class ProductController {
    @Autowired
    private ProductService productService;

    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id);
    }
}
```

---

## 7.12 다음 챕터 예고

다음 챕터에서는 **고급 의존성 패턴**을 다룹니다:
- yield를 사용한 컨텍스트 관리
- 의존성 오버라이드 (테스트용)
- 의존성 캐싱
- Spring의 `@Bean`과 비교

---

[← 이전: Chapter 6. 응답 모델과 상태 코드](../part02-request-response/chapter06-response.md) | [목차로](../README.md) | [다음: Chapter 8. 고급 의존성 패턴 →](chapter08-advanced-di.md)