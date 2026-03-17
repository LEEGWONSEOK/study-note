# Chapter 6. Null 안전성

## 개요

Null 안전성은 **Kotlin의 가장 중요한 기능** 중 하나입니다. Java에서 가장 흔한 에러인 `NullPointerException`을 **컴파일 타임에 방지**합니다.

**Tony Hoare** (Null을 발명한 사람):
> "Null 참조를 만든 것은 10억 달러짜리 실수였다"

Kotlin은 이 문제를 타입 시스템으로 해결했습니다!

---

## 6.1 Nullable 타입 (?)

### Non-Null 타입 (기본)

Kotlin에서 모든 타입은 기본적으로 **Null을 허용하지 않습니다**.

```kotlin
var name: String = "Kotlin"
name = null  // 컴파일 에러!
// Null can not be a value of a non-null type String
```

이것이 Java와의 가장 큰 차이점입니다!

---

### Nullable 타입 선언

Null을 허용하려면 타입 뒤에 **`?`를 붙여야** 합니다.

```kotlin
var name: String? = "Kotlin"
name = null  // OK!
```

**타입의 의미**:
- `String`: Null이 될 수 없는 String
- `String?`: Null이 될 수 있는 String

---

### Java와 비교

**Java**:
```java
// Java - 모든 참조 타입은 null 가능
String name = "Java";
name = null;  // OK

Integer number = 42;
number = null;  // OK

// NullPointerException 위험!
int length = name.length();  // NPE 발생!
```

**Kotlin**:
```kotlin
// Kotlin - 명시적으로 구분
var name: String = "Kotlin"
// name = null  // 컴파일 에러!

var nullableName: String? = "Kotlin"
nullableName = null  // OK

// 안전하게 처리
val length = nullableName?.length  // null이면 null 반환
```

---

### Nullable 타입의 제약

Nullable 타입은 **바로 사용할 수 없습니다**.

```kotlin
val name: String? = "Kotlin"

// 컴파일 에러!
// val length = name.length
// Only safe (?.) or non-null asserted (!!.) calls are allowed

// ✅ 올바른 사용법들
val length1 = name?.length        // 안전 호출
val length2 = name!!.length       // Non-null assertion
val length3 = if (name != null) name.length else 0  // 명시적 체크
```

---

## 6.2 안전 호출 연산자 (?.)

### 기본 사용법

**`?.`**: 객체가 null이 아니면 메소드/프로퍼티에 접근, null이면 null 반환

```kotlin
val name: String? = "Kotlin"
val length: Int? = name?.length  // name이 null이 아니므로 6
println(length)  // 6

val nullName: String? = null
val nullLength: Int? = nullName?.length  // null
println(nullLength)  // null
```

---

### 체이닝

안전 호출은 **체이닝**할 수 있습니다.

```kotlin
class Address(val city: String?)
class Company(val address: Address?)
class Person(val company: Company?)

val person: Person? = Person(Company(Address("Seoul")))

// 안전 호출 체이닝
val city: String? = person?.company?.address?.city
println(city)  // Seoul

// 중간에 null이 있으면?
val personWithoutCompany: Person? = Person(null)
val noCity = personWithoutCompany?.company?.address?.city
println(noCity)  // null
```

**Java와 비교**:
```java
// Java - 중첩된 null 체크
String city = null;
if (person != null) {
    Company company = person.getCompany();
    if (company != null) {
        Address address = company.getAddress();
        if (address != null) {
            city = address.getCity();
        }
    }
}

// Kotlin - 간결!
val city = person?.company?.address?.city
```

---

### let 함수와 함께 사용

**let**: null이 아닐 때만 코드 블록 실행

```kotlin
val name: String? = "Kotlin"

name?.let {
    println("이름: $it")
    println("길이: ${it.length}")
}
// 출력:
// 이름: Kotlin
// 길이: 6

val nullName: String? = null
nullName?.let {
    println("이것은 실행되지 않습니다")
}
// 아무것도 출력 안 됨
```

**실전 예제**:
```kotlin
// 사용자가 있을 때만 이메일 전송
fun sendEmail(user: User?) {
    user?.email?.let { email ->
        println("Sending email to $email")
        // 실제 이메일 전송 로직
    }
}
```

---

## 6.3 Elvis 연산자 (?:)

### 기본 사용법

**`?:`**: 좌변이 null이면 우변 반환 (Java의 삼항 연산자와 유사)

```kotlin
val name: String? = null
val result = name ?: "Unknown"
println(result)  // Unknown

val name2: String? = "Kotlin"
val result2 = name2 ?: "Unknown"
println(result2)  // Kotlin
```

