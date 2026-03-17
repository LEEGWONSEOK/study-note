# 부록 A. 유용한 표준 라이브러리

> Python의 강력함은 "배터리 포함(Batteries Included)" 철학에서 나옵니다. 다양한 표준 라이브러리가 이미 내장되어 있어 대부분의 작업을 쉽게 처리할 수 있습니다.

---

## 1. collections - 특수 컨테이너

### 1.1 Counter - 개수 세기

```python
from collections import Counter

# 투표 집계
votes = ['Alice', 'Bob', 'Alice', 'Charlie', 'Bob', 'Alice']
counter = Counter(votes)

print(counter)  # Counter({'Alice': 3, 'Bob': 2, 'Charlie': 1})
print(counter.most_common(2))  # [('Alice', 3), ('Bob', 2)]

# 단어 빈도수
text = "hello world hello python world world"
word_count = Counter(text.split())
print(word_count)  # Counter({'world': 3, 'hello': 2, 'python': 1})
```

### 1.2 defaultdict - 기본값이 있는 딕셔너리

```python
from collections import defaultdict

# 그룹화
students = [
    ('Alice', 'Math'),
    ('Bob', 'English'),
    ('Charlie', 'Math'),
    ('David', 'English')
]

groups = defaultdict(list)
for name, subject in students:
    groups[subject].append(name)

print(dict(groups))
# {'Math': ['Alice', 'Charlie'], 'English': ['Bob', 'David']}
```

### 1.3 deque - 양방향 큐

```python
from collections import deque

# 최근 검색 기록 (최대 5개)
recent_searches = deque(maxlen=5)

recent_searches.append('Python')
recent_searches.append('Java')
recent_searches.append('JavaScript')
recent_searches.append('Go')
recent_searches.append('Rust')
recent_searches.append('TypeScript')  # Python이 제거됨

print(list(recent_searches))
# ['Java', 'JavaScript', 'Go', 'Rust', 'TypeScript']
```

### 1.4 namedtuple - 이름 있는 튜플

```python
from collections import namedtuple

# 좌표
Point = namedtuple('Point', ['x', 'y'])
p = Point(10, 20)
print(f"x={p.x}, y={p.y}")

# 사용자
User = namedtuple('User', ['id', 'name', 'email'])
user = User(1, 'Alice', 'alice@example.com')
print(user.name)  # Alice
```

---

## 2. itertools - 반복자 도구

### 2.1 무한 반복자

```python
import itertools

# count - 무한 카운터
for i in itertools.count(10, 2):
    print(i, end=' ')
    if i >= 20:
        break
# 10 12 14 16 18 20

# cycle - 무한 반복
counter = 0
for item in itertools.cycle(['A', 'B', 'C']):
    print(item, end=' ')
    counter += 1
    if counter >= 7:
        break
# A B C A B C A

# repeat - n번 반복
print(list(itertools.repeat('Hello', 3)))
# ['Hello', 'Hello', 'Hello']
```

### 2.2 조합 반복자

```python
import itertools

# 순열 (permutations)
print(list(itertools.permutations([1, 2, 3], 2)))
# [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

# 조합 (combinations)
print(list(itertools.combinations([1, 2, 3], 2)))
# [(1, 2), (1, 3), (2, 3)]

# 중복 조합
print(list(itertools.combinations_with_replacement([1, 2], 2)))
# [(1, 1), (1, 2), (2, 2)]

# 데카르트 곱
print(list(itertools.product([1, 2], ['A', 'B'])))
# [(1, 'A'), (1, 'B'), (2, 'A'), (2, 'B')]
```

### 2.3 유틸리티 반복자

```python
import itertools

# chain - 여러 반복자 연결
print(list(itertools.chain([1, 2], [3, 4], [5, 6])))
# [1, 2, 3, 4, 5, 6]

# islice - 슬라이싱
print(list(itertools.islice(range(10), 2, 8, 2)))
# [2, 4, 6]

# groupby - 그룹화
data = [('A', 1), ('A', 2), ('B', 3), ('B', 4), ('C', 5)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(f"{key}: {list(group)}")
# A: [('A', 1), ('A', 2)]
# B: [('B', 3), ('B', 4)]
# C: [('C', 5)]
```

---

## 3. functools - 함수형 프로그래밍

### 3.1 partial - 부분 함수

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

# 제곱 함수
square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(5))  # 25
print(cube(5))    # 125
```

### 3.2 reduce - 누적 계산

```python
from functools import reduce

# 곱셈
numbers = [1, 2, 3, 4, 5]
product = reduce(lambda x, y: x * y, numbers)
print(product)  # 120

# 최대값
max_value = reduce(lambda x, y: x if x > y else y, numbers)
print(max_value)  # 5
```

### 3.3 lru_cache - 캐싱

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(100))  # 빠르게 계산!
print(fibonacci.cache_info())  # 캐시 통계
```

---

## 4. datetime - 날짜와 시간

### 4.1 datetime 기본

```python
from datetime import datetime, date, time, timedelta

# 현재 시간
now = datetime.now()
print(now)  # 2024-01-15 10:30:45.123456

# 날짜 생성
birthday = date(1990, 5, 15)
print(birthday)  # 1990-05-15

# 시간 생성
meeting_time = time(14, 30, 0)
print(meeting_time)  # 14:30:00
```

### 4.2 날짜 계산

```python
from datetime import datetime, timedelta

now = datetime.now()

# 10일 후
future = now + timedelta(days=10)
print(future)

# 2주 전
past = now - timedelta(weeks=2)
print(past)

# 날짜 차이
diff = future - now
print(f"{diff.days}일 차이")
```

### 4.3 문자열 변환

