# Chapter 10. Async/Await

## 10.1 Async/Await란?

> 🎯 **핵심**: Async/Await는 **Promise를 동기 코드처럼 작성**하는 문법입니다.
> - ES2017 (ES8) 도입
> - Promise의 Syntactic Sugar
> - 가독성 대폭 향상

**비유**: Async/Await는 **자동 번역기**입니다.
- Promise: 외국어 (then, catch 체이닝)
- Async/Await: 우리말 (순차적 코드)

**Java와 비교**:
```java
// Java - CompletableFuture
CompletableFuture<User> userFuture = fetchUserAsync(1);
CompletableFuture<List<Post>> postsFuture = userFuture.thenCompose(user ->
    fetchPostsAsync(user.getId())
);
postsFuture.thenAccept(posts -> {
    System.out.println(posts);
});

// JavaScript - Promise
fetchUser(1)
    .then(user => fetchPosts(user.id))
    .then(posts => console.log(posts));

// JavaScript - Async/Await (더 간결!)
async function displayPosts() {
    const user = await fetchUser(1);
    const posts = await fetchPosts(user.id);
    console.log(posts);
}
```

---

## 10.2 async 함수

### 기본 문법

```javascript
// async 함수는 항상 Promise를 반환
async function getData() {
    return "데이터";
}

// 위 코드는 아래와 동일
function getData() {
    return Promise.resolve("데이터");
}

getData().then(data => console.log(data));  // "데이터"
```

---

### 다양한 선언 방식

```javascript
// 1. 함수 선언식
async function fetchData() {
    return "데이터";
}

// 2. 함수 표현식
const fetchData = async function() {
    return "데이터";
};

// 3. 화살표 함수
const fetchData = async () => {
    return "데이터";
};

// 4. 메서드
const obj = {
    async fetchData() {
        return "데이터";
    }
};

// 5. 클래스 메서드
class DataService {
    async fetchData() {
        return "데이터";
    }
}
```

---

## 10.3 await 키워드

> 💡 **중요**: `await`는 **async 함수 내부에서만** 사용 가능!

### 기본 사용법

```javascript
function delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

// ❌ await는 async 함수 밖에서 사용 불가
await delay(1000);  // SyntaxError!

// ✅ async 함수 내부에서 사용
async function run() {
    console.log('시작');
    await delay(2000);  // 2초 대기
    console.log('완료');
}

run();
```

---

### Promise vs Async/Await

```javascript
// ❌ Promise 방식 - then 체이닝
function fetchUserData() {
    return fetchUser(1)
        .then(user => {
            console.log('사용자:', user);
            return fetchPosts(user.id);
        })
        .then(posts => {
            console.log('게시글:', posts);
            return fetchComments(posts[0].id);
        })
        .then(comments => {
            console.log('댓글:', comments);
            return comments;
        });
}

// ✅ Async/Await 방식 - 동기 코드처럼
async function fetchUserData() {
    const user = await fetchUser(1);
    console.log('사용자:', user);

    const posts = await fetchPosts(user.id);
    console.log('게시글:', posts);

    const comments = await fetchComments(posts[0].id);
    console.log('댓글:', comments);

    return comments;
}
```

---

## 10.4 에러 처리

### try-catch 사용

```javascript
// Promise 방식
fetchUser(1)
    .then(user => console.log(user))
    .catch(error => console.error('에러:', error));

// Async/Await 방식
async function getUser() {
    try {
        const user = await fetchUser(1);
        console.log(user);
    } catch (error) {
        console.error('에러:', error);
    }
}
```

---

### finally 블록

```javascript
async function fetchData() {
    showLoading();  // 로딩 표시

    try {
        const response = await fetch('/api/data');
        const data = await response.json();
        displayData(data);
    } catch (error) {
        showError(error);
    } finally {
        hideLoading();  // 항상 실행
    }
}
```

---

### 여러 에러 처리

