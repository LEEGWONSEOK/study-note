# Chapter 21. Spring과 Kotlin 통합

## 21.1 Bean 정의와 DI

### Java 스타일
```kotlin
@Service
class UserService @Autowired constructor(
    private val userRepository: UserRepository
) {
    fun findUser(id: Long) = userRepository.findById(id)
}
```

### Kotlin 스타일 (권장)
```kotlin
@Service
class UserService(
    private val userRepository: UserRepository  // 자동 주입
) {
    fun findUser(id: Long) = userRepository.findById(id)
}
```

---

## 21.2 Controller 작성

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    @GetMapping
    fun getAllUsers(): List<UserDto> = userService.findAll()

    @GetMapping("/{id}")
    fun getUser(@PathVariable id: Long): UserDto? =
        userService.findById(id)

    @PostMapping
    fun createUser(@RequestBody request: CreateUserRequest): UserDto =
        userService.create(request)

    @PutMapping("/{id}")
    fun updateUser(
        @PathVariable id: Long,
        @RequestBody request: UpdateUserRequest
    ): UserDto = userService.update(id, request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deleteUser(@PathVariable id: Long) =
        userService.delete(id)
}
```

---

## 21.3 Service 계층

```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository
) {
    fun findAll(): List<UserDto> =
        userRepository.findAll().map { it.toDto() }

    fun findById(id: Long): UserDto? =
        userRepository.findById(id).orElse(null)?.toDto()

    @Transactional
    fun create(request: CreateUserRequest): UserDto {
        val user = User(
            name = request.name,
            email = request.email
        )
        return userRepository.save(user).toDto()
    }
}
```

---

## 21.4 Configuration 클래스

```kotlin
@Configuration
class AppConfig {

    @Bean
    fun objectMapper(): ObjectMapper = ObjectMapper().apply {
        registerModule(JavaTimeModule())
        disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
    }

    @Bean
    fun restTemplate(): RestTemplate = RestTemplate()
}
```

---

## 21.5 데이터 클래스를 활용한 DTO

```kotlin
data class UserDto(
    val id: Long,
    val name: String,
    val email: String,
    val createdAt: LocalDateTime
)

data class CreateUserRequest(
    val name: String,
    val email: String
)

data class UpdateUserRequest(
    val name: String?,
    val email: String?
)

// Extension function for mapping
fun User.toDto() = UserDto(
    id = id,
    name = name,
    email = email,
    createdAt = createdAt
)
```

---

[← 이전](chapter20-spring-setup.md) | [다음: Chapter 22 →](chapter22-spring-data-jpa.md)