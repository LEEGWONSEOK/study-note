# Chapter 24. 성능 최적화 (Performance Optimization)

> **Java 개발자를 위한 노트**: Python은 Java보다 느리지만, 올바른 최적화 기법을 사용하면 충분히 빠른 성능을 얻을 수 있습니다. NumPy, Cython 등을 활용하면 C 수준의 성능도 가능합니다.

---

## 1. 성능 측정 - 먼저 측정하라

**비유**: 성능 최적화는 **병원 진료**와 같습니다. 먼저 **진단(측정)**을 하고, 그 다음 **처방(최적화)**을 합니다.

### 1.1 time 모듈 - 간단한 측정

```python
import time

def slow_function():
    """느린 함수"""
    total = 0
    for i in range(1000000):
        total += i
    return total

# 측정
start = time.time()
result = slow_function()
elapsed = time.time() - start

print(f"결과: {result}")
print(f"소요 시간: {elapsed:.4f}초")

# 출력:
# 결과: 499999500000
# 소요 시간: 0.0523초
```

### 1.2 timeit - 정확한 측정

```python
import timeit

# 한 줄 코드 측정
time1 = timeit.timeit('sum(range(1000))', number=10000)
print(f"sum(range(1000)): {time1:.4f}초")

# 함수 측정
def my_sum():
    total = 0
    for i in range(1000):
        total += i
    return total

time2 = timeit.timeit(my_sum, number=10000)
print(f"my_sum(): {time2:.4f}초")

# 비교
if time1 < time2:
    print(f"✅ sum()이 {time2/time1:.2f}배 빠름")

# 출력:
# sum(range(1000)): 0.1234초
# my_sum(): 0.5678초
# ✅ sum()이 4.60배 빠름
```

### 1.3 cProfile - 프로파일링

```python
import cProfile
import pstats

def complex_function():
    """복잡한 함수"""
    data = []

    for i in range(1000):
        data.append(i ** 2)

    result = sum(data)
    return result

# 프로파일링
profiler = cProfile.Profile()
profiler.enable()

complex_function()

profiler.disable()

# 결과 출력
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)  # 상위 10개

# 출력:
#    ncalls  tottime  percall  cumtime  percall filename:lineno(function)
#         1    0.001    0.001    0.002    0.002 test.py:3(complex_function)
#      1000    0.001    0.000    0.001    0.000 {built-in method builtins.append}
#         1    0.000    0.000    0.000    0.000 {built-in method builtins.sum}
```

---

## 2. 내장 함수와 컴프리헨션

### 2.1 내장 함수 활용

```python
import timeit

# ❌ 느린 방법 - 직접 구현
def sum_slow(numbers):
    total = 0
    for num in numbers:
        total += num
    return total

# ✅ 빠른 방법 - 내장 함수
def sum_fast(numbers):
    return sum(numbers)

numbers = range(10000)

time_slow = timeit.timeit(lambda: sum_slow(numbers), number=1000)
time_fast = timeit.timeit(lambda: sum_fast(numbers), number=1000)

print(f"직접 구현: {time_slow:.4f}초")
print(f"내장 함수: {time_fast:.4f}초")
print(f"⚡ 속도 향상: {time_slow/time_fast:.2f}x")

# 출력:
# 직접 구현: 0.3456초
# 내장 함수: 0.0789초
# ⚡ 속도 향상: 4.38x
```

### 2.2 리스트 컴프리헨션

```python
import timeit

# ❌ 느린 방법 - append
def create_list_slow():
    result = []
    for i in range(10000):
        result.append(i ** 2)
    return result

# ✅ 빠른 방법 - 리스트 컴프리헨션
def create_list_fast():
    return [i ** 2 for i in range(10000)]

time_slow = timeit.timeit(create_list_slow, number=1000)
time_fast = timeit.timeit(create_list_fast, number=1000)

print(f"append: {time_slow:.4f}초")
print(f"컴프리헨션: {time_fast:.4f}초")
print(f"⚡ 속도 향상: {time_slow/time_fast:.2f}x")

# 출력:
# append: 1.2345초
# 컴프리헨션: 0.8901초
# ⚡ 속도 향상: 1.39x
```

### 2.3 map과 filter