---

### 기본값 제공

```kotlin
fun getDisplayName(name: String?): String {
    return name ?: "Anonymous"
}

println(getDisplayName("홍길동"))  // 홍길동
println(getDisplayName(null))     // Anonymous
```

---

### 안전 호출과 함께 사용

가장 일반적인 패턴입니다!

```kotlin
val length = name?.length ?: 0

// 읽는 방법:
// 1. name이 null이 아니면 name.length
// 2. name이 null이면 0
```

**실전 예제**:
```kotlin
class User(
    val name: String?,
    val email: String?,
    val age: Int?
)

fun getUserInfo(user: User?): String {
    val name = user?.name ?: "Unknown"
    val email = user?.email ?: "no-email@example.com"
    val age = user?.age ?: 0

    return "User: $name, Email: $email, Age: $age"
}

val user = User("홍길동", null, 30)
println(getUserInfo(user))
// User: 홍길동, Email: no-email@example.com, Age: 30

println(getUserInfo(null))
// User: Unknown, Email: no-email@example.com, Age: 0
```

---

### early return과 함께

```kotlin
fun processUser(user: User?) {
    val validUser = user ?: return  // user가 null이면 함수 종료

    // 이 시점에서 validUser는 null이 아님이 보장됨
    println("Processing ${validUser.name}")
}

// 예외 던지기
fun validateUser(user: User?) {
    val validUser = user ?: throw IllegalArgumentException("User cannot be null")

    println("Valid user: ${validUser.name}")
}
```

---

### Java와 비교

**Java**:
```java
public String getDisplayName(String name) {
    return name != null ? name : "Unknown";
}

// 또는
public String getDisplayName(String name) {
    if (name != null) {
        return name;
    } else {
        return "Unknown";
    }
}
```

**Kotlin**:
```kotlin
fun getDisplayName(name: String?) = name ?: "Unknown"
```

---

## 6.4 !! 연산자 (Not-null assertion)

### 개념

**`!!`**: "이 값은 절대 null이 아니다"라고 컴파일러에게 단언

**주의**: 만약 null이면 `NullPointerException` 발생!

```kotlin
val name: String? = "Kotlin"
val length: Int = name!!.length  // OK, length = 6

val nullName: String? = null
val crash: Int = nullName!!.length  // NPE 발생!
```

---

### 언제 사용하나?

**사용하지 말아야 할 경우 (대부분)**:
```kotlin
// ❌ 나쁜 예
fun getUserName(userId: String): String {
    val user = findUser(userId)
    return user!!.name  // NPE 위험!
}
```

**사용해도 되는 경우 (드문)**:
```kotlin
// ✅ 로직상 null이 될 수 없음이 확실한 경우
class UserService {
    private var currentUser: User? = null

    fun login(user: User) {
        currentUser = user
    }

    fun getCurrentUserName(): String {
        // login() 후에만 호출되므로 null이 아님이 보장
        return currentUser!!.name
    }
}
```

---

### 대안: 더 안전한 방법

**`!!` 대신 사용할 수 있는 방법들**:

```kotlin
// 1. 안전 호출 + Elvis
val name = user?.name ?: "Unknown"

// 2. let 사용
user?.let {
    println(it.name)
}

// 3. 명시적 체크
if (user != null) {
    println(user.name)
}

// 4. requireNotNull
val validUser = requireNotNull(user) { "User must not be null" }

// 5. checkNotNull
val checkedUser = checkNotNull(user) { "User must not be null" }
```

---

### 실전 팁

```kotlin
// ❌ 이렇게 하지 마세요
val name = user!!.company!!.address!!.city!!

// ✅ 이렇게 하세요
val name = user?.company?.address?.city ?: "Unknown"
```

**원칙**: `!!`를 사용하는 것은 Java의 null 체크 없이 사용하는 것과 동일합니다. 가능한 피하세요!

---

## 6.5 안전한 캐스팅 (as?)

### as vs as?

**`as`**: 캐스팅 실패 시 예외 발생
```kotlin
val obj: Any = "Hello"

val str: String = obj as String  // OK
val num: Int = obj as Int  // ClassCastException!
```

**`as?`**: 캐스팅 실패 시 null 반환 (안전)
```kotlin
val obj: Any = "Hello"

val str: String? = obj as? String  // OK, "Hello"
val num: Int? = obj as? Int        // OK, null (예외 없음)
```

---

### 실전 예제

```kotlin
fun processValue(value: Any) {
    val str = value as? String
    if (str != null) {
        println("문자열: $str (길이: ${str.length})")
    } else {
        println("문자열이 아닙니다")
    }
}

processValue("Hello")  // 문자열: Hello (길이: 5)
processValue(42)       // 문자열이 아닙니다
```

