# Chapter 4. 함수

## 4.1 함수 선언과 호출

### 기본 문법

```kotlin
fun 함수명(파라미터명: 타입): 반환타입 {
    // 함수 본문
    return 반환값
}
```

**예시**:
```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

// 호출
val result = add(10, 20)
println(result)  // 30
```

---

### Java와 비교

**Java**:
```java
public int add(int a, int b) {
    return a + b;
}
```

**Kotlin**:
```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

**차이점**:
- ✅ `public` 생략 (기본이 public)
- ✅ 타입이 파라미터 뒤에 위치
- ✅ `fun` 키워드 사용

---

### 반환 타입이 없는 함수

**Unit**: Java의 `void`와 유사

```kotlin
fun printSum(a: Int, b: Int): Unit {
    println("$a + $b = ${a + b}")
}

// Unit은 생략 가능
fun printSum(a: Int, b: Int) {
    println("$a + $b = ${a + b}")
}
```

**Java와 비교**:
```java
// Java
public void printSum(int a, int b) {
    System.out.println(a + " + " + b + " = " + (a + b));
}

// Kotlin
fun printSum(a: Int, b: Int) {
    println("$a + $b = ${a + b}")
}
```

---

## 4.2 파라미터와 반환 타입

### 여러 파라미터

```kotlin
fun createUser(
    name: String,
    age: Int,
    email: String,
    isActive: Boolean
): String {
    return "User(name=$name, age=$age, email=$email, active=$isActive)"
}

val user = createUser("홍길동", 30, "hong@example.com", true)
```

---

### 반환 타입 추론

**명시적 반환 타입**:
```kotlin
fun multiply(a: Int, b: Int): Int {
    return a * b
}
```

**반환 타입 생략 가능** (단일 표현식 함수):
```kotlin
fun multiply(a: Int, b: Int) = a * b  // Int로 추론
```

---

## 4.3 기본 파라미터 값

Java에는 없는 Kotlin의 강력한 기능입니다!

### 기본값 지정

```kotlin
fun greet(name: String, message: String = "안녕하세요") {
    println("$message, $name님!")
}

// 호출
greet("홍길동")                    // 안녕하세요, 홍길동님!
greet("홍길동", "환영합니다")      // 환영합니다, 홍길동님!
```

---

### 여러 기본값

```kotlin
fun createConnection(
    host: String = "localhost",
    port: Int = 8080,
    timeout: Int = 3000,
    secure: Boolean = false
) {
    println("연결: $host:$port (timeout=${timeout}ms, secure=$secure)")
}

// 다양한 호출 방법
createConnection()
// 출력: 연결: localhost:8080 (timeout=3000ms, secure=false)

createConnection("example.com")
// 출력: 연결: example.com:8080 (timeout=3000ms, secure=false)

createConnection("example.com", 443, 5000, true)
// 출력: 연결: example.com:443 (timeout=5000ms, secure=true)
```

---

### Java와 비교

**Java** - 메소드 오버로딩 필요:
```java
public void createConnection() {
    createConnection("localhost", 8080, 3000, false);
}

public void createConnection(String host) {
    createConnection(host, 8080, 3000, false);
}

public void createConnection(String host, int port) {
    createConnection(host, port, 3000, false);
}

public void createConnection(String host, int port, int timeout, boolean secure) {
    System.out.println("연결: " + host + ":" + port +
                      " (timeout=" + timeout + "ms, secure=" + secure + ")");
}
```

**Kotlin** - 하나의 함수로 해결:
```kotlin
fun createConnection(
    host: String = "localhost",
    port: Int = 8080,
    timeout: Int = 3000,
    secure: Boolean = false
) {
    println("연결: $host:$port (timeout=${timeout}ms, secure=$secure)")
}
```

---

## 4.4 이름있는 인자 (Named Arguments)

### 기본 사용법

```kotlin
fun createUser(name: String, age: Int, email: String) {
    println("User: $name, $age, $email")
}

// 일반 호출
createUser("홍길동", 30, "hong@example.com")

// 이름있는 인자로 호출
createUser(
    name = "홍길동",
    age = 30,
    email = "hong@example.com"
)

// 순서 변경 가능!
createUser(
    email = "hong@example.com",
    name = "홍길동",
    age = 30
)
```

---

### 일부만 이름 지정

```kotlin
fun formatText(
    text: String,
    prefix: String = "[",
    suffix: String = "]",
    uppercase: Boolean = false
): String {
    val result = if (uppercase) text.uppercase() else text
    return "$prefix$result$suffix"
}

