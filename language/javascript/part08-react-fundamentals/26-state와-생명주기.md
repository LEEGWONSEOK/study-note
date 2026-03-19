# 26. State와 생명주기

## 학습 목표
- React State의 개념과 사용법 이해하기
- useState Hook 마스터하기
- 컴포넌트 생명주기와 useEffect 학습하기
- State 업데이트 패턴과 최적화 기법 익히기

## State란?

State는 **컴포넌트가 기억하고 관리하는 동적 데이터**입니다.

### 비유로 이해하기
State는 **메모장**과 같습니다:
- 중요한 정보를 기록해둠 (값 저장)
- 필요할 때 내용을 확인함 (값 읽기)
- 내용이 바뀌면 메모를 업데이트함 (값 변경)
- 메모가 바뀌면 그에 따라 행동이 바뀜 (리렌더링)

### Props vs State

```javascript
// Props: 부모로부터 받은 읽기 전용 데이터
function Child({ name }) {
    // name은 Props: 변경 불가
    // name = "새이름"; // ❌ 에러!
    return <div>안녕, {name}</div>;
}

// State: 컴포넌트가 관리하는 변경 가능한 데이터
function Counter() {
    // count는 State: 변경 가능
    const [count, setCount] = useState(0);

    return (
        <div>
            <p>카운트: {count}</p>
            {/* ✅ State 변경 가능 */}
            <button onClick={() => setCount(count + 1)}>
                증가
            </button>
        </div>
    );
}
```

### Java와 비교

```java
// Java: 클래스 인스턴스 변수로 상태 관리
public class Counter extends JFrame {
    private int count = 0;
    private JLabel label;

    public Counter() {
        label = new JLabel("카운트: " + count);
        JButton button = new JButton("증가");

        button.addActionListener(e -> {
            count++;  // 상태 변경
            label.setText("카운트: " + count);  // UI 수동 업데이트
        });
    }
}
```

```javascript
// React: useState Hook으로 상태 관리
function Counter() {
    const [count, setCount] = useState(0);

    return (
        <div>
            <p>카운트: {count}</p>
            {/* State 변경하면 자동으로 리렌더링 */}
            <button onClick={() => setCount(count + 1)}>
                증가
            </button>
        </div>
    );
}

// 차이점:
// Java: 상태 변경 후 UI 수동 업데이트 필요
// React: 상태 변경하면 자동으로 리렌더링됨
```

## useState Hook

`useState`는 **함수형 컴포넌트에서 상태를 관리**하는 Hook입니다.

### 기본 사용법

```javascript
import { useState } from 'react';

function Example() {
    // [현재값, 변경함수] = useState(초기값)
    const [state, setState] = useState(initialValue);

    return <div>{state}</div>;
}

// 다양한 타입의 State
function MultipleStates() {
    const [count, setCount] = useState(0);              // 숫자
    const [text, setText] = useState('');               // 문자열
    const [isOpen, setIsOpen] = useState(false);        // 불리언
    const [items, setItems] = useState([]);             // 배열
    const [user, setUser] = useState(null);             // 객체
    const [data, setData] = useState({ name: '', age: 0 }); // 객체

    return <div>/* ... */</div>;
}
```

### State 업데이트 방법

```javascript
function Counter() {
    const [count, setCount] = useState(0);

    // 방법 1: 새 값 직접 전달
    const increment = () => {
        setCount(count + 1);
    };

    // 방법 2: 함수형 업데이트 (이전 값 기반)
    const incrementFunc = () => {
        setCount(prevCount => prevCount + 1);
    };

    // 여러 번 업데이트할 때 차이
    const addThree = () => {
        // ❌ 잘못된 방법: 1만 증가함
        setCount(count + 1);
        setCount(count + 1);
        setCount(count + 1);

        // ✅ 올바른 방법: 3 증가함
        setCount(prev => prev + 1);
        setCount(prev => prev + 1);
        setCount(prev => prev + 1);
    };

    return (
        <div>
            <p>카운트: {count}</p>
            <button onClick={increment}>+1</button>
            <button onClick={incrementFunc}>+1 (함수형)</button>
            <button onClick={addThree}>+3</button>
        </div>
    );
}
```

### 객체와 배열 State 관리

