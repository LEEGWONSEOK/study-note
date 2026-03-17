# Chapter 14. 컨텍스트 매니저와 파일 I/O (Context Managers & File I/O)

## 14.1 컨텍스트 매니저 심화

> 🚪 **비유**: 컨텍스트 매니저는 **자동문**입니다.
> - 들어갈 때: 자동 열림 (setup)
> - 나올 때: 자동 닫힘 (cleanup)
> - 에러 발생해도: 반드시 닫힘 (보장)

---

### with문의 동작 원리

```python
# with문 없이
file = open("data.txt", "r")
try:
    content = file.read()
    print(content)
finally:
    file.close()  # 반드시 닫기

# with문 사용 (간단하고 안전)
with open("data.txt", "r") as file:
    content = file.read()
    print(content)
# 자동으로 file.close() 호출
```

---

## 14.2 파일 기본 I/O

### 파일 열기 모드

| 모드 | 설명 | 없으면 | 있으면 |
|-----|------|-------|-------|
| `r` | 읽기 (기본) | 에러 | 읽기 |
| `w` | 쓰기 | 생성 | 덮어쓰기 |
| `a` | 추가 | 생성 | 끝에 추가 |
| `x` | 배타적 생성 | 생성 | 에러 |
| `r+` | 읽기+쓰기 | 에러 | 읽기/쓰기 |
| `w+` | 읽기+쓰기 | 생성 | 덮어쓰기 |

추가 옵션:
- `b`: 바이너리 모드 (예: `rb`, `wb`)
- `t`: 텍스트 모드 (기본)

---

### 텍스트 파일 읽기

```python
# 1. 전체 읽기
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# 2. 한 줄씩 읽기
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())  # 줄바꿈 제거

# 3. 모든 줄을 리스트로
with open("data.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()
    print(lines)

# 4. 특정 개수만 읽기
with open("data.txt", "r", encoding="utf-8") as f:
    first_line = f.readline()
    second_line = f.readline()
```

---

### 텍스트 파일 쓰기

```python
# 1. 전체 쓰기 (덮어쓰기)
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello, World!\n")
    f.write("두 번째 줄\n")

# 2. 여러 줄 쓰기
lines = ["첫 번째 줄\n", "두 번째 줄\n", "세 번째 줄\n"]
with open("output.txt", "w", encoding="utf-8") as f:
    f.writelines(lines)

# 3. 추가 모드
with open("output.txt", "a", encoding="utf-8") as f:
    f.write("추가된 줄\n")

# 4. print로 파일에 쓰기
with open("output.txt", "w", encoding="utf-8") as f:
    print("Hello", file=f)
    print("World", file=f)
```

---

### 바이너리 파일

```python
# 이미지 복사
with open("image.jpg", "rb") as src:
    with open("image_copy.jpg", "wb") as dst:
        data = src.read()
        dst.write(data)

# 또는 한 줄로
with open("image.jpg", "rb") as src, open("image_copy.jpg", "wb") as dst:
    dst.write(src.read())
```

---

## 14.3 pathlib - 모던 경로 처리

> 🗺️ **비유**: pathlib는 **스마트 네비게이션**
> - os.path: 구식 종이 지도
> - pathlib: 최신 GPS

```python
from pathlib import Path

# 경로 생성
path = Path("data/users/alice.txt")
print(path)  # data/users/alice.txt

# 현재 디렉토리
cwd = Path.cwd()
print(cwd)  # /Users/alice/project

# 홈 디렉토리
home = Path.home()
print(home)  # /Users/alice

# 경로 결합 (/ 연산자!)
data_dir = Path("data")
user_file = data_dir / "users" / "alice.txt"
print(user_file)  # data/users/alice.txt
```

---

### 경로 속성

```python
path = Path("/Users/alice/project/data/file.txt")

print(path.name)       # file.txt (파일명)
print(path.stem)       # file (확장자 제외)
print(path.suffix)     # .txt (확장자)
print(path.parent)     # /Users/alice/project/data (부모 디렉토리)
print(path.parents[0]) # /Users/alice/project/data
print(path.parents[1]) # /Users/alice/project
print(path.anchor)     # / (루트)

# 절대 경로
rel_path = Path("data/file.txt")
abs_path = rel_path.absolute()
print(abs_path)  # /Users/alice/project/data/file.txt
```

---

### 파일/디렉토리 조작

