# 17. DOM 조작

## 학습 목표
- DOM(Document Object Model)의 개념 이해하기
- DOM 요소 선택과 조작 방법 학습하기
- DOM 탐색과 수정 기법 익히기
- 동적 콘텐츠 생성 방법 마스터하기

## DOM이란?

DOM은 **HTML 문서를 프로그래밍적으로 접근할 수 있는 인터페이스**입니다.

### 비유로 이해하기
DOM은 **건물의 설계도**와 같습니다:
- HTML: 건물의 청사진
- DOM: 실제로 조작 가능한 건물 모형
- JavaScript: 건물을 수정하는 도구
- 각 요소: 건물의 각 부품 (벽, 문, 창문 등)

### Java와 비교

```java
// Java Swing: UI 컴포넌트 조작
JFrame frame = new JFrame("My App");
JPanel panel = new JPanel();
JLabel label = new JLabel("Hello");

panel.add(label);
frame.add(panel);

// 나중에 수정
label.setText("World");
label.setForeground(Color.RED);
```

```javascript
// JavaScript: DOM 조작
const div = document.createElement('div');
const p = document.createElement('p');
p.textContent = 'Hello';

div.appendChild(p);
document.body.appendChild(div);

// 나중에 수정
p.textContent = 'World';
p.style.color = 'red';

// 유사하지만 DOM은 웹 페이지를 다룸
```

## DOM 요소 선택

### 기본 선택 메서드

```javascript
// 1. getElementById - ID로 선택
const header = document.getElementById('header');
console.log(header); // <div id="header">...</div>

// 2. getElementsByClassName - 클래스로 선택 (HTMLCollection 반환)
const buttons = document.getElementsByClassName('btn');
console.log(buttons); // HTMLCollection [button, button, ...]
console.log(buttons[0]); // 첫 번째 버튼

// 3. getElementsByTagName - 태그로 선택 (HTMLCollection 반환)
const paragraphs = document.getElementsByTagName('p');
console.log(paragraphs); // HTMLCollection [p, p, ...]

// 4. querySelector - CSS 선택자 (첫 번째 요소만)
const firstButton = document.querySelector('.btn');
const link = document.querySelector('a[href="#about"]');
const nested = document.querySelector('div > p.highlight');

// 5. querySelectorAll - CSS 선택자 (모든 요소, NodeList 반환)
const allButtons = document.querySelectorAll('.btn');
const allLinks = document.querySelectorAll('a');

console.log(allButtons); // NodeList [button, button, ...]
```

### HTMLCollection vs NodeList

```javascript
// HTMLCollection: 살아있는(live) 컬렉션
const htmlCollection = document.getElementsByClassName('item');
console.log(htmlCollection.length); // 3

// 새 요소 추가
const newItem = document.createElement('div');
newItem.className = 'item';
document.body.appendChild(newItem);

console.log(htmlCollection.length); // 4 (자동 업데이트!)

// NodeList: 정적(static) 컬렉션 (querySelectorAll)
const nodeList = document.querySelectorAll('.item');
console.log(nodeList.length); // 4

// 새 요소 추가
const anotherItem = document.createElement('div');
anotherItem.className = 'item';
document.body.appendChild(anotherItem);

console.log(nodeList.length); // 4 (변경 안 됨)

// 배열로 변환
const array1 = Array.from(htmlCollection);
const array2 = [...nodeList];

// forEach 사용 (NodeList만 지원)
nodeList.forEach(item => console.log(item));

// HTMLCollection은 for...of나 Array.from 사용
for (const item of htmlCollection) {
    console.log(item);
}
```

### 고급 선택 패턴

```javascript
// 여러 선택자 조합
const elements = document.querySelectorAll('div.container > p, div.sidebar > span');

// 속성 선택자
const required = document.querySelectorAll('input[required]');
const emails = document.querySelectorAll('input[type="email"]');

// 가상 클래스
const firstChild = document.querySelector('li:first-child');
const lastChild = document.querySelector('li:last-child');
const nthChild = document.querySelector('li:nth-child(3)');

// 부정 선택자
const notDisabled = document.querySelectorAll('button:not([disabled])');

// 특정 요소 내부에서 검색
const container = document.getElementById('container');
const buttonsInContainer = container.querySelectorAll('button');
```

