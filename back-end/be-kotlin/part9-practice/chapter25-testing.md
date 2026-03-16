# Chapter 25. 테스팅 (JUnit 5, MockK)

## 25.1 테스팅 개요

### Java vs Kotlin 테스팅 도구

| | Java | Kotlin |
|--|------|--------|
| 테스트 프레임워크 | JUnit 5 | JUnit 5 |
| Mocking | Mockito | MockK |
| Assertion | AssertJ | Kotest, AssertJ |
| 특징 | - | 더 간결한 DSL |

---

## 25.2 의존성 설정

**build.gradle.kts**:
```kotlin
dependencies {
    // JUnit 5
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(module = "mockito-core")  // MockK 사용 위해 제외
    }

    // MockK (Kotlin용 Mocking 라이브러리)
    testImplementation("io.mockk:mockk:1.13.8")

    // Kotest (선택)
    testImplementation("io.kotest:kotest-runner-junit5:5.7.2")
    testImplementation("io.kotest:kotest-assertions-core:5.7.2")

    // Coroutine Test
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## 25.3 기본 단위 테스트

### JUnit 5 기본

```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.DisplayName

class CalculatorTest {

    private lateinit var calculator: Calculator

    @BeforeEach
    fun setup() {
        calculator = Calculator()
    }

    @Test
    @DisplayName("두 숫자를 더한다")
    fun `add two numbers`() {
        val result = calculator.add(2, 3)
        assertEquals(5, result)
    }

    @Test
    fun `subtract two numbers`() {
        val result = calculator.subtract(5, 3)
        assertEquals(2, result)
    }

    @Test
    fun `divide by zero throws exception`() {
        assertThrows<ArithmeticException> {
            calculator.divide(10, 0)
        }
    }
}

class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun subtract(a: Int, b: Int) = a - b
    fun divide(a: Int, b: Int): Int {
        if (b == 0) throw ArithmeticException("Division by zero")
        return a / b
    }
}
```

---

### Kotlin 스타일 Assertion

```kotlin
import kotlin.test.*

class UserTest {

    @Test
    fun `user creation`() {
        val user = User(name = "Alice", age = 30)

        // Kotlin test assertions
        assertTrue(user.name == "Alice")
        assertFalse(user.age < 18)
        assertNotNull(user)
        assertEquals(30, user.age)
    }

    @Test
    fun `user validation`() {
        val user = User(name = "", age = -1)

        assertFails {
            user.validate()
        }
    }
}
```

---

## 25.4 MockK 사용법

### 기본 Mocking

```kotlin
import io.mockk.*
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class UserServiceTest {

    private val userRepository: UserRepository = mockk()
    private val userService = UserService(userRepository)

    @Test
    fun `findById returns user dto`() {
        // Given
        val user = User(id = 1, name = "Alice", email = "alice@example.com", age = 30)

        every { userRepository.findById(1) } returns user

        // When
        val result = userService.findById(1)

        // Then
        assertEquals("Alice", result?.name)
        verify(exactly = 1) { userRepository.findById(1) }
    }

    @Test
    fun `findById returns null when not found`() {
        // Given
        every { userRepository.findById(999) } returns null

        // When
        val result = userService.findById(999)

        // Then
        assertEquals(null, result)
    }
}
```

---

### Argument Matching

```kotlin
class UserServiceTest {

    private val userRepository: UserRepository = mockk()
    private val userService = UserService(userRepository)

    @Test
    fun `create saves user`() {
        // Given
        val request = CreateUserRequest("Alice", "alice@example.com", 30)
        val savedUser = User(id = 1, name = "Alice", email = "alice@example.com", age = 30)

        every { userRepository.existsByEmail(any()) } returns false
        every { userRepository.save(any()) } returns savedUser

        // When
        val result = userService.create(request)

        // Then
        assertEquals("Alice", result.name)
        verify { userRepository.existsByEmail("alice@example.com") }
        verify { userRepository.save(match { it.name == "Alice" }) }
    }

    @Test
    fun `create throws exception when email exists`() {
        // Given
        val request = CreateUserRequest("Alice", "alice@example.com", 30)
        every { userRepository.existsByEmail("alice@example.com") } returns true

        // When & Then
        assertThrows<DuplicateEmailException> {
            userService.create(request)
        }

        verify { userRepository.existsByEmail("alice@example.com") }
        verify(exactly = 0) { userRepository.save(any()) }
    }
}
```

---

### Relaxed Mock

```kotlin
@Test
fun `using relaxed mock`() {
    // relaxed mock은 설정하지 않은 메서드도 기본값 반환
    val userRepository: UserRepository = mockk(relaxed = true)
    val userService = UserService(userRepository)

    // 별도 설정 없이 호출 가능 (기본값 반환)
    val result = userService.findById(1)

    assertEquals(null, result)  // 기본값: null
}
```

---

### Spy

```kotlin
@Test
fun `using spy`() {
    val realService = UserService(mockk(relaxed = true))
    val spyService = spyk(realService)

    every { spyService.findById(1) } returns UserDto(1, "Mocked", "mock@example.com", 25, LocalDateTime.now())

    // findById(1)은 모킹됨
    val result1 = spyService.findById(1)
    assertEquals("Mocked", result1?.name)

    // findAll()은 실제 메서드 호출
    val result2 = spyService.findAll()
}
```

---

## 25.5 Spring Boot 테스트

### @SpringBootTest

```kotlin
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.beans.factory.annotation.Autowired
import org.junit.jupiter.api.Test

