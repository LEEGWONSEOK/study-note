# 28. 리스트와 Key

## 학습 목표
- 리스트 렌더링 방법 이해하기
- Key의 역할과 중요성 학습하기
- 동적 리스트 관리 패턴 익히기
- 리스트 렌더링 최적화 기법 마스터하기

## 리스트 렌더링

React에서 **배열 데이터를 UI로 변환**하는 방법을 학습합니다.

### 비유로 이해하기
리스트 렌더링은 **공장 생산 라인**과 같습니다:
- 원재료(데이터 배열)를 투입
- 각 항목을 가공(컴포넌트로 변환)
- 완제품(UI)을 생산
- Key는 각 제품의 **일련번호**

### 기본 리스트 렌더링

```javascript
function BasicList() {
    const numbers = [1, 2, 3, 4, 5];

    // map을 사용한 변환
    return (
        <ul>
            {numbers.map(number => (
                <li key={number}>
                    숫자: {number}
                </li>
            ))}
        </ul>
    );
}

// 객체 배열
function UserList() {
    const users = [
        { id: 1, name: '김철수', email: 'kim@example.com' },
        { id: 2, name: '이영희', email: 'lee@example.com' },
        { id: 3, name: '박민수', email: 'park@example.com' }
    ];

    return (
        <ul>
            {users.map(user => (
                <li key={user.id}>
                    {user.name} - {user.email}
                </li>
            ))}
        </ul>
    );
}
```

### Java와 비교

```java
// Java: 반복문으로 UI 생성
List<User> users = Arrays.asList(
    new User(1, "김철수"),
    new User(2, "이영희")
);

JPanel panel = new JPanel();
for (User user : users) {
    JLabel label = new JLabel(user.getName());
    panel.add(label);
}
```

```javascript
// React: map으로 선언적 렌더링
const users = [
    { id: 1, name: '김철수' },
    { id: 2, name: '이영희' }
];

return (
    <ul>
        {users.map(user => (
            <li key={user.id}>{user.name}</li>
        ))}
    </ul>
);

// 차이점:
// Java: 명령형 - "어떻게" 할지 기술
// React: 선언형 - "무엇을" 표시할지 기술
```

## Key의 이해

Key는 React가 **어떤 항목이 변경/추가/제거되었는지 식별**하는 데 사용됩니다.

### Key가 필요한 이유

```javascript
// ❌ Key 없이 렌더링 (경고 발생)
function WithoutKey() {
    const items = ['사과', '바나나', '오렌지'];

    return (
        <ul>
            {items.map(item => (
                <li>{item}</li> // Warning: Each child should have a unique "key" prop
            ))}
        </ul>
    );
}

// ✅ Key와 함께 렌더링
function WithKey() {
    const items = ['사과', '바나나', '오렌지'];

    return (
        <ul>
            {items.map((item, index) => (
                <li key={index}>{item}</li>
            ))}
        </ul>
    );
}
```

### Key 동작 원리

```javascript
// 초기 상태
<ul>
    <li key="1">사과</li>
    <li key="2">바나나</li>
    <li key="3">오렌지</li>
</ul>

// "딸기"를 맨 앞에 추가했을 때

// ✅ 올바른 Key 사용: 효율적 업데이트
<ul>
    <li key="4">딸기</li>    // 새로 추가
    <li key="1">사과</li>    // 그대로 유지
    <li key="2">바나나</li>  // 그대로 유지
    <li key="3">오렌지</li>  // 그대로 유지
</ul>
// → React가 key로 식별하여 딸기만 새로 생성

// ❌ index를 Key로 사용: 비효율적
<ul>
    <li key="0">딸기</li>    // 내용 변경 (사과→딸기)
    <li key="1">사과</li>    // 내용 변경 (바나나→사과)
    <li key="2">바나나</li>  // 내용 변경 (오렌지→바나나)
    <li key="3">오렌지</li>  // 새로 추가
</ul>
// → 모든 항목을 다시 렌더링 (비효율적!)
```

### 좋은 Key vs 나쁜 Key

