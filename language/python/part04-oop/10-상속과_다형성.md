# Chapter 10. 상속과 다형성 (Inheritance & Polymorphism)

## 10.1 상속이란?

> 🌳 **비유**: 상속은 **가계도**입니다.
> - 부모 클래스: 조상 (특성 물려줌)
> - 자식 클래스: 후손 (특성 물려받음)
> - 재정의: 물려받은 것을 자기 방식으로 변경

**상속 (Inheritance)**: 기존 클래스의 속성과 메서드를 **물려받아** 새로운 클래스를 만드는 것

---

### Java vs Python 비교

**Java**:
```java
// 부모 클래스
public class Animal {
    protected String name;

    public Animal(String name) {
        this.name = name;
    }

    public void speak() {
        System.out.println("동물 소리");
    }
}

// 자식 클래스
public class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }

    @Override
    public void speak() {
        System.out.println("멍멍!");
    }
}
```

**Python**:
```python
# 부모 클래스
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "동물 소리"

# 자식 클래스
class Dog(Animal):
    def speak(self):  # 오버라이딩
        return "멍멍!"

# 사용
dog = Dog("흰둥이")
print(dog.name)    # 흰둥이 (부모에게서 상속)
print(dog.speak()) # 멍멍! (재정의)
```

---

## 10.2 기본 상속

### 단순 상속

```python
class Vehicle:
    """탈것 (부모 클래스)"""

    def __init__(self, brand, model):
        self.brand = brand
        self.model = model

    def info(self):
        return f"{self.brand} {self.model}"

    def start(self):
        return "시동을 겁니다"


class Car(Vehicle):  # Vehicle 상속
    """자동차 (자식 클래스)"""

    def __init__(self, brand, model, doors):
        super().__init__(brand, model)  # 부모 생성자 호출
        self.doors = doors  # 자식만의 속성

    def info(self):  # 메서드 오버라이딩
        return f"{super().info()} ({self.doors}도어)"


# 사용
car = Car("현대", "소나타", 4)
print(car.info())   # 현대 소나타 (4도어)
print(car.start())  # 시동을 겁니다 (부모 메서드 그대로)
```

---

### super()의 역할

> 🔗 **비유**: super()는 **부모에게 전화**하는 것입니다.
> - 부모의 메서드/생성자 호출
> - 코드 중복 방지

```python
class Person:
    def __init__(self, name, age):
        print(f"Person 생성자 호출: {name}")
        self.name = name
        self.age = age

    def introduce(self):
        return f"안녕하세요, {self.name}입니다."


class Student(Person):
    def __init__(self, name, age, student_id):
        print(f"Student 생성자 호출: {name}")
        super().__init__(name, age)  # 부모 생성자 호출
        self.student_id = student_id

    def introduce(self):
        # 부모 메서드 활용 + 추가 정보
        parent_intro = super().introduce()
        return f"{parent_intro} 학번은 {self.student_id}입니다."


# 사용
student = Student("Alice", 20, "2024001")
# 출력:
# Student 생성자 호출: Alice
# Person 생성자 호출: Alice

print(student.introduce())
# 안녕하세요, Alice입니다. 학번은 2024001입니다.
```

---

## 10.3 메서드 오버라이딩

> 🎨 **비유**: 오버라이딩은 **리모델링**입니다.
> - 부모에게서 받은 집(메서드)을
> - 내 스타일로 바꿈

```python
class Shape:
    """도형 (부모)"""

    def __init__(self, color):
        self.color = color

    def area(self):
        """넓이 (자식이 구현해야 함)"""
        raise NotImplementedError("하위 클래스에서 구현하세요")

    def describe(self):
        return f"{self.color} 도형"


class Circle(Shape):
    """원"""

    def __init__(self, color, radius):
        super().__init__(color)
        self.radius = radius

    def area(self):  # 오버라이딩
        return 3.14159 * self.radius ** 2

    def describe(self):
        return f"{self.color} 원 (반지름: {self.radius})"


class Rectangle(Shape):
    """직사각형"""

    def __init__(self, color, width, height):
        super().__init__(color)
        self.width = width
        self.height = height

    def area(self):  # 오버라이딩
        return self.width * self.height

    def describe(self):
        return f"{self.color} 직사각형 ({self.width}x{self.height})"


# 사용
circle = Circle("빨강", 10)
print(circle.describe())  # 빨강 원 (반지름: 10)
print(f"넓이: {circle.area():.2f}")  # 넓이: 314.16

rect = Rectangle("파랑", 5, 10)
print(rect.describe())  # 파랑 직사각형 (5x10)
print(f"넓이: {rect.area()}")  # 넓이: 50
```

