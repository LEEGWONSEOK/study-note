# 부록 B. 유용한 라이브러리

## B.1 코루틴 & 비동기

### kotlinx-coroutines

**용도**: 비동기 프로그래밍

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")
}
```

**예제**:
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        delay(1000)
        println("World!")
    }
    println("Hello")
}
```

---

## B.2 HTTP 클라이언트

### Ktor Client

**용도**: Kotlin 네이티브 HTTP 클라이언트

```kotlin
dependencies {
    implementation("io.ktor:ktor-client-core:2.3.6")
    implementation("io.ktor:ktor-client-cio:2.3.6")
    implementation("io.ktor:ktor-client-content-negotiation:2.3.6")
    implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.6")
}
```

**예제**:
```kotlin
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.engine.cio.*
import io.ktor.client.request.*

suspend fun main() {
    val client = HttpClient(CIO)
    val response: String = client.get("https://api.github.com").body()
    println(response)
    client.close()
}
```

---

### Fuel

**용도**: 간단한 HTTP 클라이언트

```kotlin
dependencies {
    implementation("com.github.kittinunf.fuel:fuel:2.3.1")
    implementation("com.github.kittinunf.fuel:fuel-coroutines:2.3.1")
}
```

**예제**:
```kotlin
import com.github.kittinunf.fuel.httpGet
import com.github.kittinunf.fuel.coroutines.awaitString

suspend fun main() {
    val response = "https://api.github.com".httpGet().awaitString()
    println(response)
}
```

---

## B.3 JSON 처리

### kotlinx-serialization

**용도**: Kotlin 공식 직렬화 라이브러리

**build.gradle.kts**:
```kotlin
plugins {
    kotlin("plugin.serialization") version "1.9.21"
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
}
```

**예제**:
```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class User(val id: Int, val name: String, val email: String)

fun main() {
    val user = User(1, "Alice", "alice@example.com")

    // Serialize
    val json = Json.encodeToString(user)
    println(json)  // {"id":1,"name":"Alice","email":"alice@example.com"}

    // Deserialize
    val parsed = Json.decodeFromString<User>(json)
    println(parsed)
}
```

---

### Gson

```kotlin
dependencies {
    implementation("com.google.code.gson:gson:2.10.1")
}
```

**예제**:
```kotlin
import com.google.gson.Gson

data class User(val id: Int, val name: String)

fun main() {
    val gson = Gson()
    val user = User(1, "Alice")

    val json = gson.toJson(user)
    val parsed = gson.fromJson(json, User::class.java)

    println(parsed)
}
```

---

## B.4 테스팅

### MockK

**용도**: Kotlin 전용 Mocking 라이브러리

```kotlin
testImplementation("io.mockk:mockk:1.13.8")
```

**예제**:
```kotlin
import io.mockk.*

val mock = mockk<UserRepository>()
every { mock.findById(1) } returns User(1, "Alice", "alice@example.com", 30)

val result = mock.findById(1)
verify { mock.findById(1) }
```

---

### Kotest

**용도**: Kotlin 스타일 테스트 프레임워크

```kotlin
testImplementation("io.kotest:kotest-runner-junit5:5.7.2")
testImplementation("io.kotest:kotest-assertions-core:5.7.2")
```

**예제**:
```kotlin
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe

class CalculatorTest : StringSpec({
    "2 + 3 should be 5" {
        Calculator().add(2, 3) shouldBe 5
    }
})
```

---

## B.5 로깅

### kotlin-logging

**용도**: Kotlin 스타일 로깅

```kotlin
implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")
```

**예제**:
```kotlin
import mu.KotlinLogging

private val log = KotlinLogging.logger {}

fun main() {
    log.info { "Hello, Kotlin Logging!" }
    log.debug { "Debug message: ${expensiveOperation()}" }  // Lazy
}
```

---

## B.6 날짜/시간

### kotlinx-datetime

**용도**: Kotlin 멀티플랫폼 날짜/시간 라이브러리

```kotlin
implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.5.0")
```

**예제**:
```kotlin
import kotlinx.datetime.*

fun main() {
    val now = Clock.System.now()
    println(now)

    val today = Clock.System.todayIn(TimeZone.currentSystemDefault())
    println(today)

    val nextWeek = today.plus(7, DateTimeUnit.DAY)
    println(nextWeek)
}
```

---

## B.7 Validation

### konform

**용도**: Type-safe validation

```kotlin
implementation("io.konform:konform:0.4.0")
```

**예제**:
```kotlin
import io.konform.validation.Validation
import io.konform.validation.jsonschema.*

data class User(val name: String, val email: String, val age: Int)

val validateUser = Validation<User> {
    User::name {
        minLength(2)
        maxLength(50)
    }

    User::email {
        pattern(".+@.+\\..+") hint "must be valid email"
    }

    User::age {
        minimum(0)
        maximum(150)
    }
}

fun main() {
    val user = User("A", "invalid-email", -5)
    val result = validateUser(user)

    if (result is io.konform.validation.Invalid) {
        result.errors.forEach { println(it) }
    }
}
```