```javascript
function KeyExamples() {
    const users = [
        { id: 1, name: '김철수' },
        { id: 2, name: '이영희' },
        { id: 3, name: '박민수' }
    ];

    return (
        <div>
            {/* ✅ 좋은 예: 고유한 ID 사용 */}
            <ul>
                {users.map(user => (
                    <li key={user.id}>{user.name}</li>
                ))}
            </ul>

            {/* ⚠️ 주의: index를 key로 사용 */}
            <ul>
                {users.map((user, index) => (
                    <li key={index}>{user.name}</li>
                ))}
            </ul>
            {/*
            index를 key로 사용해도 되는 경우:
            - 항목이 추가/삭제되지 않을 때
            - 항목의 순서가 바뀌지 않을 때
            - 항목이 정적일 때
            */}

            {/* ❌ 나쁜 예: 불안정한 key */}
            <ul>
                {users.map(user => (
                    <li key={Math.random()}>{user.name}</li>
                ))}
            </ul>
            {/* 매 렌더링마다 새로운 key → 전체 재생성! */}

            {/* ❌ 나쁜 예: 중복 가능한 값 */}
            <ul>
                {users.map(user => (
                    <li key={user.name}>{user.name}</li>
                ))}
            </ul>
            {/* 같은 이름이 있으면 key 충돌! */}
        </div>
    );
}
```

### Key 선택 가이드

```javascript
// 1. 데이터베이스 ID (최선)
<li key={user.id}>{user.name}</li>

// 2. UUID 생성 (데이터 생성 시)
const items = data.map(item => ({
    ...item,
    uuid: crypto.randomUUID() // 또는 uuid 라이브러리
}));

<li key={item.uuid}>{item.name}</li>

// 3. 복합 key
<li key={`${user.id}-${user.type}`}>
    {user.name}
</li>

// 4. index (최후의 수단)
// 항목이 정적이고 순서가 바뀌지 않을 때만!
<li key={index}>{item.name}</li>
```

## 리스트 조작 패턴

### 1. 필터링

