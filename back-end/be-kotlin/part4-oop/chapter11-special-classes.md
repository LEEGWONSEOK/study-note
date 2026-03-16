# Chapter 11. 특별한 클래스들

## 개요

Kotlin은 특수한 목적을 위한 여러 클래스 타입을 제공합니다: Enum, Sealed, Nested, Inner, Value 클래스 등.

---

## 11.1 Enum 클래스

### 기본 사용법

```kotlin
enum class Direction {
    NORTH, SOUTH, EAST, WEST
}

fun main() {
    val direction = Direction.NORTH
    println(direction)  // NORTH

    // 문자열로 변환
    println(direction.name)  // NORTH

    // 순서 (0부터 시작)
    println(direction.ordinal)  // 0
}
```

---

### 프로퍼티와 메소드

```kotlin
enum class Color(val rgb: Int) {
    RED(0xFF0000),
    GREEN(0x00FF00),
    BLUE(0x0000FF),
    YELLOW(0xFFFF00);  // 세미콜론 필수 (멤버가 있을 때)

    fun containsRed() = (rgb and 0xFF0000) != 0
}

fun main() {
    val red = Color.RED
    println(red.rgb)  // 16711680 (0xFF0000)
    println(red.containsRed())  // true

    val green = Color.GREEN
    println(green.containsRed())  // false
}
```

---

### 인터페이스 구현

```kotlin
interface Printable {
    fun print()
}

enum class PrintType : Printable {
    NORMAL {
        override fun print() {
            println("일반 출력")
        }
    },
    BOLD {
        override fun print() {
            println("굵은 출력")
        }
    },
    ITALIC {
        override fun print() {
            println("기울임 출력")
        }
    }
}

fun main() {
    PrintType.NORMAL.print()  // 일반 출력
    PrintType.BOLD.print()    // 굵은 출력
}
```

---

### when과 함께 사용

```kotlin
enum class Status {
    PENDING, APPROVED, REJECTED, CANCELLED
}

fun getStatusMessage(status: Status): String {
    return when (status) {
        Status.PENDING -> "대기 중"
        Status.APPROVED -> "승인됨"
        Status.REJECTED -> "거절됨"
        Status.CANCELLED -> "취소됨"
    }
    // else 불필요! 모든 케이스를 다룸
}

fun main() {
    println(getStatusMessage(Status.APPROVED))  // 승인됨
}
```

---

### valueOf와 values

```kotlin
enum class Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

fun main() {
    // 모든 값 가져오기
    val allDays = Day.values()
    println(allDays.contentToString())
    // [MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY]

    // 문자열로 Enum 찾기
    val monday = Day.valueOf("MONDAY")
    println(monday)  // MONDAY

    // 안전한 변환 (Kotlin 1.9+)
    val day = Day.entries.find { it.name == "MONDAY" }
    println(day)  // MONDAY
}
```

---

### 실전 예제: HTTP 상태 코드

```kotlin
enum class HttpStatus(val code: Int, val message: String) {
    OK(200, "OK"),
    CREATED(201, "Created"),
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    FORBIDDEN(403, "Forbidden"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_SERVER_ERROR(500, "Internal Server Error");

    fun isSuccess() = code in 200..299
    fun isClientError() = code in 400..499
    fun isServerError() = code in 500..599

    companion object {
        fun fromCode(code: Int): HttpStatus? {
            return values().find { it.code == code }
        }
    }
}

fun main() {
    val status = HttpStatus.OK
    println("${status.code} - ${status.message}")  // 200 - OK
    println("Success? ${status.isSuccess()}")  // true

    val notFound = HttpStatus.fromCode(404)
    println(notFound)  // NOT_FOUND
}
```

---

## 11.2 Sealed 클래스 (봉인 클래스)

### 개념

**Sealed Class**: 제한된 클래스 계층 구조를 표현합니다.

**특징**:
- 같은 패키지/모듈 내에서만 하위 클래스 생성 가능
- `when` 표현식에서 `else` 불필요

---

### 기본 사용법

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String, val code: Int) : Result()
    object Loading : Result()
}

