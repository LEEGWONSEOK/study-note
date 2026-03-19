# Chapter 20. TypeScript 시작하기

## 20.1 TypeScript란?

> 🛡️ **비유**: TypeScript는 JavaScript에 **안전벨트**를 추가한 것입니다.
> - JavaScript: 자유롭지만 위험할 수 있음
> - TypeScript: 타입 체크로 오류 사전 방지

**TypeScript**: Microsoft가 개발한 **JavaScript의 상위집합(Superset)**
- JavaScript + 정적 타입 시스템
- 컴파일하면 순수 JavaScript로 변환
- 모든 JavaScript 코드는 유효한 TypeScript 코드

### 특징

1. **정적 타입 (Static Typing)**
   - 컴파일 시 타입 검사
   - 런타임 오류를 컴파일 타임에 발견

2. **점진적 도입 가능**
   - 기존 JavaScript 프로젝트에 단계적 적용
   - `.js` → `.ts` 파일로 변환

3. **강력한 IDE 지원**
   - 자동완성
   - 리팩토링
   - 타입 추론

4. **최신 JavaScript 기능**
   - ES6+ 기능 사용 가능
   - 구버전 브라우저용으로 컴파일

---

## 20.2 왜 TypeScript를 사용할까?

### Java 개발자 관점

**Java**:
```java
// 컴파일 시 타입 체크
public int add(int a, int b) {
    return a + b;
}

add(1, 2);     // ✅
add(1, "2");   // ❌ 컴파일 에러
```

**JavaScript**:
```javascript
// 타입 체크 없음
function add(a, b) {
  return a + b;
}

add(1, 2);     // 3 ✅
add(1, "2");   // "12" ⚠️ 런타임에 문제 발생!
```

**TypeScript**:
```typescript
// 컴파일 시 타입 체크
function add(a: number, b: number): number {
  return a + b;
}

add(1, 2);     // 3 ✅
add(1, "2");   // ❌ 컴파일 에러: Argument of type 'string' is not assignable to parameter of type 'number'
```

---

### TypeScript의 장점

**1. 버그 조기 발견**
```typescript
// JavaScript: 런타임에 오류 발견
const user = { name: "Alice", age: 25 };
console.log(user.email);  // undefined (오류 아님!)

// TypeScript: 컴파일 시 오류 발견
interface User {
  name: string;
  age: number;
}

const user: User = { name: "Alice", age: 25 };
console.log(user.email);  // ❌ Property 'email' does not exist on type 'User'
```

**2. 코드 가독성 향상**
```typescript
// JavaScript: 타입을 알 수 없음
function processUser(user) {
  // user가 뭘 가지고 있지?
}

// TypeScript: 타입이 명확
interface User {
  id: number;
  name: string;
  email: string;
}

function processUser(user: User) {
  // user.id, user.name, user.email 자동완성!
}
```

**3. 리팩토링 안전성**
```typescript
// 함수 시그니처 변경 시 모든 호출 위치에서 에러 표시
function getUser(id: number): User {
  // ...
}

// id를 string으로 변경하면
function getUser(id: string): User {  // 타입 변경
  // ...
}

// 모든 호출 위치에서 컴파일 에러 발생!
getUser(123);  // ❌ 에러 표시
```

**4. 팀 협업 효율**
- API 계약 명확
- 문서 역할
- 코드 리뷰 시간 단축

---

## 20.3 TypeScript 설치 및 설정

### 1. Node.js 프로젝트 초기화

```bash
# 프로젝트 디렉토리 생성
mkdir my-typescript-project
cd my-typescript-project

# package.json 생성
npm init -y

# TypeScript 설치
npm install --save-dev typescript

# tsconfig.json 생성
npx tsc --init
```

---

### 2. tsconfig.json 설정

**기본 설정**:
```json
{
  "compilerOptions": {
    /* 언어 버전 */
    "target": "ES2020",                    // 컴파일 대상 JavaScript 버전
    "module": "commonjs",                  // 모듈 시스템
    "lib": ["ES2020", "DOM"],              // 사용할 라이브러리

    /* 엄격한 타입 체크 */
    "strict": true,                        // 모든 엄격한 타입 체크 활성화
    "noImplicitAny": true,                 // any 타입 사용 시 경고
    "strictNullChecks": true,              // null/undefined 엄격 체크
    "strictFunctionTypes": true,           // 함수 타입 엄격 체크

    /* 모듈 해석 */
    "moduleResolution": "node",            // Node.js 방식 모듈 해석
    "esModuleInterop": true,               // CommonJS/ES 모듈 호환성
    "resolveJsonModule": true,             // JSON 파일 import 허용

    /* 출력 */
    "outDir": "./dist",                    // 컴파일된 파일 출력 디렉토리
    "rootDir": "./src",                    // 소스 파일 디렉토리
    "sourceMap": true,                     // .map 파일 생성 (디버깅용)

    /* 추가 체크 */
    "noUnusedLocals": true,                // 사용하지 않는 지역 변수 경고
    "noUnusedParameters": true,            // 사용하지 않는 매개변수 경고
    "noImplicitReturns": true,             // 함수의 모든 경로에서 반환값 필요

    /* 실험적 기능 */
    "experimentalDecorators": true,        // 데코레이터 지원
    "emitDecoratorMetadata": true          // 데코레이터 메타데이터 생성
  },
  "include": ["src/**/*"],                 // 컴파일할 파일 패턴
  "exclude": ["node_modules", "dist"]      // 제외할 디렉토리
}
```

