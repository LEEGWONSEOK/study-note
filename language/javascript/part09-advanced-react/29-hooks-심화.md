# 29. Hooks 심화

## 학습 목표
- useMemo와 useCallback의 차이와 사용법 마스터하기
- useRef의 다양한 활용 패턴 익히기
- 커스텀 Hook 작성 방법 학습하기
- Hook 규칙과 고급 패턴 이해하기

## useMemo: 값 메모이제이션

`useMemo`는 **비싼 계산의 결과를 메모이제이션**합니다.

### 비유로 이해하기
useMemo는 **계산기의 메모리 기능**과 같습니다:
- 복잡한 계산을 한 번 수행
- 결과를 저장해둠
- 입력값이 같으면 저장된 결과 재사용
- 입력값이 바뀌면 다시 계산

### 기본 사용법

```javascript
import { useMemo, useState } from 'react';

function ExpensiveComponent() {
    const [count, setCount] = useState(0);
    const [text, setText] = useState('');

    // ❌ 매 렌더링마다 계산 (비효율적)
    const expensiveValue = calculateExpensiveValue(count);

    // ✅ count가 변경될 때만 계산
    const memoizedValue = useMemo(() => {
        console.log('계산 중...');
        return calculateExpensiveValue(count);
    }, [count]); // count가 변경될 때만 재계산

    return (
        <div>
            <p>결과: {memoizedValue}</p>
            <button onClick={() => setCount(count + 1)}>
                Count: {count}
            </button>

            {/* text 변경 시에는 재계산하지 않음 */}
            <input
                value={text}
                onChange={(e) => setText(e.target.value)}
            />
        </div>
    );
}

function calculateExpensiveValue(num) {
    // 비용이 많이 드는 계산
    let result = 0;
    for (let i = 0; i < 1000000000; i++) {
        result += num;
    }
    return result;
}
```

### useMemo 사용 예시

```javascript
// 1. 필터링/정렬 등 배열 연산
function FilteredList({ items, searchQuery }) {
    // searchQuery가 변경될 때만 필터링
    const filteredItems = useMemo(() => {
        console.log('필터링 중...');
        return items.filter(item =>
            item.name.toLowerCase().includes(searchQuery.toLowerCase())
        );
    }, [items, searchQuery]);

    return (
        <ul>
            {filteredItems.map(item => (
                <li key={item.id}>{item.name}</li>
            ))}
        </ul>
    );
}

// 2. 복잡한 객체 생성
function ChartComponent({ data }) {
    // data가 변경될 때만 차트 옵션 생성
    const chartOptions = useMemo(() => {
        return {
            data: processChartData(data),
            colors: generateColors(data),
            labels: generateLabels(data)
        };
    }, [data]);

    return <Chart options={chartOptions} />;
}

// 3. 통계 계산
function Statistics({ numbers }) {
    const stats = useMemo(() => {
        const sum = numbers.reduce((a, b) => a + b, 0);
        const avg = sum / numbers.length;
        const max = Math.max(...numbers);
        const min = Math.min(...numbers);

        return { sum, avg, max, min };
    }, [numbers]);

    return (
        <div>
            <p>합계: {stats.sum}</p>
            <p>평균: {stats.avg}</p>
            <p>최대: {stats.max}</p>
            <p>최소: {stats.min}</p>
        </div>
    );
}
```

## useCallback: 함수 메모이제이션

`useCallback`은 **함수를 메모이제이션**합니다.

### useMemo vs useCallback

```javascript
// useMemo: 값을 메모이제이션
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);

// useCallback: 함수를 메모이제이션
const memoizedCallback = useCallback(() => {
    doSomething(a, b);
}, [a, b]);

// 사실 useCallback은 useMemo의 특수한 경우
const memoizedCallback = useMemo(() => {
    return () => doSomething(a, b);
}, [a, b]);
```

### useCallback 사용법

