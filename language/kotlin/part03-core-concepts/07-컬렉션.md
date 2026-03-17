# Chapter 7. 컬렉션

## 개요

Kotlin의 컬렉션은 **불변(Immutable)**과 **가변(Mutable)**을 명확히 구분하며, 풍부한 연산 함수를 제공합니다.

---

## 7.1 List, Set, Map

### List - 순서가 있는 컬렉션

**불변 List**:
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
println(numbers)  // [1, 2, 3, 4, 5]

// 읽기 전용
println(numbers[0])     // 1
println(numbers.size)   // 5
println(numbers.first())  // 1
println(numbers.last())   // 5

// numbers.add(6)  // 컴파일 에러! 불변이므로 추가 불가
```

**가변 List**:
```kotlin
val mutableNumbers = mutableListOf(1, 2, 3)
mutableNumbers.add(4)
mutableNumbers.remove(2)
mutableNumbers[0] = 10

println(mutableNumbers)  // [10, 3, 4]
```

---

### Set - 중복 없는 컬렉션

**불변 Set**:
```kotlin
val numbers = setOf(1, 2, 3, 3, 2, 1)
println(numbers)  // [1, 2, 3] - 중복 제거됨

println(1 in numbers)  // true
println(5 in numbers)  // false
```

**가변 Set**:
```kotlin
val mutableNumbers = mutableSetOf(1, 2, 3)
mutableNumbers.add(4)
mutableNumbers.add(2)  // 이미 있으므로 추가 안 됨
mutableNumbers.remove(1)

println(mutableNumbers)  // [2, 3, 4]
```

---

### Map - Key-Value 쌍

**불변 Map**:
```kotlin
val countryCode = mapOf(
    "KR" to "South Korea",
    "US" to "United States",
    "JP" to "Japan"
)

println(countryCode["KR"])  // South Korea
println(countryCode.get("US"))  // United States
println(countryCode["CN"])  // null

// 기본값과 함께
println(countryCode.getOrDefault("CN", "Unknown"))  // Unknown
```

**가변 Map**:
```kotlin
val mutableMap = mutableMapOf(
    "apple" to 1000,
    "banana" to 500
)

mutableMap["orange"] = 800
mutableMap.put("grape", 1500)
mutableMap.remove("banana")

println(mutableMap)
// {apple=1000, orange=800, grape=1500}
```

---

### Java와 비교

**Java**:
```java
// Java - 불변 리스트 만들기 복잡
List<Integer> immutable = Collections.unmodifiableList(
    Arrays.asList(1, 2, 3)
);

List<Integer> mutable = new ArrayList<>(Arrays.asList(1, 2, 3));
mutable.add(4);

Map<String, Integer> map = new HashMap<>();
map.put("apple", 1000);
map.put("banana", 500);

Set<Integer> set = new HashSet<>(Arrays.asList(1, 2, 3, 3));
```

**Kotlin**:
```kotlin
val immutable = listOf(1, 2, 3)
val mutable = mutableListOf(1, 2, 3)
mutable.add(4)

val map = mutableMapOf(
    "apple" to 1000,
    "banana" to 500
)

val set = setOf(1, 2, 3, 3)
```

---

## 7.2 불변 컬렉션 vs 가변 컬렉션

### 인터페이스 계층 구조

```
Collection (읽기 전용)
    ├── List
    ├── Set
    └── Map (별도)

MutableCollection (읽기/쓰기)
    ├── MutableList
    ├── MutableSet
    └── MutableMap (별도)
```

---

### 불변 컬렉션의 이점

```kotlin
// ✅ 불변 - 안전함
fun processNumbers(numbers: List<Int>) {
    // numbers는 변경될 수 없으므로 안전
    val sum = numbers.sum()
    // numbers.add(10)  // 컴파일 에러!
}

// ❌ 가변 - 위험할 수 있음
fun processNumbers(numbers: MutableList<Int>) {
    val sum = numbers.sum()
    numbers.add(10)  // 예상치 못한 부작용!
}
```

---

### 변환

```kotlin
// 불변 → 가변
val immutableList = listOf(1, 2, 3)
val mutableList = immutableList.toMutableList()
mutableList.add(4)

