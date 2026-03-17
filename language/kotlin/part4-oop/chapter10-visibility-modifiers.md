# Chapter 10. 가시성 제어자와 접근 제어

## 개요

Kotlin의 가시성 제어자는 Java와 유사하지만 **기본 가시성**과 **internal** 제어자에서 차이가 있습니다.

---

## 10.1 가시성 제어자 (private, protected, internal, public)

### 4가지 가시성 제어자

| 제어자 | 설명 | 접근 범위 |
|--------|------|-----------|
| `private` | 같은 파일/클래스 내에서만 | 가장 제한적 |
| `protected` | 같은 클래스와 하위 클래스 | 상속 관계 |
| `internal` | **같은 모듈 내**에서만 | 모듈 단위 |
| `public` | 어디서나 접근 가능 | **기본값** |

---

### Top-level 선언의 가시성

**파일: User.kt**
```kotlin
// public이 기본값
class User(val name: String)

private class InternalUser(val id: Long)  // 같은 파일에서만 접근 가능

internal class ModuleUser(val email: String)  // 같은 모듈에서만

fun createUser(): User = User("Alice")

private fun validateUser(user: User) {  // 같은 파일에서만
    // ...
}
```

**다른 파일에서**:
```kotlin
val user = User("Bob")  // OK
val internalUser = InternalUser(1)  // 컴파일 에러! private
val moduleUser = ModuleUser("test@example.com")  // 같은 모듈이면 OK
```

---

### 클래스 멤버의 가시성

```kotlin
class Person(
    val name: String,              // public (기본값)
    private val ssn: String,       // private
    internal val employeeId: Long  // internal
) {
    // public 프로퍼티
    var age: Int = 0

    // private 프로퍼티
    private var password: String = ""

    // protected 함수 (open 클래스에서만 의미 있음)
    protected fun validateAge() {
        require(age >= 0)
    }

    // internal 함수
    internal fun getInternalInfo() = "Employee: $employeeId"

    // private 함수
    private fun encrypt(text: String): String {
        return text.reversed()  // 간단한 예제
    }

    // public 함수
    fun setPassword(newPassword: String) {
        password = encrypt(newPassword)
    }
}

fun main() {
    val person = Person("Alice", "123-45-6789", 1001)

    println(person.name)  // OK
    // println(person.ssn)  // 컴파일 에러! private
    println(person.employeeId)  // OK (같은 모듈)

    person.age = 30  // OK
    // person.password = "test"  // 컴파일 에러! private

    person.setPassword("secret")  // OK
    // person.validateAge()  // 컴파일 에러! protected
    // person.encrypt("test")  // 컴파일 에러! private
}
```

---

### Java와 비교

**Java**:
```java
public class Person {
    public String name;           // 어디서나 접근
    private String ssn;           // 클래스 내부에서만
    protected int age;            // 상속 관계에서
    String address;               // package-private (기본값!)

    public Person(String name, String ssn) {
        this.name = name;
        this.ssn = ssn;
    }

    public String getName() {
        return name;
    }

    private void validateAge() {
        // ...
    }
}
```

**Kotlin**:
```kotlin
class Person(
    val name: String,         // public (기본값!)
    private val ssn: String
) {
    protected var age: Int = 0    // 상속 관계
    internal var address: String = ""  // 모듈 내

    fun getName() = name

    private fun validateAge() {
        // ...
    }
}
```

**주요 차이점**:
1. **기본 가시성**: Java는 package-private, Kotlin은 **public**
2. **package-private 없음**: Kotlin에는 없음 → `internal`로 대체
3. **protected**: Kotlin에서는 같은 패키지 내 접근 불가 (하위 클래스만)

---

## 10.2 internal의 활용

### internal이란?

**모듈(module)** 단위로 가시성을 제한합니다.

**모듈의 예**:
- IntelliJ IDEA 모듈
- Maven/Gradle 프로젝트
- Gradle source set
- 하나의 Ant 태스크