@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private lateinit var userService: UserService

    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `create and find user`() {
        // Given
        val request = CreateUserRequest("Alice", "alice@test.com", 30)

        // When
        val created = userService.create(request)
        val found = userService.findById(created.id)

        // Then
        assertEquals("Alice", found?.name)

        // Cleanup
        userRepository.deleteById(created.id)
    }
}
```

---

### @WebMvcTest (Controller 테스트)

```kotlin
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*
import com.ninjasquad.springmockk.MockkBean
import com.fasterxml.jackson.databind.ObjectMapper

@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @MockkBean
    private lateinit var userService: UserService

    @Test
    fun `GET users returns list`() {
        // Given
        val users = listOf(
            UserDto(1, "Alice", "alice@example.com", 30, LocalDateTime.now()),
            UserDto(2, "Bob", "bob@example.com", 25, LocalDateTime.now())
        )
        every { userService.findAll() } returns users

        // When & Then
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$[0].name").value("Alice"))
            .andExpect(jsonPath("$[1].name").value("Bob"))
    }

    @Test
    fun `POST users creates user`() {
        // Given
        val request = CreateUserRequest("Alice", "alice@example.com", 30)
        val created = UserDto(1, "Alice", "alice@example.com", 30, LocalDateTime.now())

        every { userService.create(request) } returns created

        // When & Then
        mockMvc.perform(
            post("/api/users")
                .contentType("application/json")
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.name").value("Alice"))
    }

    @Test
    fun `GET user by id returns 404 when not found`() {
        // Given
        every { userService.findById(999) } throws UserNotFoundException("User not found")

        // When & Then
        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound)
    }
}
```

---

### @DataJpaTest (Repository 테스트)

```kotlin
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.beans.factory.annotation.Autowired

@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `save and find user`() {
        // Given
        val user = User(name = "Alice", email = "alice@test.com", age = 30)

        // When
        val saved = userRepository.save(user)
        val found = userRepository.findById(saved.id).orElse(null)

        // Then
        assertNotNull(found)
        assertEquals("Alice", found?.name)
    }

    @Test
    fun `findByEmail returns user`() {
        // Given
        val user = User(name = "Alice", email = "alice@test.com", age = 30)
        userRepository.save(user)

        // When
        val found = userRepository.findByEmail("alice@test.com")

        // Then
        assertNotNull(found)
        assertEquals("Alice", found?.name)
    }

    @Test
    fun `existsByEmail returns true when exists`() {
        // Given
        val user = User(name = "Alice", email = "alice@test.com", age = 30)
        userRepository.save(user)

        // When
        val exists = userRepository.existsByEmail("alice@test.com")

        // Then
        assertTrue(exists)
    }
}
```

---

## 25.6 코루틴 테스트

### runTest

```kotlin
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.delay
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class CoroutineServiceTest {

    @Test
    fun `suspend function test`() = runTest {
        val service = CoroutineService()
        val result = service.fetchData()

        assertEquals("Data", result)
    }

    @Test
    fun `test with delay`() = runTest {
        val service = CoroutineService()

        val start = currentTime
        service.delayedOperation()
        val end = currentTime

        // Virtual time: delay는 즉시 스킵됨
        assertTrue(end - start < 100)
    }
}

class CoroutineService {
    suspend fun fetchData(): String {
        delay(1000)
        return "Data"
    }

    suspend fun delayedOperation() {
        delay(5000)
    }
}
```

---

### MockK + 코루틴

```kotlin
class UserServiceCoroutineTest {

    private val userRepository: UserRepository = mockk()
    private val userService = UserService(userRepository)

    @Test
    fun `findById with coroutine`() = runTest {
        // Given
        val user = User(id = 1, name = "Alice", email = "alice@example.com", age = 30)

        coEvery { userRepository.findById(1) } returns user

        // When
        val result = userService.findById(1)

        // Then
        assertEquals("Alice", result?.name)
        coVerify { userRepository.findById(1) }
    }

