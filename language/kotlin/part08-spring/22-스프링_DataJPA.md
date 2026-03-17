# Chapter 22. Spring Data JPA와 Kotlin

## 22.1 Entity 클래스 작성

### Java와 비교

**Java**:
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    private LocalDateTime createdAt;

    // 기본 생성자 필수
    protected User() {}

    // 생성자
    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }

    // Getter/Setter...
}
```

**Kotlin** (❌ 잘못된 방법):
```kotlin
@Entity
data class User(
    @Id @GeneratedValue
    val id: Long = 0,
    val name: String,
    val email: String,
    val createdAt: LocalDateTime = LocalDateTime.now()
)
// 문제: data class는 JPA 프록시와 충돌 가능
```

**Kotlin** (✅ 권장 방법):
```kotlin
@Entity
@Table(name = "users")
class User(
    @Column(nullable = false)
    var name: String,

    @Column(nullable = false, unique = true)
    var email: String,

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long = 0,

    var createdAt: LocalDateTime = LocalDateTime.now()
) {
    // no-arg 생성자는 kotlin-jpa 플러그인이 자동 생성
}
```

---

## 22.2 kotlin-jpa 플러그인의 역할

### 문제점

JPA는 리플렉션을 위해 **no-arg 생성자**가 필요하지만, Kotlin 클래스는 기본적으로 no-arg 생성자가 없습니다.

### 해결책

**build.gradle.kts**:
```kotlin
plugins {
    kotlin("plugin.jpa") version "1.9.21"
}
```

**자동으로 생성되는 것**:
```kotlin
@Entity
class User(var name: String) {
    // kotlin-jpa 플러그인이 자동으로 생성:
    // protected constructor() : this("") {}
}
```

---

## 22.3 Repository 인터페이스

### 기본 Repository

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    // 메서드 이름 기반 쿼리
    fun findByEmail(email: String): User?
    fun findByNameContaining(keyword: String): List<User>
    fun existsByEmail(email: String): Boolean
}
```

---

### @Query 사용

```kotlin
interface UserRepository : JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.email = :email")
    fun findByEmailCustom(@Param("email") email: String): User?

    @Query("SELECT u FROM User u WHERE u.createdAt >= :date")
    fun findRecentUsers(@Param("date") date: LocalDateTime): List<User>

    @Modifying
    @Query("UPDATE User u SET u.name = :name WHERE u.id = :id")
    fun updateName(@Param("id") id: Long, @Param("name") name: String): Int
}
```

---

### Native Query

```kotlin
interface UserRepository : JpaRepository<User, Long> {

    @Query(
        value = "SELECT * FROM users WHERE email = :email",
        nativeQuery = true
    )
    fun findByEmailNative(@Param("email") email: String): User?
}
```

---

## 22.4 관계 매핑

### OneToMany / ManyToOne

```kotlin
@Entity
class User(
    var name: String,

    @Id @GeneratedValue
    var id: Long = 0
) {
    @OneToMany(mappedBy = "user", cascade = [CascadeType.ALL])
    var orders: MutableList<Order> = mutableListOf()
}

@Entity
class Order(
    var product: String,
    var quantity: Int,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    var user: User? = null,

    @Id @GeneratedValue
    var id: Long = 0
)
```

---

### ManyToMany

```kotlin
@Entity
class Student(
    var name: String,

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = [JoinColumn(name = "student_id")],
        inverseJoinColumns = [JoinColumn(name = "course_id")]
    )
    var courses: MutableSet<Course> = mutableSetOf(),

    @Id @GeneratedValue
    var id: Long = 0
)

@Entity
class Course(
    var title: String,

    @ManyToMany(mappedBy = "courses")
    var students: MutableSet<Student> = mutableSetOf(),

    @Id @GeneratedValue
    var id: Long = 0
)
```

---

## 22.5 Kotlin-specific 패턴

### Extension Functions로 변환

```kotlin
@Entity
class User(
    var name: String,
    var email: String,

    @Id @GeneratedValue
    var id: Long = 0,

    var createdAt: LocalDateTime = LocalDateTime.now()
)

data class UserDto(
    val id: Long,
    val name: String,
    val email: String,
    val createdAt: LocalDateTime
)

// Extension function
fun User.toDto() = UserDto(
    id = id,
    name = name,
    email = email,
    createdAt = createdAt
)

// Service에서 사용
@Service
class UserService(private val userRepository: UserRepository) {

    fun findAll(): List<UserDto> =
        userRepository.findAll().map { it.toDto() }

    fun findById(id: Long): UserDto? =
        userRepository.findById(id).orElse(null)?.toDto()
}
```