---

### internal 사용 예제

**라이브러리 개발 시**:
```kotlin
// 공개 API
class UserService {
    fun createUser(name: String): User {
        val validator = UserValidator()  // internal 클래스 사용
        validator.validate(name)
        return User(name)
    }
}

// 내부 구현 (같은 모듈에서만 접근)
internal class UserValidator {
    fun validate(name: String) {
        require(name.isNotBlank()) { "Name cannot be blank" }
    }
}

// 내부 유틸리티
internal fun generateId(): Long {
    return System.currentTimeMillis()
}

data class User(val name: String, internal val id: Long = generateId())
```

**다른 모듈에서**:
```kotlin
val service = UserService()
val user = service.createUser("Alice")  // OK

// val validator = UserValidator()  // 컴파일 에러! internal
// val id = generateId()  // 컴파일 에러! internal
```

---

### internal의 장점

1. **모듈 내부 API 숨기기**
2. **리팩토링 용이** (외부에 영향 없이 내부 구현 변경)
3. **명확한 경계** (public API vs internal 구현)

```kotlin
// public API - 안정적, 하위 호환성 유지
class OrderService {
    fun createOrder(items: List<Item>): Order {
        // internal 구현 사용
        val calculator = PriceCalculator()
        val total = calculator.calculate(items)
        return Order(items, total)
    }
}

// internal 구현 - 자유롭게 변경 가능
internal class PriceCalculator {
    fun calculate(items: List<Item>): Double {
        // 복잡한 계산 로직
        return items.sumOf { it.price }
    }
}
```

---

### 실전 예제: Multi-module 프로젝트

```
myapp/
├── api/              (모듈 1)
│   └── UserApi.kt
├── domain/           (모듈 2)
│   ├── User.kt
│   └── UserRepository.kt (internal)
└── infrastructure/   (모듈 3)
    └── Database.kt
```

**domain/UserRepository.kt**:
```kotlin
// domain 모듈 내부에서만 사용
internal interface UserRepository {
    fun findById(id: Long): User?
    fun save(user: User)
}

internal class UserRepositoryImpl : UserRepository {
    override fun findById(id: Long): User? {
        // ...
    }

    override fun save(user: User) {
        // ...
    }
}
```

**domain/UserService.kt**:
```kotlin
// 공개 API
class UserService(
    private val repository: UserRepository  // internal이지만 같은 모듈이므로 OK
) {
    fun getUser(id: Long): User? {
        return repository.findById(id)
    }
}
```

**api/UserApi.kt** (다른 모듈):
```kotlin
class UserApi(private val userService: UserService) {
    fun handleGetUser(id: Long): Response {
        val user = userService.getUser(id)  // OK
        // val repo: UserRepository = ...  // 컴파일 에러! internal
        return Response(user)
    }
}
```

---

## 10.3 Java와 비교: 접근 제어 차이점

### 가시성 제어자 비교표

| | Java | Kotlin |
|------------|------|--------|
| **클래스 내부** | `private` | `private` |
| **하위 클래스** | `protected` | `protected` (패키지 접근 불가) |
| **같은 패키지** | package-private (기본) | ❌ 없음 |
| **같은 모듈** | ❌ 없음 | `internal` |
| **모든 곳** | `public` | `public` (기본) |

---

### protected의 차이

**Java protected**:
```java
package com.example;

public class Parent {
    protected void foo() {
        System.out.println("Parent.foo()");
    }
}

// 같은 패키지의 다른 클래스
package com.example;

public class Other {
    void test() {
        Parent p = new Parent();
        p.foo();  // OK! 같은 패키지이므로
    }
}

// 다른 패키지의 하위 클래스
package com.other;
import com.example.Parent;

public class Child extends Parent {
    void test() {
        foo();  // OK! 하위 클래스이므로
    }
}
```