```javascript
function UserForm() {
    const [user, setUser] = useState({
        name: '',
        email: '',
        age: 0
    });

    // ❌ 잘못된 방법: 직접 수정
    const wrongUpdate = () => {
        user.name = '김철수'; // 동작하지 않음!
        setUser(user); // 같은 참조라 리렌더링 안 됨
    };

    // ✅ 올바른 방법 1: 전개 연산자
    const updateName = (newName) => {
        setUser({
            ...user,
            name: newName
        });
    };

    // ✅ 올바른 방법 2: 함수형 업데이트
    const updateEmail = (newEmail) => {
        setUser(prev => ({
            ...prev,
            email: newEmail
        }));
    };

    return (
        <form>
            <input
                value={user.name}
                onChange={(e) => updateName(e.target.value)}
                placeholder="이름"
            />
            <input
                value={user.email}
                onChange={(e) => updateEmail(e.target.value)}
                placeholder="이메일"
            />
            <input
                type="number"
                value={user.age}
                onChange={(e) => setUser({ ...user, age: Number(e.target.value) })}
                placeholder="나이"
            />
        </form>
    );
}
```

```javascript
function TodoList() {
    const [todos, setTodos] = useState([]);

    // 추가
    const addTodo = (text) => {
        const newTodo = {
            id: Date.now(),
            text,
            completed: false
        };

        setTodos([...todos, newTodo]);
        // 또는: setTodos(prev => [...prev, newTodo]);
    };

    // 수정
    const toggleTodo = (id) => {
        setTodos(todos.map(todo =>
            todo.id === id
                ? { ...todo, completed: !todo.completed }
                : todo
        ));
    };

    // 삭제
    const deleteTodo = (id) => {
        setTodos(todos.filter(todo => todo.id !== id));
    };

    // 전체 삭제
    const clearCompleted = () => {
        setTodos(todos.filter(todo => !todo.completed));
    };

    return (
        <div>
            {todos.map(todo => (
                <div key={todo.id}>
                    <input
                        type="checkbox"
                        checked={todo.completed}
                        onChange={() => toggleTodo(todo.id)}
                    />
                    <span>{todo.text}</span>
                    <button onClick={() => deleteTodo(todo.id)}>
                        삭제
                    </button>
                </div>
            ))}
        </div>
    );
}
```

### 복잡한 State 관리

```javascript
// 중첩된 객체 State
function NestedState() {
    const [data, setData] = useState({
        user: {
            profile: {
                name: '',
                avatar: ''
            },
            settings: {
                theme: 'light',
                notifications: true
            }
        }
    });

    // 깊은 중첩 업데이트
    const updateName = (newName) => {
        setData({
            ...data,
            user: {
                ...data.user,
                profile: {
                    ...data.user.profile,
                    name: newName
                }
            }
        });
    };

    // Immer 라이브러리 사용하면 더 간단 (나중에 학습)
    return <div>{data.user.profile.name}</div>;
}
```

## 생명주기 (Lifecycle)

컴포넌트의 **생명주기**는 탄생부터 소멸까지의 과정입니다.

### 비유로 이해하기
컴포넌트 생명주기는 **사람의 인생**과 같습니다:
- 탄생 (Mount): 화면에 처음 나타남
- 성장/변화 (Update): State나 Props가 변경됨
- 죽음 (Unmount): 화면에서 사라짐

### 생명주기 단계

```javascript
function Lifecycle() {
    const [count, setCount] = useState(0);

    // 1. Mount: 컴포넌트가 처음 나타날 때
    useEffect(() => {
        console.log('컴포넌트가 마운트되었습니다');

        // 3. Unmount: 컴포넌트가 사라질 때
        return () => {
            console.log('컴포넌트가 언마운트되었습니다');
        };
    }, []); // 빈 배열: 마운트/언마운트 시에만

    // 2. Update: count가 변경될 때마다
    useEffect(() => {
        console.log('count가 변경되었습니다:', count);
    }, [count]); // count를 의존성 배열에 추가

    // 모든 렌더링마다 실행
    useEffect(() => {
        console.log('렌더링되었습니다');
    }); // 의존성 배열 없음

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => setCount(count + 1)}>
                증가
            </button>
        </div>
    );
}
```

### useEffect Hook

`useEffect`는 **부수 효과(side effects)를 처리**하는 Hook입니다.

```javascript
// 기본 구조
useEffect(() => {
    // 실행할 코드 (effect)

    return () => {
        // 정리 코드 (cleanup)
    };
}, [dependencies]); // 의존성 배열
```