```javascript
async function processUser(id) {
    try {
        const user = await fetchUser(id);

        if (!user.active) {
            throw new Error('비활성 사용자');
        }

        const posts = await fetchPosts(user.id);
        return posts;

    } catch (error) {
        if (error.message === '비활성 사용자') {
            console.warn('비활성 사용자입니다');
            return [];  // 기본값 반환
        }

        if (error.name === 'NetworkError') {
            console.error('네트워크 에러');
            throw error;  // 재발생
        }

        console.error('알 수 없는 에러:', error);
        throw error;
    }
}
```

---

## 10.5 병렬 처리

### 순차 실행 vs 병렬 실행

```javascript
// ❌ 순차 실행 (느림) - 6초 소요
async function fetchSequential() {
    const user = await fetchUser(1);      // 2초
    const posts = await fetchPosts(1);    // 2초
    const comments = await fetchComments(1); // 2초
    return { user, posts, comments };
}

// ✅ 병렬 실행 (빠름) - 2초 소요
async function fetchParallel() {
    const [user, posts, comments] = await Promise.all([
        fetchUser(1),      // 동시 실행
        fetchPosts(1),     // 동시 실행
        fetchComments(1)   // 동시 실행
    ]);
    return { user, posts, comments };
}
```

**비유**:
- 순차 실행: 음식을 하나씩 주문 (느림)
- 병렬 실행: 음식을 동시에 주문 (빠름)

---

### Promise.all과 함께 사용

```javascript
async function fetchAllUsers(ids) {
    try {
        // 모든 사용자를 병렬로 가져오기
        const users = await Promise.all(
            ids.map(id => fetchUser(id))
        );
        return users;
    } catch (error) {
        console.error('사용자 가져오기 실패:', error);
        throw error;
    }
}

// 사용
const userIds = [1, 2, 3, 4, 5];
const users = await fetchAllUsers(userIds);
console.log(users);
```

---

### Promise.allSettled와 함께 사용

```javascript
async function fetchAllData() {
    const results = await Promise.allSettled([
        fetchUser(1),
        fetchPosts(1),
        fetchComments(1)
    ]);

    const [userResult, postsResult, commentsResult] = results;

    const user = userResult.status === 'fulfilled'
        ? userResult.value
        : null;

    const posts = postsResult.status === 'fulfilled'
        ? postsResult.value
        : [];

    const comments = commentsResult.status === 'fulfilled'
        ? commentsResult.value
        : [];

    return { user, posts, comments };
}
```

---

## 10.6 고급 패턴

### 순차 실행 (배열 처리)

```javascript
const userIds = [1, 2, 3, 4, 5];

// ❌ forEach는 await 무시
userIds.forEach(async (id) => {
    const user = await fetchUser(id);  // 동시 실행됨!
    console.log(user);
});

// ✅ for...of 사용 (순차 실행)
async function processUsers() {
    for (const id of userIds) {
        const user = await fetchUser(id);  // 순차 실행
        console.log(user);
    }
}

// ✅ reduce 사용 (순차 실행)
async function processUsers2() {
    await userIds.reduce(async (prevPromise, id) => {
        await prevPromise;
        const user = await fetchUser(id);
        console.log(user);
    }, Promise.resolve());
}
```

---

### 병렬 처리 (배열)

```javascript
const userIds = [1, 2, 3, 4, 5];

// ✅ map + Promise.all (병렬 실행)
async function fetchAllUsers() {
    const users = await Promise.all(
        userIds.map(id => fetchUser(id))
    );
    return users;
}

// ✅ 결과 처리
async function processAllUsers() {
    const users = await Promise.all(
        userIds.map(async (id) => {
            const user = await fetchUser(id);
            return {
                ...user,
                posts: await fetchPosts(user.id)
            };
        })
    );
    return users;
}
```

---

### 조건부 await

```javascript
async function fetchData(useCache = true) {
    let data;

    if (useCache) {
        // 캐시에서 가져오기
        data = getFromCache();
    }

    if (!data) {
        // 캐시 없으면 API 호출
        data = await fetchFromAPI();
        saveToCache(data);
    }

    return data;
}

// 또는 삼항 연산자
async function getData(useCache) {
    return useCache
        ? getFromCache() || await fetchFromAPI()
        : await fetchFromAPI();
}
```