## DOM 요소 조작

### 콘텐츠 변경

```javascript
const element = document.getElementById('content');

// 1. textContent - 텍스트만
element.textContent = 'Hello World';
console.log(element.textContent); // "Hello World"

// 2. innerHTML - HTML 포함
element.innerHTML = '<strong>Bold</strong> text';
console.log(element.innerHTML); // "<strong>Bold</strong> text"

// 3. innerText - 렌더링된 텍스트 (CSS 고려)
element.innerText = 'Visible text';

// 차이점 예시
const div = document.createElement('div');
div.innerHTML = '<span style="display:none">Hidden</span>Visible';

console.log(div.textContent); // "HiddenVisible"
console.log(div.innerText);   // "Visible" (숨겨진 텍스트 제외)
```

### 속성 조작

```javascript
const img = document.querySelector('img');

// getAttribute / setAttribute
const src = img.getAttribute('src');
img.setAttribute('src', 'new-image.jpg');
img.setAttribute('alt', 'New image');

// 속성 존재 확인
if (img.hasAttribute('alt')) {
    console.log('alt 속성이 있습니다');
}

// 속성 제거
img.removeAttribute('title');

// 직접 접근 (권장)
img.src = 'new-image.jpg';
img.alt = 'New image';

// data 속성
const element = document.getElementById('user');
element.dataset.userId = '123';
element.dataset.userName = 'John';

console.log(element.dataset); // { userId: "123", userName: "John" }
console.log(element.getAttribute('data-user-id')); // "123"
```

### 스타일 조작

```javascript
const box = document.querySelector('.box');

// 1. style 속성 (인라인 스타일)
box.style.color = 'red';
box.style.backgroundColor = 'blue'; // CSS: background-color
box.style.fontSize = '20px';

// 여러 스타일 한 번에
Object.assign(box.style, {
    width: '200px',
    height: '200px',
    border: '1px solid black',
    margin: '10px'
});

// 2. cssText (문자열로 한 번에)
box.style.cssText = 'color: red; background: blue; font-size: 20px;';

// 3. 계산된 스타일 가져오기 (실제 렌더링된 스타일)
const computed = window.getComputedStyle(box);
console.log(computed.color);
console.log(computed.backgroundColor);
console.log(computed.width); // "200px" (계산된 실제 값)
```

### 클래스 조작

```javascript
const element = document.querySelector('.item');

// 1. className (문자열)
element.className = 'item active';
console.log(element.className); // "item active"

// 2. classList (DOMTokenList - 더 편리)
element.classList.add('highlight');
element.classList.add('selected', 'important'); // 여러 개 추가

element.classList.remove('active');
element.classList.remove('selected', 'important');

// 토글 (있으면 제거, 없으면 추가)
element.classList.toggle('active');

// 존재 확인
if (element.classList.contains('highlight')) {
    console.log('highlight 클래스가 있습니다');
}

// 교체
element.classList.replace('old-class', 'new-class');

// 실용 예제: 버튼 클릭 시 활성 토글
const button = document.querySelector('button');
button.addEventListener('click', () => {
    button.classList.toggle('active');
});
```

## DOM 요소 생성과 삽입

### 요소 생성

```javascript
// 1. createElement
const div = document.createElement('div');
const p = document.createElement('p');
const img = document.createElement('img');

// 2. 내용 추가
div.textContent = 'Hello';
p.innerHTML = '<strong>Bold</strong> text';
img.src = 'image.jpg';

// 3. 속성 설정
div.id = 'container';
div.className = 'box';
img.alt = 'Description';

// 4. createTextNode (텍스트 노드 생성)
const text = document.createTextNode('Plain text');
p.appendChild(text);
```

### 요소 삽입

