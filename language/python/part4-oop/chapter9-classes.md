# Chapter 9. 클래스와 객체 (Classes & Objects)

## 9.1 객체지향 프로그래밍이란?

> 🏗️ **비유**: OOP는 **레고 블록**입니다.
> - 클래스: 블록 설계도 (청사진)
> - 객체: 실제 만든 블록 (인스턴스)
> - 속성: 블록의 색깔, 크기 (데이터)
> - 메서드: 블록을 조립하는 방법 (동작)

**객체지향 프로그래밍 (OOP)**: 프로그램을 **객체들의 모음**으로 설계하는 패러다임

---

### Python의 OOP 철학

> 💡 **"Everything is an object"** (모든 것이 객체다)

```python
# 숫자도 객체
x = 5
print(type(x))  # <class 'int'>
print(x.bit_length())  # 메서드 호출 가능!

# 문자열도 객체
text = "hello"
print(type(text))  # <class 'str'>
print(text.upper())  # HELLO

# 함수도 객체
def greet():
    return "Hello"

print(type(greet))  # <class 'function'>
print(greet.__name__)  # greet
```

---

## 9.2 클래스 정의

### 기본 문법

```python
class ClassName:
    """클래스 설명 (Docstring)"""

    # 클래스 변수
    class_variable = "공유 데이터"

    # 생성자
    def __init__(self, param):
        self.instance_variable = param  # 인스턴스 변수

    # 메서드
    def method(self):
        return self.instance_variable
```

---

### Java vs Python 비교

**Java**:
```java
public class Person {
    // 필드
    private String name;
    private int age;

    // 생성자
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // 메서드
    public String introduce() {
        return "안녕하세요, " + name + "입니다.";
    }
}

// 사용
Person person = new Person("Alice", 25);
```

**Python**:
```python
class Person:
    # 생성자
    def __init__(self, name, age):
        self.name = name
        self.age = age

    # 메서드
    def introduce(self):
        return f"안녕하세요, {self.name}입니다."

# 사용
person = Person("Alice", 25)
```

> 💡 **차이점**:
> - Python: `self`를 명시적으로 첫 번째 매개변수로
> - Java: `this`는 암묵적
> - Python: 생성자는 `__init__`
> - Java: 생성자는 클래스명과 동일

---

## 9.3 생성자 (__init__)

> 🏭 **비유**: 생성자는 **공장의 조립 라인**입니다.
> - 부품(매개변수) 받음
> - 제품(객체) 조립
> - 완성품 출하

```python
class Person:
    def __init__(self, name, age):
        """
        Person 클래스 생성자

        Args:
            name: 이름
            age: 나이
        """
        print(f"{name} 객체 생성 중...")
        self.name = name
        self.age = age

# 객체 생성
person = Person("Alice", 25)
# 출력: Alice 객체 생성 중...

print(person.name)  # Alice
print(person.age)   # 25
```

---

### 기본값 있는 생성자

```python
class Product:
    def __init__(self, name, price, quantity=1):
        self.name = name
        self.price = price
        self.quantity = quantity

    def total_price(self):
        return self.price * self.quantity

# 다양한 방식으로 생성
laptop = Product("노트북", 1000000)
mouse = Product("마우스", 30000, quantity=2)

print(laptop.total_price())  # 1000000
print(mouse.total_price())   # 60000
```

---

## 9.4 인스턴스 변수 vs 클래스 변수

> 📦 **비유**:
> - **인스턴스 변수**: 개인 사물함 (각자 다름)
> - **클래스 변수**: 공용 게시판 (모두 공유)

### 인스턴스 변수

```python
class Student:
    def __init__(self, name, student_id):
        self.name = name           # 인스턴스 변수
        self.student_id = student_id

alice = Student("Alice", "2024001")
bob = Student("Bob", "2024002")

print(alice.name)      # Alice
print(bob.name)        # Bob
# 각자 다른 값!
```

---

### 클래스 변수

