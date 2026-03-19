# Chapter 9. Promise

## 9.1 Promise란?

> 🎯 **핵심**: Promise는 **미래의 결과를 나타내는 객체**입니다.
> - 콜백 지옥 해결
> - 에러 처리 개선
> - 체이닝 가능

**비유**: Promise는 **주문 번호표**입니다.
- 주문하면 번호표 받음 (Promise 생성)
- 나중에 음식 나옴 (resolve) 또는 취소됨 (reject)
- 번호표로 상태 확인 가능

**Java와 비교**:
```java
// Java - CompletableFuture
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {}
    return "데이터";
});

future.thenAccept(data -> {
    System.out.println(data);
}).exceptionally(error -> {
    System.err.println("에러: " + error.getMessage());
    return null;
});

// JavaScript - Promise
const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve("데이터");
    }, 1000);
});

promise
    .then(data => {
        console.log(data);
    })
    .catch(error => {
        console.error("에러:", error.message);
    });
```

---

## 9.2 Promise 상태

Promise는 3가지 상태를 가집니다:

```
┌─────────────┐
│   Pending   │  ← 대기 (초기 상태)
└─────────────┘
       │
       ├─────────────┐
       │             │
       ▼             ▼
┌─────────────┐  ┌─────────────┐
│  Fulfilled  │  │  Rejected   │  ← 완료 (성공/실패)
└─────────────┘  └─────────────┘
```

- **Pending** (대기): 초기 상태
- **Fulfilled** (이행): 성공적으로 완료
- **Rejected** (거부): 실패

```javascript
// Pending 상태
const promise = new Promise((resolve, reject) => {
    // 아직 resolve나 reject 호출 안 됨
});

// Fulfilled 상태
const promise = new Promise((resolve, reject) => {
    resolve("성공!");
});

// Rejected 상태
const promise = new Promise((resolve, reject) => {
    reject(new Error("실패!"));
});
```

---

## 9.3 Promise 생성

### 기본 문법

```javascript
const promise = new Promise((resolve, reject) => {
    // 비동기 작업 수행
    const success = true;

    if (success) {
        resolve("성공 결과");  // 성공 시
    } else {
        reject(new Error("실패 이유"));  // 실패 시
    }
});
```

---

### 실용 예제

```javascript
// 1. 타이머
function delay(ms) {
    return new Promise(resolve => {
        setTimeout(resolve, ms);
    });
}

delay(2000).then(() => {
    console.log('2초 후 실행');
});

// 2. HTTP 요청
function fetchUser(id) {
    return new Promise((resolve, reject) => {
        fetch(`https://api.example.com/users/${id}`)
            .then(response => {
                if (!response.ok) {
                    reject(new Error(`HTTP ${response.status}`));
                } else {
                    return response.json();
                }
            })
            .then(data => resolve(data))
            .catch(error => reject(error));
    });
}

// 3. 파일 읽기 (Node.js를 Promise로 래핑)
const fs = require('fs');

function readFilePromise(path) {
    return new Promise((resolve, reject) => {
        fs.readFile(path, 'utf8', (err, data) => {
            if (err) {
                reject(err);
            } else {
                resolve(data);
            }
        });
    });
}

readFilePromise('data.txt')
    .then(content => console.log(content))
    .catch(err => console.error(err));
```

---

## 9.4 Promise 사용

### then - 성공 처리

```javascript
const promise = new Promise(resolve => {
    setTimeout(() => {
        resolve("데이터");
    }, 1000);
});

promise.then(data => {
    console.log(data);  // "데이터"
});

// 반환값이 다음 then으로 전달
promise
    .then(data => {
        console.log('첫 번째:', data);
        return data.toUpperCase();
    })
    .then(upperData => {
        console.log('두 번째:', upperData);  // "데이터"의 대문자
    });
```

---

### catch - 에러 처리

```javascript
const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
        reject(new Error("에러 발생!"));
    }, 1000);
});