---

## 10.4 다형성 (Polymorphism)

> 🎭 **비유**: 다형성은 **배우**입니다.
> - 같은 배우(인터페이스)가
> - 다른 역할(구현)을 연기

**다형성**: 같은 인터페이스로 **다른 동작** 수행

```python
class Animal:
    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def speak(self):
        return "멍멍!"

class Cat(Animal):
    def speak(self):
        return "야옹~"

class Cow(Animal):
    def speak(self):
        return "음메~"

# 다형성 활용
def make_speak(animal):
    """동물 울음소리 (어떤 동물이든 상관없음)"""
    print(f"{animal.__class__.__name__}: {animal.speak()}")

# 사용
animals = [Dog(), Cat(), Cow()]

for animal in animals:
    make_speak(animal)

# 출력:
# Dog: 멍멍!
# Cat: 야옹~
# Cow: 음메~
```

---

### 실전 예제: 결제 시스템

```python
class PaymentMethod:
    """결제 수단 (추상 부모)"""

    def __init__(self, amount):
        self.amount = amount

    def pay(self):
        """결제 (자식이 구현)"""
        raise NotImplementedError

    def receipt(self):
        """영수증"""
        return f"결제 금액: {self.amount:,}원"


class CreditCard(PaymentMethod):
    """신용카드"""

    def __init__(self, amount, card_number):
        super().__init__(amount)
        self.card_number = card_number

    def pay(self):
        return f"신용카드({self.card_number[-4:]}) {self.amount:,}원 결제"


class BankTransfer(PaymentMethod):
    """계좌이체"""

    def __init__(self, amount, bank, account):
        super().__init__(amount)
        self.bank = bank
        self.account = account

    def pay(self):
        return f"{self.bank} 계좌이체 {self.amount:,}원 결제"


class Cash(PaymentMethod):
    """현금"""

    def pay(self):
        return f"현금 {self.amount:,}원 결제"


def process_payment(payment_method):
    """결제 처리 (어떤 결제 수단이든 처리 가능)"""
    print(payment_method.pay())
    print(payment_method.receipt())
    print("결제 완료!\n")


# 사용 (다형성)
payments = [
    CreditCard(50000, "1234-5678-9012-3456"),
    BankTransfer(100000, "국민은행", "123-456-789"),
    Cash(30000)
]

for payment in payments:
    process_payment(payment)

# 출력:
# 신용카드(3456) 50,000원 결제
# 결제 금액: 50,000원
# 결제 완료!
#
# 국민은행 계좌이체 100,000원 결제
# 결제 금액: 100,000원
# 결제 완료!
#
# 현금 30,000원 결제
# 결제 금액: 30,000원
# 결제 완료!
```

---

## 10.5 다중 상속

> 🧬 **비유**: 다중 상속은 **혼혈**입니다.
> - 여러 부모의 특성을 모두 물려받음
> - Java는 불가능 (인터페이스로만)
> - Python은 가능!

```python
class Flyable:
    """날 수 있는"""

    def fly(self):
        return "날아갑니다"


class Swimmable:
    """수영할 수 있는"""

    def swim(self):
        return "수영합니다"


class Duck(Flyable, Swimmable):
    """오리 (날기 + 수영)"""

    def __init__(self, name):
        self.name = name

    def quack(self):
        return "꽥꽥!"


# 사용
duck = Duck("도날드")
print(duck.fly())    # 날아갑니다 (Flyable에서 상속)
print(duck.swim())   # 수영합니다 (Swimmable에서 상속)
print(duck.quack())  # 꽥꽥! (Duck 자체 메서드)
```

---

### MRO (Method Resolution Order)

> 🗺️ **비유**: MRO는 **상속 지도**입니다.
> - 메서드 찾을 때 어느 순서로?
> - C3 선형화 알고리즘

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):  # 다중 상속
    pass

# MRO 확인
print(D.mro())
# [<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>]

