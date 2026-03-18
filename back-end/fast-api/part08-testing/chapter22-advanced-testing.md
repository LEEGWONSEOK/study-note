# Chapter 22: 고급 테스팅 (Advanced Testing)

## 학습 목표
- 인증/인가 테스트 구현
- 파일 업로드/다운로드 테스트
- WebSocket 테스트
- 성능 테스트 작성
- E2E 테스트 구현
- 테스트 더블(Mock, Stub, Spy) 활용

## 1. 인증/인가 테스트

### 1.1 JWT 인증 테스트

**app/auth.py:**
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from datetime import datetime, timedelta

SECRET_KEY = "test-secret-key"
ALGORITHM = "HS256"

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid credentials")
        return {"username": username}
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid credentials")
```

**app/main.py:**
```python
from fastapi import FastAPI, Depends
from app.auth import get_current_user, create_access_token

app = FastAPI()

@app.post("/token")
async def login(username: str, password: str):
    # 실제로는 DB에서 사용자 검증
    if username == "testuser" and password == "testpass":
        access_token = create_access_token(data={"sub": username})
        return {"access_token": access_token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Incorrect username or password")

@app.get("/users/me")
async def read_users_me(current_user: dict = Depends(get_current_user)):
    return current_user

@app.get("/protected")
async def protected_route(current_user: dict = Depends(get_current_user)):
    return {"message": f"Hello {current_user['username']}, this is protected!"}
```

**tests/test_auth.py:**
```python
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.auth import create_access_token

client = TestClient(app)

class TestAuthentication:
    """인증 테스트"""

    def test_login_success(self):
        """로그인 성공"""
        response = client.post(
            "/token",
            data={"username": "testuser", "password": "testpass"}
        )
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"

    def test_login_invalid_credentials(self):
        """잘못된 자격증명"""
        response = client.post(
            "/token",
            data={"username": "wronguser", "password": "wrongpass"}
        )
        assert response.status_code == 401
        assert "Incorrect username or password" in response.json()["detail"]

    def test_access_protected_route_with_token(self):
        """토큰으로 보호된 라우트 접근"""
        # 토큰 생성
        token = create_access_token(data={"sub": "testuser"})

        # 헤더에 토큰 포함
        response = client.get(
            "/protected",
            headers={"Authorization": f"Bearer {token}"}
        )
        assert response.status_code == 200
        assert "Hello testuser" in response.json()["message"]

    def test_access_protected_route_without_token(self):
        """토큰 없이 보호된 라우트 접근 (실패)"""
        response = client.get("/protected")
        assert response.status_code == 401

    def test_access_protected_route_with_invalid_token(self):
        """잘못된 토큰으로 접근"""
        response = client.get(
            "/protected",
            headers={"Authorization": "Bearer invalid_token"}
        )
        assert response.status_code == 401

    def test_expired_token(self):
        """만료된 토큰"""
        from datetime import timedelta

        # 이미 만료된 토큰 생성
        expired_token = create_access_token(
            data={"sub": "testuser"},
            expires_delta=timedelta(seconds=-1)
        )

        response = client.get(
            "/protected",
            headers={"Authorization": f"Bearer {expired_token}"}
        )
        assert response.status_code == 401
```

### 1.2 권한 테스트 (Role-Based Access)

**app/auth.py (추가):**
```python
from typing import List

def require_roles(allowed_roles: List[str]):
    async def role_checker(current_user: dict = Depends(get_current_user)):
        user_roles = current_user.get("roles", [])
        if not any(role in allowed_roles for role in user_roles):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return role_checker
```

**app/main.py (추가):**
```python
@app.get("/admin")
async def admin_only(current_user: dict = Depends(require_roles(["admin"]))):
    return {"message": "Admin access granted"}
```

**tests/test_authorization.py:**
```python
import pytest
from app.auth import create_access_token

class TestAuthorization:
    """권한 테스트"""

    def test_admin_access_with_admin_role(self, client):
        """Admin 역할로 Admin 엔드포인트 접근"""
        token = create_access_token(data={"sub": "admin_user", "roles": ["admin"]})

        response = client.get(
            "/admin",
            headers={"Authorization": f"Bearer {token}"}
        )
        assert response.status_code == 200

    def test_admin_access_with_user_role(self, client):
        """User 역할로 Admin 엔드포인트 접근 (실패)"""
        token = create_access_token(data={"sub": "regular_user", "roles": ["user"]})

        response = client.get(
            "/admin",
            headers={"Authorization": f"Bearer {token}"}
        )
        assert response.status_code == 403
        assert "Insufficient permissions" in response.json()["detail"]

    def test_admin_access_without_roles(self, client):
        """역할 없이 Admin 엔드포인트 접근"""
        token = create_access_token(data={"sub": "no_role_user"})

        response = client.get(
            "/admin",
            headers={"Authorization": f"Bearer {token}"}
        )
        assert response.status_code == 403
```

### 1.3 인증 Fixture

**tests/conftest.py:**
```python
import pytest
from app.auth import create_access_token

@pytest.fixture
def auth_headers():
    """인증 헤더 fixture"""
    def _auth_headers(username: str = "testuser", roles: list = None):
        data = {"sub": username}
        if roles:
            data["roles"] = roles
        token = create_access_token(data=data)
        return {"Authorization": f"Bearer {token}"}
    return _auth_headers

@pytest.fixture
def admin_headers(auth_headers):
    """Admin 인증 헤더"""
    return auth_headers(username="admin", roles=["admin"])

@pytest.fixture
def user_headers(auth_headers):
    """일반 사용자 인증 헤더"""
    return auth_headers(username="user", roles=["user"])
```

**tests/test_with_fixtures.py:**
```python
def test_protected_route_with_auth_fixture(client, user_headers):
    """인증 fixture 사용"""
    response = client.get("/protected", headers=user_headers)
    assert response.status_code == 200

def test_admin_route_with_admin_fixture(client, admin_headers):
    """Admin fixture 사용"""
    response = client.get("/admin", headers=admin_headers)
    assert response.status_code == 200

def test_admin_route_with_user_fixture(client, user_headers):
    """User fixture로 Admin 라우트 접근 (실패)"""
    response = client.get("/admin", headers=user_headers)
    assert response.status_code == 403
```

## 2. 파일 업로드/다운로드 테스트

### 2.1 파일 업로드 테스트

**app/main.py:**
```python
from fastapi import File, UploadFile
import shutil
from pathlib import Path

UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    file_path = UPLOAD_DIR / file.filename
    with file_path.open("wb") as buffer:
        shutil.copyfileobj(file.file, buffer)

    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": file_path.stat().st_size
    }

