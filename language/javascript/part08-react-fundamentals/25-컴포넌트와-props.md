# 25. 컴포넌트와 Props

## 학습 목표
- React 컴포넌트의 개념과 종류 이해하기
- Props를 통한 데이터 전달 방법 학습하기
- 컴포넌트 구조화 및 재사용 패턴 익히기
- 컴포넌트 설계 원칙 이해하기

## React 컴포넌트란?

React 컴포넌트는 **재사용 가능한 UI 빌딩 블록**입니다.

### 비유로 이해하기
컴포넌트는 **레고 블록**과 같습니다:
- 각 블록은 독립적인 기능을 가짐
- 여러 블록을 조합하여 복잡한 구조 생성
- 같은 블록을 여러 곳에서 재사용 가능
- 블록을 교체하거나 수정하기 쉬움

### Java와 비교

```java
// Java: 클래스 기반 UI 컴포넌트
public class Button extends UIComponent {
    private String text;
    private String color;

    public Button(String text, String color) {
        this.text = text;
        this.color = color;
    }

    @Override
    public void render() {
        // 버튼 렌더링 로직
    }
}

// 사용
Button submitButton = new Button("제출", "blue");
submitButton.render();
```

```javascript
// React: 함수형 컴포넌트
function Button({ text, color }) {
    return (
        <button style={{ backgroundColor: color }}>
            {text}
        </button>
    );
}

// 사용
<Button text="제출" color="blue" />
```

## 컴포넌트의 종류

### 1. 함수형 컴포넌트 (권장)

```javascript
// 기본 함수형 컴포넌트
function Welcome(props) {
    return <h1>안녕하세요, {props.name}님!</h1>;
}

// 화살표 함수 버전
const Welcome = (props) => {
    return <h1>안녕하세요, {props.name}님!</h1>;
};

// 구조 분해 할당 사용
const Welcome = ({ name }) => {
    return <h1>안녕하세요, {name}님!</h1>;
};

// 암시적 반환 (간단한 경우)
const Welcome = ({ name }) => <h1>안녕하세요, {name}님!</h1>;
```

### 2. 클래스형 컴포넌트 (레거시)

```javascript
import React, { Component } from 'react';

class Welcome extends Component {
    render() {
        return <h1>안녕하세요, {this.props.name}님!</h1>;
    }
}

// Java 개발자에게 친숙한 형태
// 하지만 React에서는 함수형 컴포넌트를 권장합니다
```

### 비교 예제

```javascript
// 함수형: 간결하고 현대적
function UserCard({ name, email, avatar }) {
    return (
        <div className="user-card">
            <img src={avatar} alt={name} />
            <h2>{name}</h2>
            <p>{email}</p>
        </div>
    );
}

// 클래스형: 더 많은 보일러플레이트
class UserCard extends Component {
    render() {
        const { name, email, avatar } = this.props;
        return (
            <div className="user-card">
                <img src={avatar} alt={name} />
                <h2>{name}</h2>
                <p>{email}</p>
            </div>
        );
    }
}
```

## Props: 컴포넌트 간 데이터 전달

Props(Properties)는 **부모 컴포넌트에서 자식 컴포넌트로 데이터를 전달**하는 방법입니다.

### Props의 특징

```javascript
// 1. Props는 읽기 전용입니다
function Button({ text }) {
    // ❌ 잘못된 방법: props 직접 수정
    // text = '새로운 텍스트'; // 에러!

    // ✅ 올바른 방법: props는 읽기만 가능
    return <button>{text}</button>;
}

// 2. Props는 어떤 타입이든 전달 가능
function Component(props) {
    // 문자열, 숫자, 배열, 객체, 함수 등 모두 가능
    console.log(props);
}

// 3. Props는 위에서 아래로만 흐름 (단방향)
<Parent>
    <Child data="from parent" /> {/* Parent → Child */}
</Parent>
```

### Props 전달 방법

