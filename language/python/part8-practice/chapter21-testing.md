# Chapter 21. 테스팅 (Testing)

> **Java 개발자를 위한 노트**: Python의 테스팅은 Java의 JUnit/TestNG와 매우 유사합니다. `unittest`는 JUnit과, `pytest`는 더 현대적인 접근 방식을 제공합니다.

---

## 1. 테스팅이란?

### 1.1 왜 테스트를 작성하나?

**비유**: 테스트는 코드의 **안전벨트**입니다. 리팩토링이나 새 기능 추가 시 기존 코드가 깨지지 않았음을 보장합니다.

```python
"""
┌─────────────────────────────────────┐
│        테스트의 장점                 │
├─────────────────────────────────────┤
│                                     │
│  ✅ 버그를 조기에 발견               │
│  ✅ 리팩토링 시 안전성 보장          │
│  ✅ 코드 문서화 역할                │
│  ✅ 설계 개선 (테스트하기 쉬운 코드) │
│  ✅ 자신감 있는 배포                │
│                                     │
└─────────────────────────────────────┘
"""
```

### Java vs Python

**Java (JUnit)**:
```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    @Test
    void testAdd() {
        Calculator calc = new Calculator();
        assertEquals(5, calc.add(2, 3));
    }
}
```

**Python (pytest)**:
```python
# test_calculator.py
def test_add():
    calc = Calculator()
    assert calc.add(2, 3) == 5
```

---

## 2. unittest - 표준 테스팅 프레임워크

### 2.1 기본 사용법

```python
# calculator.py
class Calculator:
    """간단한 계산기"""

    def add(self, a, b):
        return a + b

    def subtract(self, a, b):
        return a - b

    def multiply(self, a, b):
        return a * b

    def divide(self, a, b):
        if b == 0:
            raise ValueError("0으로 나눌 수 없습니다")
        return a / b
```

```python
# test_calculator.py
import unittest
from calculator import Calculator

class TestCalculator(unittest.TestCase):
    """Calculator 테스트"""

    def setUp(self):
        """각 테스트 전에 실행"""
        self.calc = Calculator()
        print("🔧 setUp 실행")

    def tearDown(self):
        """각 테스트 후에 실행"""
        print("🧹 tearDown 실행")

    def test_add(self):
        """덧셈 테스트"""
        result = self.calc.add(2, 3)
        self.assertEqual(result, 5)

    def test_subtract(self):
        """뺄셈 테스트"""
        result = self.calc.subtract(10, 3)
        self.assertEqual(result, 7)

    def test_multiply(self):
        """곱셈 테스트"""
        result = self.calc.multiply(4, 5)
        self.assertEqual(result, 20)

    def test_divide(self):
        """나눗셈 테스트"""
        result = self.calc.divide(10, 2)
        self.assertEqual(result, 5.0)

    def test_divide_by_zero(self):
        """0으로 나누기 예외 테스트"""
        with self.assertRaises(ValueError):
            self.calc.divide(10, 0)

if __name__ == '__main__':
    unittest.main()
```

**실행**:
```bash
python test_calculator.py

# 출력:
# 🔧 setUp 실행
# 🧹 tearDown 실행
# .🔧 setUp 실행
# 🧹 tearDown 실행
# .🔧 setUp 실행
# 🧹 tearDown 실행
# .🔧 setUp 실행
# 🧹 tearDown 실행
# .🔧 setUp 실행
# 🧹 tearDown 실행
# .
# ----------------------------------------------------------------------
# Ran 5 tests in 0.001s
#
# OK
```

### 2.2 주요 assert 메서드

