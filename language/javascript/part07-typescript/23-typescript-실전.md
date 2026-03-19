# Chapter 23. TypeScript 실전

## 23.1 클래스와 TypeScript

> 🏗️ **비유**: TypeScript 클래스는 **설계도 + 타입 명세**입니다.
> - Java와 유사하지만 더 간결
> - JavaScript 클래스 + 타입 안정성

### 기본 클래스

```typescript
class Person {
  // 속성 선언 (필수)
  name: string;
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  greet(): string {
    return `Hello, I'm ${this.name}`;
  }
}

const person = new Person("Alice", 25);
console.log(person.greet());  // Hello, I'm Alice
```

---

### 접근 제어자 (Access Modifiers)

```typescript
class User {
  public id: number;           // 공개 (기본값)
  private password: string;    // 비공개 (클래스 내부만)
  protected email: string;     // 보호 (클래스 + 서브클래스)

  constructor(id: number, password: string, email: string) {
    this.id = id;
    this.password = password;
    this.email = email;
  }

  // private 메서드
  private hashPassword(password: string): string {
    return `hashed_${password}`;
  }

  // public 메서드
  public changePassword(newPassword: string): void {
    this.password = this.hashPassword(newPassword);
  }
}

const user = new User(1, "secret", "user@example.com");
console.log(user.id);           // ✅ 공개
// console.log(user.password);  // ❌ 비공개
// console.log(user.email);     // ❌ 보호
```

---

### 생성자 축약 (Constructor Shorthand)

```typescript
// ❌ 일반 방식 (반복 코드)
class User {
  name: string;
  age: number;
  email: string;

  constructor(name: string, age: number, email: string) {
    this.name = name;
    this.age = age;
    this.email = email;
  }
}

// ✅ 축약 방식 (TypeScript 전용)
class User {
  constructor(
    public name: string,
    public age: number,
    public email: string
  ) {}
}

// 혼합 사용
class User {
  constructor(
    public name: string,
    private password: string,
    protected email: string,
    public readonly id: number  // 읽기 전용
  ) {}
}
```

---

### 상속 (Inheritance)

```typescript
// 기본 클래스
class Animal {
  constructor(protected name: string) {}

  makeSound(): string {
    return "Some sound";
  }

  move(distance: number): void {
    console.log(`${this.name} moved ${distance}m`);
  }
}

// 파생 클래스
class Dog extends Animal {
  constructor(name: string, private breed: string) {
    super(name);  // 부모 생성자 호출 필수
  }

  // 메서드 오버라이드
  makeSound(): string {
    return "Woof! Woof!";
  }

  // 새 메서드 추가
  getBreed(): string {
    return this.breed;
  }
}

const dog = new Dog("Rex", "Labrador");
console.log(dog.makeSound());  // Woof! Woof!
console.log(dog.getBreed());   // Labrador
dog.move(10);                  // Rex moved 10m
```

---

### 추상 클래스 (Abstract Class)

```typescript
// 추상 클래스 (인스턴스 생성 불가)
abstract class Shape {
  constructor(protected color: string) {}

  // 추상 메서드 (구현 필수)
  abstract getArea(): number;
  abstract getPerimeter(): number;

  // 일반 메서드 (구현 포함)
  describe(): string {
    return `A ${this.color} shape with area ${this.getArea()}`;
  }
}

// 구현
class Circle extends Shape {
  constructor(color: string, private radius: number) {
    super(color);
  }

  getArea(): number {
    return Math.PI * this.radius ** 2;
  }

  getPerimeter(): number {
    return 2 * Math.PI * this.radius;
  }
}

class Rectangle extends Shape {
  constructor(color: string, private width: number, private height: number) {
    super(color);
  }

  getArea(): number {
    return this.width * this.height;
  }

  getPerimeter(): number {
    return 2 * (this.width + this.height);
  }
}

// 사용
const circle = new Circle("red", 5);
console.log(circle.getArea());     // 78.54
console.log(circle.describe());    // A red shape with area 78.54

