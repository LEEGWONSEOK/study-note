# Chapter 23. REST API 개발

## 23.1 RestController 기본

### Java와 비교

**Java**:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public ResponseEntity<List<UserDto>> getAllUsers() {
        List<UserDto> users = userService.findAll();
        return ResponseEntity.ok(users);
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        UserDto user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }
}
```

**Kotlin**:
```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    @GetMapping
    fun getAllUsers(): List<UserDto> = userService.findAll()

    @GetMapping("/{id}")
    fun getUser(@PathVariable id: Long): ResponseEntity<UserDto> {
        val user = userService.findById(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(user)
    }
}
```

---

## 23.2 CRUD API 완성

### Entity, DTO, Request

```kotlin
@Entity
class User(
    var name: String,
    var email: String,
    var age: Int,

    @Id @GeneratedValue
    var id: Long = 0,

    var createdAt: LocalDateTime = LocalDateTime.now()
)

data class UserDto(
    val id: Long,
    val name: String,
    val email: String,
    val age: Int,
    val createdAt: LocalDateTime
)

data class CreateUserRequest(
    val name: String,
    val email: String,
    val age: Int
)

data class UpdateUserRequest(
    val name: String?,
    val email: String?,
    val age: Int?
)

fun User.toDto() = UserDto(id, name, email, age, createdAt)
```

---

### Controller

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    // GET /api/users
    @GetMapping
    fun getAllUsers(): List<UserDto> = userService.findAll()

    // GET /api/users/{id}
    @GetMapping("/{id}")
    fun getUser(@PathVariable id: Long): ResponseEntity<UserDto> {
        val user = userService.findById(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(user)
    }

    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createUser(@Valid @RequestBody request: CreateUserRequest): UserDto =
        userService.create(request)

    // PUT /api/users/{id}
    @PutMapping("/{id}")
    fun updateUser(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserDto> {
        val user = userService.update(id, request)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(user)
    }

    // DELETE /api/users/{id}
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deleteUser(@PathVariable id: Long) {
        userService.delete(id)
    }
}
```

---

### Service

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
            email = request.email,
            age = request.age
        )
        return userRepository.save(user).toDto()
    }

    @Transactional
    fun update(id: Long, request: UpdateUserRequest): UserDto? {
        val user = userRepository.findById(id).orElse(null)
            ?: return null

        request.name?.let { user.name = it }
        request.email?.let { user.email = it }
        request.age?.let { user.age = it }

        return user.toDto()
    }

    @Transactional
    fun delete(id: Long) {
        userRepository.deleteById(id)
    }
}
```

---

## 23.3 Validation

### 의존성 추가

**build.gradle.kts**:
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

---

### Request DTO에 Validation

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String,

    @field:Min(value = 0, message = "Age must be at least 0")
    @field:Max(value = 150, message = "Age must be at most 150")
    val age: Int
)

data class UpdateUserRequest(
    @field:Size(min = 2, max = 50)
    val name: String?,

    @field:Email
    val email: String?,

    @field:Min(0)
    @field:Max(150)
    val age: Int?
)
```

**주의**: Kotlin에서는 `@field:` 접두사 필수!

---

### Controller에서 사용

```kotlin
@PostMapping
fun createUser(@Valid @RequestBody request: CreateUserRequest): UserDto =
    userService.create(request)
```

---

## 23.4 예외 처리

### 커스텀 예외

```kotlin
class UserNotFoundException(message: String) : RuntimeException(message)
class DuplicateEmailException(message: String) : RuntimeException(message)
```

---

### @ControllerAdvice로 전역 예외 처리

```kotlin
data class ErrorResponse(
    val message: String,
    val status: Int,
    val timestamp: LocalDateTime = LocalDateTime.now()
)

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException::class)
    fun handleUserNotFound(e: UserNotFoundException): ResponseEntity<ErrorResponse> {
        val error = ErrorResponse(
            message = e.message ?: "User not found",
            status = HttpStatus.NOT_FOUND.value()
        )
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error)
    }

    @ExceptionHandler(DuplicateEmailException::class)
    fun handleDuplicateEmail(e: DuplicateEmailException): ResponseEntity<ErrorResponse> {
        val error = ErrorResponse(
            message = e.message ?: "Email already exists",
            status = HttpStatus.CONFLICT.value()
        )
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error)
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationException(e: MethodArgumentNotValidException): ResponseEntity<Map<String, Any>> {
        val errors = e.bindingResult.fieldErrors.associate {
            it.field to (it.defaultMessage ?: "Invalid value")
        }

        val response = mapOf(
            "message" to "Validation failed",
            "status" to HttpStatus.BAD_REQUEST.value(),
            "errors" to errors,
            "timestamp" to LocalDateTime.now()
        )

        return ResponseEntity.badRequest().body(response)
    }

    @ExceptionHandler(Exception::class)
    fun handleGenericException(e: Exception): ResponseEntity<ErrorResponse> {
        val error = ErrorResponse(
            message = "Internal server error",
            status = HttpStatus.INTERNAL_SERVER_ERROR.value()
        )
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error)
    }
}
```

