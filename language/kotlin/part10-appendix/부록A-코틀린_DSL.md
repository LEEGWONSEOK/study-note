# 부록 A. Kotlin DSL

## A.1 DSL이란?

**DSL (Domain-Specific Language)**: 특정 도메인에 특화된 언어

Kotlin은 DSL을 만들기에 아주 적합한 언어입니다:
- Lambda with receiver
- Extension functions
- Infix functions
- Operator overloading

---

## A.2 HTML DSL 예제

### 기본 구현

```kotlin
class HTML {
    private val children = mutableListOf<String>()

    fun head(init: Head.() -> Unit) {
        val head = Head()
        head.init()
        children.add("<head>${head.render()}</head>")
    }

    fun body(init: Body.() -> Unit) {
        val body = Body()
        body.init()
        children.add("<body>${body.render()}</body>")
    }

    fun render() = "<html>${children.joinToString("")}</html>"
}

class Head {
    private val children = mutableListOf<String>()

    fun title(text: String) {
        children.add("<title>$text</title>")
    }

    fun render() = children.joinToString("")
}

class Body {
    private val children = mutableListOf<String>()

    fun h1(text: String) {
        children.add("<h1>$text</h1>")
    }

    fun p(text: String) {
        children.add("<p>$text</p>")
    }

    fun render() = children.joinToString("")
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```

---

### 사용

```kotlin
val page = html {
    head {
        title("My Page")
    }
    body {
        h1("Hello, Kotlin DSL!")
        p("This is a paragraph.")
    }
}

println(page.render())
```

**출력**:
```html
<html><head><title>My Page</title></head><body><h1>Hello, Kotlin DSL!</h1><p>This is a paragraph.</p></body></html>
```

---

## A.3 Type-Safe Builders

### SQL DSL

```kotlin
class SelectBuilder {
    private val columns = mutableListOf<String>()
    private var tableName: String? = null
    private val whereClauses = mutableListOf<String>()

    fun select(vararg cols: String) {
        columns.addAll(cols)
    }

    fun from(table: String) {
        tableName = table
    }

    fun where(condition: String) {
        whereClauses.add(condition)
    }

    fun build(): String {
        val cols = if (columns.isEmpty()) "*" else columns.joinToString(", ")
        val where = if (whereClauses.isEmpty()) "" else " WHERE ${whereClauses.joinToString(" AND ")}"
        return "SELECT $cols FROM $tableName$where"
    }
}

fun select(init: SelectBuilder.() -> Unit): String {
    val builder = SelectBuilder()
    builder.init()
    return builder.build()
}
```

**사용**:
```kotlin
val query = select {
    select("id", "name", "email")
    from("users")
    where("age > 18")
    where("active = true")
}

println(query)
// SELECT id, name, email FROM users WHERE age > 18 AND active = true
```

---

## A.4 Spring DSL 예제

### Bean Definition DSL

```kotlin
import org.springframework.context.support.GenericApplicationContext
import org.springframework.context.support.beans

val context = beans {
    bean<UserRepository>()
    bean<UserService>()
    bean {
        UserController(ref())
    }
}

fun main() {
    val appContext = GenericApplicationContext()
    context.initialize(appContext)
    appContext.refresh()

    val controller = appContext.getBean<UserController>()
}
```

---

### Router DSL

```kotlin
import org.springframework.web.reactive.function.server.router

val routes = router {
    "/api".nest {
        "/users".nest {
            GET("", userHandler::findAll)
            GET("/{id}", userHandler::findById)
            POST("", userHandler::create)
            PUT("/{id}", userHandler::update)
            DELETE("/{id}", userHandler::delete)
        }
    }
}
```

---

## A.5 Test DSL

### Kotest DSL

```kotlin
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe

class CalculatorSpec : StringSpec({
    "addition should work" {
        val calc = Calculator()
        calc.add(2, 3) shouldBe 5
    }

    "subtraction should work" {
        val calc = Calculator()
        calc.subtract(5, 3) shouldBe 2
    }
})
```

---

### BehaviorSpec

