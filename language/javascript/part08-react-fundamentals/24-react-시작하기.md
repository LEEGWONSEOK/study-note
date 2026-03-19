# Chapter 24. React 시작하기

## 24.1 React란?

> ⚛️ **비유**: React는 **레고 블록**입니다.
> - 작은 컴포넌트(블록)를 조합하여 UI 구성
> - 재사용 가능하고 조립식
> - 복잡한 UI도 간단한 조각들로 분해

**React**: Facebook(Meta)이 개발한 **UI 라이브러리**
- 컴포넌트 기반 아키텍처
- 선언적 프로그래밍
- 가상 DOM으로 효율적 렌더링
- 단방향 데이터 흐름

### 특징

1. **컴포넌트 기반 (Component-Based)**
   - UI를 독립적인 컴포넌트로 분리
   - 재사용 가능한 조각

2. **선언적 (Declarative)**
   - "어떻게(How)" 대신 "무엇을(What)" 표현
   - 상태에 따라 UI 자동 업데이트

3. **가상 DOM (Virtual DOM)**
   - 실제 DOM 조작 최소화
   - 빠른 렌더링 성능

4. **단방향 데이터 흐름 (One-way Data Flow)**
   - 부모 → 자식으로만 데이터 전달
   - 예측 가능한 상태 관리

---

## 24.2 왜 React를 사용할까?

### 명령형 vs 선언형

**명령형 (Vanilla JavaScript)**:
```javascript
// "어떻게" 동작하는지 명시
const button = document.createElement("button");
button.textContent = "클릭";
button.addEventListener("click", () => {
  const count = parseInt(button.textContent);
  button.textContent = count + 1;
});
document.body.appendChild(button);
```

**선언형 (React)**:
```jsx
// "무엇을" 보여줄지 명시
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

> 💡 **차이점**:
> - 명령형: DOM 조작 단계를 직접 작성
> - 선언형: 원하는 결과만 작성, React가 DOM 업데이트

---

### Java 개발자 관점

**Java (Spring MVC)**:
```java
@Controller
public class UserController {
    @GetMapping("/users")
    public String users(Model model) {
        List<User> users = userService.findAll();
        model.addAttribute("users", users);
        return "users";  // users.html (Thymeleaf)
    }
}
```

```html
<!-- users.html (Thymeleaf) -->
<table>
  <tr th:each="user : ${users}">
    <td th:text="${user.name}">Name</td>
    <td th:text="${user.email}">Email</td>
  </tr>
</table>
```

**React**:
```jsx
function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch("/api/users")
      .then(res => res.json())
      .then(data => setUsers(data));
  }, []);

  return (
    <table>
      {users.map(user => (
        <tr key={user.id}>
          <td>{user.name}</td>
          <td>{user.email}</td>
        </tr>
      ))}
    </table>
  );
}
```

> 💡 **차이점**:
> - Spring MVC: 서버에서 HTML 생성 (SSR)
> - React: 브라우저에서 UI 생성 (CSR)
> - Next.js는 둘을 결합 (SSR + CSR)

---

## 24.3 React 개발 환경 설정

### 방법 1: Create React App (CRA)

```bash
# Create React App으로 프로젝트 생성
npx create-react-app my-app
cd my-app

# TypeScript 버전
npx create-react-app my-app --template typescript

# 개발 서버 실행
npm start
# http://localhost:3000
```

**프로젝트 구조**:
```
my-app/
├── node_modules/
├── public/
│   ├── index.html      # HTML 템플릿
│   └── favicon.ico
├── src/
│   ├── App.tsx         # 메인 컴포넌트
│   ├── App.css
│   ├── index.tsx       # 진입점
│   └── index.css
├── package.json
└── tsconfig.json
```

---

### 방법 2: Vite (권장, 빠름)

```bash
# Vite로 프로젝트 생성
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install

# 개발 서버 실행
npm run dev
# http://localhost:5173
```

**장점**:
- CRA보다 10배 빠른 시작 시간
- HMR (Hot Module Replacement) 지원
- 가볍고 빠른 빌드

---

### 방법 3: Next.js (프로덕션 권장)

```bash
# Next.js 프로젝트 생성
npx create-next-app@latest my-app --typescript
cd my-app