fun handleResult(result: Result): String {
    return when (result) {
        is Result.Success -> "성공: ${result.data}"
        is Result.Error -> "에러 [${result.code}]: ${result.message}"
        Result.Loading -> "로딩 중..."
    }
    // else 불필요! 컴파일러가 모든 케이스를 알고 있음
}

fun main() {
    val success = Result.Success("데이터 로드 완료")
    val error = Result.Error("서버 에러", 500)
    val loading = Result.Loading

    println(handleResult(success))  // 성공: 데이터 로드 완료
    println(handleResult(error))    // 에러 [500]: 서버 에러
    println(handleResult(loading))  // 로딩 중...
}
```

---

### Enum vs Sealed Class

**Enum**: 인스턴스가 고정
```kotlin
enum class Color {
    RED, GREEN, BLUE
}
// RED, GREEN, BLUE만 존재 가능
```

**Sealed Class**: 타입이 고정, 인스턴스는 다양
```kotlin
sealed class Color {
    data class RGB(val r: Int, val g: Int, val b: Int) : Color()
    data class HSV(val h: Int, val s: Int, val v: Int) : Color()
}

val color1 = Color.RGB(255, 0, 0)
val color2 = Color.RGB(0, 255, 0)  // 다른 인스턴스
```

---

### 실전 예제: UI 상태 관리

```kotlin
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val exception: Throwable) : UiState<Nothing>()
}

data class User(val name: String, val email: String)

fun renderUserScreen(state: UiState<User>) {
    when (state) {
        is UiState.Loading -> {
            println("로딩 중...")
        }
        is UiState.Success -> {
            println("사용자: ${state.data.name}")
            println("이메일: ${state.data.email}")
        }
        is UiState.Error -> {
            println("에러 발생: ${state.exception.message}")
        }
    }
}

fun main() {
    renderUserScreen(UiState.Loading)
    // 로딩 중...

    val user = User("Alice", "alice@example.com")
    renderUserScreen(UiState.Success(user))
    // 사용자: Alice
    // 이메일: alice@example.com

    renderUserScreen(UiState.Error(Exception("네트워크 오류")))
    // 에러 발생: 네트워크 오류
}
```

---

### 실전 예제: API 응답

```kotlin
sealed class ApiResponse<out T> {
    data class Success<T>(
        val data: T,
        val statusCode: Int = 200
    ) : ApiResponse<T>()

    data class Error(
        val message: String,
        val statusCode: Int,
        val exception: Throwable? = null
    ) : ApiResponse<Nothing>()

    object NetworkError : ApiResponse<Nothing>()
    object Timeout : ApiResponse<Nothing>()
}

fun <T> handleApiResponse(response: ApiResponse<T>): T? {
    return when (response) {
        is ApiResponse.Success -> {
            println("성공: ${response.statusCode}")
            response.data
        }
        is ApiResponse.Error -> {
            println("에러: ${response.message} (${response.statusCode})")
            null
        }
        ApiResponse.NetworkError -> {
            println("네트워크 에러")
            null
        }
        ApiResponse.Timeout -> {
            println("타임아웃")
            null
        }
    }
}
```

---

## 11.3 중첩 클래스와 내부 클래스

### 중첩 클래스 (Nested Class)

외부 클래스의 인스턴스 없이 독립적으로 존재:

```kotlin
class Outer {
    private val outerValue = "Outer"

    class Nested {
        fun hello() = "Hello from Nested"

        // fun accessOuter() = outerValue  // 컴파일 에러! 외부 접근 불가
    }
}

fun main() {
    val nested = Outer.Nested()  // Outer 인스턴스 불필요
    println(nested.hello())  // Hello from Nested
}
```

---

### 내부 클래스 (Inner Class)

**`inner`** 키워드로 외부 클래스 참조:

```kotlin
class Outer(val outerValue: String) {
    inner class Inner {
        fun accessOuter() = "Accessing: $outerValue"

        fun getOuter(): Outer = this@Outer
    }
}