```python
class Student:
    school = "Python 고등학교"  # 클래스 변수 (모두 공유)

    def __init__(self, name, student_id):
        self.name = name
        self.student_id = student_id

alice = Student("Alice", "2024001")
bob = Student("Bob", "2024002")

print(alice.school)  # Python 고등학교
print(bob.school)    # Python 고등학교

# 클래스 변수 변경
Student.school = "Python 대학교"

print(alice.school)  # Python 대학교 (모두 변경됨!)
print(bob.school)    # Python 대학교
```

---

### 실전 예제: 객체 카운팅

```python
class User:
    total_users = 0  # 클래스 변수 (전체 사용자 수)

    def __init__(self, username):
        self.username = username      # 인스턴스 변수
        User.total_users += 1         # 생성 시마다 증가
        self.user_number = User.total_users

    def __del__(self):
        """소멸자 (객체 삭제 시 호출)"""
        User.total_users -= 1

# 사용
alice = User("alice")
print(f"전체 사용자: {User.total_users}")  # 1

bob = User("bob")
charlie = User("charlie")
print(f"전체 사용자: {User.total_users}")  # 3

print(f"{alice.username}는 {alice.user_number}번째 사용자")
# alice는 1번째 사용자
```

---

## 9.5 메서드 (Methods)

> 🔧 **비유**: 메서드는 **도구**입니다.
> - 객체의 데이터를 조작
> - 특정 기능 수행

### 인스턴스 메서드

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        self.owner = owner
        self.balance = balance

    def deposit(self, amount):
        """입금"""
        if amount > 0:
            self.balance += amount
            return f"{amount:,}원 입금 완료. 잔액: {self.balance:,}원"
        return "입금액은 0보다 커야 합니다"

    def withdraw(self, amount):
        """출금"""
        if amount > self.balance:
            return "잔액이 부족합니다"
        if amount > 0:
            self.balance -= amount
            return f"{amount:,}원 출금 완료. 잔액: {self.balance:,}원"
        return "출금액은 0보다 커야 합니다"

    def get_balance(self):
        """잔액 조회"""
        return f"{self.owner}님의 잔액: {self.balance:,}원"

# 사용
account = BankAccount("Alice", 100000)
print(account.deposit(50000))   # 50,000원 입금 완료. 잔액: 150,000원
print(account.withdraw(30000))  # 30,000원 출금 완료. 잔액: 120,000원
print(account.get_balance())    # Alice님의 잔액: 120,000원
```

---

### 클래스 메서드 (@classmethod)

> 🏢 **비유**: **회사 차원의 업무**
> - 개인(인스턴스)이 아닌 회사(클래스) 전체 관련

```python
class Employee:
    company = "Python Corp"
    employee_count = 0

    def __init__(self, name, position):
        self.name = name
        self.position = position
        Employee.employee_count += 1

    @classmethod
    def get_employee_count(cls):
        """전체 직원 수 조회 (클래스 메서드)"""
        return f"{cls.company}의 직원 수: {cls.employee_count}명"

    @classmethod
    def change_company_name(cls, new_name):
        """회사명 변경 (클래스 메서드)"""
        cls.company = new_name

# 사용
alice = Employee("Alice", "Developer")
bob = Employee("Bob", "Designer")

# 인스턴스 없이도 호출 가능
print(Employee.get_employee_count())  # Python Corp의 직원 수: 2명

# 클래스 메서드로 클래스 변수 변경
Employee.change_company_name("Python Corporation")
print(Employee.company)  # Python Corporation
```

---

### 정적 메서드 (@staticmethod)

> 🔨 **비유**: **독립적인 유틸리티 도구**
> - 클래스/인스턴스와 무관
> - 네임스페이스 정리용

```python
class MathUtils:
    """수학 유틸리티"""

    @staticmethod
    def add(a, b):
        """덧셈 (정적 메서드)"""
        return a + b

    @staticmethod
    def is_even(n):
        """짝수 판별"""
        return n % 2 == 0

    @staticmethod
    def factorial(n):
        """팩토리얼"""
        if n <= 1:
            return 1
        return n * MathUtils.factorial(n - 1)