```javascript
// 기본 props 전달
function App() {
    return (
        <div>
            {/* 문자열 */}
            <Greeting name="김철수" />

            {/* 숫자 (중괄호 필요) */}
            <Counter count={10} />

            {/* 불리언 */}
            <Button disabled={true} />
            <Button disabled /> {/* true는 생략 가능 */}

            {/* 배열 */}
            <List items={['사과', '바나나', '오렌지']} />

            {/* 객체 */}
            <UserCard user={{ name: '김철수', age: 30 }} />

            {/* 함수 */}
            <Button onClick={() => console.log('클릭!')} />

            {/* JSX (children) */}
            <Card>
                <h2>제목</h2>
                <p>내용</p>
            </Card>
        </div>
    );
}
```

### Props 구조 분해 할당

```javascript
// 방법 1: props 객체 사용
function UserInfo(props) {
    return (
        <div>
            <h2>{props.name}</h2>
            <p>{props.email}</p>
            <span>{props.age}세</span>
        </div>
    );
}

// 방법 2: 구조 분해 할당 (파라미터에서)
function UserInfo({ name, email, age }) {
    return (
        <div>
            <h2>{name}</h2>
            <p>{email}</p>
            <span>{age}세</span>
        </div>
    );
}

// 방법 3: 일부만 구조 분해 (나머지는 ...rest)
function UserInfo({ name, ...rest }) {
    return (
        <div>
            <h2>{name}</h2>
            <pre>{JSON.stringify(rest, null, 2)}</pre>
        </div>
    );
}

// 방법 4: 중첩된 구조 분해
function UserInfo({ user: { name, email }, settings: { theme } }) {
    return (
        <div className={theme}>
            <h2>{name}</h2>
            <p>{email}</p>
        </div>
    );
}
```

### 기본값 설정

```javascript
// 방법 1: 파라미터 기본값
function Button({ text = '클릭', color = 'blue', disabled = false }) {
    return (
        <button
            style={{ backgroundColor: color }}
            disabled={disabled}
        >
            {text}
        </button>
    );
}

// 방법 2: defaultProps (클래스형에서 주로 사용)
function Button({ text, color, disabled }) {
    return (
        <button
            style={{ backgroundColor: color }}
            disabled={disabled}
        >
            {text}
        </button>
    );
}

Button.defaultProps = {
    text: '클릭',
    color: 'blue',
    disabled: false
};

// 방법 3: 논리 연산자 사용
function Button({ text, color }) {
    const buttonText = text || '클릭';
    const buttonColor = color ?? 'blue'; // null/undefined만 체크

    return (
        <button style={{ backgroundColor: buttonColor }}>
            {buttonText}
        </button>
    );
}
```

### Children Props

`children`은 특별한 prop으로, **컴포넌트 태그 사이의 내용**을 나타냅니다.

```javascript
// 기본 children 사용
function Card({ children }) {
    return (
        <div className="card">
            {children}
        </div>
    );
}

// 사용
<Card>
    <h2>제목</h2>
    <p>내용입니다.</p>
</Card>

// children 조작하기
function List({ children }) {
    // children이 배열인지 확인
    const items = React.Children.toArray(children);

    return (
        <ul>
            {items.map((child, index) => (
                <li key={index}>{child}</li>
            ))}
        </ul>
    );
}

// 조건부 children 렌더링
function Container({ children, title }) {
    return (
        <div>
            {title && <h2>{title}</h2>}
            {children}
        </div>
    );
}

// children 타입 체크
function Alert({ children }) {
    if (typeof children === 'string') {
        return <div className="alert">{children}</div>;
    }

    return (
        <div className="alert">
            {children}
        </div>
    );
}
```

## 실전 예제

### 1. 재사용 가능한 Button 컴포넌트

```javascript
// Button.jsx
function Button({
    children,
    variant = 'primary',
    size = 'medium',
    disabled = false,
    onClick
}) {
    const baseClass = 'btn';
    const variantClass = `btn-${variant}`; // btn-primary, btn-secondary
    const sizeClass = `btn-${size}`; // btn-small, btn-medium, btn-large

    return (
        <button
            className={`${baseClass} ${variantClass} ${sizeClass}`}
            disabled={disabled}
            onClick={onClick}
        >
            {children}
        </button>
    );
}

// 사용 예시
function App() {
    return (
        <div>
            <Button variant="primary" size="large">
                제출하기
            </Button>

            <Button variant="secondary" size="small" onClick={() => alert('취소')}>
                취소
            </Button>

            <Button variant="danger" disabled>
                삭제 (비활성화)
            </Button>
        </div>
    );
}
```

