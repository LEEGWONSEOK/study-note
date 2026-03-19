# 19. Browser APIs

## 학습 목표
- 주요 Browser API 이해하기
- Local Storage와 Session Storage 활용하기
- Fetch API로 네트워크 요청 처리하기
- 최신 Browser API 활용 방법 학습하기

## Web Storage API

브라우저에 **키-값 쌍을 저장**하는 API입니다.

### Local Storage vs Session Storage

```javascript
// Local Storage: 영구 저장 (브라우저 닫아도 유지)
localStorage.setItem('username', '김철수');
localStorage.setItem('theme', 'dark');

// Session Storage: 세션 동안만 (탭 닫으면 삭제)
sessionStorage.setItem('tempData', 'temporary');

// 읽기
const username = localStorage.getItem('username'); // "김철수"
const theme = localStorage.getItem('theme'); // "dark"

// 삭제
localStorage.removeItem('theme');

// 전체 삭제
localStorage.clear();

// 키 개수
console.log(localStorage.length);

// 모든 키 순회
for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    const value = localStorage.getItem(key);
    console.log(key, value);
}
```

### 객체 저장

```javascript
// ❌ 직접 저장 불가 (문자열로 변환됨)
const user = { name: '김철수', age: 30 };
localStorage.setItem('user', user); // "[object Object]"

// ✅ JSON으로 변환하여 저장
localStorage.setItem('user', JSON.stringify(user));

// 읽을 때 파싱
const savedUser = JSON.parse(localStorage.getItem('user'));
console.log(savedUser.name); // "김철수"

// 헬퍼 함수
const storage = {
    set(key, value) {
        localStorage.setItem(key, JSON.stringify(value));
    },

    get(key, defaultValue = null) {
        const value = localStorage.getItem(key);
        if (value === null) return defaultValue;

        try {
            return JSON.parse(value);
        } catch (e) {
            return value;
        }
    },

    remove(key) {
        localStorage.removeItem(key);
    },

    clear() {
        localStorage.clear();
    }
};

// 사용
storage.set('user', { name: '김철수', age: 30 });
const user = storage.get('user'); // { name: '김철수', age: 30 }
```

### 실전 예제: 다크 모드

```javascript
class ThemeManager {
    constructor() {
        this.themeKey = 'theme';
        this.init();
    }

    init() {
        // 저장된 테마 불러오기
        const saved = localStorage.getItem(this.themeKey) || 'light';
        this.setTheme(saved);

        // 토글 버튼 이벤트
        const toggle = document.querySelector('.theme-toggle');
        toggle?.addEventListener('click', () => this.toggleTheme());
    }

    setTheme(theme) {
        document.body.dataset.theme = theme;
        localStorage.setItem(this.themeKey, theme);
    }

    toggleTheme() {
        const current = document.body.dataset.theme;
        const next = current === 'light' ? 'dark' : 'light';
        this.setTheme(next);
    }
}

const themeManager = new ThemeManager();
```

## Fetch API

**네트워크 요청을 처리**하는 현대적인 API입니다.

### 기본 사용법

```javascript
// GET 요청
fetch('https://api.example.com/users')
    .then(response => {
        // 응답 상태 확인
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        return response.json();
    })
    .then(data => {
        console.log('데이터:', data);
    })
    .catch(error => {
        console.error('에러:', error);
    });

// async/await 사용 (권장)
async function getUsers() {
    try {
        const response = await fetch('https://api.example.com/users');

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        console.log('데이터:', data);
        return data;
    } catch (error) {
        console.error('에러:', error);
    }
}
```

### 다양한 요청 메서드

```javascript
// POST 요청
async function createUser(user) {
    const response = await fetch('https://api.example.com/users', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(user)
    });

    return await response.json();
}

// PUT 요청 (전체 업데이트)
async function updateUser(id, user) {
    const response = await fetch(`https://api.example.com/users/${id}`, {
        method: 'PUT',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(user)
    });

    return await response.json();
}

// PATCH 요청 (부분 업데이트)
async function patchUser(id, updates) {
    const response = await fetch(`https://api.example.com/users/${id}`, {
        method: 'PATCH',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(updates)
    });

    return await response.json();
}

// DELETE 요청
async function deleteUser(id) {
    const response = await fetch(`https://api.example.com/users/${id}`, {
        method: 'DELETE'
    });

    return response.ok;
}
```

### 헤더와 인증

```javascript
// 인증 토큰 포함
async function fetchWithAuth(url) {
    const token = localStorage.getItem('token');

    const response = await fetch(url, {
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        }
    });

    return await response.json();
}

// 커스텀 헤더
const headers = new Headers();
headers.append('X-Custom-Header', 'value');
headers.append('Content-Type', 'application/json');

const response = await fetch(url, { headers });
```

### 요청 취소

```javascript
// AbortController 사용
const controller = new AbortController();
const signal = controller.signal;