```javascript
function TodoList() {
    const [todos, setTodos] = useState([]);

    // ❌ 매 렌더링마다 새로운 함수 생성
    const handleDelete = (id) => {
        setTodos(todos.filter(todo => todo.id !== id));
    };

    // ✅ 함수 메모이제이션
    const handleDeleteOptimized = useCallback((id) => {
        setTodos(prev => prev.filter(todo => todo.id !== id));
    }, []); // 의존성 없음

    return (
        <ul>
            {todos.map(todo => (
                <TodoItem
                    key={todo.id}
                    todo={todo}
                    onDelete={handleDeleteOptimized}
                />
            ))}
        </ul>
    );
}

// React.memo와 함께 사용하면 효과적
const TodoItem = React.memo(({ todo, onDelete }) => {
    console.log('렌더링:', todo.id);

    return (
        <li>
            {todo.text}
            <button onClick={() => onDelete(todo.id)}>
                삭제
            </button>
        </li>
    );
});
```

### useCallback 실전 예제

```javascript
function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);

    // API 호출 함수 메모이제이션
    const fetchResults = useCallback(async (searchQuery) => {
        if (!searchQuery) return;

        const response = await fetch(`/api/search?q=${searchQuery}`);
        const data = await response.json();
        setResults(data);
    }, []);

    // 디바운스된 검색
    useEffect(() => {
        const timer = setTimeout(() => {
            fetchResults(query);
        }, 500);

        return () => clearTimeout(timer);
    }, [query, fetchResults]);

    return (
        <div>
            <input
                value={query}
                onChange={(e) => setQuery(e.target.value)}
                placeholder="검색..."
            />
            {/* 결과 표시 */}
        </div>
    );
}

// 이벤트 핸들러 메모이제이션
function Form() {
    const [formData, setFormData] = useState({
        name: '',
        email: '',
        message: ''
    });

    // 각 필드마다 함수를 생성하지 않고 하나만 사용
    const handleChange = useCallback((e) => {
        const { name, value } = e.target;
        setFormData(prev => ({
            ...prev,
            [name]: value
        }));
    }, []);

    const handleSubmit = useCallback((e) => {
        e.preventDefault();
        console.log('제출:', formData);
    }, [formData]);

    return (
        <form onSubmit={handleSubmit}>
            <input
                name="name"
                value={formData.name}
                onChange={handleChange}
            />
            <input
                name="email"
                value={formData.email}
                onChange={handleChange}
            />
            <textarea
                name="message"
                value={formData.message}
                onChange={handleChange}
            />
            <button type="submit">전송</button>
        </form>
    );
}
```

## useRef: 참조 관리

`useRef`는 **리렌더링되지 않는 값을 저장**하거나 **DOM 요소에 접근**합니다.

### useRef의 두 가지 용도

```javascript
// 1. DOM 요소 접근
function FocusInput() {
    const inputRef = useRef(null);

    const handleFocus = () => {
        inputRef.current.focus();
    };

    return (
        <div>
            <input ref={inputRef} />
            <button onClick={handleFocus}>
                포커스
            </button>
        </div>
    );
}

// 2. 변경 가능한 값 저장 (리렌더링 없이)
function Timer() {
    const [count, setCount] = useState(0);
    const intervalRef = useRef(null);

    const startTimer = () => {
        intervalRef.current = setInterval(() => {
            setCount(prev => prev + 1);
        }, 1000);
    };

    const stopTimer = () => {
        clearInterval(intervalRef.current);
    };

    // Cleanup
    useEffect(() => {
        return () => {
            if (intervalRef.current) {
                clearInterval(intervalRef.current);
            }
        };
    }, []);

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={startTimer}>시작</button>
            <button onClick={stopTimer}>정지</button>
        </div>
    );
}
```

### useRef vs useState

```javascript
function RefVsState() {
    const [stateCount, setStateCount] = useState(0);
    const refCount = useRef(0);

    const handleStateClick = () => {
        setStateCount(stateCount + 1); // 리렌더링 발생
        console.log('State:', stateCount);
    };

    const handleRefClick = () => {
        refCount.current += 1; // 리렌더링 없음
        console.log('Ref:', refCount.current);
    };

    console.log('렌더링됨');

    return (
        <div>
            <p>State Count: {stateCount}</p>
            <p>Ref Count: {refCount.current}</p>
            <button onClick={handleStateClick}>
                State 증가 (리렌더링)
            </button>
            <button onClick={handleRefClick}>
                Ref 증가 (리렌더링 없음)
            </button>
        </div>
    );
}
```

### useRef 실전 활용

