# 부록 B. 필수 서드파티 라이브러리

> PyPI에는 수십만 개의 패키지가 있지만, 이 중에서 거의 모든 Python 개발자가 사용하는 필수 라이브러리를 소개합니다.

---

## 1. requests - HTTP 클라이언트

**설치**: `pip install requests`

### 1.1 기본 사용법

```python
import requests

# GET 요청
response = requests.get('https://api.github.com/users/github')
print(response.status_code)  # 200
print(response.json())        # JSON 자동 파싱

# POST 요청
data = {'username': 'alice', 'password': 'secret'}
response = requests.post('https://api.example.com/login', json=data)

# 헤더 추가
headers = {'Authorization': 'Bearer token123'}
response = requests.get('https://api.example.com/protected', headers=headers)

# 쿼리 파라미터
params = {'q': 'python', 'page': 1}
response = requests.get('https://api.example.com/search', params=params)
```

### 1.2 실전 예제

```python
import requests

def get_weather(city):
    """날씨 정보 가져오기"""
    url = 'https://api.openweathermap.org/data/2.5/weather'
    params = {
        'q': city,
        'appid': 'your_api_key',
        'units': 'metric'
    }

    try:
        response = requests.get(url, params=params, timeout=5)
        response.raise_for_status()

        data = response.json()
        return {
            'city': data['name'],
            'temp': data['main']['temp'],
            'description': data['weather'][0]['description']
        }

    except requests.exceptions.RequestException as e:
        print(f"에러: {e}")
        return None

# 사용
weather = get_weather('Seoul')
if weather:
    print(f"{weather['city']}: {weather['temp']}°C, {weather['description']}")
```

---

## 2. pandas - 데이터 분석

**설치**: `pip install pandas`

### 2.1 DataFrame 기본

```python
import pandas as pd

# DataFrame 생성
data = {
    'name': ['Alice', 'Bob', 'Charlie', 'David'],
    'age': [25, 30, 35, 28],
    'salary': [50000, 60000, 75000, 55000]
}
df = pd.DataFrame(data)

print(df)
#       name  age  salary
# 0    Alice   25   50000
# 1      Bob   30   60000
# 2  Charlie   35   75000
# 3    David   28   55000

# 통계
print(df.describe())
print(df['age'].mean())  # 평균 나이
```

### 2.2 데이터 필터링

```python
import pandas as pd

df = pd.read_csv('employees.csv')

# 조건 필터링
high_earners = df[df['salary'] > 60000]
young_employees = df[df['age'] < 30]

# 여러 조건
result = df[(df['age'] > 25) & (df['salary'] < 70000)]

# 정렬
sorted_df = df.sort_values('salary', ascending=False)

# 그룹화
by_dept = df.groupby('department')['salary'].mean()
print(by_dept)
```

### 2.3 CSV 처리

```python
import pandas as pd

# CSV 읽기
df = pd.read_csv('data.csv', encoding='utf-8')

# 데이터 정제
df = df.dropna()  # 결측치 제거
df = df.drop_duplicates()  # 중복 제거

# 컬럼 선택
subset = df[['name', 'age', 'salary']]

# 새 컬럼 추가
df['bonus'] = df['salary'] * 0.1

# CSV 저장
df.to_csv('processed.csv', index=False, encoding='utf-8')
```

---

## 3. numpy - 수치 연산

**설치**: `pip install numpy`

### 3.1 배열 기본

```python
import numpy as np

# 배열 생성
arr = np.array([1, 2, 3, 4, 5])
print(arr)  # [1 2 3 4 5]

# 2D 배열
matrix = np.array([[1, 2, 3], [4, 5, 6]])
print(matrix.shape)  # (2, 3)

# 특수 배열
zeros = np.zeros((3, 3))
ones = np.ones((2, 4))
identity = np.eye(3)
range_arr = np.arange(0, 10, 2)  # [0 2 4 6 8]
linspace_arr = np.linspace(0, 1, 5)  # [0.   0.25 0.5  0.75 1.  ]
```

### 3.2 벡터화 연산

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5])
b = np.array([10, 20, 30, 40, 50])

# 벡터 연산 (매우 빠름!)
print(a + b)     # [11 22 33 44 55]
print(a * b)     # [10 40 90 160 250]
print(a ** 2)    # [1 4 9 16 25]

# 수학 함수
print(np.sqrt(a))      # 제곱근
print(np.exp(a))       # 지수
print(np.log(a))       # 로그
print(np.sin(a))       # 사인
```

### 3.3 행렬 연산

```python
import numpy as np

A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

# 행렬 곱
C = np.dot(A, B)
# 또는
C = A @ B

# 전치
print(A.T)

# 역행렬
inv_A = np.linalg.inv(A)

# 고유값
eigenvalues = np.linalg.eigvals(A)
```

---

## 4. pytest - 테스팅 프레임워크

**설치**: `pip install pytest`

### 4.1 기본 테스트

```python
# test_calculator.py
def add(a, b):
    return a + b

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0

