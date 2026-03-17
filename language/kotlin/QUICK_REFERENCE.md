# Kotlin 빠른 참조 가이드

**Java 개발자를 위한 실무 치트시트**

---

## 🔹 기본 문법

### 변수
```kotlin
val immutable = "Cannot change"  // final
var mutable = "Can change"       // non-final
```

### Null Safety
```kotlin
val name: String? = null            // Nullable
val length = name?.length           // Safe call
val length2 = name?.length ?: 0     // Elvis operator
val length3 = name!!.length         // Not-null assertion (위험!)
```

### String Template
```kotlin
val name = "Kotlin"
println("Hello, $name")             // Hello, Kotlin
println("Length: ${name.length}")   // Length: 6
```

---

## 🔹 함수

### 기본 함수
```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

// 표현식 본문
fun add(a: Int, b: Int) = a + b
```

### 기본 매개변수
```kotlin
fun greet(name: String = "Guest") = "Hello, $name"

greet()           // Hello, Guest
greet("Alice")    // Hello, Alice
```

### 확장 함수
```kotlin
fun String.isEmail() = contains("@")

"test@example.com".isEmail()  // true
```

---

## 🔹 클래스

### 기본 클래스
```kotlin
class User(val name: String, var age: Int)

val user = User("Alice", 30)
println(user.name)  // Getter 자동
user.age = 31       // Setter 자동
```

### Data Class
```kotlin
data class User(val id: Long, val name: String)

val user = User(1, "Alice")
val copy = user.copy(name = "Bob")
println(user)  // User(id=1, name=Alice)
```

### Object (싱글톤)
```kotlin
object Config {
    const val MAX_SIZE = 100
}

Config.MAX_SIZE
```

---

## 🔹 컬렉션

### 생성
```kotlin
val list = listOf(1, 2, 3)           // 불변
val mutableList = mutableListOf(1, 2, 3)  // 가변

val map = mapOf("a" to 1, "b" to 2)
val set = setOf(1, 2, 3)
```

### 연산
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

numbers.filter { it % 2 == 0 }      // [2, 4]
numbers.map { it * 2 }              // [2, 4, 6, 8, 10]
numbers.find { it > 3 }             // 4
numbers.any { it > 3 }              // true
numbers.sum()                       // 15
```

---

## 🔹 제어 흐름

### if 표현식
```kotlin
val max = if (a > b) a else b
```

### when (switch 대체)
```kotlin
when (x) {
    1 -> println("One")
    2, 3 -> println("Two or Three")
    in 4..10 -> println("4 to 10")
    else -> println("Other")
}

val result = when {
    x > 0 -> "Positive"
    x < 0 -> "Negative"
    else -> "Zero"
}
```

### for 루프
```kotlin
for (i in 1..5) { }              // 1, 2, 3, 4, 5
for (i in 1 until 5) { }         // 1, 2, 3, 4
for (i in 5 downTo 1) { }        // 5, 4, 3, 2, 1
for (i in 1..10 step 2) { }      // 1, 3, 5, 7, 9

for ((index, value) in list.withIndex()) { }
```

---

## 🔹 람다

### 기본
```kotlin
val sum = { a: Int, b: Int -> a + b }
sum(2, 3)  // 5
```

### it 사용
```kotlin
listOf(1, 2, 3).filter { it % 2 == 0 }
```

### 고차 함수
```kotlin
fun operate(a: Int, b: Int, op: (Int, Int) -> Int) = op(a, b)

operate(2, 3) { a, b -> a + b }  // 5
```

---

## 🔹 스코프 함수

### let
```kotlin
val length = name?.let {
    println("Name: $it")
    it.length
}
```

### run
```kotlin
val result = user.run {
    println("Name: $name")
    age
}
```

### apply
```kotlin
val user = User().apply {
    name = "Alice"
    age = 30
}
```

### also
```kotlin
val numbers = mutableListOf(1, 2, 3)
    .also { println("Original: $it") }
    .apply { add(4) }
```

### with
```kotlin
with(user) {
    println(name)
    println(age)
}
```

---

## 🔹 코루틴

### 기본
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        delay(1000)
        println("World")
    }
    println("Hello")
}
```

### suspend 함수
```kotlin
suspend fun fetchData(): String {
    delay(1000)
    return "Data"
}
```

### async/await
```kotlin
suspend fun loadData() = coroutineScope {
    val data1 = async { fetchData1() }
    val data2 = async { fetchData2() }
    combine(data1.await(), data2.await())
}
```

### Flow
```kotlin
fun numbers() = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

runBlocking {
    numbers().collect { println(it) }
}
```

---

## 🔹 Spring Boot

### Bean 정의
```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {
    fun findAll() = userRepository.findAll()
}
```

