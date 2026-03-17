# Chapter 8. 예외 처리 (Exception Handling)

## 8.1 예외란?

> 🚨 **비유**: 예외는 **도로의 장애물**입니다.
> - 정상 주행: 프로그램 정상 실행
> - 장애물 발견: 예외 발생
> - 우회로: 예외 처리 (try-except)
> - 충돌: 프로그램 종료 (처리 안 함)

**예외 (Exception)**: 프로그램 실행 중 발생하는 **오류 상황**

---

### Java vs Python 비교

**Java**:
```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("0으로 나눌 수 없습니다");
} finally {
    System.out.println("항상 실행");
}
```

**Python**:
```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("0으로 나눌 수 없습니다")
finally:
    print("항상 실행")
```

---

## 8.2 기본 예외 처리

### 1. try-except

> 🛡️ **비유**: **안전망**
> - try: 위험한 시도
> - except: 떨어졌을 때 받아주는 그물

```python
# 예외가 발생하면 프로그램 종료
result = 10 / 0  # ZeroDivisionError!
```

```python
# 예외 처리로 프로그램 계속 실행
try:
    result = 10 / 0
except ZeroDivisionError:
    print("0으로 나눌 수 없습니다")
    result = None

print("프로그램 계속 실행")
```

---

### 2. 예외 메시지 받기

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"에러 발생: {e}")
    print(f"에러 타입: {type(e)}")

# 출력:
# 에러 발생: division by zero
# 에러 타입: <class 'ZeroDivisionError'>
```

---

### 3. 여러 예외 처리

**방법 1: 개별 처리**
```python
try:
    age = int(input("나이 입력: "))
    result = 100 / age
except ValueError:
    print("숫자를 입력하세요")
except ZeroDivisionError:
    print("0은 입력할 수 없습니다")

# 입력: abc → ValueError
# 입력: 0   → ZeroDivisionError
```

---

**방법 2: 튜플로 묶기**
```python
try:
    age = int(input("나이 입력: "))
    result = 100 / age
except (ValueError, ZeroDivisionError) as e:
    print(f"입력 오류: {e}")
```

---

### 4. 모든 예외 잡기 (비추천)

```python
try:
    # 위험한 코드
    result = 10 / 0
except Exception as e:
    print(f"알 수 없는 에러: {e}")
```

> ⚠️ **주의**: `except Exception`은 너무 광범위!
> - 구체적인 예외 타입 사용 권장
> - 디버깅 어려움

---

## 8.3 else와 finally

### else: 예외가 없을 때

> ✅ **비유**: **보너스**
> - 예외 없이 성공하면 실행

```python
try:
    age = int(input("나이: "))
except ValueError:
    print("숫자를 입력하세요")
else:
    print(f"입력된 나이: {age}")
    print("정상 처리 완료")
```

---

### finally: 항상 실행

> 🔒 **비유**: **마무리 정리**
> - 성공하든 실패하든 항상 실행
> - 파일 닫기, 연결 해제 등

```python
file = None
try:
    file = open("data.txt", "r")
    content = file.read()
    print(content)
except FileNotFoundError:
    print("파일이 없습니다")
finally:
    if file:
        file.close()
        print("파일 닫힘")
```

---

### 전체 구조

```python
try:
    # 예외가 발생할 수 있는 코드
    risky_operation()
except SpecificError:
    # 특정 예외 처리
    handle_error()
except AnotherError as e:
    # 다른 예외 처리
    print(e)
else:
    # 예외가 없을 때
    success_operation()
finally:
    # 항상 실행
    cleanup()
```

---

## 8.4 주요 내장 예외

> 📚 **비유**: 예외는 **에러 사전**입니다.
> - 각 상황마다 고유한 예외 타입

### 1. ValueError - 값 오류

```python
# 잘못된 타입 변환
try:
    number = int("abc")
except ValueError:
    print("숫자로 변환할 수 없습니다")

# 리스트에서 없는 값 제거
try:
    numbers = [1, 2, 3]
    numbers.remove(99)