# 인스턴스 생성 없이 사용
print(MathUtils.add(10, 5))      # 15
print(MathUtils.is_even(10))     # True
print(MathUtils.factorial(5))    # 120
```

---

### 메서드 타입 비교

```python
class MyClass:
    class_var = "클래스 변수"

    def __init__(self, value):
        self.instance_var = value

    # 1. 인스턴스 메서드 (self)
    def instance_method(self):
        return f"인스턴스: {self.instance_var}"

    # 2. 클래스 메서드 (cls)
    @classmethod
    def class_method(cls):
        return f"클래스: {cls.class_var}"

    # 3. 정적 메서드 (매개변수 없음)
    @staticmethod
    def static_method():
        return "정적 메서드"

# 사용
obj = MyClass("값")

print(obj.instance_method())     # 인스턴스: 값
print(MyClass.class_method())    # 클래스: 클래스 변수
print(MyClass.static_method())   # 정적 메서드
```

| 메서드 타입 | 첫 매개변수 | 접근 가능 | 주 용도 |
|------------|-----------|---------|--------|
| 인스턴스 | `self` | 인스턴스 변수, 클래스 변수 | 객체별 동작 |
| 클래스 | `cls` | 클래스 변수 | 클래스 전체 관련 |
| 정적 | 없음 | 둘 다 불가 | 독립적 유틸리티 |

---

## 9.6 매직 메서드 (Magic Methods)

> ✨ **비유**: **마법 주문**
> - `__메서드명__` 형태
> - Python이 특정 상황에서 자동 호출
> - 연산자 오버로딩 등

### `__init__`: 생성자

```python
class Person:
    def __init__(self, name):
        print(f"{name} 객체 생성!")
        self.name = name

person = Person("Alice")  # Alice 객체 생성!
```

---

### `__str__`: 문자열 표현 (사용자용)

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self):
        """str(obj) 또는 print(obj) 시 호출"""
        return f"{self.name} ({self.age}세)"

person = Person("Alice", 25)
print(person)        # Alice (25세)
print(str(person))   # Alice (25세)
```

---

### `__repr__`: 문자열 표현 (개발자용)

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        """repr(obj) 호출 시 또는 디버깅 시"""
        return f"Person(name='{self.name}', age={self.age})"

    def __str__(self):
        return f"{self.name} ({self.age}세)"

person = Person("Alice", 25)
print(person)        # Alice (25세) - __str__
print(repr(person))  # Person(name='Alice', age=25) - __repr__

# 리스트에 넣으면 __repr__ 사용
people = [Person("Alice", 25), Person("Bob", 30)]
print(people)
# [Person(name='Alice', age=25), Person(name='Bob', age=30)]
```

---

### `__len__`: 길이

```python
class Playlist:
    def __init__(self):
        self.songs = []

    def add_song(self, song):
        self.songs.append(song)

    def __len__(self):
        """len(obj) 호출 시"""
        return len(self.songs)

playlist = Playlist()
playlist.add_song("Song 1")
playlist.add_song("Song 2")
print(len(playlist))  # 2
```

---

### `__getitem__`, `__setitem__`: 인덱싱

```python
class CustomList:
    def __init__(self):
        self.items = []

    def __getitem__(self, index):
        """obj[index] 읽기"""
        return self.items[index]

    def __setitem__(self, index, value):
        """obj[index] = value 쓰기"""
        self.items[index] = value

    def append(self, item):
        self.items.append(item)

# 사용
my_list = CustomList()
my_list.append("A")
my_list.append("B")
my_list.append("C")

print(my_list[0])   # A - __getitem__ 호출
my_list[1] = "Z"    # __setitem__ 호출
print(my_list[1])   # Z
```

---

### 연산자 오버로딩

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        """+ 연산자"""
        return Point(self.x + other.x, self.y + other.y)

    def __sub__(self, other):
        """- 연산자"""
        return Point(self.x - other.x, self.y - other.y)

    def __mul__(self, scalar):
        """* 연산자 (스칼라 곱)"""
        return Point(self.x * scalar, self.y * scalar)

    def __eq__(self, other):
        """== 연산자"""
        return self.x == other.x and self.y == other.y

    def __str__(self):
        return f"Point({self.x}, {self.y})"

# 사용
p1 = Point(1, 2)
p2 = Point(3, 4)

p3 = p1 + p2        # __add__
print(p3)           # Point(4, 6)

p4 = p2 - p1        # __sub__
print(p4)           # Point(2, 2)

p5 = p1 * 3         # __mul__
print(p5)           # Point(3, 6)

print(p1 == p2)     # False - __eq__
print(p1 == Point(1, 2))  # True
```

