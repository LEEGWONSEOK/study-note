# Chapter 4. 컬렉션

> 📦 **비유**: 컬렉션은 **수납 도구**입니다.
> - **List**: 순서가 있는 서랍장 (변경 가능)
> - **Tuple**: 순서가 있는 봉인된 상자 (변경 불가)
> - **Dict**: 라벨이 붙은 수납함 (key-value)
> - **Set**: 중복 없는 주머니

## 4.1 리스트 (List)

> 🗂️ **비유**: 리스트는 **번호표가 붙은 서랍장**입니다.
> - 순서가 있음 (0번부터 시작)
> - 중복 허용
> - 변경 가능 (추가, 삭제, 수정)

### 리스트 생성

```python
# 빈 리스트
empty = []
empty2 = list()

# 값이 있는 리스트
numbers = [1, 2, 3, 4, 5]
names = ["Alice", "Bob", "Charlie"]
mixed = [1, "two", 3.0, True]  # 타입 혼합 가능!

# 리스트 컴프리헨션 (나중에 배움)
squares = [x**2 for x in range(1, 6)]  # [1, 4, 9, 16, 25]
```

**Java와 비교**:
```java
// Java - 타입 명시 필요
List<Integer> numbers = new ArrayList<>();
numbers.add(1);
numbers.add(2);

// Python - 타입 자유로움
numbers = [1, 2, 3]
mixed = [1, "two", 3.0]  # 가능!
```

---

### 리스트 접근

> 🎯 **비유**: 리스트 인덱스는 **아파트 호수**입니다.
> - 양수: 101호, 102호, ... (앞에서부터)
> - 음수: -101호, -102호, ... (뒤에서부터)

```python
fruits = ["사과", "바나나", "체리", "딸기", "포도"]
#         0       1        2       3       4     (앞에서)
#        -5      -4       -3      -2      -1     (뒤에서)

# 인덱싱
print(fruits[0])   # 사과 (첫 번째)
print(fruits[-1])  # 포도 (마지막)
print(fruits[-2])  # 딸기 (뒤에서 두 번째)

# 슬라이싱 [시작:끝:간격]
print(fruits[1:3])    # ['바나나', '체리']
print(fruits[:3])     # ['사과', '바나나', '체리'] (처음부터 3개)
print(fruits[2:])     # ['체리', '딸기', '포도'] (2번째부터 끝까지)
print(fruits[::2])    # ['사과', '체리', '포도'] (2칸씩)
print(fruits[::-1])   # 역순!
```

**실생활 예시**:
```python
# 최근 5개 로그만 가져오기
logs = ["log1", "log2", "log3", "log4", "log5", "log6", "log7"]
recent_logs = logs[-5:]  # 뒤에서 5개
print(recent_logs)  # ['log3', 'log4', 'log5', 'log6', 'log7']

# 홀수번째만 가져오기
items = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
odd_items = items[1::2]  # 1번부터 2칸씩
print(odd_items)  # [1, 3, 5, 7, 9]
```

---

### 리스트 수정

```python
fruits = ["사과", "바나나", "체리"]

# 요소 변경
fruits[1] = "망고"
print(fruits)  # ['사과', '망고', '체리']

# 추가
fruits.append("포도")       # 끝에 추가
fruits.insert(1, "딸기")    # 1번 위치에 추가

# 삭제
fruits.remove("망고")       # 값으로 삭제
last = fruits.pop()         # 마지막 제거 및 반환
del fruits[0]               # 인덱스로 삭제

# 확장
fruits.extend(["키위", "메론"])  # 여러 개 추가
fruits += ["레몬"]               # 같은 의미

# 정렬
numbers = [3, 1, 4, 1, 5, 9, 2]
numbers.sort()              # 원본 정렬
print(numbers)              # [1, 1, 2, 3, 4, 5, 9]

sorted_nums = sorted(numbers, reverse=True)  # 새 리스트 반환
```

**실생활 예시 - 장바구니**:
```python
# 장바구니 시뮬레이션
cart = []

# 상품 추가
cart.append("노트북")
cart.append("마우스")
cart.append("키보드")

print(f"장바구니: {cart}")

# 상품 제거
cart.remove("마우스")

# 상품 개수
print(f"상품 개수: {len(cart)}")

# 특정 상품 있는지 확인
if "노트북" in cart:
    print("노트북이 장바구니에 있습니다")
```