const rectangle = new Rectangle("blue", 4, 6);
console.log(rectangle.getArea());  // 24
```

---

### Java vs TypeScript 클래스

**Java**:
```java
// 명시적 getter/setter
public class User {
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

**TypeScript**:
```typescript
// 간결한 문법
class User {
  constructor(
    public name: string,
    public age: number
  ) {}
}

// 또는 getter/setter 사용
class User {
  constructor(private _name: string, private _age: number) {}

  get name(): string {
    return this._name;
  }

  set name(value: string) {
    this._name = value;
  }

  get age(): number {
    return this._age;
  }

  set age(value: number) {
    if (value >= 0) {
      this._age = value;
    }
  }
}
```

---

## 23.2 인터페이스와 클래스

### 인터페이스 구현 (implements)

```typescript
// 인터페이스 정의
interface Printable {
  print(): void;
}

interface Saveable {
  save(): void;
  load(): void;
}

// 단일 인터페이스 구현
class Document implements Printable {
  constructor(private content: string) {}

  print(): void {
    console.log(this.content);
  }
}

// 다중 인터페이스 구현
class TextDocument implements Printable, Saveable {
  constructor(private content: string) {}

  print(): void {
    console.log(this.content);
  }

  save(): void {
    console.log("Saving document...");
  }

  load(): void {
    console.log("Loading document...");
  }
}
```

---

### 구조적 타이핑 (Structural Typing)

> 🔌 **비유**: TypeScript는 **USB 포트**처럼 생겼으면 호환됩니다.
> - 모양이 같으면 호환
> - 명시적 구현 불필요

```typescript
interface Point {
  x: number;
  y: number;
}

// 인터페이스를 명시하지 않아도 호환
class Coordinate {
  constructor(public x: number, public y: number) {}
}

// ✅ 구조가 같으면 호환됨!
function printPoint(point: Point): void {
  console.log(`(${point.x}, ${point.y})`);
}

const coord = new Coordinate(10, 20);
printPoint(coord);  // ✅ 작동함!

// 객체 리터럴도 가능
printPoint({ x: 5, y: 15 });  // ✅
```

---

### Java vs TypeScript 타이핑

**Java (명목적 타이핑)**:
```java
interface Printable {
    void print();
}

class Document {
    public void print() {
        System.out.println("Printing...");
    }
}

// ❌ 컴파일 에러: Document는 Printable을 명시적으로 구현하지 않음
Printable doc = new Document();
```

**TypeScript (구조적 타이핑)**:
```typescript
interface Printable {
  print(): void;
}

class Document {
  print(): void {
    console.log("Printing...");
  }
}

// ✅ 구조가 같으면 호환!
const doc: Printable = new Document();
```

---

## 23.3 데코레이터 (Decorators)

> 🎨 **비유**: 데코레이터는 **스티커**입니다.
> - 클래스나 메서드에 기능 추가
> - Java의 Annotation과 유사

**실험적 기능 (tsconfig.json 설정 필요)**

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

---

### 클래스 데코레이터

```typescript
// 로깅 데코레이터
function logged(constructor: Function) {
  console.log(`Class ${constructor.name} was created`);
}

@logged
class Person {
  constructor(public name: string) {}
}

const person = new Person("Alice");
// 출력: Class Person was created
```

---

### 메서드 데코레이터

```typescript
// 실행 시간 측정 데코레이터
function measure(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    const start = performance.now();
    const result = originalMethod.apply(this, args);
    const end = performance.now();
    console.log(`${propertyKey} took ${end - start}ms`);
    return result;
  };

  return descriptor;
}

class Calculator {
  @measure
  fibonacci(n: number): number {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}

const calc = new Calculator();
calc.fibonacci(10);  // fibonacci took Xms
```

---

### 속성 데코레이터

```typescript
// 읽기 전용 데코레이터
function readonly(target: any, propertyKey: string) {
  Object.defineProperty(target, propertyKey, {
    writable: false
  });
}

class Config {
  @readonly
  apiUrl: string = "https://api.example.com";
}

const config = new Config();
// config.apiUrl = "https://new-api.com";  // ❌ 에러 (런타임)
```

---

### 매개변수 데코레이터

```typescript
// 유효성 검사 데코레이터
function validate(target: any, propertyKey: string, parameterIndex: number) {
  console.log(`Validating parameter ${parameterIndex} of ${propertyKey}`);
}

class UserService {
  createUser(@validate name: string, @validate age: number) {
    console.log(`Creating user: ${name}, ${age}`);
  }
}
```

---

## 23.4 모듈 시스템

### ES6 모듈 (권장)

**내보내기 (Export)**

```typescript
// math.ts

// Named Export
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export const PI = 3.14159;

// 한 번에 내보내기
function multiply(a: number, b: number): number {
  return a * b;
}

function divide(a: number, b: number): number {
  return a / b;
}

export { multiply, divide };

// Default Export (파일당 하나)
export default class Calculator {
  add(a: number, b: number): number {
    return a + b;
  }
}
```

**가져오기 (Import)**

```typescript
// app.ts

// Named Import
import { add, subtract, PI } from "./math";

console.log(add(5, 3));        // 8
console.log(subtract(5, 3));   // 2
console.log(PI);               // 3.14159

// 별칭 사용
import { add as sum, subtract as diff } from "./math";

// 전체 가져오기
import * as Math from "./math";
console.log(Math.add(5, 3));   // 8
console.log(Math.PI);          // 3.14159

// Default Import
import Calculator from "./math";
const calc = new Calculator();
console.log(calc.add(5, 3));   // 8

// 혼합
import Calculator, { add, PI } from "./math";
```

---

### 타입 내보내기/가져오기

```typescript
// types.ts

// 타입 내보내기
export interface User {
  id: number;
  name: string;
  email: string;
}

export type Status = "active" | "inactive" | "pending";

// 타입만 내보내기 (명시적)
export type { User as UserType, Status as StatusType };
```

```typescript
// app.ts

// 타입 가져오기
import { User, Status } from "./types";

// 타입만 가져오기 (명시적, 컴파일 후 제거됨)
import type { User, Status } from "./types";

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com"
};

const status: Status = "active";
```

---

### Java vs TypeScript 모듈

**Java**:
```java
// Math.java
package com.example.utils;

public class Math {
    public static int add(int a, int b) {
        return a + b;
    }
}

// App.java
import com.example.utils.Math;

public class App {
    public static void main(String[] args) {
        System.out.println(Math.add(5, 3));
    }
}
```

**TypeScript**:
```typescript
// math.ts
export function add(a: number, b: number): number {
  return a + b;
}

// app.ts
import { add } from "./math";
console.log(add(5, 3));
```

---

## 23.5 타입 선언 파일 (.d.ts)

> 📋 **비유**: 타입 선언 파일은 **사용 설명서**입니다.
> - JavaScript 라이브러리에 타입 정보 추가
> - 컴파일 시 제거됨

### 기본 선언 파일

```typescript
// math.d.ts
declare function add(a: number, b: number): number;
declare function subtract(a: number, b: number): number;
declare const PI: number;

// app.ts
// JavaScript 파일이 있다고 가정
import { add, PI } from "./math";
console.log(add(5, 3));
console.log(PI);
```

---

### 전역 타입 선언

```typescript
// global.d.ts
declare global {
  interface Window {
    myCustomApi: {
      get(key: string): any;
      set(key: string, value: any): void;
    };
  }

  var VERSION: string;
}

export {};

// app.ts
// 타입 체크 없이 사용 가능
console.log(VERSION);
window.myCustomApi.set("key", "value");
```

---

### DefinitelyTyped

**외부 JavaScript 라이브러리의 타입 정의**

```bash
# @types 패키지 설치
npm install --save-dev @types/node
npm install --save-dev @types/express
npm install --save-dev @types/react
npm install --save-dev @types/lodash
```

```typescript
// Node.js 타입 사용
import * as fs from "fs";
fs.readFileSync("file.txt", "utf-8");

// Express 타입 사용
import express, { Request, Response } from "express";
const app = express();

app.get("/", (req: Request, res: Response) => {
  res.send("Hello");
});

// Lodash 타입 사용
import _ from "lodash";
const result = _.uniq([1, 2, 2, 3, 4, 4, 5]);
```

---

## 23.6 실전 프로젝트 구조

### 프로젝트 구조 예시

```
my-typescript-app/
├── src/
│   ├── models/          # 데이터 모델
│   │   ├── User.ts
│   │   └── Post.ts
│   ├── services/        # 비즈니스 로직
│   │   ├── UserService.ts
│   │   └── PostService.ts
│   ├── repositories/    # 데이터 접근
│   │   ├── UserRepository.ts
│   │   └── PostRepository.ts
│   ├── controllers/     # 요청 처리
│   │   ├── UserController.ts
│   │   └── PostController.ts
│   ├── utils/           # 유틸리티
│   │   ├── logger.ts
│   │   └── validator.ts
│   ├── types/           # 타입 정의
│   │   ├── index.ts
│   │   └── api.ts
│   └── index.ts         # 진입점
├── tests/               # 테스트
│   └── unit/
├── dist/                # 컴파일 결과
├── node_modules/
├── package.json
├── tsconfig.json
└── .gitignore
```

---

### 실전 예제: MVC 패턴

**models/User.ts**:
```typescript
export interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
}
```

**repositories/UserRepository.ts**:
```typescript
import { User, CreateUserDto } from "../models/User";

export class UserRepository {
  private users: User[] = [];
  private nextId = 1;

  create(dto: CreateUserDto): User {
    const user: User = {
      id: this.nextId++,
      ...dto,
      createdAt: new Date()
    };
    this.users.push(user);
    return user;
  }

  findById(id: number): User | undefined {
    return this.users.find(u => u.id === id);
  }

  findByEmail(email: string): User | undefined {
    return this.users.find(u => u.email === email);
  }

  findAll(): User[] {
    return this.users;
  }

  update(id: number, updates: Partial<User>): User | undefined {
    const user = this.findById(id);
    if (user) {
      Object.assign(user, updates);
    }
    return user;
  }

  delete(id: number): boolean {
    const index = this.users.findIndex(u => u.id === id);
    if (index !== -1) {
      this.users.splice(index, 1);
      return true;
    }
    return false;
  }
}
```

**services/UserService.ts**:
```typescript
import { User, CreateUserDto, UpdateUserDto } from "../models/User";
import { UserRepository } from "../repositories/UserRepository";

export class UserService {
  constructor(private userRepository: UserRepository) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    // 검증
    if (!this.isValidEmail(dto.email)) {
      throw new Error("Invalid email format");
    }

    // 중복 체크
    const existing = this.userRepository.findByEmail(dto.email);
    if (existing) {
      throw new Error("Email already exists");
    }

    // 비밀번호 해싱 (실제로는 bcrypt 사용)
    const hashedPassword = this.hashPassword(dto.password);

    return this.userRepository.create({
      ...dto,
      password: hashedPassword
    });
  }

  async getUserById(id: number): Promise<User | undefined> {
    return this.userRepository.findById(id);
  }

  async getAllUsers(): Promise<User[]> {
    return this.userRepository.findAll();
  }

  async updateUser(id: number, dto: UpdateUserDto): Promise<User | undefined> {
    return this.userRepository.update(id, dto);
  }

  async deleteUser(id: number): Promise<boolean> {
    return this.userRepository.delete(id);
  }

  private isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  private hashPassword(password: string): string {
    return `hashed_${password}`;
  }
}
```

**controllers/UserController.ts**:
```typescript
import { UserService } from "../services/UserService";
import { CreateUserDto, UpdateUserDto } from "../models/User";

interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
}

export class UserController {
  constructor(private userService: UserService) {}

  async create(dto: CreateUserDto): Promise<ApiResponse<any>> {
    try {
      const user = await this.userService.createUser(dto);
      return {
        success: true,
        data: this.sanitizeUser(user)
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }

  async getById(id: number): Promise<ApiResponse<any>> {
    const user = await this.userService.getUserById(id);
    if (!user) {
      return {
        success: false,
        error: "User not found"
      };
    }
    return {
      success: true,
      data: this.sanitizeUser(user)
    };
  }

  async getAll(): Promise<ApiResponse<any[]>> {
    const users = await this.userService.getAllUsers();
    return {
      success: true,
      data: users.map(u => this.sanitizeUser(u))
    };
  }

  // 비밀번호 제거
  private sanitizeUser(user: any) {
    const { password, ...rest } = user;
    return rest;
  }
}
```

**index.ts**:
```typescript
import { UserRepository } from "./repositories/UserRepository";
import { UserService } from "./services/UserService";
import { UserController } from "./controllers/UserController";

// 의존성 주입
const userRepository = new UserRepository();
const userService = new UserService(userRepository);
const userController = new UserController(userService);

// 사용
async function main() {
  // 사용자 생성
  const createResult = await userController.create({
    name: "Alice",
    email: "alice@example.com",
    password: "password123"
  });
  console.log(createResult);

  // 모든 사용자 조회
  const allUsers = await userController.getAll();
  console.log(allUsers);
}

main();
```

---

## 23.7 설정 파일 (tsconfig.json)

### 프로덕션 설정

```json
{
  "compilerOptions": {
    /* 언어 버전 */
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],

    /* 모듈 해석 */
    "moduleResolution": "node",
    "esModuleInterop": true,
    "resolveJsonModule": true,

    /* 출력 */
    "outDir": "./dist",
    "rootDir": "./src",
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "removeComments": true,

    /* 엄격한 타입 체크 */
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,

    /* 추가 체크 */
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,

    /* 실험적 기능 */
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,

    /* 고급 옵션 */
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

---

## 실전 팁

### 💡 Tip 1: 타입 가드 함수

```typescript
// 타입 가드 유틸리티
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNumber(value: unknown): value is number {
  return typeof value === "number";
}

function isArray<T>(value: unknown): value is T[] {
  return Array.isArray(value);
}

// 사용
function process(value: unknown) {
  if (isString(value)) {
    console.log(value.toUpperCase());
  } else if (isNumber(value)) {
    console.log(value.toFixed(2));
  }
}
```

---

### 💡 Tip 2: 환경 변수 타입

```typescript
// env.d.ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "production" | "test";
      PORT: string;
      DATABASE_URL: string;
      JWT_SECRET: string;
    }
  }
}