---

### 주요 매직 메서드

| 메서드 | 용도 | 예시 |
|-------|------|------|
| `__init__` | 생성자 | `obj = MyClass()` |
| `__str__` | 문자열 표현 (사용자) | `print(obj)` |
| `__repr__` | 문자열 표현 (개발자) | `repr(obj)` |
| `__len__` | 길이 | `len(obj)` |
| `__getitem__` | 인덱싱 읽기 | `obj[key]` |
| `__setitem__` | 인덱싱 쓰기 | `obj[key] = value` |
| `__add__` | `+` 연산자 | `obj1 + obj2` |
| `__sub__` | `-` 연산자 | `obj1 - obj2` |
| `__mul__` | `*` 연산자 | `obj * n` |
| `__eq__` | `==` 연산자 | `obj1 == obj2` |
| `__lt__` | `<` 연산자 | `obj1 < obj2` |
| `__call__` | 함수처럼 호출 | `obj()` |

---

## 9.7 프로퍼티 (Property)

> 🎭 **비유**: 프로퍼티는 **변장한 메서드**입니다.
> - 겉으로는 변수처럼 보임
> - 실제로는 메서드 호출

### Java의 Getter/Setter

**Java**:
```java
public class Person {
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        if (age < 0) {
            throw new IllegalArgumentException("나이는 0 이상");
        }
        this.age = age;
    }
}

// 사용
person.setAge(25);
int age = person.getAge();
```

---

### Python의 Property

**방법 1: @property 데코레이터 (권장)**
```python
class Person:
    def __init__(self, name, age):
        self._name = name  # _는 내부 변수 관례
        self._age = age

    @property
    def age(self):
        """Getter: 읽기"""
        print("age getter 호출")
        return self._age

    @age.setter
    def age(self, value):
        """Setter: 쓰기"""
        print("age setter 호출")
        if value < 0:
            raise ValueError("나이는 0 이상이어야 합니다")
        self._age = value

    @property
    def name(self):
        """읽기 전용 (setter 없음)"""
        return self._name

# 사용 (변수처럼!)
person = Person("Alice", 25)

# Getter
print(person.age)     # age getter 호출 → 25

# Setter
person.age = 30       # age setter 호출

# 읽기 전용
print(person.name)    # Alice
# person.name = "Bob"  # AttributeError! (setter 없음)
```

---

### 계산된 프로퍼티

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    @property
    def area(self):
        """넓이 (계산된 값)"""
        return self.width * self.height

    @property
    def perimeter(self):
        """둘레 (계산된 값)"""
        return 2 * (self.width + self.height)

# 사용
rect = Rectangle(10, 5)
print(rect.area)       # 50 (메서드지만 변수처럼 접근)
print(rect.perimeter)  # 30

# 값 변경 시 자동 재계산
rect.width = 20
print(rect.area)       # 100 (자동 업데이트)
```

---

### 실전 예제: 온도 변환

```python
class Temperature:
    def __init__(self, celsius=0):
        self._celsius = celsius

    @property
    def celsius(self):
        """섭씨 (읽기/쓰기)"""
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("절대영도 이하는 불가능합니다")
        self._celsius = value

    @property
    def fahrenheit(self):
        """화씨 (계산됨)"""
        return self._celsius * 9/5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value):
        self._celsius = (value - 32) * 5/9

    @property
    def kelvin(self):
        """켈빈 (계산됨)"""
        return self._celsius + 273.15

    def __str__(self):
        return f"{self.celsius}°C = {self.fahrenheit}°F = {self.kelvin}K"

# 사용
temp = Temperature(25)
print(temp)  # 25°C = 77.0°F = 298.15K

# 섭씨로 설정
temp.celsius = 100
print(temp)  # 100°C = 212.0°F = 373.15K