// 중간 파라미터만 변경
val result = formatText("hello", suffix = ")", uppercase = true)
println(result)  // [HELLO)
```

---

### 가독성 향상

**이름 없이**:
```kotlin
sendEmail("hong@example.com", "제목", "내용", true, false, 5)
// 마지막 3개 파라미터가 무엇을 의미하는지 불명확
```

**이름과 함께**:
```kotlin
sendEmail(
    to = "hong@example.com",
    subject = "제목",
    body = "내용",
    isHtml = true,
    saveToSent = false,
    retryCount = 5
)
// 매우 명확함!
```

---

## 4.5 단일 표현식 함수

함수 본문이 단일 표현식이면 중괄호와 return을 생략할 수 있습니다.

### 기본 형태

**일반 함수**:
```kotlin
fun double(x: Int): Int {
    return x * 2
}
```

**단일 표현식 함수**:
```kotlin
fun double(x: Int): Int = x * 2

// 반환 타입도 생략 가능
fun double(x: Int) = x * 2
```

---

### 다양한 예제

```kotlin
// 최댓값
fun max(a: Int, b: Int) = if (a > b) a else b

// 절댓값
fun abs(x: Int) = if (x >= 0) x else -x

// 문자열 길이 체크
fun isLongText(text: String) = text.length > 100

// when 표현식
fun getStatus(code: Int) = when (code) {
    200 -> "OK"
    404 -> "Not Found"
    500 -> "Server Error"
    else -> "Unknown"
}

// 계산식
fun calculateArea(width: Double, height: Double) = width * height

// 문자열 포매팅
fun formatName(first: String, last: String) = "$last $first"
```

---

### Java와 비교

**Java**:
```java
public int max(int a, int b) {
    return a > b ? a : b;
}

public String formatName(String first, String last) {
    return last + " " + first;
}
```

**Kotlin**:
```kotlin
fun max(a: Int, b: Int) = if (a > b) a else b

fun formatName(first: String, last: String) = "$last $first"
```

---

## 4.6 확장 함수 (Extension Functions)

Kotlin의 가장 강력한 기능 중 하나입니다!

### 개념

기존 클래스에 **새로운 함수를 추가**할 수 있습니다 (클래스 수정 없이!).

### 기본 문법

```kotlin
fun 타입.함수명(파라미터): 반환타입 {
    // this는 확장되는 객체를 가리킴
}
```

---

### String 확장

```kotlin
// String에 이메일 검증 함수 추가
fun String.isValidEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

// 사용
val email = "hong@example.com"
println(email.isValidEmail())  // true

val invalid = "notanemail"
println(invalid.isValidEmail())  // false
```

**this 생략 가능**:
```kotlin
fun String.isValidEmail(): Boolean {
    return contains("@") && contains(".")
}
```

---

### Int 확장

```kotlin
// Int에 짝수 판별 함수 추가
fun Int.isEven(): Boolean = this % 2 == 0

fun Int.isOdd(): Boolean = this % 2 != 0

// 사용
println(4.isEven())   // true
println(5.isOdd())    // true

// 실전 예제
for (i in 1..10) {
    if (i.isEven()) {
        println("$i is even")
    }
}
```

---

### List 확장

```kotlin
// List에 두 번째 요소 가져오는 함수
fun <T> List<T>.secondOrNull(): T? {
    return if (this.size >= 2) this[1] else null
}

// 사용
val numbers = listOf(1, 2, 3)
println(numbers.secondOrNull())  // 2

val single = listOf(1)
println(single.secondOrNull())   // null
```

---

### 실전 예제

```kotlin
// 문자열 마스킹
fun String.maskEmail(): String {
    val parts = this.split("@")
    if (parts.size != 2) return this

    val username = parts[0]
    val domain = parts[1]

    val masked = username.take(2) + "***"
    return "$masked@$domain"
}

// 사용
println("hong@example.com".maskEmail())  // ho***@example.com

// 숫자 포매팅
fun Int.toWon(): String = "${this}원"

println(10000.toWon())  // 10000원

// 날짜 계산
fun Int.daysFromNow(): String = "D+${this}"

println(7.daysFromNow())  // D+7
```

---

### Java와 비교

**Java** - 유틸리티 클래스 필요:
```java
public class StringUtils {
    public static boolean isValidEmail(String email) {
        return email.contains("@") && email.contains(".");
    }
}

