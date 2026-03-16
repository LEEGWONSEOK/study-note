# Chapter 18. 코루틴 기초

## 개요

코루틴(Coroutines)은 Kotlin의 **비동기 프로그래밍 솔루션**입니다. 스레드보다 가볍고, 콜백 지옥을 피할 수 있으며, 동기 코드처럼 작성하면서도 비동기로 실행됩니다.

---

## 18.1 코루틴이란?

### 정의

**코루틴**: 실행을 일시 중단(suspend)하고 나중에 재개(resume)할 수 있는 경량 스레드

### 특징

1. **경량**: 수천 개의 코루틴을 동시에 실행 가능
2. **구조화된 동시성**: 코루틴 스코프로 생명주기 관리
3. **동기 스타일**: 비동기 코드를 동기 코드처럼 작성
4. **예외 처리**: 일반 try-catch 사용 가능

---

### 의존성 추가

**build.gradle.kts**:
```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
}
```

---

## 18.2 코루틴과 스레드의 차이

### 스레드의 문제점

```kotlin
// ❌ 스레드 - 무거움
fun withThreads() {
    repeat(100_000) {
        thread {
            Thread.sleep(1000)
            print(".")
        }
    }
}
// OutOfMemoryError 발생 가능!
```

---

### 코루틴의 장점

```kotlin
// ✅ 코루틴 - 가벼움
import kotlinx.coroutines.*

fun withCoroutines() = runBlocking {
    repeat(100_000) {
        launch {
            delay(1000)  // 스레드를 차단하지 않음
            print(".")
        }
    }
}
// 문제없이 실행됨!
```

---

### 비교표

| | 스레드 | 코루틴 |
|--|--------|--------|
| 무게 | 무거움 (1MB+) | 가벼움 (수 KB) |
| 생성 비용 | 높음 | 낮음 |
| 컨텍스트 전환 | 비용 높음 | 비용 낮음 |
| 동시 실행 가능 수 | 제한적 | 수십만 개 |
| 중단/재개 | 불가능 | 가능 |

---

## 18.3 첫 번째 코루틴 (launch, runBlocking)

### runBlocking - 코루틴 실행

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {  // 코루틴 스코프
    println("Hello")
    delay(1000)  // 1초 대기 (non-blocking)
    println("World")
}
```

**출력**:
```
Hello
(1초 대기)
World
```

---

### launch - 새 코루틴 시작

```kotlin
fun main() = runBlocking {
    launch {  // 새 코루틴 시작
        delay(1000)
        println("World!")
    }
    println("Hello")
}
```

**출력**:
```
Hello
(1초 대기)
World!
```

---

### 여러 코루틴 동시 실행

```kotlin
fun main() = runBlocking {
    launch {
        delay(1000)
        println("World 1!")
    }
    launch {
        delay(1500)
        println("World 2!")
    }
    println("Hello")
}
```

**출력**:
```
Hello
(1초 후)
World 1!
(0.5초 후)
World 2!
```

---

## 18.4 suspend 함수

### 개념

**suspend**: 함수를 일시 중단할 수 있음을 표시하는 키워드

```kotlin
suspend fun doSomething() {
    delay(1000)
    println("Done!")
}
```

---

### 규칙

1. **suspend 함수는 코루틴 또는 다른 suspend 함수에서만 호출 가능**

```kotlin
suspend fun fetchData(): String {
    delay(1000)  // suspend 함수
    return "Data"
}

suspend fun processData() {
    val data = fetchData()  // OK
    println(data)
}

fun main() {
    // fetchData()  // 컴파일 에러! suspend 함수는 여기서 호출 불가

    // 코루틴 스코프에서 호출
    runBlocking {
        fetchData()  // OK
    }
}
```

---

### 실전 예제

```kotlin
suspend fun fetchUser(id: Long): User {
    delay(1000)  // API 호출 시뮬레이션
    return User(id, "User $id")
}

suspend fun fetchUserOrders(userId: Long): List<Order> {
    delay(500)  // API 호출 시뮬레이션
    return listOf(Order(1, userId), Order(2, userId))
}

suspend fun getUserWithOrders(id: Long): UserWithOrders {
    val user = fetchUser(id)
    val orders = fetchUserOrders(id)
    return UserWithOrders(user, orders)
}

fun main() = runBlocking {
    val result = getUserWithOrders(123)
    println(result)
}
```

---

## 18.5 코루틴 스코프

### GlobalScope (권장하지 않음)

```kotlin
// ❌ GlobalScope - 생명주기 관리 어려움
GlobalScope.launch {
    delay(1000)
    println("World")
}
println("Hello")
Thread.sleep(2000)  // 메인 스레드가 종료되지 않도록
```

---

### CoroutineScope (권장)

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {  // this: CoroutineScope
    launch {  // this의 자식 코루틴
        delay(1000)
        println("Task from coroutine scope")
    }

    launch {
        delay(500)
        println("Task from another coroutine")
    }

    println("Main coroutine completes")
}
// 모든 자식 코루틴이 완료될 때까지 대기
```