// 가변 → 불변
val mutable = mutableListOf(1, 2, 3)
val immutable: List<Int> = mutable.toList()
```

---

## 7.3 컬렉션 생성과 초기화

### 다양한 생성 방법

**List**:
```kotlin
// 빈 리스트
val empty = emptyList<Int>()
val empty2 = listOf<Int>()

// 요소로 생성
val numbers = listOf(1, 2, 3, 4, 5)

// 크기와 초기화 함수로 생성
val squares = List(5) { i -> i * i }
println(squares)  // [0, 1, 4, 9, 16]

// 같은 값으로 채우기
val zeros = List(5) { 0 }
println(zeros)  // [0, 0, 0, 0, 0]
```

**Set**:
```kotlin
val emptySet = emptySet<String>()
val fruits = setOf("apple", "banana", "orange")
```

**Map**:
```kotlin
val emptyMap = emptyMap<String, Int>()

val scores = mapOf(
    "Alice" to 95,
    "Bob" to 88,
    "Charlie" to 92
)

// Pair를 사용한 생성
val scores2 = mapOf(
    Pair("Alice", 95),
    Pair("Bob", 88)
)
```

---

### buildList, buildSet, buildMap

Kotlin 1.6+에서 추가된 빌더 함수:

```kotlin
val list = buildList {
    add(1)
    add(2)
    addAll(listOf(3, 4, 5))
}
println(list)  // [1, 2, 3, 4, 5]

val set = buildSet {
    add("apple")
    add("banana")
    add("apple")  // 중복은 무시됨
}
println(set)  // [apple, banana]

val map = buildMap {
    put("A", 1)
    put("B", 2)
    this["C"] = 3
}
println(map)  // {A=1, B=2, C=3}
```

---

## 7.4 컬렉션 연산 (filter, map, reduce 등)

### filter - 조건에 맞는 요소만

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 짝수만
val evenNumbers = numbers.filter { it % 2 == 0 }
println(evenNumbers)  // [2, 4, 6, 8, 10]

// 5보다 큰 수
val greaterThanFive = numbers.filter { it > 5 }
println(greaterThanFive)  // [6, 7, 8, 9, 10]
```

**filterNot**:
```kotlin
val oddNumbers = numbers.filterNot { it % 2 == 0 }
println(oddNumbers)  // [1, 3, 5, 7, 9]
```

**filterNotNull**:
```kotlin
val mixed: List<Int?> = listOf(1, null, 2, null, 3)
val nonNull = mixed.filterNotNull()
println(nonNull)  // [1, 2, 3]
```

---

### map - 각 요소를 변환

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// 각 숫자를 제곱
val squares = numbers.map { it * it }
println(squares)  // [1, 4, 9, 16, 25]

// 문자열로 변환
val strings = numbers.map { "Number: $it" }
println(strings)
// [Number: 1, Number: 2, Number: 3, Number: 4, Number: 5]
```

**mapNotNull**:
```kotlin
val strings = listOf("1", "2", "abc", "3", "def")
val numbers = strings.mapNotNull { it.toIntOrNull() }
println(numbers)  // [1, 2, 3]
```

**mapIndexed**:
```kotlin
val fruits = listOf("apple", "banana", "orange")
val indexed = fruits.mapIndexed { index, fruit ->
    "$index: $fruit"
}
println(indexed)
// [0: apple, 1: banana, 2: orange]
```

---

### flatMap - 중첩 컬렉션 평탄화

```kotlin
val lists = listOf(
    listOf(1, 2, 3),
    listOf(4, 5),
    listOf(6, 7, 8, 9)
)

val flattened = lists.flatMap { it }
println(flattened)  // [1, 2, 3, 4, 5, 6, 7, 8, 9]

// 또는 flatten()
val flattened2 = lists.flatten()
println(flattened2)  // [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

**실전 예제**:
```kotlin
data class Person(val name: String, val hobbies: List<String>)

val people = listOf(
    Person("Alice", listOf("Reading", "Gaming")),
    Person("Bob", listOf("Gaming", "Cooking")),
    Person("Charlie", listOf("Reading", "Sports"))
)

// 모든 취미를 하나의 리스트로
val allHobbies = people.flatMap { it.hobbies }
println(allHobbies)
// [Reading, Gaming, Gaming, Cooking, Reading, Sports]

// 중복 제거
val uniqueHobbies = people.flatMap { it.hobbies }.toSet()
println(uniqueHobbies)
// [Reading, Gaming, Cooking, Sports]
```

