# Chapter 23. 패키지 배포 (Packaging & Distribution)

> **Java 개발자를 위한 노트**: Python의 패키징은 Java의 Maven/Gradle + JAR 파일 배포와 유사합니다. `setup.py`/`pyproject.toml`은 `pom.xml`/`build.gradle`에, PyPI는 Maven Central에 대응됩니다.

---

## 1. Python 패키지란?

### 1.1 패키지 구조

**비유**: 패키지는 **도서관의 책**입니다. 다른 사람들이 쉽게 찾아 사용할 수 있도록 체계적으로 정리되어 있습니다.

```python
"""
기본 패키지 구조:

my_package/
├── README.md              # 프로젝트 설명
├── LICENSE                # 라이선스
├── setup.py              # 패키지 설정 (전통 방식)
├── pyproject.toml        # 패키지 설정 (현대 방식)
├── requirements.txt      # 의존성 목록
├── src/
│   └── my_package/       # 실제 코드
│       ├── __init__.py
│       ├── module1.py
│       └── module2.py
└── tests/
    ├── __init__.py
    └── test_module1.py
"""
```

### Java vs Python

**Java (Maven)**:
```xml
<!-- pom.xml -->
<project>
  <groupId>com.example</groupId>
  <artifactId>my-library</artifactId>
  <version>1.0.0</version>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.8.2</version>
    </dependency>
  </dependencies>
</project>
```

**Python (pyproject.toml)**:
```toml
[project]
name = "my-package"
version = "1.0.0"
dependencies = [
    "requests>=2.28.0",
    "pytest>=7.0.0"
]
```

---

## 2. 간단한 패키지 만들기

### 2.1 프로젝트 구조 생성

```python
# 디렉토리 구조
my_calculator/
├── README.md
├── LICENSE
├── pyproject.toml
├── src/
│   └── calculator/
│       ├── __init__.py
│       ├── basic.py
│       └── advanced.py
└── tests/
    ├── __init__.py
    └── test_basic.py
```

### 2.2 코드 작성

```python
# src/calculator/__init__.py
"""간단한 계산기 패키지"""

__version__ = "1.0.0"

from .basic import add, subtract, multiply, divide
from .advanced import power, sqrt

__all__ = ['add', 'subtract', 'multiply', 'divide', 'power', 'sqrt']

# src/calculator/basic.py
"""기본 연산"""

def add(a, b):
    """덧셈"""
    return a + b

def subtract(a, b):
    """뺄셈"""
    return a - b

def multiply(a, b):
    """곱셈"""
    return a * b

def divide(a, b):
    """나눗셈"""
    if b == 0:
        raise ValueError("0으로 나눌 수 없습니다")
    return a / b

# src/calculator/advanced.py
"""고급 연산"""

import math

def power(base, exponent):
    """거듭제곱"""
    return base ** exponent

def sqrt(n):
    """제곱근"""
    if n < 0:
        raise ValueError("음수의 제곱근은 계산할 수 없습니다")
    return math.sqrt(n)
```

### 2.3 README.md 작성

```markdown
# Calculator

간단한 계산기 라이브러리

## 설치

```bash
pip install my-calculator
```

## 사용법

```python
from calculator import add, subtract, power

# 기본 연산
result = add(5, 3)
print(result)  # 8

# 고급 연산
result = power(2, 10)
print(result)  # 1024
```

## 기능

- 기본 연산: 덧셈, 뺄셈, 곱셈, 나눗셈
- 고급 연산: 거듭제곱, 제곱근

## 라이선스

MIT License
```

---

## 3. pyproject.toml - 현대적 방식

### 3.1 기본 설정

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-calculator"
version = "1.0.0"
description = "간단한 계산기 라이브러리"
readme = "README.md"
requires-python = ">=3.8"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "your.email@example.com"}
]
keywords = ["calculator", "math", "arithmetic"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
]