---

### 3. 첫 TypeScript 프로그램

**디렉토리 구조**:
```
my-typescript-project/
├── src/
│   └── index.ts
├── dist/
├── package.json
└── tsconfig.json
```

**src/index.ts**:
```typescript
// 타입 선언
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// 사용
const message: string = greet("TypeScript");
console.log(message);

// 타입 에러 예제
// const error = greet(123);  // ❌ Argument of type 'number' is not assignable to parameter of type 'string'
```

**컴파일 및 실행**:
```bash
# TypeScript → JavaScript 컴파일
npx tsc

# 컴파일된 JavaScript 실행
node dist/index.js
# Hello, TypeScript!
```

---

### 4. 개발 환경 개선

**package.json 스크립트 추가**:
```json
{
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "start": "node dist/index.js"
  }
}
```

**사용**:
```bash
# 개발 모드 (파일 변경 시 자동 컴파일)
npm run dev

# 빌드
npm run build

# 실행
npm start
```

---

### 5. ts-node로 빠른 실행

```bash
# ts-node 설치
npm install --save-dev ts-node

# TypeScript 파일 직접 실행 (컴파일 없이)
npx ts-node src/index.ts
```

**package.json 스크립트**:
```json
{
  "scripts": {
    "dev": "ts-node src/index.ts",
    "dev:watch": "nodemon --exec ts-node src/index.ts"
  }
}
```

---

## 20.4 기본 타입 시스템

### JavaScript vs TypeScript 타입

| JavaScript | TypeScript | 설명 |
|-----------|-----------|------|
| 동적 타입 | 정적 타입 | 컴파일 시 타입 체크 |
| 런타임 에러 | 컴파일 에러 | 개발 중 오류 발견 |
| 타입 추론 없음 | 타입 추론 지원 | 명시 안 해도 자동 추론 |
| any 기본 | 명시적 타입 | 타입 안정성 보장 |

---

### 기본 타입 선언

```typescript
// JavaScript
let name = "Alice";
let age = 25;
let isActive = true;

// TypeScript: 타입 명시
let name: string = "Alice";
let age: number = 25;
let isActive: boolean = true;

// 타입 추론 (권장)
let name = "Alice";        // string으로 자동 추론
let age = 25;              // number로 자동 추론
let isActive = true;       // boolean으로 자동 추론

// 타입 변경 불가!
age = "스물다섯";  // ❌ Type 'string' is not assignable to type 'number'
```

---

### Java vs TypeScript 타입 비교

**Java**:
```java
// 변수 선언
String name = "Alice";
int age = 25;
boolean isActive = true;

// 배열
String[] names = {"Alice", "Bob"};
int[] numbers = {1, 2, 3};

// 제네릭
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();
```

**TypeScript**:
```typescript
// 변수 선언
let name: string = "Alice";
let age: number = 25;
let isActive: boolean = true;

// 배열
let names: string[] = ["Alice", "Bob"];
let numbers: number[] = [1, 2, 3];

// 제네릭
let list: Array<string> = [];
let map: Map<string, number> = new Map();
```

> 💡 **차이점**:
> - Java: 원시 타입(int, boolean) vs 참조 타입(Integer, Boolean)
> - TypeScript: 모든 타입이 소문자 (number, boolean, string)

---

## 20.5 타입 추론 (Type Inference)

> 🧠 **비유**: TypeScript는 **똑똑한 비서**입니다.
> - 명시 안 해도 문맥에서 타입 추론
> - 명시가 필요한 경우만 알려줌

### 자동 타입 추론

```typescript
// 초기값으로 타입 추론
let message = "Hello";  // string
let count = 0;          // number
let isValid = true;     // boolean

// 함수 반환 타입 추론
function add(a: number, b: number) {
  return a + b;  // 반환 타입 number로 자동 추론
}

// 배열 타입 추론
let numbers = [1, 2, 3];           // number[]
let names = ["Alice", "Bob"];      // string[]
let mixed = [1, "two", 3];         // (string | number)[]
```

---

### 언제 타입을 명시해야 할까?