promise
    .then(data => {
        console.log(data);  // 실행 안 됨
    })
    .catch(error => {
        console.error('에러:', error.message);  // "에러 발생!"
    });

// then의 두 번째 인자로도 가능 (비권장)
promise.then(
    data => console.log(data),
    error => console.error(error)
);
```

---

### finally - 항상 실행

```javascript
promise
    .then(data => {
        console.log('성공:', data);
    })
    .catch(error => {
        console.error('실패:', error);
    })
    .finally(() => {
        console.log('완료 (성공/실패 무관)');
        // 로딩 스피너 제거 등
    });

// 실용 예제
function fetchData() {
    showLoading();  // 로딩 표시

    return fetch('/api/data')
        .then(response => response.json())
        .then(data => {
            displayData(data);
        })
        .catch(error => {
            showError(error);
        })
        .finally(() => {
            hideLoading();  // 항상 로딩 제거
        });
}
```

---

## 9.5 Promise 체이닝

### 기본 체이닝

```javascript
// ❌ 콜백 지옥
fetchUser(1, (err, user) => {
    if (err) return console.error(err);

    fetchPosts(user.id, (err, posts) => {
        if (err) return console.error(err);

        fetchComments(posts[0].id, (err, comments) => {
            if (err) return console.error(err);
            console.log(comments);
        });
    });
});

// ✅ Promise 체이닝
fetchUser(1)
    .then(user => fetchPosts(user.id))
    .then(posts => fetchComments(posts[0].id))
    .then(comments => console.log(comments))
    .catch(error => console.error(error));
```

---

### 값 전달

```javascript
Promise.resolve(5)
    .then(num => {
        console.log(num);  // 5
        return num * 2;
    })
    .then(num => {
        console.log(num);  // 10
        return num + 3;
    })
    .then(num => {
        console.log(num);  // 13
    });
```

---

### Promise 반환

```javascript
function step1() {
    return Promise.resolve('Step 1 완료');
}

function step2(prevResult) {
    return new Promise(resolve => {
        setTimeout(() => {
            resolve(`Step 2 완료, 이전: ${prevResult}`);
        }, 1000);
    });
}

function step3(prevResult) {
    return Promise.resolve(`Step 3 완료, 이전: ${prevResult}`);
}

// 체이닝
step1()
    .then(result1 => {
        console.log(result1);
        return step2(result1);
    })
    .then(result2 => {
        console.log(result2);
        return step3(result2);
    })
    .then(result3 => {
        console.log(result3);
    })
    .catch(error => {
        console.error('에러:', error);
    });
```

---

### 중간 에러 처리

```javascript
fetchUser(1)
    .then(user => {
        if (!user.active) {
            throw new Error('비활성 사용자');
        }
        return user;
    })
    .then(user => fetchPosts(user.id))
    .catch(error => {
        console.error('에러 발생:', error);
        // 기본값 반환으로 체이닝 계속
        return [];
    })
    .then(posts => {
        console.log('게시글:', posts);  // 에러 시 []
    });
```

---

## 9.6 Promise 정적 메서드

### Promise.resolve / Promise.reject

```javascript
// 즉시 이행된 Promise
Promise.resolve(42)
    .then(value => console.log(value));  // 42

// 즉시 거부된 Promise
Promise.reject(new Error('에러'))
    .catch(error => console.error(error));

// Promise가 아닌 값을 Promise로 변환
async function getData() {
    return 42;  // 자동으로 Promise.resolve(42)
}
```

---

### Promise.all - 모두 완료 대기

```javascript
const promise1 = delay(1000).then(() => 'A');
const promise2 = delay(2000).then(() => 'B');
const promise3 = delay(1500).then(() => 'C');

// 모든 Promise가 완료될 때까지 대기
Promise.all([promise1, promise2, promise3])
    .then(results => {
        console.log(results);  // ['A', 'B', 'C'] (2초 후)
    })
    .catch(error => {
        console.error('하나라도 실패하면 즉시 catch');
    });