# 화씨로 설정 (자동으로 섭씨 변환)
temp.fahrenheit = 32
print(temp)  # 0.0°C = 32.0°F = 273.15K
```

---

## 9.8 실전 예제

### 예제 1: 도서 관리 시스템

```python
class Book:
    """도서 클래스"""

    total_books = 0  # 클래스 변수

    def __init__(self, title, author, isbn, price):
        self.title = title
        self.author = author
        self.isbn = isbn
        self._price = price
        self.is_available = True
        Book.total_books += 1

    @property
    def price(self):
        """가격 (읽기/쓰기)"""
        return self._price

    @price.setter
    def price(self, value):
        if value < 0:
            raise ValueError("가격은 0 이상이어야 합니다")
        self._price = value

    def borrow(self):
        """대출"""
        if not self.is_available:
            return f"'{self.title}'은 이미 대출 중입니다"
        self.is_available = False
        return f"'{self.title}' 대출 완료"

    def return_book(self):
        """반납"""
        if self.is_available:
            return f"'{self.title}'은 대출 중이 아닙니다"
        self.is_available = True
        return f"'{self.title}' 반납 완료"

    @classmethod
    def get_total_books(cls):
        """전체 도서 수"""
        return f"전체 도서: {cls.total_books}권"

    def __str__(self):
        status = "대출 가능" if self.is_available else "대출 중"
        return f"[{self.title}] {self.author} - {self.price:,}원 ({status})"

    def __repr__(self):
        return f"Book('{self.title}', '{self.author}', '{self.isbn}')"


class Library:
    """도서관 클래스"""

    def __init__(self, name):
        self.name = name
        self.books = []

    def add_book(self, book):
        """도서 추가"""
        self.books.append(book)
        return f"'{book.title}' 추가 완료"

    def find_by_title(self, title):
        """제목으로 검색"""
        results = [book for book in self.books if title.lower() in book.title.lower()]
        return results if results else None

    def find_available(self):
        """대출 가능한 도서"""
        return [book for book in self.books if book.is_available]

    def __len__(self):
        return len(self.books)

    def __str__(self):
        return f"{self.name} (도서 {len(self)}권)"


# 사용
library = Library("Python 도서관")

# 도서 추가
book1 = Book("파이썬 완전정복", "Alice", "123-456", 30000)
book2 = Book("클린 코드", "Bob", "789-012", 25000)
book3 = Book("파이썬 쿡북", "Charlie", "345-678", 35000)

print(library.add_book(book1))
print(library.add_book(book2))
print(library.add_book(book3))

print(f"\n{library}")
print(Book.get_total_books())

# 도서 검색
print("\n=== '파이썬' 검색 ===")
results = library.find_by_title("파이썬")
for book in results:
    print(book)

# 대출
print(f"\n{book1.borrow()}")
print(f"{book1.borrow()}")  # 이미 대출 중

# 대출 가능한 도서
print("\n=== 대출 가능한 도서 ===")
for book in library.find_available():
    print(book)

# 반납
print(f"\n{book1.return_book()}")
print(book1)

# 출력:
# '파이썬 완전정복' 추가 완료
# '클린 코드' 추가 완료
# '파이썬 쿡북' 추가 완료
#
# Python 도서관 (도서 3권)
# 전체 도서: 3권
#
# === '파이썬' 검색 ===
# [파이썬 완전정복] Alice - 30,000원 (대출 가능)
# [파이썬 쿡북] Charlie - 35,000원 (대출 가능)
#
# '파이썬 완전정복' 대출 완료
# '파이썬 완전정복'은 이미 대출 중입니다
#
# === 대출 가능한 도서 ===
# [클린 코드] Bob - 25,000원 (대출 가능)
# [파이썬 쿡북] Charlie - 35,000원 (대출 가능)
#
# '파이썬 완전정복' 반납 완료
# [파이썬 완전정복] Alice - 30,000원 (대출 가능)
```

---

### 예제 2: 쇼핑 카트

```python
class Product:
    """상품 클래스"""

    def __init__(self, name, price, stock):
        self.name = name
        self._price = price
        self._stock = stock

    @property
    def price(self):
        return self._price

    @price.setter
    def price(self, value):
        if value < 0:
            raise ValueError("가격은 0 이상이어야 합니다")
        self._price = value

    @property
    def stock(self):
        return self._stock

    def decrease_stock(self, quantity):
        """재고 감소"""
        if quantity > self._stock:
            raise ValueError(f"재고 부족 (현재: {self._stock})")
        self._stock -= quantity

    def increase_stock(self, quantity):
        """재고 증가"""
        self._stock += quantity

    def __str__(self):
        return f"{self.name} - {self.price:,}원 (재고: {self.stock})"