```javascript
const container = document.getElementById('container');
const newElement = document.createElement('div');
newElement.textContent = 'New element';

// 1. appendChild - 마지막 자식으로 추가
container.appendChild(newElement);

// 2. insertBefore - 특정 요소 앞에 삽입
const reference = container.children[0];
container.insertBefore(newElement, reference);

// 3. append - 여러 요소를 한 번에 (최신)
container.append(
    document.createElement('div'),
    'Text node',
    document.createElement('span')
);

// 4. prepend - 첫 번째 자식으로 추가
container.prepend(newElement);

// 5. insertAdjacentElement
const target = document.querySelector('.target');

// beforebegin: 요소 앞
target.insertAdjacentElement('beforebegin', newElement);

// afterbegin: 첫 번째 자식으로
target.insertAdjacentElement('afterbegin', newElement);

// beforeend: 마지막 자식으로
target.insertAdjacentElement('beforeend', newElement);

// afterend: 요소 뒤
target.insertAdjacentElement('afterend', newElement);

// 6. insertAdjacentHTML - HTML 문자열 삽입
target.insertAdjacentHTML('beforeend', '<p>New paragraph</p>');
```

### 요소 제거

```javascript
const element = document.querySelector('.item');

// 1. remove - 자기 자신을 제거 (최신)
element.remove();

// 2. removeChild - 부모에서 자식 제거 (이전 방식)
const parent = element.parentElement;
parent.removeChild(element);

// 3. 모든 자식 제거
const container = document.getElementById('container');

// 방법 1: innerHTML
container.innerHTML = '';

// 방법 2: 반복문
while (container.firstChild) {
    container.removeChild(container.firstChild);
}

// 방법 3: replaceChildren (최신)
container.replaceChildren();
```

### 요소 복제

```javascript
const original = document.querySelector('.original');

// 1. cloneNode(false) - 자식 없이 복제
const shallowCopy = original.cloneNode(false);

// 2. cloneNode(true) - 자식 포함 복제
const deepCopy = original.cloneNode(true);

// 복제된 요소 삽입
document.body.appendChild(deepCopy);

// 주의: 이벤트 리스너는 복제되지 않음!
original.addEventListener('click', () => console.log('clicked'));
const clone = original.cloneNode(true);
clone.click(); // 이벤트 발생 안 함
```

## DOM 탐색

### 부모/자식/형제 탐색

```javascript
const element = document.querySelector('.item');

// 부모 요소
const parent = element.parentElement;
const parentNode = element.parentNode; // 텍스트 노드도 포함

// 자식 요소
const children = element.children; // HTMLCollection (요소만)
const childNodes = element.childNodes; // NodeList (텍스트 노드 포함)

const firstChild = element.firstElementChild;
const lastChild = element.lastElementChild;

// 형제 요소
const next = element.nextElementSibling;
const prev = element.previousElementSibling;

// 가장 가까운 조상 요소 찾기
const closestContainer = element.closest('.container');
const closestForm = element.closest('form');

// 자식 요소 매칭
const hasButton = element.querySelector('button') !== null;
const matchesSelector = element.matches('.active'); // 선택자와 매칭되는지 확인
```

### 실전 탐색 예제

```javascript
// 1. 부모 컨테이너의 모든 버튼 찾기
const button = document.querySelector('.action-btn');
const container = button.closest('.container');
const allButtons = container.querySelectorAll('button');

// 2. 다음 형제 요소 모두 선택
function getNextSiblings(element) {
    const siblings = [];
    let next = element.nextElementSibling;

    while (next) {
        siblings.push(next);
        next = next.nextElementSibling;
    }

    return siblings;
}

// 3. 특정 조건의 자식 요소만 선택
const container = document.querySelector('.list');
const activeItems = Array.from(container.children)
    .filter(child => child.classList.contains('active'));

// 4. 부모 요소 체인 순회
function getParentChain(element) {
    const parents = [];
    let current = element.parentElement;

    while (current) {
        parents.push(current);
        current = current.parentElement;
    }

    return parents;
}
```

## 실전 예제

### 1. 동적 테이블 생성

```javascript
function createTable(data) {
    const table = document.createElement('table');
    table.className = 'data-table';

    // 헤더 생성
    const thead = document.createElement('thead');
    const headerRow = document.createElement('tr');

    Object.keys(data[0]).forEach(key => {
        const th = document.createElement('th');
        th.textContent = key;
        headerRow.appendChild(th);
    });

    thead.appendChild(headerRow);
    table.appendChild(thead);

    // 본문 생성
    const tbody = document.createElement('tbody');

    data.forEach(row => {
        const tr = document.createElement('tr');

        Object.values(row).forEach(value => {
            const td = document.createElement('td');
            td.textContent = value;
            tr.appendChild(td);
        });

        tbody.appendChild(tr);
    });

    table.appendChild(tbody);

    return table;
}

// 사용
const users = [
    { name: '김철수', age: 30, email: 'kim@example.com' },
    { name: '이영희', age: 25, email: 'lee@example.com' },
    { name: '박민수', age: 35, email: 'park@example.com' }
];

const table = createTable(users);
document.body.appendChild(table);
```