fun main() {
    val outer = Outer("Hello")
    val inner = outer.Inner()  // Outer 인스턴스 필요

    println(inner.accessOuter())  // Accessing: Hello
    println(inner.getOuter().outerValue)  // Hello
}
```

---

### Java와 비교

**Java**:
```java
public class Outer {
    private String value = "Outer";

    // 중첩 클래스 (static)
    static class Nested {
        void hello() {
            System.out.println("Nested");
            // System.out.println(value);  // 컴파일 에러
        }
    }

    // 내부 클래스 (non-static, 기본값!)
    class Inner {
        void accessOuter() {
            System.out.println(value);  // OK
        }
    }
}

// 사용
Outer.Nested nested = new Outer.Nested();
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

**Kotlin**:
```kotlin
class Outer(val value: String) {
    // 중첩 클래스 (기본값!)
    class Nested {
        fun hello() = "Nested"
        // fun accessOuter() = value  // 컴파일 에러
    }

    // 내부 클래스 (inner 필요)
    inner class Inner {
        fun accessOuter() = value  // OK
    }
}

// 사용
val nested = Outer.Nested()
val outer = Outer("Hello")
val inner = outer.Inner()
```

**차이점**:
- Java: 기본이 내부 클래스 (메모리 누수 위험)
- Kotlin: 기본이 중첩 클래스 (더 안전)

---

### 실전 예제: Builder 패턴

```kotlin
class Person private constructor(
    val name: String,
    val age: Int,
    val email: String?
) {
    class Builder {
        private var name: String = ""
        private var age: Int = 0
        private var email: String? = null

        fun name(name: String) = apply { this.name = name }
        fun age(age: Int) = apply { this.age = age }
        fun email(email: String) = apply { this.email = email }

        fun build(): Person {
            require(name.isNotBlank()) { "Name is required" }
            require(age > 0) { "Age must be positive" }
            return Person(name, age, email)
        }
    }

    companion object {
        fun builder() = Builder()
    }
}

fun main() {
    val person = Person.builder()
        .name("Alice")
        .age(25)
        .email("alice@example.com")
        .build()

    println("${person.name}, ${person.age}, ${person.email}")
}
```

---

## 11.4 인라인 클래스 (Value Class)

### 개념

**Value Class**: 단일 값을 감싸는 래퍼 클래스지만 런타임 오버헤드가 없습니다.

**사용 이유**:
- 타입 안전성
- 의미 명확성
- 성능 유지 (인라인화)

---

### 기본 사용법

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class Email(val value: String)

data class User(
    val id: UserId,
    val email: Email
)

fun sendEmail(email: Email) {
    println("Sending email to: ${email.value}")
}

fun main() {
    val userId = UserId(123)
    val email = Email("user@example.com")

    val user = User(userId, email)

    sendEmail(email)  // OK
    // sendEmail(userId)  // 컴파일 에러! 타입 안전성
}
```

---

### 장점: 타입 안전성

```kotlin
// ❌ 타입 안전하지 않음
fun transfer(fromAccount: Long, toAccount: Long, amount: Double) {
    // ...
}

transfer(123, 456, 1000.0)
transfer(456, 123, 1000.0)  // 순서 바뀌어도 컴파일러가 모름!

// ✅ Value Class로 타입 안전성 확보
@JvmInline
value class AccountId(val value: Long)

@JvmInline
value class Amount(val value: Double)

fun transfer(from: AccountId, to: AccountId, amount: Amount) {
    println("Transfer ${amount.value} from ${from.value} to ${to.value}")
}

val from = AccountId(123)
val to = AccountId(456)
val amount = Amount(1000.0)

transfer(from, to, amount)  // OK
// transfer(to, from, amount)  // 의도 명확!
// transfer(amount, from, to)  // 컴파일 에러!
```

---

### 멤버 함수

```kotlin
@JvmInline
value class Password(val value: String) {
    init {
        require(value.length >= 8) { "Password must be at least 8 characters" }
    }

    fun isStrong(): Boolean {
        return value.length >= 12 &&
               value.any { it.isUpperCase() } &&
               value.any { it.isLowerCase() } &&
               value.any { it.isDigit() }
    }

    fun masked(): String {
        return "*".repeat(value.length)
    }
}