except ValueError:
    print("리스트에 없는 값입니다")
```

---

### 2. TypeError - 타입 오류

```python
# 잘못된 타입 연산
try:
    result = "10" + 5
except TypeError:
    print("문자열과 숫자를 더할 수 없습니다")

# 잘못된 인수 개수
try:
    def greet(name):
        return f"Hello, {name}"

    greet()  # 인수 누락
except TypeError:
    print("인수가 부족합니다")
```

---

### 3. IndexError - 인덱스 오류

```python
# 범위를 벗어난 인덱스
try:
    numbers = [1, 2, 3]
    print(numbers[10])
except IndexError:
    print("인덱스 범위 초과")
```

---

### 4. KeyError - 키 오류

```python
# 딕셔너리에 없는 키
try:
    user = {"name": "Alice", "age": 25}
    print(user["email"])
except KeyError:
    print("존재하지 않는 키입니다")

# 안전한 방법
email = user.get("email", "없음")  # KeyError 발생 안 함
```

---

### 5. FileNotFoundError - 파일 없음

```python
try:
    with open("nonexistent.txt", "r") as f:
        content = f.read()
except FileNotFoundError:
    print("파일을 찾을 수 없습니다")
```

---

### 6. ZeroDivisionError - 0으로 나눔

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("0으로 나눌 수 없습니다")
```

---

### 7. AttributeError - 속성 없음

```python
try:
    text = "hello"
    text.append("!")  # 문자열에 append 메서드 없음
except AttributeError:
    print("해당 속성이 없습니다")
```

---

### 8. ImportError / ModuleNotFoundError

```python
try:
    import nonexistent_module
except ModuleNotFoundError:
    print("모듈을 찾을 수 없습니다")
```

---

## 8.5 예외 발생시키기 (raise)

> 🚀 **비유**: **수동 경보**
> - 조건에 맞지 않으면 직접 예외 발생

### 기본 사용법

```python
def divide(a, b):
    if b == 0:
        raise ZeroDivisionError("0으로 나눌 수 없습니다")
    return a / b

try:
    result = divide(10, 0)
except ZeroDivisionError as e:
    print(e)  # 0으로 나눌 수 없습니다
```

---

### 입력 검증

```python
def set_age(age):
    """나이 설정 (0~150)"""
    if not isinstance(age, int):
        raise TypeError("나이는 정수여야 합니다")
    if age < 0:
        raise ValueError("나이는 0 이상이어야 합니다")
    if age > 150:
        raise ValueError("나이는 150 이하여야 합니다")
    return age

# 사용
try:
    set_age("25")  # TypeError
except TypeError as e:
    print(e)

try:
    set_age(-5)  # ValueError
except ValueError as e:
    print(e)
```

---

### 예외 재발생

```python
def process_file(filename):
    try:
        with open(filename, "r") as f:
            return f.read()
    except FileNotFoundError:
        print(f"로그: {filename} 파일 없음")
        raise  # 예외 다시 발생

try:
    content = process_file("missing.txt")
except FileNotFoundError:
    print("상위에서 처리")

# 출력:
# 로그: missing.txt 파일 없음
# 상위에서 처리
```

---

## 8.6 사용자 정의 예외

> 🎨 **비유**: **맞춤 경보**
> - 애플리케이션 특화 예외

### 기본 정의

```python
class CustomError(Exception):
    """사용자 정의 예외"""
    pass

def risky_operation():
    raise CustomError("무언가 잘못됨")

try:
    risky_operation()
except CustomError as e:
    print(f"커스텀 에러: {e}")
```

---

### 실전 예제: 인증 예외

