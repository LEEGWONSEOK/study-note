# Chapter 23: 테스트 커버리지와 품질 (Test Coverage and Quality)

## 학습 목표
- 코드 커버리지 측정 및 분석
- 뮤테이션 테스트로 테스트 품질 검증
- 테스트 메트릭과 품질 지표
- TDD (Test-Driven Development) 실천
- 테스트 리팩토링 기법

## 1. 코드 커버리지

### 1.1 커버리지란?

코드 커버리지는 테스트가 실행하는 코드의 비율을 측정합니다.

**커버리지 유형:**
- **Line Coverage**: 실행된 코드 라인 비율
- **Branch Coverage**: 실행된 분기 비율 (if/else)
- **Function Coverage**: 실행된 함수 비율
- **Statement Coverage**: 실행된 문장 비율

### 1.2 pytest-cov 설정

```bash
pip install pytest-cov
```

**pyproject.toml:**
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "--cov=app",
    "--cov-report=html",
    "--cov-report=term-missing",
    "--cov-fail-under=80"  # 80% 미만이면 실패
]

[tool.coverage.run]
source = ["app"]
omit = [
    "*/tests/*",
    "*/venv/*",
    "*/__pycache__/*",
    "*/migrations/*"
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "@abstract"
]
```

**실행:**
```bash
# 기본 커버리지
pytest --cov=app tests/

# HTML 리포트 생성
pytest --cov=app --cov-report=html tests/
# htmlcov/index.html 에서 확인

# 누락된 라인 표시
pytest --cov=app --cov-report=term-missing tests/

# 특정 모듈만
pytest --cov=app.users tests/
```

### 1.3 커버리지 리포트 분석

**터미널 출력:**
```
---------- coverage: platform darwin, python 3.11.0 -----------
Name                    Stmts   Miss Branch BrPart  Cover   Missing
-------------------------------------------------------------------
app/__init__.py             2      0      0      0   100%
app/main.py                45      3     12      2    91%   78-79, 92
app/auth.py                32      5      8      1    82%   45-49, 67
app/models.py              15      0      0      0   100%
app/crud.py                28      4      6      1    83%   34-37
app/database.py            10      0      0      0   100%
-------------------------------------------------------------------
TOTAL                     132     12     26      4    88%
```

**해석:**
- **Stmts**: 전체 문장 수
- **Miss**: 실행되지 않은 문장 수
- **Branch**: 분기(if/else) 수
- **BrPart**: 부분적으로만 실행된 분기
- **Cover**: 커버리지 비율
- **Missing**: 누락된 라인 번호

### 1.4 커버리지 개선 예제

**커버리지가 낮은 코드:**
```python
# app/utils.py
def calculate_discount(price: float, user_type: str) -> float:
    """할인 계산"""
    if user_type == "premium":
        return price * 0.8  # 20% 할인
    elif user_type == "regular":
        return price * 0.9  # 10% 할인
    else:
        return price  # 할인 없음
```

**불완전한 테스트 (50% 커버리지):**
```python
def test_calculate_discount_premium():
    result = calculate_discount(100, "premium")
    assert result == 80
```

**개선된 테스트 (100% 커버리지):**
```python
import pytest

@pytest.mark.parametrize("price,user_type,expected", [
    (100, "premium", 80),    # premium 분기
    (100, "regular", 90),    # regular 분기
    (100, "guest", 100),     # else 분기
])
def test_calculate_discount(price, user_type, expected):
    result = calculate_discount(price, user_type)
    assert result == expected
```

## 2. Branch Coverage (분기 커버리지)

### 2.1 분기 커버리지의 중요성

```python
def check_eligibility(age: int, has_license: bool) -> str:
    """운전 자격 확인"""
    if age >= 18:
        if has_license:
            return "eligible"
        else:
            return "need_license"
    else:
        return "too_young"
```

**Line Coverage만 100%인 테스트:**
```python
def test_check_eligibility():
    # 한 가지 경로만 테스트 (라인은 모두 실행됨)
    assert check_eligibility(20, True) == "eligible"
```

**Branch Coverage 100%인 테스트:**
```python
@pytest.mark.parametrize("age,has_license,expected", [
    (20, True, "eligible"),        # age >= 18, has_license = True
    (20, False, "need_license"),   # age >= 18, has_license = False
    (16, True, "too_young"),       # age < 18
])
def test_check_eligibility_all_branches(age, has_license, expected):
    assert check_eligibility(age, has_license) == expected