```python
from pathlib import Path

path = Path("data/users/alice.txt")

# 1. 존재 여부
print(path.exists())      # False
print(path.is_file())     # False
print(path.is_dir())      # False

# 2. 디렉토리 생성
path.parent.mkdir(parents=True, exist_ok=True)
# parents=True: 중간 디렉토리도 생성
# exist_ok=True: 이미 있어도 에러 안 남

# 3. 파일 쓰기
path.write_text("Hello, Alice!", encoding="utf-8")

# 4. 파일 읽기
content = path.read_text(encoding="utf-8")
print(content)  # Hello, Alice!

# 5. 바이너리
path.write_bytes(b"Binary data")
data = path.read_bytes()

# 6. 파일 삭제
path.unlink(missing_ok=True)  # missing_ok: 없어도 에러 안 남

# 7. 디렉토리 삭제
path.parent.rmdir()  # 비어있어야 함
```

---

### 파일 검색 (glob)

```python
from pathlib import Path

# 현재 디렉토리의 모든 .py 파일
for py_file in Path(".").glob("*.py"):
    print(py_file)

# 재귀적으로 모든 .py 파일 검색
for py_file in Path(".").rglob("*.py"):
    print(py_file)

# 패턴 매칭
for file in Path("data").glob("**/*.txt"):
    print(file)

# 디렉토리만
for dir in Path(".").glob("*/"):
    if dir.is_dir():
        print(dir)
```

---

### 실전 예제: 파일 정리

```python
from pathlib import Path
import shutil

def organize_files(source_dir, target_dir):
    """파일을 확장자별로 정리"""
    source = Path(source_dir)
    target = Path(target_dir)

    for file in source.rglob("*"):
        if file.is_file():
            # 확장자별 폴더
            ext = file.suffix[1:] if file.suffix else "no_ext"
            dest_dir = target / ext

            # 폴더 생성
            dest_dir.mkdir(parents=True, exist_ok=True)

            # 파일 이동
            dest_file = dest_dir / file.name
            shutil.move(str(file), str(dest_file))
            print(f"{file.name} → {ext}/")

# 사용
# organize_files("downloads", "organized")
```

---

## 14.4 JSON 처리

> 📦 **비유**: JSON은 **보편적인 택배 상자**
> - 어디서나 사용 가능
> - 표준 형식

```python
import json

# Python → JSON
data = {
    "name": "Alice",
    "age": 25,
    "skills": ["Python", "Java", "JavaScript"],
    "is_active": True,
    "score": 95.5
}

# 1. 문자열로 변환
json_str = json.dumps(data, ensure_ascii=False, indent=2)
print(json_str)
# {
#   "name": "Alice",
#   "age": 25,
#   "skills": [
#     "Python",
#     "Java",
#     "JavaScript"
#   ],
#   "is_active": true,
#   "score": 95.5
# }

# 2. 파일에 저장
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# JSON → Python
# 1. 문자열에서 읽기
parsed = json.loads(json_str)
print(parsed["name"])  # Alice

# 2. 파일에서 읽기
with open("data.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
    print(loaded["skills"])  # ['Python', 'Java', 'JavaScript']
```

---

### 커스텀 인코더/디코더

```python
import json
from datetime import datetime

class DateTimeEncoder(json.JSONEncoder):
    """datetime 인코더"""
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)

# 사용
data = {
    "name": "Alice",
    "created_at": datetime.now()
}

json_str = json.dumps(data, cls=DateTimeEncoder, indent=2)
print(json_str)
# {
#   "name": "Alice",
#   "created_at": "2024-01-15T14:30:00.123456"
# }
```

---

## 14.5 CSV 처리

```python
import csv

# 쓰기
data = [
    ["name", "age", "city"],
    ["Alice", 25, "Seoul"],
    ["Bob", 30, "Busan"],
    ["Charlie", 28, "Incheon"]
]

with open("users.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerows(data)

# 읽기
with open("users.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)
# ['name', 'age', 'city']
# ['Alice', '25', 'Seoul']
# ['Bob', '30', 'Busan']
# ['Charlie', '28', 'Incheon']

# DictReader (딕셔너리로)
with open("users.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['name']} ({row['age']}세) - {row['city']}")
# Alice (25세) - Seoul
# Bob (30세) - Busan
# Charlie (28세) - Incheon

# DictWriter
users = [
    {"name": "Alice", "age": 25, "city": "Seoul"},
    {"name": "Bob", "age": 30, "city": "Busan"}
]

with open("users.csv", "w", newline="", encoding="utf-8") as f:
    fieldnames = ["name", "age", "city"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)

    writer.writeheader()  # 헤더 쓰기
    writer.writerows(users)
```

---

## 14.6 Pickle - Python 객체 직렬화

