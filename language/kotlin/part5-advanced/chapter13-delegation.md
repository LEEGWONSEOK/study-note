# Chapter 13. 델리게이션

## 개요

델리게이션(Delegation)은 상속의 대안으로, 다른 객체에게 작업을 위임하는 패턴입니다. Kotlin은 언어 차원에서 델리게이션을 지원합니다.

---

## 13.1 클래스 델리게이션 (by)

### 기본 개념

**"상속보다 합성을 우선하라"** - Effective Java

```kotlin
interface Printer {
    fun print(message: String)
}

class ConsolePrinter : Printer {
    override fun print(message: String) {
        println("Console: $message")
    }
}

// 수동 위임 (Java 스타일)
class ManualLogger(private val printer: Printer) : Printer {
    override fun print(message: String) {
        printer.print(message)  // 수동으로 위임
    }
}

// Kotlin 델리게이션 (by)
class AutoLogger(printer: Printer) : Printer by printer

fun main() {
    val printer = ConsolePrinter()

    val manualLogger = ManualLogger(printer)
    manualLogger.print("Manual")  // Console: Manual

    val autoLogger = AutoLogger(printer)
    autoLogger.print("Auto")  // Console: Auto
}
```

---

### 일부 메소드 오버라이드

```kotlin
interface Repository {
    fun save(data: String)
    fun load(): String
    fun delete()
}

class DatabaseRepository : Repository {
    private var data: String = ""

    override fun save(data: String) {
        println("Saving to database: $data")
        this.data = data
    }

    override fun load(): String {
        println("Loading from database")
        return data
    }

    override fun delete() {
        println("Deleting from database")
        data = ""
    }
}

// 대부분은 위임하되, 일부만 오버라이드
class LoggingRepository(
    private val repository: Repository
) : Repository by repository {

    override fun save(data: String) {
        println("[LOG] Saving: $data")
        repository.save(data)  // 원본 호출
        println("[LOG] Save completed")
    }

    // load()와 delete()는 자동으로 위임됨
}

fun main() {
    val db = DatabaseRepository()
    val logger = LoggingRepository(db)

    logger.save("Important data")
    // [LOG] Saving: Important data
    // Saving to database: Important data
    // [LOG] Save completed

    println(logger.load())
    // Loading from database
    // Important data
}
```

---

### 실전 예제: Decorator 패턴

```kotlin
interface DataSource {
    fun readData(): String
    fun writeData(data: String)
}

class FileDataSource(private val filename: String) : DataSource {
    override fun readData(): String {
        println("Reading from file: $filename")
        return "File content"
    }

    override fun writeData(data: String) {
        println("Writing to file: $filename - $data")
    }
}

// 암호화 데코레이터
class EncryptionDecorator(
    private val dataSource: DataSource
) : DataSource by dataSource {

    override fun readData(): String {
        val data = dataSource.readData()
        return decrypt(data)
    }

    override fun writeData(data: String) {
        val encrypted = encrypt(data)
        dataSource.writeData(encrypted)
    }

    private fun encrypt(data: String) = "Encrypted($data)"
    private fun decrypt(data: String) = data.replace("Encrypted(", "").replace(")", "")
}

// 압축 데코레이터
class CompressionDecorator(
    private val dataSource: DataSource
) : DataSource by dataSource {

    override fun readData(): String {
        val data = dataSource.readData()
        return decompress(data)
    }

    override fun writeData(data: String) {
        val compressed = compress(data)
        dataSource.writeData(compressed)
    }

    private fun compress(data: String) = "Compressed($data)"
    private fun decompress(data: String) = data.replace("Compressed(", "").replace(")", "")
}

fun main() {
    val file = FileDataSource("data.txt")
    val encrypted = EncryptionDecorator(file)
    val compressed = CompressionDecorator(encrypted)

    compressed.writeData("Secret message")
    // Writing to file: data.txt - Compressed(Encrypted(Secret message))

    println(compressed.readData())
    // Reading from file: data.txt
    // File content
}
```

---

## 13.2 프로퍼티 델리게이션

### 기본 사용법

```kotlin
import kotlin.properties.Delegates

class User {
    var name: String by Delegates.notNull()

    var age: Int by Delegates.observable(0) { property, oldValue, newValue ->
        println("${property.name} changed from $oldValue to $newValue")
    }

    var email: String by Delegates.vetoable("") { property, oldValue, newValue ->
        println("Trying to change ${property.name} from $oldValue to $newValue")
        newValue.contains("@")  // @ 포함되어야 변경 허용
    }
}

fun main() {
    val user = User()

    user.name = "Alice"
    println(user.name)  // Alice

    user.age = 25
    // age changed from 0 to 25

    user.age = 30
    // age changed from 25 to 30

    user.email = "invalid"
    // Trying to change email from  to invalid
    println(user.email)  // (변경 거부)

    user.email = "alice@example.com"
    // Trying to change email from  to alice@example.com
    println(user.email)  // alice@example.com
}
```

