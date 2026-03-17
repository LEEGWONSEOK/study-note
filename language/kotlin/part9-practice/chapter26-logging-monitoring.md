# Chapter 26. 로깅과 모니터링

## 26.1 로깅 기초

### SLF4J + Logback

Spring Boot는 기본적으로 **SLF4J + Logback**을 사용합니다.

**build.gradle.kts** (이미 포함됨):
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    // SLF4J + Logback 자동 포함
}
```

---

## 26.2 Logger 선언

### Java 방식

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    public void createUser() {
        log.info("Creating user...");
    }
}
```

---

### Kotlin 방식 1: Companion Object

```kotlin
import org.slf4j.LoggerFactory

class UserService {

    private val log = LoggerFactory.getLogger(javaClass)

    fun createUser() {
        log.info("Creating user...")
    }
}
```

---

### Kotlin 방식 2: Extension Function (권장)

```kotlin
import org.slf4j.Logger
import org.slf4j.LoggerFactory

inline fun <reified T> T.logger(): Logger =
    LoggerFactory.getLogger(T::class.java)

// 사용
class UserService {
    private val log = logger()

    fun createUser() {
        log.info("Creating user...")
    }
}
```

---

### Kotlin 방식 3: kotlin-logging 라이브러리

**build.gradle.kts**:
```kotlin
dependencies {
    implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")
}
```

**사용**:
```kotlin
import mu.KotlinLogging

private val log = KotlinLogging.logger {}

class UserService {

    fun createUser() {
        log.info { "Creating user..." }
    }

    fun processUser(user: User) {
        // Lazy evaluation
        log.debug { "Processing user: ${user.toDetailedString()}" }
    }
}
```

**장점**:
- Lazy evaluation: 로그 레벨이 맞지 않으면 람다 실행 안 함
- 더 간결함

---

## 26.3 로그 레벨

### 5가지 레벨

```kotlin
class UserService {
    private val log = KotlinLogging.logger {}

    fun example() {
        log.trace { "Trace level: 매우 상세한 정보" }
        log.debug { "Debug level: 디버깅 정보" }
        log.info { "Info level: 일반 정보" }
        log.warn { "Warn level: 경고" }
        log.error { "Error level: 에러" }
    }
}
```

---

### application.yml에서 레벨 설정

```yaml
logging:
  level:
    root: INFO
    com.example.demo: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

---

## 26.4 구조화된 로깅

### MDC (Mapped Diagnostic Context)

```kotlin
import org.slf4j.MDC

class UserService {
    private val log = KotlinLogging.logger {}

    fun createUser(userId: Long) {
        MDC.put("userId", userId.toString())
        try {
            log.info { "Creating user" }
            // 모든 로그에 userId가 자동으로 추가됨
        } finally {
            MDC.clear()
        }
    }
}
```

---

### Filter에서 MDC 설정

```kotlin
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.MDC
import java.util.UUID

@Component
class RequestIdFilter : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val requestId = UUID.randomUUID().toString()
        MDC.put("requestId", requestId)
        try {
            response.addHeader("X-Request-ID", requestId)
            filterChain.doFilter(request, response)
        } finally {
            MDC.clear()
        }
    }
}
```

---

### logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{requestId}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{requestId}] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

---

## 26.5 JSON 로깅

### Logstash Encoder

**build.gradle.kts**:
```kotlin
dependencies {
    implementation("net.logstash.logback:logstash-logback-encoder:7.4")
}
```

---

### logback-spring.xml (JSON)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>requestId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
        </encoder>
    </appender>

    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.json</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.json</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE"/>
        <appender-ref ref="JSON_FILE"/>
    </root>
</configuration>
```

**출력 예시**:
```json
{
  "@timestamp": "2024-01-01T10:00:00.000+00:00",
  "level": "INFO",
  "thread": "http-nio-8080-exec-1",
  "logger": "com.example.demo.UserService",
  "message": "Creating user",
  "requestId": "abc-123-def",
  "userId": "42"
}
```

---

## 26.6 모니터링 - Spring Boot Actuator

### 의존성 추가

**build.gradle.kts**:
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")  // Prometheus용
}
```

---

### application.yml

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
```

---

### Endpoints

| Endpoint | 설명 |
|----------|------|
| `/actuator/health` | 헬스 체크 |
| `/actuator/info` | 애플리케이션 정보 |
| `/actuator/metrics` | 메트릭 목록 |
| `/actuator/prometheus` | Prometheus 형식 메트릭 |

---

## 26.7 커스텀 Health Indicator

```kotlin
import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.stereotype.Component

@Component
class DatabaseHealthIndicator(
    private val userRepository: UserRepository
) : HealthIndicator {

    override fun health(): Health {
        return try {
            val count = userRepository.count()
            Health.up()
                .withDetail("database", "Available")
                .withDetail("userCount", count)
                .build()
        } catch (e: Exception) {
            Health.down()
                .withDetail("database", "Unavailable")
                .withException(e)
                .build()
        }
    }
}
```

**응답**:
```json
{
  "status": "UP",
  "components": {
    "database": {
      "status": "UP",
      "details": {
        "database": "Available",
        "userCount": 150
      }
    }
  }
}
```

---

## 26.8 커스텀 Metrics

### Counter

```kotlin
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Counter
import org.springframework.stereotype.Service

@Service
class UserService(
    private val userRepository: UserRepository,
    private val meterRegistry: MeterRegistry
) {
    private val userCreatedCounter: Counter =
        Counter.builder("users.created")
            .description("Total number of users created")
            .register(meterRegistry)

    fun create(request: CreateUserRequest): UserDto {
        val user = User(request.name, request.email, request.age)
        val saved = userRepository.save(user)

        userCreatedCounter.increment()  // 카운터 증가

        return saved.toDto()
    }
}
```

