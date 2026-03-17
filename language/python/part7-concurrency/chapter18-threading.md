# Chapter 18. 멀티스레딩 (Multithreading)

## 18.1 동시성이란?

> 🎭 **비유**: 동시성은 **멀티태스킹**입니다.
> - 병렬성 (Parallelism): 실제로 동시에 (여러 사람)
> - 동시성 (Concurrency): 빠르게 번갈아가며 (한 사람이 여러 일)

**동시성**: 여러 작업을 동시에 진행하는 것처럼 보이게 만드는 것

---

### Python의 동시성 방법

| 방법 | 사용 | 장점 | 단점 |
|-----|------|------|------|
| **Threading** | I/O 작업 | 간단, 메모리 공유 | GIL 제약 |
| **Multiprocessing** | CPU 작업 | 진짜 병렬 | 메모리 복사 비용 |
| **AsyncIO** | I/O 작업 | 효율적 | 복잡함 |

---

## 18.2 GIL (Global Interpreter Lock)

> 🔒 **비유**: GIL은 **화장실 열쇠**
> - 한 번에 한 스레드만 Python 코드 실행
> - I/O 작업 중에는 GIL 해제 (다른 스레드 실행 가능)

**GIL**: Python 인터프리터가 한 번에 하나의 스레드만 실행하도록 하는 뮤텍스

```python
# CPU 작업 - GIL 때문에 멀티스레딩 효과 없음
def cpu_bound():
    total = 0
    for i in range(10000000):
        total += i
    return total

# I/O 작업 - GIL 해제되어 멀티스레딩 효과 있음
def io_bound():
    import time
    time.sleep(1)  # I/O 대기 (GIL 해제)
    return "Done"
```

> 💡 **핵심**:
> - **CPU 작업**: Multiprocessing 사용
> - **I/O 작업**: Threading 또는 AsyncIO 사용

---

## 18.3 기본 스레드 사용

### Thread 클래스

```python
import threading
import time

def worker(name, delay):
    """작업 함수"""
    print(f"{name} 시작")
    time.sleep(delay)
    print(f"{name} 완료 ({delay}초 걸림)")

# 스레드 생성 및 시작
t1 = threading.Thread(target=worker, args=("작업1", 2))
t2 = threading.Thread(target=worker, args=("작업2", 1))

t1.start()
t2.start()

# 스레드 종료 대기
t1.join()
t2.join()

print("모든 작업 완료")

# 출력:
# 작업1 시작
# 작업2 시작
# 작업2 완료 (1초 걸림)
# 작업1 완료 (2초 걸림)
# 모든 작업 완료
```

---

### 현재 스레드 정보

```python
import threading

def show_thread_info():
    current = threading.current_thread()
    print(f"스레드 이름: {current.name}")
    print(f"스레드 ID: {current.ident}")
    print(f"활성 스레드 수: {threading.active_count()}")

# 메인 스레드
show_thread_info()

# 새 스레드
t = threading.Thread(target=show_thread_info, name="WorkerThread")
t.start()
t.join()
```

---

## 18.4 Thread 서브클래싱

```python
import threading
import time

class WorkerThread(threading.Thread):
    """커스텀 스레드 클래스"""

    def __init__(self, name, task_count):
        super().__init__()
        self.name = name
        self.task_count = task_count

    def run(self):
        """스레드가 실행할 작업"""
        print(f"{self.name} 시작 ({self.task_count}개 작업)")

        for i in range(self.task_count):
            print(f"{self.name}: 작업 {i+1}/{self.task_count}")
            time.sleep(0.5)

        print(f"{self.name} 완료")

# 사용
workers = [
    WorkerThread("Worker-1", 3),
    WorkerThread("Worker-2", 2)
]

# 시작
for worker in workers:
    worker.start()

# 대기
for worker in workers:
    worker.join()

print("모든 워커 완료")
```

---

## 18.5 데이터 공유와 경쟁 조건 (Race Condition)

> ⚠️ **비유**: 경쟁 조건은 **동시에 ATM 사용**
> - 두 사람이 동시에 잔액 확인 → 출금
> - 잔액이 음수가 될 수 있음!

### 문제 예제

```python
import threading

# 공유 변수
counter = 0

def increment():
    """카운터 증가 (안전하지 않음!)"""
    global counter
    for _ in range(100000):
        counter += 1  # 3단계: 읽기 → 계산 → 쓰기

# 2개 스레드 실행
t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)

t1.start()
t2.start()

t1.join()
t2.join()

print(f"Counter: {counter}")
# 예상: 200000
# 실제: 156789 (매번 다름!) - 경쟁 조건!
```