> 🧊 **비유**: Pickle은 **냉동실**
> - Python 객체를 그대로 저장
> - 나중에 꺼내서 사용

```python
import pickle

# 복잡한 객체
data = {
    "users": [
        {"name": "Alice", "scores": [90, 85, 92]},
        {"name": "Bob", "scores": [88, 91, 87]}
    ],
    "metadata": {
        "version": 1.0,
        "created": "2024-01-15"
    }
}

# 저장
with open("data.pickle", "wb") as f:
    pickle.dump(data, f)

# 로드
with open("data.pickle", "rb") as f:
    loaded = pickle.load(f)
    print(loaded["users"][0]["name"])  # Alice
```

> ⚠️ **주의**: Pickle은 보안 위험! 신뢰할 수 없는 소스의 파일은 로드하지 마세요.

---

## 14.7 임시 파일

```python
import tempfile

# 1. 임시 파일 (자동 삭제)
with tempfile.TemporaryFile(mode="w+", encoding="utf-8") as f:
    f.write("Temporary data")
    f.seek(0)
    print(f.read())
# 여기서 파일 자동 삭제

# 2. 이름이 있는 임시 파일
with tempfile.NamedTemporaryFile(mode="w+", delete=False, suffix=".txt") as f:
    print(f.name)  # /tmp/tmpxxx.txt
    f.write("Data")

# 3. 임시 디렉토리
with tempfile.TemporaryDirectory() as tmpdir:
    print(tmpdir)  # /tmp/tmpxxx
    # 작업...
# 여기서 디렉토리와 내용물 자동 삭제
```

---

## 14.8 실전 예제

### 예제 1: 로그 파일 관리

```python
from pathlib import Path
from datetime import datetime
import gzip

class LogManager:
    """로그 파일 관리"""

    def __init__(self, log_dir="logs", max_size_mb=10):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self.max_size = max_size_mb * 1024 * 1024
        self.current_file = None

    def _get_log_file(self):
        """현재 로그 파일 경로"""
        date = datetime.now().strftime("%Y-%m-%d")
        return self.log_dir / f"app_{date}.log"

    def _rotate_if_needed(self, log_file):
        """파일 크기가 크면 압축 후 새 파일 시작"""
        if log_file.exists() and log_file.stat().st_size > self.max_size:
            # 압축
            timestamp = datetime.now().strftime("%H%M%S")
            gz_file = self.log_dir / f"{log_file.stem}_{timestamp}.log.gz"

            with log_file.open("rb") as f_in:
                with gzip.open(gz_file, "wb") as f_out:
                    f_out.writelines(f_in)

            # 원본 삭제
            log_file.unlink()

    def log(self, level, message):
        """로그 작성"""
        log_file = self._get_log_file()
        self._rotate_if_needed(log_file)

        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        log_line = f"[{timestamp}] [{level}] {message}\n"

        with log_file.open("a", encoding="utf-8") as f:
            f.write(log_line)

    def info(self, message):
        self.log("INFO", message)

    def error(self, message):
        self.log("ERROR", message)

# 사용
logger = LogManager()
logger.info("애플리케이션 시작")
logger.error("에러 발생")
```

---

### 예제 2: 설정 파일 관리

```python
from pathlib import Path
import json

class Config:
    """설정 파일 관리"""

    def __init__(self, config_file="config.json"):
        self.config_file = Path(config_file)
        self.data = self._load()

    def _load(self):
        """설정 로드"""
        if self.config_file.exists():
            return json.loads(self.config_file.read_text(encoding="utf-8"))
        return {}

    def get(self, key, default=None):
        """값 가져오기 (중첩된 키 지원)"""
        keys = key.split(".")
        value = self.data

        for k in keys:
            if isinstance(value, dict):
                value = value.get(k)
            else:
                return default

        return value if value is not None else default

    def set(self, key, value):
        """값 설정 (중첩된 키 지원)"""
        keys = key.split(".")
        data = self.data

        for k in keys[:-1]:
            if k not in data:
                data[k] = {}
            data = data[k]

        data[keys[-1]] = value

    def save(self):
        """설정 저장"""
        self.config_file.write_text(
            json.dumps(self.data, ensure_ascii=False, indent=2),
            encoding="utf-8"
        )

# 사용
config = Config()

# 설정
config.set("app.name", "MyApp")
config.set("app.version", "1.0.0")
config.set("database.host", "localhost")
config.set("database.port", 5432)

# 저장
config.save()

# 읽기
print(config.get("app.name"))  # MyApp
print(config.get("database.host"))  # localhost
print(config.get("unknown", "default"))  # default
```

