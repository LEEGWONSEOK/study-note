# Chapter 13. this와 실행 컨텍스트

## 13.1 this란?

> 🎯 **핵심**: `this`는 **함수 호출 방식에 따라 결정**됩니다.
> - Java와 다르게 동적으로 결정
> - 호출 컨텍스트가 중요
> - 화살표 함수는 예외

**비유**: `this`는 **대명사 "나"**입니다.
- 상황에 따라 가리키는 대상이 다름
- "나는 학생이다" (학교에서)
- "나는 직원이다" (회사에서)

**Java와 비교**:
```java
// Java - this는 항상 현재 인스턴스
public class Person {
    private String name;

    public Person(String name) {
        this.name = name;  // 항상 현재 인스턴스
    }

    public void greet() {
        System.out.println(this.name);  // 항상 현재 인스턴스
    }
}

// JavaScript - this는 호출 방식에 따라 다름
class Person {
    constructor(name) {
        this.name = name;
    }

    greet() {
        console.log(this.name);  // this는 호출 방식에 따라 다름!
    }
}

const person = new Person('홍길동');
person.greet();  // this = person

const greet = person.greet;
greet();  // this = undefined (strict mode) 또는 window
```

---

## 13.2 this 바인딩 규칙

### 1. 기본 바인딩 (Default Binding)

```javascript
function greet() {
    console.log(this);
}

greet();  // strict mode: undefined, non-strict: window/global
```

---

### 2. 암시적 바인딩 (Implicit Binding)

```javascript
const person = {
    name: '홍길동',
    greet() {
        console.log(`안녕, ${this.name}`);
    }
};

person.greet();  // this = person, "안녕, 홍길동"

// ⚠️ 메서드를 변수에 할당하면?
const greet = person.greet;
greet();  // this = undefined (strict mode), TypeError!
```

---

### 3. 명시적 바인딩 (Explicit Binding)

```javascript
function greet() {
    console.log(`안녕, ${this.name}`);
}

const person1 = { name: '홍길동' };
const person2 = { name: '김철수' };

// call - 즉시 호출
greet.call(person1);  // "안녕, 홍길동"
greet.call(person2);  // "안녕, 김철수"

// apply - 즉시 호출 (인자를 배열로)
function introduce(age, city) {
    console.log(`${this.name}, ${age}세, ${city}`);
}

introduce.call(person1, 25, '서울');  // "홍길동, 25세, 서울"
introduce.apply(person1, [25, '서울']);  // 동일

// bind - 새 함수 반환
const greetPerson1 = greet.bind(person1);
greetPerson1();  // "안녕, 홍길동"
```

---

### 4. new 바인딩

```javascript
function Person(name) {
    this.name = name;
    console.log(this);  // 새로 생성된 인스턴스
}

const person = new Person('홍길동');
// this = 새로 생성된 person 객체
```

---

### 5. 화살표 함수 (Lexical this)

```javascript
const person = {
    name: '홍길동',
    greet: function() {
        // 일반 함수 - this는 person
        console.log(this.name);  // "홍길동"

        setTimeout(function() {
            // 일반 함수 - this는 undefined (strict mode)
            console.log(this.name);  // undefined
        }, 1000);
    }
};

person.greet();

// ✅ 화살표 함수 사용
const person2 = {
    name: '김철수',
    greet: function() {
        console.log(this.name);  // "김철수"

        setTimeout(() => {
            // 화살표 함수 - this는 외부 스코프(person2)
            console.log(this.name);  // "김철수"
        }, 1000);
    }
};

person2.greet();
```

**비유**: 화살표 함수는 **아빠 찬스**
- 자신의 this가 없음
- 부모의 this를 물려받음

---

## 13.3 this 바인딩 우선순위

```
1. new 바인딩 (최우선)
2. 명시적 바인딩 (call, apply, bind)
3. 암시적 바인딩 (obj.method())
4. 기본 바인딩 (최하위)
5. 화살표 함수 (예외 - 렉시컬 스코프)
```