fun main() {
    val password = Password("MyP@ssw0rd123")

    println(password.isStrong())  // true
    println(password.masked())    // *************
    // println(password.value)    // 직접 접근 가능하지만 권장하지 않음
}
```

---

### 실전 예제: 도메인 모델

```kotlin
@JvmInline
value class OrderId(val value: String)

@JvmInline
value class ProductId(val value: String)

@JvmInline
value class Quantity(val value: Int) {
    init {
        require(value > 0) { "Quantity must be positive" }
    }

    operator fun plus(other: Quantity) = Quantity(value + other.value)
    operator fun times(price: Double) = value * price
}

@JvmInline
value class Price(val value: Double) {
    init {
        require(value >= 0) { "Price must be non-negative" }
    }
}

data class OrderItem(
    val productId: ProductId,
    val quantity: Quantity,
    val price: Price
) {
    fun totalPrice() = quantity.value * price.value
}

data class Order(
    val id: OrderId,
    val items: List<OrderItem>
) {
    fun totalAmount() = items.sumOf { it.totalPrice() }
}

fun main() {
    val order = Order(
        id = OrderId("ORD-001"),
        items = listOf(
            OrderItem(ProductId("PROD-1"), Quantity(2), Price(10.0)),
            OrderItem(ProductId("PROD-2"), Quantity(1), Price(25.0))
        )
    )

    println("Order total: ${order.totalAmount()}")  // Order total: 45.0
}
```

---

## 11.5 Java와 비교: 특수 클래스 활용

### Enum 비교

**Java**:
```java
public enum Status {
    PENDING("대기"),
    APPROVED("승인"),
    REJECTED("거절");

    private final String description;

    Status(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }
}
```

**Kotlin**:
```kotlin
enum class Status(val description: String) {
    PENDING("대기"),
    APPROVED("승인"),
    REJECTED("거절")
}

// 사용이 더 간단
println(Status.PENDING.description)
```

---

### Sealed Class vs Java

Java에는 sealed class가 없습니다 (Java 17+에서 추가됨).

**Java 17+ sealed class**:
```java
public sealed interface Result
    permits Success, Error, Loading {
}

public final class Success implements Result {
    private final String data;
    public Success(String data) { this.data = data; }
    public String getData() { return data; }
}

public final class Error implements Result {
    private final String message;
    public Error(String message) { this.message = message; }
    public String getMessage() { return message; }
}

public final class Loading implements Result { }
```

**Kotlin**:
```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
    object Loading : Result()
}
```

---

## 실전 팁

### 💡 Tip 1: Sealed Class로 상태 모델링

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Failure(val error: Throwable) : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
    object Idle : NetworkResult<Nothing>()
}

fun <T> handleNetworkResult(result: NetworkResult<T>) {
    when (result) {
        is NetworkResult.Success -> processData(result.data)
        is NetworkResult.Failure -> showError(result.error)
        NetworkResult.Loading -> showLoading()
        NetworkResult.Idle -> hideLoading()
    }
}
```

---

### 💡 Tip 2: Value Class로 도메인 모델 강화

```kotlin
@JvmInline
value class EmailAddress(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email" }
    }
}

@JvmInline
value class PhoneNumber(val value: String) {
    init {
        require(value.matches(Regex("\\d{3}-\\d{4}-\\d{4}"))) {
            "Invalid phone format"
        }
    }
}

// 컴파일 타임에 타입 체크
fun sendNotification(email: EmailAddress, phone: PhoneNumber) {
    // ...
}
```

---

### 💡 Tip 3: Enum으로 상수 그룹화

```kotlin
enum class HttpMethod {
    GET, POST, PUT, DELETE, PATCH;

    fun isModifying() = this in listOf(POST, PUT, DELETE, PATCH)
}

enum class LogLevel(val priority: Int) {
    DEBUG(1), INFO(2), WARN(3), ERROR(4);

    fun shouldLog(minLevel: LogLevel) = priority >= minLevel.priority
}
```

---

## 연습 문제

### 문제 1: Sealed Class로 결제 상태 모델링

<details>
<summary>정답 보기</summary>

