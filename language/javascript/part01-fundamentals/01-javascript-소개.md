# Chapter 1. JavaScript 소개

## 1.1 JavaScript란?

**JavaScript**는 1995년 Brendan Eich가 Netscape에서 개발한 **웹 프로그래밍 언어**입니다.

> 🌐 **놀라운 사실**: JavaScript는 단 **10일** 만에 만들어졌습니다!
> 처음 이름은 "Mocha" → "LiveScript" → "JavaScript"로 변경되었습니다.

### 특징

> 💡 **핵심 철학**: "한 번 작성하고, 어디서나 실행" (Write Once, Run Anywhere)

1. **웹의 언어**
   - 모든 브라우저에서 실행
   - HTML, CSS와 함께 웹의 3대 기술
   - 프론트엔드와 백엔드 모두 가능 (Node.js)

   **비유**: JavaScript는 **웹의 공용어**입니다.
   - 영어가 세계 공용어이듯
   - JavaScript는 브라우저의 유일한 공식 언어

2. **동적 타입 (Dynamic Typing)**
   - 변수 선언 시 타입 명시 불필요
   - 런타임에 타입 결정

   **비유**:
   - Java: 박스에 "정수만", "문자열만" 미리 표시
   - JavaScript: 박스에 뭐든 넣고 나중에 확인

3. **인터프리터 언어**
   - 컴파일 불필요
   - 브라우저나 Node.js가 바로 실행

   **비유**:
   - Java: 한국어 → 번역서 제작 → 읽기
   - JavaScript: 한국어 → 동시통역 → 바로 이해

4. **프로토타입 기반**
   - 클래스 없이도 객체 생성 가능 (ES5)
   - ES6부터 class 문법 지원

   **비유**: **레고 블록**처럼 기존 객체를 복제하고 확장

5. **이벤트 기반**
   - 사용자 상호작용에 반응
   - 비동기 처리 (async/await, Promise)

   **비유**: **리모컨**처럼 버튼 누르면 반응

---

## 1.2 Java vs JavaScript 비교

### ⚠️ 중요: Java ≠ JavaScript

> **Java와 JavaScript는 완전히 다른 언어입니다!**
>
> 비유: "Ham(햄)"과 "Hamster(햄스터)"가 다르듯이!

| 특성 | Java | JavaScript |
|------|------|------------|
| 타입 시스템 | 정적 타입 (Static) | 동적 타입 (Dynamic) |
| 실행 환경 | JVM | 브라우저, Node.js |
| 컴파일 | 컴파일 필요 (.class) | 인터프리터 실행 |
| 상속 | 클래스 기반 | 프로토타입 기반 |
| 스레드 | 멀티스레딩 지원 | 싱글 스레드 (이벤트 루프) |
| 주 용도 | Enterprise, Android | 웹 프론트엔드/백엔드 |
| 타입 표기 | 필수 | 선택 (TypeScript) |
| null 처리 | null | null, undefined |

---

### 코드 비교: Hello World

**Java**:
```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

**JavaScript**:
```javascript
console.log("Hello, World!");
```

> 💪 **왜 이렇게 짧을까?**
> - Java: 모든 코드는 클래스 안에 있어야 함
> - JavaScript: 함수나 클래스 없이 바로 실행 가능

---

### 코드 비교: 변수 선언

**Java**:
```java
// 타입 명시 필수
int age = 25;
String name = "홍길동";
List<String> hobbies = new ArrayList<>();
hobbies.add("축구");

// 타입 변경 불가
age = "스물다섯";  // ❌ 컴파일 에러!
```

**JavaScript**:
```javascript
// 타입 명시 불필요
let age = 25;
let name = "홍길동";
let hobbies = [];
hobbies.push("축구");

// 타입 변경 가능
age = "스물다섯";  // ✅ 가능! (권장하진 않음)
```

---

### 코드 비교: 배열 필터링

**Java**:
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
List<Integer> even = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
System.out.println(even);  // [2, 4]
```

**JavaScript**:
```javascript
const numbers = [1, 2, 3, 4, 5];
const even = numbers.filter(n => n % 2 === 0);
console.log(even);  // [2, 4]
```

> 🎯 **차이점**:
> - Java: Stream API 필요, collect로 수집
> - JavaScript: 배열 메서드가 내장, 바로 반환

---

## 1.3 JavaScript 역사

### 탄생 (1995년)
- Netscape Navigator 브라우저에 탑재
- 웹 페이지에 동적 기능 추가 목적

### 표준화 (1997년)
- ECMAScript (ES)로 표준화
- ECMA International에서 관리

### 주요 버전

| 버전 | 연도 | 주요 기능 |
|------|------|-----------|
| ES3 | 1999 | 정규표현식, try/catch |
| ES5 | 2009 | JSON, strict mode, Array 메서드 |
| **ES6 (ES2015)** | 2015 | let/const, 화살표 함수, 클래스, Promise, 모듈 |
| ES7 (ES2016) | 2016 | Array.includes(), 지수 연산자 |
| ES8 (ES2017) | 2017 | async/await, Object.entries() |
| ES9 (ES2018) | 2018 | Rest/Spread, Promise.finally() |
| ES10 (ES2019) | 2019 | Array.flat(), Object.fromEntries() |
| ES11 (ES2020) | 2020 | Optional Chaining (?.), Nullish (??) |
| ES12 (ES2021) | 2021 | Logical Assignment |
| ES13 (ES2022) | 2022 | Top-level await, class fields |
| ES14 (ES2023) | 2023 | Array.findLast() |

> 🚀 **ES6가 가장 중요합니다!**
> - 현대 JavaScript의 기준
> - 대부분의 모던 문법 추가
> - **이 가이드는 ES6+ 기준으로 작성됩니다**