---

### 커스텀 스코프

```kotlin
class MyService {
    private val scope = CoroutineScope(Dispatchers.Default)

    fun doWork() {
        scope.launch {
            delay(1000)
            println("Work done!")
        }
    }

    fun cleanup() {
        scope.cancel()  // 모든 코루틴 취소
    }
}

fun main() = runBlocking {
    val service = MyService()
    service.doWork()
    delay(2000)
    service.cleanup()
}
```

---

### 구조화된 동시성

```kotlin
fun main() = runBlocking {
    launch {  // 부모 코루틴
        launch {  // 자식 1
            delay(1000)
            println("Child 1")
        }

        launch {  // 자식 2
            delay(500)
            println("Child 2")
        }

        println("Parent started")
    }

    println("Main started")
}

// 출력:
// Main started
// Parent started
// Child 2
// Child 1
```

---

## 18.6 Java와 비교: 비동기 처리 방식

### Java - CompletableFuture

**Java**:
```java
public CompletableFuture<User> fetchUser(Long id) {
    return CompletableFuture.supplyAsync(() -> {
        // 시뮬레이션
        try { Thread.sleep(1000); } catch (Exception e) {}
        return new User(id, "User " + id);
    });
}

public CompletableFuture<List<Order>> fetchOrders(Long userId) {
    return CompletableFuture.supplyAsync(() -> {
        try { Thread.sleep(500); } catch (Exception e) {}
        return Arrays.asList(new Order(1, userId), new Order(2, userId));
    });
}

public CompletableFuture<UserWithOrders> getUserWithOrders(Long id) {
    return fetchUser(id).thenCompose(user ->
        fetchOrders(id).thenApply(orders ->
            new UserWithOrders(user, orders)
        )
    );
}
```

---

### Kotlin - 코루틴

**Kotlin**:
```kotlin
suspend fun fetchUser(id: Long): User {
    delay(1000)
    return User(id, "User $id")
}

suspend fun fetchOrders(userId: Long): List<Order> {
    delay(500)
    return listOf(Order(1, userId), Order(2, userId))
}

suspend fun getUserWithOrders(id: Long): UserWithOrders {
    val user = fetchUser(id)
    val orders = fetchOrders(id)
    return UserWithOrders(user, orders)
}
```

**차이점**:
- ✅ 동기 코드처럼 읽기 쉬움
- ✅ 콜백 지옥 없음
- ✅ 간결함

---

### Java - 콜백 지옥

**Java**:
```java
fetchUser(123, user -> {
    fetchOrders(user.getId(), orders -> {
        processOrders(orders, result -> {
            saveResult(result, () -> {
                notifyUser(user, () -> {
                    log("Complete!");
                });
            });
        });
    });
});
```

**Kotlin**:
```kotlin
val user = fetchUser(123)
val orders = fetchOrders(user.id)
val result = processOrders(orders)
saveResult(result)
notifyUser(user)
log("Complete!")
```

---

## 실전 예제

### 예제 1: 병렬 API 호출

```kotlin
data class User(val id: Long, val name: String)
data class Profile(val userId: Long, val bio: String)
data class Settings(val userId: Long, val theme: String)

suspend fun fetchUser(id: Long): User {
    delay(1000)
    return User(id, "User $id")
}

suspend fun fetchProfile(userId: Long): Profile {
    delay(800)
    return Profile(userId, "Bio for user $userId")
}

suspend fun fetchSettings(userId: Long): Settings {
    delay(600)
    return Settings(userId, "Dark")
}

// ❌ 순차 실행 - 느림 (2400ms)
suspend fun loadUserDataSequential(id: Long) {
    val user = fetchUser(id)        // 1000ms
    val profile = fetchProfile(id)  // 800ms
    val settings = fetchSettings(id) // 600ms
    println("$user, $profile, $settings")
}

// ✅ 병렬 실행 - 빠름 (1000ms)
suspend fun loadUserDataParallel(id: Long) = coroutineScope {
    val user = async { fetchUser(id) }
    val profile = async { fetchProfile(id) }
    val settings = async { fetchSettings(id) }

    println("${user.await()}, ${profile.await()}, ${settings.await()}")
}

fun main() = runBlocking {
    println("Sequential:")
    val time1 = measureTimeMillis {
        loadUserDataSequential(1)
    }
    println("Time: ${time1}ms")

    println("\nParallel:")
    val time2 = measureTimeMillis {
        loadUserDataParallel(1)
    }
    println("Time: ${time2}ms")
}
```