```javascript
// 1. 이전 값 추적
function usePrevious(value) {
    const ref = useRef();

    useEffect(() => {
        ref.current = value;
    }, [value]);

    return ref.current;
}

function Counter() {
    const [count, setCount] = useState(0);
    const prevCount = usePrevious(count);

    return (
        <div>
            <p>현재: {count}</p>
            <p>이전: {prevCount}</p>
            <button onClick={() => setCount(count + 1)}>
                증가
            </button>
        </div>
    );
}

// 2. 스크롤 위치 저장
function ScrollableList({ items }) {
    const listRef = useRef(null);
    const scrollPositionRef = useRef(0);

    const saveScrollPosition = () => {
        if (listRef.current) {
            scrollPositionRef.current = listRef.current.scrollTop;
        }
    };

    const restoreScrollPosition = () => {
        if (listRef.current) {
            listRef.current.scrollTop = scrollPositionRef.current;
        }
    };

    useEffect(() => {
        restoreScrollPosition();
    });

    return (
        <div
            ref={listRef}
            onScroll={saveScrollPosition}
            style={{ height: '400px', overflow: 'auto' }}
        >
            {items.map(item => (
                <div key={item.id}>{item.name}</div>
            ))}
        </div>
    );
}

// 3. 렌더링 횟수 추적
function RenderCounter() {
    const renderCount = useRef(0);

    useEffect(() => {
        renderCount.current += 1;
    });

    return <div>렌더링 횟수: {renderCount.current}</div>;
}

// 4. Debounce 구현
function useDebounce(callback, delay) {
    const timeoutRef = useRef(null);

    return useCallback((...args) => {
        if (timeoutRef.current) {
            clearTimeout(timeoutRef.current);
        }

        timeoutRef.current = setTimeout(() => {
            callback(...args);
        }, delay);
    }, [callback, delay]);
}

function SearchWithDebounce() {
    const [query, setQuery] = useState('');

    const search = (value) => {
        console.log('검색:', value);
        // API 호출
    };

    const debouncedSearch = useDebounce(search, 500);

    const handleChange = (e) => {
        const value = e.target.value;
        setQuery(value);
        debouncedSearch(value);
    };

    return (
        <input
            value={query}
            onChange={handleChange}
            placeholder="검색..."
        />
    );
}
```

## 커스텀 Hook

커스텀 Hook은 **로직을 재사용 가능한 함수로 추출**하는 방법입니다.

### 커스텀 Hook 작성 규칙

```javascript
// 1. 이름은 "use"로 시작
// 2. 다른 Hook을 호출할 수 있음
// 3. 일반 함수처럼 매개변수와 반환값 사용

function useCustomHook(initialValue) {
    const [value, setValue] = useState(initialValue);
    // ... 로직

    return [value, setValue];
}
```

### 실용적인 커스텀 Hook 예제