@app.post("/upload-multiple")
async def upload_multiple_files(files: List[UploadFile] = File(...)):
    uploaded_files = []
    for file in files:
        file_path = UPLOAD_DIR / file.filename
        with file_path.open("wb") as buffer:
            shutil.copyfileobj(file.file, buffer)
        uploaded_files.append(file.filename)

    return {"uploaded_files": uploaded_files}
```

**tests/test_file_upload.py:**
```python
import pytest
from io import BytesIO
from pathlib import Path

class TestFileUpload:
    """파일 업로드 테스트"""

    def test_upload_text_file(self, client):
        """텍스트 파일 업로드"""
        content = b"Hello, this is a test file!"

        response = client.post(
            "/upload",
            files={"file": ("test.txt", BytesIO(content), "text/plain")}
        )

        assert response.status_code == 200
        data = response.json()
        assert data["filename"] == "test.txt"
        assert data["content_type"] == "text/plain"
        assert data["size"] == len(content)

    def test_upload_image_file(self, client):
        """이미지 파일 업로드"""
        # 가짜 이미지 데이터
        image_data = b"\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR..."

        response = client.post(
            "/upload",
            files={"file": ("image.png", BytesIO(image_data), "image/png")}
        )

        assert response.status_code == 200
        assert response.json()["filename"] == "image.png"

    def test_upload_multiple_files(self, client):
        """여러 파일 동시 업로드"""
        files = [
            ("files", ("file1.txt", BytesIO(b"Content 1"), "text/plain")),
            ("files", ("file2.txt", BytesIO(b"Content 2"), "text/plain")),
            ("files", ("file3.txt", BytesIO(b"Content 3"), "text/plain")),
        ]

        response = client.post("/upload-multiple", files=files)

        assert response.status_code == 200
        data = response.json()
        assert len(data["uploaded_files"]) == 3
        assert "file1.txt" in data["uploaded_files"]

    def test_upload_large_file(self, client):
        """큰 파일 업로드"""
        # 1MB 파일 생성
        large_content = b"x" * (1024 * 1024)

        response = client.post(
            "/upload",
            files={"file": ("large.bin", BytesIO(large_content), "application/octet-stream")}
        )

        assert response.status_code == 200
        assert response.json()["size"] == len(large_content)

    @pytest.fixture(autouse=True)
    def cleanup_uploads(self):
        """테스트 후 업로드 파일 정리"""
        yield
        upload_dir = Path("uploads")
        if upload_dir.exists():
            for file in upload_dir.iterdir():
                file.unlink()
