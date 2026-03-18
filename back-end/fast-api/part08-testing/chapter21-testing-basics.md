# Chapter 21: 테스팅 기초 (Testing Basics)

## 학습 목표
- pytest와 FastAPI TestClient 사용법 학습
- 단위 테스트와 통합 테스트 작성
- 비동기 테스트 구현
- Fixture와 Mock 활용
- 테스트 데이터베이스 설정
- Spring Boot의 @SpringBootTest와 비교

## 1. 테스팅 개요

### 1.1 왜 테스트가 중요한가?

1. **버그 조기 발견**: 프로덕션 배포 전 문제 발견
2. **리팩토링 안정성**: 코드 변경 시 기존 기능 보호
3. **문서화**: 테스트 코드가 사용법을 보여줌
4. **자신감**: 배포 시 안심
5. **유지보수성**: 장기적으로 코드 품질 유지

### 1.2 테스트 종류

**단위 테스트 (Unit Test):**
- 개별 함수/메서드 테스트
- 빠르고 격리됨
- 외부 의존성 없음 (Mock 사용)

**통합 테스트 (Integration Test):**
- 여러 컴포넌트 함께 테스트
- 실제 데이터베이스 사용
- 외부 API 호출 테스트

**E2E 테스트 (End-to-End Test):**
- 전체 시스템 테스트
- 사용자 시나리오 검증
- 가장 느림

### 1.3 Spring Boot와 비교

| 구분 | Spring Boot | FastAPI |
|------|-------------|---------|
| **테스트 프레임워크** | JUnit 5 | pytest |
| **클라이언트** | MockMvc, WebTestClient | TestClient |
| **의존성 주입** | `@MockBean`, `@SpyBean` | `app.dependency_overrides` |
| **DB 테스트** | `@DataJpaTest` | 테스트 DB 직접 설정 |
| **비동기 테스트** | `@Test` + `reactor-test` | `@pytest.mark.asyncio` |
| **트랜잭션 롤백** | `@Transactional` (자동) | 수동 설정 필요 |

## 2. 테스트 환경 설정

### 2.1 pytest 설치

```bash
pip install pytest pytest-asyncio httpx
```

### 2.2 프로젝트 구조

```
my_fastapi_project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   ├── schemas.py
│   └── crud.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py        # Fixture 정의
│   ├── test_main.py       # 엔드포인트 테스트
│   ├── test_crud.py       # CRUD 로직 테스트
│   └── test_integration.py # 통합 테스트
├── pytest.ini             # pytest 설정
└── requirements.txt
```

**pytest.ini:**
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
asyncio_mode = auto
```

## 3. TestClient 기본 사용

### 3.1 간단한 테스트

**app/main.py:**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id, "name": f"Item {item_id}"}
```

**tests/test_main.py:**
```python
from fastapi.testclient import TestClient
from app.main import app

# TestClient 생성
client = TestClient(app)

def test_root():
    """루트 엔드포인트 테스트"""
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_get_item():
    """아이템 조회 테스트"""
    response = client.get("/items/123")
    assert response.status_code == 200
    assert response.json() == {"item_id": 123, "name": "Item 123"}

def test_get_item_not_integer():
    """잘못된 타입 테스트"""
    response = client.get("/items/abc")
    assert response.status_code == 422  # Validation error
```

**테스트 실행:**
```bash
pytest                    # 모든 테스트 실행
pytest tests/test_main.py # 특정 파일만
pytest -v                 # 상세 출력
pytest -k test_root       # 이름으로 필터링
pytest -x                 # 첫 실패 시 중단
```

### 3.2 다양한 HTTP 메서드 테스트

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from fastapi.testclient import TestClient

app = FastAPI()

# 간단한 인메모리 DB
items_db = {}