dependencies = [
    "requests>=2.28.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "black>=22.0.0",
    "flake8>=4.0.0",
]

[project.urls]
Homepage = "https://github.com/yourusername/my-calculator"
Documentation = "https://my-calculator.readthedocs.io"
Repository = "https://github.com/yourusername/my-calculator"
```

### 3.2 빌드 및 설치

```bash
# 개발 모드로 설치 (수정 사항 즉시 반영)
pip install -e .

# 또는 Poetry 사용
poetry install

# 패키지 빌드
python -m build

# dist/ 디렉토리에 생성:
# - my_calculator-1.0.0.tar.gz (소스 배포)
# - my_calculator-1.0.0-py3-none-any.whl (바이너리 배포)
```

---

## 4. Poetry - 현대적 패키지 관리

**비유**: Poetry는 Python의 **종합 선물 세트**입니다. 의존성 관리, 가상환경, 빌드, 배포를 모두 처리합니다.

### 4.1 Poetry 설치 및 프로젝트 생성

```bash
# Poetry 설치
curl -sSL https://install.python-poetry.org | python3 -

# 새 프로젝트 생성
poetry new my-calculator

# 생성된 구조:
# my-calculator/
# ├── pyproject.toml
# ├── README.md
# ├── my_calculator/
# │   └── __init__.py
# └── tests/
#     └── __init__.py
```

### 4.2 pyproject.toml (Poetry 버전)

```toml
[tool.poetry]
name = "my-calculator"
version = "1.0.0"
description = "간단한 계산기 라이브러리"
authors = ["Your Name <your.email@example.com>"]
readme = "README.md"
homepage = "https://github.com/yourusername/my-calculator"
repository = "https://github.com/yourusername/my-calculator"
keywords = ["calculator", "math"]
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
]

[tool.poetry.dependencies]
python = "^3.8"
requests = "^2.28.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0.0"
black = "^22.0.0"
flake8 = "^4.0.0"
mypy = "^0.991"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

### 4.3 Poetry 주요 명령어

```bash
# 의존성 추가
poetry add requests
poetry add --group dev pytest

# 의존성 제거
poetry remove requests

# 의존성 설치
poetry install

# 가상환경 활성화
poetry shell

# 스크립트 실행
poetry run python script.py
poetry run pytest

# 패키지 빌드
poetry build

# PyPI에 배포
poetry publish
```

---

## 5. 실전 예제 - 웹 스크래퍼 패키지

### 5.1 프로젝트 구조

```python
web-scraper/
├── README.md
├── LICENSE
├── pyproject.toml
├── src/
│   └── web_scraper/
│       ├── __init__.py
│       ├── scraper.py
│       ├── parser.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   ├── test_scraper.py
│   └── test_parser.py
└── examples/
    └── basic_usage.py
```

### 5.2 코드 구현