```python
class AuthenticationError(Exception):
    """인증 실패 예외"""
    pass

class PermissionError(Exception):
    """권한 부족 예외"""
    pass

def login(username, password):
    """로그인"""
    if username != "admin":
        raise AuthenticationError("사용자를 찾을 수 없습니다")
    if password != "1234":
        raise AuthenticationError("비밀번호가 틀렸습니다")
    return f"{username} 로그인 성공"

def delete_user(user, is_admin):
    """사용자 삭제"""
    if not is_admin:
        raise PermissionError("관리자만 삭제할 수 있습니다")
    return f"{user} 삭제됨"

# 사용
try:
    login("alice", "wrong")
except AuthenticationError as e:
    print(f"로그인 실패: {e}")

try:
    delete_user("bob", is_admin=False)
except PermissionError as e:
    print(f"권한 오류: {e}")
```

---

### 추가 정보 포함

```python
class ValidationError(Exception):
    """데이터 검증 예외"""

    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

def validate_user(data):
    """사용자 데이터 검증"""
    if "email" not in data:
        raise ValidationError("email", "이메일은 필수입니다")

    if "@" not in data["email"]:
        raise ValidationError("email", "유효하지 않은 이메일")

    if len(data.get("password", "")) < 8:
        raise ValidationError("password", "비밀번호는 8자 이상")

# 사용
try:
    validate_user({"email": "invalid", "password": "123"})
except ValidationError as e:
    print(f"검증 실패")
    print(f"  필드: {e.field}")
    print(f"  메시지: {e.message}")

# 출력:
# 검증 실패
#   필드: email
#   메시지: 유효하지 않은 이메일
```

---

## 8.7 컨텍스트 매니저 (with문)

> 🚪 **비유**: **자동문**
> - 들어갈 때: 자동으로 열림 (리소스 획득)
> - 나올 때: 자동으로 닫힘 (리소스 해제)

### 파일 처리

**❌ 수동 관리** (위험):
```python
file = open("data.txt", "r")
try:
    content = file.read()
    print(content)
finally:
    file.close()  # 잊어버리면 메모리 누수!
```

**✅ with문 사용** (안전):
```python
with open("data.txt", "r") as file:
    content = file.read()
    print(content)
# 자동으로 file.close() 호출됨
```

---

### 여러 리소스

```python
with open("input.txt", "r") as infile, \
     open("output.txt", "w") as outfile:
    content = infile.read()
    outfile.write(content.upper())
# 둘 다 자동으로 닫힘
```

---

### 사용자 정의 컨텍스트 매니저

#### 클래스 기반

```python
class Timer:
    """시간 측정 컨텍스트 매니저"""

    def __enter__(self):
        """진입 시"""
        import time
        self.start = time.time()
        print("타이머 시작")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """종료 시"""
        import time
        elapsed = time.time() - self.start
        print(f"걸린 시간: {elapsed:.2f}초")
        return False  # 예외를 다시 발생시킴

# 사용
with Timer():
    # 시간 측정할 코드
    total = sum(range(1000000))
    print(f"합계: {total}")

# 출력:
# 타이머 시작
# 합계: 499999500000
# 걸린 시간: 0.03초
```

---

#### 함수 기반 (contextlib)

```python
from contextlib import contextmanager

@contextmanager
def timer():
    """시간 측정"""
    import time
    start = time.time()
    print("타이머 시작")

    try:
        yield  # 이 지점에서 with 블록 실행
    finally:
        elapsed = time.time() - start
        print(f"걸린 시간: {elapsed:.2f}초")

# 사용
with timer():
    total = sum(range(1000000))
```

---

### 데이터베이스 연결 예제

```python
@contextmanager
def database_connection(host, port):
    """데이터베이스 연결 관리"""
    print(f"연결 중: {host}:{port}")
    conn = f"Connection to {host}:{port}"  # 실제로는 DB 연결 객체

    try:
        yield conn  # 연결 객체 반환
    finally:
        print("연결 종료")

# 사용
with database_connection("localhost", 5432) as conn:
    print(f"쿼리 실행: {conn}")
    # 쿼리 실행...

# 출력:
# 연결 중: localhost:5432
# 쿼리 실행: Connection to localhost:5432
# 연결 종료
```

---

## 8.8 실전 예제

### 예제 1: 안전한 파일 처리