```javascript
function FilteredList() {
    const [filter, setFilter] = useState('all');
    const [todos] = useState([
        { id: 1, text: '공부하기', completed: false },
        { id: 2, text: '운동하기', completed: true },
        { id: 3, text: '책 읽기', completed: false },
        { id: 4, text: '코딩하기', completed: true }
    ]);

    // 필터링 로직
    const filteredTodos = todos.filter(todo => {
        if (filter === 'active') return !todo.completed;
        if (filter === 'completed') return todo.completed;
        return true; // 'all'
    });

    return (
        <div>
            <div>
                <button onClick={() => setFilter('all')}>
                    전체 ({todos.length})
                </button>
                <button onClick={() => setFilter('active')}>
                    활성 ({todos.filter(t => !t.completed).length})
                </button>
                <button onClick={() => setFilter('completed')}>
                    완료 ({todos.filter(t => t.completed).length})
                </button>
            </div>

            <ul>
                {filteredTodos.map(todo => (
                    <li key={todo.id}>
                        {todo.text}
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

### 2. 정렬

```javascript
function SortedList() {
    const [sortBy, setSortBy] = useState('name');
    const [sortOrder, setSortOrder] = useState('asc');

    const users = [
        { id: 1, name: '김철수', age: 30, score: 85 },
        { id: 2, name: '이영희', age: 25, score: 92 },
        { id: 3, name: '박민수', age: 35, score: 78 }
    ];

    // 정렬 로직
    const sortedUsers = [...users].sort((a, b) => {
        let aVal = a[sortBy];
        let bVal = b[sortBy];

        // 문자열은 localeCompare 사용
        if (typeof aVal === 'string') {
            return sortOrder === 'asc'
                ? aVal.localeCompare(bVal)
                : bVal.localeCompare(aVal);
        }

        // 숫자는 직접 비교
        return sortOrder === 'asc'
            ? aVal - bVal
            : bVal - aVal;
    });

    const toggleSort = (field) => {
        if (sortBy === field) {
            // 같은 필드면 순서 반전
            setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc');
        } else {
            // 다른 필드면 새로 설정
            setSortBy(field);
            setSortOrder('asc');
        }
    };

    return (
        <div>
            <div>
                <button onClick={() => toggleSort('name')}>
                    이름 {sortBy === 'name' && (sortOrder === 'asc' ? '↑' : '↓')}
                </button>
                <button onClick={() => toggleSort('age')}>
                    나이 {sortBy === 'age' && (sortOrder === 'asc' ? '↑' : '↓')}
                </button>
                <button onClick={() => toggleSort('score')}>
                    점수 {sortBy === 'score' && (sortOrder === 'asc' ? '↑' : '↓')}
                </button>
            </div>

            <ul>
                {sortedUsers.map(user => (
                    <li key={user.id}>
                        {user.name} - {user.age}세 - {user.score}점
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

### 3. 검색

```javascript
function SearchableList() {
    const [query, setQuery] = useState('');

    const items = [
        { id: 1, name: '사과', category: '과일' },
        { id: 2, name: '바나나', category: '과일' },
        { id: 3, name: '당근', category: '채소' },
        { id: 4, name: '브로콜리', category: '채소' }
    ];

    // 검색 로직
    const searchedItems = items.filter(item =>
        item.name.includes(query) ||
        item.category.includes(query)
    );

    return (
        <div>
            <input
                value={query}
                onChange={(e) => setQuery(e.target.value)}
                placeholder="검색..."
            />

            {query && (
                <p>"{query}" 검색 결과: {searchedItems.length}개</p>
            )}

            <ul>
                {searchedItems.map(item => (
                    <li key={item.id}>
                        {item.name} ({item.category})
                    </li>
                ))}
            </ul>

            {searchedItems.length === 0 && (
                <p>검색 결과가 없습니다.</p>
            )}
        </div>
    );
}
```

### 4. 페이지네이션

```javascript
function PaginatedList() {
    const [currentPage, setCurrentPage] = useState(1);
    const itemsPerPage = 5;

    const items = Array.from({ length: 50 }, (_, i) => ({
        id: i + 1,
        name: `항목 ${i + 1}`
    }));

    // 페이지네이션 계산
    const totalPages = Math.ceil(items.length / itemsPerPage);
    const startIndex = (currentPage - 1) * itemsPerPage;
    const endIndex = startIndex + itemsPerPage;
    const currentItems = items.slice(startIndex, endIndex);

    const goToPage = (page) => {
        setCurrentPage(Math.max(1, Math.min(page, totalPages)));
    };

    return (
        <div>
            <ul>
                {currentItems.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>

            <div>
                <button
                    onClick={() => goToPage(currentPage - 1)}
                    disabled={currentPage === 1}
                >
                    이전
                </button>

                <span>
                    페이지 {currentPage} / {totalPages}
                </span>

                <button
                    onClick={() => goToPage(currentPage + 1)}
                    disabled={currentPage === totalPages}
                >
                    다음
                </button>
            </div>

            <div>
                {Array.from({ length: totalPages }, (_, i) => i + 1).map(page => (
                    <button
                        key={page}
                        onClick={() => goToPage(page)}
                        style={{
                            fontWeight: page === currentPage ? 'bold' : 'normal'
                        }}
                    >
                        {page}
                    </button>
                ))}
            </div>
        </div>
    );
}
```

## 중첩 리스트

```javascript
function NestedList() {
    const categories = [
        {
            id: 1,
            name: '과일',
            items: [
                { id: 101, name: '사과' },
                { id: 102, name: '바나나' },
                { id: 103, name: '오렌지' }
            ]
        },
        {
            id: 2,
            name: '채소',
            items: [
                { id: 201, name: '당근' },
                { id: 202, name: '브로콜리' },
                { id: 203, name: '시금치' }
            ]
        }
    ];

    return (
        <div>
            {categories.map(category => (
                <div key={category.id}>
                    <h3>{category.name}</h3>
                    <ul>
                        {category.items.map(item => (
                            <li key={item.id}>{item.name}</li>
                        ))}
                    </ul>
                </div>
            ))}
        </div>
    );
}

// 재귀적 중첩 (트리 구조)
function TreeNode({ node }) {
    return (
        <li>
            {node.name}
            {node.children && node.children.length > 0 && (
                <ul>
                    {node.children.map(child => (
                        <TreeNode key={child.id} node={child} />
                    ))}
                </ul>
            )}
        </li>
    );
}

function Tree() {
    const tree = {
        id: 1,
        name: '루트',
        children: [
            {
                id: 2,
                name: '자식 1',
                children: [
                    { id: 4, name: '손자 1' },
                    { id: 5, name: '손자 2' }
                ]
            },
            {
                id: 3,
                name: '자식 2',
                children: [
                    { id: 6, name: '손자 3' }
                ]
            }
        ]
    };

    return (
        <ul>
            <TreeNode node={tree} />
        </ul>
    );
}
```

## 실전 예제

### 1. Todo 리스트 (완전한 버전)

```javascript
function TodoList() {
    const [todos, setTodos] = useState([]);
    const [input, setInput] = useState('');
    const [filter, setFilter] = useState('all');
    const [editingId, setEditingId] = useState(null);
    const [editText, setEditText] = useState('');

    // 추가
    const addTodo = (e) => {
        e.preventDefault();
        if (!input.trim()) return;

        const newTodo = {
            id: Date.now(),
            text: input,
            completed: false,
            createdAt: new Date()
        };

        setTodos([...todos, newTodo]);
        setInput('');
    };

    // 토글
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

    // 수정 시작
    const startEdit = (todo) => {
        setEditingId(todo.id);
        setEditText(todo.text);
    };

    // 수정 완료
    const finishEdit = () => {
        if (editText.trim()) {
            setTodos(todos.map(todo =>
                todo.id === editingId
                    ? { ...todo, text: editText }
                    : todo
            ));
        }
        setEditingId(null);
        setEditText('');
    };

    // 수정 취소
    const cancelEdit = () => {
        setEditingId(null);
        setEditText('');
    };

    // 완료된 항목 삭제
    const clearCompleted = () => {
        setTodos(todos.filter(todo => !todo.completed));
    };

    // 필터링
    const filteredTodos = todos.filter(todo => {
        if (filter === 'active') return !todo.completed;
        if (filter === 'completed') return todo.completed;
        return true;
    });

    // 통계
    const stats = {
        total: todos.length,
        active: todos.filter(t => !t.completed).length,
        completed: todos.filter(t => t.completed).length
    };

    return (
        <div className="todo-app">
            <h1>Todo List</h1>

            {/* 입력 폼 */}
            <form onSubmit={addTodo}>
                <input
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    placeholder="할 일을 입력하세요..."
                />
                <button type="submit">추가</button>
            </form>

            {/* 필터 버튼 */}
            <div className="filters">
                <button
                    onClick={() => setFilter('all')}
                    className={filter === 'all' ? 'active' : ''}
                >
                    전체 ({stats.total})
                </button>
                <button
                    onClick={() => setFilter('active')}
                    className={filter === 'active' ? 'active' : ''}
                >
                    활성 ({stats.active})
                </button>
                <button
                    onClick={() => setFilter('completed')}
                    className={filter === 'completed' ? 'active' : ''}
                >
                    완료 ({stats.completed})
                </button>
            </div>

            {/* Todo 리스트 */}
            <ul className="todo-list">
                {filteredTodos.map(todo => (
                    <li key={todo.id} className={todo.completed ? 'completed' : ''}>
                        {editingId === todo.id ? (
                            // 수정 모드
                            <div className="edit-mode">
                                <input
                                    value={editText}
                                    onChange={(e) => setEditText(e.target.value)}
                                    onKeyDown={(e) => {
                                        if (e.key === 'Enter') finishEdit();
                                        if (e.key === 'Escape') cancelEdit();
                                    }}
                                    autoFocus
                                />
                                <button onClick={finishEdit}>완료</button>
                                <button onClick={cancelEdit}>취소</button>
                            </div>
                        ) : (
                            // 일반 모드
                            <div className="view-mode">
                                <input
                                    type="checkbox"
                                    checked={todo.completed}
                                    onChange={() => toggleTodo(todo.id)}
                                />
                                <span onDoubleClick={() => startEdit(todo)}>
                                    {todo.text}
                                </span>
                                <button onClick={() => startEdit(todo)}>
                                    수정
                                </button>
                                <button onClick={() => deleteTodo(todo.id)}>
                                    삭제
                                </button>
                            </div>
                        )}
                    </li>
                ))}
            </ul>

            {/* 하단 액션 */}
            {stats.completed > 0 && (
                <div className="actions">
                    <button onClick={clearCompleted}>
                        완료된 항목 삭제 ({stats.completed})
                    </button>
                </div>
            )}

            {filteredTodos.length === 0 && (
                <p className="empty">할 일이 없습니다!</p>
            )}
        </div>
    );
}
```

### 2. 데이터 테이블

```javascript
function DataTable() {
    const [data, setData] = useState([
        { id: 1, name: '김철수', age: 30, email: 'kim@example.com', role: 'Developer' },
        { id: 2, name: '이영희', age: 25, email: 'lee@example.com', role: 'Designer' },
        { id: 3, name: '박민수', age: 35, email: 'park@example.com', role: 'Manager' }
    ]);

    const [sortConfig, setSortConfig] = useState({
        key: null,
        direction: 'asc'
    });

    const [selectedRows, setSelectedRows] = useState([]);

    // 정렬
    const handleSort = (key) => {
        let direction = 'asc';
        if (sortConfig.key === key && sortConfig.direction === 'asc') {
            direction = 'desc';
        }
        setSortConfig({ key, direction });
    };

    const sortedData = [...data].sort((a, b) => {
        if (!sortConfig.key) return 0;

        const aVal = a[sortConfig.key];
        const bVal = b[sortConfig.key];

        if (typeof aVal === 'string') {
            return sortConfig.direction === 'asc'
                ? aVal.localeCompare(bVal)
                : bVal.localeCompare(aVal);
        }

        return sortConfig.direction === 'asc'
            ? aVal - bVal
            : bVal - aVal;
    });

    // 행 선택
    const toggleRow = (id) => {
        setSelectedRows(prev =>
            prev.includes(id)
                ? prev.filter(rowId => rowId !== id)
                : [...prev, id]
        );
    };

    const toggleAll = () => {
        setSelectedRows(
            selectedRows.length === data.length
                ? []
                : data.map(row => row.id)
        );
    };

    // 삭제
    const deleteSelected = () => {
        setData(data.filter(row => !selectedRows.includes(row.id)));
        setSelectedRows([]);
    };

    return (
        <div>
            <div className="table-actions">
                <span>선택됨: {selectedRows.length}개</span>
                {selectedRows.length > 0 && (
                    <button onClick={deleteSelected}>
                        선택 항목 삭제
                    </button>
                )}
            </div>

            <table>
                <thead>
                    <tr>
                        <th>
                            <input
                                type="checkbox"
                                checked={selectedRows.length === data.length}
                                onChange={toggleAll}
                            />
                        </th>
                        <th onClick={() => handleSort('name')}>
                            이름 {sortConfig.key === 'name' && (
                                sortConfig.direction === 'asc' ? '↑' : '↓'
                            )}
                        </th>
                        <th onClick={() => handleSort('age')}>
                            나이 {sortConfig.key === 'age' && (
                                sortConfig.direction === 'asc' ? '↑' : '↓'
                            )}
                        </th>
                        <th onClick={() => handleSort('email')}>
                            이메일 {sortConfig.key === 'email' && (
                                sortConfig.direction === 'asc' ? '↑' : '↓'
                            )}
                        </th>
                        <th onClick={() => handleSort('role')}>
                            역할 {sortConfig.key === 'role' && (
                                sortConfig.direction === 'asc' ? '↑' : '↓'
                            )}
                        </th>
                    </tr>
                </thead>
                <tbody>
                    {sortedData.map(row => (
                        <tr
                            key={row.id}
                            className={selectedRows.includes(row.id) ? 'selected' : ''}
                        >
                            <td>
                                <input
                                    type="checkbox"
                                    checked={selectedRows.includes(row.id)}
                                    onChange={() => toggleRow(row.id)}
                                />
                            </td>
                            <td>{row.name}</td>
                            <td>{row.age}</td>
                            <td>{row.email}</td>
                            <td>{row.role}</td>
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
}
```

## 성능 최적화

### 1. React.memo로 컴포넌트 메모이제이션

```javascript
// 자식 컴포넌트 메모이제이션
const TodoItem = React.memo(({ todo, onToggle, onDelete }) => {
    console.log('렌더링:', todo.id);

    return (
        <li>
            <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => onToggle(todo.id)}
            />
            <span>{todo.text}</span>
            <button onClick={() => onDelete(todo.id)}>삭제</button>
        </li>
    );
});

function TodoList() {
    const [todos, setTodos] = useState([
        { id: 1, text: '공부하기', completed: false },
        { id: 2, text: '운동하기', completed: false }
    ]);

    // useCallback으로 함수 메모이제이션
    const handleToggle = useCallback((id) => {
        setTodos(prev => prev.map(todo =>
            todo.id === id
                ? { ...todo, completed: !todo.completed }
                : todo
        ));
    }, []);

    const handleDelete = useCallback((id) => {
        setTodos(prev => prev.filter(todo => todo.id !== id));
    }, []);

    return (
        <ul>
            {todos.map(todo => (
                <TodoItem
                    key={todo.id}
                    todo={todo}
                    onToggle={handleToggle}
                    onDelete={handleDelete}
                />
            ))}
        </ul>
    );
}
```

### 2. 가상화 (Virtualization)

큰 리스트의 경우 **보이는 항목만 렌더링**합니다.

```javascript
// react-window 라이브러리 사용 예시
import { FixedSizeList } from 'react-window';

function VirtualizedList() {
    const items = Array.from({ length: 10000 }, (_, i) => `항목 ${i + 1}`);

    const Row = ({ index, style }) => (
        <div style={style}>
            {items[index]}
        </div>
    );

    return (
        <FixedSizeList
            height={600}        // 컨테이너 높이
            itemCount={items.length}  // 전체 항목 수
            itemSize={50}       // 각 항목 높이
            width="100%"
        >
            {Row}
        </FixedSizeList>
    );
}
```

## 핵심 요약

### 1. 리스트 렌더링은 map 사용
```javascript
{items.map(item => (
    <li key={item.id}>{item.name}</li>
))}
```

### 2. Key는 필수
- 고유하고 안정적인 값 사용
- ID가 최선, index는 최후의 수단
- 매 렌더링마다 바뀌면 안 됨

### 3. 리스트 조작
- 필터링: `filter()`
- 정렬: `sort()`
- 검색: `filter()` + 조건
- 페이지네이션: `slice()`

### 4. 성능 최적화
```javascript
// React.memo로 컴포넌트 메모이제이션
const Item = React.memo(({ data }) => <div>{data}</div>);

// useCallback으로 함수 메모이제이션
const handleClick = useCallback(() => {}, []);
```

## 연습 문제

### 문제 1: 정렬 가능한 리스트
사용자가 클릭하여 정렬할 수 있는 리스트를 만들어보세요.

```javascript
// 요구사항:
// - 이름, 나이, 점수로 정렬
// - 오름차순/내림차순 토글
// - 현재 정렬 상태 표시

const data = [
    { id: 1, name: '김철수', age: 30, score: 85 },
    { id: 2, name: '이영희', age: 25, score: 92 },
    { id: 3, name: '박민수', age: 35, score: 78 }
];

function SortableList() {
    // 여기에 구현
}
```

### 문제 2: 무한 스크롤 리스트
스크롤하면 자동으로 데이터를 로드하는 리스트를 만들어보세요.

```javascript
// 요구사항:
// - 초기 20개 항목 표시
// - 스크롤이 바닥에 닿으면 20개 더 로드
// - 로딩 상태 표시
// - 더 이상 데이터가 없으면 메시지 표시

function InfiniteScrollList() {
    // 여기에 구현
}
```

### 문제 3: 드래그 앤 드롭 리스트
항목을 드래그하여 순서를 바꿀 수 있는 리스트를 만들어보세요.

```javascript
// 요구사항:
// - 드래그하여 순서 변경
// - 드래그 중인 항목 하이라이트
// - 순서 변경 후 상태 업데이트

function DraggableList() {
    // 여기에 구현
    // 힌트: onDragStart, onDragOver, onDrop 이벤트 사용
}
```

### 문제 4: 트리 뷰
폴더 구조를 표시하고 펼치기/접기가 가능한 트리를 만들어보세요.

```javascript
// 요구사항:
// - 중첩된 구조 표시
// - 클릭하여 펼치기/접기
// - 재귀적 렌더링
// - 파일과 폴더 구분

const fileSystem = {
    name: 'root',
    type: 'folder',
    children: [
        {
            name: 'src',
            type: 'folder',
            children: [
                { name: 'index.js', type: 'file' },
                { name: 'App.js', type: 'file' }
            ]
        },
        { name: 'package.json', type: 'file' }
    ]
};

function TreeView() {
    // 여기에 구현
}
```

## 다음 단계
다음 장에서는 **Hooks 심화**를 학습합니다:
- useMemo와 useCallback
- useRef 활용
- 커스텀 Hook 작성
- Hook 규칙과 패턴