d = D()
print(d.method())  # B (D → B → C → A 순서로 검색)
```

---

### 다이아몬드 문제

```python
class Animal:
    def __init__(self):
        print("Animal 생성자")

    def speak(self):
        return "동물 소리"


class Mammal(Animal):
    def __init__(self):
        print("Mammal 생성자")
        super().__init__()


class Bird(Animal):
    def __init__(self):
        print("Bird 생성자")
        super().__init__()


class Bat(Mammal, Bird):  # 박쥐 (포유류 + 조류?)
    def __init__(self):
        print("Bat 생성자")
        super().__init__()


# 생성
bat = Bat()
# 출력:
# Bat 생성자
# Mammal 생성자
# Bird 생성자
# Animal 생성자 (한 번만!)

print(Bat.mro())
# [Bat, Mammal, Bird, Animal, object]
```

> 💡 **Python의 해결책**: super()가 MRO를 따라 **순서대로** 호출

---

## 10.6 추상 클래스 (Abstract Base Class)

> 📐 **비유**: 추상 클래스는 **설계도**입니다.
> - 구현 없이 인터페이스만 정의
> - 자식이 반드시 구현하도록 강제

```python
from abc import ABC, abstractmethod

class Database(ABC):
    """데이터베이스 추상 클래스"""

    @abstractmethod
    def connect(self):
        """연결 (반드시 구현)"""
        pass

    @abstractmethod
    def execute(self, query):
        """쿼리 실행 (반드시 구현)"""
        pass

    def log(self, message):
        """로그 (선택적)"""
        print(f"[LOG] {message}")


class MySQL(Database):
    """MySQL 구현"""

    def connect(self):
        self.log("MySQL 연결")
        return "MySQL 연결됨"

    def execute(self, query):
        self.log(f"쿼리 실행: {query}")
        return f"MySQL에서 '{query}' 실행"


class PostgreSQL(Database):
    """PostgreSQL 구현"""

    def connect(self):
        self.log("PostgreSQL 연결")
        return "PostgreSQL 연결됨"

    def execute(self, query):
        self.log(f"쿼리 실행: {query}")
        return f"PostgreSQL에서 '{query}' 실행"


# 추상 클래스는 인스턴스 생성 불가
# db = Database()  # TypeError!

# 구현 클래스는 가능
mysql = MySQL()
print(mysql.connect())
print(mysql.execute("SELECT * FROM users"))

print()

postgres = PostgreSQL()
print(postgres.connect())
print(postgres.execute("SELECT * FROM products"))

# 출력:
# [LOG] MySQL 연결
# MySQL 연결됨
# [LOG] 쿼리 실행: SELECT * FROM users
# MySQL에서 'SELECT * FROM users' 실행
#
# [LOG] PostgreSQL 연결
# PostgreSQL 연결됨
# [LOG] 쿼리 실행: SELECT * FROM products
# PostgreSQL에서 'SELECT * FROM products' 실행
```

---

## 10.7 믹스인 (Mixin)

> 🧩 **비유**: 믹스인은 **플러그인**입니다.
> - 기능을 모듈처럼 추가
> - 상속이지만 "is-a"가 아닌 "has-a" 관계

```python
class JSONMixin:
    """JSON 변환 기능"""

    def to_json(self):
        import json
        return json.dumps(self.__dict__, ensure_ascii=False)

    @classmethod
    def from_json(cls, json_str):
        import json
        data = json.loads(json_str)
        return cls(**data)


class LogMixin:
    """로깅 기능"""

    def log(self, message):
        print(f"[{self.__class__.__name__}] {message}")


class User(JSONMixin, LogMixin):
    """사용자 (믹스인 활용)"""

    def __init__(self, username, email):
        self.username = username
        self.email = email

    def save(self):
        self.log(f"{self.username} 저장 중...")
        # 실제 저장 로직...
        self.log("저장 완료")


# 사용
user = User("alice", "alice@example.com")

# LogMixin 기능
user.save()
# [User] alice 저장 중...
# [User] 저장 완료

# JSONMixin 기능
json_str = user.to_json()
print(json_str)
# {"username": "alice", "email": "alice@example.com"}

restored = User.from_json(json_str)
print(f"복원: {restored.username}, {restored.email}")
# 복원: alice, alice@example.com
```

---

## 10.8 isinstance()와 issubclass()

### isinstance(): 인스턴스 확인

```python
class Animal:
    pass

class Dog(Animal):
    pass