---

### 예제 2: 타임아웃

```kotlin
suspend fun fetchDataWithTimeout(): String? {
    return try {
        withTimeout(2000) {  // 2초 타임아웃
            delay(3000)  // 3초 걸림
            "Data"
        }
    } catch (e: TimeoutCancellationException) {
        println("Timeout!")
        null
    }
}

fun main() = runBlocking {
    val result = fetchDataWithTimeout()
    println("Result: $result")
}
// 출력: Timeout!
//      Result: null
```

---

### 예제 3: 반복 작업

```kotlin
fun main() = runBlocking {
    val job = launch {
        repeat(10) { i ->
            println("Job: Iteration $i")
            delay(500)
        }
    }

    delay(2000)
    println("Cancelling job...")
    job.cancel()  // 코루틴 취소
    job.join()    // 취소 완료 대기
    println("Job cancelled")
}
```

---

## 실전 팁

### 💡 Tip 1: runBlocking은 테스트/메인 함수에서만

```kotlin
// ✅ 메인 함수
fun main() = runBlocking {
    // ...
}

// ✅ 테스트
@Test
fun testSomething() = runBlocking {
    // ...
}

// ❌ 프로덕션 코드 (UI 스레드 차단!)
fun onClick() = runBlocking {  // 나쁜 예
    fetchData()
}

// ✅ 프로덕션 코드
fun onClick() {
    viewModelScope.launch {
        fetchData()
    }
}
```

---

### 💡 Tip 2: suspend 함수로 명확한 비동기

```kotlin
// ✅ suspend 함수
suspend fun fetchData(): String {
    delay(1000)
    return "Data"
}

// 호출자가 비동기임을 알 수 있음
val data = fetchData()  // suspend 함수이므로 코루틴에서만 호출 가능

// ❌ 일반 함수 (비동기임을 알기 어려움)
fun fetchDataAsync(): CompletableFuture<String> {
    // ...
}
```

---

### 💡 Tip 3: 구조화된 동시성

```kotlin
// ✅ 구조화된 동시성
suspend fun loadData() = coroutineScope {
    val data1 = async { fetchData1() }
    val data2 = async { fetchData2() }

    combine(data1.await(), data2.await())
}  // 모든 async가 완료될 때까지 대기

// ❌ GlobalScope (생명주기 관리 어려움)
fun loadDataBad() {
    GlobalScope.launch {
        val data = fetchData()
        // 언제 완료될지 모름!
    }
}
```

---

## 연습 문제

### 문제 1: 기본 코루틴 작성

<details>
<summary>정답 보기</summary>

```kotlin
import kotlinx.coroutines.*

suspend fun printDelayed(message: String, delayMs: Long) {
    delay(delayMs)
    println(message)
}

fun main() = runBlocking {
    launch {
        printDelayed("World!", 1000)
    }

    launch {
        printDelayed("Hello", 500)
    }

    println("Start")
}

// 출력:
// Start
// Hello
// World!
```
</details>

### 문제 2: 데이터 가져오기

<details>
<summary>정답 보기</summary>

```kotlin
data class Post(val id: Int, val title: String)
data class Comment(val postId: Int, val text: String)

suspend fun fetchPost(id: Int): Post {
    delay(1000)
    return Post(id, "Post $id")
}

suspend fun fetchComments(postId: Int): List<Comment> {
    delay(500)
    return listOf(
        Comment(postId, "Comment 1"),
        Comment(postId, "Comment 2")
    )
}

suspend fun loadPostWithComments(id: Int) = coroutineScope {
    val post = async { fetchPost(id) }
    val comments = async { fetchComments(id) }

    println("Post: ${post.await()}")
    println("Comments: ${comments.await()}")
}

fun main() = runBlocking {
    loadPostWithComments(1)
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **코루틴 특징**
   - 경량 스레드
   - 일시 중단/재개 가능
   - 구조화된 동시성

2. **기본 빌더**
   - `runBlocking`: 블로킹
   - `launch`: Fire and forget
   - `async`: 결과 반환

3. **suspend 함수**
   - 코루틴에서만 호출
   - `delay`, `await` 등

4. **스코프**
   - `CoroutineScope`
   - 구조화된 동시성

---

## 다음 챕터 예고

Chapter 19에서는 **코루틴 심화**를 다룹니다:
- Job과 취소
- async와 await
- Dispatcher
- 예외 처리
- Channel과 Flow

---

[← 이전: Chapter 17. 고급 컬렉션 연산](../part6-functional/chapter17-advanced-collections.md) | [다음: Chapter 19. 코루틴 심화 →](chapter19-coroutines-advanced.md)