---

## 18.6 Lock - 동기화

> 🔐 **비유**: Lock은 **화장실 문 잠금**
> - 한 사람이 사용 중이면 다른 사람은 대기

```python
import threading

counter = 0
lock = threading.Lock()  # Lock 생성

def increment_safe():
    """카운터 증가 (안전)"""
    global counter
    for _ in range(100000):
        with lock:  # Lock 획득
            counter += 1
        # Lock 자동 해제

# 2개 스레드 실행
t1 = threading.Thread(target=increment_safe)
t2 = threading.Thread(target=increment_safe)

t1.start()
t2.start()

t1.join()
t2.join()

print(f"Counter: {counter}")  # 200000 (정확!)
```

---

### Lock 사용 방법

```python
import threading

lock = threading.Lock()

# 방법 1: with문 (권장)
with lock:
    # 임계 영역 (critical section)
    pass

# 방법 2: 수동
lock.acquire()
try:
    # 임계 영역
    pass
finally:
    lock.release()
```

---

## 18.7 RLock - 재진입 가능 Lock

> 🔄 **비유**: RLock은 **지문 인식 문**
> - 같은 사람은 여러 번 들어갈 수 있음

```python
import threading

class BankAccount:
    """은행 계좌 (RLock 사용)"""

    def __init__(self, balance):
        self.balance = balance
        self.lock = threading.RLock()  # 재진입 가능!

    def deposit(self, amount):
        """입금"""
        with self.lock:
            self.balance += amount

    def withdraw(self, amount):
        """출금"""
        with self.lock:
            if self.balance >= amount:
                self.balance -= amount
                return True
            return False

    def transfer(self, other, amount):
        """이체 (Lock 중첩!)"""
        with self.lock:  # 첫 번째 lock
            if self.withdraw(amount):  # 두 번째 lock (같은 스레드!)
                other.deposit(amount)
                return True
        return False

# 사용
acc1 = BankAccount(1000)
acc2 = BankAccount(500)

acc1.transfer(acc2, 300)
print(f"계좌1: {acc1.balance}")  # 700
print(f"계좌2: {acc2.balance}")  # 800
```

---

## 18.8 Semaphore - 제한된 리소스

> 🎫 **비유**: Semaphore는 **입장권**
> - 최대 N개까지만 입장 허용

```python
import threading
import time

# 최대 3개 동시 실행
semaphore = threading.Semaphore(3)

def access_resource(name):
    """리소스 접근"""
    print(f"{name} 대기 중...")

    with semaphore:  # 세마포어 획득
        print(f"{name} 리소스 사용 시작")
        time.sleep(2)  # 작업
        print(f"{name} 리소스 사용 완료")

# 5개 스레드 생성 (3개만 동시 실행)
threads = []
for i in range(5):
    t = threading.Thread(target=access_resource, args=(f"Thread-{i+1}",))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# 출력:
# Thread-1 대기 중...
# Thread-1 리소스 사용 시작
# Thread-2 대기 중...
# Thread-2 리소스 사용 시작
# Thread-3 대기 중...
# Thread-3 리소스 사용 시작
# Thread-4 대기 중...  <- 대기
# Thread-5 대기 중...  <- 대기
# Thread-1 리소스 사용 완료
# Thread-4 리소스 사용 시작  <- 진입
# ...
```

---

### 실전 예제: 연결 풀

```python
import threading
import time

class ConnectionPool:
    """데이터베이스 연결 풀"""

    def __init__(self, max_connections):
        self.semaphore = threading.Semaphore(max_connections)
        self.connections = []

    def get_connection(self):
        """연결 획득"""
        self.semaphore.acquire()
        print(f"{threading.current_thread().name} 연결 획득")
        return f"Connection-{threading.current_thread().name}"

    def release_connection(self, conn):
        """연결 해제"""
        print(f"{threading.current_thread().name} 연결 해제")
        self.semaphore.release()

# 사용
pool = ConnectionPool(max_connections=2)

def use_database(name):
    conn = pool.get_connection()
    time.sleep(1)  # DB 작업
    pool.release_connection(conn)

threads = [threading.Thread(target=use_database, args=(f"Worker-{i}",)) for i in range(5)]

for t in threads:
    t.start()

for t in threads:
    t.join()
```

---

## 18.9 Event - 신호 대기

> 🚦 **비유**: Event는 **신호등**
> - 빨강: 대기
> - 초록: 진행