dog = Dog()

print(isinstance(dog, Dog))     # True
print(isinstance(dog, Animal))  # True (부모도 True!)
print(isinstance(dog, object))  # True (모든 클래스는 object 상속)

# 여러 타입 동시 확인
print(isinstance(dog, (Dog, Cat)))  # True (하나라도 맞으면)
```

---

### issubclass(): 상속 관계 확인

```python
class Animal:
    pass

class Dog(Animal):
    pass

print(issubclass(Dog, Animal))   # True
print(issubclass(Dog, object))   # True
print(issubclass(Animal, Dog))   # False
```

---

### 실전 활용: 타입 체크

```python
class Shape:
    pass

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

def calculate_area(shape):
    """도형 넓이 계산"""
    if isinstance(shape, Circle):
        return 3.14159 * shape.radius ** 2
    elif isinstance(shape, Rectangle):
        return shape.width * shape.height
    else:
        raise TypeError("지원하지 않는 도형입니다")

# 사용
circle = Circle(10)
rect = Rectangle(5, 10)

print(f"원 넓이: {calculate_area(circle):.2f}")    # 314.16
print(f"사각형 넓이: {calculate_area(rect)}")     # 50
```

---

## 10.9 실전 예제

### 예제 1: 직원 관리 시스템

```python
from abc import ABC, abstractmethod

class Employee(ABC):
    """직원 (추상 클래스)"""

    employee_count = 0

    def __init__(self, name, emp_id):
        self.name = name
        self.emp_id = emp_id
        Employee.employee_count += 1

    @abstractmethod
    def calculate_salary(self):
        """급여 계산 (자식이 구현)"""
        pass

    @abstractmethod
    def get_role(self):
        """직책 (자식이 구현)"""
        pass

    def __str__(self):
        return f"[{self.get_role()}] {self.name} ({self.emp_id})"


class FullTimeEmployee(Employee):
    """정규직"""

    def __init__(self, name, emp_id, monthly_salary):
        super().__init__(name, emp_id)
        self.monthly_salary = monthly_salary

    def calculate_salary(self):
        return self.monthly_salary

    def get_role(self):
        return "정규직"


class PartTimeEmployee(Employee):
    """파트타임"""

    def __init__(self, name, emp_id, hourly_rate, hours_worked):
        super().__init__(name, emp_id)
        self.hourly_rate = hourly_rate
        self.hours_worked = hours_worked

    def calculate_salary(self):
        return self.hourly_rate * self.hours_worked

    def get_role(self):
        return "파트타임"


class Contractor(Employee):
    """계약직"""

    def __init__(self, name, emp_id, project_fee):
        super().__init__(name, emp_id)
        self.project_fee = project_fee

    def calculate_salary(self):
        return self.project_fee

    def get_role(self):
        return "계약직"


class Company:
    """회사"""

    def __init__(self, name):
        self.name = name
        self.employees = []

    def hire(self, employee):
        """채용"""
        self.employees.append(employee)
        print(f"✅ {employee.name} 채용 완료")

    def fire(self, emp_id):
        """해고"""
        for emp in self.employees:
            if emp.emp_id == emp_id:
                self.employees.remove(emp)
                print(f"❌ {emp.name} 퇴사")
                return
        print("직원을 찾을 수 없습니다")

    def total_payroll(self):
        """총 급여"""
        return sum(emp.calculate_salary() for emp in self.employees)

    def show_employees(self):
        """직원 목록"""
        print(f"\n=== {self.name} 직원 목록 ===")
        for emp in self.employees:
            salary = emp.calculate_salary()
            print(f"{emp} - 급여: {salary:,}원")
        print(f"총 급여: {self.total_payroll():,}원\n")


# 사용
company = Company("Python Corp")

# 직원 채용
company.hire(FullTimeEmployee("Alice", "E001", 3000000))
company.hire(FullTimeEmployee("Bob", "E002", 3500000))
company.hire(PartTimeEmployee("Charlie", "P001", 15000, 80))
company.hire(Contractor("David", "C001", 5000000))

company.show_employees()

# 퇴사
company.fire("P001")

company.show_employees()

print(f"전체 직원 수: {Employee.employee_count}명")