---

### 리스트 메서드

```python
numbers = [1, 2, 3, 2, 4, 2, 5]

# 개수 세기
print(numbers.count(2))  # 3 (2가 3개)

# 인덱스 찾기
print(numbers.index(3))  # 2 (3의 위치)

# 뒤집기
numbers.reverse()
print(numbers)  # [5, 2, 4, 2, 3, 2, 1]

# 복사
copy1 = numbers.copy()   # 얕은 복사
copy2 = numbers[:]       # 슬라이싱 복사
copy3 = list(numbers)    # list() 복사

# 비우기
numbers.clear()
print(numbers)  # []
```

---

## 4.2 튜플 (Tuple)

> 🔒 **비유**: 튜플은 **봉인된 상자**입니다.
> - 한번 만들면 변경 불가 (Immutable)
> - 리스트보다 빠름
> - 안전함 (실수로 수정 못함)

### 튜플 생성

```python
# 빈 튜플
empty = ()
empty2 = tuple()

# 값이 있는 튜플
point = (3, 5)
rgb = (255, 128, 0)
info = ("Alice", 30, "alice@example.com")

# 요소가 1개인 튜플 (쉼표 필수!)
single = (42,)   # 튜플
not_tuple = (42) # 그냥 정수

# 괄호 생략 가능
coords = 10, 20  # (10, 20)
```

**언제 사용할까?**
```python
# 1. 좌표, 색상 등 변경하면 안 되는 값
SCREEN_SIZE = (1920, 1080)  # 변경 불가

# 2. 함수에서 여러 값 반환
def get_user():
    return ("Alice", 30, "alice@example.com")

name, age, email = get_user()  # 언패킹

# 3. Dictionary의 키 (리스트는 안 됨!)
locations = {
    (0, 0): "원점",
    (10, 20): "A지점"
}
```

---

### 튜플 언패킹

> 🎁 **비유**: 튜플 언패킹은 **선물 포장 풀기**입니다.

```python
# 기본 언패킹
point = (3, 5)
x, y = point
print(f"x={x}, y={y}")  # x=3, y=5

# 여러 개 언패킹
person = ("Alice", 30, "alice@example.com")
name, age, email = person

# 나머지는 리스트로
numbers = (1, 2, 3, 4, 5)
first, *rest = numbers
print(first)  # 1
print(rest)   # [2, 3, 4, 5]

# 중간에서도 가능
first, *middle, last = numbers
print(middle)  # [2, 3, 4]

# 값 교환 (임시 변수 불필요!)
a, b = 10, 20
a, b = b, a  # swap!
print(a, b)  # 20 10
```

**실생활 예시**:
```python
# 함수에서 여러 값 반환
def calculate_stats(numbers):
    return (
        min(numbers),
        max(numbers),
        sum(numbers) / len(numbers)
    )

scores = [85, 90, 78, 92, 88]
min_score, max_score, avg_score = calculate_stats(scores)

print(f"최소: {min_score}, 최대: {max_score}, 평균: {avg_score:.1f}")
```

---

## 4.3 딕셔너리 (Dictionary)

> 🏷️ **비유**: 딕셔너리는 **라벨이 붙은 수납함**입니다.
> - Key: 라벨 (고유해야 함)
> - Value: 내용물 (중복 가능)
> - 순서 유지 (Python 3.7+)

### 딕셔너리 생성

```python
# 빈 딕셔너리
empty = {}
empty2 = dict()

# 값이 있는 딕셔너리
user = {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com"
}

# dict() 생성자
user2 = dict(name="Bob", age=25)

# 리스트의 튜플로부터
pairs = [("a", 1), ("b", 2)]
d = dict(pairs)  # {'a': 1, 'b': 2}
```

**Java와 비교**:
```java
// Java
Map<String, Object> user = new HashMap<>();
user.put("name", "Alice");
user.put("age", 30);

// Python
user = {
    "name": "Alice",
    "age": 30
}
```

---

### 딕셔너리 접근