def test_add_negative():
    assert add(-5, -3) == -8

# 실행: pytest test_calculator.py
```

### 4.2 Fixture 사용

```python
import pytest

@pytest.fixture
def sample_data():
    return [1, 2, 3, 4, 5]

def test_sum(sample_data):
    assert sum(sample_data) == 15

def test_len(sample_data):
    assert len(sample_data) == 5
```

---

## 5. black - 코드 포매터

**설치**: `pip install black`

### 5.1 사용법

```bash
# 파일 포맷팅
black script.py

# 디렉토리 전체
black src/

# 체크만 (변경 안 함)
black --check src/

# diff 보기
black --diff script.py
```

### 5.2 설정 파일

```toml
# pyproject.toml
[tool.black]
line-length = 100
target-version = ['py38', 'py39', 'py310', 'py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | build
  | dist
)/
'''
```

---

## 6. SQLAlchemy - ORM

**설치**: `pip install sqlalchemy`

### 6.1 모델 정의

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    email = Column(String(100))

# 데이터베이스 연결
engine = create_engine('sqlite:///database.db')
Base.metadata.create_all(engine)

# 세션
Session = sessionmaker(bind=engine)
session = Session()
```

### 6.2 CRUD 작업

```python
# Create
user = User(name='Alice', email='alice@example.com')
session.add(user)
session.commit()

# Read
users = session.query(User).all()
alice = session.query(User).filter_by(name='Alice').first()

# Update
alice.email = 'newemail@example.com'
session.commit()

# Delete
session.delete(alice)
session.commit()
```

---

## 7. python-dotenv - 환경 변수 관리

**설치**: `pip install python-dotenv`

### 7.1 사용법

```bash
# .env 파일
DATABASE_URL=postgresql://localhost/mydb
API_KEY=secret_key_123
DEBUG=True
```

```python
from dotenv import load_dotenv
import os

# .env 파일 로드
load_dotenv()

# 환경 변수 사용
db_url = os.getenv('DATABASE_URL')
api_key = os.getenv('API_KEY')
debug = os.getenv('DEBUG') == 'True'

print(f"DB: {db_url}")
print(f"API Key: {api_key}")
print(f"Debug: {debug}")
```

---

## 8. httpx - 비동기 HTTP 클라이언트

**설치**: `pip install httpx`

### 8.1 비동기 요청

```python
import asyncio
import httpx

async def fetch_url(url):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

async def main():
    urls = [
        'https://api.github.com/users/github',
        'https://api.github.com/users/python',
        'https://api.github.com/users/django'
    ]

    # 동시에 요청
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)

    for response in responses:
        print(response.json()['login'])

asyncio.run(main())
```

---

## 9. rich - 터미널 출력 향상

**설치**: `pip install rich`

### 9.1 예쁜 출력

```python
from rich import print
from rich.console import Console
from rich.table import Table
from rich.progress import track
import time

# 컬러 출력
print("[bold green]성공![/bold green]")
print("[bold red]에러 발생![/bold red]")

# 테이블
table = Table(title="사용자 목록")
table.add_column("ID", style="cyan")
table.add_column("이름", style="green")
table.add_column("나이", style="yellow")

table.add_row("1", "Alice", "25")
table.add_row("2", "Bob", "30")

console = Console()
console.print(table)

# 진행률 바
for i in track(range(100), description="처리 중..."):
    time.sleep(0.01)
```

---

## 10. pydantic - 데이터 검증

**설치**: `pip install pydantic`

### 10.1 모델 정의

```python
from pydantic import BaseModel, EmailStr, validator
from typing import Optional

class User(BaseModel):
    id: int
    name: str
    email: EmailStr
    age: Optional[int] = None

    @validator('age')
    def check_age(cls, v):
        if v is not None and v < 0:
            raise ValueError('나이는 0 이상이어야 합니다')
        return v

# 사용
user = User(id=1, name='Alice', email='alice@example.com', age=25)
print(user.json())

# 검증 에러
try:
    invalid_user = User(id=1, name='Bob', email='invalid-email', age=-5)
except ValueError as e:
    print(f"에러: {e}")
```

---

## 핵심 요약

**필수 서드파티 라이브러리**:

1. **requests**: HTTP 클라이언트 (동기)
2. **httpx**: HTTP 클라이언트 (비동기)
3. **pandas**: 데이터 분석
4. **numpy**: 수치 연산
5. **pytest**: 테스팅
6. **black**: 코드 포매터
7. **SQLAlchemy**: ORM
8. **python-dotenv**: 환경 변수
9. **rich**: 터미널 출력
10. **pydantic**: 데이터 검증

**설치**:
```bash
pip install requests pandas numpy pytest black sqlalchemy python-dotenv httpx rich pydantic
```

---

**다음**: [부록 C. 커뮤니티 리소스](부록C-커뮤니티_리소스.md)