```

### 2.2 분기 커버리지 측정

```bash
# 분기 커버리지 포함
pytest --cov=app --cov-branch tests/
```

## 3. 뮤테이션 테스트 (Mutation Testing)

### 3.1 뮤테이션 테스트란?

뮤테이션 테스트는 코드를 의도적으로 변경(mutate)하여 테스트가 이를 감지하는지 확인합니다.

**예시:**
```python
# 원본 코드
def is_adult(age: int) -> bool:
    return age >= 18

# 뮤턴트 1: >= → >
def is_adult(age: int) -> bool:
    return age > 18  # 18세가 성인이 아니게 됨

# 뮤턴트 2: >= → <=
def is_adult(age: int) -> bool:
    return age <= 18  # 논리가 반대로

# 뮤턴트 3: 18 → 19
def is_adult(age: int) -> bool:
    return age >= 19
```

**좋은 테스트는 모든 뮤턴트를 잡아냄 (kill):**
```python
def test_is_adult():
    assert is_adult(18) is True   # 뮤턴트 1, 3 잡음
    assert is_adult(17) is False  # 뮤턴트 2 잡음
    assert is_adult(25) is True
```

### 3.2 mutmut 사용

```bash
pip install mutmut
```

**실행:**
```bash
# 뮤테이션 테스트 실행
mutmut run --paths-to-mutate=app/

# 결과 확인
mutmut results

# 상세 결과
mutmut show all
```

**출력 예시:**
```
- Mutation testing starting -

1. Running tests without mutations
⠋ Running... Done