---

### Nullable 처리

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?  // Nullable
}

@Service
class UserService(private val userRepository: UserRepository) {

    fun getUserByEmail(email: String): UserDto {
        val user = userRepository.findByEmail(email)
            ?: throw UserNotFoundException("User not found: $email")
        return user.toDto()
    }

    // 또는 Optional 대신 nullable
    fun getUserById(id: Long): User? =
        userRepository.findById(id).orElse(null)
}
```

---

## 22.6 주의사항과 해결책

### ⚠️ 문제 1: data class 사용

**문제**:
```kotlin
@Entity
data class User(...)
// 문제점:
// 1. equals/hashCode가 모든 필드 기반 → JPA 프록시 문제
// 2. copy() 메서드가 불필요함
// 3. 상속 불가능 (data class는 open 불가)
```

**해결**:
```kotlin
@Entity
class User(...) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is User) return false
        return id == other.id  // ID만 비교
    }

    override fun hashCode(): Int = id.hashCode()
}
```

---

### ⚠️ 문제 2: val vs var

**문제**:
```kotlin
@Entity
class User(
    val name: String,  // ❌ JPA가 값을 설정할 수 없음
    val email: String
)
```

**해결**:
```kotlin
@Entity
class User(
    var name: String,  // ✅ var 사용
    var email: String
)
```

---

### ⚠️ 문제 3: Lazy Loading

**문제**:
```kotlin
@Entity
class User(
    var name: String,

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    val orders: List<Order> = emptyList()  // ❌ 불변 리스트
)
```

**해결**:
```kotlin
@Entity
class User(
    var name: String,

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    var orders: MutableList<Order> = mutableListOf()  // ✅ Mutable
)
```

---

## 22.7 실전 예제: 블로그 시스템

### Entity 정의

```kotlin
@Entity
@Table(name = "posts")
class Post(
    @Column(nullable = false)
    var title: String,

    @Column(columnDefinition = "TEXT")
    var content: String,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    var author: User,

    @OneToMany(mappedBy = "post", cascade = [CascadeType.ALL], orphanRemoval = true)
    var comments: MutableList<Comment> = mutableListOf(),

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long = 0,

    var createdAt: LocalDateTime = LocalDateTime.now(),
    var updatedAt: LocalDateTime = LocalDateTime.now()
)

@Entity
@Table(name = "comments")
class Comment(
    @Column(nullable = false)
    var content: String,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    var post: Post,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    var author: User,

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long = 0,

    var createdAt: LocalDateTime = LocalDateTime.now()
)
```

---

### Repository

```kotlin
interface PostRepository : JpaRepository<Post, Long> {

    fun findByAuthor(author: User): List<Post>

    @Query("SELECT p FROM Post p WHERE p.title LIKE %:keyword% OR p.content LIKE %:keyword%")
    fun searchByKeyword(@Param("keyword") keyword: String): List<Post>

    @Query("SELECT p FROM Post p JOIN FETCH p.author WHERE p.id = :id")
    fun findByIdWithAuthor(@Param("id") id: Long): Post?
}

interface CommentRepository : JpaRepository<Comment, Long> {

    fun findByPost(post: Post): List<Comment>

    @Query("SELECT c FROM Comment c JOIN FETCH c.author WHERE c.post = :post")
    fun findByPostWithAuthor(@Param("post") post: Post): List<Comment>
}
```

---

### DTO

```kotlin
data class PostDto(
    val id: Long,
    val title: String,
    val content: String,
    val authorName: String,
    val commentCount: Int,
    val createdAt: LocalDateTime
)

data class CommentDto(
    val id: Long,
    val content: String,
    val authorName: String,
    val createdAt: LocalDateTime
)

// Extension functions
fun Post.toDto() = PostDto(
    id = id,
    title = title,
    content = content,
    authorName = author.name,
    commentCount = comments.size,
    createdAt = createdAt
)