---

## 13.3 lazy 델리게이션

### 개념

**lazy**: 처음 접근할 때 초기화되는 프로퍼티

```kotlin
class ExpensiveResource {
    init {
        println("Creating expensive resource...")
        Thread.sleep(1000)  // 비용이 큰 초기화
    }

    fun use() {
        println("Using resource")
    }
}

class Manager {
    val resource: ExpensiveResource by lazy {
        println("Initializing lazy property")
        ExpensiveResource()
    }

    fun doWork() {
        println("Starting work")
        resource.use()  // 처음 접근 시 초기화
    }
}

fun main() {
    println("Creating manager")
    val manager = Manager()

    println("Manager created")
    Thread.sleep(2000)

    println("Calling doWork")
    manager.doWork()
    // Creating manager
    // Manager created
    // (2초 대기)
    // Calling doWork
    // Initializing lazy property
    // Creating expensive resource...
    // (1초 대기)
    // Using resource
}
```

---

### lazy 모드

```kotlin
// 기본 모드: SYNCHRONIZED (스레드 안전)
val lazyValue1: String by lazy {
    println("Computed once (synchronized)")
    "Value 1"
}

// PUBLICATION 모드: 여러 스레드에서 초기화 가능하지만 하나만 사용
val lazyValue2: String by lazy(LazyThreadSafetyMode.PUBLICATION) {
    println("Computed (publication)")
    "Value 2"
}

// NONE 모드: 스레드 안전 보장 안 함 (싱글 스레드 환경)
val lazyValue3: String by lazy(LazyThreadSafetyMode.NONE) {
    println("Computed (none)")
    "Value 3"
}
```

---

### 실전 예제: 설정 로딩

```kotlin
class AppConfig {
    val databaseUrl: String by lazy {
        println("Loading database URL from config")
        loadFromConfig("db.url")
    }

    val apiKey: String by lazy {
        println("Loading API key from config")
        loadFromConfig("api.key")
    }

    val maxConnections: Int by lazy {
        println("Loading max connections from config")
        loadFromConfig("max.connections").toInt()
    }

    private fun loadFromConfig(key: String): String {
        // 실제로는 파일이나 환경 변수에서 읽음
        return when (key) {
            "db.url" -> "jdbc:mysql://localhost:3306/mydb"
            "api.key" -> "secret-api-key-12345"
            "max.connections" -> "10"
            else -> ""
        }
    }
}

fun main() {
    val config = AppConfig()

    println("Config created")

    // 필요할 때만 로드됨
    println("Database URL: ${config.databaseUrl}")
    // Loading database URL from config
    // Database URL: jdbc:mysql://localhost:3306/mydb

    println("Database URL again: ${config.databaseUrl}")
    // Database URL again: jdbc:mysql://localhost:3306/mydb (다시 로드 안 함)
}
```

---

## 13.4 observable 델리게이션

### 값 변경 감지

```kotlin
import kotlin.properties.Delegates

class Product {
    var name: String by Delegates.observable("") { property, oldValue, newValue ->
        println("${property.name}: '$oldValue' -> '$newValue'")
    }

    var price: Double by Delegates.observable(0.0) { _, old, new ->
        if (new > old) {
            println("Price increased: $old -> $new")
        } else if (new < old) {
            println("Price decreased: $old -> $new")
        }
    }

    var stock: Int by Delegates.observable(0) { _, old, new ->
        when {
            old == 0 && new > 0 -> println("Product back in stock!")
            new == 0 -> println("Product out of stock!")
            new < 10 -> println("Low stock warning: $new items left")
        }
    }
}

fun main() {
    val product = Product()

    product.name = "Laptop"
    // name: '' -> 'Laptop'

    product.price = 1000.0
    // Price increased: 0.0 -> 1000.0

    product.price = 900.0
    // Price decreased: 1000.0 -> 900.0

    product.stock = 50
    product.stock = 5
    // Low stock warning: 5 items left

    product.stock = 0
    // Product out of stock!
}
```

---

### vetoable - 변경 승인/거부

