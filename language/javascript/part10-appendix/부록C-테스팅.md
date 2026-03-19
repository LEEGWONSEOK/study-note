# 부록 C. 테스팅

## 테스팅이 중요한 이유

테스트는 **코드가 의도대로 동작하는지 자동으로 검증**합니다.

### 비유로 이해하기
테스트는 **품질 검사**와 같습니다:
- 제품 출하 전 검사
- 결함 조기 발견
- 안정적인 변경
- 문서화 역할

## Jest: JavaScript 테스팅 프레임워크

### 설치

```bash
npm install --save-dev jest @types/jest
```

```json
// package.json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

### 첫 테스트

```javascript
// sum.js
export function sum(a, b) {
    return a + b;
}

// sum.test.js
import { sum } from './sum';

test('1 + 2는 3이다', () => {
    expect(sum(1, 2)).toBe(3);
});

test('2 + 3은 5이다', () => {
    expect(sum(2, 3)).toBe(5);
});
```

### 매처 (Matchers)

```javascript
// 동등성
test('객체 비교', () => {
    const data = { name: 'John' };
    expect(data).toEqual({ name: 'John' }); // 값 비교
    expect(data).toBe(data); // 참조 비교
});

// 진위
test('불리언', () => {
    expect(true).toBeTruthy();
    expect(false).toBeFalsy();
    expect(null).toBeNull();
    expect(undefined).toBeUndefined();
});

// 숫자
test('숫자 비교', () => {
    expect(4).toBeGreaterThan(3);
    expect(4).toBeGreaterThanOrEqual(4);
    expect(4).toBeLessThan(5);
    expect(0.1 + 0.2).toBeCloseTo(0.3); // 부동소수점
});

// 문자열
test('문자열 매칭', () => {
    expect('Hello World').toMatch(/World/);
    expect('Hello World').toContain('Hello');
});

// 배열
test('배열 포함', () => {
    const fruits = ['apple', 'banana', 'orange'];
    expect(fruits).toContain('banana');
    expect(fruits).toHaveLength(3);
});

// 예외
test('에러 발생', () => {
    expect(() => {
        throw new Error('에러!');
    }).toThrow('에러!');
});
```

### 비동기 테스트

```javascript
// Promises
test('비동기 데이터', () => {
    return fetchData().then(data => {
        expect(data).toBe('peanut butter');
    });
});

// async/await (권장)
test('비동기 데이터', async () => {
    const data = await fetchData();
    expect(data).toBe('peanut butter');
});

// 에러 처리
test('비동기 에러', async () => {
    expect.assertions(1);
    try {
        await fetchData();
    } catch (error) {
        expect(error).toMatch('error');
    }
});

// 또는
test('비동기 에러', async () => {
    await expect(fetchData()).rejects.toThrow();
});
```

### Setup과 Teardown

```javascript
// 각 테스트 전/후
beforeEach(() => {
    // 테스트 전 실행
    initDatabase();
});

afterEach(() => {
    // 테스트 후 실행
    clearDatabase();
});

// 모든 테스트 전/후 (한 번만)
beforeAll(() => {
    // 모든 테스트 전에 한 번
    connectDatabase();
});

afterAll(() => {
    // 모든 테스트 후에 한 번
    disconnectDatabase();
});

// 예제
describe('User 클래스', () => {
    let user;

    beforeEach(() => {
        user = new User('John');
    });

    test('이름 가져오기', () => {
        expect(user.getName()).toBe('John');
    });

    test('이름 설정', () => {
        user.setName('Jane');
        expect(user.getName()).toBe('Jane');
    });
});
```

### Mock 함수

```javascript
// Mock 함수 생성
const mockCallback = jest.fn(x => x + 1);

// 사용
[0, 1].forEach(mockCallback);

// 검증
expect(mockCallback).toHaveBeenCalledTimes(2);
expect(mockCallback).toHaveBeenCalledWith(0);
expect(mockCallback).toHaveBeenCalledWith(1);
expect(mockCallback.mock.results[0].value).toBe(1);

// 반환값 설정
const mock = jest.fn();
mock.mockReturnValue(42);
expect(mock()).toBe(42);

// 여러 번 호출 시 다른 값
mock
    .mockReturnValueOnce(10)
    .mockReturnValueOnce(20)
    .mockReturnValue(30);

expect(mock()).toBe(10);
expect(mock()).toBe(20);
expect(mock()).toBe(30);

// Promise 반환
mock.mockResolvedValue('async value');
await expect(mock()).resolves.toBe('async value');
```

### 모듈 모킹

```javascript
// api.js
export async function fetchUser(id) {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
}

// user.test.js
import { fetchUser } from './api';

jest.mock('./api');

test('사용자 가져오기', async () => {
    // 모킹된 함수 설정
    fetchUser.mockResolvedValue({
        id: 1,
        name: 'John'
    });

    const user = await fetchUser(1);
    expect(user.name).toBe('John');
});
```

## React Testing Library

React 컴포넌트를 테스트하는 라이브러리입니다.

### 설치

```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