```python
import timeit

numbers = range(10000)

# map 사용
time_map = timeit.timeit(
    lambda: list(map(lambda x: x ** 2, numbers)),
    number=1000
)

# 리스트 컴프리헨션
time_comp = timeit.timeit(
    lambda: [x ** 2 for x in numbers],
    number=1000
)

print(f"map: {time_map:.4f}초")
print(f"컴프리헨션: {time_comp:.4f}초")

# 대부분의 경우 컴프리헨션이 더 빠르고 가독성도 좋음
```

---

## 3. 문자열 최적화

### 3.1 문자열 연결

```python
import timeit

# ❌ 매우 느린 방법 - + 연산자
def concat_slow():
    result = ""
    for i in range(10000):
        result += str(i)
    return result

# ✅ 빠른 방법 - join
def concat_fast():
    return "".join(str(i) for i in range(10000))

time_slow = timeit.timeit(concat_slow, number=10)
time_fast = timeit.timeit(concat_fast, number=10)

print(f"+ 연산자: {time_slow:.4f}초")
print(f"join: {time_fast:.4f}초")
print(f"⚡ 속도 향상: {time_slow/time_fast:.2f}x")

# 출력:
# + 연산자: 0.8234초
# join: 0.0123초
# ⚡ 속도 향상: 66.94x
```

### 3.2 f-string vs format

```python
import timeit

name = "Alice"
age = 30

# 여러 포맷팅 방법 비교
time_percent = timeit.timeit(lambda: "%s is %d years old" % (name, age), number=100000)
time_format = timeit.timeit(lambda: "{} is {} years old".format(name, age), number=100000)
time_fstring = timeit.timeit(lambda: f"{name} is {age} years old", number=100000)

print(f"% 포맷: {time_percent:.4f}초")
print(f".format(): {time_format:.4f}초")
print(f"f-string: {time_fstring:.4f}초 ⚡")

# f-string이 가장 빠름!
```

---

## 4. 자료구조 선택

### 4.1 리스트 vs 셋 - 멤버십 테스트

```python
import timeit

# 데이터 준비
list_data = list(range(10000))
set_data = set(range(10000))
target = 9999

# 리스트에서 검색
time_list = timeit.timeit(lambda: target in list_data, number=10000)

# 셋에서 검색
time_set = timeit.timeit(lambda: target in set_data, number=10000)

print(f"리스트: {time_list:.4f}초")
print(f"셋: {time_set:.4f}초")
print(f"⚡ 속도 향상: {time_list/time_set:.2f}x")

# 출력:
# 리스트: 1.2345초
# 셋: 0.0001초
# ⚡ 속도 향상: 12345.00x  # 엄청난 차이!
```

### 4.2 deque - 양끝 삽입/삭제

```python
import timeit
from collections import deque

# 리스트 - 앞에 삽입 (느림)
def list_insert_front():
    lst = []
    for i in range(1000):
        lst.insert(0, i)
    return lst

# deque - 앞에 삽입 (빠름)
def deque_insert_front():
    dq = deque()
    for i in range(1000):
        dq.appendleft(i)
    return dq

time_list = timeit.timeit(list_insert_front, number=1000)
time_deque = timeit.timeit(deque_insert_front, number=1000)

print(f"리스트: {time_list:.4f}초")
print(f"deque: {time_deque:.4f}초")
print(f"⚡ 속도 향상: {time_list/time_deque:.2f}x")

# 출력:
# 리스트: 2.3456초
# deque: 0.0123초
# ⚡ 속도 향상: 190.78x
```

### 4.3 딕셔너리 vs 리스트 - 키 기반 조회

```python
import timeit

# 데이터 준비
users_list = [{'id': i, 'name': f'User{i}'} for i in range(10000)]
users_dict = {i: {'id': i, 'name': f'User{i}'} for i in range(10000)}
target_id = 9999

# 리스트에서 검색
def find_in_list():
    for user in users_list:
        if user['id'] == target_id:
            return user

# 딕셔너리에서 검색
def find_in_dict():
    return users_dict.get(target_id)

time_list = timeit.timeit(find_in_list, number=1000)
time_dict = timeit.timeit(find_in_dict, number=1000)

print(f"리스트: {time_list:.4f}초")
print(f"딕셔너리: {time_dict:.4f}초")
print(f"⚡ 속도 향상: {time_list/time_dict:.2f}x")
```