```python
user = {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com"
}

# 값 가져오기
print(user["name"])  # Alice

# ❌ 없는 키 접근 (KeyError 발생)
# print(user["phone"])  # 오류!

# ✅ get() 사용 (안전)
print(user.get("phone"))  # None
print(user.get("phone", "없음"))  # 기본값 지정

# 값 변경
user["age"] = 31

# 값 추가
user["phone"] = "010-1234-5678"

# 값 삭제
del user["email"]
removed = user.pop("phone", "없음")  # 제거 및 반환
```

**실생활 예시 - 학생 성적부**:
```python
# 학생 정보 관리
students = {
    "Alice": {"age": 20, "grade": "A", "score": 95},
    "Bob": {"age": 21, "grade": "B", "score": 85},
    "Charlie": {"age": 20, "grade": "A", "score": 92}
}

# 특정 학생 정보
print(students["Alice"]["score"])  # 95

# 학생 추가
students["David"] = {"age": 22, "grade": "C", "score": 75}

# A 학점 학생만 추출
a_students = {
    name: info
    for name, info in students.items()
    if info["grade"] == "A"
}
print(a_students)
```

---

### 딕셔너리 메서드

```python
user = {"name": "Alice", "age": 30, "city": "Seoul"}

# 키 목록
print(user.keys())  # dict_keys(['name', 'age', 'city'])

# 값 목록
print(user.values())  # dict_values(['Alice', 30, 'Seoul'])

# (키, 값) 쌍
print(user.items())  # dict_items([('name', 'Alice'), ('age', 30), ...])

# 반복
for key in user:
    print(f"{key}: {user[key]}")

# 더 좋은 방법
for key, value in user.items():
    print(f"{key}: {value}")

# 키 존재 확인
if "name" in user:
    print("이름이 있습니다")

# 합치기
default = {"country": "Korea", "lang": "Korean"}
user.update(default)
print(user)

# 복사
copy = user.copy()
```

**실생활 예시 - 단어 빈도수**:
```python
text = "apple banana apple cherry banana apple"
words = text.split()

# 단어 개수 세기
word_count = {}
for word in words:
    word_count[word] = word_count.get(word, 0) + 1

print(word_count)
# {'apple': 3, 'banana': 2, 'cherry': 1}

# 또는 Counter 사용
from collections import Counter
word_count = Counter(words)
print(word_count.most_common(2))  # 가장 많은 2개
# [('apple', 3), ('banana', 2)]
```

---

## 4.4 셋 (Set)

> 🎲 **비유**: 셋은 **주머니**입니다.
> - 중복 없음 (같은 구슬 여러 개 안 됨)
> - 순서 없음 (섞여있음)
> - 빠른 검색 (집합 연산)

### 셋 생성

```python
# 빈 셋 (주의: {}는 딕셔너리!)
empty = set()

# 값이 있는 셋
numbers = {1, 2, 3, 4, 5}
fruits = {"apple", "banana", "cherry"}

# 중복 자동 제거!
duplicates = {1, 2, 2, 3, 3, 3}
print(duplicates)  # {1, 2, 3}

# 리스트에서 셋 생성
lst = [1, 2, 2, 3, 3, 3]
unique = set(lst)  # {1, 2, 3}
```

---

### 셋 연산

> 🧮 **비유**: 셋은 **수학의 집합**입니다.

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# 합집합 (Union)
print(a | b)  # {1, 2, 3, 4, 5, 6, 7, 8}
print(a.union(b))

# 교집합 (Intersection)
print(a & b)  # {4, 5}
print(a.intersection(b))

# 차집합 (Difference)
print(a - b)  # {1, 2, 3}
print(a.difference(b))

# 대칭 차집합 (XOR)
print(a ^ b)  # {1, 2, 3, 6, 7, 8}
print(a.symmetric_difference(b))

# 부분집합
c = {1, 2}
print(c < a)   # True (c는 a의 부분집합)
print(c.issubset(a))
```

**실생활 예시 - 태그 시스템**:
```python
# 게시글 태그
post1_tags = {"python", "programming", "tutorial"}
post2_tags = {"python", "web", "django"}
post3_tags = {"java", "programming", "oop"}

# 공통 태그 (교집합)
common = post1_tags & post2_tags
print(f"공통 태그: {common}")  # {'python'}