```python
def safe_read_file(filename, default=""):
    """
    안전한 파일 읽기

    Args:
        filename: 파일 경로
        default: 실패 시 기본값

    Returns:
        파일 내용 또는 기본값
    """
    try:
        with open(filename, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        print(f"경고: {filename} 파일을 찾을 수 없습니다")
        return default
    except PermissionError:
        print(f"경고: {filename} 읽기 권한이 없습니다")
        return default
    except Exception as e:
        print(f"에러: {e}")
        return default

# 사용
content = safe_read_file("config.txt", default="기본 설정")
print(content)
```

---

### 예제 2: API 요청 재시도

```python
import time

def retry_request(url, max_retries=3, delay=1):
    """
    API 요청 재시도

    Args:
        url: 요청 URL
        max_retries: 최대 재시도 횟수
        delay: 재시도 간격 (초)

    Returns:
        응답 데이터

    Raises:
        Exception: 모든 재시도 실패 시
    """
    for attempt in range(max_retries):
        try:
            # 실제로는 requests.get(url)
            print(f"시도 {attempt + 1}/{max_retries}: {url}")

            # 시뮬레이션: 처음 2번 실패, 3번째 성공
            if attempt < 2:
                raise ConnectionError("연결 실패")

            return {"status": "success", "data": "결과"}

        except ConnectionError as e:
            print(f"  실패: {e}")

            if attempt < max_retries - 1:
                print(f"  {delay}초 후 재시도...")
                time.sleep(delay)
            else:
                print("  모든 재시도 실패")
                raise

# 사용
try:
    result = retry_request("https://api.example.com/data")
    print(f"성공: {result}")
except Exception as e:
    print(f"최종 실패: {e}")

# 출력:
# 시도 1/3: https://api.example.com/data
#   실패: 연결 실패
#   1초 후 재시도...
# 시도 2/3: https://api.example.com/data
#   실패: 연결 실패
#   1초 후 재시도...
# 시도 3/3: https://api.example.com/data
# 성공: {'status': 'success', 'data': '결과'}
```

---

### 예제 3: 입력 검증 시스템

```python
class InputValidationError(Exception):
    """입력 검증 예외"""
    def __init__(self, field, value, reason):
        self.field = field
        self.value = value
        self.reason = reason
        super().__init__(f"{field} 검증 실패: {reason} (입력값: {value})")

class UserInput:
    """사용자 입력 검증"""

    @staticmethod
    def get_int(prompt, min_val=None, max_val=None):
        """정수 입력"""
        while True:
            try:
                value = int(input(prompt))

                if min_val is not None and value < min_val:
                    raise InputValidationError(
                        "정수", value, f"{min_val} 이상이어야 합니다"
                    )

                if max_val is not None and value > max_val:
                    raise InputValidationError(
                        "정수", value, f"{max_val} 이하여야 합니다"
                    )

                return value

            except ValueError:
                print("⚠️ 정수를 입력하세요")
            except InputValidationError as e:
                print(f"⚠️ {e.reason}")

    @staticmethod
    def get_email(prompt):
        """이메일 입력"""
        while True:
            email = input(prompt)

            if "@" not in email:
                print("⚠️ '@'가 포함되어야 합니다")
                continue

            if "." not in email.split("@")[1]:
                print("⚠️ 도메인에 '.'이 포함되어야 합니다")
                continue

            return email

# 사용
age = UserInput.get_int("나이 (0-150): ", min_val=0, max_val=150)
print(f"입력된 나이: {age}")

email = UserInput.get_email("이메일: ")
print(f"입력된 이메일: {email}")
```

---

### 예제 4: 트랜잭션 관리