class CartItem:
    """장바구니 아이템"""

    def __init__(self, product, quantity):
        self.product = product
        self._quantity = quantity

    @property
    def quantity(self):
        return self._quantity

    @quantity.setter
    def quantity(self, value):
        if value <= 0:
            raise ValueError("수량은 1 이상이어야 합니다")
        if value > self.product.stock:
            raise ValueError(f"재고 부족 (최대: {self.product.stock})")
        self._quantity = value

    @property
    def subtotal(self):
        """소계"""
        return self.product.price * self.quantity

    def __str__(self):
        return f"{self.product.name} x {self.quantity} = {self.subtotal:,}원"


class ShoppingCart:
    """쇼핑 카트"""

    def __init__(self):
        self.items = []

    def add_item(self, product, quantity=1):
        """상품 추가"""
        if quantity > product.stock:
            return f"재고 부족 (현재: {product.stock})"

        # 이미 있는 상품이면 수량 증가
        for item in self.items:
            if item.product == product:
                item.quantity += quantity
                return f"{product.name} 수량 증가: {item.quantity}"

        # 새 상품 추가
        self.items.append(CartItem(product, quantity))
        return f"{product.name} 추가 완료"

    def remove_item(self, product):
        """상품 제거"""
        for item in self.items:
            if item.product == product:
                self.items.remove(item)
                return f"{product.name} 제거 완료"
        return "상품을 찾을 수 없습니다"

    @property
    def total(self):
        """총액"""
        return sum(item.subtotal for item in self.items)

    def checkout(self):
        """결제"""
        if not self.items:
            return "장바구니가 비어있습니다"

        # 재고 확인 및 감소
        for item in self.items:
            item.product.decrease_stock(item.quantity)

        total = self.total
        self.items.clear()
        return f"결제 완료: {total:,}원"

    def __len__(self):
        return len(self.items)

    def __str__(self):
        if not self.items:
            return "장바구니가 비어있습니다"

        result = ["=== 장바구니 ==="]
        for item in self.items:
            result.append(str(item))
        result.append(f"총액: {self.total:,}원")
        return "\n".join(result)


# 사용
laptop = Product("노트북", 1000000, 5)
mouse = Product("마우스", 30000, 10)
keyboard = Product("키보드", 80000, 7)

cart = ShoppingCart()

print(cart.add_item(laptop, 1))
print(cart.add_item(mouse, 2))
print(cart.add_item(keyboard, 1))

print(f"\n{cart}")

print(f"\n상품 수: {len(cart)}개")

print(f"\n{cart.checkout()}")
print(f"\n{cart}")

print("\n=== 재고 확인 ===")
print(laptop)  # 재고 1 감소
print(mouse)   # 재고 2 감소

# 출력:
# 노트북 추가 완료
# 마우스 추가 완료
# 키보드 추가 완료
#
# === 장바구니 ===
# 노트북 x 1 = 1,000,000원
# 마우스 x 2 = 60,000원
# 키보드 x 1 = 80,000원
# 총액: 1,140,000원
#
# 상품 수: 3개
#
# 결제 완료: 1,140,000원
#
# 장바구니가 비어있습니다
#
# === 재고 확인 ===
# 노트북 - 1,000,000원 (재고: 4)
# 마우스 - 30,000원 (재고: 8)
```

---

## 실전 팁

### 💡 Tip 1: 네이밍 컨벤션

```python
class MyClass:
    # 공개 (public)
    public_var = "누구나 접근"

    # 내부 사용 (protected - 관례)
    _protected_var = "서브클래스에서 접근"

    # 비공개 (private - 네임 맹글링)
    __private_var = "외부 접근 불가"

    def __init__(self):
        self.public = "공개"
        self._internal = "내부용"
        self.__private = "비공개"