```python
# src/web_scraper/__init__.py
"""웹 스크래퍼 라이브러리"""

__version__ = "1.0.0"

from .scraper import Scraper
from .parser import HtmlParser

__all__ = ['Scraper', 'HtmlParser']

# src/web_scraper/scraper.py
"""웹 스크래핑 핵심 기능"""

import requests
from bs4 import BeautifulSoup
from typing import Optional

class Scraper:
    """웹 스크래퍼"""

    def __init__(self, user_agent: Optional[str] = None):
        self.session = requests.Session()
        if user_agent:
            self.session.headers['User-Agent'] = user_agent

    def fetch(self, url: str) -> str:
        """URL에서 HTML 가져오기"""
        response = self.session.get(url)
        response.raise_for_status()
        return response.text

    def fetch_soup(self, url: str) -> BeautifulSoup:
        """BeautifulSoup 객체 반환"""
        html = self.fetch(url)
        return BeautifulSoup(html, 'html.parser')

# src/web_scraper/parser.py
"""HTML 파싱 유틸리티"""

from bs4 import BeautifulSoup
from typing import List, Dict

class HtmlParser:
    """HTML 파서"""

    def __init__(self, html: str):
        self.soup = BeautifulSoup(html, 'html.parser')

    def get_title(self) -> str:
        """페이지 제목"""
        title = self.soup.find('title')
        return title.text if title else ""

    def get_links(self) -> List[Dict[str, str]]:
        """모든 링크"""
        links = []
        for a in self.soup.find_all('a', href=True):
            links.append({
                'text': a.get_text(strip=True),
                'href': a['href']
            })
        return links

    def get_images(self) -> List[str]:
        """모든 이미지 URL"""
        return [img['src'] for img in self.soup.find_all('img', src=True)]

# examples/basic_usage.py
"""사용 예제"""

from web_scraper import Scraper, HtmlParser

def main():
    # 스크래퍼 생성
    scraper = Scraper(user_agent='MyBot 1.0')

    # 페이지 가져오기
    html = scraper.fetch('https://www.python.org')

    # 파싱
    parser = HtmlParser(html)

    print(f"제목: {parser.get_title()}")
    print(f"링크 수: {len(parser.get_links())}")
    print(f"이미지 수: {len(parser.get_images())}")

if __name__ == '__main__':
    main()
```

### 5.3 pyproject.toml

```toml
[tool.poetry]
name = "web-scraper"
version = "1.0.0"
description = "간단한 웹 스크래핑 라이브러리"
authors = ["Your Name <your.email@example.com>"]
readme = "README.md"
license = "MIT"

[tool.poetry.dependencies]
python = "^3.8"
requests = "^2.28.0"
beautifulsoup4 = "^4.11.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0.0"
black = "^22.0.0"
flake8 = "^4.0.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

---

## 6. 가상환경 관리

### 6.1 venv - 표준 가상환경

```bash
# 가상환경 생성
python -m venv venv

# 활성화
# Windows:
venv\Scripts\activate
# Mac/Linux:
source venv/bin/activate

# 의존성 설치
pip install -r requirements.txt

# 비활성화
deactivate

# requirements.txt 생성
pip freeze > requirements.txt
```

### 6.2 Poetry로 가상환경 관리

```bash
# Poetry가 자동으로 가상환경 생성/관리
poetry install

# 가상환경 활성화
poetry shell

# 가상환경 없이 명령 실행
poetry run python script.py

# 가상환경 위치 확인
poetry env info --path

# 가상환경 제거
poetry env remove python3.10
```

---

## 7. PyPI에 배포하기

### 7.1 PyPI 계정 생성

```bash
# 1. https://pypi.org 에서 계정 생성
# 2. API 토큰 생성 (Account Settings > API tokens)
```

### 7.2 배포 준비

```bash
# .pypirc 파일 생성 (홈 디렉토리)
[pypi]
username = __token__
password = pypi-your-api-token-here

# 또는 환경 변수 사용
export POETRY_PYPI_TOKEN_PYPI=pypi-your-api-token-here
```

### 7.3 배포 과정

```bash
# 1. 버전 업데이트
# pyproject.toml에서 version 수정

# 2. 빌드
poetry build

# 3. Test PyPI에 먼저 배포 (선택)
poetry config repositories.testpypi https://test.pypi.org/legacy/
poetry publish -r testpypi

# 4. 실제 PyPI에 배포
poetry publish

# 5. 설치 확인
pip install my-calculator

# 6. 사용
python -c "from calculator import add; print(add(2, 3))"
```

---

## 8. Docker 이미지 만들기

### 8.1 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Poetry 설치
RUN pip install poetry

# 의존성 복사
COPY pyproject.toml poetry.lock ./

# 의존성 설치 (가상환경 생성 안 함)
RUN poetry config virtualenvs.create false \
    && poetry install --no-dev --no-interaction --no-ansi

# 코드 복사
COPY src/ ./src/

# 실행
CMD ["python", "-m", "my_package"]
```