export {};

// app.ts
// 타입 안전한 환경 변수 접근
const port = parseInt(process.env.PORT);
const dbUrl = process.env.DATABASE_URL;
```

---

### 💡 Tip 3: 에러 처리

```typescript
// 커스텀 에러 클래스
class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, "NOT_FOUND");
  }
}

class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400, "VALIDATION_ERROR");
  }
}

// 사용
function getUser(id: number): User {
  const user = userRepository.findById(id);
  if (!user) {
    throw new NotFoundError("User");
  }
  return user;
}

// 에러 핸들링
try {
  const user = getUser(999);
} catch (error) {
  if (error instanceof NotFoundError) {
    console.log("User not found");
  } else if (error instanceof ValidationError) {
    console.log("Validation failed");
  } else {
    console.log("Unknown error");
  }
}
```

---

## 연습 문제

### 문제 1: Todo List 시스템

타입 안전한 Todo List 시스템을 구현하세요.

<details>
<summary>정답 보기</summary>

```typescript
// models/Todo.ts
export interface Todo {
  id: number;
  title: string;
  description: string;
  completed: boolean;
  priority: "low" | "medium" | "high";
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateTodoDto {
  title: string;
  description: string;
  priority: Todo["priority"];
}

// repositories/TodoRepository.ts
export class TodoRepository {
  private todos: Todo[] = [];
  private nextId = 1;