class Item(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create_item(item: Item):
    item_id = len(items_db) + 1
    items_db[item_id] = item.dict()
    return {"id": item_id, **items_db[item_id]}

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    return items_db[item_id]

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    items_db[item_id] = item.dict()
    return items_db[item_id]

@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    del items_db[item_id]
    return {"message": "Item deleted"}

# 테스트
client = TestClient(app)

def test_create_item():
    """POST 테스트"""
    response = client.post(
        "/items",
        json={"name": "Apple", "price": 1.5}
    )
    assert response.status_code == 200
    assert response.json()["name"] == "Apple"
    assert response.json()["price"] == 1.5

def test_get_item():
    """GET 테스트"""
    # 먼저 생성
    create_response = client.post("/items", json={"name": "Banana", "price": 2.0})
    item_id = create_response.json()["id"]

    # 조회
    response = client.get(f"/items/{item_id}")
    assert response.status_code == 200
    assert response.json()["name"] == "Banana"

def test_get_item_not_found():
    """404 에러 테스트"""
    response = client.get("/items/999")
    assert response.status_code == 404
    assert response.json() == {"detail": "Item not found"}

def test_update_item():
    """PUT 테스트"""
    # 생성
    create_response = client.post("/items", json={"name": "Orange", "price": 3.0})
    item_id = create_response.json()["id"]

    # 업데이트
    response = client.put(
        f"/items/{item_id}",
        json={"name": "Orange Updated", "price": 3.5}
    )
    assert response.status_code == 200
    assert response.json()["name"] == "Orange Updated"
    assert response.json()["price"] == 3.5

def test_delete_item():
    """DELETE 테스트"""
    # 생성
    create_response = client.post("/items", json={"name": "Grape", "price": 4.0})
    item_id = create_response.json()["id"]

    # 삭제
    response = client.delete(f"/items/{item_id}")
    assert response.status_code == 200
    assert response.json() == {"message": "Item deleted"}

    # 삭제 확인
    get_response = client.get(f"/items/{item_id}")
    assert get_response.status_code == 404
```

## 4. Fixture 사용

Fixture는 테스트에 필요한 설정을 재사용 가능하게 만듭니다.

### 4.1 conftest.py

**tests/conftest.py:**
```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client():
    """TestClient fixture"""
    return TestClient(app)

@pytest.fixture
def sample_item():
    """샘플 아이템 데이터"""
    return {"name": "Test Item", "price": 10.0}
```

**tests/test_with_fixtures.py:**
```python
def test_create_item(client, sample_item):
    """Fixture 사용"""
    response = client.post("/items", json=sample_item)
    assert response.status_code == 200
    assert response.json()["name"] == sample_item["name"]

def test_another_test(client):
    """동일한 client fixture 재사용"""
    response = client.get("/")
    assert response.status_code == 200
```

### 4.2 Setup/Teardown Fixture

```python
import pytest

@pytest.fixture
def setup_database():
    """테스트 전후 DB 설정"""
    # Setup: 테스트 전 실행
    print("Setting up database...")
    db = setup_test_db()

    yield db  # 테스트 실행

    # Teardown: 테스트 후 실행
    print("Tearing down database...")
    db.close()

def test_with_db(setup_database):
    """데이터베이스 사용 테스트"""
    db = setup_database
    # 테스트 로직...
    assert db is not None
```

### 4.3 Scope Fixture

```python
import pytest

@pytest.fixture(scope="function")  # 각 테스트마다 (기본값)
def function_fixture():
    return "function"

@pytest.fixture(scope="class")  # 클래스당 한 번
def class_fixture():
    return "class"

@pytest.fixture(scope="module")  # 모듈당 한 번
def module_fixture():
    return "module"

@pytest.fixture(scope="session")  # 전체 세션당 한 번
def session_fixture():
    return "session"
```

## 5. 데이터베이스 테스트

### 5.1 테스트 데이터베이스 설정

**conftest.py:**
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.database import Base
from app.main import app, get_db

# 테스트 DB (SQLite 인메모리)
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture
def db_session():
    """테스트 DB 세션"""
    # 테이블 생성
    Base.metadata.create_all(bind=engine)

    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()

    # 테이블 삭제 (클린업)
    Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(db_session):
    """DB 의존성을 오버라이드한 TestClient"""
    def override_get_db():
        try:
            yield db_session
        finally:
            db_session.close()

    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()
```

**tests/test_users.py:**
```python
from app import models

def test_create_user(client):
    """사용자 생성 테스트"""
    response = client.post(
        "/users",
        json={"username": "testuser", "email": "test@example.com"}
    )
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"

def test_get_user(client, db_session):
    """사용자 조회 테스트"""
    # DB에 직접 데이터 삽입
    user = models.User(username="alice", email="alice@example.com")
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)

    # API로 조회
    response = client.get(f"/users/{user.id}")
    assert response.status_code == 200
    assert response.json()["username"] == "alice"

def test_list_users(client, db_session):
    """사용자 목록 조회 테스트"""
    # 여러 사용자 생성
    users = [
        models.User(username=f"user{i}", email=f"user{i}@example.com")
        for i in range(5)
    ]
    db_session.add_all(users)
    db_session.commit()

    # 목록 조회
    response = client.get("/users")
    assert response.status_code == 200
    assert len(response.json()) == 5
```

### 5.2 트랜잭션 롤백

각 테스트 후 DB를 원상태로 되돌립니다.

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def db_session():
    """트랜잭션 롤백을 지원하는 세션"""
    connection = engine.connect()
    transaction = connection.begin()
    session = TestingSessionLocal(bind=connection)

    yield session

    # 롤백
    session.close()
    transaction.rollback()
    connection.close()
```

**Spring Boot 비교:**
```java
@SpringBootTest
@Transactional  // 자동 롤백
public class UserServiceTest {
    @Autowired
    private UserRepository userRepository;

    @Test
    public void testCreateUser() {
        User user = new User("testuser", "test@example.com");
        userRepository.save(user);

        User found = userRepository.findByUsername("testuser");
        assertEquals("testuser", found.getUsername());
    }
    // 테스트 후 자동 롤백
}
```

## 6. Mock과 Patch

외부 의존성을 Mock으로 대체합니다.

### 6.1 unittest.mock 사용

```python
from unittest.mock import patch, MagicMock
import pytest

# 테스트 대상 함수
async def send_email(to: str, subject: str):
    """이메일 발송 (외부 서비스 호출)"""
    # 실제로는 SMTP 서버에 연결
    smtp_client.send(to, subject)
    return True

# 테스트
@patch("app.email.smtp_client")
def test_send_email(mock_smtp):
    """이메일 발송 Mock 테스트"""
    # Mock 설정
    mock_smtp.send.return_value = True

    # 함수 호출
    result = send_email("test@example.com", "Hello")

    # 검증
    assert result is True
    mock_smtp.send.assert_called_once_with("test@example.com", "Hello")
```

### 6.2 의존성 오버라이드

FastAPI의 의존성 주입을 활용한 Mock입니다.

```python
from fastapi import Depends, FastAPI
from fastapi.testclient import TestClient

app = FastAPI()

# 실제 의존성
def get_current_user():
    # 실제로는 토큰 검증
    return {"id": 1, "username": "real_user"}

@app.get("/me")
async def read_current_user(current_user=Depends(get_current_user)):
    return current_user

# 테스트
def test_read_current_user():
    """의존성 오버라이드 테스트"""
    # Mock 의존성
    def mock_get_current_user():
        return {"id": 999, "username": "test_user"}

    # 의존성 오버라이드
    app.dependency_overrides[get_current_user] = mock_get_current_user

    client = TestClient(app)
    response = client.get("/me")

    assert response.status_code == 200
    assert response.json() == {"id": 999, "username": "test_user"}

    # 정리
    app.dependency_overrides.clear()
```

### 6.3 외부 API Mock

```python
from unittest.mock import patch, AsyncMock
import pytest

# 외부 API 호출 함수
async def fetch_external_data(user_id: int):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.example.com/users/{user_id}")
        return response.json()

# 테스트
@pytest.mark.asyncio
@patch("httpx.AsyncClient.get")
async def test_fetch_external_data(mock_get):
    """외부 API Mock 테스트"""
    # Mock 응답 설정
    mock_response = AsyncMock()
    mock_response.json.return_value = {"id": 123, "name": "Test User"}
    mock_get.return_value = mock_response

    # 함수 호출
    result = await fetch_external_data(123)

    # 검증
    assert result == {"id": 123, "name": "Test User"}
    mock_get.assert_called_once()
```

## 7. 비동기 테스트

### 7.1 pytest-asyncio 사용

```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_async_endpoint():
    """비동기 엔드포인트 테스트"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/")
    assert response.status_code == 200

@pytest.mark.asyncio
async def test_async_database_operation():
    """비동기 DB 테스트"""
    async with async_session() as session:
        user = User(username="testuser", email="test@example.com")
        session.add(user)
        await session.commit()

        result = await session.execute(select(User).filter_by(username="testuser"))
        fetched_user = result.scalar_one()

        assert fetched_user.username == "testuser"
```

### 7.2 비동기 Fixture

```python
import pytest
from httpx import AsyncClient

@pytest.fixture
async def async_client():
    """비동기 클라이언트 fixture"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_with_async_fixture(async_client):
    """비동기 fixture 사용"""
    response = await async_client.get("/users")
    assert response.status_code == 200
```

## 8. 테스트 조직화

### 8.1 테스트 클래스

```python
import pytest

class TestUserEndpoints:
    """사용자 엔드포인트 테스트 그룹"""

    def test_create_user(self, client):
        response = client.post("/users", json={"username": "alice", "email": "alice@example.com"})
        assert response.status_code == 200

    def test_get_user(self, client):
        response = client.get("/users/1")
        assert response.status_code == 200

    def test_update_user(self, client):
        response = client.put("/users/1", json={"username": "alice_updated", "email": "alice@example.com"})
        assert response.status_code == 200

    def test_delete_user(self, client):
        response = client.delete("/users/1")
        assert response.status_code == 200

class TestItemEndpoints:
    """아이템 엔드포인트 테스트 그룹"""

    def test_create_item(self, client):
        response = client.post("/items", json={"name": "Item 1", "price": 10.0})
        assert response.status_code == 200

    def test_get_item(self, client):
        response = client.get("/items/1")
        assert response.status_code == 200
```

### 8.2 Parametrize (파라미터화)

```python
import pytest

@pytest.mark.parametrize("item_id,expected_status", [
    (1, 200),
    (2, 200),
    (999, 404),
    (-1, 422),
])
def test_get_item_parametrized(client, item_id, expected_status):
    """여러 케이스를 파라미터로 테스트"""
    response = client.get(f"/items/{item_id}")
    assert response.status_code == expected_status

@pytest.mark.parametrize("username,email,expected_status", [
    ("valid_user", "valid@example.com", 200),
    ("", "valid@example.com", 422),  # 빈 username
    ("user", "invalid_email", 422),  # 잘못된 이메일
    ("a" * 100, "long@example.com", 422),  # 너무 긴 username
])
def test_create_user_validation(client, username, email, expected_status):
    """유효성 검증 테스트"""
    response = client.post("/users", json={"username": username, "email": email})
    assert response.status_code == expected_status
```

## 9. 실전 예제: 완전한 테스트 세트

**tests/conftest.py:**
```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app, get_db
from app.database import Base

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture(scope="function")
def db_session():
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
    Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def client(db_session):
    def override_get_db():
        try:
            yield db_session
        finally:
            db_session.close()

    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()

@pytest.fixture
def sample_user():
    return {"username": "testuser", "email": "test@example.com", "password": "password123"}
```

**tests/test_users_complete.py:**
```python
import pytest
from app import models

class TestUserCRUD:
    """사용자 CRUD 테스트"""

    def test_create_user(self, client, sample_user):
        """사용자 생성"""
        response = client.post("/users", json=sample_user)
        assert response.status_code == 200
        data = response.json()
        assert data["username"] == sample_user["username"]
        assert data["email"] == sample_user["email"]
        assert "id" in data

    def test_create_duplicate_user(self, client, sample_user):
        """중복 사용자 생성 실패"""
        client.post("/users", json=sample_user)
        response = client.post("/users", json=sample_user)
        assert response.status_code == 409

    def test_get_user(self, client, db_session):
        """사용자 조회"""
        user = models.User(username="alice", email="alice@example.com")
        db_session.add(user)
        db_session.commit()
        db_session.refresh(user)

        response = client.get(f"/users/{user.id}")
        assert response.status_code == 200
        assert response.json()["username"] == "alice"

    def test_get_nonexistent_user(self, client):
        """존재하지 않는 사용자 조회"""
        response = client.get("/users/999")
        assert response.status_code == 404

    def test_list_users(self, client, db_session):
        """사용자 목록 조회"""
        users = [
            models.User(username=f"user{i}", email=f"user{i}@example.com")
            for i in range(3)
        ]
        db_session.add_all(users)
        db_session.commit()

        response = client.get("/users")
        assert response.status_code == 200
        assert len(response.json()) == 3

    def test_update_user(self, client, db_session):
        """사용자 업데이트"""
        user = models.User(username="bob", email="bob@example.com")
        db_session.add(user)
        db_session.commit()
        db_session.refresh(user)

        response = client.put(
            f"/users/{user.id}",
            json={"username": "bob_updated", "email": "bob_new@example.com"}
        )
        assert response.status_code == 200
        assert response.json()["username"] == "bob_updated"

    def test_delete_user(self, client, db_session):
        """사용자 삭제"""
        user = models.User(username="charlie", email="charlie@example.com")
        db_session.add(user)
        db_session.commit()
        db_session.refresh(user)

        response = client.delete(f"/users/{user.id}")
        assert response.status_code == 200

        # 삭제 확인
        response = client.get(f"/users/{user.id}")
        assert response.status_code == 404
```

## 10. 모범 사례

### 10.1 테스트 작성 원칙

1. **AAA 패턴**: Arrange (준비), Act (실행), Assert (검증)
```python
def test_create_user(client):
    # Arrange
    user_data = {"username": "test", "email": "test@example.com"}

    # Act
    response = client.post("/users", json=user_data)

    # Assert
    assert response.status_code == 200
    assert response.json()["username"] == "test"
```

2. **하나의 테스트, 하나의 개념**
```python
# ✅ 좋은 예
def test_create_user_success(client):
    response = client.post("/users", json={"username": "alice", "email": "alice@example.com"})
    assert response.status_code == 200

def test_create_user_duplicate_fails(client):
    client.post("/users", json={"username": "alice", "email": "alice@example.com"})
    response = client.post("/users", json={"username": "alice", "email": "alice@example.com"})
    assert response.status_code == 409

# ❌ 나쁜 예: 여러 개념 테스트
def test_create_user_everything(client):
    response1 = client.post("/users", json={"username": "alice", "email": "alice@example.com"})
    assert response1.status_code == 200

    response2 = client.post("/users", json={"username": "alice", "email": "alice@example.com"})
    assert response2.status_code == 409

    response3 = client.get("/users/1")
    assert response3.status_code == 200
```

3. **테스트 격리**: 각 테스트는 독립적
4. **명확한 이름**: 테스트가 무엇을 검증하는지 명확하게
5. **빠른 실행**: 단위 테스트는 빨라야 함

### 10.2 테스트 네이밍 컨벤션

```python
# 패턴: test_<function>_<scenario>_<expected_result>

def test_create_user_with_valid_data_returns_200():
    pass

def test_create_user_with_duplicate_username_returns_409():
    pass

def test_get_user_with_nonexistent_id_returns_404():
    pass

def test_update_user_with_invalid_email_returns_422():
    pass
```

## 11. 요약

### 핵심 포인트

1. **pytest + TestClient**: FastAPI 테스트의 기본
   - `from fastapi.testclient import TestClient`
   - 동기 테스트에 적합

2. **Fixture**: 재사용 가능한 테스트 설정
   - `@pytest.fixture`
   - Setup/Teardown

3. **데이터베이스 테스트**:
   - 테스트 DB 분리 (SQLite 인메모리)
   - `app.dependency_overrides`로 의존성 교체
   - 트랜잭션 롤백

4. **Mock**: 외부 의존성 격리
   - `unittest.mock.patch`
   - 의존성 오버라이드

5. **비동기 테스트**:
   - `@pytest.mark.asyncio`
   - `AsyncClient` 사용

6. **모범 사례**:
   - AAA 패턴
   - 하나의 테스트, 하나의 개념
   - 명확한 네이밍

### 다음 챕터 예고

**Chapter 22: 고급 테스팅 (Advanced Testing)**

다음 챕터에서는 고급 테스팅 기법을 다룹니다:
- 인증/인가 테스트
- 파일 업로드/다운로드 테스트
- WebSocket 테스트
- 성능 테스트
- E2E 테스트
- CI/CD 통합

견고한 테스트를 작성하면 자신 있게 코드를 배포할 수 있습니다!