```

### 2.2 파일 다운로드 테스트

**app/main.py:**
```python
from fastapi.responses import FileResponse

@app.get("/download/{filename}")
async def download_file(filename: str):
    file_path = UPLOAD_DIR / filename
    if not file_path.exists():
        raise HTTPException(status_code=404, detail="File not found")

    return FileResponse(
        path=file_path,
        filename=filename,
        media_type="application/octet-stream"
    )
```

**tests/test_file_download.py:**
```python
from pathlib import Path

class TestFileDownload:
    """파일 다운로드 테스트"""

    @pytest.fixture
    def sample_file(self):
        """샘플 파일 생성"""
        file_path = Path("uploads") / "sample.txt"
        file_path.parent.mkdir(exist_ok=True)
        file_path.write_text("Sample content")
        yield file_path
        file_path.unlink()

    def test_download_existing_file(self, client, sample_file):
        """존재하는 파일 다운로드"""
        response = client.get("/download/sample.txt")

        assert response.status_code == 200
        assert response.content == b"Sample content"
        assert "sample.txt" in response.headers.get("content-disposition", "")

    def test_download_nonexistent_file(self, client):
        """존재하지 않는 파일 다운로드"""
        response = client.get("/download/nonexistent.txt")
        assert response.status_code == 404
```

## 3. WebSocket 테스트

### 3.1 기본 WebSocket 테스트

**app/main.py:**
```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

**tests/test_websocket.py:**
```python
from fastapi.testclient import TestClient

class TestWebSocket:
    """WebSocket 테스트"""

    def test_websocket_echo(self, client):
        """WebSocket 에코 테스트"""
        with client.websocket_connect("/ws") as websocket:
            # 메시지 전송
            websocket.send_text("Hello")

            # 응답 수신
            data = websocket.receive_text()
            assert data == "Echo: Hello"

            # 여러 메시지 테스트
            websocket.send_text("World")
            data = websocket.receive_text()
            assert data == "Echo: World"

    def test_websocket_json(self, client):
        """JSON 데이터 WebSocket 테스트"""
        with client.websocket_connect("/ws/json") as websocket:
            # JSON 전송
            websocket.send_json({"message": "test", "value": 123})

            # JSON 수신
            data = websocket.receive_json()
            assert data["message"] == "test"
            assert data["value"] == 123
```

### 3.2 채팅 애플리케이션 테스트

**app/main.py:**
```python
from typing import List
from fastapi import WebSocket

class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Message: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

**tests/test_websocket_chat.py:**
```python
import pytest
from fastapi.testclient import TestClient

class TestWebSocketChat:
    """WebSocket 채팅 테스트"""

    def test_chat_broadcast(self, client):
        """채팅 브로드캐스트 테스트"""
        # 두 클라이언트 연결
        with client.websocket_connect("/ws/chat") as ws1:
            with client.websocket_connect("/ws/chat") as ws2:
                # 첫 번째 클라이언트가 메시지 전송
                ws1.send_text("Hello from client 1")

                # 두 클라이언트 모두 메시지 수신
                msg1 = ws1.receive_text()
                msg2 = ws2.receive_text()

                assert msg1 == "Message: Hello from client 1"
                assert msg2 == "Message: Hello from client 1"
```

## 4. 외부 API Mock 테스트

### 4.1 httpx Mock

**app/external.py:**
```python
import httpx