  create(dto: CreateTodoDto): Todo {
    const todo: Todo = {
      id: this.nextId++,
      ...dto,
      completed: false,
      createdAt: new Date(),
      updatedAt: new Date()
    };
    this.todos.push(todo);
    return todo;
  }

  findAll(): Todo[] {
    return this.todos;
  }

  findById(id: number): Todo | undefined {
    return this.todos.find(t => t.id === id);
  }

  update(id: number, updates: Partial<Todo>): Todo | undefined {
    const todo = this.findById(id);
    if (todo) {
      Object.assign(todo, { ...updates, updatedAt: new Date() });
    }
    return todo;
  }

  delete(id: number): boolean {
    const index = this.todos.findIndex(t => t.id === id);
    if (index !== -1) {
      this.todos.splice(index, 1);
      return true;
    }
    return false;
  }
}

// services/TodoService.ts
export class TodoService {
  constructor(private repository: TodoRepository) {}

  createTodo(dto: CreateTodoDto): Todo {
    if (!dto.title.trim()) {
      throw new Error("Title is required");
    }
    return this.repository.create(dto);
  }

  getTodoById(id: number): Todo | undefined {
    return this.repository.findById(id);
  }

  getAllTodos(): Todo[] {
    return this.repository.findAll();
  }

  getCompletedTodos(): Todo[] {
    return this.repository.findAll().filter(t => t.completed);
  }