---

## 5. 제너레이터 활용

### 5.1 메모리 효율

```python
import sys

# 리스트 - 모든 데이터를 메모리에 저장
def get_numbers_list(n):
    return [i ** 2 for i in range(n)]

# 제너레이터 - 필요할 때만 생성
def get_numbers_gen(n):
    return (i ** 2 for i in range(n))

n = 1000000

# 메모리 사용량 비교
list_obj = get_numbers_list(n)
gen_obj = get_numbers_gen(n)

print(f"리스트 크기: {sys.getsizeof(list_obj):,} bytes")
print(f"제너레이터 크기: {sys.getsizeof(gen_obj):,} bytes")

# 출력:
# 리스트 크기: 8,448,728 bytes
# 제너레이터 크기: 112 bytes
# ⚡ 약 75,000배 메모리 절약!
```

### 5.2 지연 평가 (Lazy Evaluation)

```python
def process_large_file(filename):
    """대용량 파일 처리 - 제너레이터 사용"""
    with open(filename, 'r') as f:
        for line in f:
            # 한 줄씩 처리 (메모리 효율적)
            yield line.strip().upper()

# 사용
for processed_line in process_large_file('large_file.txt'):
    # 필요할 때만 처리
    if 'ERROR' in processed_line:
        print(processed_line)
```

---

## 6. NumPy로 속도 향상

**비유**: NumPy는 Python의 **터보 엔진**입니다. 특히 수치 연산에서 C 수준의 성능을 제공합니다.

### 6.1 리스트 vs NumPy 배열

```python
import timeit
import numpy as np

# 리스트
def list_operations():
    a = list(range(10000))
    b = list(range(10000))
    c = [x + y for x, y in zip(a, b)]
    return c

# NumPy
def numpy_operations():
    a = np.arange(10000)
    b = np.arange(10000)
    c = a + b
    return c

time_list = timeit.timeit(list_operations, number=1000)
time_numpy = timeit.timeit(numpy_operations, number=1000)

print(f"리스트: {time_list:.4f}초")
print(f"NumPy: {time_numpy:.4f}초")
print(f"⚡ 속도 향상: {time_list/time_numpy:.2f}x")

# 출력:
# 리스트: 1.2345초
# NumPy: 0.0123초
# ⚡ 속도 향상: 100.37x
```

### 6.2 NumPy 벡터화 연산

```python
import numpy as np
import timeit

# ❌ 느린 방법 - 반복문
def calculate_slow():
    a = list(range(100000))
    result = []
    for x in a:
        result.append(x ** 2 + 2 * x + 1)
    return result

# ✅ 빠른 방법 - NumPy 벡터화
def calculate_fast():
    a = np.arange(100000)
    result = a ** 2 + 2 * a + 1
    return result

time_slow = timeit.timeit(calculate_slow, number=100)
time_fast = timeit.timeit(calculate_fast, number=100)

print(f"반복문: {time_slow:.4f}초")
print(f"NumPy: {time_fast:.4f}초")
print(f"⚡ 속도 향상: {time_slow/time_fast:.2f}x")
```

---

## 7. 캐싱과 메모이제이션

### 7.1 functools.lru_cache

```python
import timeit
from functools import lru_cache

# 캐싱 없음
def fibonacci_slow(n):
    if n < 2:
        return n
    return fibonacci_slow(n-1) + fibonacci_slow(n-2)

# 캐싱 있음
@lru_cache(maxsize=None)
def fibonacci_fast(n):
    if n < 2:
        return n
    return fibonacci_fast(n-1) + fibonacci_fast(n-2)

# 비교
n = 35

time_slow = timeit.timeit(lambda: fibonacci_slow(n), number=1)
time_fast = timeit.timeit(lambda: fibonacci_fast(n), number=1)

print(f"캐싱 없음: {time_slow:.4f}초")
print(f"캐싱 있음: {time_fast:.6f}초")
print(f"⚡ 속도 향상: {time_slow/time_fast:.2f}x")

# 출력:
# 캐싱 없음: 2.3456초
# 캐싱 있음: 0.000012초
# ⚡ 속도 향상: 195466.67x  # 엄청난 차이!
```

### 7.2 커스텀 캐시