---

### Service에서 예외 발생

```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository
) {

    fun findById(id: Long): UserDto =
        userRepository.findById(id)
            .orElseThrow { UserNotFoundException("User not found: $id") }
            .toDto()

    @Transactional
    fun create(request: CreateUserRequest): UserDto {
        // 이메일 중복 체크
        if (userRepository.existsByEmail(request.email)) {
            throw DuplicateEmailException("Email already exists: ${request.email}")
        }

        val user = User(
            name = request.name,
            email = request.email,
            age = request.age
        )
        return userRepository.save(user).toDto()
    }
}
```

---

## 23.5 페이징과 정렬

### Controller

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    @GetMapping
    fun getAllUsers(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int,
        @RequestParam(defaultValue = "id") sortBy: String,
        @RequestParam(defaultValue = "ASC") direction: String
    ): Page<UserDto> {
        val sort = Sort.by(Sort.Direction.fromString(direction), sortBy)
        val pageable = PageRequest.of(page, size, sort)
        return userService.findAll(pageable)
    }
}
```

---

### Service

```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository
) {

    fun findAll(pageable: Pageable): Page<UserDto> =
        userRepository.findAll(pageable).map { it.toDto() }
}
```

---

### 응답 예시

```json
GET /api/users?page=0&size=5&sortBy=name&direction=DESC

