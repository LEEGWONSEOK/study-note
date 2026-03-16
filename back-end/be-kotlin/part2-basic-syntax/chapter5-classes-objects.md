# Chapter 5. 클래스와 객체 기초

## 5.1 클래스 선언

### 가장 간단한 클래스

```kotlin
class Person
```

이것만으로도 완전한 클래스입니다! (Java처럼 중괄호조차 필요 없음)

**사용**:
```kotlin
val person = Person()
```

---

### 프로퍼티가 있는 클래스

```kotlin
class Person {
    var name: String = ""
    var age: Int = 0
}

// 사용
val person = Person()
person.name = "홍길동"
person.age = 30

println("${person.name}, ${person.age}세")  // 홍길동, 30세
```

---

### Java와 비교

**Java**:
```java
public class Person {
    private String name = "";
    private int age = 0;

    public Person() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

**Kotlin**:
```kotlin
class Person {
    var name: String = ""
    var age: Int = 0
}
```

**차이점**:
- ✅ Getter/Setter 자동 생성
- ✅ public이 기본
- ✅ 훨씬 간결

---

## 5.2 생성자 (주 생성자와 부 생성자)

### 주 생성자 (Primary Constructor)

클래스 헤더에 직접 선언하는 생성자입니다.

**기본 형태**:
```kotlin
class Person constructor(name: String, age: Int) {
    val name: String = name
    val age: Int = age
}
```

**`constructor` 키워드 생략 가능**:
```kotlin
class Person(name: String, age: Int) {
    val name: String = name
    val age: Int = age
}
```

**더 간결하게 - 프로퍼티 선언과 동시에**:
```kotlin
class Person(val name: String, val age: Int)
```

이것만으로 완전한 클래스입니다!

```kotlin
val person = Person("홍길동", 30)
println("${person.name}, ${person.age}세")  // 홍길동, 30세
```

---

### val vs var in 생성자

```kotlin
class Person(
    val name: String,   // 읽기 전용 (불변)
    var age: Int        // 읽기/쓰기 가능 (가변)
)

val person = Person("홍길동", 30)
println(person.name)  // OK

// person.name = "김철수"  // 컴파일 에러! val은 변경 불가

person.age = 31  // OK (var는 변경 가능)
println(person.age)  // 31
```

---

### 초기화 블록 (init)

주 생성자는 코드를 포함할 수 없으므로 `init` 블록을 사용합니다.

```kotlin
class Person(val name: String, val age: Int) {
    init {
        println("Person 객체 생성: $name, $age세")
        require(age >= 0) { "나이는 0 이상이어야 합니다" }
    }
}

val person = Person("홍길동", 30)
// 출력: Person 객체 생성: 홍길동, 30세

val invalid = Person("test", -1)  // 예외 발생!
```

**여러 init 블록**:
```kotlin
class Person(val name: String, val age: Int) {
    init {
        println("첫 번째 init")
    }

    val uppercaseName = name.uppercase()

    init {
        println("두 번째 init: $uppercaseName")
    }
}

// 실행 순서: 프로퍼티 → init → 프로퍼티 → init
```

---

### 부 생성자 (Secondary Constructor)

추가 생성자가 필요할 때 사용합니다.

```kotlin
class Person(val name: String, val age: Int) {
    var email: String = ""

    // 부 생성자 - 주 생성자를 반드시 호출해야 함
    constructor(name: String, age: Int, email: String) : this(name, age) {
        this.email = email
    }
}

// 사용
val person1 = Person("홍길동", 30)
val person2 = Person("김철수", 25, "kim@example.com")
```

**하지만**: 부 생성자보다 **기본 파라미터**를 사용하는 것이 더 Kotlin 스럽습니다!

```kotlin
// ✅ 더 나은 방법: 기본 파라미터 사용
class Person(
    val name: String,
    val age: Int,
    val email: String = ""  // 기본값
)

val person1 = Person("홍길동", 30)
val person2 = Person("김철수", 25, "kim@example.com")
```

---

### Java와 비교

**Java**:
```java
public class Person {
    private final String name;
    private final int age;
    private String email;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        this.email = "";
    }

    public Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