// 실용 예제: 여러 API 동시 호출
Promise.all([
    fetch('/api/user'),
    fetch('/api/posts'),
    fetch('/api/comments')
])
    .then(responses => Promise.all(responses.map(r => r.json())))
    .then(([user, posts, comments]) => {
        console.log('사용자:', user);
        console.log('게시글:', posts);
        console.log('댓글:', comments);
    })
    .catch(error => console.error('API 에러:', error));
```

**비유**: Promise.all은 **단체 사진**입니다.
- 모두 준비되어야 찍음
- 한 명이라도 빠지면 실패

---

### Promise.allSettled - 모두 완료 (성공/실패 무관)

```javascript
const promises = [
    Promise.resolve('성공 1'),
    Promise.reject(new Error('실패')),
    Promise.resolve('성공 2')
];

Promise.allSettled(promises)
    .then(results => {
        console.log(results);
        // [
        //   { status: 'fulfilled', value: '성공 1' },
        //   { status: 'rejected', reason: Error: 실패 },
        //   { status: 'fulfilled', value: '성공 2' }
        // ]

        results.forEach((result, index) => {
            if (result.status === 'fulfilled') {
                console.log(`${index}: 성공`, result.value);
            } else {
                console.log(`${index}: 실패`, result.reason);
            }
        });
    });

// 실용 예제: 여러 API 호출, 일부 실패해도 계속
Promise.allSettled([
    fetch('/api/user'),
    fetch('/api/posts'),
    fetch('/api/settings')  // 실패 가능
])
    .then(results => {
        const [userResult, postsResult, settingsResult] = results;

        if (userResult.status === 'fulfilled') {
            displayUser(userResult.value);
        }

        if (postsResult.status === 'fulfilled') {
            displayPosts(postsResult.value);
        }

        if (settingsResult.status === 'rejected') {
            console.warn('설정 로딩 실패, 기본값 사용');
        }
    });
```

---

### Promise.race - 가장 빨리 완료된 것

```javascript
const promise1 = delay(1000).then(() => 'A');
const promise2 = delay(2000).then(() => 'B');
const promise3 = delay(1500).then(() => 'C');

Promise.race([promise1, promise2, promise3])
    .then(result => {
        console.log(result);  // 'A' (1초 후, 가장 빠름)
    });

// 실용 예제: 타임아웃 구현
function fetchWithTimeout(url, timeout = 5000) {
    return Promise.race([
        fetch(url),
        new Promise((_, reject) =>
            setTimeout(() => reject(new Error('타임아웃')), timeout)
        )
    ]);
}

fetchWithTimeout('/api/data', 3000)
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error(error));  // 3초 초과 시 타임아웃
```

---

### Promise.any - 가장 빨리 성공한 것

```javascript
const promises = [
    Promise.reject(new Error('에러 1')),
    delay(1000).then(() => 'B'),
    delay(500).then(() => 'C')
];

Promise.any(promises)
    .then(result => {
        console.log(result);  // 'C' (가장 빠른 성공)
    })
    .catch(error => {
        console.error('모두 실패:', error);  // AggregateError
    });

// 실용 예제: 여러 서버 중 가장 빠른 응답 사용
Promise.any([
    fetch('https://server1.example.com/api/data'),
    fetch('https://server2.example.com/api/data'),
    fetch('https://server3.example.com/api/data')
])
    .then(response => response.json())
    .then(data => console.log('가장 빠른 서버 응답:', data))
    .catch(error => console.error('모든 서버 실패'));
```

---

## 9.7 Promise 패턴

### 순차 실행 (Sequential)

```javascript
const tasks = [
    () => delay(1000).then(() => console.log('작업 1')),
    () => delay(1000).then(() => console.log('작업 2')),
    () => delay(1000).then(() => console.log('작업 3'))
];