### 2. 아코디언 메뉴

```javascript
class Accordion {
    constructor(element) {
        this.element = element;
        this.items = element.querySelectorAll('.accordion-item');
        this.init();
    }

    init() {
        this.items.forEach(item => {
            const header = item.querySelector('.accordion-header');
            const content = item.querySelector('.accordion-content');

            // 초기 상태: 닫힘
            content.style.display = 'none';

            header.addEventListener('click', () => {
                this.toggle(item);
            });
        });
    }

    toggle(item) {
        const content = item.querySelector('.accordion-content');
        const isOpen = content.style.display !== 'none';

        // 다른 항목 모두 닫기
        this.items.forEach(otherItem => {
            if (otherItem !== item) {
                const otherContent = otherItem.querySelector('.accordion-content');
                otherContent.style.display = 'none';
                otherItem.classList.remove('active');
            }
        });

        // 현재 항목 토글
        if (isOpen) {
            content.style.display = 'none';
            item.classList.remove('active');
        } else {
            content.style.display = 'block';
            item.classList.add('active');
        }
    }
}

// 사용
const accordion = new Accordion(document.querySelector('.accordion'));
```

### 3. 탭 UI

```javascript
class Tabs {
    constructor(element) {
        this.element = element;
        this.tabs = element.querySelectorAll('[data-tab]');
        this.panels = element.querySelectorAll('[data-panel]');
        this.init();
    }

    init() {
        this.tabs.forEach(tab => {
            tab.addEventListener('click', () => {
                const target = tab.dataset.tab;
                this.showPanel(target);
            });
        });

        // 첫 번째 탭 활성화
        if (this.tabs.length > 0) {
            this.showPanel(this.tabs[0].dataset.tab);
        }
    }

    showPanel(target) {
        // 모든 탭 비활성화
        this.tabs.forEach(tab => {
            tab.classList.remove('active');
        });

        // 모든 패널 숨기기
        this.panels.forEach(panel => {
            panel.classList.remove('active');
            panel.style.display = 'none';
        });

        // 선택된 탭/패널 활성화
        const selectedTab = this.element.querySelector(`[data-tab="${target}"]`);
        const selectedPanel = this.element.querySelector(`[data-panel="${target}"]`);

        selectedTab.classList.add('active');
        selectedPanel.classList.add('active');
        selectedPanel.style.display = 'block';
    }
}

// HTML 구조:
// <div class="tabs">
//   <button data-tab="home">홈</button>
//   <button data-tab="profile">프로필</button>
//   <div data-panel="home">홈 내용</div>
//   <div data-panel="profile">프로필 내용</div>
// </div>

// 사용
const tabs = new Tabs(document.querySelector('.tabs'));
```

### 4. Todo 리스트

```javascript
class TodoList {
    constructor(element) {
        this.element = element;
        this.input = element.querySelector('.todo-input');
        this.addBtn = element.querySelector('.add-btn');
        this.list = element.querySelector('.todo-list');
        this.todos = [];
        this.init();
    }

    init() {
        this.addBtn.addEventListener('click', () => this.addTodo());
        this.input.addEventListener('keypress', (e) => {
            if (e.key === 'Enter') this.addTodo();
        });
    }

    addTodo() {
        const text = this.input.value.trim();
        if (!text) return;

        const todo = {
            id: Date.now(),
            text,
            completed: false
        };

        this.todos.push(todo);
        this.render();
        this.input.value = '';
    }

    toggleTodo(id) {
        const todo = this.todos.find(t => t.id === id);
        if (todo) {
            todo.completed = !todo.completed;
            this.render();
        }
    }

    deleteTodo(id) {
        this.todos = this.todos.filter(t => t.id !== id);
        this.render();
    }

    render() {
        this.list.innerHTML = '';

        this.todos.forEach(todo => {
            const li = document.createElement('li');
            li.className = 'todo-item';
            if (todo.completed) li.classList.add('completed');

            const checkbox = document.createElement('input');
            checkbox.type = 'checkbox';
            checkbox.checked = todo.completed;
            checkbox.addEventListener('change', () => this.toggleTodo(todo.id));

            const span = document.createElement('span');
            span.textContent = todo.text;

            const deleteBtn = document.createElement('button');
            deleteBtn.textContent = '삭제';
            deleteBtn.addEventListener('click', () => this.deleteTodo(todo.id));

            li.append(checkbox, span, deleteBtn);
            this.list.appendChild(li);
        });
    }
}

// 사용
const todoList = new TodoList(document.querySelector('.todo-app'));
```