{
  "content": [
    {
      "id": 1,
      "name": "Alice",
      "email": "alice@example.com",
      "age": 30,
      "createdAt": "2024-01-01T10:00:00"
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 5
  },
  "totalElements": 100,
  "totalPages": 20,
  "last": false
}
```

---

## 23.6 검색 API

### Query Parameters

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService
) {

    @GetMapping("/search")
    fun searchUsers(
        @RequestParam(required = false) name: String?,
        @RequestParam(required = false) email: String?,
        @RequestParam(required = false) minAge: Int?,
        @RequestParam(required = false) maxAge: Int?
    ): List<UserDto> =
        userService.search(name, email, minAge, maxAge)
}
```

---

### Service

```kotlin
@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository
) {

    fun search(
        name: String?,
        email: String?,
        minAge: Int?,
        maxAge: Int?
    ): List<UserDto> {
        // 간단한 구현
        var users = userRepository.findAll()

        name?.let { users = users.filter { user -> user.name.contains(it, ignoreCase = true) } }
        email?.let { users = users.filter { user -> user.email.contains(it, ignoreCase = true) } }
        minAge?.let { users = users.filter { user -> user.age >= it } }
        maxAge?.let { users = users.filter { user -> user.age <= it } }

        return users.map { it.toDto() }
    }
}
```

---

## 23.7 파일 업로드

### Controller

```kotlin
@RestController
@RequestMapping("/api/files")
class FileController(
    private val fileService: FileService
) {

    @PostMapping("/upload")
    fun uploadFile(@RequestParam("file") file: MultipartFile): ResponseEntity<Map<String, String>> {
        val filename = fileService.save(file)

        val response = mapOf(
            "filename" to filename,
            "url" to "/api/files/$filename"
        )

        return ResponseEntity.ok(response)
    }

    @GetMapping("/{filename:.+}")
    fun downloadFile(@PathVariable filename: String): ResponseEntity<Resource> {
        val resource = fileService.load(filename)

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"${resource.filename}\"")
            .body(resource)
    }
}
```

---

### Service

```kotlin
@Service
class FileService {

    private val uploadDir = Paths.get("uploads")

    init {
        Files.createDirectories(uploadDir)
    }

    fun save(file: MultipartFile): String {
        val filename = "${System.currentTimeMillis()}_${file.originalFilename}"
        val path = uploadDir.resolve(filename)
        Files.copy(file.inputStream, path, StandardCopyOption.REPLACE_EXISTING)
        return filename
    }

    fun load(filename: String): Resource {
        val path = uploadDir.resolve(filename)
        val resource = UrlResource(path.toUri())

        if (resource.exists() && resource.isReadable) {
            return resource
        } else {
            throw FileNotFoundException("File not found: $filename")
        }
    }
}
```

---

## 23.8 CORS 설정

### Global CORS

```kotlin
@Configuration
class WebConfig : WebMvcConfigurer {

    override fun addCorsMappings(registry: CorsRegistry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600)
    }
}
```

---

### Controller-level CORS

```kotlin
@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = ["http://localhost:3000"])
class UserController(
    private val userService: UserService
) {
    // ...
}
```

---

## 23.9 실전 예제: 블로그 API

### Post API

```kotlin
@RestController
@RequestMapping("/api/posts")
class PostController(
    private val postService: PostService
) {

    @GetMapping
    fun getAllPosts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int
    ): Page<PostDto> {
        val pageable = PageRequest.of(page, size, Sort.by("createdAt").descending())
        return postService.findAll(pageable)
    }

    @GetMapping("/{id}")
    fun getPost(@PathVariable id: Long): PostDto =
        postService.findById(id)

    @GetMapping("/{id}/comments")
    fun getPostComments(@PathVariable id: Long): List<CommentDto> =
        postService.findComments(id)

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createPost(
        @Valid @RequestBody request: CreatePostRequest,
        @RequestParam authorId: Long
    ): PostDto =
        postService.create(request, authorId)

    @PostMapping("/{id}/comments")
    @ResponseStatus(HttpStatus.CREATED)
    fun addComment(
        @PathVariable id: Long,
        @Valid @RequestBody request: CreateCommentRequest,
        @RequestParam authorId: Long
    ): CommentDto =
        postService.addComment(id, request, authorId)

    @PutMapping("/{id}")
    fun updatePost(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdatePostRequest
    ): PostDto =
        postService.update(id, request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deletePost(@PathVariable id: Long) {
        postService.delete(id)
    }

    @GetMapping("/search")
    fun searchPosts(@RequestParam keyword: String): List<PostDto> =
        postService.search(keyword)
}
```

---

### Request DTOs

```kotlin
data class CreatePostRequest(
    @field:NotBlank(message = "Title is required")
    @field:Size(min = 1, max = 200)
    val title: String,

    @field:NotBlank(message = "Content is required")
    val content: String
)

data class UpdatePostRequest(
    @field:Size(min = 1, max = 200)
    val title: String?,

    val content: String?
)

data class CreateCommentRequest(
    @field:NotBlank(message = "Content is required")
    @field:Size(min = 1, max = 500)
    val content: String
)
```

---

## 실전 팁

### 💡 Tip 1: Elvis Operator로 간결하게

```kotlin
// ✅ Kotlin
@GetMapping("/{id}")
fun getUser(@PathVariable id: Long): UserDto =
    userService.findById(id) ?: throw UserNotFoundException("User not found: $id")

// ❌ Java 스타일
@GetMapping("/{id}")
fun getUser(@PathVariable id: Long): ResponseEntity<UserDto> {
    val user = userService.findById(id)
    if (user == null) {
        return ResponseEntity.notFound().build()
    }
    return ResponseEntity.ok(user)
}
```

---

### 💡 Tip 2: Extension Functions

```kotlin
fun <T> T?.orThrow(message: String): T =
    this ?: throw IllegalArgumentException(message)

// 사용
@GetMapping("/{id}")
fun getUser(@PathVariable id: Long): UserDto =
    userService.findById(id).orThrow("User not found: $id")
```

---

### 💡 Tip 3: @field: 접두사 필수

```kotlin
// ✅ 올바름
data class CreateUserRequest(
    @field:NotBlank
    val name: String
)

// ❌ 작동 안 함
data class CreateUserRequest(
    @NotBlank
    val name: String
)
```

---

## 핵심 요약

### 꼭 기억할 것

1. **RestController**
   - 생성자 주입 (간결함)
   - Elvis operator 활용

2. **Validation**
   - `@field:` 접두사 필수
   - `@Valid` 어노테이션

3. **예외 처리**
   - `@RestControllerAdvice`
   - 커스텀 예외

4. **페이징/검색**
   - `Pageable`, `Page<T>`
   - 기본값 제공

---

[← 이전](22-스프링_DataJPA.md) | [다음: Chapter 24 →](24-코루틴과_WebFlux.md)