// reduce로 순차 실행
tasks.reduce(
    (prevPromise, task) => prevPromise.then(task),
    Promise.resolve()
);

// 또는 for...of + await (더 직관적)
async function runSequential(tasks) {
    for (const task of tasks) {
        await task();
    }
}

runSequential(tasks);
```

---

### 병렬 실행 (Parallel)

```javascript
const tasks = [
    () => delay(1000).then(() => console.log('작업 1')),
    () => delay(1000).then(() => console.log('작업 2')),
    () => delay(1000).then(() => console.log('작업 3'))
];

// Promise.all로 병렬 실행
Promise.all(tasks.map(task => task()))
    .then(() => console.log('모두 완료'));
// 모든 작업이 동시에 시작, 1초 후 완료
```

---

### 재시도 (Retry)

```javascript
function retry(fn, maxRetries = 3, delay = 1000) {
    return fn().catch(error => {
        if (maxRetries <= 0) {
            throw error;
        }

        console.log(`재시도 (${maxRetries}번 남음)`);
        return new Promise(resolve =>
            setTimeout(() => resolve(retry(fn, maxRetries - 1, delay)), delay)
        );
    });
}

// 사용
retry(() => fetch('/api/data'))
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error('모든 재시도 실패:', error));
```

---

### 캐싱 (Memoization)

```javascript
function memoizePromise(fn) {
    const cache = new Map();

    return function(...args) {
        const key = JSON.stringify(args);

        if (cache.has(key)) {
            console.log('캐시 사용');
            return cache.get(key);
        }

        const promise = fn(...args);
        cache.set(key, promise);
        return promise;
    };
}

// 사용
const fetchUser = memoizePromise((id) => {
    console.log('API 호출');
    return fetch(`/api/users/${id}`).then(r => r.json());
});

fetchUser(1).then(user => console.log(user));  // API 호출
fetchUser(1).then(user => console.log(user));  // 캐시 사용
```

---

## 9.8 에러 처리 심화

### 에러 복구

```javascript
fetchUser(1)
    .then(user => {
        if (!user) {
            throw new Error('사용자 없음');
        }
        return user;
    })
    .catch(error => {
        console.warn('에러 발생, 기본 사용자 사용:', error);
        // 기본값 반환으로 체이닝 계속
        return { id: 1, name: 'Guest' };
    })
    .then(user => {
        console.log('사용자:', user);  // Guest 또는 실제 사용자
    });
```

---

### 에러 재발생

```javascript
fetchData()
    .catch(error => {
        console.error('에러 발생:', error);

        // 에러 처리 후 다시 발생
        if (error.status === 404) {
            return null;  // 404는 복구
        }

        throw error;  // 다른 에러는 재발생
    })
    .then(data => {
        if (data === null) {
            console.log('데이터 없음');
        } else {
            console.log('데이터:', data);
        }
    })
    .catch(error => {
        console.error('처리 불가 에러:', error);
    });
```

---

### 커스텀 에러

```javascript
class NetworkError extends Error {
    constructor(message, status) {
        super(message);
        this.name = 'NetworkError';
        this.status = status;
    }
}

function fetchData(url) {
    return fetch(url)
        .then(response => {
            if (!response.ok) {
                throw new NetworkError(
                    `HTTP ${response.status}`,
                    response.status
                );
            }
            return response.json();
        });
}

fetchData('/api/data')
    .catch(error => {
        if (error instanceof NetworkError) {
            if (error.status === 404) {
                console.log('데이터 없음');
            } else if (error.status >= 500) {
                console.error('서버 에러');
            }
        } else {
            console.error('알 수 없는 에러:', error);
        }
    });
```

---

## 9.9 실전 예제

### API 클라이언트

```javascript
class ApiClient {
    constructor(baseURL) {
        this.baseURL = baseURL;
    }