```python
import unittest

class TestAssertions(unittest.TestCase):
    """다양한 assertion 테스트"""

    def test_equality(self):
        """동등성"""
        self.assertEqual(1 + 1, 2)
        self.assertNotEqual(1 + 1, 3)

    def test_truth(self):
        """참/거짓"""
        self.assertTrue(True)
        self.assertFalse(False)

    def test_none(self):
        """None 체크"""
        self.assertIsNone(None)
        self.assertIsNotNone("hello")

    def test_in(self):
        """포함 여부"""
        self.assertIn(3, [1, 2, 3])
        self.assertNotIn(4, [1, 2, 3])

    def test_instance(self):
        """타입 체크"""
        self.assertIsInstance("hello", str)
        self.assertIsInstance(123, int)

    def test_greater_less(self):
        """크기 비교"""
        self.assertGreater(10, 5)
        self.assertLess(5, 10)
        self.assertGreaterEqual(10, 10)
        self.assertLessEqual(5, 5)

    def test_almost_equal(self):
        """부동소수점 비교"""
        self.assertAlmostEqual(0.1 + 0.2, 0.3, places=7)

    def test_regex(self):
        """정규표현식 매칭"""
        self.assertRegex("hello world", r"hello")
        self.assertNotRegex("hello world", r"goodbye")
```

---

## 3. pytest - 현대적 테스팅 프레임워크

### 3.1 pytest 기본

**비유**: pytest는 unittest보다 **더 간결하고 강력한 스위스 아미 나이프**입니다.

```python
# test_calculator_pytest.py
from calculator import Calculator
import pytest

def test_add():
    """덧셈 테스트 - 간결함!"""
    calc = Calculator()
    assert calc.add(2, 3) == 5

def test_subtract():
    """뺄셈 테스트"""
    calc = Calculator()
    assert calc.subtract(10, 3) == 7

def test_divide_by_zero():
    """0으로 나누기 예외"""
    calc = Calculator()
    with pytest.raises(ValueError):
        calc.divide(10, 0)

def test_divide_by_zero_with_message():
    """예외 메시지 확인"""
    calc = Calculator()
    with pytest.raises(ValueError, match="0으로 나눌 수 없습니다"):
        calc.divide(10, 0)
```

**실행**:
```bash
pytest test_calculator_pytest.py -v

# 출력:
# test_calculator_pytest.py::test_add PASSED                   [ 25%]
# test_calculator_pytest.py::test_subtract PASSED              [ 50%]
# test_calculator_pytest.py::test_divide_by_zero PASSED        [ 75%]
# test_calculator_pytest.py::test_divide_by_zero_with_message PASSED [100%]
#
# ============================== 4 passed in 0.01s ==============================
```

### 3.2 Fixtures - 테스트 설정

**비유**: Fixture는 테스트를 위한 **무대 장치**입니다. 매번 같은 준비 과정을 자동화합니다.

```python
# conftest.py (pytest 설정 파일)
import pytest
from calculator import Calculator

@pytest.fixture
def calc():
    """Calculator 인스턴스 제공"""
    print("\n🔧 Calculator 생성")
    calculator = Calculator()
    yield calculator
    print("\n🧹 Calculator 정리")

@pytest.fixture
def sample_numbers():
    """샘플 숫자 제공"""
    return [1, 2, 3, 4, 5]

# test_with_fixtures.py
def test_add_with_fixture(calc):
    """fixture 사용"""
    assert calc.add(2, 3) == 5

def test_multiply_with_fixture(calc):
    """fixture 재사용"""
    assert calc.multiply(4, 5) == 20

def test_sum_numbers(sample_numbers):
    """샘플 데이터 사용"""
    assert sum(sample_numbers) == 15
```

### 3.3 Parametrize - 여러 케이스 테스트

```python
import pytest
from calculator import Calculator

@pytest.fixture
def calc():
    return Calculator()

@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (10, 5, 15),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300)
])
def test_add_multiple_cases(calc, a, b, expected):
    """여러 케이스를 한 번에 테스트"""
    assert calc.add(a, b) == expected

@pytest.mark.parametrize("a, b, expected", [
    (10, 2, 5.0),
    (20, 4, 5.0),
    (100, 10, 10.0),
    (7, 2, 3.5)
])
def test_divide_multiple_cases(calc, a, b, expected):
    """나눗셈 여러 케이스"""
    assert calc.divide(a, b) == expected

# 실행:
# pytest test_parametrize.py -v
#
# test_add_multiple_cases[2-3-5] PASSED                        [ 11%]
# test_add_multiple_cases[10-5-15] PASSED                      [ 22%]
# test_add_multiple_cases[-1-1-0] PASSED                       [ 33%]
# test_add_multiple_cases[0-0-0] PASSED                        [ 44%]
# test_add_multiple_cases[100-200-300] PASSED                  [ 55%]
# test_divide_multiple_cases[10-2-5.0] PASSED                  [ 66%]
# ...
```