**Elvis 연산자와 함께**:
```kotlin
fun getStringLength(value: Any): Int {
    val str = value as? String ?: return 0
    return str.length
}

println(getStringLength("Hello"))  // 5
println(getStringLength(42))       // 0
```

---

### when과 스마트 캐스팅

더 좋은 방법은 `when`과 `is`를 사용하는 것입니다.

```kotlin
fun describe(obj: Any): String {
    return when (obj) {
        is String -> "문자열 (길이: ${obj.length})"  // 자동 캐스팅!
        is Int -> "정수 ($obj)"
        is List<*> -> "리스트 (크기: ${obj.size})"
        else -> "알 수 없는 타입"
    }
}
```

---

## 6.6 let, run, apply 함수와 Null 처리

### let: null이 아닐 때만 실행

```kotlin
val name: String? = "Kotlin"

// Null 체크 후 처리
name?.let {
    println("이름: $it")
    println("대문자: ${it.uppercase()}")
}

// 여러 작업
val user: User? = getUser()
user?.let {
    validateUser(it)
    sendEmail(it.email)
    log("User processed: ${it.name}")
}
```

---

### run: this로 접근

```kotlin
val user: User? = getUser()

val message = user?.run {
    // this로 user 접근
    validateUser(this)
    "User ${this.name} processed"
} ?: "No user found"

println(message)
```

---

### apply: 객체 초기화

```kotlin
val user: User? = User()

user?.apply {
    name = "홍길동"
    email = "hong@example.com"
    age = 30
}
```

---

### also: 부수 효과

```kotlin
val user: User? = createUser()

user?.also {
    println("User created: ${it.name}")
    logUserCreation(it)
}?.let {
    sendWelcomeEmail(it)
}
```

---

## 6.7 Java와 비교: Null 처리 방식

### 전체 비교

**시나리오**: 사용자 정보에서 도시 이름 가져오기

**Java (전통적인 방식)**:
```java
public String getCityName(User user) {
    if (user == null) {
        return "Unknown";
    }

    Company company = user.getCompany();
    if (company == null) {
        return "Unknown";
    }

    Address address = company.getAddress();
    if (address == null) {
        return "Unknown";
    }

    String city = address.getCity();
    return city != null ? city : "Unknown";
}
```

**Java (Optional 사용)**:
```java
public String getCityName(User user) {
    return Optional.ofNullable(user)
        .map(User::getCompany)
        .map(Company::getAddress)
        .map(Address::getCity)
        .orElse("Unknown");
}
```

**Kotlin**:
```kotlin
fun getCityName(user: User?): String {
    return user?.company?.address?.city ?: "Unknown"
}
```

---

### NullPointerException 비교

**Java**:
```java
// NPE 발생 가능
String name = user.getName();
int length = name.length();  // name이 null이면 NPE!

// 안전하게 처리
String name = user.getName();
int length = 0;
if (name != null) {
    length = name.length();
}
```

**Kotlin**:
```kotlin
// 컴파일 에러 - 실행 전에 잡아냄!
// val name: String = user.name
// val length = name.length

// 올바른 방법
val name: String? = user.name
val length = name?.length ?: 0
```

---

## 실전 팁

### 💡 Tip 1: Nullable보다 Non-null을 기본으로

```kotlin
// ❌ 피하기
data class User(
    val name: String?,
    val email: String?,
    val age: Int?
)

// ✅ 권장
data class User(
    val name: String,
    val email: String,
    val age: Int,
    val phone: String? = null  // 정말 선택적인 경우만
)
```

---

### 💡 Tip 2: 안전 호출 + Elvis 패턴 활용

```kotlin
// ✅ 매우 일반적인 패턴
val displayName = user?.name ?: "Guest"
val email = user?.email ?: "no-email@example.com"
val profileImage = user?.profileUrl ?: DEFAULT_PROFILE_IMAGE

// API 응답 처리
val data = response?.body?.data ?: emptyList()
```

---

### 💡 Tip 3: let으로 null이 아닐 때만 처리

```kotlin
// ✅ 명확하고 안전
user?.let { u ->
    database.save(u)
    logger.info("User saved: ${u.name}")
    return u.id
}

// 여러 nullable 값 처리
val result = value1?.let { v1 ->
    value2?.let { v2 ->
        v1 + v2
    }
}
```

---

### 💡 Tip 4: requireNotNull / checkNotNull 활용