---

### groupBy - 그룹화

```kotlin
val words = listOf("apple", "banana", "avocado", "blueberry", "apricot")

// 첫 글자로 그룹화
val grouped = words.groupBy { it.first() }
println(grouped)
// {a=[apple, avocado, apricot], b=[banana, blueberry]}

// 길이로 그룹화
val byLength = words.groupBy { it.length }
println(byLength)
// {5=[apple], 6=[banana], 7=[avocado], 9=[blueberry], 7=[apricot]}
```

---

### partition - 두 그룹으로 분리

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

val (even, odd) = numbers.partition { it % 2 == 0 }
println("짝수: $even")  // 짝수: [2, 4, 6, 8, 10]
println("홀수: $odd")    // 홀수: [1, 3, 5, 7, 9]
```

---

### reduce와 fold - 집계

**reduce**:
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val sum = numbers.reduce { acc, n -> acc + n }
println(sum)  // 15

val product = numbers.reduce { acc, n -> acc * n }
println(product)  // 120
```

**fold** (초기값 지정):
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val sum = numbers.fold(0) { acc, n -> acc + n }
println(sum)  // 15

val sumWith100 = numbers.fold(100) { acc, n -> acc + n }
println(sumWith100)  // 115
```

**실전 예제**:
```kotlin
data class Order(val amount: Int)

val orders = listOf(
    Order(1000),
    Order(2000),
    Order(1500)
)

val totalAmount = orders.fold(0) { total, order ->
    total + order.amount
}
println("총액: $totalAmount")  // 총액: 4500
```

---

### 기타 유용한 연산

**take / drop**:
```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

println(numbers.take(3))      // [1, 2, 3]
println(numbers.takeLast(3))  // [8, 9, 10]
println(numbers.drop(3))      // [4, 5, 6, 7, 8, 9, 10]
println(numbers.dropLast(3))  // [1, 2, 3, 4, 5, 6, 7]
```

**takeWhile / dropWhile**:
```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 4, 3, 2, 1)

println(numbers.takeWhile { it < 5 })  // [1, 2, 3, 4]
println(numbers.dropWhile { it < 5 })  // [5, 4, 3, 2, 1]
```

**distinct**:
```kotlin
val numbers = listOf(1, 2, 2, 3, 3, 3, 4, 4, 4, 4)
println(numbers.distinct())  // [1, 2, 3, 4]

data class Person(val name: String, val age: Int)
val people = listOf(
    Person("Alice", 30),
    Person("Bob", 25),
    Person("Alice", 35)
)
println(people.distinctBy { it.name })
// [Person(name=Alice, age=30), Person(name=Bob, age=25)]
```

**sorted**:
```kotlin
val numbers = listOf(5, 2, 8, 1, 9, 3)

println(numbers.sorted())           // [1, 2, 3, 5, 8, 9]
println(numbers.sortedDescending()) // [9, 8, 5, 3, 2, 1]

val words = listOf("banana", "apple", "cherry")
println(words.sorted())  // [apple, banana, cherry]

data class Person(val name: String, val age: Int)
val people = listOf(
    Person("Charlie", 30),
    Person("Alice", 25),
    Person("Bob", 35)
)

println(people.sortedBy { it.age })
// [Person(name=Alice, age=25), Person(name=Charlie, age=30), Person(name=Bob, age=35)]

println(people.sortedByDescending { it.name })
// [Person(name=Charlie, age=30), Person(name=Bob, age=35), Person(name=Alice, age=25)]
```

**zip**:
```kotlin
val names = listOf("Alice", "Bob", "Charlie")
val ages = listOf(25, 30, 35)

val people = names.zip(ages)
println(people)
// [(Alice, 25), (Bob, 30), (Charlie, 35)]

val peopleWithNames = names.zip(ages) { name, age ->
    "$name is $age years old"
}
println(peopleWithNames)
// [Alice is 25 years old, Bob is 30 years old, Charlie is 35 years old]
```

**associate**:
```kotlin
val names = listOf("Alice", "Bob", "Charlie")