# 개발 서버 실행
npm run dev
# http://localhost:3000
```

**장점**:
- SSR (Server-Side Rendering)
- 파일 기반 라우팅
- API Routes
- 이미지 최적화
- **프로덕션 환경에서 권장**

---

## 24.4 첫 React 컴포넌트

### JSX (JavaScript XML)

> 🎨 **비유**: JSX는 **HTML + JavaScript의 하이브리드**입니다.
> - JavaScript 안에서 HTML처럼 작성
> - 컴파일되면 순수 JavaScript로 변환

**JSX 예제**:
```jsx
// JSX
const element = <h1>Hello, React!</h1>;

// 컴파일 후 JavaScript
const element = React.createElement("h1", null, "Hello, React!");
```

---

### 함수형 컴포넌트

```tsx
// src/App.tsx
function App() {
  return (
    <div className="App">
      <h1>Hello, React!</h1>
      <p>Welcome to React development!</p>
    </div>
  );
}

export default App;
```

---

### JSX 문법 규칙

**1. 단일 루트 요소**
```jsx
// ❌ 에러: 여러 개의 루트 요소
function App() {
  return (
    <h1>Title</h1>
    <p>Paragraph</p>
  );
}

// ✅ 올바른 방법 1: div로 감싸기
function App() {
  return (
    <div>
      <h1>Title</h1>
      <p>Paragraph</p>
    </div>
  );
}

// ✅ 올바른 방법 2: Fragment 사용 (권장)
function App() {
  return (
    <>
      <h1>Title</h1>
      <p>Paragraph</p>
    </>
  );
}

// ✅ 올바른 방법 3: React.Fragment (명시적)
function App() {
  return (
    <React.Fragment>
      <h1>Title</h1>
      <p>Paragraph</p>
    </React.Fragment>
  );
}
```

---

**2. 표현식 삽입 ({})**
```jsx
function Greeting() {
  const name = "Alice";
  const age = 25;

  return (
    <div>
      <h1>Hello, {name}!</h1>
      <p>You are {age} years old.</p>
      <p>Next year you'll be {age + 1}.</p>
    </div>
  );
}
```

---

**3. 속성 (Attributes)**
```jsx
function Image() {
  const imageUrl = "https://example.com/image.jpg";
  const altText = "Example Image";

  return (
    <div>
      {/* className (not class) */}
      <div className="container">
        {/* htmlFor (not for) */}
        <label htmlFor="input">Label</label>
        <input id="input" type="text" />

        {/* 속성에 표현식 사용 */}
        <img src={imageUrl} alt={altText} />

        {/* 스타일 객체 */}
        <div style={{ color: "red", fontSize: "20px" }}>
          Styled Text
        </div>
      </div>
    </div>
  );
}
```

---

**4. 조건부 렌더링**
```jsx
function Greeting({ isLoggedIn }: { isLoggedIn: boolean }) {
  // if 문
  if (isLoggedIn) {
    return <h1>Welcome back!</h1>;
  }
  return <h1>Please log in.</h1>;

  // 삼항 연산자 (권장)
  return (
    <div>
      {isLoggedIn ? (
        <h1>Welcome back!</h1>
      ) : (
        <h1>Please log in.</h1>
      )}
    </div>
  );

  // && 연산자 (조건이 true일 때만)
  return (
    <div>
      {isLoggedIn && <h1>Welcome back!</h1>}
    </div>
  );
}
```

---

**5. 리스트 렌더링**
```jsx
function UserList() {
  const users = [
    { id: 1, name: "Alice", email: "alice@example.com" },
    { id: 2, name: "Bob", email: "bob@example.com" },
    { id: 3, name: "Charlie", email: "charlie@example.com" }
  ];

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          {user.name} ({user.email})
        </li>
      ))}
    </ul>
  );
}
```

> ⚠️ **중요**: `key` prop은 필수!
> - React가 변경사항을 효율적으로 추적
> - 고유한 값 사용 (id 등)
> - 인덱스는 피하기 (성능 문제)

---

## 24.5 TypeScript와 React

### 컴포넌트 타입

```tsx
// Props 타입 정의
interface GreetingProps {
  name: string;
  age?: number;  // 선택적
}