```javascript
// 1. useLocalStorage: 로컬 스토리지 동기화
function useLocalStorage(key, initialValue) {
    const [storedValue, setStoredValue] = useState(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            console.error(error);
            return initialValue;
        }
    });

    const setValue = (value) => {
        try {
            const valueToStore =
                value instanceof Function ? value(storedValue) : value;

            setStoredValue(valueToStore);
            window.localStorage.setItem(key, JSON.stringify(valueToStore));
        } catch (error) {
            console.error(error);
        }
    };

    return [storedValue, setValue];
}

// 사용
function App() {
    const [name, setName] = useLocalStorage('name', '');

    return (
        <input
            value={name}
            onChange={(e) => setName(e.target.value)}
        />
    );
}

// 2. useFetch: 데이터 가져오기
function useFetch(url) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        const fetchData = async () => {
            try {
                setLoading(true);
                const response = await fetch(url);
                const json = await response.json();
                setData(json);
                setError(null);
            } catch (err) {
                setError(err);
            } finally {
                setLoading(false);
            }
        };

        fetchData();
    }, [url]);

    return { data, loading, error };
}

// 사용
function UserProfile({ userId }) {
    const { data, loading, error } = useFetch(`/api/users/${userId}`);

    if (loading) return <div>로딩 중...</div>;
    if (error) return <div>에러: {error.message}</div>;
    return <div>{data.name}</div>;
}

// 3. useToggle: 불리언 토글
function useToggle(initialValue = false) {
    const [value, setValue] = useState(initialValue);

    const toggle = useCallback(() => {
        setValue(v => !v);
    }, []);

    const setTrue = useCallback(() => {
        setValue(true);
    }, []);

    const setFalse = useCallback(() => {
        setValue(false);
    }, []);

    return [value, toggle, setTrue, setFalse];
}

// 사용
function Modal() {
    const [isOpen, toggle, open, close] = useToggle(false);

    return (
        <>
            <button onClick={open}>열기</button>
            {isOpen && (
                <div className="modal">
                    <p>모달 내용</p>
                    <button onClick={close}>닫기</button>
                </div>
            )}
        </>
    );
}

// 4. useDebounce: 값 디바운싱
function useDebounce(value, delay) {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        const timer = setTimeout(() => {
            setDebouncedValue(value);
        }, delay);

        return () => {
            clearTimeout(timer);
        };
    }, [value, delay]);

    return debouncedValue;
}

// 사용
function SearchComponent() {
    const [query, setQuery] = useState('');
    const debouncedQuery = useDebounce(query, 500);

    useEffect(() => {
        if (debouncedQuery) {
            // API 호출
            console.log('검색:', debouncedQuery);
        }
    }, [debouncedQuery]);

    return (
        <input
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            placeholder="검색..."
        />
    );
}

// 5. useWindowSize: 윈도우 크기 추적
function useWindowSize() {
    const [size, setSize] = useState({
        width: window.innerWidth,
        height: window.innerHeight
    });

    useEffect(() => {
        const handleResize = () => {
            setSize({
                width: window.innerWidth,
                height: window.innerHeight
            });
        };

        window.addEventListener('resize', handleResize);
        return () => window.removeEventListener('resize', handleResize);
    }, []);

    return size;
}

// 사용
function ResponsiveComponent() {
    const { width } = useWindowSize();

    return (
        <div>
            {width < 768 ? (
                <MobileView />
            ) : (
                <DesktopView />
            )}
        </div>
    );
}

// 6. useOnClickOutside: 외부 클릭 감지
function useOnClickOutside(ref, handler) {
    useEffect(() => {
        const listener = (event) => {
            if (!ref.current || ref.current.contains(event.target)) {
                return;
            }
            handler(event);
        };

        document.addEventListener('mousedown', listener);
        document.addEventListener('touchstart', listener);

        return () => {
            document.removeEventListener('mousedown', listener);
            document.removeEventListener('touchstart', listener);
        };
    }, [ref, handler]);
}

// 사용
function Dropdown() {
    const [isOpen, setIsOpen] = useState(false);
    const dropdownRef = useRef(null);

    useOnClickOutside(dropdownRef, () => setIsOpen(false));

    return (
        <div ref={dropdownRef}>
            <button onClick={() => setIsOpen(!isOpen)}>
                메뉴 열기
            </button>
            {isOpen && (
                <ul className="dropdown">
                    <li>항목 1</li>
                    <li>항목 2</li>
                </ul>
            )}
        </div>
    );
}
```

## Hook 규칙

### Rules of Hooks

```javascript
// ✅ 올바른 사용
function GoodComponent() {
    const [count, setCount] = useState(0);
    const [name, setName] = useState('');

    useEffect(() => {
        console.log('Effect');
    }, []);

    return <div>{count}</div>;
}

// ❌ 조건문 안에서 Hook 호출
function BadComponent1({ shouldUseEffect }) {
    const [count, setCount] = useState(0);

    if (shouldUseEffect) {
        useEffect(() => { // 에러!
            console.log('Effect');
        }, []);
    }

    return <div>{count}</div>;
}

// ❌ 반복문 안에서 Hook 호출
function BadComponent2() {
    for (let i = 0; i < 5; i++) {
        const [count, setCount] = useState(0); // 에러!
    }

    return <div>Bad</div>;
}

// ❌ 일반 함수에서 Hook 호출
function normalFunction() {
    const [count, setCount] = useState(0); // 에러!
}

// ✅ 커스텀 Hook에서는 가능
function useCustomHook() {
    const [count, setCount] = useState(0); // OK
    return count;
}

// ✅ 조건부 로직은 Hook 내부에서
function GoodComponent2({ shouldLog }) {
    const [count, setCount] = useState(0);

    useEffect(() => {
        if (shouldLog) { // 조건은 안에서!
            console.log('Effect');
        }
    }, [shouldLog]);

    return <div>{count}</div>;
}
```