async def fetch_user_from_api(user_id: int):
    """외부 API에서 사용자 정보 가져오기"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.example.com/users/{user_id}")
        response.raise_for_status()
        return response.json()

@app.get("/external-user/{user_id}")
async def get_external_user(user_id: int):
    user_data = await fetch_user_from_api(user_id)
    return user_data
```

**tests/test_external_api.py:**
```python
import pytest
from unittest.mock import AsyncMock, patch
import httpx

class TestExternalAPI:
    """외부 API Mock 테스트"""

    @pytest.mark.asyncio
    @patch("httpx.AsyncClient.get")
    async def test_fetch_user_success(self, mock_get):
        """외부 API 성공 응답"""
        # Mock 응답 설정
        mock_response = AsyncMock()
        mock_response.json.return_value = {"id": 123, "name": "Test User"}
        mock_response.raise_for_status = AsyncMock()
        mock_get.return_value = mock_response

        # 함수 호출
        from app.external import fetch_user_from_api
        result = await fetch_user_from_api(123)

        # 검증
        assert result == {"id": 123, "name": "Test User"}
        mock_get.assert_called_once()

    @pytest.mark.asyncio
    @patch("httpx.AsyncClient.get")
    async def test_fetch_user_api_error(self, mock_get):
        """외부 API 에러"""
        # HTTP 에러 발생
        mock_response = AsyncMock()
        mock_response.raise_for_status.side_effect = httpx.HTTPStatusError(
            "Not Found", request=None, response=None
        )
        mock_get.return_value = mock_response

        # 예외 발생 확인
        from app.external import fetch_user_from_api
        with pytest.raises(httpx.HTTPStatusError):
            await fetch_user_from_api(999)
```

### 4.2 respx 라이브러리 사용

```bash
pip install respx
```

**tests/test_external_with_respx.py:**
```python
import pytest
import httpx
import respx
from app.external import fetch_user_from_api

@pytest.mark.asyncio
@respx.mock
async def test_fetch_user_with_respx():
    """respx를 사용한 외부 API Mock"""
    # Mock 응답 설정
    respx.get("https://api.example.com/users/123").mock(
        return_value=httpx.Response(
            200,
            json={"id": 123, "name": "Test User", "email": "test@example.com"}
        )
    )

    # 함수 호출
    result = await fetch_user_from_api(123)

    # 검증
    assert result["id"] == 123
    assert result["name"] == "Test User"

@pytest.mark.asyncio
@respx.mock
async def test_fetch_user_404_with_respx():
    """404 에러 Mock"""
    respx.get("https://api.example.com/users/999").mock(
        return_value=httpx.Response(404, json={"error": "Not found"})
    )

    # 예외 발생 확인
    with pytest.raises(httpx.HTTPStatusError):
        await fetch_user_from_api(999)
```

## 5. 데이터베이스 Mock 테스트

### 5.1 Repository Mock

**app/repository.py:**
```python
from sqlalchemy.orm import Session
from app import models

class UserRepository:
    def __init__(self, db: Session):
        self.db = db

    def get_user(self, user_id: int):
        return self.db.query(models.User).filter(models.User.id == user_id).first()

    def create_user(self, username: str, email: str):
        user = models.User(username=username, email=email)
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user
```

**tests/test_repository_mock.py:**
```python
from unittest.mock import Mock
from app.repository import UserRepository
from app import models

class TestUserRepositoryMock:
    """Repository Mock 테스트"""

    def test_get_user(self):
        """사용자 조회 Mock"""
        # Mock DB 세션
        mock_db = Mock()
        mock_query = Mock()
        mock_filter = Mock()

        # Mock 체이닝 설정
        mock_db.query.return_value = mock_query
        mock_query.filter.return_value = mock_filter

        # 반환값 설정
        expected_user = models.User(id=1, username="testuser", email="test@example.com")
        mock_filter.first.return_value = expected_user

        # Repository 테스트
        repo = UserRepository(mock_db)
        user = repo.get_user(1)

        # 검증
        assert user.id == 1
        assert user.username == "testuser"
        mock_db.query.assert_called_once_with(models.User)

    def test_create_user(self):
        """사용자 생성 Mock"""
        mock_db = Mock()

        repo = UserRepository(mock_db)
        user = repo.create_user("newuser", "new@example.com")

        # DB 메서드 호출 확인
        mock_db.add.assert_called_once()
        mock_db.commit.assert_called_once()
        mock_db.refresh.assert_called_once()
```

## 6. 성능 테스트

### 6.1 Locust를 사용한 부하 테스트

```bash
pip install locust
```

**locustfile.py:**
```python
from locust import HttpUser, task, between

class FastAPIUser(HttpUser):
    wait_time = between(1, 3)  # 1-3초 대기

    @task(3)  # 가중치 3
    def get_users(self):
        """사용자 목록 조회"""
        self.client.get("/users")

    @task(2)  # 가중치 2
    def get_user(self):
        """특정 사용자 조회"""
        self.client.get("/users/1")

    @task(1)  # 가중치 1
    def create_user(self):
        """사용자 생성"""
        self.client.post("/users", json={
            "username": "loadtest_user",
            "email": "loadtest@example.com"
        })

    def on_start(self):
        """테스트 시작 시 로그인"""
        response = self.client.post("/token", data={
            "username": "testuser",
            "password": "testpass"
        })
        self.token = response.json()["access_token"]
        self.client.headers.update({"Authorization": f"Bearer {self.token}"})
```

**실행:**
```bash
# 웹 UI로 실행
locust -f locustfile.py --host=http://localhost:8000

# 헤드리스 모드 (CLI)
locust -f locustfile.py --host=http://localhost:8000 \
    --users 100 --spawn-rate 10 --run-time 1m --headless
```

### 6.2 pytest-benchmark

```bash
pip install pytest-benchmark
```

**tests/test_performance.py:**
```python
import pytest

def test_create_user_performance(benchmark, client):
    """사용자 생성 성능 테스트"""
    def create_user():
        return client.post("/users", json={
            "username": "perftest",
            "email": "perf@example.com"
        })

    result = benchmark(create_user)
    assert result.status_code == 200

def test_list_users_performance(benchmark, client, db_session):
    """사용자 목록 조회 성능 테스트"""
    # 1000명의 사용자 생성
    from app import models
    users = [
        models.User(username=f"user{i}", email=f"user{i}@example.com")
        for i in range(1000)
    ]
    db_session.add_all(users)
    db_session.commit()

    # 벤치마크
    result = benchmark(lambda: client.get("/users"))
    assert result.status_code == 200
```

**실행:**
```bash
pytest tests/test_performance.py --benchmark-only
```

## 7. E2E 테스트

### 7.1 Playwright 사용

```bash
pip install playwright
playwright install
```

**tests/test_e2e.py:**
```python
import pytest
from playwright.sync_api import Page, expect

@pytest.fixture(scope="session")
def browser():
    """브라우저 fixture"""
    from playwright.sync_api import sync_playwright
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        yield browser
        browser.close()

@pytest.fixture
def page(browser):
    """페이지 fixture"""
    page = browser.new_page()
    yield page
    page.close()

def test_user_registration_flow(page: Page):
    """사용자 등록 E2E 테스트"""
    # 페이지 방문
    page.goto("http://localhost:8000/register")

    # 폼 입력
    page.fill("#username", "e2e_user")
    page.fill("#email", "e2e@example.com")
    page.fill("#password", "password123")

    # 제출
    page.click("button[type='submit']")

    # 성공 메시지 확인
    expect(page.locator(".success-message")).to_contain_text("Registration successful")

def test_login_and_view_profile(page: Page):
    """로그인 후 프로필 조회 E2E"""
    # 로그인
    page.goto("http://localhost:8000/login")
    page.fill("#username", "testuser")
    page.fill("#password", "testpass")
    page.click("button[type='submit']")

    # 프로필 페이지로 이동
    page.goto("http://localhost:8000/profile")

    # 프로필 정보 확인
    expect(page.locator(".username")).to_contain_text("testuser")
```

## 8. CI/CD 통합

### 8.1 GitHub Actions 설정

**.github/workflows/test.yml:**
```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov

    - name: Run tests
      run: |
        pytest tests/ -v --cov=app --cov-report=xml

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        fail_ci_if_error: true
```

### 8.2 Docker 테스트 환경

**docker-compose.test.yml:**
```yaml
version: '3.8'

services:
  test-app:
    build: .
    command: pytest tests/ -v
    environment:
      - DATABASE_URL=postgresql://test:test@test-db/test_db
      - TESTING=true
    depends_on:
      - test-db

  test-db:
    image: postgres:15
    environment:
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
      - POSTGRES_DB=test_db
```

**실행:**
```bash
docker-compose -f docker-compose.test.yml up --abort-on-container-exit
```

## 9. 테스트 커버리지

### 9.1 pytest-cov 사용

```bash
pip install pytest-cov
```

**실행:**
```bash
# 커버리지 측정
pytest --cov=app tests/

# HTML 리포트 생성
pytest --cov=app --cov-report=html tests/

# 누락된 라인 표시
pytest --cov=app --cov-report=term-missing tests/
```

**출력 예시:**
```
---------- coverage: platform darwin, python 3.11.0 -----------
Name                Stmts   Miss  Cover   Missing
-------------------------------------------------
app/__init__.py         0      0   100%
app/main.py            45      2    96%   78-79
app/auth.py            32      5    84%   45-49
app/models.py          15      0   100%
-------------------------------------------------
TOTAL                  92      7    92%
```

### 9.2 커버리지 설정

**.coveragerc:**
```ini
[run]
source = app
omit =
    */tests/*
    */venv/*
    */__pycache__/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
```

## 10. 모범 사례

### 10.1 테스트 피라미드

```
        E2E (소수)
       /        \
      /          \
     /  통합 테스트  \
    /    (중간)     \
   /                \
  /   단위 테스트      \
 /     (다수)         \
/____________________\
```

**비율:**
- 단위 테스트: 70%
- 통합 테스트: 20%
- E2E 테스트: 10%

### 10.2 Given-When-Then 패턴

```python
def test_user_can_update_profile():
    # Given: 사용자가 로그인된 상태
    token = login_user("testuser", "password")
    headers = {"Authorization": f"Bearer {token}"}

    # When: 프로필을 업데이트할 때
    response = client.put(
        "/profile",
        json={"bio": "Updated bio"},
        headers=headers
    )

    # Then: 성공적으로 업데이트됨
    assert response.status_code == 200
    assert response.json()["bio"] == "Updated bio"
```

### 10.3 테스트 데이터 빌더 패턴

```python
class UserBuilder:
    """테스트 사용자 빌더"""
    def __init__(self):
        self.username = "default_user"
        self.email = "default@example.com"
        self.roles = ["user"]

    def with_username(self, username):
        self.username = username
        return self

    def with_email(self, email):
        self.email = email
        return self

    def with_roles(self, roles):
        self.roles = roles
        return self

    def build(self):
        return {
            "username": self.username,
            "email": self.email,
            "roles": self.roles
        }

# 사용
def test_admin_user():
    admin = UserBuilder() \
        .with_username("admin") \
        .with_roles(["admin", "user"]) \
        .build()

    response = client.post("/users", json=admin)
    assert response.status_code == 200
```

## 11. 요약

### 핵심 포인트

1. **인증/인가 테스트**
   - JWT 토큰 테스트
   - 권한 검증
   - 인증 Fixture 활용

2. **파일 처리 테스트**
   - BytesIO로 파일 시뮬레이션
   - 업로드/다운로드 검증

3. **WebSocket 테스트**
   - TestClient.websocket_connect()
   - 브로드캐스트 테스트

4. **Mock과 Stub**
   - 외부 API Mock (respx)
   - 의존성 오버라이드
   - unittest.mock 활용

5. **성능 테스트**
   - Locust (부하 테스트)
   - pytest-benchmark

6. **E2E 테스트**
   - Playwright
   - 실제 사용자 시나리오

7. **CI/CD 통합**
   - GitHub Actions
   - Docker 테스트 환경

### 다음 챕터 예고

**Chapter 23: 테스트 커버리지와 품질 (Test Coverage and Quality)**

다음 챕터에서는 테스트 품질과 커버리지를 다룹니다:
- 코드 커버리지 분석
- 뮤테이션 테스트
- 테스트 품질 메트릭
- 테스트 리팩토링
- TDD (Test-Driven Development)

완벽한 테스트는 자신 있는 배포와 유지보수를 가능하게 합니다!