// 사용
StringUtils.isValidEmail(email);  // 정적 메소드 호출
```

**Kotlin** - 확장 함수:
```kotlin
fun String.isValidEmail() = contains("@") && contains(".")

// 사용
email.isValidEmail()  // 마치 원래 메소드처럼!
```

---

## 4.7 중위 함수 (Infix Functions)

### 개념

파라미터가 하나인 함수를 **중위 표기법**으로 호출할 수 있게 합니다.

### 기본 사용법

```kotlin
// infix 키워드 사용
infix fun Int.times(str: String): String {
    return str.repeat(this)
}

// 일반 호출
println(3.times("Hello "))  // Hello Hello Hello

// 중위 표기법
println(3 times "Hello ")   // Hello Hello Hello
```

---

### 실전 예제

**Pair 생성**:
```kotlin
// Kotlin 표준 라이브러리의 to 함수
val pair = "name" to "Kotlin"
// 실제로는: val pair = "name".to("Kotlin")

val map = mapOf(
    "name" to "Kotlin",
    "version" to "1.9"
)
```

**커스텀 중위 함수**:
```kotlin
// 범위 체크
infix fun Int.within(range: IntRange): Boolean {
    return this in range
}

val age = 25
if (age within 18..64) {
    println("성인입니다")
}

// 문자열 결합
infix fun String.concat(other: String): String {
    return this + " " + other
}

val result = "Hello" concat "World"
println(result)  // Hello World
```

---

### 주의사항

1. **멤버 함수 또는 확장 함수**여야 함
2. **파라미터가 정확히 1개**여야 함
3. **기본값이나 가변 인자 불가**

```kotlin
// ✅ 올바른 infix 함수
infix fun String.append(text: String) = this + text

// ❌ 잘못된 infix 함수
infix fun combine(a: String, b: String) = a + b  // 멤버/확장 함수 아님
infix fun String.join(a: String, b: String) = this + a + b  // 파라미터 2개
```

---

## 4.8 Java와 비교: 함수 작성법

### 종합 비교

**Java**:
```java
public class MathUtils {
    // 1. 일반 메소드
    public static int add(int a, int b) {
        return a + b;
    }

    // 2. 오버로딩 (기본값 대신)
    public static int add(int a) {
        return add(a, 0);
    }

    // 3. 유틸리티 메소드
    public static boolean isEven(int number) {
        return number % 2 == 0;
    }
}

// 사용
int result = MathUtils.add(1, 2);
boolean even = MathUtils.isEven(4);
```

**Kotlin**:
```kotlin
// 1. Top-level 함수 (클래스 불필요)
fun add(a: Int, b: Int = 0) = a + b

// 2. 확장 함수
fun Int.isEven() = this % 2 == 0

// 사용
val result = add(1, 2)
val result2 = add(1)  // 기본값 사용

val even = 4.isEven()  // 마치 메소드처럼
```

---

### 실전 비교: 문자열 처리

**Java**:
```java
public class StringHelper {
    public static String truncate(String text, int maxLength) {
        return truncate(text, maxLength, "...");
    }

    public static String truncate(String text, int maxLength, String suffix) {
        if (text.length() <= maxLength) {
            return text;
        }
        return text.substring(0, maxLength) + suffix;
    }

    public static boolean isBlank(String text) {
        return text == null || text.trim().isEmpty();
    }
}

// 사용
String result = StringHelper.truncate("Hello World", 5);
boolean blank = StringHelper.isBlank(text);
```

**Kotlin**:
```kotlin
// 확장 함수와 기본 파라미터 활용
fun String.truncate(maxLength: Int, suffix: String = "...") =
    if (length <= maxLength) this
    else substring(0, maxLength) + suffix

fun String?.isBlankOrNull() = this == null || this.isBlank()

// 사용
val result = "Hello World".truncate(5)
val blank = text.isBlankOrNull()
```

---

## 실전 팁

### 💡 Tip 1: 기본 파라미터로 오버로딩 줄이기

**Before (Java 스타일)**:
```kotlin
fun connect(host: String) {
    connect(host, 8080)
}

fun connect(host: String, port: Int) {
    connect(host, port, 3000)
}

fun connect(host: String, port: Int, timeout: Int) {
    println("Connecting to $host:$port (timeout: $timeout)")
}
```

**After (Kotlin 스타일)**:
```kotlin
fun connect(
    host: String,
    port: Int = 8080,
    timeout: Int = 3000
) {
    println("Connecting to $host:$port (timeout: $timeout)")
}
```

---

### 💡 Tip 2: 확장 함수로 가독성 향상

**Before**:
```kotlin
fun isValidUser(user: User): Boolean {
    return user.name.isNotBlank() &&
           user.email.contains("@") &&
           user.age >= 18
}