```python
import threading
import time

event = threading.Event()

def waiter():
    """신호 대기"""
    print("신호 대기 중...")
    event.wait()  # 신호가 올 때까지 대기
    print("신호 받음! 작업 시작")

def setter():
    """신호 전송"""
    print("5초 후 신호 전송...")
    time.sleep(5)
    print("신호 전송!")
    event.set()  # 신호 설정

# 스레드 생성
t1 = threading.Thread(target=waiter)
t2 = threading.Thread(target=setter)

t1.start()
t2.start()

t1.join()
t2.join()

# 출력:
# 신호 대기 중...
# 5초 후 신호 전송...
# 신호 전송!
# 신호 받음! 작업 시작
```

---

### 실전 예제: 생산자-소비자

```python
import threading
import time
import random

class Producer(threading.Thread):
    """생산자"""

    def __init__(self, queue, event, max_items):
        super().__init__()
        self.queue = queue
        self.event = event
        self.max_items = max_items

    def run(self):
        for i in range(self.max_items):
            item = f"Item-{i+1}"
            self.queue.append(item)
            print(f"생산: {item}")

            if len(self.queue) >= 5:
                print("큐 가득 참 - 소비자에게 신호")
                self.event.set()

            time.sleep(random.uniform(0.1, 0.5))

        self.event.set()  # 완료 신호

class Consumer(threading.Thread):
    """소비자"""

    def __init__(self, queue, event):
        super().__init__()
        self.queue = queue
        self.event = event

    def run(self):
        while True:
            self.event.wait()  # 신호 대기

            while self.queue:
                item = self.queue.pop(0)
                print(f"  소비: {item}")
                time.sleep(0.2)

            self.event.clear()  # 신호 초기화

            if not producer.is_alive():
                break

# 사용
queue = []
event = threading.Event()

producer = Producer(queue, event, max_items=10)
consumer = Consumer(queue, event)

producer.start()
consumer.start()

producer.join()
consumer.join()
```

---

## 18.10 ThreadPoolExecutor - 스레드 풀

> 🏊 **비유**: 스레드 풀은 **수영장**
> - 미리 만들어진 스레드들
> - 작업을 스레드에 할당

```python
from concurrent.futures import ThreadPoolExecutor
import time

def task(name, duration):
    """작업 함수"""
    print(f"{name} 시작")
    time.sleep(duration)
    print(f"{name} 완료")
    return f"{name} 결과"

# 스레드 풀 (최대 3개 스레드)
with ThreadPoolExecutor(max_workers=3) as executor:
    # 여러 작업 제출
    futures = []
    for i in range(5):
        future = executor.submit(task, f"작업{i+1}", 1)
        futures.append(future)

    # 결과 수집
    for future in futures:
        result = future.result()  # 완료 대기
        print(f"결과: {result}")
```

---

### map() 사용

```python
from concurrent.futures import ThreadPoolExecutor
import time

def process_url(url):
    """URL 처리"""
    print(f"다운로드 중: {url}")
    time.sleep(1)  # 시뮬레이션
    return f"{url} 완료"

urls = [
    "http://example.com/page1",
    "http://example.com/page2",
    "http://example.com/page3",
    "http://example.com/page4"
]

# map으로 간단하게
with ThreadPoolExecutor(max_workers=3) as executor:
    results = executor.map(process_url, urls)

    for result in results:
        print(result)
```

---

## 18.11 실전 예제

### 예제 1: 웹 스크래퍼

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def fetch_url(url):
    """URL 다운로드 시뮬레이션"""
    print(f"📥 다운로드 시작: {url}")
    time.sleep(1)  # 실제로는 requests.get(url)
    print(f"✅ 다운로드 완료: {url}")
    return {
        'url': url,
        'status': 200,
        'size': len(url) * 100
    }

# URL 목록
urls = [f"http://example.com/page{i}" for i in range(1, 11)]

# 순차 실행
print("=== 순차 실행 ===")
start = time.time()
results_sequential = [fetch_url(url) for url in urls]
sequential_time = time.time() - start
print(f"걸린 시간: {sequential_time:.2f}초\n")

# 병렬 실행
print("=== 병렬 실행 (5개 스레드) ===")
start = time.time()
with ThreadPoolExecutor(max_workers=5) as executor:
    results_parallel = list(executor.map(fetch_url, urls))
parallel_time = time.time() - start
print(f"걸린 시간: {parallel_time:.2f}초")