### 기본 테스트

```javascript
// Button.jsx
export function Button({ onClick, children }) {
    return <button onClick={onClick}>{children}</button>;
}

// Button.test.jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

test('버튼 렌더링', () => {
    render(<Button>클릭</Button>);

    const button = screen.getByText('클릭');
    expect(button).toBeInTheDocument();
});

test('버튼 클릭', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>클릭</Button>);

    const button = screen.getByText('클릭');
    await userEvent.click(button);

    expect(handleClick).toHaveBeenCalledTimes(1);
});
```

### 쿼리 메서드

```javascript
// getBy* - 요소 찾기 (없으면 에러)
const button = screen.getByText('클릭');
const input = screen.getByLabelText('이름');
const heading = screen.getByRole('heading');

// queryBy* - 요소 찾기 (없으면 null)
const button = screen.queryByText('없는 버튼');
expect(button).not.toBeInTheDocument();

// findBy* - 비동기로 요소 찾기
const button = await screen.findByText('로딩 완료');

// getAllBy* - 여러 요소 찾기
const buttons = screen.getAllByRole('button');
expect(buttons).toHaveLength(3);

// 우선순위 (권장 순서)
// 1. getByRole
// 2. getByLabelText
// 3. getByPlaceholderText
// 4. getByText
// 5. getByTestId (최후의 수단)
```

### 폼 테스트

```javascript
// LoginForm.jsx
function LoginForm({ onSubmit }) {
    const [email, setEmail] = useState('');
    const [password, setPassword] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        onSubmit({ email, password });
    };

    return (
        <form onSubmit={handleSubmit}>
            <label htmlFor="email">이메일</label>
            <input
                id="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
            />

            <label htmlFor="password">비밀번호</label>
            <input
                id="password"
                type="password"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
            />

            <button type="submit">로그인</button>
        </form>
    );
}

// LoginForm.test.jsx
test('로그인 폼 제출', async () => {
    const handleSubmit = jest.fn();
    render(<LoginForm onSubmit={handleSubmit} />);

    // 입력
    await userEvent.type(
        screen.getByLabelText('이메일'),
        'test@example.com'
    );
    await userEvent.type(
        screen.getByLabelText('비밀번호'),
        'password123'
    );

    // 제출
    await userEvent.click(screen.getByText('로그인'));

    // 검증
    expect(handleSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123'
    });
});
```

### 비동기 컴포넌트 테스트

```javascript
// UserProfile.jsx
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => {
                setUser(data);
                setLoading(false);
            });
    }, [userId]);

    if (loading) return <div>로딩 중...</div>;
    return <div>{user.name}</div>;
}

// UserProfile.test.jsx
test('사용자 프로필 로드', async () => {
    // fetch 모킹
    global.fetch = jest.fn(() =>
        Promise.resolve({
            json: () => Promise.resolve({ name: 'John' })
        })
    );

    render(<UserProfile userId={1} />);

    // 로딩 상태 확인
    expect(screen.getByText('로딩 중...')).toBeInTheDocument();

    // 로드 완료 대기
    const name = await screen.findByText('John');
    expect(name).toBeInTheDocument();
});
```

### Context 테스트

```javascript
// ThemeContext.jsx
const ThemeContext = createContext();

export function ThemeProvider({ children }) {
    const [theme, setTheme] = useState('light');

    return (
        <ThemeContext.Provider value={{ theme, setTheme }}>
            {children}
        </ThemeContext.Provider>
    );
}

// ThemedButton.jsx
function ThemedButton() {
    const { theme, setTheme } = useContext(ThemeContext);

    return (
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
            현재 테마: {theme}
        </button>
    );
}

// ThemedButton.test.jsx
test('테마 토글', async () => {
    render(
        <ThemeProvider>
            <ThemedButton />
        </ThemeProvider>
    );

    const button = screen.getByText(/현재 테마: light/);
    await userEvent.click(button);

    expect(screen.getByText(/현재 테마: dark/)).toBeInTheDocument();
});
```

### 커스텀 렌더 함수

```javascript
// test-utils.js
import { render } from '@testing-library/react';
import { ThemeProvider } from './ThemeContext';
import { AuthProvider } from './AuthContext';

function AllTheProviders({ children }) {
    return (
        <ThemeProvider>
            <AuthProvider>
                {children}
            </AuthProvider>
        </ThemeProvider>
    );
}

export function renderWithProviders(ui, options) {
    return render(ui, { wrapper: AllTheProviders, ...options });
}

export * from '@testing-library/react';

// 사용
import { renderWithProviders } from './test-utils';

test('프로바이더와 함께 렌더링', () => {
    renderWithProviders(<MyComponent />);
    // 테스트...
});
```

## E2E 테스팅 (Playwright)

전체 애플리케이션을 **실제 브라우저에서 테스트**합니다.