```kotlin
// ✅ 명확한 에러 메시지
fun processUser(user: User?) {
    val validUser = requireNotNull(user) {
        "User must not be null at this point"
    }

    // 이제 validUser는 User 타입 (nullable 아님)
    println(validUser.name)
}

// 빠른 실패 (fail-fast)
class UserService(database: Database?) {
    private val db = requireNotNull(database) {
        "Database must be initialized"
    }
}
```

---

### 💡 Tip 5: !! 연산자는 최대한 피하기

```kotlin
// ❌ 나쁜 예
fun badExample(user: User?) {
    val name = user!!.name
    val company = user!!.company!!.name
}

// ✅ 좋은 예
fun goodExample(user: User?) {
    val name = user?.name ?: return
    val company = user.company?.name ?: "No Company"
}
```

---

## 연습 문제

### 문제 1: 안전한 이메일 가져오기
사용자 객체에서 이메일을 안전하게 가져오는 함수를 작성하세요. 없으면 "no-email@example.com" 반환

<details>
<summary>정답 보기</summary>

```kotlin
data class User(val name: String, val email: String?)

fun getEmail(user: User?): String {
    return user?.email ?: "no-email@example.com"
}

fun main() {
    val user1 = User("홍길동", "hong@example.com")
    val user2 = User("김철수", null)

    println(getEmail(user1))  // hong@example.com
    println(getEmail(user2))  // no-email@example.com
    println(getEmail(null))   // no-email@example.com
}
```
</details>

### 문제 2: 안전한 체이닝
Person → Company → Address → City 구조에서 도시 이름을 안전하게 가져오기

<details>
<summary>정답 보기</summary>

```kotlin
data class Address(val city: String?)
data class Company(val address: Address?)
data class Person(val name: String, val company: Company?)

fun getCityName(person: Person?): String {
    return person?.company?.address?.city ?: "Unknown City"
}

fun main() {
    val person1 = Person("홍길동", Company(Address("Seoul")))
    val person2 = Person("김철수", Company(Address(null)))
    val person3 = Person("이영희", Company(null))
    val person4 = Person("박민수", null)

    println(getCityName(person1))  // Seoul
    println(getCityName(person2))  // Unknown City
    println(getCityName(person3))  // Unknown City
    println(getCityName(person4))  // Unknown City
    println(getCityName(null))     // Unknown City
}
```
</details>

### 문제 3: let을 활용한 처리
사용자가 null이 아닐 때만 환영 메시지를 출력하는 함수

<details>
<summary>정답 보기</summary>

```kotlin
data class User(val name: String, val email: String)

fun welcomeUser(user: User?) {
    user?.let {
        println("환영합니다, ${it.name}님!")
        println("가입하신 이메일: ${it.email}")
    } ?: println("사용자 정보가 없습니다")
}

fun main() {
    val user = User("홍길동", "hong@example.com")
    welcomeUser(user)
    // 환영합니다, 홍길동님!
    // 가입하신 이메일: hong@example.com

    welcomeUser(null)
    // 사용자 정보가 없습니다
}
```
</details>

### 문제 4: 안전한 캐스팅
Any 타입을 받아서 Int로 변환, 실패하면 0 반환

<details>
<summary>정답 보기</summary>

```kotlin
fun toIntOrZero(value: Any?): Int {
    return value as? Int ?: 0
}

fun main() {
    println(toIntOrZero(42))       // 42
    println(toIntOrZero("hello"))  // 0
    println(toIntOrZero(null))     // 0
    println(toIntOrZero(3.14))     // 0
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **Nullable 타입**
   - `Type?`: Null 허용
   - `Type`: Null 불허 (기본)

2. **안전 호출 (`?.`)**
   - null이면 null 반환
   - 체이닝 가능

3. **Elvis 연산자 (`?:`)**
   - 기본값 제공
   - early return 가능

4. **`!!` 연산자**
   - 가능한 피하기
   - NPE 위험

5. **let, run, apply**
   - null이 아닐 때만 실행
   - 안전한 처리

6. **Java 대비 장점**
   - 컴파일 타임에 NPE 방지
   - 명시적인 null 처리
   - 간결한 문법

---

## 다음 챕터 예고

Chapter 7에서는 **컬렉션**을 다룹니다:
- List, Set, Map
- 불변 vs 가변 컬렉션
- 컬렉션 연산 (filter, map, reduce)
- Sequence

Kotlin의 컬렉션은 Java보다 훨씬 강력하고 사용하기 쉽습니다!

---

[← 이전: Chapter 5. 클래스와 객체 기초](../part02-basic-syntax/05-클래스와_객체_기초.md) | [다음: Chapter 7. 컬렉션 →](07-컬렉션.md)