## Java 개발자를 위한 비교

```java
// Java: 캐싱 메커니즘
public class Calculator {
    private Map<Integer, Integer> cache = new HashMap<>();

    public int calculate(int input) {
        if (cache.containsKey(input)) {
            return cache.get(input); // 캐시된 값 반환
        }

        int result = expensiveOperation(input);
        cache.put(input, result); // 결과 캐싱
        return result;
    }

    private int expensiveOperation(int input) {
        // 비용이 큰 연산
        return input * input;
    }
}
```

```javascript
// React: useMemo로 메모이제이션
function Calculator({ input }) {
    const result = useMemo(() => {
        return expensiveOperation(input);
    }, [input]); // input이 변경될 때만 재계산

    return <div>결과: {result}</div>;
}

function expensiveOperation(input) {
    return input * input;
}

// 차이점:
// Java: 명시적으로 캐시 관리
// React: 선언적으로 의존성만 지정
```

## 핵심 요약

### 1. 메모이제이션 Hook
```javascript
// 값 메모이제이션
const value = useMemo(() => compute(), [deps]);

// 함수 메모이제이션
const callback = useCallback(() => {}, [deps]);
```

### 2. useRef 용도
- DOM 요소 접근
- 리렌더링 없이 값 저장
- 이전 값 추적
- 타이머/인터벌 참조

### 3. 커스텀 Hook
- "use"로 시작
- 로직 재사용
- 다른 Hook 조합 가능
- 컴포넌트처럼 사용

### 4. Hook 규칙
- 최상위에서만 호출
- React 함수에서만 호출
- 조건문/반복문 안에서 호출 금지

## 연습 문제

### 문제 1: useAsync Hook
비동기 작업을 처리하는 커스텀 Hook을 만들어보세요.

```javascript
// 요구사항:
// - 로딩, 에러, 데이터 상태 관리
// - 재시도 기능
// - 취소 기능

function useAsync(asyncFunction) {
    // 여기에 구현
}

// 사용 예시
function Component() {
    const { data, loading, error, retry } = useAsync(() =>
        fetch('/api/data').then(res => res.json())
    );

    if (loading) return <div>로딩 중...</div>;
    if (error) return <button onClick={retry}>재시도</button>;
    return <div>{data}</div>;
}
```

### 문제 2: useForm Hook
폼 상태를 관리하는 커스텀 Hook을 만들어보세요.

```javascript
// 요구사항:
// - 입력값 관리
// - 유효성 검사
// - 에러 메시지
// - 제출 처리

function useForm(initialValues, validate) {
    // 여기에 구현
}

// 사용 예시
function LoginForm() {
    const { values, errors, handleChange, handleSubmit } = useForm(
        { email: '', password: '' },
        (values) => {
            const errors = {};
            if (!values.email) errors.email = '이메일을 입력하세요';
            return errors;
        }
    );

    return (
        <form onSubmit={handleSubmit}>
            {/* ... */}
        </form>
    );
}
```

### 문제 3: useIntersectionObserver Hook
요소가 뷰포트에 보이는지 감지하는 Hook을 만들어보세요.

```javascript
// 요구사항:
// - Intersection Observer API 사용
// - threshold 옵션 지원
// - 정리(cleanup) 구현

function useIntersectionObserver(options) {
    // 여기에 구현
}

// 사용 예시
function LazyImage({ src }) {
    const [ref, isVisible] = useIntersectionObserver({
        threshold: 0.1
    });

    return (
        <div ref={ref}>
            {isVisible && <img src={src} />}
        </div>
    );
}
```

### 문제 4: useInterval Hook
setInterval을 Hook으로 만들어보세요.

```javascript
// 요구사항:
// - 일시정지/재개 기능
// - 딜레이 변경 가능
// - 자동 정리(cleanup)

function useInterval(callback, delay) {
    // 여기에 구현
}

// 사용 예시
function Timer() {
    const [count, setCount] = useState(0);

    useInterval(() => {
        setCount(count + 1);
    }, 1000);

    return <div>{count}초</div>;
}
```

## 다음 단계
다음 장에서는 **Context API**를 학습합니다:
- Context 생성과 사용
- Provider와 Consumer
- useContext Hook
- Context 최적화 패턴