### 2. UserCard 컴포넌트

```javascript
// UserCard.jsx
function UserCard({ user, onEdit, onDelete }) {
    const { id, name, email, role, avatar } = user;

    return (
        <div className="user-card">
            <div className="user-card__header">
                <img
                    src={avatar || '/default-avatar.png'}
                    alt={name}
                    className="user-card__avatar"
                />
                <div className="user-card__info">
                    <h3>{name}</h3>
                    <p>{email}</p>
                    <span className="badge">{role}</span>
                </div>
            </div>

            <div className="user-card__actions">
                <button onClick={() => onEdit(id)}>
                    수정
                </button>
                <button onClick={() => onDelete(id)}>
                    삭제
                </button>
            </div>
        </div>
    );
}

// 사용 예시
function UserList() {
    const users = [
        { id: 1, name: '김철수', email: 'kim@example.com', role: 'Admin' },
        { id: 2, name: '이영희', email: 'lee@example.com', role: 'User' }
    ];

    const handleEdit = (id) => {
        console.log('Edit user:', id);
    };

    const handleDelete = (id) => {
        console.log('Delete user:', id);
    };

    return (
        <div className="user-list">
            {users.map(user => (
                <UserCard
                    key={user.id}
                    user={user}
                    onEdit={handleEdit}
                    onDelete={handleDelete}
                />
            ))}
        </div>
    );
}
```

### 3. 레이아웃 컴포넌트

```javascript
// Layout.jsx
function Layout({ children }) {
    return (
        <div className="layout">
            <Header />
            <main className="layout__content">
                {children}
            </main>
            <Footer />
        </div>
    );
}

function Header() {
    return (
        <header className="header">
            <nav>
                <a href="/">홈</a>
                <a href="/about">소개</a>
                <a href="/contact">연락처</a>
            </nav>
        </header>
    );
}

function Footer() {
    return (
        <footer className="footer">
            <p>&copy; 2024 My Company</p>
        </footer>
    );
}

// 사용 예시
function App() {
    return (
        <Layout>
            <h1>페이지 제목</h1>
            <p>페이지 내용입니다.</p>
        </Layout>
    );
}
```

### 4. 복합 컴포넌트 패턴

```javascript
// Accordion.jsx - 컴포넌트를 속성으로 제공
function Accordion({ children }) {
    return <div className="accordion">{children}</div>;
}

function AccordionItem({ title, children, isOpen = false }) {
    const [open, setOpen] = React.useState(isOpen);

    return (
        <div className="accordion-item">
            <button
                className="accordion-header"
                onClick={() => setOpen(!open)}
            >
                {title}
                <span>{open ? '−' : '+'}</span>
            </button>

            {open && (
                <div className="accordion-content">
                    {children}
                </div>
            )}
        </div>
    );
}

// 서브 컴포넌트로 연결
Accordion.Item = AccordionItem;

// 사용 예시
function FAQ() {
    return (
        <Accordion>
            <Accordion.Item title="React란 무엇인가요?">
                <p>React는 사용자 인터페이스를 구축하기 위한 JavaScript 라이브러리입니다.</p>
            </Accordion.Item>

            <Accordion.Item title="Props란 무엇인가요?" isOpen>
                <p>Props는 컴포넌트 간에 데이터를 전달하는 방법입니다.</p>
            </Accordion.Item>

            <Accordion.Item title="State와 Props의 차이는?">
                <p>State는 컴포넌트 내부에서 관리되고, Props는 외부에서 전달됩니다.</p>
            </Accordion.Item>
        </Accordion>
    );
}
```

## 컴포넌트 설계 원칙

### 1. 단일 책임 원칙 (Single Responsibility)