# 출력:
# ✅ Alice 채용 완료
# ✅ Bob 채용 완료
# ✅ Charlie 채용 완료
# ✅ David 채용 완료
#
# === Python Corp 직원 목록 ===
# [정규직] Alice (E001) - 급여: 3,000,000원
# [정규직] Bob (E002) - 급여: 3,500,000원
# [파트타임] Charlie (P001) - 급여: 1,200,000원
# [계약직] David (C001) - 급여: 5,000,000원
# 총 급여: 12,700,000원
#
# ❌ Charlie 퇴사
#
# === Python Corp 직원 목록 ===
# [정규직] Alice (E001) - 급여: 3,000,000원
# [정규직] Bob (E002) - 급여: 3,500,000원
# [계약직] David (C001) - 급여: 5,000,000원
# 총 급여: 11,500,000원
#
# 전체 직원 수: 4명
```

---

### 예제 2: 게임 캐릭터 시스템

```python
from abc import ABC, abstractmethod

class Character(ABC):
    """캐릭터 (추상 클래스)"""

    def __init__(self, name, hp, attack):
        self.name = name
        self.max_hp = hp
        self.hp = hp
        self.attack = attack

    @abstractmethod
    def special_attack(self, target):
        """특수 공격 (자식이 구현)"""
        pass

    def normal_attack(self, target):
        """일반 공격"""
        damage = self.attack
        target.take_damage(damage)
        return f"{self.name}의 일반 공격! {target.name}에게 {damage} 데미지"

    def take_damage(self, damage):
        """데미지 받기"""
        self.hp = max(0, self.hp - damage)
        if self.hp == 0:
            return f"{self.name} 전투 불능!"
        return f"{self.name} HP: {self.hp}/{self.max_hp}"

    def is_alive(self):
        return self.hp > 0

    def heal(self, amount):
        """치유"""
        self.hp = min(self.max_hp, self.hp + amount)
        return f"{self.name} {amount} 회복! HP: {self.hp}/{self.max_hp}"

    def __str__(self):
        return f"{self.name} (HP: {self.hp}/{self.max_hp})"


class Warrior(Character):
    """전사"""

    def __init__(self, name):
        super().__init__(name, hp=150, attack=20)
        self.defense = 10

    def special_attack(self, target):
        """강력한 일격"""
        damage = self.attack * 2
        target.take_damage(damage)
        return f"{self.name}의 강력한 일격! {target.name}에게 {damage} 데미지"

    def take_damage(self, damage):
        """방어력 적용"""
        reduced = max(1, damage - self.defense)
        return super().take_damage(reduced)


class Mage(Character):
    """마법사"""

    def __init__(self, name):
        super().__init__(name, hp=80, attack=30)
        self.mana = 100

    def special_attack(self, target):
        """파이어볼"""
        if self.mana < 30:
            return f"{self.name}의 마나 부족!"

        self.mana -= 30
        damage = self.attack * 2.5
        target.take_damage(int(damage))
        return f"{self.name}의 파이어볼! {target.name}에게 {int(damage)} 데미지 (마나: {self.mana}/100)"


class Archer(Character):
    """궁수"""

    def __init__(self, name):
        super().__init__(name, hp=100, attack=25)
        self.crit_chance = 0.3  # 30% 치명타

    def special_attack(self, target):
        """정밀 사격"""
        import random
        is_crit = random.random() < self.crit_chance
        damage = self.attack * 3 if is_crit else self.attack * 1.5

        target.take_damage(int(damage))
        crit_msg = " [치명타!]" if is_crit else ""
        return f"{self.name}의 정밀 사격{crit_msg} {target.name}에게 {int(damage)} 데미지"


# 전투 시스템
def battle(char1, char2):
    """1:1 전투"""
    print(f"\n⚔️ {char1.name} VS {char2.name}")
    print(f"{char1}")
    print(f"{char2}\n")

    turn = 1
    while char1.is_alive() and char2.is_alive():
        print(f"--- Turn {turn} ---")

        # char1 공격
        if turn % 2 == 1:
            action = char1.special_attack(char2)
        else:
            action = char1.normal_attack(char2)
        print(action)
        print(char2.take_damage(0))  # 상태 출력

        if not char2.is_alive():
            print(f"\n🏆 {char1.name} 승리!")
            break

        print()

        # char2 공격
        if turn % 2 == 1:
            action = char2.normal_attack(char1)
        else:
            action = char2.special_attack(char1)
        print(action)
        print(char1.take_damage(0))

        if not char1.is_alive():
            print(f"\n🏆 {char2.name} 승리!")
            break

        print()
        turn += 1