**Kotlin protected**:
```kotlin
package com.example

open class Parent {
    protected fun foo() {
        println("Parent.foo()")
    }
}

// 같은 패키지의 다른 클래스
package com.example

class Other {
    fun test() {
        val p = Parent()
        // p.foo()  // 컴파일 에러! 같은 패키지여도 접근 불가
    }
}

// 하위 클래스
class Child : Parent() {
    fun test() {
        foo()  // OK! 하위 클래스이므로
    }
}
```

---

### 기본 가시성 차이

**Java**:
```java
// 기본 가시성 = package-private
class MyClass {  // package-private
    int value;   // package-private
    void method() { }  // package-private
}
```

**Kotlin**:
```kotlin
// 기본 가시성 = public
class MyClass {  // public
    val value = 0  // public
    fun method() { }  // public
}
```

---

### internal vs package-private

**Java package-private**: 같은 패키지 내에서만 접근 가능
```java
// com/example/Internal.java
package com.example;

class InternalClass {  // package-private
    void internalMethod() { }
}

// com/example/User.java
package com.example;

class User {
    void test() {
        InternalClass obj = new InternalClass();  // OK
        obj.internalMethod();  // OK
    }
}

// com/other/Other.java
package com.other;

class Other {
    void test() {
        // InternalClass obj = new InternalClass();  // 컴파일 에러
    }
}
```

**Kotlin internal**: 같은 모듈 내에서만 접근 가능
```kotlin
// 모듈 A
internal class InternalClass {
    internal fun internalMethod() { }
}

class User {
    fun test() {
        val obj = InternalClass()  // OK (같은 모듈)
        obj.internalMethod()  // OK
    }
}

// 모듈 B
class Other {
    fun test() {
        // val obj = InternalClass()  // 컴파일 에러 (다른 모듈)
    }
}
```

---

## 실전 팁

### 💡 Tip 1: 최소 권한 원칙

```kotlin
// ✅ 가능한 가장 제한적인 가시성 사용
class UserService {
    private val cache = mutableMapOf<Long, User>()

    private fun validateUser(user: User) {
        // ...
    }

    // 필요한 것만 public
    fun getUser(id: Long): User? {
        return cache[id]
    }
}

// ❌ 모든 것을 public으로
class BadUserService {
    val cache = mutableMapOf<Long, User>()  // 외부에서 직접 수정 가능!

    fun validateUser(user: User) {  // 내부 구현이 노출됨
        // ...
    }

    fun getUser(id: Long): User? {
        return cache[id]
    }
}
```

---

### 💡 Tip 2: internal로 모듈 API 관리

```kotlin
// 라이브러리 공개 API
class UserRepository {
    fun save(user: User) {
        val validator = UserValidator()  // internal 사용
        validator.validate(user)
        internalSave(user)
    }

    private fun internalSave(user: User) {
        // ...
    }
}

// 모듈 내부 유틸리티
internal class UserValidator {
    fun validate(user: User) {
        // ...
    }
}
```

---

### 💡 Tip 3: Getter/Setter 가시성 분리

```kotlin
class User(name: String) {
    var name: String = name
        private set  // getter는 public, setter는 private

    fun updateName(newName: String) {
        // 검증 로직
        require(newName.isNotBlank())
        name = newName
    }
}

val user = User("Alice")
println(user.name)  // OK
// user.name = "Bob"  // 컴파일 에러! setter는 private
user.updateName("Bob")  // OK
```

---

### 💡 Tip 4: 테스트를 위한 internal

```kotlin
// src/main/kotlin
class PaymentService {
    internal fun calculateFee(amount: Double): Double {
        return amount * 0.03
    }

    fun processPayment(amount: Double): Double {
        val fee = calculateFee(amount)
        return amount + fee
    }
}

// src/test/kotlin (같은 모듈)
class PaymentServiceTest {
    @Test
    fun testCalculateFee() {
        val service = PaymentService()
        assertEquals(3.0, service.calculateFee(100.0))  // internal이지만 테스트 가능
    }
}
```

---

## 연습 문제