  getPendingTodos(): Todo[] {
    return this.repository.findAll().filter(t => !t.completed);
  }

  completeTodo(id: number): Todo | undefined {
    return this.repository.update(id, { completed: true });
  }

  deleteTodo(id: number): boolean {
    return this.repository.delete(id);
  }
}

// 사용
const repository = new TodoRepository();
const service = new TodoService(repository);

const todo = service.createTodo({
  title: "Learn TypeScript",
  description: "Complete all chapters",
  priority: "high"
});

console.log(todo);
service.completeTodo(todo.id);
console.log(service.getCompletedTodos());
```
</details>

---

## 핵심 요약

1. **클래스**: Java와 유사하지만 더 간결한 문법
2. **접근 제어자**: `public`, `private`, `protected`
3. **데코레이터**: 메타프로그래밍 기능 (실험적)
4. **모듈 시스템**: ES6 모듈 사용
5. **타입 선언 파일**: `.d.ts`로 JavaScript 라이브러리에 타입 추가
6. **프로젝트 구조**: MVC 패턴, 레이어 아키텍처
7. **strict 모드**: 항상 활성화하여 타입 안정성 보장

---

## 다음 단계

[Part 08. React Fundamentals →](../part08-react-fundamentals/24-react-시작하기.md)

---

**작성일**: 2024년
**TypeScript 버전**: 5.0+
**대상**: Java Spring Backend 개발자