### 8.2 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - PYTHONUNBUFFERED=1
    ports:
      - "8000:8000"
```

### 8.3 빌드 및 실행

```bash
# 이미지 빌드
docker build -t my-calculator:1.0.0 .

# 실행
docker run my-calculator:1.0.0

# Docker Compose 사용
docker-compose up
```

---

## 9. 실전 팁

### 💡 Tip 1: 시맨틱 버저닝

```python
"""
시맨틱 버저닝 (Semantic Versioning):

MAJOR.MINOR.PATCH (예: 1.2.3)

MAJOR: 호환되지 않는 API 변경
MINOR: 하위 호환되는 기능 추가
PATCH: 하위 호환되는 버그 수정

예시:
1.0.0 → 첫 릴리스
1.0.1 → 버그 수정
1.1.0 → 새 기능 추가 (하위 호환)
2.0.0 → 호환 안 되는 변경 (Breaking Change)
"""

# pyproject.toml
[tool.poetry]
version = "1.2.3"
```

### 💡 Tip 2: 의존성 버전 지정

```toml
[tool.poetry.dependencies]
# 정확한 버전
requests = "2.28.0"

# 최소 버전
requests = ">=2.28.0"

# 호환 버전 (^)
requests = "^2.28.0"  # >=2.28.0, <3.0.0

# 틸데 버전 (~)
requests = "~2.28.0"  # >=2.28.0, <2.29.0

# 버전 범위
requests = ">=2.28.0,<3.0.0"

# 와일드카드
requests = "2.28.*"
```

### 💡 Tip 3: 개발 의존성 분리

```toml
[tool.poetry.dependencies]
python = "^3.8"
requests = "^2.28.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0.0"
black = "^22.0.0"
mypy = "^0.991"

[tool.poetry.group.docs.dependencies]
sphinx = "^5.0.0"
sphinx-rtd-theme = "^1.0.0"
```

### 💡 Tip 4: Entry Points

```toml
# CLI 명령 추가
[tool.poetry.scripts]
my-calc = "calculator.cli:main"

# calculator/cli.py
def main():
    import sys
    from calculator import add

    if len(sys.argv) != 3:
        print("Usage: my-calc <num1> <num2>")
        sys.exit(1)

    a = float(sys.argv[1])
    b = float(sys.argv[2])
    result = add(a, b)
    print(f"{a} + {b} = {result}")

# 설치 후 사용:
# $ my-calc 5 3
# 5.0 + 3.0 = 8.0
```

---

## 10. 연습 문제

### 문제 1: 간단한 패키지 만들기
텍스트 처리 유틸리티 패키지를 만들고 Poetry로 관리하세요.

**요구사항**:
- 대소문자 변환
- 공백 제거
- 단어 수 세기
- pyproject.toml 작성
- 테스트 작성

<details>
<summary>정답 보기</summary>

```bash
# 1. 프로젝트 생성
poetry new text-utils
cd text-utils
```

```python
# text_utils/text_utils.py
"""텍스트 처리 유틸리티"""

def to_upper(text: str) -> str:
    """대문자로 변환"""
    return text.upper()

def to_lower(text: str) -> str:
    """소문자로 변환"""
    return text.lower()

def strip_whitespace(text: str) -> str:
    """앞뒤 공백 제거"""
    return text.strip()

def count_words(text: str) -> int:
    """단어 수 세기"""
    return len(text.split())

def reverse(text: str) -> str:
    """문자열 뒤집기"""
    return text[::-1]

# text_utils/__init__.py
"""텍스트 처리 유틸리티 패키지"""

__version__ = "1.0.0"

from .text_utils import (
    to_upper,
    to_lower,
    strip_whitespace,
    count_words,
    reverse
)

__all__ = [
    'to_upper',
    'to_lower',
    'strip_whitespace',
    'count_words',
    'reverse'
]