### 문제 1: 은행 계좌 클래스
잔액은 private, 입출금은 public으로 구현

<details>
<summary>정답 보기</summary>

```kotlin
class BankAccount(
    val accountNumber: String,
    initialBalance: Double = 0.0
) {
    private var balance: Double = initialBalance

    fun getBalance(): Double = balance

    fun deposit(amount: Double) {
        require(amount > 0) { "입금액은 0보다 커야 합니다" }
        balance += amount
        println("입금: $amount, 잔액: $balance")
    }

    fun withdraw(amount: Double): Boolean {
        return if (amount > 0 && balance >= amount) {
            balance -= amount
            println("출금: $amount, 잔액: $balance")
            true
        } else {
            println("출금 실패: 잔액 부족 또는 잘못된 금액")
            false
        }
    }
}

fun main() {
    val account = BankAccount("1234-5678", 10000.0)

    println("잔액: ${account.getBalance()}")  // 10000.0
    account.deposit(5000.0)   // 입금: 5000.0, 잔액: 15000.0
    account.withdraw(3000.0)  // 출금: 3000.0, 잔액: 12000.0

    // account.balance = 1000000.0  // 컴파일 에러! private
}
```
</details>

### 문제 2: 설정 싱글톤
internal로 모듈 내에서만 접근 가능한 설정 클래스

<details>
<summary>정답 보기</summary>

```kotlin
internal object AppConfig {
    private val settings = mutableMapOf<String, String>()

    fun set(key: String, value: String) {
        settings[key] = value
    }

    fun get(key: String): String? {
        return settings[key]
    }

    fun getOrDefault(key: String, default: String): String {
        return settings[key] ?: default
    }
}

// 같은 모듈에서 사용
fun initializeApp() {
    AppConfig.set("api_url", "https://api.example.com")
    AppConfig.set("timeout", "30")
}

fun makeApiCall() {
    val url = AppConfig.get("api_url")
    val timeout = AppConfig.getOrDefault("timeout", "10")
    println("API Call to $url with timeout $timeout")
}
```
</details>

### 문제 3: 가시성 분리
name은 public getter/private setter, email은 internal

<details>
<summary>정답 보기</summary>

```kotlin
class User(
    name: String,
    email: String
) {
    var name: String = name
        private set

    internal var email: String = email

    private var loginAttempts = 0

    fun updateName(newName: String) {
        require(newName.isNotBlank()) { "이름은 비어있을 수 없습니다" }
        name = newName
    }

    internal fun recordLoginAttempt() {
        loginAttempts++
    }

    internal fun getLoginAttempts() = loginAttempts
}

fun main() {
    val user = User("Alice", "alice@example.com")

    println(user.name)  // OK
    // user.name = "Bob"  // 컴파일 에러! private setter

    user.updateName("Bob")  // OK
    println(user.name)  // Bob

    println(user.email)  // OK (같은 모듈)
    user.recordLoginAttempt()  // OK (같은 모듈)
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **기본 가시성**
   - Kotlin: `public` (기본)
   - Java: package-private (기본)

2. **4가지 제어자**
   - `private`: 클래스/파일 내
   - `protected`: 하위 클래스
   - `internal`: 모듈 내
   - `public`: 모든 곳

3. **internal**
   - Kotlin의 독특한 제어자
   - 모듈 단위 캡슐화

4. **protected 차이**
   - Java: 같은 패키지 + 하위 클래스
   - Kotlin: 하위 클래스만

5. **Getter/Setter 분리**
   - `var name: String = ... private set`

---

## 다음 챕터 예고

Chapter 11에서는 **특별한 클래스들**을 다룹니다:
- Enum 클래스
- Sealed 클래스
- 중첩 클래스와 내부 클래스
- 인라인 클래스 (Value Class)

---

[← 이전: Chapter 9. 상속과 인터페이스](chapter9-inheritance-interfaces.md) | [다음: Chapter 11. 특별한 클래스들 →](chapter11-special-classes.md)