---

### 재시도 패턴

```javascript
async function retry(fn, maxRetries = 3, delay = 1000) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await fn();
        } catch (error) {
            if (i === maxRetries - 1) {
                throw error;  // 마지막 시도 실패
            }
            console.log(`재시도 ${i + 1}/${maxRetries}`);
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}

// 사용
async function fetchData() {
    return await retry(
        () => fetch('/api/data').then(r => r.json()),
        3,
        2000
    );
}
```

---

### 타임아웃 패턴

```javascript
function timeout(ms) {
    return new Promise((_, reject) =>
        setTimeout(() => reject(new Error('타임아웃')), ms)
    );
}

async function fetchWithTimeout(url, ms = 5000) {
    try {
        const response = await Promise.race([
            fetch(url),
            timeout(ms)
        ]);
        return await response.json();
    } catch (error) {
        if (error.message === '타임아웃') {
            console.error('요청 시간 초과');
        }
        throw error;
    }
}

// 사용
const data = await fetchWithTimeout('/api/data', 3000);
```

---

### 병렬 + 순차 조합

```javascript
async function complexFlow() {
    // 1단계: 병렬로 초기 데이터 가져오기
    const [user, settings] = await Promise.all([
        fetchUser(1),
        fetchSettings()
    ]);

    // 2단계: user 기반으로 순차 처리
    const posts = await fetchPosts(user.id);
    const comments = await fetchComments(posts[0].id);

    // 3단계: 병렬로 추가 데이터
    const [likes, shares] = await Promise.all([
        fetchLikes(comments[0].id),
        fetchShares(comments[0].id)
    ]);

    return { user, settings, posts, comments, likes, shares };
}
```

---

## 10.7 Top-level await (ES2022)

> 💡 **최신 기능**: 모듈 최상위에서 await 사용 가능!

```javascript
// ❌ 예전에는 불가능
await fetch('/api/data');  // SyntaxError!

// ✅ ES2022부터 가능 (모듈에서)
// app.js (type="module")
const response = await fetch('/api/data');
const data = await response.json();
console.log(data);

// 조건부 import
const theme = await loadTheme();
if (theme === 'dark') {
    await import('./dark-theme.js');
} else {
    await import('./light-theme.js');
}

// 설정 로딩
const config = await fetch('/config.json').then(r => r.json());
export default config;
```

**주의사항**:
- ES 모듈에서만 가능 (`type="module"`)
- 브라우저: 최신 버전에서만 지원
- Node.js: 14.8+ (`.mjs` 또는 `"type": "module"`)

---

## 10.8 실전 예제

### API 클라이언트 (Async/Await 버전)

```javascript
class ApiClient {
    constructor(baseURL) {
        this.baseURL = baseURL;
    }

    async request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`;
        const config = {
            headers: {
                'Content-Type': 'application/json',
                ...options.headers
            },
            ...options
        };

        try {
            const response = await fetch(url, config);

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }

            return await response.json();
        } catch (error) {
            console.error('API 요청 실패:', error);
            throw error;
        }
    }

    async get(endpoint) {
        return this.request(endpoint);
    }

    async post(endpoint, data) {
        return this.request(endpoint, {
            method: 'POST',
            body: JSON.stringify(data)
        });
    }

    async put(endpoint, data) {
        return this.request(endpoint, {
            method: 'PUT',
            body: JSON.stringify(data)
        });
    }

    async delete(endpoint) {
        return this.request(endpoint, {
            method: 'DELETE'
        });
    }
}

// 사용
const api = new ApiClient('https://api.example.com');

async function main() {
    try {
        const users = await api.get('/users');
        console.log('사용자:', users);

        const newUser = await api.post('/users', {
            name: '홍길동',
            email: 'hong@example.com'
        });
        console.log('생성됨:', newUser);

    } catch (error) {
        console.error('에러:', error);
    }
}