```javascript
// ❌ 나쁜 예: 너무 많은 책임
function UserDashboard({ userId }) {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [comments, setComments] = useState([]);
    const [loading, setLoading] = useState(true);

    // 사용자 정보, 게시물, 댓글 모두 처리
    // 너무 복잡함!

    return (
        <div>
            {/* 복잡한 렌더링 로직 */}
        </div>
    );
}

// ✅ 좋은 예: 책임 분리
function UserDashboard({ userId }) {
    return (
        <div className="dashboard">
            <UserProfile userId={userId} />
            <UserPosts userId={userId} />
            <UserComments userId={userId} />
        </div>
    );
}

function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    // 사용자 정보만 처리
    return <div>{/* 사용자 정보 */}</div>;
}

function UserPosts({ userId }) {
    const [posts, setPosts] = useState([]);
    // 게시물만 처리
    return <div>{/* 게시물 목록 */}</div>;
}
```

### 2. Props 인터페이스 명확하게 정의

```javascript
// PropTypes 사용 (타입 체크)
import PropTypes from 'prop-types';

function Product({ name, price, image, onAddToCart }) {
    return (
        <div className="product">
            <img src={image} alt={name} />
            <h3>{name}</h3>
            <p>{price.toLocaleString()}원</p>
            <button onClick={onAddToCart}>장바구니 담기</button>
        </div>
    );
}

Product.propTypes = {
    name: PropTypes.string.isRequired,
    price: PropTypes.number.isRequired,
    image: PropTypes.string,
    onAddToCart: PropTypes.func.isRequired
};

Product.defaultProps = {
    image: '/default-product.png'
};

// TypeScript 사용 (더 강력한 타입 체크)
interface ProductProps {
    name: string;
    price: number;
    image?: string;
    onAddToCart: () => void;
}

function Product({
    name,
    price,
    image = '/default-product.png',
    onAddToCart
}: ProductProps) {
    return (
        <div className="product">
            <img src={image} alt={name} />
            <h3>{name}</h3>
            <p>{price.toLocaleString()}원</p>
            <button onClick={onAddToCart}>장바구니 담기</button>
        </div>
    );
}
```

### 3. 컴포넌트 조합 (Composition)

```javascript
// 상속 대신 조합 사용
// ❌ 나쁜 예: 상속 시도 (React에서는 권장하지 않음)
class BaseButton extends Component { }
class PrimaryButton extends BaseButton { }

// ✅ 좋은 예: 조합 사용
function Button({ children, variant, ...props }) {
    return (
        <button className={`btn btn-${variant}`} {...props}>
            {children}
        </button>
    );
}

function PrimaryButton({ children, ...props }) {
    return (
        <Button variant="primary" {...props}>
            {children}
        </Button>
    );
}

function SecondaryButton({ children, ...props }) {
    return (
        <Button variant="secondary" {...props}>
            {children}
        </Button>
    );
}
```

### 4. Props Spreading과 Rest 파라미터

```javascript
// Props 전달하기
function Input({ label, error, ...inputProps }) {
    return (
        <div className="form-group">
            <label>{label}</label>
            {/* 나머지 모든 props를 input에 전달 */}
            <input {...inputProps} className="form-control" />
            {error && <span className="error">{error}</span>}
        </div>
    );
}

// 사용
<Input
    label="이메일"
    type="email"
    placeholder="email@example.com"
    required
    error="유효한 이메일을 입력하세요"
/>

// 조건부 props 전달
function Button({ primary, secondary, ...props }) {
    const className = primary
        ? 'btn-primary'
        : secondary
        ? 'btn-secondary'
        : 'btn-default';

    return <button {...props} className={className} />;
}
```

## Java 개발자를 위한 비교

```java
// Java: 클래스 기반, 상속 중심
public class UserPanel extends JPanel {
    private User user;

    public UserPanel(User user) {
        this.user = user;
        initialize();
    }

    private void initialize() {
        setLayout(new BorderLayout());
        add(new JLabel(user.getName()), BorderLayout.NORTH);
        add(new JLabel(user.getEmail()), BorderLayout.CENTER);
    }
}

// 사용
UserPanel panel = new UserPanel(user);
frame.add(panel);
```

```javascript
// React: 함수형, 조합 중심
function UserPanel({ user }) {
    return (
        <div className="user-panel">
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
}

// 사용
<UserPanel user={user} />

// 특징 비교:
// Java:
//   - 클래스 인스턴스 생성 필요
//   - 생명주기 메서드로 관리
//   - 상태는 인스턴스 변수에 저장
//
// React:
//   - 함수 호출로 즉시 사용
//   - Hooks로 생명주기 관리
//   - 상태는 Hook으로 관리
//   - 더 간결하고 조합이 쉬움
```