fetch('https://api.example.com/data', { signal })
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => {
        if (error.name === 'AbortError') {
            console.log('요청 취소됨');
        }
    });

// 5초 후 취소
setTimeout(() => controller.abort(), 5000);

// 실전 예제: 검색 요청 취소
class SearchComponent {
    constructor() {
        this.controller = null;
    }

    async search(query) {
        // 이전 요청 취소
        if (this.controller) {
            this.controller.abort();
        }

        // 새 컨트롤러 생성
        this.controller = new AbortController();

        try {
            const response = await fetch(`/api/search?q=${query}`, {
                signal: this.controller.signal
            });

            const data = await response.json();
            this.displayResults(data);
        } catch (error) {
            if (error.name !== 'AbortError') {
                console.error('검색 에러:', error);
            }
        }
    }
}
```

## Intersection Observer API

**요소가 뷰포트에 보이는지 감지**하는 API입니다.

### 기본 사용법

```javascript
// 옵저버 생성
const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
        if (entry.isIntersecting) {
            console.log('요소가 보입니다:', entry.target);
            // 작업 수행
            entry.target.classList.add('visible');
        }
    });
}, {
    root: null,           // null = 뷰포트
    rootMargin: '0px',    // 여백
    threshold: 0.5        // 50% 보일 때
});

// 요소 관찰 시작
const element = document.querySelector('.lazy-load');
observer.observe(element);

// 관찰 중지
observer.unobserve(element);

// 전체 중지
observer.disconnect();
```

### 실전 예제: 이미지 지연 로딩

```javascript
class LazyLoader {
    constructor() {
        this.observer = new IntersectionObserver(
            this.handleIntersection.bind(this),
            { threshold: 0.1 }
        );
        this.init();
    }

    init() {
        const images = document.querySelectorAll('img[data-src]');
        images.forEach(img => this.observer.observe(img));
    }

    handleIntersection(entries) {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                this.loadImage(entry.target);
                this.observer.unobserve(entry.target);
            }
        });
    }

    loadImage(img) {
        const src = img.dataset.src;
        if (!src) return;

        img.src = src;
        img.removeAttribute('data-src');
        img.classList.add('loaded');
    }
}

// HTML: <img data-src="real-image.jpg" src="placeholder.jpg">
const lazyLoader = new LazyLoader();
```

### 무한 스크롤

```javascript
class InfiniteScroll {
    constructor(loadMore) {
        this.loadMore = loadMore;
        this.loading = false;
        this.hasMore = true;
        this.createSentinel();
    }

    createSentinel() {
        this.sentinel = document.createElement('div');
        this.sentinel.className = 'scroll-sentinel';
        document.body.appendChild(this.sentinel);

        this.observer = new IntersectionObserver(
            this.handleIntersection.bind(this),
            { threshold: 0.5 }
        );

        this.observer.observe(this.sentinel);
    }

    async handleIntersection(entries) {
        const entry = entries[0];

        if (entry.isIntersecting && !this.loading && this.hasMore) {
            this.loading = true;

            try {
                const hasMore = await this.loadMore();
                this.hasMore = hasMore;
            } finally {
                this.loading = false;
            }
        }
    }

    destroy() {
        this.observer.disconnect();
        this.sentinel.remove();
    }
}

// 사용
const infiniteScroll = new InfiniteScroll(async () => {
    const response = await fetch('/api/items?page=' + nextPage);
    const items = await response.json();

    displayItems(items);

    return items.length > 0; // 더 있으면 true
});
```

## Geolocation API

**사용자의 위치 정보**를 가져옵니다.

```javascript
// 현재 위치 가져오기
navigator.geolocation.getCurrentPosition(
    (position) => {
        const { latitude, longitude } = position.coords;
        console.log(`위치: ${latitude}, ${longitude}`);
    },
    (error) => {
        console.error('위치 에러:', error.message);
    },
    {
        enableHighAccuracy: true,  // 정확도 높임
        timeout: 5000,             // 타임아웃
        maximumAge: 0              // 캐시 사용 안 함
    }
);

// 위치 추적
const watchId = navigator.geolocation.watchPosition(
    (position) => {
        console.log('위치 업데이트:', position.coords);
    }
);

// 추적 중지
navigator.geolocation.clearWatch(watchId);
```

## Notification API

**데스크톱 알림**을 표시합니다.

```javascript
// 권한 요청
async function requestNotificationPermission() {
    const permission = await Notification.requestPermission();

    if (permission === 'granted') {
        console.log('알림 권한 허용됨');
    }
}

// 알림 표시
function showNotification(title, options = {}) {
    if (Notification.permission === 'granted') {
        new Notification(title, {
            body: options.body || '',
            icon: options.icon || '/icon.png',
            badge: options.badge || '/badge.png',
            tag: options.tag || 'default',
            requireInteraction: options.requireInteraction || false
        });
    }
}