**1. 함수 매개변수 (필수)**
```typescript
// ❌ 타입 없으면 any
function greet(name) {  // Parameter 'name' implicitly has an 'any' type
  return `Hello, ${name}`;
}

// ✅ 타입 명시
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

**2. 초기값이 없는 변수**
```typescript
// ❌ any로 추론됨
let data;  // any
data = "text";
data = 123;

// ✅ 타입 명시
let data: string;
data = "text";
// data = 123;  // ❌ 에러
```

**3. 복잡한 객체 구조**
```typescript
// 명시하지 않아도 추론되지만, 가독성을 위해 명시 권장
interface User {
  id: number;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com"
};
```

---

## 20.6 TypeScript의 컴파일 과정

### 컴파일 흐름

```
TypeScript (.ts)
    ↓ (타입 체크)
    ↓ (컴파일)
JavaScript (.js)
    ↓ (실행)
    출력
```

### 예제

**src/example.ts**:
```typescript
interface User {
  name: string;
  age: number;
}

function greetUser(user: User): string {
  return `Hello, ${user.name}! You are ${user.age} years old.`;
}

const user: User = { name: "Alice", age: 25 };
console.log(greetUser(user));
```

**컴파일 후 dist/example.js**:
```javascript
"use strict";

function greetUser(user) {
  return `Hello, ${user.name}! You are ${user.age} years old.`;
}

const user = { name: "Alice", age: 25 };
console.log(greetUser(user));
```

> 💡 **중요**:
> - 타입 정보는 컴파일 후 모두 제거됨
> - 런타임에는 순수 JavaScript로 동작
> - 타입은 개발 시에만 존재

---

## 20.7 TypeScript Playground

**온라인 테스트**: [TypeScript Playground](https://www.typescriptlang.org/play)

- 설치 없이 브라우저에서 TypeScript 실습
- 실시간 컴파일 결과 확인
- 다양한 설정 테스트 가능

---

## 20.8 실전 예제

### 예제 1: 간단한 계산기

```typescript
// calculator.ts
type Operation = "add" | "subtract" | "multiply" | "divide";

class Calculator {
  calculate(a: number, b: number, operation: Operation): number {
    switch (operation) {
      case "add":
        return a + b;
      case "subtract":
        return a - b;
      case "multiply":
        return a * b;
      case "divide":
        if (b === 0) {
          throw new Error("Cannot divide by zero");
        }
        return a / b;
      default:
        throw new Error(`Unknown operation: ${operation}`);
    }
  }
}

// 사용
const calc = new Calculator();
console.log(calc.calculate(10, 5, "add"));       // 15
console.log(calc.calculate(10, 5, "multiply"));  // 50

// 타입 안전성
// calc.calculate(10, 5, "power");  // ❌ Argument of type '"power"' is not assignable to parameter of type 'Operation'
```

---

### 예제 2: 사용자 관리

```typescript
// user.ts
interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "user" | "guest";
}

class UserManager {
  private users: User[] = [];

  addUser(user: User): void {
    this.users.push(user);
  }

  findUserById(id: number): User | undefined {
    return this.users.find(user => user.id === id);
  }

  findUsersByRole(role: User["role"]): User[] {
    return this.users.filter(user => user.role === role);
  }

  getUserCount(): number {
    return this.users.length;
  }
}

// 사용
const manager = new UserManager();

manager.addUser({
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  role: "admin"
});

manager.addUser({
  id: 2,
  name: "Bob",
  email: "bob@example.com",
  role: "user"
});

console.log(manager.findUserById(1));           // { id: 1, name: 'Alice', ... }
console.log(manager.findUsersByRole("admin"));  // [{ id: 1, name: 'Alice', ... }]
console.log(manager.getUserCount());            // 2

// 타입 안전성
// manager.addUser({ id: 3, name: "Charlie" });  // ❌ Property 'email' is missing
```

---

## 20.9 Java 개발자를 위한 TypeScript 가이드

### 유사점

| 개념 | Java | TypeScript |
|-----|------|-----------|
| 정적 타입 | O | O |
| 인터페이스 | `interface` | `interface` |
| 클래스 | `class` | `class` |
| 제네릭 | `<T>` | `<T>` |
| 접근 제어자 | `public`, `private`, `protected` | `public`, `private`, `protected` |
| 추상 클래스 | `abstract class` | `abstract class` |

---

### 차이점

| 특성 | Java | TypeScript |
|-----|------|-----------|
| 런타임 타입 체크 | O | X (컴파일 후 제거) |
| 인터페이스 구현 | 명시적 `implements` | 구조적 타이핑 |
| null 처리 | `null` | `null`, `undefined` |
| 타입 추론 | 제한적 | 강력함 |
| 오버로딩 | 가능 | 선언만 가능 |
| 다중 상속 | 불가 | 인터페이스는 가능 |

---

### 코드 비교

**Java**:
```java
// 명시적 타입
public class User {
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return this.name;
    }
}