# tests/test_text_utils.py
import pytest
from text_utils import to_upper, to_lower, count_words, reverse

def test_to_upper():
    assert to_upper("hello") == "HELLO"

def test_to_lower():
    assert to_lower("HELLO") == "hello"

def test_count_words():
    assert count_words("hello world python") == 3

def test_reverse():
    assert reverse("hello") == "olleh"

# pyproject.toml
[tool.poetry]
name = "text-utils"
version = "1.0.0"
description = "텍스트 처리 유틸리티"
authors = ["Your Name <your.email@example.com>"]

[tool.poetry.dependencies]
python = "^3.8"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

```bash
# 테스트 실행
poetry run pytest

# 빌드
poetry build
```
</details>

### 문제 2: CLI 도구 만들기
파일 정보를 출력하는 CLI 도구를 만드세요.

<details>
<summary>정답 보기</summary>

```python
# file_info/cli.py
"""파일 정보 CLI"""

import sys
import os
from pathlib import Path

def get_file_info(file_path: str) -> dict:
    """파일 정보 수집"""
    path = Path(file_path)

    if not path.exists():
        raise FileNotFoundError(f"파일을 찾을 수 없습니다: {file_path}")

    return {
        'name': path.name,
        'size': path.stat().st_size,
        'extension': path.suffix,
        'is_file': path.is_file(),
        'is_dir': path.is_dir(),
        'absolute_path': str(path.absolute())
    }

def main():
    """CLI 진입점"""
    if len(sys.argv) != 2:
        print("Usage: file-info <file_path>")
        sys.exit(1)

    file_path = sys.argv[1]

    try:
        info = get_file_info(file_path)

        print(f"📄 파일 정보:")
        print(f"  이름: {info['name']}")
        print(f"  크기: {info['size']} bytes")
        print(f"  확장자: {info['extension']}")
        print(f"  타입: {'파일' if info['is_file'] else '디렉토리'}")
        print(f"  경로: {info['absolute_path']}")

    except FileNotFoundError as e:
        print(f"❌ 에러: {e}")
        sys.exit(1)

if __name__ == '__main__':
    main()

# pyproject.toml
[tool.poetry]
name = "file-info"
version = "1.0.0"
description = "파일 정보 CLI 도구"
authors = ["Your Name <your.email@example.com>"]

[tool.poetry.dependencies]
python = "^3.8"

[tool.poetry.scripts]
file-info = "file_info.cli:main"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

```bash
# 설치
poetry install

# 사용
file-info README.md

# 출력:
# 📄 파일 정보:
#   이름: README.md
#   크기: 1234 bytes
#   확장자: .md
#   타입: 파일
#   경로: /path/to/README.md
```
</details>

---

## 핵심 요약

1. **패키지 구조**
   ```
   my-package/
   ├── pyproject.toml
   ├── README.md
   ├── src/my_package/
   └── tests/
   ```

2. **Poetry 주요 명령**
   ```bash
   poetry new my-package
   poetry add requests
   poetry install
   poetry build
   poetry publish
   ```

3. **pyproject.toml 기본**
   ```toml
   [tool.poetry]
   name = "my-package"
   version = "1.0.0"

   [tool.poetry.dependencies]
   python = "^3.8"
   ```

4. **가상환경**
   - venv: 표준 라이브러리
   - Poetry: 자동 관리 (권장)

5. **배포 과정**
   1. 코드 작성
   2. 테스트
   3. 버전 업데이트
   4. 빌드
   5. PyPI 배포

6. **Java와 비교**
   - pyproject.toml ≈ pom.xml/build.gradle
   - Poetry ≈ Maven/Gradle
   - PyPI ≈ Maven Central

---

**다음 장**: [Chapter 24. 성능 최적화](chapter24-optimization.md)에서는 Python 코드의 성능을 개선하는 방법을 배웁니다.