// 사용
requestNotificationPermission().then(() => {
    showNotification('새 메시지', {
        body: '김철수님이 메시지를 보냈습니다.',
        icon: '/avatar.png'
    });
});
```

## Clipboard API

**클립보드 읽기/쓰기**를 처리합니다.

```javascript
// 텍스트 복사
async function copyText(text) {
    try {
        await navigator.clipboard.writeText(text);
        console.log('복사 완료');
    } catch (error) {
        console.error('복사 실패:', error);
    }
}

// 텍스트 읽기
async function pasteText() {
    try {
        const text = await navigator.clipboard.readText();
        console.log('붙여넣기:', text);
        return text;
    } catch (error) {
        console.error('읽기 실패:', error);
    }
}

// 실전 예제: 복사 버튼
document.querySelectorAll('.copy-btn').forEach(btn => {
    btn.addEventListener('click', async () => {
        const text = btn.dataset.text;
        await copyText(text);
        btn.textContent = '복사됨!';
        setTimeout(() => btn.textContent = '복사', 2000);
    });
});
```

## Web Workers

**백그라운드 스레드**에서 JavaScript를 실행합니다.

```javascript
// worker.js
self.addEventListener('message', (e) => {
    const result = heavyCalculation(e.data);
    self.postMessage(result);
});

function heavyCalculation(data) {
    // 무거운 계산
    let result = 0;
    for (let i = 0; i < 1000000000; i++) {
        result += i;
    }
    return result;
}

// main.js
const worker = new Worker('worker.js');

// 메시지 전송
worker.postMessage({ data: 'some data' });

// 결과 수신
worker.addEventListener('message', (e) => {
    console.log('결과:', e.data);
});

// 에러 처리
worker.addEventListener('error', (error) => {
    console.error('Worker 에러:', error);
});

// Worker 종료
worker.terminate();
```

## 실전 종합 예제

```javascript
// 사용자 인증 및 데이터 관리 시스템
class AuthManager {
    constructor() {
        this.apiUrl = 'https://api.example.com';
        this.tokenKey = 'auth_token';
    }

    // 로그인
    async login(email, password) {
        try {
            const response = await fetch(`${this.apiUrl}/login`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ email, password })
            });

            if (!response.ok) {
                throw new Error('로그인 실패');
            }

            const { token, user } = await response.json();

            // 토큰 저장
            localStorage.setItem(this.tokenKey, token);

            // 사용자 정보 저장
            localStorage.setItem('user', JSON.stringify(user));

            // 알림 표시
            if (Notification.permission === 'granted') {
                new Notification('로그인 성공', {
                    body: `환영합니다, ${user.name}님!`
                });
            }

            return user;
        } catch (error) {
            console.error('로그인 에러:', error);
            throw error;
        }
    }

    // 로그아웃
    logout() {
        localStorage.removeItem(this.tokenKey);
        localStorage.removeItem('user');
        window.location.href = '/login';
    }

    // 인증된 요청
    async fetchWithAuth(url, options = {}) {
        const token = localStorage.getItem(this.tokenKey);

        if (!token) {
            throw new Error('인증되지 않음');
        }

        const response = await fetch(url, {
            ...options,
            headers: {
                ...options.headers,
                'Authorization': `Bearer ${token}`
            }
        });

        if (response.status === 401) {
            // 토큰 만료
            this.logout();
        }

        return response;
    }

    // 현재 사용자
    getCurrentUser() {
        const user = localStorage.getItem('user');
        return user ? JSON.parse(user) : null;
    }

    // 인증 상태 확인
    isAuthenticated() {
        return !!localStorage.getItem(this.tokenKey);
    }
}

// 사용
const auth = new AuthManager();

// 로그인
await auth.login('user@example.com', 'password');

// 인증된 API 호출
const response = await auth.fetchWithAuth('/api/profile');
const profile = await response.json();
```

## 핵심 요약

### 1. Web Storage
```javascript
localStorage.setItem('key', 'value')
localStorage.getItem('key')
// 객체는 JSON.stringify/parse 사용
```

### 2. Fetch API
```javascript
const response = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
})
const data = await response.json()
```

### 3. Intersection Observer
```javascript
const observer = new IntersectionObserver(callback)
observer.observe(element)
```

### 4. 기타 API
- Geolocation: 위치 정보
- Notification: 알림
- Clipboard: 클립보드
- Web Workers: 백그라운드 작업

## 연습 문제

### 문제 1: Todo 앱 (Local Storage 사용)
새로고침해도 데이터가 유지되는 Todo 앱을 만들어보세요.

### 문제 2: 날씨 앱 (Fetch + Geolocation)
현재 위치의 날씨를 가져와 표시하는 앱을 만들어보세요.

### 문제 3: 이미지 갤러리 (Intersection Observer)
스크롤하면 이미지가 로드되는 갤러리를 만들어보세요.

### 문제 4: 채팅 알림 (Notification API)
새 메시지가 오면 데스크톱 알림을 표시하세요.

## 마무리
Browser API를 마스터했습니다! 이제 실전 프로젝트에 활용할 수 있습니다.