```kotlin
import io.kotest.core.spec.style.BehaviorSpec

class UserServiceSpec : BehaviorSpec({
    given("a user repository") {
        val repository = mockk<UserRepository>()
        val service = UserService(repository)

        `when`("finding a user by id") {
            val user = User(1, "Alice", "alice@example.com", 30)
            every { repository.findById(1) } returns user

            val result = service.findById(1)

            then("should return the user") {
                result shouldNotBe null
                result?.name shouldBe "Alice"
            }
        }
    }
})
```

---

## A.6 Configuration DSL

### Custom Config DSL

```kotlin
class DatabaseConfig {
    var url: String = ""
    var username: String = ""
    var password: String = ""
    var maxPoolSize: Int = 10

    fun validate() {
        require(url.isNotBlank()) { "URL is required" }
        require(username.isNotBlank()) { "Username is required" }
    }
}

class ServerConfig {
    var port: Int = 8080
    var host: String = "localhost"
}

class AppConfig {
    private val _database = DatabaseConfig()
    private val _server = ServerConfig()

    fun database(init: DatabaseConfig.() -> Unit) {
        _database.init()
    }

    fun server(init: ServerConfig.() -> Unit) {
        _server.init()
    }

    fun build(): Config {
        _database.validate()
        return Config(_database, _server)
    }
}

data class Config(
    val database: DatabaseConfig,
    val server: ServerConfig
)

fun appConfig(init: AppConfig.() -> Unit): Config {
    val config = AppConfig()
    config.init()
    return config.build()
}
```

**사용**:
```kotlin
val config = appConfig {
    database {
        url = "jdbc:postgresql://localhost:5432/mydb"
        username = "postgres"
        password = "secret"
        maxPoolSize = 20
    }

    server {
        port = 9000
        host = "0.0.0.0"
    }
}

println("Database URL: ${config.database.url}")
println("Server Port: ${config.server.port}")
```

---

## A.7 Gradle Kotlin DSL

### build.gradle.kts 예제

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    kotlin("plugin.jpa") version "1.9.21"
}

group = "com.example"
version = "0.0.1-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")

    runtimeOnly("com.h2database:h2")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.mockk:mockk:1.13.8")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "17"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## A.8 실전 팁

### 💡 Tip 1: @DslMarker로 스코프 제어

```kotlin
@DslMarker
annotation class HtmlTagMarker

@HtmlTagMarker
class HTML {
    fun body(init: Body.() -> Unit) { }
}

@HtmlTagMarker
class Body {
    fun p(text: String) { }
}

// ✅ OK
html {
    body {
        p("text")
    }
}

// ❌ 컴파일 에러 (암시적 수신자 사용 불가)
html {
    body {
        this@html.body {  // 명시적으로만 가능
            p("text")
        }
    }
}
```

---

### 💡 Tip 2: Infix로 가독성 향상

```kotlin
infix fun <T> T.should(matcher: Matcher<T>) {
    matcher.test(this)
}

interface Matcher<T> {
    fun test(value: T)
}

class BeEqualTo<T>(private val expected: T) : Matcher<T> {
    override fun test(value: T) {
        if (value != expected) {
            throw AssertionError("Expected $expected but got $value")
        }
    }
}

fun <T> be(value: T) = BeEqualTo(value)

// 사용
val result = 2 + 3
result should be(5)
```

---

### 💡 Tip 3: Operator Overloading

```kotlin
class Query {
    private val conditions = mutableListOf<String>()

    operator fun String.unaryPlus() {
        conditions.add(this)
    }

    fun build() = conditions.joinToString(" AND ")
}

fun query(init: Query.() -> Unit): String {
    val q = Query()
    q.init()
    return q.build()
}

// 사용
val sql = query {
    +"age > 18"
    +"active = true"
    +"country = 'USA'"
}

println(sql)
// age > 18 AND active = true AND country = 'USA'
```

---

## 핵심 요약

### DSL 핵심 요소

1. **Lambda with Receiver**
   - `init: Type.() -> Unit`

2. **Extension Functions**
   - 타입에 메서드 추가

3. **Infix Functions**
   - `value should be(expected)`

4. **@DslMarker**
   - 스코프 제어

---

[← 이전](../part09-practice/28-배포.md) | [다음: 부록 B →](부록B-주요_라이브러리.md)