```python
from functools import wraps
import time

def timed_cache(seconds=60):
    """시간 제한 캐시 데코레이터"""
    def decorator(func):
        cache = {}
        cache_time = {}

        @wraps(func)
        def wrapper(*args):
            now = time.time()

            # 캐시 확인
            if args in cache:
                if now - cache_time[args] < seconds:
                    print(f"✅ 캐시 히트: {args}")
                    return cache[args]

            # 캐시 갱신
            print(f"🔄 계산 실행: {args}")
            result = func(*args)
            cache[args] = result
            cache_time[args] = now

            return result

        return wrapper
    return decorator

@timed_cache(seconds=5)
def expensive_operation(n):
    """비용이 큰 연산"""
    time.sleep(1)
    return n ** 2

# 사용
print(expensive_operation(10))  # 계산 실행
print(expensive_operation(10))  # 캐시 히트
time.sleep(6)
print(expensive_operation(10))  # 캐시 만료, 계산 실행
```

---

## 8. 실전 예제 - 데이터 처리 최적화

### 8.1 CSV 파일 처리

```python
import csv
import pandas as pd
import timeit

# ❌ 느린 방법 - 표준 라이브러리
def process_csv_slow(filename):
    data = []
    with open(filename, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            if int(row['age']) > 30:
                data.append({
                    'name': row['name'],
                    'salary': float(row['salary']) * 1.1
                })
    return data

# ✅ 빠른 방법 - pandas
def process_csv_fast(filename):
    df = pd.read_csv(filename)
    df_filtered = df[df['age'] > 30].copy()
    df_filtered['salary'] = df_filtered['salary'] * 1.1
    return df_filtered[['name', 'salary']].to_dict('records')

# 대용량 CSV 파일에서 pandas가 훨씬 빠름
```

### 8.2 병렬 처리

```python
from concurrent.futures import ProcessPoolExecutor
import timeit

def process_item(item):
    """CPU 집약적 작업"""
    result = 0
    for i in range(1000000):
        result += i ** 2
    return result * item

items = list(range(10))

# 순차 처리
def sequential():
    return [process_item(item) for item in items]

# 병렬 처리
def parallel():
    with ProcessPoolExecutor() as executor:
        return list(executor.map(process_item, items))

time_seq = timeit.timeit(sequential, number=1)
time_par = timeit.timeit(parallel, number=1)

print(f"순차: {time_seq:.2f}초")
print(f"병렬: {time_par:.2f}초")
print(f"⚡ 속도 향상: {time_seq/time_par:.2f}x")
```

---

## 9. 실전 팁

### 💡 Tip 1: 성능 최적화 가이드라인

```python
"""
최적화 우선순위:

1. 알고리즘 개선
   - O(n²) → O(n log n) 변경이 가장 효과적

2. 적절한 자료구조 선택
   - 리스트 vs 셋 vs 딕셔너리

3. 내장 함수/라이브러리 활용
   - sum(), map(), NumPy

4. 불필요한 연산 제거
   - 루프 밖으로 이동 가능한 연산

5. 캐싱
   - 반복 계산 피하기

6. 병렬 처리
   - CPU 집약적 작업

7. Cython/C 확장
   - 최후의 수단
"""
```

### 💡 Tip 2: 조기 최적화 피하기

```python
# ❌ 조기 최적화 - 가독성 저하
def bad_optimization():
    # 이해하기 어려운 코드
    return [x for x in [i**2 for i in range(100)] if x%2==0]

# ✅ 먼저 명확하게 작성
def good_approach():
    # 1. 먼저 명확하게 작성
    numbers = range(100)
    squares = [i ** 2 for i in numbers]
    even_squares = [x for x in squares if x % 2 == 0]
    return even_squares

    # 2. 측정
    # 3. 필요하면 최적화
```

### 💡 Tip 3: 프로파일링 먼저

```python
import cProfile
import pstats

def main():
    # 여러 함수 호출
    process_data()
    analyze_results()
    generate_report()

# 프로파일링
profiler = cProfile.Profile()
profiler.enable()
main()
profiler.disable()

# 어느 함수가 느린지 확인
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)

# → 가장 느린 부분만 최적화
```

---

## 10. 연습 문제