```

> ⚠️ **주의**: Python은 진짜 private이 없음 (관례일 뿐)

---

### 💡 Tip 2: `__str__` vs `__repr__`

```python
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email

    def __str__(self):
        """사용자에게 보여줄 때"""
        return f"User: {self.username}"

    def __repr__(self):
        """디버깅/개발 시"""
        return f"User(username='{self.username}', email='{self.email}')"
```

> 💡 **규칙**:
> - `__str__`: 읽기 쉽게
> - `__repr__`: 객체 재생성 가능하도록

---

### 💡 Tip 3: 프로퍼티 vs 일반 메서드

**프로퍼티 사용**:
- 계산된 값 (area, total)
- 유효성 검사 필요
- 변수처럼 보이는 게 자연스러울 때

**일반 메서드 사용**:
- 복잡한 로직
- 시간이 오래 걸리는 작업
- 부수 효과가 있을 때

---

## 연습 문제

### 문제 1: Counter 클래스

증가/감소 가능한 카운터 클래스 작성:
- 초기값 설정
- `increment()`, `decrement()` 메서드
- 음수 방지

<details>
<summary>정답 보기</summary>

```python
class Counter:
    """카운터 클래스"""

    def __init__(self, initial=0):
        self._count = initial

    @property
    def count(self):
        return self._count

    def increment(self):
        """증가"""
        self._count += 1

    def decrement(self):
        """감소 (음수 방지)"""
        if self._count > 0:
            self._count -= 1

    def reset(self):
        """초기화"""
        self._count = 0

    def __str__(self):
        return f"Count: {self.count}"

# 테스트
counter = Counter(5)
print(counter)       # Count: 5

counter.increment()
counter.increment()
print(counter)       # Count: 7

counter.decrement()
print(counter)       # Count: 6

counter.reset()
print(counter)       # Count: 0
```
</details>

---

### 문제 2: Student 클래스

학생 클래스 작성:
- 이름, 학번, 성적 리스트
- 성적 추가 메서드
- 평균 계산 (프로퍼티)

<details>
<summary>정답 보기</summary>

```python
class Student:
    """학생 클래스"""

    def __init__(self, name, student_id):
        self.name = name
        self.student_id = student_id
        self.grades = []

    def add_grade(self, grade):
        """성적 추가"""
        if 0 <= grade <= 100:
            self.grades.append(grade)
        else:
            raise ValueError("성적은 0~100 사이여야 합니다")

    @property
    def average(self):
        """평균 점수"""
        if not self.grades:
            return 0
        return sum(self.grades) / len(self.grades)

    @property
    def grade_letter(self):
        """학점"""
        avg = self.average
        if avg >= 90: return "A"
        if avg >= 80: return "B"
        if avg >= 70: return "C"
        if avg >= 60: return "D"
        return "F"

    def __str__(self):
        return f"{self.name} ({self.student_id}): 평균 {self.average:.1f}점 ({self.grade_letter})"

# 테스트
student = Student("Alice", "2024001")
student.add_grade(85)
student.add_grade(90)
student.add_grade(95)

print(student)
# Alice (2024001): 평균 90.0점 (A)
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **클래스 정의**
   ```python
   class MyClass:
       def __init__(self, param):
           self.instance_var = param
   ```

2. **메서드 타입**
   - 인스턴스: `def method(self)`
   - 클래스: `@classmethod def method(cls)`
   - 정적: `@staticmethod def method()`

3. **매직 메서드**
   - `__init__`: 생성자
   - `__str__`: 문자열 표현
   - `__len__`: 길이
   - `__add__`, `__eq__`: 연산자 오버로딩

4. **프로퍼티**
   ```python
   @property
   def name(self):
       return self._name

   @name.setter
   def name(self, value):
       self._name = value
   ```

5. **네이밍**
   - 공개: `public`
   - 내부: `_protected`
   - 비공개: `__private`

---

## 다음 챕터 예고

Chapter 10에서는 **상속과 다형성**을 다룹니다:
- 클래스 상속
- 메서드 오버라이딩
- super()
- 다중 상속

---

[다음: Chapter 10. 상속과 다형성 →](chapter10-inheritance.md)