```javascript
function greet() {
    console.log(this.name);
}

const person1 = { name: '홍길동' };
const person2 = { name: '김철수' };

// 3. 암시적 바인딩
const obj = {
    name: '이영희',
    greet: greet
};
obj.greet();  // "이영희"

// 2. 명시적 바인딩 > 암시적 바인딩
obj.greet.call(person1);  // "홍길동"

// 1. new 바인딩 > 명시적 바인딩
function Person(name) {
    this.name = name;
}
const boundPerson = Person.bind(person2);
const newPerson = new boundPerson('박민수');
console.log(newPerson.name);  // "박민수" (new가 우선)
```

---

## 13.4 화살표 함수 심화

### 화살표 함수 this

```javascript
// 일반 함수
const obj1 = {
    name: '홍길동',
    greet: function() {
        console.log(this.name);  // this = obj1
    }
};

// 화살표 함수
const obj2 = {
    name: '김철수',
    greet: () => {
        console.log(this.name);  // this = 외부 스코프 (전역)
    }
};

obj1.greet();  // "홍길동" ✅
obj2.greet();  // undefined ❌
```

---

### 언제 사용하면 안 되나?

```javascript
// ❌ 객체 메서드
const person = {
    name: '홍길동',
    greet: () => {
        console.log(this.name);  // undefined (this = 전역)
    }
};

// ❌ 프로토타입 메서드
function Person(name) {
    this.name = name;
}

Person.prototype.greet = () => {
    console.log(this.name);  // undefined
};

// ❌ 생성자 함수
const Person = (name) => {
    this.name = name;  // 에러!
};
new Person('홍길동');  // TypeError: Person is not a constructor

// ❌ 이벤트 리스너 (this가 필요한 경우)
button.addEventListener('click', () => {
    console.log(this);  // window (button이 아님!)
});
```

---

### 언제 사용하면 좋은가?

```javascript
// ✅ 콜백 함수
const numbers = [1, 2, 3];
numbers.map(n => n * 2);  // 간결!

// ✅ 중첩 함수
const person = {
    name: '홍길동',
    hobbies: ['축구', '야구'],
    printHobbies() {
        this.hobbies.forEach(hobby => {
            // 화살표 함수로 외부 this 사용
            console.log(`${this.name}의 취미: ${hobby}`);
        });
    }
};

// ✅ 타이머 콜백
const counter = {
    count: 0,
    start() {
        setInterval(() => {
            this.count++;  // 화살표 함수로 외부 this 사용
            console.log(this.count);
        }, 1000);
    }
};
```

---

## 13.5 call, apply, bind

### call

```javascript
function greet(greeting, punctuation) {
    console.log(`${greeting}, ${this.name}${punctuation}`);
}

const person = { name: '홍길동' };

greet.call(person, '안녕하세요', '!');
// "안녕하세요, 홍길동!"
```

---

### apply

```javascript
function greet(greeting, punctuation) {
    console.log(`${greeting}, ${this.name}${punctuation}`);
}

const person = { name: '홍길동' };

// call과 동일하지만 인자를 배열로
greet.apply(person, ['안녕하세요', '!']);
// "안녕하세요, 홍길동!"

// 실용 예제: Math.max
const numbers = [1, 5, 3, 9, 2];
const max = Math.max.apply(null, numbers);
console.log(max);  // 9

// ES6 스프레드 연산자 (더 간결)
const max2 = Math.max(...numbers);
```

---

### bind

```javascript
function greet(greeting) {
    console.log(`${greeting}, ${this.name}`);
}

const person = { name: '홍길동' };

// 새 함수 반환 (즉시 호출 안 함)
const greetPerson = greet.bind(person);
greetPerson('안녕하세요');  // "안녕하세요, 홍길동"

// 부분 적용 (Partial Application)
const greetHello = greet.bind(person, '안녕하세요');
greetHello();  // "안녕하세요, 홍길동"

// 실용 예제: 이벤트 리스너
class Counter {
    constructor() {
        this.count = 0;
        this.button = document.getElementById('btn');
        // bind로 this 고정
        this.button.addEventListener('click', this.increment.bind(this));
    }

    increment() {
        this.count++;
        console.log(this.count);
    }
}
```