    @Test
    fun `parallel execution test`() = runTest {
        // Given
        val user1 = User(id = 1, name = "Alice", email = "alice@example.com", age = 30)
        val user2 = User(id = 2, name = "Bob", email = "bob@example.com", age = 25)

        coEvery { userRepository.findById(1) } coAnswers {
            delay(100)
            user1
        }

        coEvery { userRepository.findById(2) } coAnswers {
            delay(100)
            user2
        }

        // When
        val start = currentTime
        val results = userService.findMultiple(listOf(1, 2))
        val end = currentTime

        // Then
        assertEquals(2, results.size)
        // 병렬 실행되므로 100ms 정도만 소요
        assertTrue(end - start < 150)
    }
}
```

---

## 25.7 Kotest 사용 (선택)

### BehaviorSpec

```kotlin
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.mockk.every
import io.mockk.mockk

class UserServiceKotestTest : BehaviorSpec({

    val userRepository = mockk<UserRepository>()
    val userService = UserService(userRepository)

    given("a user exists") {
        val user = User(id = 1, name = "Alice", email = "alice@example.com", age = 30)
        every { userRepository.findById(1) } returns user

        `when`("finding by id") {
            val result = userService.findById(1)

            then("should return user dto") {
                result shouldNotBe null
                result?.name shouldBe "Alice"
            }
        }
    }

    given("a user does not exist") {
        every { userRepository.findById(999) } returns null

        `when`("finding by id") {
            val result = userService.findById(999)

            then("should return null") {
                result shouldBe null
            }
        }
    }
})
```

---

### StringSpec (간결한 테스트)

```kotlin
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe

class CalculatorKotestTest : StringSpec({

    val calculator = Calculator()

    "2 + 3 should be 5" {
        calculator.add(2, 3) shouldBe 5
    }

    "5 - 3 should be 2" {
        calculator.subtract(5, 3) shouldBe 2
    }

    "10 / 2 should be 5" {
        calculator.divide(10, 2) shouldBe 5
    }
})
```

---

## 25.8 테스트 더블 전략

### Stub, Mock, Spy 비교

```kotlin
class TestDoubleExampleTest {

    @Test
    fun `stub example`() {
        // Stub: 미리 정의된 응답 반환
        val stub: UserRepository = mockk()
        every { stub.findById(any()) } returns User(1, "Stub User", "stub@example.com", 25)

        val result = stub.findById(1)
        assertEquals("Stub User", result?.name)
    }

    @Test
    fun `mock example`() {
        // Mock: 호출 검증
        val mock: UserRepository = mockk(relaxed = true)

        mock.findById(1)
        mock.findById(2)

        verify(exactly = 2) { mock.findById(any()) }
    }

    @Test
    fun `spy example`() {
        // Spy: 일부만 모킹, 나머지는 실제 구현
        val realService = UserService(mockk(relaxed = true))
        val spy = spyk(realService)

        every { spy.findById(1) } returns UserDto(1, "Spy", "spy@example.com", 30, LocalDateTime.now())

        val result = spy.findById(1)
        assertEquals("Spy", result?.name)
    }
}
```

---

## 실전 팁

### 💡 Tip 1: 백틱으로 가독성 높이기

```kotlin
// ✅ Kotlin
@Test
fun `should return user when user exists`() {
    // ...
}

// ❌ Java 스타일
@Test
fun shouldReturnUserWhenUserExists() {
    // ...
}
```

---

### 💡 Tip 2: lateinit vs lazy

```kotlin
class UserServiceTest {

    // ✅ lateinit (매 테스트마다 초기화)
    private lateinit var userService: UserService

    @BeforeEach
    fun setup() {
        userService = UserService(mockk())
    }

    // ✅ lazy (한 번만 초기화)
    private val calculator by lazy { Calculator() }
}
```

---

### 💡 Tip 3: coEvery, coVerify

```kotlin
// ✅ suspend 함수 테스트
coEvery { repository.findById(1) } returns user
coVerify { repository.findById(1) }

// ❌ 일반 함수처럼 사용하면 에러
every { repository.findById(1) } returns user  // 컴파일 에러!
```

---

## 핵심 요약

### 꼭 기억할 것

1. **MockK**
   - `every`, `verify`
   - `coEvery`, `coVerify` (코루틴)
   - `relaxed = true`

2. **Spring Boot 테스트**
   - `@SpringBootTest`: 통합 테스트
   - `@WebMvcTest`: Controller 테스트
   - `@DataJpaTest`: Repository 테스트

3. **코루틴 테스트**
   - `runTest`
   - Virtual time

4. **Kotest** (선택)
   - 더 읽기 쉬운 DSL
   - BehaviorSpec, StringSpec

---

[← 이전](../part8-spring/chapter24-coroutines-webflux.md) | [다음: Chapter 26 →](chapter26-logging-monitoring.md)