---

## 4. Mock - 의존성 격리

**비유**: Mock은 영화의 **스턴트맨**입니다. 실제 배우(객체) 대신 위험한 장면(외부 의존성)을 대신합니다.

### 4.1 기본 Mock 사용

```python
# user_service.py
import requests

class UserService:
    """사용자 서비스"""

    def __init__(self, api_url):
        self.api_url = api_url

    def get_user(self, user_id):
        """API에서 사용자 정보 가져오기"""
        response = requests.get(f"{self.api_url}/users/{user_id}")
        response.raise_for_status()
        return response.json()

    def is_admin(self, user_id):
        """관리자 여부 확인"""
        user = self.get_user(user_id)
        return user.get('role') == 'admin'
```

```python
# test_user_service.py
from unittest.mock import Mock, patch
import pytest
from user_service import UserService

def test_get_user_with_mock():
    """Mock을 사용한 테스트"""
    # Mock 객체 생성
    mock_response = Mock()
    mock_response.json.return_value = {
        'id': 1,
        'name': 'Alice',
        'role': 'admin'
    }
    mock_response.raise_for_status.return_value = None

    # requests.get을 Mock으로 대체
    with patch('user_service.requests.get', return_value=mock_response):
        service = UserService('https://api.example.com')
        user = service.get_user(1)

        assert user['name'] == 'Alice'
        assert user['role'] == 'admin'

def test_is_admin_true():
    """관리자 확인 - True"""
    mock_response = Mock()
    mock_response.json.return_value = {'id': 1, 'role': 'admin'}
    mock_response.raise_for_status.return_value = None

    with patch('user_service.requests.get', return_value=mock_response):
        service = UserService('https://api.example.com')
        assert service.is_admin(1) is True

def test_is_admin_false():
    """관리자 확인 - False"""
    mock_response = Mock()
    mock_response.json.return_value = {'id': 2, 'role': 'user'}
    mock_response.raise_for_status.return_value = None

    with patch('user_service.requests.get', return_value=mock_response):
        service = UserService('https://api.example.com')
        assert service.is_admin(2) is False
```

### 4.2 Mock 호출 검증

```python
from unittest.mock import Mock, call

def test_mock_call_verification():
    """Mock 호출 검증"""
    # Mock 객체 생성
    mock_db = Mock()

    # 메서드 호출
    mock_db.insert('user', {'name': 'Alice'})
    mock_db.insert('user', {'name': 'Bob'})
    mock_db.update('user', 1, {'name': 'Charlie'})

    # 호출 검증
    assert mock_db.insert.call_count == 2
    assert mock_db.update.call_count == 1

    # 특정 인수로 호출되었는지 확인
    mock_db.insert.assert_called_with('user', {'name': 'Bob'})

    # 모든 호출 확인
    assert mock_db.insert.call_args_list == [
        call('user', {'name': 'Alice'}),
        call('user', {'name': 'Bob'})
    ]
```

### 4.3 실전 예제 - 이메일 전송 서비스

```python
# email_service.py
import smtplib
from email.mime.text import MIMEText

class EmailService:
    """이메일 전송 서비스"""

    def __init__(self, smtp_server, smtp_port):
        self.smtp_server = smtp_server
        self.smtp_port = smtp_port

    def send_email(self, to, subject, body):
        """이메일 전송"""
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['To'] = to

        with smtplib.SMTP(self.smtp_server, self.smtp_port) as server:
            server.send_message(msg)

        return True

# test_email_service.py
from unittest.mock import patch, MagicMock
import pytest
from email_service import EmailService

def test_send_email():
    """이메일 전송 테스트"""
    # SMTP Mock
    with patch('email_service.smtplib.SMTP') as mock_smtp:
        # Mock 인스턴스 설정
        mock_server = MagicMock()
        mock_smtp.return_value.__enter__.return_value = mock_server

        # 이메일 전송
        service = EmailService('smtp.example.com', 587)
        result = service.send_email(
            'user@example.com',
            'Test Subject',
            'Test Body'
        )

        # 검증
        assert result is True
        mock_smtp.assert_called_once_with('smtp.example.com', 587)
        mock_server.send_message.assert_called_once()
```