2. Checking mutants
⠙ Running... [##########] 45/45 done

- Results -

Survived: 3
Killed: 38
Timeout: 2
Suspicious: 1
Skipped: 1

Mutation score: 84.4%
```

**용어:**
- **Killed**: 테스트가 뮤턴트를 잡아냄 (좋음)
- **Survived**: 뮤턴트가 살아남음 (테스트 부족)
- **Timeout**: 테스트가 타임아웃
- **Suspicious**: 의심스러운 케이스

### 3.3 Survived Mutant 분석

```bash
# 살아남은 뮤턴트 확인
mutmut show 5
```

**예시:**
```python
# 원본
def calculate_total(items: list) -> float:
    total = 0
    for item in items:
        total += item.price
    return total

# Survived Mutant: += → -=
def calculate_total(items: list) -> float:
    total = 0
    for item in items:
        total -= item.price  # 살아남음!
    return total
```

**문제:** 테스트가 음수 결과를 확인하지 않음

**개선된 테스트:**
```python
def test_calculate_total():
    items = [Item(price=10), Item(price=20), Item(price=30)]
    total = calculate_total(items)
    assert total == 60  # 이제 mutant를 잡음
```

## 4. 테스트 품질 메트릭

### 4.1 주요 메트릭

**1. 코드 커버리지**
```bash
pytest --cov=app --cov-report=term
```

**목표:**
- 전체: 80% 이상
- 핵심 비즈니스 로직: 90% 이상
- 유틸리티 함수: 100%

**2. 뮤테이션 스코어**
```bash
mutmut run
```

**목표:**
- 80% 이상 (killed / total mutants)

**3. 테스트 실행 시간**
```bash
pytest --durations=10  # 가장 느린 10개 테스트 표시
```

**목표:**
- 단위 테스트: < 0.1초
- 통합 테스트: < 1초
- 전체 테스트 스위트: < 5분

**4. 테스트 안정성**
- Flaky Test (간헐적 실패) 비율: 0%

### 4.2 SonarQube 통합

**sonar-project.properties:**
```properties
sonar.projectKey=my-fastapi-project
sonar.sources=app
sonar.tests=tests
sonar.python.coverage.reportPaths=coverage.xml
sonar.python.version=3.11
```

**실행:**
```bash
# 커버리지 XML 생성
pytest --cov=app --cov-report=xml

# SonarQube 분석
sonar-scanner
```

## 5. TDD (Test-Driven Development)

### 5.1 TDD 사이클

```
1. Red: 실패하는 테스트 작성
   ↓
2. Green: 테스트를 통과하는 최소 코드 작성
   ↓
3. Refactor: 코드 개선
   ↓
(반복)
```

### 5.2 TDD 실전 예제

**요구사항:** 사용자 비밀번호 검증 함수 작성
- 최소 8자 이상
- 대문자 1개 이상
- 소문자 1개 이상
- 숫자 1개 이상

**Step 1: Red (실패하는 테스트)**
```python
# tests/test_password_validator.py
import pytest
from app.validators import validate_password

def test_password_length():
    """비밀번호는 최소 8자 이상"""
    assert validate_password("Short1A") is False
    assert validate_password("LongEnough1A") is True

# 실행: FAILED (함수가 없음)
```

**Step 2: Green (최소 코드)**
```python
# app/validators.py
def validate_password(password: str) -> bool:
    return len(password) >= 8

# 실행: PASSED
```

**Step 3: Refactor (다음 요구사항 추가)**
```python
# tests/test_password_validator.py (추가)
def test_password_requires_uppercase():
    """대문자 필수"""
    assert validate_password("lowercase1") is False
    assert validate_password("Uppercase1") is True
```

**Step 4: Green (코드 개선)**
```python
# app/validators.py
def validate_password(password: str) -> bool:
    if len(password) < 8:
        return False
    if not any(c.isupper() for c in password):
        return False
    return True
```

**Step 5: 모든 요구사항 완성**
```python
# tests/test_password_validator.py (완성)
@pytest.mark.parametrize("password,expected", [
    ("Short1A", False),           # 너무 짧음
    ("LongEnough1A", True),       # 통과
    ("nouppercase1", False),      # 대문자 없음
    ("NOLOWERCASE1", False),      # 소문자 없음
    ("NoNumbers", False),         # 숫자 없음
    ("Perfect1Pass", True),       # 모든 조건 만족
])
def test_password_validation(password, expected):
    assert validate_password(password) is expected

# app/validators.py (최종)
import re

def validate_password(password: str) -> bool:
    """비밀번호 검증"""
    if len(password) < 8:
        return False
    if not re.search(r"[A-Z]", password):  # 대문자
        return False
    if not re.search(r"[a-z]", password):  # 소문자
        return False
    if not re.search(r"\d", password):     # 숫자
        return False
    return True
```

### 5.3 TDD의 장점

1. **설계 개선**: 테스트 가능한 코드 작성
2. **문서화**: 테스트가 사용법을 보여줌
3. **회귀 방지**: 리팩토링 시 안전성
4. **빠른 피드백**: 즉시 문제 발견

## 6. 테스트 리팩토링

### 6.1 테스트 냄새 (Test Smells)

**1. 중복 코드**
```python
# ❌ 나쁜 예: 중복
def test_create_user_admin():
    user = User(username="admin", email="admin@example.com")
    db.add(user)
    db.commit()
    assert user.id is not None

def test_create_user_regular():
    user = User(username="user", email="user@example.com")
    db.add(user)
    db.commit()
    assert user.id is not None

# ✅ 좋은 예: Fixture 사용
@pytest.fixture
def create_user(db):
    def _create_user(username, email):
        user = User(username=username, email=email)
        db.add(user)
        db.commit()
        return user
    return _create_user

def test_create_user_admin(create_user):
    user = create_user("admin", "admin@example.com")
    assert user.id is not None

def test_create_user_regular(create_user):
    user = create_user("user", "user@example.com")
    assert user.id is not None
```

**2. 긴 테스트**
```python
# ❌ 나쁜 예: 너무 긴 테스트
def test_user_registration_flow():
    # 50 줄의 테스트 코드...
    pass

# ✅ 좋은 예: 분리
def test_user_can_register():
    # 등록만 테스트
    pass

def test_registered_user_receives_email():
    # 이메일 발송만 테스트
    pass

def test_registered_user_can_login():
    # 로그인만 테스트
    pass
```

**3. 불명확한 테스트 이름**
```python
# ❌ 나쁜 예
def test_user():
    pass

def test_case_1():
    pass

# ✅ 좋은 예
def test_create_user_with_valid_email_succeeds():
    pass

def test_create_user_with_duplicate_email_raises_409():
    pass
```

### 6.2 Object Mother 패턴

테스트 데이터 생성을 중앙화합니다.

```python
# tests/factories.py
from app.models import User, Post

class UserFactory:
    """사용자 테스트 데이터 생성"""

    @staticmethod
    def create_user(
        username: str = "testuser",
        email: str = "test@example.com",
        is_admin: bool = False,
        **kwargs
    ):
        return User(
            username=username,
            email=email,
            is_admin=is_admin,
            **kwargs
        )

    @staticmethod
    def create_admin(username: str = "admin"):
        return UserFactory.create_user(
            username=username,
            email=f"{username}@example.com",
            is_admin=True
        )

    @staticmethod
    def create_batch(count: int):
        return [
            UserFactory.create_user(
                username=f"user{i}",
                email=f"user{i}@example.com"
            )
            for i in range(count)
        ]

# 사용
def test_list_users(db):
    users = UserFactory.create_batch(5)
    db.add_all(users)
    db.commit()

    response = client.get("/users")
    assert len(response.json()) == 5

def test_admin_can_delete_user(db):
    admin = UserFactory.create_admin()
    user = UserFactory.create_user()
    # ...
```

### 6.3 Page Object 패턴 (E2E)

```python
# tests/pages/login_page.py
class LoginPage:
    """로그인 페이지 객체"""

    def __init__(self, page):
        self.page = page
        self.username_input = page.locator("#username")
        self.password_input = page.locator("#password")
        self.submit_button = page.locator("button[type='submit']")

    def navigate(self):
        self.page.goto("http://localhost:8000/login")

    def login(self, username: str, password: str):
        self.username_input.fill(username)
        self.password_input.fill(password)
        self.submit_button.click()

# 사용
def test_successful_login(page):
    login_page = LoginPage(page)
    login_page.navigate()
    login_page.login("testuser", "password")

    # 로그인 성공 확인
    expect(page).to_have_url("http://localhost:8000/dashboard")
```

## 7. 지속적인 테스트 개선

### 7.1 테스트 리뷰 체크리스트

```markdown
## 테스트 리뷰 체크리스트

### 가독성
- [ ] 테스트 이름이 명확한가?
- [ ] AAA 패턴을 따르는가? (Arrange, Act, Assert)
- [ ] 주석 없이도 이해 가능한가?

### 격리
- [ ] 각 테스트가 독립적인가?
- [ ] 테스트 순서에 의존하지 않는가?
- [ ] 외부 상태에 의존하지 않는가?

### 완전성
- [ ] 모든 분기를 테스트하는가?
- [ ] 엣지 케이스를 다루는가?
- [ ] 에러 케이스를 테스트하는가?

### 성능
- [ ] 테스트가 빠른가? (< 1초)
- [ ] 불필요한 설정이 없는가?
- [ ] 병렬 실행 가능한가?

### 유지보수성
- [ ] 중복이 없는가?
- [ ] Fixture를 적절히 사용하는가?
- [ ] 테스트가 깨지기 쉽지 않은가?
```

### 7.2 Pre-commit Hook

**.pre-commit-config.yaml:**
```yaml
repos:
  - repo: local
    hooks:
      - id: pytest-check
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
        args: [
          "tests/",
          "--cov=app",
          "--cov-fail-under=80",
          "-x"  # 첫 실패 시 중단
        ]

      - id: mypy
        name: mypy
        entry: mypy
        language: system
        types: [python]
        args: ["app/"]
```

**설치:**
```bash
pip install pre-commit
pre-commit install
```

## 8. 실전 예제: 완전한 테스트 스위트

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app, get_db
from app.database import Base
from app.auth import create_access_token

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

@pytest.fixture(scope="function")
def db():
    engine = create_engine(SQLALCHEMY_DATABASE_URL)
    TestingSessionLocal = sessionmaker(bind=engine)
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(db):
    def override_get_db():
        try:
            yield db
        finally:
            db.close()

    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()

@pytest.fixture
def auth_headers():
    def _auth_headers(username="testuser", roles=None):
        token = create_access_token({"sub": username, "roles": roles or []})
        return {"Authorization": f"Bearer {token}"}
    return _auth_headers

# tests/test_users_complete.py
import pytest
from app import models

class TestUserAPI:
    """사용자 API 완전한 테스트"""

    def test_create_user_success(self, client):
        """정상적인 사용자 생성"""
        response = client.post("/users", json={
            "username": "newuser",
            "email": "new@example.com",
            "password": "Password123"
        })
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "newuser"
        assert "id" in data

    @pytest.mark.parametrize("username,email,password,expected_status,expected_error", [
        ("", "valid@example.com", "Pass123", 422, "username"),
        ("valid", "invalid-email", "Pass123", 422, "email"),
        ("valid", "valid@example.com", "short", 422, "password"),
        ("a" * 100, "valid@example.com", "Pass123", 422, "username"),
    ])
    def test_create_user_validation(
        self, client, username, email, password, expected_status, expected_error
    ):
        """입력 검증 테스트"""
        response = client.post("/users", json={
            "username": username,
            "email": email,
            "password": password
        })
        assert response.status_code == expected_status
        assert expected_error in str(response.json())

    def test_create_duplicate_user(self, client):
        """중복 사용자 생성 실패"""
        user_data = {
            "username": "duplicate",
            "email": "dup@example.com",
            "password": "Pass123"
        }
        client.post("/users", json=user_data)
        response = client.post("/users", json=user_data)
        assert response.status_code == 409

    def test_get_user_by_id(self, client, db):
        """ID로 사용자 조회"""
        user = models.User(username="alice", email="alice@example.com")
        db.add(user)
        db.commit()
        db.refresh(user)

        response = client.get(f"/users/{user.id}")
        assert response.status_code == 200
        assert response.json()["username"] == "alice"

    def test_get_nonexistent_user(self, client):
        """존재하지 않는 사용자"""
        response = client.get("/users/9999")
        assert response.status_code == 404

    def test_update_user(self, client, db, auth_headers):
        """사용자 정보 업데이트"""
        user = models.User(username="bob", email="bob@example.com")
        db.add(user)
        db.commit()
        db.refresh(user)

        headers = auth_headers("bob")
        response = client.put(
            f"/users/{user.id}",
            json={"email": "new@example.com"},
            headers=headers
        )
        assert response.status_code == 200
        assert response.json()["email"] == "new@example.com"

    def test_delete_user(self, client, db, auth_headers):
        """사용자 삭제"""
        user = models.User(username="charlie", email="charlie@example.com")
        db.add(user)
        db.commit()
        db.refresh(user)

        headers = auth_headers("admin", roles=["admin"])
        response = client.delete(f"/users/{user.id}", headers=headers)
        assert response.status_code == 204

        # 삭제 확인
        assert client.get(f"/users/{user.id}").status_code == 404

    def test_list_users_pagination(self, client, db):
        """페이지네이션 테스트"""
        users = [
            models.User(username=f"user{i}", email=f"user{i}@example.com")
            for i in range(25)
        ]
        db.add_all(users)
        db.commit()

        # 첫 페이지
        response = client.get("/users?page=1&size=10")
        assert response.status_code == 200
        data = response.json()
        assert len(data["items"]) == 10
        assert data["total"] == 25

        # 두 번째 페이지
        response = client.get("/users?page=2&size=10")
        assert len(response.json()["items"]) == 10

        # 마지막 페이지
        response = client.get("/users?page=3&size=10")
        assert len(response.json()["items"]) == 5
```

## 9. 요약

### 핵심 포인트

1. **코드 커버리지**
   - pytest-cov로 측정
   - 80% 이상 목표
   - Branch coverage 포함

2. **뮤테이션 테스트**
   - mutmut 사용
   - 테스트 품질 검증
   - Survived mutant 분석

3. **테스트 메트릭**
   - 커버리지 비율
   - 뮤테이션 스코어
   - 실행 시간
   - 안정성

4. **TDD**
   - Red-Green-Refactor 사이클
   - 테스트 우선 작성
   - 설계 개선

5. **테스트 리팩토링**
   - Fixture 활용
   - Factory 패턴
   - 중복 제거

6. **지속적 개선**
   - Pre-commit hook
   - CI/CD 통합
   - 정기적 리뷰

### 다음 챕터 예고

**Chapter 24: 프로젝트 구조 (Project Structure)**

다음 챕터에서는 확장 가능한 프로젝트 구조를 다룹니다:
- 모듈화된 프로젝트 구조
- 레이어드 아키텍처
- 도메인 주도 설계 (DDD)
- 프로젝트 템플릿과 보일러플레이트
- Spring Boot의 패키지 구조와 비교

완벽한 테스트 전략으로 안정적인 애플리케이션을 구축하세요!