### useEffect 사용 예시

```javascript
// 1. 데이터 가져오기
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        // userId가 변경될 때마다 데이터 가져오기
        setLoading(true);

        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => {
                setUser(data);
                setLoading(false);
            });
    }, [userId]); // userId가 변경되면 다시 실행

    if (loading) return <div>로딩 중...</div>;
    return <div>{user.name}</div>;
}

// 2. 타이머
function Timer() {
    const [seconds, setSeconds] = useState(0);

    useEffect(() => {
        const interval = setInterval(() => {
            setSeconds(prev => prev + 1);
        }, 1000);

        // Cleanup: 컴포넌트가 언마운트되면 타이머 정리
        return () => {
            clearInterval(interval);
        };
    }, []); // 빈 배열: 마운트 시 한 번만

    return <div>{seconds}초</div>;
}

// 3. 이벤트 리스너
function WindowSize() {
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

        // Cleanup: 이벤트 리스너 제거
        return () => {
            window.removeEventListener('resize', handleResize);
        };
    }, []);

    return (
        <div>
            {size.width} x {size.height}
        </div>
    );
}

// 4. 로컬 스토리지 동기화
function PersistentCounter() {
    const [count, setCount] = useState(() => {
        // 초기값을 로컬 스토리지에서 가져오기
        const saved = localStorage.getItem('count');
        return saved ? Number(saved) : 0;
    });

    useEffect(() => {
        // count가 변경될 때마다 로컬 스토리지에 저장
        localStorage.setItem('count', count);
    }, [count]);

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => setCount(count + 1)}>
                증가
            </button>
        </div>
    );
}

// 5. 문서 제목 업데이트
function PageTitle({ title }) {
    useEffect(() => {
        document.title = title;

        // Cleanup: 원래 제목으로 복원
        return () => {
            document.title = 'React App';
        };
    }, [title]);

    return <h1>{title}</h1>;
}
```

### 의존성 배열 (Dependency Array)

```javascript
function DependencyExample() {
    const [count, setCount] = useState(0);
    const [name, setName] = useState('');

    // 1. 의존성 배열 없음: 매 렌더링마다 실행
    useEffect(() => {
        console.log('매번 실행됨');
    });

    // 2. 빈 배열: 마운트 시 한 번만
    useEffect(() => {
        console.log('마운트 시 한 번만');
    }, []);

    // 3. count만 의존: count 변경 시에만
    useEffect(() => {
        console.log('count 변경:', count);
    }, [count]);

    // 4. 여러 의존성: 둘 중 하나라도 변경되면
    useEffect(() => {
        console.log('count 또는 name 변경');
    }, [count, name]);

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => setCount(count + 1)}>증가</button>
            <input
                value={name}
                onChange={(e) => setName(e.target.value)}
            />
        </div>
    );
}
```

### useEffect 주의사항

```javascript
function EffectPitfalls() {
    const [count, setCount] = useState(0);

    // ❌ 무한 루프!
    useEffect(() => {
        setCount(count + 1); // count 변경
        // count가 변경되면 useEffect 다시 실행
        // 무한 반복!
    }, [count]);

    // ❌ 의존성 배열에 빠진 값
    useEffect(() => {
        console.log(count); // count 사용
        // 하지만 의존성 배열에 없음 -> 최신 count 사용 안 함
    }, []); // 경고 발생!

    // ✅ 올바른 방법
    useEffect(() => {
        console.log(count);
    }, [count]); // 사용하는 모든 값 포함

    // ❌ 비동기 함수를 직접 useEffect에 사용
    useEffect(async () => {
        // 에러 발생!
        const data = await fetch('/api');
    }, []);

    // ✅ 올바른 방법: 내부에서 비동기 함수 정의
    useEffect(() => {
        const fetchData = async () => {
            const data = await fetch('/api');
        };

        fetchData();
    }, []);

    return <div>{count}</div>;
}
```

## 실전 예제

### 1. 검색 기능