---

## 5. 테스트 커버리지

**비유**: 커버리지는 코드의 **지도**입니다. 어떤 부분이 테스트되었고, 어떤 부분이 누락되었는지 보여줍니다.

### 5.1 pytest-cov 사용

```bash
# 설치
pip install pytest-cov

# 실행
pytest --cov=calculator --cov-report=html

# 출력:
# ----------- coverage: platform darwin, python 3.10.0 -----------
# Name                Stmts   Miss  Cover
# ---------------------------------------
# calculator.py          10      0   100%
# ---------------------------------------
# TOTAL                  10      0   100%
#
# Coverage HTML written to dir htmlcov
```

### 5.2 커버리지 리포트 해석

```python
# calculator.py (일부 누락)
class Calculator:
    def add(self, a, b):
        return a + b

    def subtract(self, a, b):
        return a - b

    def complex_operation(self, a, b):
        """복잡한 연산 - 테스트 누락!"""
        if a > b:
            result = a * b  # 테스트되지 않음
        else:
            result = a + b

        if result > 100:
            return result * 2  # 테스트되지 않음
        return result

# 커버리지:
# Name                Stmts   Miss  Cover   Missing
# -------------------------------------------------
# calculator.py          12      2    83%    12, 15
```

---

## 6. TDD (Test-Driven Development)

**비유**: TDD는 **지도를 먼저 그리고 여행하는 것**입니다.

### 6.1 TDD 사이클

```python
"""
┌───────────────────────────────┐
│      TDD 사이클 (Red-Green-Refactor)     │
├───────────────────────────────┤
│                               │
│  1. 🔴 Red: 실패하는 테스트 작성  │
│  2. 🟢 Green: 최소한의 코드로 통과 │
│  3. 🔵 Refactor: 코드 개선        │
│                               │
│  → 반복                        │
│                               │
└───────────────────────────────┘
"""
```

### 6.2 TDD 실습 예제

**요구사항**: 은행 계좌 클래스 구현

```python
# test_bank_account.py
import pytest
from bank_account import BankAccount

# 🔴 Step 1: 실패하는 테스트 작성
def test_initial_balance():
    """초기 잔액 테스트"""
    account = BankAccount(1000)
    assert account.balance == 1000

# 이 시점에서 BankAccount가 없으므로 테스트 실패!
```

```python
# bank_account.py
# 🟢 Step 2: 최소한의 코드로 테스트 통과
class BankAccount:
    def __init__(self, initial_balance):
        self.balance = initial_balance

# 이제 테스트 통과!
```

```python
# test_bank_account.py
# 🔴 Step 3: 다음 기능 테스트 추가
def test_deposit():
    """입금 테스트"""
    account = BankAccount(1000)
    account.deposit(500)
    assert account.balance == 1500

# 테스트 실패 - deposit 메서드 없음!
```

```python
# bank_account.py
# 🟢 Step 4: deposit 구현
class BankAccount:
    def __init__(self, initial_balance):
        self.balance = initial_balance

    def deposit(self, amount):
        self.balance += amount

# 테스트 통과!
```

```python
# test_bank_account.py
# 🔴 Step 5: 출금 테스트 추가
def test_withdraw():
    """출금 테스트"""
    account = BankAccount(1000)
    account.withdraw(300)
    assert account.balance == 700

def test_withdraw_insufficient_funds():
    """잔액 부족 테스트"""
    account = BankAccount(100)
    with pytest.raises(ValueError, match="잔액 부족"):
        account.withdraw(200)

# 테스트 실패!
```

