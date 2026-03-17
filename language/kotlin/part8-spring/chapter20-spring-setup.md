# Chapter 20. Spring Boot 프로젝트 설정

## 20.1 Spring Initializr로 Kotlin 프로젝트 생성

### 웹에서 생성

1. [start.spring.io](https://start.spring.io) 접속
2. 설정:
   - Project: Gradle - Kotlin
   - Language: Kotlin
   - Spring Boot: 3.2.x
   - JDK: 17 이상
3. Dependencies 추가:
   - Spring Web
   - Spring Data JPA
   - H2 Database (개발용)
   - Spring Boot DevTools

---

## 20.2 Gradle 설정

**build.gradle.kts**:
```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"  // 필수!
    kotlin("plugin.jpa") version "1.9.21"     // JPA 사용 시
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

## 20.3 필수 플러그인

### kotlin-spring 플러그인

**이유**: Kotlin 클래스는 기본적으로 `final`이지만, Spring은 CGLIB 프록시를 위해 `open`이 필요

```kotlin
// 자동으로 open 추가:
// @Component, @Service, @Repository, @Controller, @RestController,
// @Configuration, @Transactional
```

### kotlin-jpa 플러그인

**이유**: JPA 엔티티를 위한 no-arg 생성자 자동 생성

```kotlin
@Entity
class User(
    @Id @GeneratedValue
    val id: Long = 0,
    val name: String
)
// no-arg 생성자가 자동으로 생성됨
```

---

## 20.4 프로젝트 구조

```
src/
├── main/
│   ├── kotlin/
│   │   └── com/example/demo/
│   │       ├── DemoApplication.kt
│   │       ├── controller/
│   │       ├── service/
│   │       ├── repository/
│   │       ├── domain/
│   │       └── dto/
│   └── resources/
│       ├── application.yml
│       └── static/
└── test/
    └── kotlin/
```

---

## 20.5 Java Spring과 비교

### 메인 클래스

**Java**:
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

**Kotlin**:
```kotlin
@SpringBootApplication
class DemoApplication

fun main(args: Array<String>) {
    runApplication<DemoApplication>(*args)
}
```

---

[다음: Chapter 21 →](chapter21-spring-kotlin.md)