### 설치

```bash
npm init playwright@latest
```

### 기본 테스트

```javascript
// e2e/login.spec.js
import { test, expect } from '@playwright/test';

test('로그인 플로우', async ({ page }) => {
    // 페이지 이동
    await page.goto('http://localhost:3000/login');

    // 폼 입력
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'password123');

    // 로그인 버튼 클릭
    await page.click('button[type="submit"]');

    // 리다이렉션 확인
    await expect(page).toHaveURL('http://localhost:3000/dashboard');

    // 사용자 이름 확인
    await expect(page.locator('h1')).toContainText('환영합니다');
});

test('로그인 실패', async ({ page }) => {
    await page.goto('http://localhost:3000/login');

    await page.fill('input[name="email"]', 'wrong@example.com');
    await page.fill('input[name="password"]', 'wrongpassword');
    await page.click('button[type="submit"]');

    // 에러 메시지 확인
    await expect(page.locator('.error')).toContainText('로그인 실패');
});
```

### 스크린샷과 비디오

```javascript
test('페이지 스크린샷', async ({ page }) => {
    await page.goto('http://localhost:3000');
    await page.screenshot({ path: 'screenshot.png' });
});

// playwright.config.js
export default {
    use: {
        screenshot: 'only-on-failure',
        video: 'retain-on-failure'
    }
};
```

## TDD (Test-Driven Development)

### TDD 사이클

1. **Red**: 실패하는 테스트 작성
2. **Green**: 테스트를 통과하는 최소 코드 작성
3. **Refactor**: 코드 개선

```javascript
// 1. Red: 실패하는 테스트
test('할 일 추가', () => {
    const todoList = new TodoList();
    todoList.add('공부하기');
    expect(todoList.getAll()).toEqual(['공부하기']);
});

// 2. Green: 통과하는 최소 코드
class TodoList {
    constructor() {
        this.items = [];
    }

    add(item) {
        this.items.push(item);
    }

    getAll() {
        return this.items;
    }
}

// 3. Refactor: 개선
class TodoList {
    constructor() {
        this.items = [];
    }

    add(item) {
        if (!item) throw new Error('할 일을 입력하세요');
        this.items.push(item);
    }

    getAll() {
        return [...this.items]; // 복사본 반환
    }
}
```

## 테스트 커버리지

```bash
# 커버리지 리포트
npm run test:coverage
```

```javascript
// jest.config.js
module.exports = {
    collectCoverageFrom: [
        'src/**/*.{js,jsx}',
        '!src/index.js',
        '!src/**/*.test.{js,jsx}'
    ],
    coverageThreshold: {
        global: {
            branches: 80,
            functions: 80,
            lines: 80,
            statements: 80
        }
    }
};
```

## 테스팅 모범 사례

### 1. AAA 패턴

```javascript
test('할 일 추가', () => {
    // Arrange (준비)
    const todoList = new TodoList();

    // Act (실행)
    todoList.add('공부하기');

    // Assert (검증)
    expect(todoList.getAll()).toContain('공부하기');
});
```

### 2. 하나의 테스트, 하나의 개념

```javascript
// ❌ 나쁜 예: 여러 개념 테스트
test('할 일 목록', () => {
    const list = new TodoList();
    list.add('공부');
    expect(list.getAll()).toHaveLength(1);
    list.remove(0);
    expect(list.getAll()).toHaveLength(0);
});

// ✅ 좋은 예: 개념별 분리
test('할 일 추가', () => {
    const list = new TodoList();
    list.add('공부');
    expect(list.getAll()).toHaveLength(1);
});

test('할 일 삭제', () => {
    const list = new TodoList();
    list.add('공부');
    list.remove(0);
    expect(list.getAll()).toHaveLength(0);
});
```

### 3. 구현이 아닌 동작 테스트

```javascript
// ❌ 구현 테스트 (취약)
test('버튼 클래스 확인', () => {
    render(<Button>클릭</Button>);
    expect(screen.getByText('클릭')).toHaveClass('btn-primary');
});

// ✅ 동작 테스트 (견고)
test('버튼 클릭 시 핸들러 호출', async () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>클릭</Button>);
    await userEvent.click(screen.getByText('클릭'));
    expect(handleClick).toHaveBeenCalled();
});
```

## 핵심 요약

### 1. Jest 기본
```javascript
test('테스트 이름', () => {
    expect(value).toBe(expected)
})
```

### 2. React Testing Library
```javascript
render(<Component />)
const element = screen.getByText('text')
await userEvent.click(element)
```

### 3. 테스트 원칙
- AAA 패턴 (Arrange, Act, Assert)
- 하나의 테스트, 하나의 개념
- 동작 테스트, 구현 아님
- 독립적인 테스트

### 4. 커버리지 목표
- 핵심 로직: 100%
- UI 컴포넌트: 80%
- 유틸리티: 100%

## 다음 단계
부록 D에서 학습 리소스를 확인합니다.