```python
# bank_account.py
# 🟢 Step 6: withdraw 구현
class BankAccount:
    def __init__(self, initial_balance):
        self.balance = initial_balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("입금액은 0보다 커야 합니다")
        self.balance += amount

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("출금액은 0보다 커야 합니다")
        if amount > self.balance:
            raise ValueError("잔액 부족")
        self.balance -= amount

# 🔵 Step 7: 리팩토링 (필요시)
# - 중복 코드 제거
# - 가독성 개선
# - 성능 최적화
```

---

## 7. 실전 팁

### 💡 Tip 1: 테스트 이름을 명확하게

```python
# ❌ 나쁜 예
def test_1():
    assert add(1, 2) == 3

# ✅ 좋은 예
def test_add_positive_numbers_returns_correct_sum():
    assert add(1, 2) == 3

def test_add_negative_numbers_returns_correct_sum():
    assert add(-1, -2) == -3
```

### 💡 Tip 2: AAA 패턴 사용

```python
def test_user_registration():
    # Arrange (준비)
    user_service = UserService()
    email = "test@example.com"
    password = "secure_password"

    # Act (실행)
    result = user_service.register(email, password)

    # Assert (검증)
    assert result.success is True
    assert result.user.email == email
```

### 💡 Tip 3: 테스트는 독립적으로

```python
# ❌ 나쁜 예 - 테스트 간 의존성
class TestBad:
    user_id = None

    def test_create_user(self):
        user = create_user("Alice")
        self.user_id = user.id  # 상태 저장!

    def test_get_user(self):
        user = get_user(self.user_id)  # 이전 테스트에 의존!
        assert user.name == "Alice"

# ✅ 좋은 예 - 독립적
def test_create_user():
    user = create_user("Alice")
    assert user.name == "Alice"

def test_get_user():
    # 각 테스트가 독립적으로 데이터 준비
    user = create_user("Bob")
    fetched_user = get_user(user.id)
    assert fetched_user.name == "Bob"
```

### 💡 Tip 4: 실패 메시지 제공

```python
def test_with_custom_message():
    result = complex_calculation()
    expected = 42

    assert result == expected, f"Expected {expected}, but got {result}"
```

---

## 8. 연습 문제

### 문제 1: 쇼핑 카트 TDD
TDD 방식으로 쇼핑 카트 클래스를 구현하세요.

**요구사항**:
- 상품 추가 (add_item)
- 상품 제거 (remove_item)
- 총 금액 계산 (total)
- 할인 적용 (apply_discount)

<details>
<summary>정답 보기</summary>

```python
# test_shopping_cart.py
import pytest
from shopping_cart import ShoppingCart, Item

def test_empty_cart_has_zero_total():
    """빈 카트의 총액은 0"""
    cart = ShoppingCart()
    assert cart.total() == 0

def test_add_single_item():
    """단일 상품 추가"""
    cart = ShoppingCart()
    cart.add_item(Item("Apple", 100, 2))
    assert cart.total() == 200

def test_add_multiple_items():
    """여러 상품 추가"""
    cart = ShoppingCart()
    cart.add_item(Item("Apple", 100, 2))
    cart.add_item(Item("Banana", 50, 3))
    assert cart.total() == 350  # 200 + 150

def test_remove_item():
    """상품 제거"""
    cart = ShoppingCart()
    apple = Item("Apple", 100, 2)
    cart.add_item(apple)
    cart.remove_item("Apple")
    assert cart.total() == 0

def test_apply_discount():
    """할인 적용"""
    cart = ShoppingCart()
    cart.add_item(Item("Apple", 100, 2))
    cart.apply_discount(0.1)  # 10% 할인
    assert cart.total() == 180

# shopping_cart.py
from dataclasses import dataclass

@dataclass
class Item:
    name: str
    price: float
    quantity: int

class ShoppingCart:
    def __init__(self):
        self.items = []
        self.discount = 0.0

    def add_item(self, item: Item):
        self.items.append(item)

    def remove_item(self, item_name: str):
        self.items = [item for item in self.items if item.name != item_name]

    def total(self):
        subtotal = sum(item.price * item.quantity for item in self.items)
        return subtotal * (1 - self.discount)

    def apply_discount(self, discount: float):
        if not 0 <= discount <= 1:
            raise ValueError("할인율은 0~1 사이여야 합니다")
        self.discount = discount
```
</details>