fun Comment.toDto() = CommentDto(
    id = id,
    content = content,
    authorName = author.name,
    createdAt = createdAt
)
```

---

### Service

```kotlin
@Service
@Transactional(readOnly = true)
class PostService(
    private val postRepository: PostRepository,
    private val commentRepository: CommentRepository,
    private val userRepository: UserRepository
) {

    fun findAll(): List<PostDto> =
        postRepository.findAll().map { it.toDto() }

    fun findById(id: Long): PostDto? =
        postRepository.findById(id).orElse(null)?.toDto()

    @Transactional
    fun create(request: CreatePostRequest, authorId: Long): PostDto {
        val author = userRepository.findById(authorId).orElseThrow {
            IllegalArgumentException("User not found")
        }

        val post = Post(
            title = request.title,
            content = request.content,
            author = author
        )

        return postRepository.save(post).toDto()
    }

    @Transactional
    fun addComment(postId: Long, request: CreateCommentRequest, authorId: Long): CommentDto {
        val post = postRepository.findById(postId).orElseThrow {
            IllegalArgumentException("Post not found")
        }

        val author = userRepository.findById(authorId).orElseThrow {
            IllegalArgumentException("User not found")
        }

        val comment = Comment(
            content = request.content,
            post = post,
            author = author
        )

        post.comments.add(comment)

        return commentRepository.save(comment).toDto()
    }

    fun search(keyword: String): List<PostDto> =
        postRepository.searchByKeyword(keyword).map { it.toDto() }
}
```

---

## 22.8 Querydsl 사용 (선택)

### 의존성 추가

**build.gradle.kts**:
```kotlin
plugins {
    kotlin("kapt") version "1.9.21"
}

dependencies {
    implementation("com.querydsl:querydsl-jpa:5.0.0:jakarta")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
    kapt("jakarta.persistence:jakarta.persistence-api")
}

kotlin {
    sourceSets.main {
        kotlin.srcDir("$buildDir/generated/source/kapt/main")
    }
}
```

---

### Repository 구현

```kotlin
interface PostRepositoryCustom {
    fun searchPosts(keyword: String?, authorId: Long?): List<Post>
}

class PostRepositoryImpl : PostRepositoryCustom {

    @Autowired
    private lateinit var queryFactory: JPAQueryFactory

    override fun searchPosts(keyword: String?, authorId: Long?): List<Post> {
        val post = QPost.post

        return queryFactory
            .selectFrom(post)
            .where(
                keywordContains(keyword),
                authorIdEq(authorId)
            )
            .fetch()
    }

    private fun keywordContains(keyword: String?): BooleanExpression? {
        if (keyword.isNullOrBlank()) return null
        val post = QPost.post
        return post.title.contains(keyword).or(post.content.contains(keyword))
    }

    private fun authorIdEq(authorId: Long?): BooleanExpression? {
        if (authorId == null) return null
        val post = QPost.post
        return post.author.id.eq(authorId)
    }
}

interface PostRepository : JpaRepository<Post, Long>, PostRepositoryCustom
```

---

## 실전 팁

### 💡 Tip 1: Entity는 일반 class 사용

```kotlin
// ✅ 권장
@Entity
class User(var name: String)

// ❌ 권장하지 않음
@Entity
data class User(var name: String)
```

---

### 💡 Tip 2: ID는 기본값 제공

```kotlin
@Entity
class User(
    var name: String,

    @Id @GeneratedValue
    var id: Long = 0  // ✅ 기본값 제공
)
```

---

### 💡 Tip 3: Nullable vs Optional

```kotlin
// ✅ Kotlin 스타일 (Nullable)
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
}

// ❌ Java 스타일 (Optional)
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): Optional<User>
}
```

---

## 핵심 요약

### 꼭 기억할 것

1. **kotlin-jpa 플러그인**
   - no-arg 생성자 자동 생성
   - `build.gradle.kts`에 필수

2. **Entity 작성**
   - `class` 사용 (data class ❌)
   - 프로퍼티는 `var` 사용
   - ID는 기본값 제공

3. **Repository**
   - Nullable 반환 (`?`)
   - Optional 대신 null 사용

4. **Extension Functions**
   - Entity → DTO 변환
   - 코드 간결화

---

[← 이전](21-스프링과_코틀린_통합.md) | [다음: Chapter 23 →](23-REST_API_개발.md)