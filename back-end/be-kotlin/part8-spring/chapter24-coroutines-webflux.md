# Chapter 24. 코루틴과 WebFlux

## 개요

Spring WebFlux는 **비동기 논블로킹** 웹 프레임워크입니다. Kotlin의 코루틴과 결합하면 반응형 프로그래밍을 동기 코드처럼 작성할 수 있습니다.

---

## 24.1 WebFlux vs Spring MVC

### 비교

| | Spring MVC | Spring WebFlux |
|--|------------|----------------|
| 모델 | 동기/블로킹 | 비동기/논블로킹 |
| 스레드 | 요청당 스레드 | 적은 수의 스레드 |
| 처리량 | 중간 | 높음 |
| 지연시간 | 높을 수 있음 | 낮음 |
| 사용 사례 | 일반적인 웹 | 고트래픽, I/O 집약적 |

---

### 의존성

**build.gradle.kts**:
```kotlin
dependencies {
    // WebFlux
    implementation("org.springframework.boot:spring-boot-starter-webflux")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")

    // R2DBC (Reactive Database)
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    runtimeOnly("com.h2database:h2")
    runtimeOnly("io.r2dbc:r2dbc-h2")
}
```

---

## 24.2 코루틴 Controller

### Spring MVC 방식

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    // ❌ 블로킹
    @GetMapping
    fun getAllUsers(): List<UserDto> = userService.findAll()

    @GetMapping("/{id}")
    fun getUser(@PathVariable id: Long): UserDto =
        userService.findById(id)
}
```

---

### WebFlux + 코루틴 방식

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    // ✅ 논블로킹
    @GetMapping
    suspend fun getAllUsers(): List<UserDto> = userService.findAll()

    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: Long): UserDto =
        userService.findById(id)
}
```

**차이점**: `suspend` 키워드만 추가!

---

## 24.3 R2DBC로 Reactive Repository

### Entity

```kotlin
@Table("users")
data class User(
    @Id
    val id: Long? = null,
    val name: String,
    val email: String,
    val age: Int,
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

**주의**: R2DBC는 `@Entity` 대신 `@Table` 사용

---

### Repository

```kotlin
interface UserRepository : CoroutineCrudRepository<User, Long> {
    // 메서드 이름 기반 쿼리
    suspend fun findByEmail(email: String): User?
    suspend fun findByNameContaining(keyword: String): Flow<User>
    suspend fun existsByEmail(email: String): Boolean
}
```

**주목**:
- `CoroutineCrudRepository` 사용
- 반환 타입: `suspend fun`, `Flow<T>`

---

### Service

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {

    suspend fun findAll(): List<UserDto> =
        userRepository.findAll().toList().map { it.toDto() }

    suspend fun findById(id: Long): UserDto =
        userRepository.findById(id)?.toDto()
            ?: throw UserNotFoundException("User not found: $id")

    suspend fun create(request: CreateUserRequest): UserDto {
        if (userRepository.existsByEmail(request.email)) {
            throw DuplicateEmailException("Email already exists")
        }

        val user = User(
            name = request.name,
            email = request.email,
            age = request.age
        )

        return userRepository.save(user).toDto()
    }

    suspend fun update(id: Long, request: UpdateUserRequest): UserDto {
        val user = userRepository.findById(id)
            ?: throw UserNotFoundException("User not found: $id")

        val updated = user.copy(
            name = request.name ?: user.name,
            email = request.email ?: user.email,
            age = request.age ?: user.age
        )

        return userRepository.save(updated).toDto()
    }

    suspend fun delete(id: Long) {
        userRepository.deleteById(id)
    }

    suspend fun search(keyword: String): List<UserDto> =
        userRepository.findByNameContaining(keyword).toList().map { it.toDto() }
}
```

---

## 24.4 Flow를 활용한 스트리밍

### Controller

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    // ✅ 스트리밍 응답
    @GetMapping("/stream")
    fun streamUsers(): Flow<UserDto> = userService.streamAll()

    @GetMapping("/search/stream")
    fun searchUsersStream(@RequestParam keyword: String): Flow<UserDto> =
        userService.searchStream(keyword)
}
```

---

### Service

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository
) {

    fun streamAll(): Flow<UserDto> =
        userRepository.findAll().map { it.toDto() }

    fun searchStream(keyword: String): Flow<UserDto> =
        userRepository.findByNameContaining(keyword).map { it.toDto() }
}
```

---

### 클라이언트에서 스트림 받기