---

### 예제 3: 데이터 백업

```python
from pathlib import Path
import shutil
from datetime import datetime
import gzip

class BackupManager:
    """데이터 백업 관리"""

    def __init__(self, backup_dir="backups"):
        self.backup_dir = Path(backup_dir)
        self.backup_dir.mkdir(parents=True, exist_ok=True)

    def backup(self, source_path, compress=True):
        """백업 생성"""
        source = Path(source_path)

        if not source.exists():
            raise FileNotFoundError(f"{source} 파일을 찾을 수 없습니다")

        # 백업 파일명
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_name = f"{source.stem}_{timestamp}{source.suffix}"

        if compress:
            backup_file = self.backup_dir / f"{backup_name}.gz"
            with source.open("rb") as f_in:
                with gzip.open(backup_file, "wb") as f_out:
                    shutil.copyfileobj(f_in, f_out)
        else:
            backup_file = self.backup_dir / backup_name
            shutil.copy2(source, backup_file)

        return backup_file

    def list_backups(self, pattern="*"):
        """백업 목록"""
        backups = sorted(
            self.backup_dir.glob(pattern),
            key=lambda p: p.stat().st_mtime,
            reverse=True
        )
        return backups

    def restore(self, backup_file, target_path):
        """백업 복원"""
        backup = Path(backup_file)
        target = Path(target_path)

        if backup.suffix == ".gz":
            with gzip.open(backup, "rb") as f_in:
                with target.open("wb") as f_out:
                    shutil.copyfileobj(f_in, f_out)
        else:
            shutil.copy2(backup, target)

        return target

    def cleanup(self, keep_count=5):
        """오래된 백업 삭제"""
        backups = self.list_backups()

        for backup in backups[keep_count:]:
            backup.unlink()
            print(f"삭제: {backup.name}")

# 사용
backup_mgr = BackupManager()

# 백업
backup_file = backup_mgr.backup("data.json", compress=True)
print(f"백업 생성: {backup_file}")

# 목록
print("\n백업 목록:")
for backup in backup_mgr.list_backups():
    size = backup.stat().st_size
    print(f"  {backup.name} ({size:,} bytes)")

# 복원
# backup_mgr.restore(backup_file, "data_restored.json")

# 정리 (최근 5개만 유지)
# backup_mgr.cleanup(keep_count=5)
```

---

## 실전 팁

### 💡 Tip 1: 항상 with 사용

```python
# ❌ 나쁜 예
f = open("file.txt", "r")
content = f.read()
f.close()  # 에러 발생 시 호출 안 됨!

# ✅ 좋은 예
with open("file.txt", "r") as f:
    content = f.read()
# 자동으로 닫힘
```

---

### 💡 Tip 2: pathlib 사용

```python
# ❌ 구식 (os.path)
import os
path = os.path.join("data", "users", "alice.txt")
if os.path.exists(path):
    with open(path, "r") as f:
        content = f.read()

# ✅ 모던 (pathlib)
from pathlib import Path
path = Path("data") / "users" / "alice.txt"
if path.exists():
    content = path.read_text()
```

---

### 💡 Tip 3: 인코딩 명시

```python
# ✅ 항상 인코딩 명시
with open("file.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

---

## 핵심 요약

### 꼭 기억할 것

1. **with문**
   ```python
   with open("file.txt") as f:
       content = f.read()
   ```

2. **pathlib**
   ```python
   from pathlib import Path
   path = Path("data") / "file.txt"
   content = path.read_text()
   ```

3. **JSON**
   ```python
   import json
   data = json.loads(json_str)
   json.dump(data, f)
   ```

4. **파일 모드**
   - `r`: 읽기
   - `w`: 쓰기 (덮어쓰기)
   - `a`: 추가

5. **인코딩**
   - 항상 `encoding="utf-8"` 명시

---

## Part 5 완료!

Part 5 (고급 기능)을 마쳤습니다:
- ✅ Chapter 12: 데코레이터
- ✅ Chapter 13: 제너레이터와 이터레이터
- ✅ Chapter 14: 컨텍스트 매니저와 파일 I/O

현재 진행률: **14/32 챕터 완료 (43.75%)**

---

## 다음 챕터 예고

Chapter 15에서는 **함수형 프로그래밍**을 다룹니다:
- map, filter, reduce
- 람다 표현식
- partial, functools
- 순수 함수

---

[다음: Chapter 15. 함수형 프로그래밍 →](../part6-functional/chapter15-functional.md)