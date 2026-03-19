# 부록 A. 필수 도구

## 개발 환경 설정

### Node.js와 npm

**Node.js**는 JavaScript 런타임이고, **npm**은 패키지 관리자입니다.

```bash
# Node.js 설치 확인
node --version  # v18.0.0 이상 권장

# npm 버전 확인
npm --version

# 프로젝트 초기화
npm init -y

# 패키지 설치
npm install react
npm install --save-dev jest

# 전역 설치
npm install -g create-react-app

# 패키지 실행
npx create-react-app my-app
```

### 패키지 관리자 비교

```bash
# npm (기본)
npm install package-name
npm run dev

# yarn (빠름)
yarn add package-name
yarn dev

# pnpm (디스크 효율적)
pnpm add package-name
pnpm dev
```

## 빌드 도구

### Vite (권장)

현대적이고 빠른 빌드 도구입니다.

```bash
# 프로젝트 생성
npm create vite@latest my-app -- --template react

# 개발 서버 실행
npm run dev

# 프로덕션 빌드
npm run build
```

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000
  },
  build: {
    outDir: 'dist'
  }
})
```

### Webpack (전통적)

```bash
# 설치
npm install --save-dev webpack webpack-cli webpack-dev-server

# webpack.config.js 설정
```

```javascript
// webpack.config.js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: 'babel-loader'
      }
    ]
  },
  devServer: {
    port: 3000
  }
};
```

## 코드 품질 도구

### ESLint (린터)

코드 스타일과 오류를 검사합니다.

```bash
# 설치
npm install --save-dev eslint

# 초기화
npx eslint --init

# 실행
npx eslint src/
```

```javascript
// .eslintrc.json
{
  "env": {
    "browser": true,
    "es2021": true
  },
  "extends": [
    "eslint:recommended",
    "plugin:react/recommended"
  ],
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "rules": {
    "indent": ["error", 2],
    "quotes": ["error", "single"],
    "semi": ["error", "always"]
  }
}
```

### Prettier (포매터)

코드를 자동으로 포맷팅합니다.

```bash
# 설치
npm install --save-dev prettier

# 실행
npx prettier --write src/
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 80
}
```

### ESLint + Prettier 통합

```bash
npm install --save-dev eslint-config-prettier eslint-plugin-prettier
```

```javascript
// .eslintrc.json
{
  "extends": [
    "eslint:recommended",
    "plugin:prettier/recommended"
  ]
}
```

## 버전 관리

### Git 기본 명령어

```bash
# 저장소 초기화
git init

# 상태 확인
git status

# 파일 추가
git add .
git add filename.js

# 커밋
git commit -m "커밋 메시지"

# 원격 저장소 연결
git remote add origin https://github.com/username/repo.git

# 푸시
git push origin main

# 풀
git pull origin main

# 브랜치
git branch feature-name
git checkout feature-name
git checkout -b feature-name  # 생성하고 전환

# 병합
git checkout main
git merge feature-name

# 로그
git log --oneline
```

### .gitignore

```bash
# .gitignore
node_modules/
dist/
build/
.env
.env.local
*.log
.DS_Store
coverage/
```

## VS Code 확장 프로그램

### 필수 확장

1. **ESLint** - 코드 린팅
2. **Prettier** - 코드 포맷팅
3. **ES7+ React/Redux/React-Native snippets** - 스니펫
4. **Auto Rename Tag** - 태그 자동 수정
5. **Bracket Pair Colorizer** - 괄호 색상
6. **GitLens** - Git 통합
7. **Path Intellisense** - 경로 자동완성
8. **Import Cost** - 임포트 크기 표시

### VS Code 설정

```json
// settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "javascript.updateImportsOnFileMove.enabled": "always",
  "files.autoSave": "onFocusChange"
}
```

### 유용한 단축키

```
// 편집
Ctrl/Cmd + D      - 다음 동일 단어 선택
Ctrl/Cmd + /      - 주석 토글
Alt + ↑/↓         - 줄 이동
Shift + Alt + ↑/↓ - 줄 복사
Ctrl/Cmd + Shift + K - 줄 삭제