```kotlin
import kotlin.properties.Delegates

class BankAccount(initialBalance: Double) {
    var balance: Double by Delegates.vetoable(initialBalance) { _, _, newValue ->
        newValue >= 0  // 0 이상일 때만 변경 허용
    }

    fun withdraw(amount: Double): Boolean {
        val newBalance = balance - amount
        balance = newBalance
        return balance == newBalance
    }

    fun deposit(amount: Double) {
        balance += amount
    }
}

fun main() {
    val account = BankAccount(1000.0)

    println("Balance: ${account.balance}")  // 1000.0

    account.deposit(500.0)
    println("After deposit: ${account.balance}")  // 1500.0

    val success = account.withdraw(2000.0)
    println("Withdrawal success: $success")  // false
    println("Balance: ${account.balance}")  // 1500.0 (변경 거부됨)

    account.withdraw(500.0)
    println("After withdrawal: ${account.balance}")  // 1000.0
}
```

---

## 13.5 커스텀 델리게이션 만들기

### ReadOnlyProperty

```kotlin
import kotlin.reflect.KProperty

class UppercaseDelegate {
    private var value: String = ""

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return value.uppercase()
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: String) {
        value = newValue
    }
}

class Message {
    var text: String by UppercaseDelegate()
}

fun main() {
    val message = Message()
    message.text = "hello world"

    println(message.text)  // HELLO WORLD
}
```

---

### 커스텀 델리게이트: 로깅

```kotlin
import kotlin.reflect.KProperty

class LoggedProperty<T>(private var value: T) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        println("[GET] ${property.name} = $value")
        return value
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        println("[SET] ${property.name}: $value -> $newValue")
        value = newValue
    }
}

fun <T> logged(initialValue: T) = LoggedProperty(initialValue)

class User {
    var name: String by logged("Unknown")
    var age: Int by logged(0)
}

fun main() {
    val user = User()

    user.name = "Alice"
    // [SET] name: Unknown -> Alice

    println(user.name)
    // [GET] name = Alice
    // Alice

    user.age = 25
    // [SET] age: 0 -> 25
}
```

---

### 실전 예제: Preferences 델리게이트

```kotlin
import kotlin.reflect.KProperty

class Preferences {
    private val map = mutableMapOf<String, Any>()

    inner class Preference<T>(private val default: T) {
        operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
            @Suppress("UNCHECKED_CAST")
            return map.getOrDefault(property.name, default) as T
        }

        operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
            map[property.name] = value as Any
        }
    }

    fun <T> preference(default: T) = Preference(default)
}

class AppSettings : Preferences() {
    var username: String by preference("guest")
    var fontSize: Int by preference(14)
    var darkMode: Boolean by preference(false)
}

fun main() {
    val settings = AppSettings()

    println("Username: ${settings.username}")  // guest
    println("Font size: ${settings.fontSize}")  // 14

    settings.username = "alice"
    settings.fontSize = 16
    settings.darkMode = true

    println("Updated username: ${settings.username}")  // alice
    println("Updated font size: ${settings.fontSize}")  // 16
    println("Dark mode: ${settings.darkMode}")  // true
}
```

---

## 13.6 Java와 비교: 위임 패턴

### Java 수동 위임

**Java**:
```java
interface Printer {
    void print(String message);
}

class ConsolePrinter implements Printer {
    public void print(String message) {
        System.out.println("Console: " + message);
    }
}

class Logger implements Printer {
    private final Printer printer;

    public Logger(Printer printer) {
        this.printer = printer;
    }

    @Override
    public void print(String message) {
        // 수동으로 모든 메소드를 위임해야 함
        printer.print(message);
    }
}
```

**Kotlin**:
```kotlin
interface Printer {
    fun print(message: String)
}

class ConsolePrinter : Printer {
    override fun print(message: String) {
        println("Console: $message")
    }
}

// 자동 위임
class Logger(printer: Printer) : Printer by printer
```

---

## 실전 팁

### 💡 Tip 1: lazy로 비용 절감

```kotlin
class DataProcessor {
    // ✅ 필요할 때만 초기화
    val expensiveData: List<String> by lazy {
        println("Loading expensive data...")
        loadLargeDataset()
    }

    // ❌ 항상 초기화 (불필요할 수도)
    // val eagerData: List<String> = loadLargeDataset()

    private fun loadLargeDataset(): List<String> {
        Thread.sleep(2000)
        return listOf("data1", "data2", "data3")
    }
}
```

---

### 💡 Tip 2: observable로 상태 동기화

```kotlin
class ViewModel {
    var isLoading: Boolean by Delegates.observable(false) { _, _, newValue ->
        if (newValue) {
            showLoadingSpinner()
        } else {
            hideLoadingSpinner()
        }
    }

    var errorMessage: String? by Delegates.observable(null) { _, _, newValue ->
        newValue?.let { showError(it) } ?: hideError()
    }

    private fun showLoadingSpinner() = println("Showing spinner")
    private fun hideLoadingSpinner() = println("Hiding spinner")
    private fun showError(message: String) = println("Error: $message")
    private fun hideError() = println("Hiding error")
}
```

---

### 💡 Tip 3: Map 델리게이션