main();
```

---

### 데이터 로딩 컴포넌트 (React)

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        async function loadUser() {
            try {
                setLoading(true);
                setError(null);

                // 병렬로 데이터 가져오기
                const [userData, posts, followers] = await Promise.all([
                    fetchUser(userId),
                    fetchPosts(userId),
                    fetchFollowers(userId)
                ]);

                setUser({
                    ...userData,
                    posts,
                    followers
                });
            } catch (err) {
                setError(err.message);
            } finally {
                setLoading(false);
            }
        }

        loadUser();
    }, [userId]);

    if (loading) return <div>로딩 중...</div>;
    if (error) return <div>에러: {error}</div>;
    if (!user) return <div>사용자 없음</div>;

    return (
        <div>
            <h1>{user.name}</h1>
            <p>게시글: {user.posts.length}</p>
            <p>팔로워: {user.followers.length}</p>
        </div>
    );
}
```

---

### 폼 제출 처리

```javascript
async function handleSubmit(event) {
    event.preventDefault();

    const form = event.target;
    const formData = new FormData(form);
    const data = Object.fromEntries(formData);

    // 로딩 상태
    const submitButton = form.querySelector('button[type="submit"]');
    submitButton.disabled = true;
    submitButton.textContent = '제출 중...';

    try {
        // 유효성 검사
        await validateForm(data);

        // API 전송
        const response = await fetch('/api/submit', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data)
        });

        if (!response.ok) {
            throw new Error('제출 실패');
        }

        const result = await response.json();

        // 성공 처리
        showSuccessMessage('제출 완료!');
        form.reset();

    } catch (error) {
        // 에러 처리
        showErrorMessage(error.message);

    } finally {
        // 버튼 복구
        submitButton.disabled = false;
        submitButton.textContent = '제출';
    }
}

// 폼에 이벤트 리스너 등록
document.querySelector('form').addEventListener('submit', handleSubmit);
```

---

### 파일 업로드 (진행률 표시)

```javascript
async function uploadFile(file) {
    const formData = new FormData();
    formData.append('file', file);

    try {
        // XMLHttpRequest 대신 fetch 사용 시 진행률은 불가
        // 진행률이 필요하면 XMLHttpRequest 사용
        const response = await fetch('/api/upload', {
            method: 'POST',
            body: formData
        });

        if (!response.ok) {
            throw new Error('업로드 실패');
        }

        const result = await response.json();
        return result;

    } catch (error) {
        console.error('업로드 에러:', error);
        throw error;
    }
}

// 진행률이 필요한 경우 (Promise로 래핑)
function uploadFileWithProgress(file, onProgress) {
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        const formData = new FormData();
        formData.append('file', file);

        xhr.upload.addEventListener('progress', (e) => {
            if (e.lengthComputable) {
                const percent = (e.loaded / e.total) * 100;
                onProgress(percent);
            }
        });

        xhr.addEventListener('load', () => {
            if (xhr.status === 200) {
                resolve(JSON.parse(xhr.response));
            } else {
                reject(new Error('업로드 실패'));
            }
        });

        xhr.addEventListener('error', () => {
            reject(new Error('네트워크 에러'));
        });

        xhr.open('POST', '/api/upload');
        xhr.send(formData);
    });
}

// 사용
async function handleFileUpload(file) {
    try {
        const result = await uploadFileWithProgress(file, (percent) => {
            console.log(`업로드 진행률: ${percent.toFixed(2)}%`);
            updateProgressBar(percent);
        });

        console.log('업로드 완료:', result);
    } catch (error) {
        console.error('업로드 실패:', error);
    }
}
```

---

### 무한 스크롤