```kotlin
sealed class PaymentStatus {
    object Pending : PaymentStatus()
    data class Processing(val transactionId: String) : PaymentStatus()
    data class Completed(val transactionId: String, val amount: Double) : PaymentStatus()
    data class Failed(val reason: String) : PaymentStatus()
    object Cancelled : PaymentStatus()
}

fun handlePayment(status: PaymentStatus): String {
    return when (status) {
        PaymentStatus.Pending -> "결제 대기 중"
        is PaymentStatus.Processing -> "결제 처리 중: ${status.transactionId}"
        is PaymentStatus.Completed -> "결제 완료: ${status.amount}원 (${status.transactionId})"
        is PaymentStatus.Failed -> "결제 실패: ${status.reason}"
        PaymentStatus.Cancelled -> "결제 취소됨"
    }
}

fun main() {
    val statuses = listOf(
        PaymentStatus.Pending,
        PaymentStatus.Processing("TXN-001"),
        PaymentStatus.Completed("TXN-001", 50000.0),
        PaymentStatus.Failed("카드 한도 초과"),
        PaymentStatus.Cancelled
    )

    statuses.forEach { status ->
        println(handlePayment(status))
    }
}
```
</details>

### 문제 2: Value Class로 측정 단위 구현

<details>
<summary>정답 보기</summary>

```kotlin
@JvmInline
value class Meter(val value: Double) {
    fun toKilometer() = Kilometer(value / 1000)
    fun toCentimeter() = Centimeter(value * 100)
}

@JvmInline
value class Kilometer(val value: Double) {
    fun toMeter() = Meter(value * 1000)
}

@JvmInline
value class Centimeter(val value: Double) {
    fun toMeter() = Meter(value / 100)
}

fun main() {
    val distance = Meter(1500.0)

    println("${distance.value}m")
    println("${distance.toKilometer().value}km")
    println("${distance.toCentimeter().value}cm")

    val km = Kilometer(5.0)
    println("${km.toMeter().value}m")
}
```
</details>

### 문제 3: Enum으로 요일과 메소드 구현

<details>
<summary>정답 보기</summary>

```kotlin
enum class DayOfWeek(val korean: String) {
    MONDAY("월요일"),
    TUESDAY("화요일"),
    WEDNESDAY("수요일"),
    THURSDAY("목요일"),
    FRIDAY("금요일"),
    SATURDAY("토요일"),
    SUNDAY("일요일");

    fun isWeekend() = this in listOf(SATURDAY, SUNDAY)

    fun isWeekday() = !isWeekend()

    fun next(): DayOfWeek {
        val values = values()
        return values[(ordinal + 1) % values.size]
    }
}

fun main() {
    println(DayOfWeek.MONDAY.korean)  // 월요일
    println(DayOfWeek.SATURDAY.isWeekend())  // true
    println(DayOfWeek.FRIDAY.next())  // SATURDAY
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **Enum Class**
   - 고정된 상수 집합
   - 프로퍼티와 메소드 가질 수 있음

2. **Sealed Class**
   - 제한된 타입 계층
   - when에서 else 불필요
   - 상태 모델링에 최적

3. **Nested vs Inner**
   - Nested: 외부 참조 없음 (기본, 안전)
   - Inner: `inner` 키워드, 외부 참조

4. **Value Class**
   - 타입 안전성
   - 런타임 오버헤드 없음
   - `@JvmInline` 필요

---

## Part 4 완료!

Part 4: 객체지향 프로그래밍을 모두 마쳤습니다!

**배운 내용**:
- ✅ Chapter 9: 상속과 인터페이스
- ✅ Chapter 10: 가시성 제어자
- ✅ Chapter 11: 특별한 클래스들

---

## 다음 Part 예고

**Part 5: 고급 기능**에서 다룰 내용:
- Chapter 12: 제네릭
- Chapter 13: 델리게이션
- Chapter 14: 연산자 오버로딩

---

[← 이전: Chapter 10. 가시성 제어자](chapter10-visibility-modifiers.md) | [다음: Chapter 12. 제네릭 →](../part5-advanced/chapter12-generics.md)