```kotlin
// WebClient 사용
val client = WebClient.create("http://localhost:8080")

runBlocking {
    client.get()
        .uri("/api/users/stream")
        .retrieve()
        .bodyToFlow<UserDto>()
        .collect { user ->
            println("Received: $user")
        }
}
```

---

## 24.5 외부 API 호출

### WebClient로 비동기 호출

```kotlin
@Service
class ExternalApiService {

    private val webClient = WebClient.builder()
        .baseUrl("https://jsonplaceholder.typicode.com")
        .build()

    suspend fun fetchPost(id: Long): ExternalPost? =
        webClient.get()
            .uri("/posts/$id")
            .retrieve()
            .awaitBodyOrNull()

    suspend fun fetchAllPosts(): List<ExternalPost> =
        webClient.get()
            .uri("/posts")
            .retrieve()
            .awaitBody()

    suspend fun createPost(request: CreatePostRequest): ExternalPost =
        webClient.post()
            .uri("/posts")
            .bodyValue(request)
            .retrieve()
            .awaitBody()
}

data class ExternalPost(
    val id: Long,
    val userId: Long,
    val title: String,
    val body: String
)
```

---

### 병렬 API 호출

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val externalApiService: ExternalApiService
) {

    suspend fun getUserWithPosts(id: Long): UserWithPosts = coroutineScope {
        // 병렬 실행
        val userDeferred = async { userRepository.findById(id) }
        val postsDeferred = async { externalApiService.fetchAllPosts() }

        val user = userDeferred.await()
            ?: throw UserNotFoundException("User not found: $id")

        val posts = postsDeferred.await()

        UserWithPosts(user.toDto(), posts)
    }
}

data class UserWithPosts(
    val user: UserDto,
    val posts: List<ExternalPost>
)
```

---

## 24.6 에러 핸들링

### Reactive 예외 처리

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException::class)
    suspend fun handleUserNotFound(e: UserNotFoundException): ResponseEntity<ErrorResponse> {
        val error = ErrorResponse(
            message = e.message ?: "User not found",
            status = HttpStatus.NOT_FOUND.value()
        )
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error)
    }

    @ExceptionHandler(WebClientResponseException::class)
    suspend fun handleWebClientException(e: WebClientResponseException): ResponseEntity<ErrorResponse> {
        val error = ErrorResponse(
            message = "External API error: ${e.message}",
            status = e.statusCode.value()
        )
        return ResponseEntity.status(e.statusCode).body(error)
    }
}
```

---

## 24.7 실전 예제: 실시간 주식 가격

### Entity & Repository

```kotlin
@Table("stock_prices")
data class StockPrice(
    @Id
    val id: Long? = null,
    val symbol: String,
    val price: Double,
    val timestamp: LocalDateTime = LocalDateTime.now()
)

interface StockPriceRepository : CoroutineCrudRepository<StockPrice, Long> {
    suspend fun findBySymbol(symbol: String): Flow<StockPrice>
    suspend fun findTop10BySymbolOrderByTimestampDesc(symbol: String): Flow<StockPrice>
}
```

---

### Service

```kotlin
@Service
class StockService(
    private val stockPriceRepository: StockPriceRepository
) {

    // 실시간 가격 스트림 (시뮬레이션)
    fun observePrices(symbol: String): Flow<StockPriceDto> = flow {
        while (true) {
            val price = generateRandomPrice()
            val stockPrice = StockPrice(
                symbol = symbol,
                price = price
            )

            stockPriceRepository.save(stockPrice)

            emit(stockPrice.toDto())
            delay(1000)  // 1초마다
        }
    }

    private fun generateRandomPrice(): Double =
        (100..200).random().toDouble() + Random.nextDouble()

    suspend fun getHistory(symbol: String): List<StockPriceDto> =
        stockPriceRepository.findTop10BySymbolOrderByTimestampDesc(symbol)
            .toList()
            .map { it.toDto() }
}

data class StockPriceDto(
    val symbol: String,
    val price: Double,
    val timestamp: LocalDateTime
)

fun StockPrice.toDto() = StockPriceDto(symbol, price, timestamp)
```

---

### Controller

```kotlin
@RestController
@RequestMapping("/api/stocks")
class StockController(
    private val stockService: StockService
) {

    @GetMapping("/{symbol}/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun streamPrices(@PathVariable symbol: String): Flow<StockPriceDto> =
        stockService.observePrices(symbol)

    @GetMapping("/{symbol}/history")
    suspend fun getHistory(@PathVariable symbol: String): List<StockPriceDto> =
        stockService.getHistory(symbol)
}
```

---

### 프론트엔드에서 SSE 받기