val nameToLength = names.associateWith { it.length }
println(nameToLength)
// {Alice=5, Bob=3, Charlie=7}

val lengthToName = names.associateBy { it.length }
println(lengthToName)
// {5=Alice, 3=Bob, 7=Charlie}
```

---

### Java Stream API와 비교

**Java**:
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

List<Integer> result = numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .collect(Collectors.toList());

int sum = numbers.stream()
    .filter(n -> n % 2 == 0)
    .mapToInt(Integer::intValue)
    .sum();
```

**Kotlin**:
```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

val result = numbers
    .filter { it % 2 == 0 }
    .map { it * it }

val sum = numbers
    .filter { it % 2 == 0 }
    .sum()
```

**차이점**:
- ✅ Stream 생성 불필요
- ✅ collect() 불필요
- ✅ 더 간결한 람다 문법

---

## 7.5 Sequence를 통한 지연 평가

### Collection vs Sequence

**Collection** (즉시 평가):
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val result = numbers
    .map {
        println("map: $it")
        it * 2
    }
    .filter {
        println("filter: $it")
        it > 5
    }

println(result)

// 출력:
// map: 1
// map: 2
// map: 3
// map: 4
// map: 5
// filter: 2
// filter: 4
// filter: 6
// filter: 8
// filter: 10
// [6, 8, 10]
```

**Sequence** (지연 평가):
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val result = numbers.asSequence()
    .map {
        println("map: $it")
        it * 2
    }
    .filter {
        println("filter: $it")
        it > 5
    }
    .toList()

println(result)

// 출력:
// map: 1
// filter: 2
// map: 2
// filter: 4
// map: 3
// filter: 6
// map: 4
// filter: 8
// map: 5
// filter: 10
// [6, 8, 10]
```

---

### 언제 Sequence를 사용하나?

**Collection 사용** (대부분의 경우):
```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
val result = numbers.filter { it % 2 == 0 }.map { it * it }
```

**Sequence 사용** (큰 데이터, 많은 연산):
```kotlin
val largeList = (1..1_000_000).toList()

val result = largeList.asSequence()
    .filter { it % 2 == 0 }
    .map { it * it }
    .take(10)  // 처음 10개만 필요
    .toList()
```

---

### generateSequence - 무한 시퀀스

```kotlin
// 1부터 시작해서 계속 증가
val naturalNumbers = generateSequence(1) { it + 1 }

val first10 = naturalNumbers.take(10).toList()
println(first10)  // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

// 피보나치 수열
val fibonacci = generateSequence(Pair(0, 1)) { (a, b) ->
    Pair(b, a + b)
}.map { it.first }

println(fibonacci.take(10).toList())
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

## 7.6 Java와 비교: 컬렉션 처리

### 종합 예제

**시나리오**: 사용자 목록에서 성인만 필터링하고 이름을 대문자로 변환

**Java**:
```java
List<User> users = getUsers();

List<String> result = users.stream()
    .filter(user -> user.getAge() >= 18)
    .map(user -> user.getName().toUpperCase())
    .collect(Collectors.toList());

// 그룹화
Map<Integer, List<User>> byAge = users.stream()
    .collect(Collectors.groupingBy(User::getAge));

// 합계
int totalAge = users.stream()
    .mapToInt(User::getAge)
    .sum();
```

**Kotlin**:
```kotlin
val users = getUsers()

val result = users
    .filter { it.age >= 18 }
    .map { it.name.uppercase() }

// 그룹화
val byAge = users.groupBy { it.age }

// 합계
val totalAge = users.sumOf { it.age }
```

---

## 실전 팁

### 💡 Tip 1: 불변 컬렉션을 기본으로

```kotlin
// ✅ 권장
fun getUsers(): List<User> {
    return listOf(user1, user2, user3)
}

// ❌ 피하기
fun getUsers(): MutableList<User> {
    return mutableListOf(user1, user2, user3)
}
```

---

### 💡 Tip 2: 체이닝 활용

```kotlin
val users = getUsers()

val result = users
    .filter { it.isActive }
    .filter { it.age >= 18 }
    .map { it.name }
    .sorted()
    .take(10)
```

---

### 💡 Tip 3: sumOf, count 등 특화 함수 활용

```kotlin
data class Order(val amount: Int, val isPaid: Boolean)