```python
class TransactionError(Exception):
    """트랜잭션 예외"""
    pass

class BankAccount:
    """은행 계좌"""

    def __init__(self, name, balance=0):
        self.name = name
        self.balance = balance

    def deposit(self, amount):
        """입금"""
        if amount <= 0:
            raise ValueError("입금액은 0보다 커야 합니다")
        self.balance += amount

    def withdraw(self, amount):
        """출금"""
        if amount <= 0:
            raise ValueError("출금액은 0보다 커야 합니다")
        if amount > self.balance:
            raise TransactionError("잔액이 부족합니다")
        self.balance -= amount

    def __str__(self):
        return f"{self.name}: {self.balance:,}원"

def transfer(from_account, to_account, amount):
    """
    계좌 이체

    Args:
        from_account: 출금 계좌
        to_account: 입금 계좌
        amount: 이체 금액

    Raises:
        TransactionError: 이체 실패 시
    """
    print(f"\n이체 시작: {amount:,}원")
    print(f"  출금: {from_account}")
    print(f"  입금: {to_account}")

    # 원래 잔액 저장 (롤백용)
    original_from = from_account.balance
    original_to = to_account.balance

    try:
        # 출금
        from_account.withdraw(amount)
        print(f"  ✓ 출금 완료")

        # 입금
        to_account.deposit(amount)
        print(f"  ✓ 입금 완료")

        print("✅ 이체 성공")

    except (ValueError, TransactionError) as e:
        # 롤백
        from_account.balance = original_from
        to_account.balance = original_to
        print(f"❌ 이체 실패: {e}")
        print("  (원래 상태로 롤백됨)")
        raise

# 사용
alice = BankAccount("Alice", 100000)
bob = BankAccount("Bob", 50000)

print("초기 상태:")
print(f"  {alice}")
print(f"  {bob}")

# 정상 이체
try:
    transfer(alice, bob, 30000)
except TransactionError:
    pass

print("\n이체 후:")
print(f"  {alice}")
print(f"  {bob}")

# 실패 이체 (잔액 부족)
try:
    transfer(alice, bob, 100000)
except TransactionError:
    pass

print("\n최종 상태:")
print(f"  {alice}")
print(f"  {bob}")

# 출력:
# 초기 상태:
#   Alice: 100,000원
#   Bob: 50,000원
#
# 이체 시작: 30,000원
#   출금: Alice: 100,000원
#   입금: Bob: 50,000원
#   ✓ 출금 완료
#   ✓ 입금 완료
# ✅ 이체 성공
#
# 이체 후:
#   Alice: 70,000원
#   Bob: 80,000원
#
# 이체 시작: 100,000원
#   출금: Alice: 70,000원
#   입금: Bob: 80,000원
# ❌ 이체 실패: 잔액이 부족합니다
#   (원래 상태로 롤백됨)
#
# 최종 상태:
#   Alice: 70,000원
#   Bob: 80,000원
```

---

## 실전 팁

### 💡 Tip 1: 구체적인 예외 먼저

**❌ 나쁜 예**:
```python
try:
    # ...
except Exception:  # 너무 광범위
    pass
except ValueError:  # 도달 불가능
    pass
```

**✅ 좋은 예**:
```python
try:
    # ...
except ValueError:      # 구체적인 것 먼저
    pass
except TypeError:
    pass
except Exception as e:  # 마지막에
    print(f"예상치 못한 에러: {e}")
```

---

### 💡 Tip 2: 예외 무시하지 않기

**❌ 나쁜 예**:
```python
try:
    risky_operation()
except:
    pass  # 에러를 숨김 (디버깅 불가능)
```

**✅ 좋은 예**:
```python
try:
    risky_operation()
except SpecificError as e:
    logger.error(f"에러 발생: {e}")  # 로그 남김
    # 또는 적절한 처리
```

---

### 💡 Tip 3: EAFP vs LBYL

**LBYL** (Look Before You Leap - 도약 전 확인):
```python
# Java 스타일
if key in dictionary:
    value = dictionary[key]
else:
    value = default
```

**EAFP** (Easier to Ask Forgiveness than Permission - Python 스타일):
```python
# Python 스타일 (권장)
try:
    value = dictionary[key]
except KeyError:
    value = default
```

> 💡 **Python 철학**: "허락보다 용서가 쉽다"
> - EAFP가 더 Pythonic
> - 성능도 더 좋음 (대부분의 경우)