# 모든 태그 (합집합)
all_tags = post1_tags | post2_tags | post3_tags
print(f"모든 태그: {all_tags}")

# Python 태그가 있는지
if "python" in post1_tags:
    print("Python 게시글입니다")
```

---

### 셋 메서드

```python
fruits = {"apple", "banana"}

# 추가
fruits.add("cherry")

# 여러 개 추가
fruits.update(["mango", "grape"])

# 제거
fruits.remove("banana")  # 없으면 KeyError
fruits.discard("banana") # 없어도 오류 안 남 (안전)

# 임의 제거
item = fruits.pop()

# 비우기
fruits.clear()
```

**실생활 예시 - 중복 제거**:
```python
# 이메일 중복 제거
emails = [
    "alice@example.com",
    "bob@example.com",
    "alice@example.com",  # 중복!
    "charlie@example.com"
]

# 중복 제거
unique_emails = list(set(emails))
print(unique_emails)

# 또는 순서 유지하면서 중복 제거
from collections import OrderedDict
unique_emails = list(OrderedDict.fromkeys(emails))
```

---

## 4.5 컬렉션 선택 가이드

| 필요한 기능 | 선택 |
|------------|------|
| 순서 유지 + 변경 가능 | **List** |
| 순서 유지 + 변경 불가 | **Tuple** |
| Key-Value 매핑 | **Dict** |
| 중복 제거 + 빠른 검색 | **Set** |

```python
# ✅ 언제 뭘 쓸까?

# 장바구니 → List (순서 있고, 같은 상품 여러 개 가능)
cart = ["apple", "banana", "apple"]

# 좌표 → Tuple (변경하면 안 됨)
point = (10, 20)

# 사용자 정보 → Dict (이름으로 찾기)
user = {"name": "Alice", "age": 30}

# 고유한 방문자 → Set (중복 제거)
visitors = {"alice", "bob", "alice"}  # {"alice", "bob"}
```

---

## 실전 팁

### 💡 Tip 1: in 연산자 성능

```python
# List: O(n) - 느림
items_list = [1, 2, 3, ..., 1000]
print(500 in items_list)  # 처음부터 끝까지 검색

# Set: O(1) - 빠름!
items_set = {1, 2, 3, ..., 1000}
print(500 in items_set)  # 즉시 찾음

# 교훈: 검색이 많으면 Set 사용!
```

---

### 💡 Tip 2: 딕셔너리 기본값

```python
# ❌ 긴 방법
if key in d:
    d[key] += 1
else:
    d[key] = 1

# ✅ get() 사용
d[key] = d.get(key, 0) + 1

# ✅ setdefault() 사용
d.setdefault(key, 0)
d[key] += 1

# ✅ defaultdict 사용 (최고!)
from collections import defaultdict
d = defaultdict(int)  # 기본값 0
d[key] += 1  # 간결!
```

---

## 연습 문제

### 문제 1: 리스트 중복 제거 (순서 유지)

<details>
<summary>정답 보기</summary>

```python
def remove_duplicates(lst):
    seen = set()
    result = []
    for item in lst:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

numbers = [1, 2, 2, 3, 1, 4, 3, 5]
print(remove_duplicates(numbers))  # [1, 2, 3, 4, 5]
```
</details>

---

### 문제 2: 두 리스트의 공통 요소

<details>
<summary>정답 보기</summary>

```python
list1 = [1, 2, 3, 4, 5]
list2 = [4, 5, 6, 7, 8]

# 방법 1: Set 사용
common = list(set(list1) & set(list2))

# 방법 2: 리스트 컴프리헨션
common = [x for x in list1 if x in list2]

print(common)  # [4, 5]
```
</details>

---

## 핵심 요약

1. **List**: 순서O, 중복O, 변경O → `[1, 2, 3]`
2. **Tuple**: 순서O, 중복O, 변경X → `(1, 2, 3)`
3. **Dict**: Key-Value, 변경O → `{"a": 1}`
4. **Set**: 순서X, 중복X, 변경O → `{1, 2, 3}`

---

[← 이전](chapter3-data-types.md) | [다음: Chapter 5 →](chapter5-control-flow.md)