print(f"\n속도 향상: {sequential_time/parallel_time:.1f}배")
```

---

### 예제 2: 파일 처리

```python
from concurrent.futures import ThreadPoolExecutor
import time

class FileProcessor:
    """파일 처리기"""

    def __init__(self, max_workers=4):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.results = []

    def process_file(self, filename):
        """파일 처리"""
        print(f"처리 중: {filename}")
        time.sleep(0.5)  # 실제 처리 시뮬레이션

        # 결과
        return {
            'file': filename,
            'lines': 100,
            'words': 500
        }

    def process_batch(self, filenames):
        """여러 파일 병렬 처리"""
        futures = {}

        for filename in filenames:
            future = self.executor.submit(self.process_file, filename)
            futures[future] = filename

        # 완료되는 대로 처리
        from concurrent.futures import as_completed

        for future in as_completed(futures):
            filename = futures[future]
            try:
                result = future.result()
                self.results.append(result)
                print(f"✅ 완료: {filename}")
            except Exception as e:
                print(f"❌ 에러 ({filename}): {e}")

        return self.results

    def shutdown(self):
        """종료"""
        self.executor.shutdown(wait=True)

# 사용
processor = FileProcessor(max_workers=3)

files = [f"file{i}.txt" for i in range(1, 11)]
results = processor.process_batch(files)

print(f"\n총 {len(results)}개 파일 처리 완료")

processor.shutdown()
```

---

### 예제 3: 데이터베이스 작업

```python
import threading
import time
from queue import Queue

class DatabaseWorker(threading.Thread):
    """데이터베이스 워커 스레드"""

    def __init__(self, queue, results):
        super().__init__()
        self.queue = queue
        self.results = results
        self.daemon = True

    def run(self):
        while True:
            task = self.queue.get()
            if task is None:
                break

            # 데이터베이스 작업 시뮬레이션
            user_id = task
            time.sleep(0.1)
            user_data = {'id': user_id, 'name': f'User{user_id}'}

            self.results.append(user_data)
            self.queue.task_done()

# 사용
queue = Queue()
results = []

# 5개 워커 스레드 생성
workers = []
for i in range(5):
    worker = DatabaseWorker(queue, results)
    worker.start()
    workers.append(worker)

# 작업 추가
for user_id in range(1, 51):
    queue.put(user_id)

# 모든 작업 완료 대기
queue.join()

# 워커 종료
for _ in workers:
    queue.put(None)

for worker in workers:
    worker.join()

print(f"{len(results)}개 사용자 데이터 로드 완료")
```

---

## 실전 팁

### 💡 Tip 1: I/O 작업에만 Threading

```python
# ✅ I/O 작업 (효과적)
- 파일 읽기/쓰기
- 네트워크 요청
- 데이터베이스 쿼리

# ❌ CPU 작업 (효과 없음)
- 이미지 처리
- 데이터 분석
- 암호화
```

---

### 💡 Tip 2: 항상 Lock 사용

```python
# ❌ 위험
counter = 0

def increment():
    global counter
    counter += 1

# ✅ 안전
counter = 0
lock = threading.Lock()

def increment():
    global counter
    with lock:
        counter += 1
```

---

### 💡 Tip 3: ThreadPoolExecutor 사용

```python
# ❌ 수동 스레드 관리 (복잡)
threads = []
for task in tasks:
    t = threading.Thread(target=process, args=(task,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# ✅ ThreadPoolExecutor (간단)
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=5) as executor:
    executor.map(process, tasks)
```

---

## 핵심 요약

### 꼭 기억할 것

1. **GIL**
   - I/O 작업: Threading 효과적
   - CPU 작업: Multiprocessing 사용

2. **기본 사용**
   ```python
   t = threading.Thread(target=func)
   t.start()
   t.join()
   ```

3. **동기화**
   - `Lock`: 상호 배제
   - `RLock`: 재진입 가능
   - `Semaphore`: 제한된 리소스
   - `Event`: 신호 대기

4. **ThreadPoolExecutor**
   ```python
   with ThreadPoolExecutor(max_workers=5) as executor:
       results = executor.map(func, items)
   ```

5. **주의사항**
   - 경쟁 조건 방지
   - 데드락 주의
   - 리소스 정리

---

## 다음 챕터 예고

Chapter 19에서는 **멀티프로세싱**을 다룹니다:
- multiprocessing 모듈
- Process vs Thread
- 프로세스 풀
- 프로세스 간 통신

---

[다음: Chapter 19. 멀티프로세싱 →](chapter19-multiprocessing.md)