---

## 13.6 실행 컨텍스트 (Execution Context)

### 개념

```javascript
// 전역 실행 컨텍스트
const globalVar = 'global';

function outer() {
    // outer 함수 실행 컨텍스트
    const outerVar = 'outer';

    function inner() {
        // inner 함수 실행 컨텍스트
        const innerVar = 'inner';
        console.log(globalVar, outerVar, innerVar);
    }

    inner();
}

outer();  // "global outer inner"
```

**스코프 체인**:
```
inner 컨텍스트 → outer 컨텍스트 → 전역 컨텍스트
```

---

### 호이스팅 (Hoisting)

```javascript
// 함수 선언식 - 호이스팅됨
console.log(add(1, 2));  // 3 ✅

function add(a, b) {
    return a + b;
}

// 함수 표현식 - 호이스팅 안 됨
console.log(subtract(5, 3));  // TypeError ❌

const subtract = function(a, b) {
    return a - b;
};

// var - 호이스팅되지만 undefined
console.log(x);  // undefined ⚠️
var x = 10;

// let/const - Temporal Dead Zone
console.log(y);  // ReferenceError ❌
let y = 20;
```

---

## 13.7 클로저 (Closure)

### 기본 개념

```javascript
function outer() {
    const message = '안녕하세요';

    function inner() {
        console.log(message);  // 외부 변수 접근
    }

    return inner;
}

const fn = outer();
fn();  // "안녕하세요" (outer 실행 종료 후에도 접근 가능!)
```

**비유**: 클로저는 **배낭**입니다.
- 함수가 생성될 때 주변 환경을 배낭에 담음
- 나중에 사용할 수 있음

---

### Private 변수 구현

```javascript
function createCounter() {
    let count = 0;  // private 변수

    return {
        increment() {
            count++;
            return count;
        },
        decrement() {
            count--;
            return count;
        },
        getCount() {
            return count;
        }
    };
}

const counter = createCounter();
console.log(counter.increment());  // 1
console.log(counter.increment());  // 2
console.log(counter.getCount());   // 2
console.log(counter.count);        // undefined (접근 불가!)
```

---

### 실용 예제

```javascript
// 1. 부분 적용 함수
function multiply(a) {
    return function(b) {
        return a * b;
    };
}

const double = multiply(2);
const triple = multiply(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15

// 2. 메모이제이션
function memoize(fn) {
    const cache = {};

    return function(...args) {
        const key = JSON.stringify(args);

        if (key in cache) {
            console.log('캐시 사용');
            return cache[key];
        }

        console.log('계산 실행');
        const result = fn(...args);
        cache[key] = result;
        return result;
    };
}

const expensiveOperation = (n) => {
    return n * 2;
};

const memoized = memoize(expensiveOperation);
console.log(memoized(5));  // "계산 실행", 10
console.log(memoized(5));  // "캐시 사용", 10
```

---

## 13.8 실전 패턴

### 이벤트 핸들러 this 처리

```javascript
class Button {
    constructor(label) {
        this.label = label;
        this.clickCount = 0;
    }

    // ❌ 일반 메서드 - this 문제
    handleClick() {
        this.clickCount++;
        console.log(`${this.label} 클릭됨: ${this.clickCount}번`);
    }

    // ✅ 해결 1: bind 사용
    init1() {
        document.getElementById('btn').addEventListener(
            'click',
            this.handleClick.bind(this)
        );
    }

    // ✅ 해결 2: 화살표 함수
    init2() {
        document.getElementById('btn').addEventListener('click', () => {
            this.handleClick();
        });
    }

    // ✅ 해결 3: 클래스 필드 화살표 함수
    handleClick2 = () => {
        this.clickCount++;
        console.log(`${this.label} 클릭됨: ${this.clickCount}번`);
    }

    init3() {
        document.getElementById('btn').addEventListener(
            'click',
            this.handleClick2
        );
    }
}
```