val orders = getOrders()

// ✅ 좋은 방법
val totalAmount = orders.sumOf { it.amount }
val paidCount = orders.count { it.isPaid }

// ❌ 비효율적
val totalAmount2 = orders.map { it.amount }.sum()
val paidCount2 = orders.filter { it.isPaid }.size
```

---

### 💡 Tip 4: 큰 데이터는 Sequence 고려

```kotlin
val largeData = getLargeDataset()

// ✅ 지연 평가로 효율적
val result = largeData.asSequence()
    .filter { heavyComputation(it) }
    .map { transform(it) }
    .take(100)
    .toList()
```

---

## 연습 문제

### 문제 1: 짝수의 제곱
1부터 20까지의 숫자 중 짝수만 골라서 제곱한 리스트 만들기

<details>
<summary>정답 보기</summary>

```kotlin
fun main() {
    val result = (1..20)
        .filter { it % 2 == 0 }
        .map { it * it }

    println(result)
    // [4, 16, 36, 64, 100, 144, 196, 256, 324, 400]
}
```
</details>

### 문제 2: 학생 성적 처리
학생 목록에서 평균 80점 이상인 학생의 이름만 추출

<details>
<summary>정답 보기</summary>

```kotlin
data class Student(val name: String, val scores: List<Int>)

fun main() {
    val students = listOf(
        Student("Alice", listOf(90, 85, 88)),
        Student("Bob", listOf(70, 75, 72)),
        Student("Charlie", listOf(95, 92, 98))
    )

    val topStudents = students
        .filter { it.scores.average() >= 80 }
        .map { it.name }

    println(topStudents)  // [Alice, Charlie]
}
```
</details>

### 문제 3: 단어 빈도수
문자열 리스트에서 각 단어의 등장 횟수 계산

<details>
<summary>정답 보기</summary>

```kotlin
fun main() {
    val words = listOf("apple", "banana", "apple", "cherry", "banana", "apple")

    val frequency = words.groupingBy { it }.eachCount()
    println(frequency)
    // {apple=3, banana=2, cherry=1}

    // 또는
    val frequency2 = words.groupBy { it }.mapValues { it.value.size }
    println(frequency2)
}
```
</details>

### 문제 4: 중첩 리스트 평탄화
여러 부서의 직원 목록을 하나의 리스트로 만들고, 이름순 정렬

<details>
<summary>정답 보기</summary>

```kotlin
data class Employee(val name: String, val department: String)
data class Department(val name: String, val employees: List<Employee>)

fun main() {
    val departments = listOf(
        Department("IT", listOf(
            Employee("Alice", "IT"),
            Employee("Charlie", "IT")
        )),
        Department("HR", listOf(
            Employee("Bob", "HR"),
            Employee("David", "HR")
        ))
    )

    val allEmployees = departments
        .flatMap { it.employees }
        .sortedBy { it.name }

    println(allEmployees)
    // [Employee(name=Alice, department=IT), Employee(name=Bob, department=HR),
    //  Employee(name=Charlie, department=IT), Employee(name=David, department=HR)]
}
```
</details>

---

## 핵심 요약

### 꼭 기억할 것

1. **불변 vs 가변**
   - `listOf`, `setOf`, `mapOf`: 불변 (권장)
   - `mutableListOf`, `mutableSetOf`, `mutableMapOf`: 가변

2. **주요 연산**
   - `filter`: 조건 필터링
   - `map`: 변환
   - `flatMap`: 평탄화
   - `groupBy`: 그룹화
   - `reduce`/`fold`: 집계

3. **Collection vs Sequence**
   - Collection: 즉시 평가 (기본)
   - Sequence: 지연 평가 (대용량 데이터)

4. **Java 대비 장점**
   - Stream 생성 불필요
   - collect() 불필요
   - 더 간결한 문법

---

## 다음 챕터 예고

Chapter 8에서는 **람다와 고차 함수**를 다룹니다:
- 람다 표현식
- 고차 함수
- 인라인 함수
- 수신 객체 지정 람다

함수형 프로그래밍의 핵심을 배웁니다!

---

[← 이전: Chapter 6. Null 안전성](chapter6-null-safety.md) | [다음: Chapter 8. 람다와 고차 함수 →](chapter8-lambda-higher-order.md)