```kotlin
class User(val map: Map<String, Any>) {
    val name: String by map
    val age: Int by map
    val email: String by map
}

fun main() {
    val user = User(mapOf(
        "name" to "Alice",
        "age" to 25,
        "email" to "alice@example.com"
    ))

    println("${user.name}, ${user.age}, ${user.email}")
    // Alice, 25, alice@example.com
}
```

---

## 연습 문제

### 문제 1: 클래스 델리게이션으로 캐싱 추가

<details>
<summary>정답 보기</summary>

```kotlin
interface Calculator {
    fun add(a: Int, b: Int): Int
    fun multiply(a: Int, b: Int): Int
}

class SimpleCalculator : Calculator {
    override fun add(a: Int, b: Int): Int {
        println("Computing $a + $b")
        return a + b
    }

    override fun multiply(a: Int, b: Int): Int {
        println("Computing $a * $b")
        return a * b
    }
}

class CachingCalculator(
    private val calculator: Calculator
) : Calculator by calculator {

    private val cache = mutableMapOf<String, Int>()

    override fun add(a: Int, b: Int): Int {
        val key = "add:$a:$b"
        return cache.getOrPut(key) {
            calculator.add(a, b)
        }
    }

    override fun multiply(a: Int, b: Int): Int {
        val key = "mul:$a:$b"
        return cache.getOrPut(key) {
            calculator.multiply(a, b)
        }
    }
}

fun main() {
    val calc = CachingCalculator(SimpleCalculator())

    println(calc.add(2, 3))  // Computing 2 + 3 -> 5
    println(calc.add(2, 3))  // 5 (캐시에서)

    println(calc.multiply(4, 5))  // Computing 4 * 5 -> 20
    println(calc.multiply(4, 5))  // 20 (캐시에서)
}
```
</details>

### 문제 2: 범위 검증 델리게이트

<details>
<summary>정답 보기</summary>

```kotlin
import kotlin.properties.Delegates
import kotlin.reflect.KProperty

class RangeValidator<T : Comparable<T>>(
    private var value: T,
    private val range: ClosedRange<T>
) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        require(newValue in range) {
            "${property.name} must be in range $range, but was $newValue"
        }
        value = newValue
    }
}

fun <T : Comparable<T>> rangeValidated(initial: T, range: ClosedRange<T>) =
    RangeValidator(initial, range)

class GameCharacter {
    var health: Int by rangeValidated(100, 0..100)
    var level: Int by rangeValidated(1, 1..99)
}

fun main() {
    val character = GameCharacter()

    println("Health: ${character.health}")  // 100
    println("Level: ${character.level}")    // 1

    character.health = 80  // OK
    character.level = 50   // OK

    try {
        character.health = 150  // 예외!
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }

    try {
        character.level = 0  // 예외!
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }
}
```
</details>

### 문제 3: 변경 이력 추적 델리게이트

<details>
<summary>정답 보기</summary>

```kotlin
import kotlin.reflect.KProperty

class HistoryTracker<T>(private var value: T) {
    private val history = mutableListOf<T>()

    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        history.add(value)
        value = newValue
    }

    fun getHistory(): List<T> = history.toList()

    fun undo(): T? {
        return if (history.isNotEmpty()) {
            value = history.removeAt(history.lastIndex)
            value
        } else {
            null
        }
    }
}

fun <T> tracked(initial: T) = HistoryTracker(initial)

class Document {
    var content: String by tracked("Initial content")

    fun getContentHistory() = (content as? HistoryTracker<String>)?.getHistory()
}

fun main() {
    val doc = Document()

    println(doc.content)  // Initial content

    doc.content = "First edit"
    doc.content = "Second edit"
    doc.content = "Third edit"

    println("Current: ${doc.content}")
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **클래스 델리게이션**
   - `by` 키워드
   - 상속의 대안
   - 일부 메소드 오버라이드 가능

2. **프로퍼티 델리게이션**
   - `by` 키워드
   - `getValue`/`setValue` 구현

3. **표준 델리게이트**
   - `lazy`: 지연 초기화
   - `observable`: 변경 감지
   - `vetoable`: 변경 승인/거부

4. **커스텀 델리게이트**
   - `getValue`/`setValue` 구현
   - 재사용 가능한 로직

5. **장점**
   - 코드 재사용
   - 관심사 분리
   - 보일러플레이트 감소

---

## 다음 챕터 예고

Chapter 14에서는 **연산자 오버로딩**을 다룹니다:
- 산술 연산자
- 비교 연산자
- 인덱스 접근 연산자
- invoke 연산자

---

[← 이전: Chapter 12. 제네릭](chapter12-generics.md) | [다음: Chapter 14. 연산자 오버로딩 →](chapter14-operator-overloading.md)