---

## 연습 문제

### 문제 1: 안전한 리스트 접근

리스트에서 인덱스로 값을 안전하게 가져오는 함수 작성:
- 인덱스가 범위 내: 값 반환
- 인덱스가 범위 밖: 기본값 반환

<details>
<summary>정답 보기</summary>

```python
def safe_get(lst, index, default=None):
    """
    안전한 리스트 접근

    Args:
        lst: 리스트
        index: 인덱스
        default: 기본값

    Returns:
        값 또는 기본값
    """
    try:
        return lst[index]
    except IndexError:
        return default

# 테스트
numbers = [1, 2, 3]
print(safe_get(numbers, 0))      # 1
print(safe_get(numbers, 10))     # None
print(safe_get(numbers, 10, 0))  # 0
```
</details>

---

### 문제 2: 나이 검증 함수

나이를 검증하는 함수 작성:
- 정수가 아니면 TypeError
- 0~150 범위 밖이면 ValueError

<details>
<summary>정답 보기</summary>

```python
def validate_age(age):
    """
    나이 검증

    Args:
        age: 나이

    Returns:
        검증된 나이

    Raises:
        TypeError: 정수가 아닐 때
        ValueError: 범위 밖일 때
    """
    if not isinstance(age, int):
        raise TypeError(f"나이는 정수여야 합니다: {type(age)}")

    if age < 0 or age > 150:
        raise ValueError(f"나이는 0~150 사이여야 합니다: {age}")

    return age

# 테스트
try:
    validate_age("25")
except TypeError as e:
    print(e)  # 나이는 정수여야 합니다

try:
    validate_age(-5)
except ValueError as e:
    print(e)  # 나이는 0~150 사이여야 합니다

print(validate_age(25))  # 25
```
</details>

---

### 문제 3: JSON 파일 읽기

JSON 파일을 읽되, 여러 예외 상황 처리:
- 파일 없음
- JSON 파싱 오류
- 기타 오류

<details>
<summary>정답 보기</summary>

```python
import json

def load_json(filename, default=None):
    """
    JSON 파일 로드

    Args:
        filename: 파일 경로
        default: 실패 시 기본값

    Returns:
        JSON 데이터 또는 기본값
    """
    try:
        with open(filename, "r", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        print(f"파일을 찾을 수 없습니다: {filename}")
        return default
    except json.JSONDecodeError as e:
        print(f"JSON 파싱 오류: {e}")
        return default
    except Exception as e:
        print(f"알 수 없는 오류: {e}")
        return default

# 테스트
data = load_json("config.json", default={})
print(data)
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **기본 구조**
   ```python
   try:
       # 시도
   except SpecificError:
       # 예외 처리
   else:
       # 성공 시
   finally:
       # 항상 실행
   ```

2. **주요 예외**
   - `ValueError`: 값 오류
   - `TypeError`: 타입 오류
   - `KeyError`: 키 없음
   - `IndexError`: 인덱스 범위 초과
   - `FileNotFoundError`: 파일 없음

3. **예외 발생**
   ```python
   raise ValueError("에러 메시지")
   ```

4. **사용자 정의 예외**
   ```python
   class CustomError(Exception):
       pass
   ```

5. **컨텍스트 매니저**
   ```python
   with open("file.txt") as f:
       content = f.read()
   ```

6. **Python 스타일**: EAFP (허락보다 용서)

---

## Part 3 완료!

Part 3 (함수와 모듈)을 마쳤습니다:
- ✅ Chapter 6: 함수
- ✅ Chapter 7: 모듈과 패키지
- ✅ Chapter 8: 예외 처리

---

## 다음 챕터 예고

Chapter 9에서는 **객체지향 프로그래밍 (OOP)**을 다룹니다:
- 클래스와 객체
- 속성과 메서드
- 생성자와 소멸자
- 접근 제어

---

[다음: Chapter 9. 클래스와 객체 →](../part4-oop/chapter9-classes.md)