if (isValidUser(user)) {
    // ...
}
```

**After**:
```kotlin
fun User.isValid(): Boolean {
    return name.isNotBlank() &&
           email.contains("@") &&
           age >= 18
}

if (user.isValid()) {  // 더 자연스러움
    // ...
}
```

---

### 💡 Tip 3: 단일 표현식 함수로 간결하게

**Before**:
```kotlin
fun calculateDiscount(price: Int, rate: Double): Double {
    return price * rate
}

fun isAdult(age: Int): Boolean {
    return age >= 18
}
```

**After**:
```kotlin
fun calculateDiscount(price: Int, rate: Double) = price * rate

fun isAdult(age: Int) = age >= 18
```

---

### 💡 Tip 4: 이름있는 인자로 명확성 확보

```kotlin
// ❌ 무엇을 의미하는지 불명확
createAccount("john", "1234", true, false, 30)

// ✅ 명확함
createAccount(
    username = "john",
    password = "1234",
    isActive = true,
    sendEmail = false,
    maxLoginAttempts = 30
)
```

---

## 연습 문제

### 문제 1: 온도 변환 함수
섭씨를 화씨로 변환하는 확장 함수를 작성하세요.
공식: F = C × 9/5 + 32

<details>
<summary>정답 보기</summary>

```kotlin
fun Double.toFahrenheit(): Double = this * 9 / 5 + 32

fun main() {
    println("${0.0.toFahrenheit()}°F")   // 32.0°F
    println("${100.0.toFahrenheit()}°F") // 212.0°F
    println("${37.0.toFahrenheit()}°F")  // 98.6°F
}
```
</details>

### 문제 2: 문자열 반복 함수
문자열을 n번 반복하는 중위 함수를 작성하세요.

<details>
<summary>정답 보기</summary>

```kotlin
infix fun String.repeat(times: Int): String {
    return this.repeat(times)
}

fun main() {
    println("*" repeat 10)        // **********
    println("Hello " repeat 3)    // Hello Hello Hello
}
```
</details>

### 문제 3: 리스트 확장 함수
리스트의 짝수만 필터링하는 확장 함수를 작성하세요.

<details>
<summary>정답 보기</summary>

```kotlin
fun List<Int>.filterEven(): List<Int> {
    return this.filter { it % 2 == 0 }
}

fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    println(numbers.filterEven())  // [2, 4, 6, 8, 10]
}
```
</details>

### 문제 4: 기본 파라미터 활용
사용자를 생성하는 함수를 작성하세요. isActive는 기본값 true, role은 기본값 "USER"

<details>
<summary>정답 보기</summary>

```kotlin
data class User(
    val name: String,
    val email: String,
    val isActive: Boolean,
    val role: String
)

fun createUser(
    name: String,
    email: String,
    isActive: Boolean = true,
    role: String = "USER"
): User {
    return User(name, email, isActive, role)
}

fun main() {
    val user1 = createUser("홍길동", "hong@example.com")
    println(user1)
    // User(name=홍길동, email=hong@example.com, isActive=true, role=USER)

    val admin = createUser("관리자", "admin@example.com", role = "ADMIN")
    println(admin)
    // User(name=관리자, email=admin@example.com, isActive=true, role=ADMIN)
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **기본 파라미터**
   - Java의 오버로딩 대체
   - 더 간결한 API

2. **이름있는 인자**
   - 가독성 향상
   - 순서 변경 가능

3. **단일 표현식 함수**
   - `= 표현식` 형태
   - return과 중괄호 생략

4. **확장 함수**
   - 기존 클래스에 함수 추가
   - 유틸리티 클래스 불필요

5. **중위 함수**
   - 자연스러운 DSL 작성
   - `to`, `until` 등

---

## 다음 챕터 예고

Chapter 5에서는 **클래스와 객체 기초**를 다룹니다:
- 클래스 선언
- 생성자 (주 생성자, 부 생성자)
- 프로퍼티
- 데이터 클래스
- object 키워드

Kotlin의 클래스는 Java보다 훨씬 간결합니다!

---

[← 이전: Chapter 3. 제어 흐름](chapter3-control-flow.md) | [다음: Chapter 5. 클래스와 객체 기초 →](chapter5-classes-objects.md)