---

### Timer

```kotlin
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service

@Service
class UserService(
    private val userRepository: UserRepository,
    private val meterRegistry: MeterRegistry
) {
    private val findAllTimer: Timer =
        Timer.builder("users.findAll.time")
            .description("Time taken to find all users")
            .register(meterRegistry)

    fun findAll(): List<UserDto> {
        return findAllTimer.recordCallable {
            userRepository.findAll().map { it.toDto() }
        } ?: emptyList()
    }
}
```

---

### Gauge

```kotlin
import io.micrometer.core.instrument.Gauge
import org.springframework.stereotype.Component

@Component
class UserMetrics(
    private val userRepository: UserRepository,
    meterRegistry: MeterRegistry
) {
    init {
        Gauge.builder("users.total", userRepository) {
            it.count().toDouble()
        }
            .description("Total number of users in database")
            .register(meterRegistry)
    }
}
```

---

## 26.9 @Timed 어노테이션

### AOP로 자동 측정

**의존성**:
```kotlin
dependencies {
    implementation("io.micrometer:micrometer-core")
}
```

**설정**:
```kotlin
import io.micrometer.core.aop.TimedAspect
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class MetricsConfig {

    @Bean
    fun timedAspect(registry: MeterRegistry): TimedAspect =
        TimedAspect(registry)
}
```

**사용**:
```kotlin
import io.micrometer.core.annotation.Timed
import org.springframework.stereotype.Service

@Service
class UserService(
    private val userRepository: UserRepository
) {

    @Timed(value = "users.findAll", description = "Time to find all users")
    fun findAll(): List<UserDto> =
        userRepository.findAll().map { it.toDto() }

    @Timed("users.create")
    fun create(request: CreateUserRequest): UserDto {
        // ...
    }
}
```

---

## 26.10 실전 예제: 통합 모니터링

### Service with Logging & Metrics

```kotlin
import mu.KotlinLogging
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.Timer
import org.slf4j.MDC

private val log = KotlinLogging.logger {}

@Service
@Transactional(readOnly = true)
class UserService(
    private val userRepository: UserRepository,
    meterRegistry: MeterRegistry
) {

    private val createdCounter = Counter.builder("users.created.total")
        .description("Total users created")
        .register(meterRegistry)

    private val errorCounter = Counter.builder("users.errors.total")
        .tag("type", "creation")
        .register(meterRegistry)

    private val findTimer = Timer.builder("users.find.duration")
        .description("Time to find user")
        .register(meterRegistry)

    fun findAll(): List<UserDto> {
        log.debug { "Finding all users" }

        return findTimer.recordCallable {
            val users = userRepository.findAll()
            log.info { "Found ${users.size} users" }
            users.map { it.toDto() }
        } ?: emptyList()
    }

    fun findById(id: Long): UserDto {
        MDC.put("userId", id.toString())
        try {
            log.debug { "Finding user by id: $id" }

            return findTimer.recordCallable {
                userRepository.findById(id)
                    .orElseThrow {
                        log.warn { "User not found: $id" }
                        UserNotFoundException("User not found: $id")
                    }
                    .toDto()
            } ?: throw UserNotFoundException("User not found: $id")
        } finally {
            MDC.clear()
        }
    }

    @Transactional
    fun create(request: CreateUserRequest): UserDto {
        log.info { "Creating user: ${request.email}" }

        try {
            if (userRepository.existsByEmail(request.email)) {
                log.warn { "Email already exists: ${request.email}" }
                errorCounter.increment()
                throw DuplicateEmailException("Email already exists")
            }

            val user = User(request.name, request.email, request.age)
            val saved = userRepository.save(user)

            createdCounter.increment()
            log.info { "User created successfully: ${saved.id}" }

            return saved.toDto()
        } catch (e: Exception) {
            log.error(e) { "Error creating user: ${request.email}" }
            errorCounter.increment()
            throw e
        }
    }
}
```

---

## 26.11 Prometheus + Grafana 연동

### docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

---

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

---

## 실전 팁

### 💡 Tip 1: kotlin-logging 사용

```kotlin
// ✅ kotlin-logging (Lazy)
private val log = KotlinLogging.logger {}
log.debug { "User: ${expensiveOperation()}" }  // DEBUG 레벨이 아니면 실행 안 됨

// ❌ SLF4J (Not lazy)
private val log = LoggerFactory.getLogger(javaClass)
log.debug("User: ${expensiveOperation()}")  // 항상 실행됨
```

---

### 💡 Tip 2: 환경별 로그 설정

**application-dev.yml**:
```yaml
logging:
  level:
    root: DEBUG
    com.example.demo: TRACE
```

**application-prod.yml**:
```yaml
logging:
  level:
    root: WARN
    com.example.demo: INFO
```

---

### 💡 Tip 3: 민감정보 로깅 주의

```kotlin
// ❌ 나쁜 예
log.info { "User password: ${user.password}" }

// ✅ 좋은 예
log.info { "User created: ${user.email}" }
```

---

## 핵심 요약

### 꼭 기억할 것

1. **kotlin-logging**
   - Lazy evaluation
   - 더 간결한 DSL

2. **구조화된 로깅**
   - MDC 활용
   - JSON 형식

3. **Actuator**
   - Health, Metrics
   - Prometheus 연동

4. **커스텀 Metrics**
   - Counter, Timer, Gauge
   - @Timed 어노테이션

---

[← 이전](chapter25-testing.md) | [다음: Chapter 27 →](chapter27-security.md)