---

## B.8 Arrow (함수형 프로그래밍)

### Arrow Core

**용도**: 함수형 프로그래밍 도구

```kotlin
implementation("io.arrow-kt:arrow-core:1.2.1")
```

**예제**:
```kotlin
import arrow.core.*

fun divide(a: Int, b: Int): Either<String, Int> =
    if (b == 0) Either.Left("Division by zero")
    else Either.Right(a / b)

fun main() {
    val result1 = divide(10, 2)
    val result2 = divide(10, 0)

    when (result1) {
        is Either.Left -> println("Error: ${result1.value}")
        is Either.Right -> println("Result: ${result1.value}")
    }

    when (result2) {
        is Either.Left -> println("Error: ${result2.value}")
        is Either.Right -> println("Result: ${result2.value}")
    }
}
```

---

### Option

```kotlin
import arrow.core.Option
import arrow.core.Some
import arrow.core.None

fun findUser(id: Int): Option<User> =
    if (id > 0) Some(User(id, "Alice", "alice@example.com", 30))
    else None

fun main() {
    val user = findUser(1)

    user.fold(
        { println("User not found") },
        { println("Found: $it") }
    )
}
```

---

## B.9 Database

### Exposed

**용도**: Kotlin SQL 라이브러리

```kotlin
implementation("org.jetbrains.exposed:exposed-core:0.44.1")
implementation("org.jetbrains.exposed:exposed-dao:0.44.1")
implementation("org.jetbrains.exposed:exposed-jdbc:0.44.1")
```

**예제**:
```kotlin
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction

object Users : Table() {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 50)
    val email = varchar("email", 100)
    override val primaryKey = PrimaryKey(id)
}

fun main() {
    Database.connect("jdbc:h2:mem:test", driver = "org.h2.Driver")

    transaction {
        SchemaUtils.create(Users)

        Users.insert {
            it[name] = "Alice"
            it[email] = "alice@example.com"
        }

        Users.selectAll().forEach {
            println("${it[Users.id]}: ${it[Users.name]}")
        }
    }
}
```

---

## B.10 유틸리티

### Kotlin-retry

**용도**: Retry 로직

```kotlin
implementation("com.michael-bull.kotlin-retry:kotlin-retry:1.0.9")
```

**예제**:
```kotlin
import com.github.michaelbull.retry.*

suspend fun main() {
    val result = retry(times = 3, delay = 1000) {
        callExternalApi()
    }
}

suspend fun callExternalApi(): String {
    // API 호출
    return "Success"
}
```

---

### KotlinPoet

**용도**: Kotlin 코드 생성

```kotlin
implementation("com.squareup:kotlinpoet:1.15.0")
```

**예제**:
```kotlin
import com.squareup.kotlinpoet.*

fun main() {
    val greeter = FunSpec.builder("greet")
        .addParameter("name", String::class)
        .returns(String::class)
        .addStatement("return \"Hello, \$name!\"")
        .build()

    val file = FileSpec.builder("com.example", "Greeter")
        .addFunction(greeter)
        .build()

    println(file.toString())
}
```

---

## B.11 추천 라이브러리 모음

### Spring Boot 프로젝트

```kotlin
dependencies {
    // Web
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-webflux")

    // Database
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")

    // Security
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")

    // Monitoring
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")

    // Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")

    // Logging
    implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")

    // Validation
    implementation("org.springframework.boot:spring-boot-starter-validation")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(module = "mockito-core")
    }
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("io.kotest:kotest-runner-junit5:5.7.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
}
```

---

## B.12 Gradle 플러그인

### 추천 플러그인

```kotlin
plugins {
    // Kotlin
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    kotlin("plugin.jpa") version "1.9.21"
    kotlin("plugin.serialization") version "1.9.21"

    // Spring
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"

    // Code Quality
    id("org.jlleitschuh.gradle.ktlint") version "11.6.1"
    id("io.gitlab.arturbosch.detekt") version "1.23.3"

    // Code Coverage
    jacoco
}
```

---

## 핵심 요약

### 카테고리별 추천

1. **비동기**: kotlinx-coroutines
2. **HTTP**: Ktor Client, Fuel
3. **JSON**: kotlinx-serialization
4. **테스트**: MockK, Kotest
5. **로깅**: kotlin-logging
6. **함수형**: Arrow
7. **DB**: Exposed (Type-safe SQL)
8. **Validation**: konform

---

[← 이전](부록A-코틀린_DSL.md) | [다음: 부록 C →](부록C-학습_리소스.md)