## 성능 최적화

### DocumentFragment 사용

```javascript
// ❌ 비효율적: 매번 DOM 업데이트
const list = document.querySelector('ul');
for (let i = 0; i < 1000; i++) {
    const li = document.createElement('li');
    li.textContent = `Item ${i}`;
    list.appendChild(li); // 1000번 DOM 업데이트!
}

// ✅ 효율적: DocumentFragment 사용
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
    const li = document.createElement('li');
    li.textContent = `Item ${i}`;
    fragment.appendChild(li);
}
list.appendChild(fragment); // 1번만 DOM 업데이트
```

### 배치 스타일 변경

```javascript
const element = document.querySelector('.box');

// ❌ 비효율적: 여러 번 리플로우
element.style.width = '100px';    // 리플로우
element.style.height = '100px';   // 리플로우
element.style.border = '1px solid'; // 리플로우

// ✅ 효율적: cssText로 한 번에
element.style.cssText = 'width: 100px; height: 100px; border: 1px solid;';

// ✅ 또는 클래스 사용
element.className = 'box-styled';
```

## 핵심 요약

### 1. 요소 선택
```javascript
document.getElementById('id')
document.querySelector('.class')
document.querySelectorAll('div')
```

### 2. 콘텐츠 조작
```javascript
element.textContent = 'text'
element.innerHTML = '<b>html</b>'
element.setAttribute('attr', 'value')
element.classList.add('class')
```

### 3. 요소 생성/삽입
```javascript
const el = document.createElement('div')
parent.appendChild(el)
parent.insertAdjacentElement('beforeend', el)
```

### 4. 탐색
```javascript
element.parentElement
element.children
element.nextElementSibling
element.closest('.container')
```

## 연습 문제

### 문제 1: 드롭다운 메뉴
클릭하면 열리는 드롭다운 메뉴를 만들어보세요.

```javascript
// 요구사항:
// - 버튼 클릭 시 메뉴 토글
// - 외부 클릭 시 닫기
// - ESC 키로 닫기

function createDropdown(buttonSelector, menuSelector) {
    // 여기에 구현
}
```

### 문제 2: 이미지 갤러리
썸네일 클릭 시 큰 이미지를 표시하는 갤러리를 만들어보세요.

```javascript
// 요구사항:
// - 썸네일 클릭 시 메인 이미지 변경
// - 현재 선택된 썸네일 표시
// - 이전/다음 버튼

function createGallery(images) {
    // 여기에 구현
}
```

### 문제 3: 동적 폼
사용자가 입력 필드를 추가/삭제할 수 있는 폼을 만들어보세요.

```javascript
// 요구사항:
// - "필드 추가" 버튼
// - 각 필드에 삭제 버튼
// - 유효성 검사

function createDynamicForm() {
    // 여기에 구현
}
```

### 문제 4: 정렬 가능한 리스트
드래그 앤 드롭으로 순서를 바꿀 수 있는 리스트를 만들어보세요.

```javascript
// 요구사항:
// - 항목 드래그 가능
// - 드롭 위치 표시
// - 순서 변경 후 업데이트

function createSortableList(items) {
    // 여기에 구현
}
```

## 다음 단계
다음 장에서는 **이벤트**를 학습합니다:
- 이벤트 리스너
- 이벤트 객체
- 이벤트 전파
- 이벤트 위임