```javascript
function SearchBox() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);

    useEffect(() => {
        // 빈 검색어는 무시
        if (!query) {
            setResults([]);
            return;
        }

        // 디바운싱: 입력 후 500ms 대기
        const timer = setTimeout(() => {
            setLoading(true);

            fetch(`/api/search?q=${query}`)
                .then(res => res.json())
                .then(data => {
                    setResults(data);
                    setLoading(false);
                });
        }, 500);

        // Cleanup: 타이머 취소
        return () => clearTimeout(timer);
    }, [query]);

    return (
        <div>
            <input
                value={query}
                onChange={(e) => setQuery(e.target.value)}
                placeholder="검색..."
            />

            {loading && <p>검색 중...</p>}

            <ul>
                {results.map(result => (
                    <li key={result.id}>{result.title}</li>
                ))}
            </ul>
        </div>
    );
}
```

### 2. 폼 관리

```javascript
function ContactForm() {
    const [formData, setFormData] = useState({
        name: '',
        email: '',
        message: ''
    });

    const [errors, setErrors] = useState({});
    const [submitted, setSubmitted] = useState(false);

    // 입력 핸들러
    const handleChange = (e) => {
        const { name, value } = e.target;

        setFormData(prev => ({
            ...prev,
            [name]: value
        }));

        // 에러 제거
        if (errors[name]) {
            setErrors(prev => ({
                ...prev,
                [name]: ''
            }));
        }
    };

    // 유효성 검사
    const validate = () => {
        const newErrors = {};

        if (!formData.name) {
            newErrors.name = '이름을 입력하세요';
        }

        if (!formData.email) {
            newErrors.email = '이메일을 입력하세요';
        } else if (!/\S+@\S+\.\S+/.test(formData.email)) {
            newErrors.email = '유효한 이메일을 입력하세요';
        }

        if (!formData.message) {
            newErrors.message = '메시지를 입력하세요';
        }

        return newErrors;
    };

    // 제출 핸들러
    const handleSubmit = (e) => {
        e.preventDefault();

        const newErrors = validate();

        if (Object.keys(newErrors).length > 0) {
            setErrors(newErrors);
            return;
        }

        // API 호출
        fetch('/api/contact', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(formData)
        })
            .then(() => {
                setSubmitted(true);
                setFormData({ name: '', email: '', message: '' });
            });
    };

    if (submitted) {
        return <div>메시지가 전송되었습니다!</div>;
    }

    return (
        <form onSubmit={handleSubmit}>
            <div>
                <input
                    name="name"
                    value={formData.name}
                    onChange={handleChange}
                    placeholder="이름"
                />
                {errors.name && <span className="error">{errors.name}</span>}
            </div>

            <div>
                <input
                    name="email"
                    value={formData.email}
                    onChange={handleChange}
                    placeholder="이메일"
                />
                {errors.email && <span className="error">{errors.email}</span>}
            </div>

            <div>
                <textarea
                    name="message"
                    value={formData.message}
                    onChange={handleChange}
                    placeholder="메시지"
                />
                {errors.message && <span className="error">{errors.message}</span>}
            </div>

            <button type="submit">전송</button>
        </form>
    );
}
```

### 3. 장바구니

```javascript
function ShoppingCart() {
    const [cart, setCart] = useState([]);

    // 로컬 스토리지에서 로드
    useEffect(() => {
        const saved = localStorage.getItem('cart');
        if (saved) {
            setCart(JSON.parse(saved));
        }
    }, []);

    // 로컬 스토리지에 저장
    useEffect(() => {
        localStorage.setItem('cart', JSON.stringify(cart));
    }, [cart]);

    // 상품 추가
    const addItem = (product) => {
        setCart(prev => {
            const existing = prev.find(item => item.id === product.id);

            if (existing) {
                // 이미 있으면 수량 증가
                return prev.map(item =>
                    item.id === product.id
                        ? { ...item, quantity: item.quantity + 1 }
                        : item
                );
            } else {
                // 새 상품 추가
                return [...prev, { ...product, quantity: 1 }];
            }
        });
    };

    // 수량 변경
    const updateQuantity = (id, quantity) => {
        if (quantity <= 0) {
            removeItem(id);
            return;
        }

        setCart(prev =>
            prev.map(item =>
                item.id === id
                    ? { ...item, quantity }
                    : item
            )
        );
    };

    // 상품 제거
    const removeItem = (id) => {
        setCart(prev => prev.filter(item => item.id !== id));
    };

    // 전체 삭제
    const clearCart = () => {
        setCart([]);
    };

    // 총액 계산
    const total = cart.reduce(
        (sum, item) => sum + item.price * item.quantity,
        0
    );

    return (
        <div>
            <h2>장바구니</h2>

            {cart.length === 0 ? (
                <p>장바구니가 비어있습니다</p>
            ) : (
                <>
                    {cart.map(item => (
                        <div key={item.id}>
                            <span>{item.name}</span>
                            <input
                                type="number"
                                value={item.quantity}
                                onChange={(e) =>
                                    updateQuantity(item.id, Number(e.target.value))
                                }
                                min="0"
                            />
                            <span>{(item.price * item.quantity).toLocaleString()}원</span>
                            <button onClick={() => removeItem(item.id)}>
                                삭제
                            </button>
                        </div>
                    ))}

                    <div>
                        <strong>총액: {total.toLocaleString()}원</strong>
                    </div>

                    <button onClick={clearCart}>전체 삭제</button>
                    <button>결제하기</button>
                </>
            )}
        </div>
    );
}
```