function Greeting({ name, age }: GreetingProps) {
  return (
    <div>
      <h1>Hello, {name}!</h1>
      {age && <p>You are {age} years old.</p>}
    </div>
  );
}

// 사용
<Greeting name="Alice" />
<Greeting name="Bob" age={30} />
```

---

### 이벤트 핸들러 타입

```tsx
function Button() {
  // 버튼 클릭
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log("Button clicked", e.currentTarget);
  };

  // 입력 변경
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log("Input value:", e.target.value);
  };

  // 폼 제출
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    console.log("Form submitted");
  };

  return (
    <div>
      <button onClick={handleClick}>Click</button>
      <input type="text" onChange={handleChange} />
      <form onSubmit={handleSubmit}>
        <button type="submit">Submit</button>
      </form>
    </div>
  );
}
```

---

### 자식 컴포넌트 타입

```tsx
interface ContainerProps {
  children: React.ReactNode;  // 모든 React 요소
  title?: string;
}

function Container({ children, title }: ContainerProps) {
  return (
    <div className="container">
      {title && <h2>{title}</h2>}
      {children}
    </div>
  );
}

// 사용
<Container title="My Container">
  <p>This is content</p>
  <button>Click me</button>
</Container>
```

---

## 24.6 실전 예제

### 예제 1: Counter

```tsx
// src/components/Counter.tsx
import { useState } from "react";

interface CounterProps {
  initialValue?: number;
}

function Counter({ initialValue = 0 }: CounterProps) {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount(count + 1);
  const decrement = () => setCount(count - 1);
  const reset = () => setCount(initialValue);

  return (
    <div className="counter">
      <h2>Counter: {count}</h2>
      <div className="buttons">
        <button onClick={decrement}>-</button>
        <button onClick={reset}>Reset</button>
        <button onClick={increment}>+</button>
      </div>
    </div>
  );
}