```javascript
class InfiniteScroll {
    constructor() {
        this.page = 1;
        this.loading = false;
        this.hasMore = true;

        window.addEventListener('scroll', () => {
            if (this.shouldLoadMore()) {
                this.loadMore();
            }
        });
    }

    shouldLoadMore() {
        const scrollY = window.scrollY;
        const visible = document.documentElement.clientHeight;
        const pageHeight = document.documentElement.scrollHeight;
        const bottomOfPage = visible + scrollY >= pageHeight - 300;

        return bottomOfPage && !this.loading && this.hasMore;
    }

    async loadMore() {
        this.loading = true;
        showLoadingSpinner();

        try {
            const response = await fetch(`/api/posts?page=${this.page}`);
            const posts = await response.json();

            if (posts.length === 0) {
                this.hasMore = false;
                showEndMessage();
            } else {
                appendPosts(posts);
                this.page++;
            }
        } catch (error) {
            console.error('로딩 실패:', error);
            showErrorMessage('데이터를 불러올 수 없습니다');
        } finally {
            this.loading = false;
            hideLoadingSpinner();
        }
    }
}

// 사용
const scroller = new InfiniteScroll();
```

---

## 📝 핵심 요약

1. **async 함수**: 항상 Promise 반환
   - `return "값"` → `Promise.resolve("값")`
   - `throw Error` → `Promise.reject(Error)`

2. **await**: Promise가 완료될 때까지 대기
   - async 함수 내부에서만 사용
   - Promise가 아닌 값도 사용 가능 (즉시 반환)

3. **에러 처리**: try-catch 사용
   - 동기 코드와 동일한 방식
   - finally로 정리 작업

4. **병렬 처리**: Promise.all 활용
   - 순차 실행: for...of + await
   - 병렬 실행: map + Promise.all

5. **패턴**:
   - 재시도, 타임아웃
   - 조건부 await
   - Top-level await (ES2022)

---

## 🎯 다음 단계

[Chapter 11. 객체와 프로토타입 →](../part04-oop/11-객체와-프로토타입.md)

---

## 💪 연습 문제

```javascript
// 1. 다음 Promise를 async/await로 변환
function getData() {
    return fetchUser(1)
        .then(user => fetchPosts(user.id))
        .then(posts => {
            console.log(posts);
            return posts;
        })
        .catch(error => {
            console.error(error);
            throw error;
        });
}

// 2. 배열의 모든 ID에 대해 순차적으로 사용자 가져오기
const userIds = [1, 2, 3, 4, 5];
// 힌트: for...of 사용

// 3. 배열의 모든 ID에 대해 병렬로 사용자 가져오기
// 힌트: Promise.all 사용

// 4. 재시도 기능이 있는 fetch 함수 작성
// 최대 3번 재시도, 각 시도 사이 1초 대기
```

<details>
<summary>정답 보기</summary>

```javascript
// 1. async/await 변환
async function getData() {
    try {
        const user = await fetchUser(1);
        const posts = await fetchPosts(user.id);
        console.log(posts);
        return posts;
    } catch (error) {
        console.error(error);
        throw error;
    }
}

// 2. 순차 실행
async function fetchUsersSequential(userIds) {
    const users = [];
    for (const id of userIds) {
        const user = await fetchUser(id);
        users.push(user);
        console.log(`사용자 ${id} 로딩 완료`);
    }
    return users;
}

// 3. 병렬 실행
async function fetchUsersParallel(userIds) {
    const users = await Promise.all(
        userIds.map(id => fetchUser(id))
    );
    return users;
}

// 4. 재시도 함수
async function fetchWithRetry(url, maxRetries = 3, delay = 1000) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            const response = await fetch(url);
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}`);
            }
            return await response.json();
        } catch (error) {
            console.log(`시도 ${i + 1} 실패:`, error.message);

            if (i === maxRetries - 1) {
                throw error;  // 마지막 시도 실패
            }

            // delay 후 재시도
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}

// 테스트
try {
    const data = await fetchWithRetry('/api/data', 3, 1000);
    console.log('성공:', data);
} catch (error) {
    console.error('모든 재시도 실패:', error);
}
```

</details>

---

**작성일**: 2024년
**JavaScript 버전**: ES2017+ (ES8+)
**대상**: Java Spring Backend 개발자