**Kotlin**:
```kotlin
class Person(
    val name: String,
    val age: Int,
    var email: String = ""
)
```

---

## 5.3 프로퍼티 (Properties)

Kotlin의 프로퍼티 = 필드 + Getter + Setter

### 기본 프로퍼티

```kotlin
class Person {
    var name: String = ""    // var: getter + setter
    val birthYear: Int = 2000  // val: getter만
}

val person = Person()
person.name = "홍길동"  // setter 호출
println(person.name)    // getter 호출
```

---

### Backing Field

```kotlin
class Person(val birthYear: Int) {
    var age: Int = 0
        get() = 2024 - birthYear  // 커스텀 getter
        set(value) {
            field = value  // field는 backing field
        }
}
```

**주의**: `field` 키워드는 getter/setter 내에서만 사용 가능!

---

## 5.4 Getter와 Setter

### 커스텀 Getter

```kotlin
class Rectangle(val width: Int, val height: Int) {
    val area: Int
        get() = width * height  // 매번 계산

    val isSquare: Boolean
        get() = width == height
}

val rect = Rectangle(10, 20)
println(rect.area)      // 200
println(rect.isSquare)  // false
```

**간단한 getter는 단일 표현식으로**:
```kotlin
class Person(val birthYear: Int) {
    val age: Int
        get() = 2024 - birthYear
}
```

---

### 커스텀 Setter

```kotlin
class Person(var name: String) {
    var age: Int = 0
        set(value) {
            if (value < 0) {
                throw IllegalArgumentException("나이는 0 이상이어야 합니다")
            }
            field = value
        }
}

val person = Person("홍길동")
person.age = 30   // OK
person.age = -1   // 예외 발생!
```

---

### 가시성 변경

**Setter만 private**:
```kotlin
class Person(name: String) {
    var name: String = name
        private set  // setter는 private

    fun changeName(newName: String) {
        // 클래스 내부에서만 변경 가능
        name = newName
    }
}

val person = Person("홍길동")
println(person.name)  // OK (getter는 public)
// person.name = "김철수"  // 컴파일 에러! (setter는 private)
person.changeName("김철수")  // OK
```

---

### Java와 비교

**Java**:
```java
public class Rectangle {
    private int width;
    private int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    public int getWidth() { return width; }
    public int getHeight() { return height; }

    public int getArea() {
        return width * height;
    }

    public boolean isSquare() {
        return width == height;
    }
}
```

**Kotlin**:
```kotlin
class Rectangle(val width: Int, val height: Int) {
    val area get() = width * height
    val isSquare get() = width == height
}
```

---

## 5.5 데이터 클래스 (Data Class)

데이터를 담는 클래스를 위한 **가장 강력한 기능**!

### 기본 사용

```kotlin
data class Person(
    val name: String,
    val age: Int,
    val email: String
)
```

**자동으로 생성되는 것**:
1. `equals()` / `hashCode()`
2. `toString()`
3. `copy()`
4. `componentN()` (구조 분해)

---

### toString()

```kotlin
data class Person(val name: String, val age: Int)

val person = Person("홍길동", 30)
println(person)
// 출력: Person(name=홍길동, age=30)
```

**일반 클래스**는 `Person@1f32e575` 같은 해시코드만 출력됩니다.

---

### equals()와 hashCode()

```kotlin
data class Person(val name: String, val age: Int)

val person1 = Person("홍길동", 30)
val person2 = Person("홍길동", 30)
val person3 = Person("김철수", 25)

println(person1 == person2)  // true (내용 비교!)
println(person1 == person3)  // false

// Set에 넣기 (hashCode 활용)
val set = setOf(person1, person2, person3)
println(set.size)  // 2 (person1과 person2는 동일하게 취급)
```

---

### copy()

불변 객체의 일부만 변경한 새 객체를 만듭니다.