---

### React에서 this 바인딩

```javascript
class MyComponent extends React.Component {
    constructor(props) {
        super(props);
        this.state = { count: 0 };

        // 방법 1: constructor에서 bind
        this.handleClick = this.handleClick.bind(this);
    }

    handleClick() {
        this.setState({ count: this.state.count + 1 });
    }

    // 방법 2: 클래스 필드 화살표 함수 (권장!)
    handleClick2 = () => {
        this.setState({ count: this.state.count + 1 });
    }

    render() {
        return (
            <div>
                <button onClick={this.handleClick}>클릭 1</button>
                <button onClick={this.handleClick2}>클릭 2</button>
                {/* 방법 3: 인라인 화살표 함수 (비권장 - 성능) */}
                <button onClick={() => this.handleClick()}>클릭 3</button>
            </div>
        );
    }
}
```

---

## 📝 핵심 요약

1. **this 바인딩**: 호출 방식에 따라 결정
   - 기본: undefined (strict mode)
   - 암시적: 객체 메서드 호출
   - 명시적: call, apply, bind
   - new: 새 인스턴스
   - 화살표: 렉시컬 스코프

2. **화살표 함수**: 자신의 this 없음
   - 외부 스코프의 this 사용
   - 콜백 함수에 적합
   - 메서드로는 부적합

3. **클로저**: 함수와 환경
   - 외부 변수 접근 유지
   - private 변수 구현
   - 메모이제이션 등

4. **실전 팁**:
   - 이벤트 핸들러는 bind 또는 화살표 함수
   - React에서는 클래스 필드 화살표 함수

---

## 🎯 다음 단계

[Chapter 14. 함수형 개념 →](../part05-functional/14-함수형-개념.md)

---

## 💪 연습 문제

```javascript
// 1. 다음 코드의 출력은?
const obj = {
    name: '홍길동',
    greet() {
        console.log(this.name);
    }
};

const greet = obj.greet;
greet();

// 2. 클로저를 사용한 카운터 구현
// increment, decrement, reset 메서드

// 3. 다음을 화살표 함수로 올바르게 변환
const person = {
    name: '홍길동',
    friends: ['김철수', '이영희'],
    printFriends: function() {
        this.friends.forEach(function(friend) {
            console.log(this.name + '의 친구: ' + friend);
        });
    }
};
```

<details>
<summary>정답 보기</summary>

```javascript
// 1. undefined (strict mode) 또는 TypeError
// 이유: greet 함수가 일반 함수로 호출되어 this가 undefined

// 해결책
const greet = obj.greet.bind(obj);
greet();  // "홍길동"

// 2. 클로저 카운터
function createCounter() {
    let count = 0;

    return {
        increment() {
            return ++count;
        },
        decrement() {
            return --count;
        },
        reset() {
            count = 0;
            return count;
        },
        getCount() {
            return count;
        }
    };
}

const counter = createCounter();
console.log(counter.increment());  // 1
console.log(counter.increment());  // 2
console.log(counter.decrement());  // 1
console.log(counter.reset());      // 0

// 3. 화살표 함수로 변환
const person = {
    name: '홍길동',
    friends: ['김철수', '이영희'],
    printFriends() {  // 메서드 축약
        this.friends.forEach(friend => {  // 화살표 함수
            console.log(`${this.name}의 친구: ${friend}`);
        });
    }
};

person.printFriends();
// "홍길동의 친구: 김철수"
// "홍길동의 친구: 이영희"
```

</details>

---

**작성일**: 2024년
**JavaScript 버전**: ES2015+ (ES6+)
**대상**: Java Spring Backend 개발자