export default Counter;
```

---

### 예제 2: Todo List

```tsx
// src/components/TodoList.tsx
import { useState } from "react";

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [inputValue, setInputValue] = useState("");

  const addTodo = () => {
    if (inputValue.trim() === "") return;

    const newTodo: Todo = {
      id: Date.now(),
      text: inputValue,
      completed: false
    };

    setTodos([...todos, newTodo]);
    setInputValue("");
  };

  const toggleTodo = (id: number) => {
    setTodos(
      todos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  };

  const deleteTodo = (id: number) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };

  return (
    <div className="todo-list">
      <h2>Todo List</h2>

      <div className="input-group">
        <input
          type="text"
          value={inputValue}
          onChange={e => setInputValue(e.target.value)}
          onKeyPress={e => e.key === "Enter" && addTodo()}
          placeholder="Add a new todo..."
        />
        <button onClick={addTodo}>Add</button>
      </div>

      <ul>
        {todos.map(todo => (
          <li key={todo.id} className={todo.completed ? "completed" : ""}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span>{todo.text}</span>
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>

      <p>
        Total: {todos.length} | Completed: {todos.filter(t => t.completed).length}
      </p>
    </div>
  );
}

export default TodoList;
```

---

### 예제 3: User Card

```tsx
// src/components/UserCard.tsx
interface User {
  id: number;
  name: string;
  email: string;
  avatar?: string;
}

interface UserCardProps {
  user: User;
  onEdit?: (user: User) => void;
  onDelete?: (id: number) => void;
}

function UserCard({ user, onEdit, onDelete }: UserCardProps) {
  return (
    <div className="user-card">
      {user.avatar && (
        <img src={user.avatar} alt={user.name} className="avatar" />
      )}
      <div className="info">
        <h3>{user.name}</h3>
        <p>{user.email}</p>
      </div>
      <div className="actions">
        {onEdit && <button onClick={() => onEdit(user)}>Edit</button>}
        {onDelete && <button onClick={() => onDelete(user.id)}>Delete</button>}
      </div>
    </div>
  );
}

export default UserCard;
```

---

## 24.7 React Developer Tools

**브라우저 확장 프로그램**:
- [Chrome](https://chrome.google.com/webstore/detail/react-developer-tools)
- [Firefox](https://addons.mozilla.org/en-US/firefox/addon/react-devtools/)

**기능**:
- 컴포넌트 트리 확인
- Props/State 검사
- 성능 프로파일링
- Hooks 디버깅

---

## 실전 팁

### 💡 Tip 1: 컴포넌트 파일 구조

```
src/
├── components/
│   ├── common/          # 공통 컴포넌트
│   │   ├── Button.tsx
│   │   └── Input.tsx
│   ├── features/        # 기능별 컴포넌트
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   └── todos/
│   │       ├── TodoList.tsx
│   │       └── TodoItem.tsx
│   └── layout/          # 레이아웃 컴포넌트
│       ├── Header.tsx
│       ├── Footer.tsx
│       └── Sidebar.tsx
```

---

### 💡 Tip 2: 네이밍 컨벤션

```tsx
// 컴포넌트: PascalCase
function UserProfile() {}

// 함수/변수: camelCase
const handleClick = () => {};
const isLoading = false;

// 상수: UPPER_SNAKE_CASE
const API_URL = "https://api.example.com";

// 타입/인터페이스: PascalCase
interface UserProps {}
type Status = "pending" | "success" | "error";
```

---

### 💡 Tip 3: 조건부 렌더링 패턴

```tsx
// ❌ 나쁜 예: 중첩된 삼항 연산자
{isLoading ? (
  <Spinner />
) : error ? (
  <Error message={error} />
) : data ? (
  <Content data={data} />
) : (
  <Empty />
)}

// ✅ 좋은 예: Early return
function Component() {
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error} />;
  if (!data) return <Empty />;
  return <Content data={data} />;
}

// ✅ 좋은 예: 상태별 컴포넌트 분리
function Component() {
  const status = getStatus();
  const components = {
    loading: <Spinner />,
    error: <Error />,
    empty: <Empty />,
    success: <Content />
  };
  return components[status];
}
```

---

## 연습 문제

### 문제 1: Greeting 컴포넌트

이름과 시간대를 받아서 인사말을 표시하는 컴포넌트를 작성하세요.

<details>
<summary>정답 보기</summary>

```tsx
interface GreetingProps {
  name: string;
  timeOfDay: "morning" | "afternoon" | "evening";
}

function Greeting({ name, timeOfDay }: GreetingProps) {
  const greetings = {
    morning: "Good morning",
    afternoon: "Good afternoon",
    evening: "Good evening"
  };

  return (
    <div className="greeting">
      <h1>{greetings[timeOfDay]}, {name}!</h1>
    </div>
  );
}

// 사용
<Greeting name="Alice" timeOfDay="morning" />
```
</details>

---

### 문제 2: Product Card

제품 정보를 표시하는 카드 컴포넌트를 작성하세요.

<details>
<summary>정답 보기</summary>

```tsx
interface Product {
  id: number;
  name: string;
  price: number;
  image: string;
  inStock: boolean;
}

interface ProductCardProps {
  product: Product;
  onAddToCart: (id: number) => void;
}

function ProductCard({ product, onAddToCart }: ProductCardProps) {
  return (
    <div className="product-card">
      <img src={product.image} alt={product.name} />
      <h3>{product.name}</h3>
      <p className="price">${product.price.toFixed(2)}</p>
      {product.inStock ? (
        <button onClick={() => onAddToCart(product.id)}>
          Add to Cart
        </button>
      ) : (
        <p className="out-of-stock">Out of Stock</p>
      )}
    </div>
  );
}
```
</details>

---

## 핵심 요약

1. **React = 컴포넌트 기반 UI 라이브러리**
2. **JSX = JavaScript + XML** (HTML처럼 작성)
3. **선언적 프로그래밍**: 상태 → UI 자동 업데이트
4. **함수형 컴포넌트**: 간결하고 현대적
5. **TypeScript**: 타입 안정성으로 대규모 프로젝트 관리
6. **key prop**: 리스트 렌더링 시 필수
7. **단일 루트 요소**: Fragment 사용 권장

---

## 다음 단계

[Chapter 25. 컴포넌트와 Props →](25-컴포넌트와-props.md)

---

**작성일**: 2024년
**React 버전**: 18+
**대상**: Java Spring Backend 개발자