```kotlin
data class Person(val name: String, val age: Int, val email: String)

val person = Person("홍길동", 30, "hong@example.com")

// 나이만 변경
val olderPerson = person.copy(age = 31)
println(olderPerson)
// Person(name=홍길동, age=31, email=hong@example.com)

// 여러 프로퍼티 변경
val different = person.copy(name = "김철수", age = 25)
println(different)
// Person(name=김철수, age=25, email=hong@example.com)
```

---

### 구조 분해 (Destructuring)

```kotlin
data class Person(val name: String, val age: Int, val email: String)

val person = Person("홍길동", 30, "hong@example.com")

// 구조 분해
val (name, age, email) = person
println("이름: $name, 나이: $age, 이메일: $email")
// 이름: 홍길동, 나이: 30, 이메일: hong@example.com

// 일부만 사용
val (name2, age2) = person
println("$name2는 ${age2}세입니다")

// 사용하지 않는 변수는 _ 로
val (name3, _, email3) = person
```

**for 루프에서 활용**:
```kotlin
data class Person(val name: String, val age: Int)

val people = listOf(
    Person("홍길동", 30),
    Person("김철수", 25),
    Person("이영희", 28)
)

for ((name, age) in people) {
    println("$name: ${age}세")
}
```

---

### Java와 비교

**Java** (Lombok 없이):
```java
public class Person {
    private final String name;
    private final int age;
    private final String email;

    public Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public String getEmail() { return email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age &&
               Objects.equals(name, person.name) &&
               Objects.equals(email, person.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age, email);
    }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age +
               ", email='" + email + "'}";
    }

    // copy 메소드도 직접 작성해야 함
    public Person withAge(int newAge) {
        return new Person(this.name, newAge, this.email);
    }
}
```

**Kotlin**:
```kotlin
data class Person(val name: String, val age: Int, val email: String)
```

**56줄 → 1줄!**

---

## 5.6 object 키워드 (싱글톤, 동반 객체)

### Object Declaration (싱글톤)

**싱글톤 패턴**을 언어 차원에서 지원!

```kotlin
object DatabaseConfig {
    val url = "jdbc:mysql://localhost:3306/mydb"
    val username = "admin"
    val password = "secret"

    fun connect() {
        println("Connecting to $url")
    }
}

// 사용 - 인스턴스 생성 없이 바로 사용
println(DatabaseConfig.url)
DatabaseConfig.connect()
```

---

### Java와 비교

**Java**:
```java
public class DatabaseConfig {
    private static final DatabaseConfig INSTANCE = new DatabaseConfig();

    private final String url;
    private final String username;
    private final String password;

    private DatabaseConfig() {
        this.url = "jdbc:mysql://localhost:3306/mydb";
        this.username = "admin";
        this.password = "secret";
    }

    public static DatabaseConfig getInstance() {
        return INSTANCE;
    }

    public void connect() {
        System.out.println("Connecting to " + url);
    }
}

// 사용
DatabaseConfig.getInstance().connect();
```

**Kotlin**:
```kotlin
object DatabaseConfig {
    val url = "jdbc:mysql://localhost:3306/mydb"

    fun connect() {
        println("Connecting to $url")
    }
}

// 사용
DatabaseConfig.connect()
```

---

### Companion Object (동반 객체)

클래스 내부의 **정적 멤버**를 위한 공간

```kotlin
class User(val name: String, val email: String) {
    companion object {
        const val MIN_AGE = 18
        const val MAX_NAME_LENGTH = 50

        fun create(name: String, email: String): User {
            require(name.length <= MAX_NAME_LENGTH)
            return User(name, email)
        }
    }
}

// 사용
println(User.MIN_AGE)  // 18
val user = User.create("홍길동", "hong@example.com")
```

---

### Companion Object with Name

```kotlin
class User(val name: String) {
    companion object Factory {
        fun create(name: String): User {
            return User(name)
        }
    }
}

// 사용
val user1 = User.create("홍길동")
val user2 = User.Factory.create("김철수")  // 이름으로도 접근 가능
```

---

### Java와 비교