// 탐색
Ctrl/Cmd + P      - 파일 검색
Ctrl/Cmd + Shift + F - 전체 검색
Ctrl/Cmd + G      - 줄 이동
Ctrl/Cmd + B      - 사이드바 토글

// 실행
F5               - 디버깅 시작
```

## 브라우저 개발자 도구

### Chrome DevTools 주요 기능

```javascript
// Console
console.log('일반 로그')
console.warn('경고')
console.error('에러')
console.table([{ name: 'John', age: 30 }])
console.time('작업')
// ... 코드
console.timeEnd('작업')

// 그룹
console.group('그룹 이름')
console.log('항목 1')
console.log('항목 2')
console.groupEnd()

// 조건부 로그
const x = 5
console.assert(x > 10, 'x는 10보다 커야 함')
```

### 디버깅

```javascript
// debugger 문 사용
function calculate(a, b) {
    debugger; // 여기서 중단
    return a + b;
}

// 브레이크포인트 설정: Sources 탭에서 줄 번호 클릭
```

### React DevTools

```bash
# Chrome 확장 프로그램 설치
# React Developer Tools
```

- Components 탭: 컴포넌트 트리 확인
- Profiler 탭: 성능 프로파일링

## package.json 스크립트

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "vite": "^4.0.0"
  }
}
```

```bash
# 실행
npm run dev
npm run build
npm run lint
```

## 유용한 npm 패키지

### 유틸리티

```bash
# Lodash - 유틸리티 함수
npm install lodash

# Day.js - 날짜 처리
npm install dayjs

# Axios - HTTP 클라이언트
npm install axios

# UUID - 고유 ID 생성
npm install uuid
```

### UI 라이브러리

```bash
# Material-UI
npm install @mui/material @emotion/react @emotion/styled

# Ant Design
npm install antd

# Chakra UI
npm install @chakra-ui/react @emotion/react @emotion/styled framer-motion
```

### 상태 관리

```bash
# Redux Toolkit
npm install @reduxjs/toolkit react-redux

# Zustand
npm install zustand

# Jotai
npm install jotai
```

### 라우팅

```bash
# React Router
npm install react-router-dom
```

### 폼 관리

```bash
# React Hook Form
npm install react-hook-form

# Formik
npm install formik
```

## TypeScript 설정

```bash
# TypeScript 설치
npm install --save-dev typescript @types/react @types/react-dom

# tsconfig.json 생성
npx tsc --init
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

## 프로젝트 구조

```
my-app/
├── public/
│   ├── index.html
│   └── favicon.ico
├── src/
│   ├── assets/           # 이미지, 폰트 등
│   ├── components/       # 재사용 컴포넌트
│   │   ├── Button/
│   │   │   ├── Button.jsx
│   │   │   ├── Button.test.jsx
│   │   │   └── Button.module.css
│   │   └── index.js
│   ├── pages/            # 페이지 컴포넌트
│   │   ├── Home/
│   │   ├── About/
│   │   └── index.js
│   ├── hooks/            # 커스텀 훅
│   │   ├── useAuth.js
│   │   └── useFetch.js
│   ├── contexts/         # Context
│   │   └── AuthContext.jsx
│   ├── utils/            # 유틸리티 함수
│   │   └── helpers.js
│   ├── services/         # API 서비스
│   │   └── api.js
│   ├── styles/           # 전역 스타일
│   │   └── global.css
│   ├── App.jsx
│   └── main.jsx
├── .gitignore
├── .eslintrc.json
├── .prettierrc
├── package.json
├── vite.config.js
└── README.md
```

## 핵심 요약

### 1. 필수 도구
- Node.js & npm
- Vite/Webpack (빌드)
- ESLint (린터)
- Prettier (포매터)
- Git (버전 관리)

### 2. VS Code 확장
- ESLint
- Prettier
- React 스니펫
- GitLens

### 3. 개발 워크플로우
```bash
npm init -y
npm install react react-dom
npm install --save-dev vite eslint prettier
npm run dev
```

### 4. 코드 품질
- ESLint로 에러 체크
- Prettier로 포맷팅
- Git으로 버전 관리
- 테스트 작성

## 다음 단계
부록 B에서 상태 관리에 대해 학습합니다.