```javascript
const eventSource = new EventSource('http://localhost:8080/api/stocks/AAPL/stream');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Price update:', data);
    // UI 업데이트
};
```

---

## 24.8 Spring MVC와 코루틴 함께 사용

### application.yml

```yaml
spring:
  application:
    name: demo-app

# WebFlux 대신 Spring MVC 사용
# spring-boot-starter-web 의존성 유지
```

**build.gradle.kts**:
```kotlin
dependencies {
    // Spring MVC (WebFlux 대신)
    implementation("org.springframework.boot:spring-boot-starter-web")

    // 코루틴만 추가
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
}
```

---

### Controller (Spring MVC + 코루틴)

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService,
    private val externalApiService: ExternalApiService
) {

    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: Long): UserDto =
        userService.findById(id)

    @GetMapping("/{id}/with-posts")
    suspend fun getUserWithPosts(@PathVariable id: Long): UserWithPosts = coroutineScope {
        // 병렬 실행
        val user = async { userService.findById(id) }
        val posts = async { externalApiService.fetchAllPosts() }

        UserWithPosts(user.await(), posts.await())
    }
}
```

**장점**: Spring MVC를 유지하면서도 코루틴의 장점 활용 가능!

---

## 24.9 성능 비교

### 블로킹 방식 (Spring MVC)

```kotlin
@GetMapping("/users/{id}/orders")
fun getUserOrders(@PathVariable id: Long): UserWithOrders {
    val user = userService.findById(id)  // 100ms
    val orders = orderService.findByUserId(id)  // 100ms
    return UserWithOrders(user, orders)
}
// 총 소요 시간: 200ms
```

---

### 논블로킹 방식 (코루틴)

```kotlin
@GetMapping("/users/{id}/orders")
suspend fun getUserOrders(@PathVariable id: Long): UserWithOrders = coroutineScope {
    val user = async { userService.findById(id) }  // 100ms
    val orders = async { orderService.findByUserId(id) }  // 100ms
    UserWithOrders(user.await(), orders.await())
}
// 총 소요 시간: 100ms (병렬 실행)
```

---

## 실전 팁

### 💡 Tip 1: suspend + Spring MVC 조합

```kotlin
// ✅ Spring MVC에서도 suspend 사용 가능
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
}

@RestController
class UserController(private val userService: UserService) {
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long) = userService.findById(id)
}
```

---

### 💡 Tip 2: Flow for 스트리밍

```kotlin
// ✅ 대용량 데이터는 Flow 사용
@GetMapping("/users/export")
fun exportUsers(): Flow<UserDto> = flow {
    var page = 0
    while (true) {
        val users = userRepository.findAll(PageRequest.of(page, 100))
        if (users.isEmpty) break

        users.forEach { emit(it.toDto()) }
        page++
    }
}
```

---

### 💡 Tip 3: WebClient vs RestTemplate

```kotlin
// ❌ 블로킹
val restTemplate = RestTemplate()
val result = restTemplate.getForObject("...", String::class.java)

// ✅ 논블로킹
val webClient = WebClient.create()
suspend fun fetch(): String = webClient.get()
    .uri("...")
    .retrieve()
    .awaitBody()
```

---

## 핵심 요약

### 꼭 기억할 것

1. **suspend 함수**
   - Controller에서 `suspend` 키워드
   - 자동으로 코루틴 실행

2. **Flow**
   - 스트리밍 응답
   - 대용량 데이터 처리

3. **R2DBC**
   - Reactive Database
   - `CoroutineCrudRepository`

4. **병렬 실행**
   - `async`/`await`
   - 성능 향상

5. **WebClient**
   - 비동기 HTTP 클라이언트
   - `awaitBody()`, `awaitBodyOrNull()`

---

## Part 8 완료!

Part 8: Kotlin과 Spring Boot를 모두 마쳤습니다!

**배운 내용**:
- ✅ Chapter 20: Spring Boot 프로젝트 설정
- ✅ Chapter 21: Spring과 Kotlin 통합
- ✅ Chapter 22: Spring Data JPA
- ✅ Chapter 23: REST API 개발
- ✅ Chapter 24: 코루틴과 WebFlux

---

## 다음 Part 예고

**Part 9: 실전 활용**에서 다룰 내용:
- Chapter 25: 테스팅 (JUnit 5, MockK)
- Chapter 26: 로깅과 모니터링
- Chapter 27: 보안 (Spring Security)
- Chapter 28: 배포와 운영

---

[← 이전](chapter23-rest-api.md) | [다음: Chapter 25 →](../part9-practice/chapter25-testing.md)