**Java**:
```java
public class User {
    public static final int MIN_AGE = 18;
    public static final int MAX_NAME_LENGTH = 50;

    private String name;
    private String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public static User create(String name, String email) {
        if (name.length() > MAX_NAME_LENGTH) {
            throw new IllegalArgumentException();
        }
        return new User(name, email);
    }
}

// 사용
User user = User.create("홍길동", "hong@example.com");
```

**Kotlin**:
```kotlin
class User(val name: String, val email: String) {
    companion object {
        const val MIN_AGE = 18
        const val MAX_NAME_LENGTH = 50

        fun create(name: String, email: String): User {
            require(name.length <= MAX_NAME_LENGTH)
            return User(name, email)
        }
    }
}
```

---

### Object Expression (익명 객체)

**일회성 객체**를 만들 때 사용

```kotlin
interface ClickListener {
    fun onClick()
}

val button = object : ClickListener {
    override fun onClick() {
        println("Button clicked!")
    }
}

button.onClick()
```

**Java의 익명 클래스**와 유사하지만 더 간결합니다.

---

## 5.7 Java와 비교: 클래스 작성법

### DTO 클래스 비교

**시나리오**: 사용자 정보를 담는 DTO

**Java**:
```java
public class UserDto {
    private final String name;
    private final int age;
    private final String email;
    private final boolean active;

    public UserDto(String name, int age, String email, boolean active) {
        this.name = name;
        this.age = age;
        this.email = email;
        this.active = active;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public String getEmail() { return email; }
    public boolean isActive() { return active; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        UserDto userDto = (UserDto) o;
        return age == userDto.age &&
               active == userDto.active &&
               Objects.equals(name, userDto.name) &&
               Objects.equals(email, userDto.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age, email, active);
    }

    @Override
    public String toString() {
        return "UserDto{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", email='" + email + '\'' +
                ", active=" + active +
                '}';
    }
}
```

**Kotlin**:
```kotlin
data class UserDto(
    val name: String,
    val age: Int,
    val email: String,
    val active: Boolean
)
```

**47줄 → 5줄!**

---

## 실전 팁

### 💡 Tip 1: 데이터 클래스를 적극 활용하세요

```kotlin
// ✅ DTO, Response, Request 모델은 모두 data class
data class UserRequest(val name: String, val email: String)
data class UserResponse(val id: Long, val name: String)
data class ApiResult<T>(val success: Boolean, val data: T?)
```

### 💡 Tip 2: val을 기본으로, var는 필요할 때만

```kotlin
// ✅ 불변 객체 (권장)
data class User(val name: String, val email: String)

// ❌ 가변 객체 (꼭 필요한 경우만)
data class Counter(var count: Int)
```

### 💡 Tip 3: Companion Object로 팩토리 메소드 패턴

```kotlin
data class User private constructor(
    val name: String,
    val email: String
) {
    companion object {
        fun create(name: String, email: String): User {
            require(name.isNotBlank()) { "이름은 필수입니다" }
            require(email.contains("@")) { "올바른 이메일이 아닙니다" }
            return User(name, email)
        }

        fun createGuest(): User {
            return User("Guest", "guest@example.com")
        }
    }
}

val user = User.create("홍길동", "hong@example.com")
val guest = User.createGuest()
```

### 💡 Tip 4: copy()로 불변성 유지하며 변경

```kotlin
data class Order(
    val id: Long,
    val items: List<String>,
    val status: String
)

val order = Order(1, listOf("Item1", "Item2"), "PENDING")

// 상태만 변경한 새 객체
val processedOrder = order.copy(status = "PROCESSING")
val completedOrder = processedOrder.copy(status = "COMPLETED")

// 원본은 변경되지 않음
println(order.status)  // PENDING
```

---

## 연습 문제

### 문제 1: 도서 클래스
제목, 저자, 가격을 프로퍼티로 갖는 Book 데이터 클래스를 작성하세요.

<details>
<summary>정답 보기</summary>

```kotlin
data class Book(
    val title: String,
    val author: String,
    val price: Int
)

fun main() {
    val book = Book("Kotlin In Action", "Dmitry Jemerov", 35000)
    println(book)

    // 할인된 가격의 새 객체
    val discounted = book.copy(price = 28000)
    println(discounted)
}
```
</details>