### 문제 2: Mock을 사용한 날씨 서비스 테스트
외부 API를 호출하는 날씨 서비스를 Mock으로 테스트하세요.

<details>
<summary>정답 보기</summary>

```python
# weather_service.py
import requests

class WeatherService:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.weather.com"

    def get_temperature(self, city):
        """도시의 현재 온도 가져오기"""
        url = f"{self.base_url}/current"
        params = {'city': city, 'apikey': self.api_key}

        response = requests.get(url, params=params)
        response.raise_for_status()

        data = response.json()
        return data['temperature']

    def is_raining(self, city):
        """비가 오는지 확인"""
        url = f"{self.base_url}/current"
        params = {'city': city, 'apikey': self.api_key}

        response = requests.get(url, params=params)
        response.raise_for_status()

        data = response.json()
        return data['conditions'] == 'rain'

# test_weather_service.py
from unittest.mock import Mock, patch
import pytest
from weather_service import WeatherService

def test_get_temperature():
    """온도 조회 테스트"""
    mock_response = Mock()
    mock_response.json.return_value = {'temperature': 25, 'conditions': 'sunny'}
    mock_response.raise_for_status.return_value = None

    with patch('weather_service.requests.get', return_value=mock_response):
        service = WeatherService('test_api_key')
        temperature = service.get_temperature('Seoul')

        assert temperature == 25

def test_is_raining_true():
    """비 오는지 확인 - True"""
    mock_response = Mock()
    mock_response.json.return_value = {'temperature': 18, 'conditions': 'rain'}
    mock_response.raise_for_status.return_value = None

    with patch('weather_service.requests.get', return_value=mock_response):
        service = WeatherService('test_api_key')
        assert service.is_raining('Seoul') is True

def test_is_raining_false():
    """비 오는지 확인 - False"""
    mock_response = Mock()
    mock_response.json.return_value = {'temperature': 25, 'conditions': 'sunny'}
    mock_response.raise_for_status.return_value = None

    with patch('weather_service.requests.get', return_value=mock_response):
        service = WeatherService('test_api_key')
        assert service.is_raining('Seoul') is False

def test_api_error():
    """API 에러 처리"""
    mock_response = Mock()
    mock_response.raise_for_status.side_effect = requests.HTTPError("API Error")

    with patch('weather_service.requests.get', return_value=mock_response):
        service = WeatherService('test_api_key')

        with pytest.raises(requests.HTTPError):
            service.get_temperature('Seoul')
```
</details>

---

## 핵심 요약

1. **테스팅 프레임워크**
   - `unittest`: 표준 라이브러리, JUnit 스타일
   - `pytest`: 현대적, 간결, 강력 (권장)

2. **핵심 개념**
   ```python
   # unittest
   class TestExample(unittest.TestCase):
       def test_something(self):
           self.assertEqual(actual, expected)

   # pytest
   def test_something():
       assert actual == expected
   ```

3. **Mock 사용**
   - 외부 의존성 격리
   - 테스트 속도 향상
   - 예측 가능한 결과

4. **TDD 사이클**
   - Red: 실패하는 테스트 작성
   - Green: 최소 코드로 통과
   - Refactor: 코드 개선

5. **베스트 프랙티스**
   - 명확한 테스트 이름
   - AAA 패턴 (Arrange-Act-Assert)
   - 독립적인 테스트
   - 높은 커버리지 (80%+ 목표)

6. **Java와 비교**
   - unittest ≈ JUnit
   - pytest ≈ Modern JUnit with extensions
   - Mock ≈ Mockito

---

**다음 장**: [Chapter 22. 로깅과 디버깅](chapter22-logging-debugging.md)에서는 애플리케이션 로깅과 디버깅 기법을 배웁니다.