```python
from datetime import datetime

now = datetime.now()

# datetime → 문자열
formatted = now.strftime("%Y년 %m월 %d일 %H:%M:%S")
print(formatted)  # 2024년 01월 15일 10:30:45

# 문자열 → datetime
date_str = "2024-01-15 10:30:45"
parsed = datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S")
print(parsed)
```

---

## 5. json - JSON 처리

### 5.1 JSON 직렬화/역직렬화

```python
import json

# Python → JSON
data = {
    'name': 'Alice',
    'age': 30,
    'hobbies': ['reading', 'coding'],
    'active': True
}

json_str = json.dumps(data, indent=2, ensure_ascii=False)
print(json_str)

# JSON → Python
parsed = json.loads(json_str)
print(parsed['name'])  # Alice
```

### 5.2 파일 읽기/쓰기

```python
import json

# 파일에 쓰기
data = {'name': 'Bob', 'age': 25}
with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, indent=2, ensure_ascii=False)

# 파일에서 읽기
with open('data.json', 'r', encoding='utf-8') as f:
    loaded = json.load(f)
    print(loaded)
```

---

## 6. pathlib - 경로 처리

### 6.1 Path 객체

```python
from pathlib import Path

# 경로 생성
p = Path('data/files/document.txt')

print(p.name)       # document.txt
print(p.stem)       # document
print(p.suffix)     # .txt
print(p.parent)     # data/files
print(p.absolute()) # /full/path/to/data/files/document.txt
```

### 6.2 파일/디렉토리 작업

```python
from pathlib import Path

# 디렉토리 생성
Path('data/temp').mkdir(parents=True, exist_ok=True)

# 파일 존재 확인
p = Path('data.txt')
if p.exists():
    print("파일 존재")

# 파일 읽기/쓰기
p.write_text("Hello, World!", encoding='utf-8')
content = p.read_text(encoding='utf-8')

# glob 패턴
for py_file in Path('.').glob('**/*.py'):
    print(py_file)
```

---

## 7. re - 정규표현식

### 7.1 기본 매칭

```python
import re

# 검색
text = "My email is alice@example.com"
match = re.search(r'[\w\.-]+@[\w\.-]+', text)
if match:
    print(f"이메일: {match.group()}")  # alice@example.com

# 모두 찾기
text = "Phone: 010-1234-5678, 010-9876-5432"
phones = re.findall(r'\d{3}-\d{4}-\d{4}', text)
print(phones)  # ['010-1234-5678', '010-9876-5432']
```

### 7.2 치환

```python
import re

# 공백 제거
text = "  Hello   World  "
cleaned = re.sub(r'\s+', ' ', text).strip()
print(cleaned)  # "Hello World"

# 이메일 마스킹
text = "Contact: alice@example.com"
masked = re.sub(r'([\w.-]+)@([\w.-]+)', r'\1@***', text)
print(masked)  # Contact: alice@***
```

### 7.3 그룹 캡처

```python
import re

# URL 파싱
url = "https://www.example.com/path/page.html"
pattern = r'(https?)://([^/]+)(/.+)?'
match = re.match(pattern, url)

if match:
    print(f"프로토콜: {match.group(1)}")  # https
    print(f"도메인: {match.group(2)}")    # www.example.com
    print(f"경로: {match.group(3)}")      # /path/page.html
```

---

## 8. csv - CSV 파일 처리

### 8.1 CSV 읽기

```python
import csv

# 읽기
with open('data.csv', 'r', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['name']}: {row['age']}")
```

### 8.2 CSV 쓰기

```python
import csv

data = [
    {'name': 'Alice', 'age': 30, 'city': 'Seoul'},
    {'name': 'Bob', 'age': 25, 'city': 'Busan'}
]

with open('output.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age', 'city'])
    writer.writeheader()
    writer.writerows(data)
```

---

## 9. random - 난수 생성

### 9.1 기본 난수

```python
import random

# 0.0 ~ 1.0
print(random.random())

# 정수
print(random.randint(1, 10))  # 1 ~ 10

# 범위
print(random.randrange(0, 10, 2))  # 0, 2, 4, 6, 8 중 하나

# 선택
colors = ['red', 'green', 'blue']
print(random.choice(colors))

# 여러 개 선택
print(random.sample(colors, 2))

# 셔플
numbers = [1, 2, 3, 4, 5]
random.shuffle(numbers)
print(numbers)
```

---

## 10. os - 운영체제 인터페이스

### 10.1 경로 작업

```python
import os

# 현재 디렉토리
print(os.getcwd())

# 디렉토리 변경
os.chdir('/path/to/directory')

# 환경 변수
print(os.environ.get('HOME'))
os.environ['MY_VAR'] = 'value'
```

### 10.2 파일/디렉토리 작업

```python
import os

# 디렉토리 생성
os.makedirs('data/temp', exist_ok=True)

# 파일 목록
for filename in os.listdir('.'):
    print(filename)

# 파일 정보
stat_info = os.stat('file.txt')
print(f"크기: {stat_info.st_size} bytes")

# 파일 삭제
os.remove('temp.txt')

# 디렉토리 삭제
os.rmdir('empty_dir')
```

---

## 핵심 요약

**자주 사용하는 표준 라이브러리**:

1. **collections**: Counter, defaultdict, deque, namedtuple
2. **itertools**: 조합, 순열, 무한 반복자
3. **functools**: partial, reduce, lru_cache
4. **datetime**: 날짜/시간 처리
5. **json**: JSON 직렬화/역직렬화
6. **pathlib**: 현대적 경로 처리
7. **re**: 정규표현식
8. **csv**: CSV 파일 처리
9. **random**: 난수 생성
10. **os**: 운영체제 인터페이스

---

**다음**: [부록 B. 필수 서드파티 라이브러리](부록B-서드파티_라이브러리.md)