    request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`;
        const config = {
            headers: {
                'Content-Type': 'application/json',
                ...options.headers
            },
            ...options
        };

        return fetch(url, config)
            .then(response => {
                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }
                return response.json();
            });
    }

    get(endpoint) {
        return this.request(endpoint);
    }

    post(endpoint, data) {
        return this.request(endpoint, {
            method: 'POST',
            body: JSON.stringify(data)
        });
    }

    put(endpoint, data) {
        return this.request(endpoint, {
            method: 'PUT',
            body: JSON.stringify(data)
        });
    }

    delete(endpoint) {
        return this.request(endpoint, {
            method: 'DELETE'
        });
    }
}

// 사용
const api = new ApiClient('https://api.example.com');

api.get('/users')
    .then(users => console.log(users))
    .catch(error => console.error(error));

api.post('/users', { name: '홍길동' })
    .then(user => console.log('생성됨:', user))
    .catch(error => console.error(error));
```

---

### 이미지 프리로딩

```javascript
function loadImage(url) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => resolve(img);
        img.onerror = () => reject(new Error(`이미지 로딩 실패: ${url}`));
        img.src = url;
    });
}

// 여러 이미지 프리로딩
const imageUrls = [
    '/images/img1.jpg',
    '/images/img2.jpg',
    '/images/img3.jpg'
];

Promise.all(imageUrls.map(loadImage))
    .then(images => {
        console.log('모든 이미지 로딩 완료');
        images.forEach(img => document.body.appendChild(img));
    })
    .catch(error => console.error(error));
```

---

## 📝 핵심 요약

1. **Promise**: 비동기 작업의 최종 결과를 나타내는 객체
   - Pending → Fulfilled / Rejected

2. **체이닝**: `.then()` 연결로 순차 처리
   - 콜백 지옥 해결
   - 에러는 `.catch()`로 한 번에 처리

3. **정적 메서드**:
   - `Promise.all`: 모두 성공 필요
   - `Promise.allSettled`: 모두 완료 (결과 무관)
   - `Promise.race`: 가장 빠른 것
   - `Promise.any`: 가장 빠른 성공

4. **패턴**:
   - 순차/병렬 실행
   - 재시도, 타임아웃
   - 캐싱

---

## 🎯 다음 단계

[Chapter 10. Async/Await →](10-async-await.md)

---

## 💪 연습 문제

```javascript
// 1. delay 함수 구현
// ms 후에 resolve되는 Promise 반환
function delay(ms) {
    // TODO
}

// 2. 순차 실행 함수 구현
// 배열의 Promise를 순서대로 실행
function sequential(promises) {
    // TODO
}

// 3. 타임아웃 함수 구현
// promise가 timeout ms 내에 완료되지 않으면 에러
function withTimeout(promise, timeout) {
    // TODO
}
```

<details>
<summary>정답 보기</summary>

```javascript
// 1. delay 구현
function delay(ms) {
    return new Promise(resolve => {
        setTimeout(resolve, ms);
    });
}

// 테스트
delay(2000).then(() => console.log('2초 후'));

// 2. sequential 구현
function sequential(promises) {
    return promises.reduce(
        (chain, promise) => chain.then(() => promise()),
        Promise.resolve()
    );
}

// 테스트
const tasks = [
    () => delay(1000).then(() => console.log('1')),
    () => delay(1000).then(() => console.log('2')),
    () => delay(1000).then(() => console.log('3'))
];
sequential(tasks);  // 1초마다 1, 2, 3 출력

// 3. withTimeout 구현
function withTimeout(promise, timeout) {
    return Promise.race([
        promise,
        new Promise((_, reject) =>
            setTimeout(
                () => reject(new Error('타임아웃')),
                timeout
            )
        )
    ]);
}

// 테스트
withTimeout(fetch('/api/data'), 3000)
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error(error));  // 3초 초과 시 타임아웃
```

</details>

---

**작성일**: 2024년
**JavaScript 버전**: ES2015+ (ES6+)
**대상**: Java Spring Backend 개발자