### 문제 1: 중복 제거 최적화
리스트에서 중복을 제거하는 여러 방법을 비교하세요.

<details>
<summary>정답 보기</summary>

```python
import timeit

data = list(range(10000)) * 10  # 중복이 많은 데이터

# 방법 1: 반복문
def remove_duplicates_loop(lst):
    result = []
    for item in lst:
        if item not in result:
            result.append(item)
    return result

# 방법 2: 셋 사용
def remove_duplicates_set(lst):
    return list(set(lst))

# 방법 3: dict.fromkeys
def remove_duplicates_dict(lst):
    return list(dict.fromkeys(lst))

# 성능 비교
time_loop = timeit.timeit(lambda: remove_duplicates_loop(data), number=10)
time_set = timeit.timeit(lambda: remove_duplicates_set(data), number=10)
time_dict = timeit.timeit(lambda: remove_duplicates_dict(data), number=10)

print(f"반복문: {time_loop:.4f}초")
print(f"셋: {time_set:.4f}초 ⚡")
print(f"dict.fromkeys: {time_dict:.4f}초 ⚡")

print(f"\n속도 향상:")
print(f"  셋: {time_loop/time_set:.2f}x")
print(f"  dict: {time_loop/time_dict:.2f}x")

# 출력:
# 반복문: 12.3456초
# 셋: 0.0012초 ⚡
# dict.fromkeys: 0.0015초 ⚡
#
# 속도 향상:
#   셋: 10288.00x
#   dict: 8230.40x
```
</details>

### 문제 2: 데이터 필터링 최적화
대량의 데이터에서 조건에 맞는 항목을 찾는 코드를 최적화하세요.

<details>
<summary>정답 보기</summary>

```python
import timeit
import numpy as np

# 데이터 준비
data_list = list(range(1000000))
data_numpy = np.arange(1000000)

# 방법 1: 리스트 컴프리헨션
def filter_list_comp():
    return [x for x in data_list if x % 2 == 0 and x > 500000]

# 방법 2: filter() 함수
def filter_function():
    return list(filter(lambda x: x % 2 == 0 and x > 500000, data_list))

# 방법 3: NumPy 마스킹
def filter_numpy():
    mask = (data_numpy % 2 == 0) & (data_numpy > 500000)
    return data_numpy[mask]

# 성능 비교
time_comp = timeit.timeit(filter_list_comp, number=10)
time_func = timeit.timeit(filter_function, number=10)
time_numpy = timeit.timeit(filter_numpy, number=10)

print(f"리스트 컴프리헨션: {time_comp:.4f}초")
print(f"filter() 함수: {time_func:.4f}초")
print(f"NumPy: {time_numpy:.4f}초 ⚡")

print(f"\n속도 향상:")
print(f"  NumPy: {time_comp/time_numpy:.2f}x")

# 출력:
# 리스트 컴프리헨션: 1.2345초
# filter() 함수: 1.5678초
# NumPy: 0.0123초 ⚡
#
# 속도 향상:
#   NumPy: 100.37x
```
</details>

---

## 핵심 요약

1. **성능 측정 도구**
   - `time`: 간단한 측정
   - `timeit`: 정확한 벤치마크
   - `cProfile`: 프로파일링

2. **최적화 기법**
   ```python
   # 내장 함수 활용
   sum(numbers)  # ✅ 빠름

   # 컴프리헨션
   [x**2 for x in numbers]  # ✅ 빠름

   # 적절한 자료구조
   set_data  # ✅ 검색 빠름
   deque  # ✅ 양끝 삽입/삭제 빠름
   ```

3. **성능 비교**
   - 셋 검색 > 리스트 검색 (수천 배)
   - join > + 연산자 (수십 배)
   - NumPy > 리스트 (수십~수백 배)
   - lru_cache로 재귀 (수만 배 이상)

4. **원칙**
   - 먼저 측정, 그 다음 최적화
   - 조기 최적화 피하기
   - 알고리즘 개선이 최우선

5. **Java와 비교**
   - Python은 느리지만, NumPy/Cython으로 극복 가능
   - GIL 때문에 CPU 작업은 멀티프로세싱 필요

---

**축하합니다!** Part 8 (실전 활용)을 완료했습니다. 이제 Python 언어의 핵심을 모두 마스터했습니다! 🎉