User user = new User("Alice", 25);
```

**TypeScript**:
```typescript
// 간결한 문법
class User {
  constructor(
    private name: string,
    private age: number
  ) {}

  getName(): string {
    return this.name;
  }
}

const user = new User("Alice", 25);  // new 필수
```

---

## 실전 팁

### 💡 Tip 1: strict 모드 활성화

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true  // 모든 엄격한 타입 체크 활성화
  }
}
```

- 초기부터 strict 모드 사용 권장
- 나중에 켜기는 어려움 (기존 코드 대량 에러)

---

### 💡 Tip 2: any 사용 최소화

```typescript
// ❌ 나쁜 예: any 남용
function processData(data: any): any {
  return data;
}

// ✅ 좋은 예: 명확한 타입
function processData<T>(data: T): T {
  return data;
}

// 또는 unknown 사용 (타입 체크 강제)
function processData(data: unknown): string {
  if (typeof data === "string") {
    return data.toUpperCase();
  }
  return String(data);
}
```

---

### 💡 Tip 3: 타입 가드 활용

```typescript
// 타입 좁히기
function processValue(value: string | number) {
  if (typeof value === "string") {
    // 이 블록에서는 value가 string
    return value.toUpperCase();
  } else {
    // 이 블록에서는 value가 number
    return value.toFixed(2);
  }
}
```

---

## 연습 문제

### 문제 1: 온도 변환기 타입 추가

JavaScript 코드를 TypeScript로 변환하세요.

```javascript
// JavaScript
function celsiusToFahrenheit(celsius) {
  return celsius * 9/5 + 32;
}

function fahrenheitToCelsius(fahrenheit) {
  return (fahrenheit - 32) * 5/9;
}
```

<details>
<summary>정답 보기</summary>

```typescript
function celsiusToFahrenheit(celsius: number): number {
  return celsius * 9/5 + 32;
}

function fahrenheitToCelsius(fahrenheit: number): number {
  return (fahrenheit - 32) * 5/9;
}

// 사용
console.log(celsiusToFahrenheit(25));   // 77
console.log(fahrenheitToCelsius(77));   // 25

// 타입 체크
// celsiusToFahrenheit("25");  // ❌ 에러
```
</details>

---

### 문제 2: 사용자 인터페이스 정의

사용자 정보를 담는 인터페이스를 작성하고, 사용자 배열을 필터링하는 함수를 작성하세요.

<details>
<summary>정답 보기</summary>

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  isActive: boolean;
}

function filterActiveUsers(users: User[]): User[] {
  return users.filter(user => user.isActive);
}

function filterUsersByAge(users: User[], minAge: number): User[] {
  return users.filter(user => user.age >= minAge);
}

// 사용
const users: User[] = [
  { id: 1, name: "Alice", email: "alice@example.com", age: 25, isActive: true },
  { id: 2, name: "Bob", email: "bob@example.com", age: 30, isActive: false },
  { id: 3, name: "Charlie", email: "charlie@example.com", age: 35, isActive: true }
];

console.log(filterActiveUsers(users));
// [{ id: 1, name: 'Alice', ... }, { id: 3, name: 'Charlie', ... }]

console.log(filterUsersByAge(users, 30));
// [{ id: 2, name: 'Bob', ... }, { id: 3, name: 'Charlie', ... }]
```
</details>

---

### 문제 3: 제네릭 스택 구현

제네릭을 사용한 스택 자료구조를 구현하세요.

<details>
<summary>정답 보기</summary>

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }
}

// 사용
const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
numberStack.push(3);
console.log(numberStack.pop());      // 3
console.log(numberStack.peek());     // 2
console.log(numberStack.size());     // 2

const stringStack = new Stack<string>();
stringStack.push("hello");
stringStack.push("world");
console.log(stringStack.pop());      // world
```
</details>

---

## 핵심 요약

1. **TypeScript = JavaScript + 정적 타입**
   - 컴파일 시 타입 체크
   - 순수 JavaScript로 변환

2. **타입 추론**
   - 명시하지 않아도 자동 추론
   - 함수 매개변수는 명시 필수

3. **개발 환경**
   - `tsconfig.json`: 컴파일 설정
   - `strict: true`: 엄격한 타입 체크 권장

4. **Java와 유사하지만 더 유연함**
   - 구조적 타이핑
   - 강력한 타입 추론
   - 선택적 타입

5. **Next.js 필수**
   - React와 완벽 통합
   - 타입 안전성으로 대규모 프로젝트 관리

---

## 다음 단계

[Chapter 21. 타입 시스템 →](21-타입-시스템.md)

---

**작성일**: 2024년
**TypeScript 버전**: 5.0+
**대상**: Java Spring Backend 개발자