## 핵심 요약

### 1. 컴포넌트는 재사용 가능한 UI 블록
- 함수형 컴포넌트 사용 권장
- 작고 집중된 컴포넌트로 분리
- 조합을 통해 복잡한 UI 구성

### 2. Props는 데이터 전달 수단
- 읽기 전용 (불변)
- 위에서 아래로 단방향 흐름
- 모든 타입의 데이터 전달 가능

### 3. 설계 원칙
- 단일 책임 원칙
- 명확한 Props 인터페이스
- 상속보다는 조합
- Props 구조 분해로 가독성 향상

### 4. 실전 팁
```javascript
// ✅ 좋은 컴포넌트
function GoodComponent({ data, onAction }) {
    // 명확한 목적
    // 적절한 크기
    // 재사용 가능
    return <div>{/* ... */}</div>;
}

// ❌ 피해야 할 패턴
function BadComponent(props) {
    props.data = 'modified'; // Props 수정 금지!
    // 너무 많은 책임
    // 복잡한 로직
    return <div>{/* ... */}</div>;
}
```

## 연습 문제

### 문제 1: 프로필 카드 컴포넌트
사용자 프로필 카드를 만들어보세요.

```javascript
// 요구사항:
// - 이름, 이메일, 역할, 프로필 이미지 표시
// - 기본 이미지 제공
// - 활성/비활성 상태 표시
// - 수정/삭제 버튼 (콜백 함수로 처리)

function ProfileCard({ /* props */ }) {
    // 여기에 구현
}

// 사용 예시
<ProfileCard
    name="김철수"
    email="kim@example.com"
    role="개발자"
    avatar="https://..."
    active={true}
    onEdit={() => console.log('수정')}
    onDelete={() => console.log('삭제')}
/>
```

### 문제 2: 재사용 가능한 Input 컴포넌트
다양한 타입의 입력을 지원하는 Input 컴포넌트를 만들어보세요.

```javascript
// 요구사항:
// - label, error, helper text 지원
// - 모든 HTML input 속성 전달
// - 에러 상태 시 빨간색 테두리
// - required 필드는 * 표시

function FormInput({ /* props */ }) {
    // 여기에 구현
}

// 사용 예시
<FormInput
    label="이메일"
    type="email"
    required
    error="유효한 이메일을 입력하세요"
    helperText="example@domain.com 형식으로 입력"
/>
```

### 문제 3: Card 컴포넌트 조합
Header, Body, Footer를 가진 Card 컴포넌트를 만들어보세요.

```javascript
// 요구사항:
// - Card, Card.Header, Card.Body, Card.Footer 서브 컴포넌트
// - 각 부분은 선택적으로 사용 가능
// - children으로 내용 전달

function Card({ /* props */ }) {
    // 여기에 구현
}

// 사용 예시
<Card>
    <Card.Header>
        <h2>제목</h2>
    </Card.Header>
    <Card.Body>
        <p>본문 내용</p>
    </Card.Body>
    <Card.Footer>
        <button>확인</button>
    </Card.Footer>
</Card>
```

### 문제 4: 제품 목록 컴포넌트
제품 목록을 표시하는 컴포넌트를 만들어보세요.

```javascript
// 요구사항:
// - ProductList와 ProductCard 컴포넌트 분리
// - 장바구니 추가 기능
// - 빈 목록 시 메시지 표시
// - 로딩 상태 표시

const products = [
    { id: 1, name: '노트북', price: 1500000, image: '...' },
    { id: 2, name: '마우스', price: 50000, image: '...' }
];

function ProductList({ /* props */ }) {
    // 여기에 구현
}

function ProductCard({ /* props */ }) {
    // 여기에 구현
}

// 사용
<ProductList
    products={products}
    loading={false}
    onAddToCart={(productId) => console.log('추가:', productId)}
/>
```

## 다음 단계
다음 장에서는 **State와 생명주기**를 학습합니다:
- State를 사용한 동적 UI
- 컴포넌트 생명주기
- useEffect Hook
- State 업데이트 패턴