## 클래스형 vs 함수형 컴포넌트

```javascript
// 클래스형 컴포넌트 (레거시)
class CounterClass extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            count: 0
        };
    }

    componentDidMount() {
        console.log('마운트됨');
    }

    componentDidUpdate(prevProps, prevState) {
        console.log('업데이트됨');
    }

    componentWillUnmount() {
        console.log('언마운트됨');
    }

    render() {
        return (
            <div>
                <p>{this.state.count}</p>
                <button onClick={() => this.setState({ count: this.state.count + 1 })}>
                    증가
                </button>
            </div>
        );
    }
}

// 함수형 컴포넌트 (현대적)
function CounterFunction() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        console.log('마운트됨');

        return () => {
            console.log('언마운트됨');
        };
    }, []);

    useEffect(() => {
        console.log('업데이트됨');
    });

    return (
        <div>
            <p>{count}</p>
            <button onClick={() => setCount(count + 1)}>
                증가
            </button>
        </div>
    );
}
```

## 핵심 요약

### 1. State는 컴포넌트의 메모리
- useState로 상태 관리
- setState로 업데이트
- 불변성 유지 (새 객체/배열 생성)

### 2. useEffect는 부수 효과 처리
- 데이터 가져오기
- 타이머/이벤트 리스너
- 정리(cleanup) 함수 제공

### 3. 의존성 배열이 핵심
```javascript
useEffect(effect, []);           // 마운트 시
useEffect(effect, [dep]);        // dep 변경 시
useEffect(effect);               // 매 렌더링
```

### 4. 실전 패턴
```javascript
// ✅ 함수형 업데이트
setState(prev => prev + 1);

// ✅ 객체 불변성
setState({ ...prev, key: value });

// ✅ 배열 불변성
setState([...prev, newItem]);

// ✅ Cleanup
useEffect(() => {
    const timer = setInterval(...);
    return () => clearInterval(timer);
}, []);
```

## 연습 문제

### 문제 1: 카운터 앱
다양한 기능을 가진 카운터를 만들어보세요.

```javascript
// 요구사항:
// - 증가, 감소, 리셋 버튼
// - 증가/감소 스텝 설정 (1, 5, 10)
// - 최소값(0), 최대값(100) 제한
// - 로컬 스토리지에 저장

function Counter() {
    // 여기에 구현
}
```

### 문제 2: 실시간 검색
검색어를 입력하면 실시간으로 결과를 보여주는 컴포넌트를 만들어보세요.

```javascript
// 요구사항:
// - 입력 후 500ms 디바운싱
// - 로딩 상태 표시
// - 검색 결과 표시
// - 빈 검색어일 때 초기화

function LiveSearch() {
    // 여기에 구현
}
```

### 문제 3: Todo 앱
할 일 목록 관리 앱을 만들어보세요.

```javascript
// 요구사항:
// - 할 일 추가/삭제/완료 토글
// - 전체/활성/완료 필터
// - 완료된 항목 일괄 삭제
// - 로컬 스토리지 저장

function TodoApp() {
    // 여기에 구현
}
```

### 문제 4: 타이머
카운트다운 타이머를 만들어보세요.

```javascript
// 요구사항:
// - 시작/일시정지/리셋 버튼
// - 분:초 형식으로 표시
// - 0초가 되면 알림
// - 타이머 종료 시 cleanup

function CountdownTimer() {
    // 여기에 구현
}
```

## 다음 단계
다음 장에서는 **이벤트 처리**를 학습합니다:
- React의 이벤트 시스템
- 이벤트 핸들러 작성
- 폼 처리
- 이벤트 위임과 최적화