# 사용
warrior = Warrior("전사")
mage = Mage("마법사")
archer = Archer("궁수")

battle(warrior, mage)
```

---

## 실전 팁

### 💡 Tip 1: 상속 vs 컴포지션

**상속 (is-a)**:
```python
class Dog(Animal):  # Dog is an Animal
    pass
```

**컴포지션 (has-a)** - 더 권장:
```python
class Car:
    def __init__(self):
        self.engine = Engine()  # Car has an Engine
        self.wheels = [Wheel() for _ in range(4)]
```

> 💡 **원칙**: "상속보다 컴포지션을 선호하라"

---

### 💡 Tip 2: 얕은 상속 트리

**❌ 나쁜 예** (깊은 상속):
```python
class A:
    pass
class B(A):
    pass
class C(B):
    pass
class D(C):  # 너무 깊음!
    pass
```

**✅ 좋은 예** (얕은 상속):
```python
class Base:
    pass

class TypeA(Base):
    pass

class TypeB(Base):
    pass
```

---

### 💡 Tip 3: super() 항상 사용

```python
class Parent:
    def __init__(self, name):
        self.name = name

# ❌ 나쁜 예
class Child1(Parent):
    def __init__(self, name, age):
        Parent.__init__(self, name)  # 하드코딩
        self.age = age

# ✅ 좋은 예
class Child2(Parent):
    def __init__(self, name, age):
        super().__init__(name)  # 유연함
        self.age = age
```

---

## 연습 문제

### 문제 1: 도형 계층

추상 클래스 `Shape`와 자식 클래스 `Triangle`, `Square` 작성:
- 모두 `area()` 메서드 구현
- 삼각형: (밑변 * 높이) / 2
- 정사각형: 한 변 ** 2

<details>
<summary>정답 보기</summary>

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Triangle(Shape):
    def __init__(self, base, height):
        self.base = base
        self.height = height

    def area(self):
        return (self.base * self.height) / 2

class Square(Shape):
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side ** 2

# 테스트
triangle = Triangle(10, 5)
square = Square(5)

print(f"삼각형 넓이: {triangle.area()}")  # 25.0
print(f"정사각형 넓이: {square.area()}")  # 25
```
</details>

---

### 문제 2: 동물원

`Animal` 부모 클래스와 여러 동물 클래스 작성:
- 모두 `speak()` 메서드 구현
- `Zoo` 클래스로 동물 관리

<details>
<summary>정답 보기</summary>

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        raise NotImplementedError

class Lion(Animal):
    def speak(self):
        return "어흥!"

class Elephant(Animal):
    def speak(self):
        return "뿌우~"

class Monkey(Animal):
    def speak(self):
        return "끼끼!"

class Zoo:
    def __init__(self):
        self.animals = []

    def add_animal(self, animal):
        self.animals.append(animal)

    def make_all_speak(self):
        for animal in self.animals:
            print(f"{animal.name}: {animal.speak()}")

# 테스트
zoo = Zoo()
zoo.add_animal(Lion("심바"))
zoo.add_animal(Elephant("덤보"))
zoo.add_animal(Monkey("조지"))

zoo.make_all_speak()
# 심바: 어흥!
# 덤보: 뿌우~
# 조지: 끼끼!
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **상속 문법**
   ```python
   class Child(Parent):
       def __init__(self):
           super().__init__()
   ```

2. **메서드 오버라이딩**
   - 부모 메서드를 자식이 재정의
   - `super()`로 부모 메서드 활용

3. **다형성**
   - 같은 인터페이스, 다른 구현
   - `isinstance()`, `issubclass()` 활용

4. **추상 클래스**
   ```python
   from abc import ABC, abstractmethod

   class Base(ABC):
       @abstractmethod
       def method(self):
           pass
   ```

5. **다중 상속**
   - Python만 지원
   - MRO 주의

---

## 다음 챕터 예고

Chapter 11에서는 **특수 메서드와 연산자 오버로딩**을 다룹니다:
- 컨테이너 프로토콜
- 비교 연산자
- 컨텍스트 매니저
- 디스크립터

---

[다음: Chapter 11. 특수 메서드 →](chapter11-special-methods.md)