### Controller
```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {
    @GetMapping
    fun getAll() = userService.findAll()

    @GetMapping("/{id}")
    suspend fun getById(@PathVariable id: Long) =
        userService.findById(id)
}
```

### Entity
```kotlin
@Entity
class User(
    var name: String,
    var email: String,

    @Id @GeneratedValue
    var id: Long = 0
)
```

### DTO 변환
```kotlin
data class UserDto(val id: Long, val name: String)

fun User.toDto() = UserDto(id, name)

// Service
fun findAll(): List<UserDto> =
    userRepository.findAll().map { it.toDto() }
```

---

## 🔹 테스트

### JUnit 5
```kotlin
@Test
fun `test user creation`() {
    val user = User("Alice", 30)
    assertEquals("Alice", user.name)
}
```

### MockK
```kotlin
val mock = mockk<UserRepository>()
every { mock.findById(1) } returns user

verify { mock.findById(1) }
```

### 코루틴 테스트
```kotlin
@Test
fun `test suspend function`() = runTest {
    val result = fetchData()
    assertEquals("Data", result)
}
```

---

## 🔹 자주 쓰는 패턴

### Null 안전 호출
```kotlin
// Java
String email = null;
if (user != null && user.getEmail() != null) {
    email = user.getEmail().toUpperCase();
}

// Kotlin
val email = user?.email?.uppercase()
```

### Elvis + throw
```kotlin
val user = userRepository.findById(id)
    ?: throw UserNotFoundException("Not found")
```

### let + Elvis
```kotlin
val result = value?.let { process(it) } ?: defaultValue
```

### run + return
```kotlin
val result = user.run {
    if (age >= 18) "Adult" else "Minor"
}
```

### apply로 초기화
```kotlin
val config = Config().apply {
    host = "localhost"
    port = 8080
    timeout = 30
}
```

---

## 🔹 Gradle (build.gradle.kts)

### 플러그인
```kotlin
plugins {
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    kotlin("plugin.jpa") version "1.9.21"
}
```

### 의존성
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    testImplementation("io.mockk:mockk:1.13.8")
}
```

---

## 🔹 유용한 확장 함수

### String
```kotlin
"hello".capitalize()              // Hello
"  text  ".trim()                 // text
"a,b,c".split(",")               // [a, b, c]
"123".toIntOrNull()              // 123 or null
```

### Collection
```kotlin
list.first()
list.last()
list.firstOrNull()
list.getOrNull(index)
list.takeIf { it.isNotEmpty() }
list.takeUnless { it.isEmpty() }
```

---

## 🔹 자주 하는 실수

### ❌ 잘못된 코드
```kotlin
// 1. data class를 Entity로 사용
@Entity
data class User(...)  // JPA 문제 발생

// 2. !! 남용
val name = user!!.name!!  // NPE 위험

// 3. val로 컬렉션 선언 후 add
val list = listOf(1, 2, 3)
list.add(4)  // 컴파일 에러

// 4. GlobalScope 사용
GlobalScope.launch { }  // 생명주기 관리 어려움
```

### ✅ 올바른 코드
```kotlin
// 1. Entity는 일반 class
@Entity
class User(...)

// 2. ?. 또는 ?: 사용
val name = user?.name ?: "Unknown"

// 3. mutableListOf 사용
val list = mutableListOf(1, 2, 3)
list.add(4)

// 4. CoroutineScope 사용
viewModelScope.launch { }
```

---

## 🔹 단축키 (IntelliJ IDEA)

| 기능 | 단축키 (Mac) | 단축키 (Windows) |
|------|--------------|------------------|
| 코드 자동완성 | `Ctrl + Space` | `Ctrl + Space` |
| Quick Fix | `⌥ + Enter` | `Alt + Enter` |
| 리팩토링 메뉴 | `⌃ + T` | `Ctrl + Alt + Shift + T` |
| 코드 포맷 | `⌥ + ⌘ + L` | `Ctrl + Alt + L` |
| Import 정리 | `⌥ + ⌃ + O` | `Ctrl + Alt + O` |

---

## 🔹 마무리

**자주 참조할 링크**:
- [Kotlin 공식 문서](https://kotlinlang.org/docs/home.html)
- [Spring Kotlin 가이드](https://spring.io/guides/tutorials/spring-boot-kotlin/)
- [Coroutines 가이드](https://kotlinlang.org/docs/coroutines-guide.html)

**팁**:
1. IntelliJ IDEA 적극 활용
2. Java → Kotlin 변환기 사용 (`⌘ + ⌥ + Shift + K`)
3. 공식 문서 즐겨찾기
4. Kotlin Slack 커뮤니티 참여

---

[전체 학습 노트](README.md) | [요약](SUMMARY.md)