### 문제 2: 사각형 클래스
너비와 높이를 받아서 넓이와 둘레를 계산하는 Rectangle 클래스를 작성하세요.

<details>
<summary>정답 보기</summary>

```kotlin
class Rectangle(val width: Int, val height: Int) {
    val area: Int
        get() = width * height

    val perimeter: Int
        get() = 2 * (width + height)

    val isSquare: Boolean
        get() = width == height
}

fun main() {
    val rect = Rectangle(10, 20)
    println("넓이: ${rect.area}")       // 200
    println("둘레: ${rect.perimeter}")  // 60
    println("정사각형? ${rect.isSquare}")  // false

    val square = Rectangle(10, 10)
    println("정사각형? ${square.isSquare}")  // true
}
```
</details>

### 문제 3: 싱글톤 설정 객체
애플리케이션 설정을 담는 싱글톤 객체를 작성하세요.

<details>
<summary>정답 보기</summary>

```kotlin
object AppConfig {
    const val APP_NAME = "MyApp"
    const val VERSION = "1.0.0"

    var debugMode: Boolean = false
    var maxRetries: Int = 3

    fun printConfig() {
        println("=== $APP_NAME v$VERSION ===")
        println("Debug Mode: $debugMode")
        println("Max Retries: $maxRetries")
    }
}

fun main() {
    AppConfig.printConfig()

    AppConfig.debugMode = true
    AppConfig.maxRetries = 5

    AppConfig.printConfig()
}
```
</details>

### 문제 4: 사용자 팩토리
이메일과 비밀번호로 사용자를 생성하는 팩토리 메소드를 companion object에 작성하세요. 유효성 검사도 포함하세요.

<details>
<summary>정답 보기</summary>

```kotlin
data class User(
    val email: String,
    val passwordHash: String
) {
    companion object {
        private const val MIN_PASSWORD_LENGTH = 8

        fun create(email: String, password: String): User {
            require(email.contains("@")) {
                "유효한 이메일 주소가 아닙니다"
            }

            require(password.length >= MIN_PASSWORD_LENGTH) {
                "비밀번호는 최소 ${MIN_PASSWORD_LENGTH}자 이상이어야 합니다"
            }

            val passwordHash = password.hashCode().toString()
            return User(email, passwordHash)
        }
    }
}

fun main() {
    val user = User.create("hong@example.com", "password123")
    println(user)

    // 예외 발생
    // val invalid = User.create("invalid-email", "short")
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **주 생성자**
   - 클래스 헤더에 선언
   - `val`/`var`로 프로퍼티 동시 선언

2. **데이터 클래스**
   - `data class` 키워드
   - toString, equals, hashCode, copy 자동 생성
   - DTO에 최적

3. **프로퍼티**
   - 자동 getter/setter
   - 커스텀 가능

4. **object**
   - 싱글톤: `object`
   - 정적 멤버: `companion object`

5. **Java 대비 장점**
   - 훨씬 간결
   - Getter/Setter 자동
   - 불변성 쉽게 구현

---

## Part 2 완료!

Part 2: Kotlin 기초 문법을 모두 마쳤습니다!

**지금까지 배운 내용**:
- ✅ Chapter 3: 제어 흐름 (if, when, for, while)
- ✅ Chapter 4: 함수 (기본 파라미터, 확장 함수, 중위 함수)
- ✅ Chapter 5: 클래스와 객체 (생성자, 데이터 클래스, object)

---

## 다음 Part 예고

**Part 3: Kotlin 핵심 개념**에서 다룰 내용:
- Chapter 6: Null 안전성 (Nullable 타입, 안전 호출, Elvis 연산자)
- Chapter 7: 컬렉션 (List, Set, Map, 연산)
- Chapter 8: 람다와 고차 함수

Kotlin의 가장 강력한 기능들을 배울 차례입니다!

---

[← 이전: Chapter 4. 함수](chapter4-functions.md) | [다음: Chapter 6. Null 안전성 →](../part3-core-concepts/chapter6-null-safety.md)