---

## 1.4 JavaScript 실행 환경

### 1. 브라우저 (Frontend)

모든 브라우저가 JavaScript 엔진 내장:
- **Chrome**: V8 엔진 (가장 빠름)
- **Firefox**: SpiderMonkey
- **Safari**: JavaScriptCore
- **Edge**: V8 (Chromium 기반)

**사용 예시**:
```javascript
// DOM 조작
document.getElementById('button').addEventListener('click', () => {
  alert('버튼 클릭!');
});

// 브라우저 API
console.log(window.location.href);
localStorage.setItem('name', '홍길동');
```

---

### 2. Node.js (Backend)

- Chrome V8 엔진 기반
- 서버 사이드 JavaScript
- npm (Node Package Manager) 포함

**사용 예시**:
```javascript
// 파일 시스템
const fs = require('fs');
fs.readFile('data.txt', 'utf8', (err, data) => {
  console.log(data);
});

// HTTP 서버
const http = require('http');
http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('Hello World!');
}).listen(3000);
```

---

## 1.5 개발 환경 설정

### 1. Node.js 설치

**macOS/Linux**:
```bash
# nvm 설치 (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Node.js LTS 설치
nvm install --lts
nvm use --lts

# 버전 확인
node --version  # v20.x.x
npm --version   # v10.x.x
```

**Windows**:
- [nodejs.org](https://nodejs.org) 에서 LTS 버전 다운로드
- 설치 후 CMD에서 확인

---

### 2. VSCode 설치 및 확장

**필수 확장 프로그램**:
- **ESLint**: 코드 오류 검사
- **Prettier**: 코드 포맷팅
- **JavaScript (ES6) code snippets**: 코드 스니펫
- **Path Intellisense**: 경로 자동완성
- **npm Intellisense**: npm 패키지 자동완성

**설정 (settings.json)**:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

---

### 3. 첫 JavaScript 프로그램

**브라우저에서 실행**:

1. `index.html` 생성:
```html
<!DOCTYPE html>
<html>
<head>
  <title>첫 JavaScript</title>
</head>
<body>
  <h1>JavaScript 시작!</h1>
  <script src="app.js"></script>
</body>
</html>
```

2. `app.js` 생성:
```javascript
console.log("Hello, JavaScript!");
alert("환영합니다!");
```

3. 브라우저로 `index.html` 열기
4. F12 → Console 탭에서 출력 확인

---

**Node.js에서 실행**:

1. `hello.js` 생성:
```javascript
console.log("Hello, Node.js!");
console.log("현재 시간:", new Date());
```

2. 터미널에서 실행:
```bash
node hello.js
```

---

## 1.6 Java 개발자가 알아야 할 JavaScript의 특징

### 1. 타입이 자유롭다 (하지만 위험할 수 있음)

**Java**:
```java
int sum(int a, int b) {
    return a + b;
}
sum(1, 2);     // 3
sum(1, "2");   // ❌ 컴파일 에러
```

**JavaScript**:
```javascript
function sum(a, b) {
  return a + b;
}
sum(1, 2);     // 3
sum(1, "2");   // "12" ⚠️ 문자열 연결됨!
```

> 💡 **해결책**: TypeScript 사용 (Part 7에서 학습)

---

### 2. 함수가 일급 객체 (First-class)

**Java** (제한적):
```java
// Java 8+ 람다
Function<Integer, Integer> double = x -> x * 2;
```

**JavaScript** (자유로움):
```javascript
// 함수를 변수에 저장
const double = x => x * 2;

// 함수를 인자로 전달
[1, 2, 3].map(double);  // [2, 4, 6]

// 함수를 반환
function makeMultiplier(factor) {
  return x => x * factor;
}
const triple = makeMultiplier(3);
triple(5);  // 15
```

---

### 3. 비동기가 기본

**Java**:
```java
// 동기 방식 (블로킹)
String data = fetchData();  // 완료될 때까지 대기
System.out.println(data);
```

**JavaScript**:
```javascript
// 비동기 방식 (논블로킹)
fetchData()
  .then(data => console.log(data))
  .catch(err => console.error(err));

// 또는 async/await
async function getData() {
  try {
    const data = await fetchData();
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
```

---

### 4. 프로토타입 vs 클래스

**Java**:
```java
// 클래스만 가능
class Person {
    private String name;
    public Person(String name) { this.name = name; }
}
```

**JavaScript**:
```javascript
// ES6 이전: 프로토타입
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function() {
  console.log(`안녕, ${this.name}`);
};

// ES6+: 클래스 (내부는 프로토타입)
class Person {
  constructor(name) {
    this.name = name;
  }
  greet() {
    console.log(`안녕, ${this.name}`);
  }
}
```

---

## 📝 핵심 요약

1. **JavaScript ≠ Java**: 완전히 다른 언어
2. **동적 타입**: 유연하지만 주의 필요
3. **ES6+ 기준**: 현대 JavaScript는 ES2015부터
4. **실행 환경**: 브라우저(Frontend) + Node.js(Backend)
5. **비동기 중심**: Promise, async/await가 핵심

---

## 🎯 다음 단계

[Chapter 2. 기본 문법 →](02-기본-문법.md)

---

## 💪 실습 과제

1. Node.js 설치 및 버전 확인
2. VSCode 설치 및 확장 프로그램 설치
3. `hello.js` 작성 및 실행
4. 브라우저 콘솔에서 JavaScript 실행해보기

---

**작성일**: 2024년
**JavaScript 버전**: ES2015+ (ES6+)
**대상**: Java Spring Backend 개발자