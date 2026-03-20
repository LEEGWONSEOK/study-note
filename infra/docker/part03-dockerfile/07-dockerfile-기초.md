# Chapter 7. Dockerfile 기초

## 7.1 Dockerfile이란?

Dockerfile은 Docker 이미지를 자동으로 빌드하기 위한 **텍스트 형식의 설정 파일**입니다.

> **핵심**: Dockerfile은 이미지를 만드는 **레시피(Recipe)** 입니다.

**비유**: 셰프의 레시피처럼:
- 재료 준비 (베이스 이미지)
- 조리 과정 (패키지 설치, 파일 복사)
- 최종 상태 (실행 명령어 설정)

```dockerfile
# 기본 구조
FROM ubuntu:22.04          # 베이스 이미지 (재료)
RUN apt-get install -y vim  # 명령어 실행 (조리 과정)
COPY app.sh /app/           # 파일 복사
CMD ["/app/app.sh"]         # 기본 실행 명령어
```

---

## 7.2 Dockerfile 기본 명령어

### FROM - 베이스 이미지

```dockerfile
# 필수: 항상 첫 번째 명령어
FROM ubuntu:22.04
FROM node:18-alpine
FROM openjdk:17-jdk-slim

# 별칭 부여 (멀티스테이지 빌드에서 사용)
FROM maven:3.9-openjdk-17 AS builder

# 아무것도 없는 빈 이미지 (Go 바이너리 등에 사용)
FROM scratch
```

### RUN - 명령어 실행

빌드 시 실행됩니다 (이미지 빌드 단계).

```dockerfile
# Shell form
RUN apt-get update && apt-get install -y curl

# Exec form (권장 - shell injection 방지)
RUN ["apt-get", "install", "-y", "curl"]

# 여러 명령어 합치기 (레이어 최소화)
RUN apt-get update \
    && apt-get install -y \
        curl \
        git \
        vim \
    && rm -rf /var/lib/apt/lists/*
```

### COPY - 파일/디렉토리 복사

```dockerfile
# 기본 사용법: COPY <소스> <대상>
COPY app.jar /app/app.jar
COPY . /app/

# 특정 파일만 복사
COPY package*.json ./
COPY src/ ./src/

# 권한 설정
COPY --chown=node:node . /app/

# 멀티스테이지에서 이전 스테이지 파일 복사
COPY --from=builder /app/target/app.jar /app/
```

### ADD - COPY의 확장 버전

```dockerfile
# URL에서 파일 다운로드 가능
ADD https://example.com/file.tar.gz /tmp/

# tar.gz 자동 압축 해제
ADD app.tar.gz /app/

# 일반 파일 복사 (COPY와 동일)
ADD app.jar /app/app.jar
```

> **권장**: 단순 파일 복사는 `COPY` 사용. `ADD`는 URL 다운로드나 tar 압축 해제가 필요할 때만 사용.

### WORKDIR - 작업 디렉토리 설정

```dockerfile
# 이후 명령어의 기준 디렉토리 설정
WORKDIR /app

# 없으면 자동 생성됨
WORKDIR /usr/src/app

# 여러 번 사용 가능
WORKDIR /app
WORKDIR src
# 현재 위치: /app/src
```

### CMD - 기본 실행 명령어

```dockerfile
# Exec form (권장)
CMD ["node", "server.js"]
CMD ["java", "-jar", "/app/app.jar"]
CMD ["nginx", "-g", "daemon off;"]

# Shell form
CMD node server.js

# 파라미터 제공 (ENTRYPOINT와 함께)
CMD ["--port", "8080"]
```

> **중요**: CMD는 `docker run` 시 덮어쓸 수 있습니다.
> ```bash
> docker run myapp node other.js  # CMD 무시
> ```

### ENTRYPOINT - 항상 실행되는 명령어

```dockerfile
# 항상 실행되는 메인 프로세스
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

# CMD와 함께 사용 (CMD가 기본 인수)
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

### EXPOSE - 포트 문서화

```dockerfile
# 컨테이너가 사용하는 포트 명시 (문서화 목적)
EXPOSE 80
EXPOSE 8080
EXPOSE 8080/tcp
EXPOSE 53/udp

# 실제 포트 바인딩은 docker run -p로 설정
```

> **주의**: EXPOSE는 실제로 포트를 열지 않습니다. 문서화 목적입니다.

### ENV - 환경변수

```dockerfile
# 빌드 시 + 런타임 모두 사용 가능
ENV NODE_ENV=production
ENV PORT=8080

# 여러 개 한번에
ENV NODE_ENV=production \
    PORT=8080 \
    APP_NAME=myapp

# 컨테이너 실행 시 -e 옵션으로 재정의 가능
```

### ARG - 빌드 인수

```dockerfile
# 빌드 시에만 사용 가능 (런타임에서는 사용 불가)
ARG VERSION=1.0
ARG BUILD_DATE

# 사용
RUN echo "Building version: ${VERSION}"

# 빌드 시 전달
# docker build --build-arg VERSION=2.0 .
```

### LABEL - 메타데이터

```dockerfile
LABEL maintainer="홍길동 <hong@example.com>"
LABEL version="1.0"
LABEL description="My application"

# 여러 레이블
LABEL \
    maintainer="hong@example.com" \
    version="1.0" \
    org.opencontainers.image.title="My App" \
    org.opencontainers.image.source="https://github.com/org/repo"
```

---

## 7.3 docker build 명령어

```bash
# 기본 빌드 (현재 디렉토리의 Dockerfile)
docker build .

# 태그 지정
docker build -t myapp:1.0 .

# 여러 태그
docker build -t myapp:1.0 -t myapp:latest .

# 다른 Dockerfile 지정
docker build -f Dockerfile.prod -t myapp:prod .

# 빌드 컨텍스트 지정
docker build -t myapp ./myapp-directory

# 빌드 인수 전달
docker build --build-arg VERSION=2.0 -t myapp:2.0 .

# 캐시 사용 안 함
docker build --no-cache -t myapp .

# 플랫폼 지정 (Apple Silicon)
docker build --platform linux/amd64 -t myapp .

# 빌드 로그 상세 출력
docker build --progress=plain -t myapp .
```

---

## 7.4 .dockerignore 파일

빌드 컨텍스트에서 제외할 파일/디렉토리를 지정합니다.

```
# .dockerignore

# 버전 관리
.git
.gitignore
.gitattributes

# 빌드 결과물 (이미 빌드된 파일)
node_modules
target/
build/
dist/
*.class

# 환경 설정 (보안)
.env
.env.*
*.secret

# 로그 파일
*.log
logs/

# IDE 설정
.idea/
.vscode/
*.iml

# 문서 (이미지에 불필요)
*.md
docs/

# OS 파일
.DS_Store
Thumbs.db
```

**효과**:
```bash
# .dockerignore 없을 때
Sending build context to Docker daemon  450.2MB

# .dockerignore 있을 때
Sending build context to Docker daemon  2.048MB
```

---

## 7.5 첫 Dockerfile: Node.js 앱 예제

### 프로젝트 구조

```
myapp/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
└── src/
    └── server.js
```

### server.js

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Docker!', port: PORT });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### package.json

```json
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  },
  "scripts": {
    "start": "node src/server.js"
  }
}
```

### Dockerfile

```dockerfile
# 베이스 이미지
FROM node:18-alpine

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일만 먼저 복사 (캐시 최적화)
COPY package*.json ./

# 의존성 설치
RUN npm ci --only=production

# 소스 코드 복사
COPY src/ ./src/

# 포트 노출 (문서화)
EXPOSE 3000

# 비root 사용자로 실행 (보안)
USER node

# 실행 명령어
CMD ["node", "src/server.js"]
```

### 빌드 및 실행

```bash
# 빌드
docker build -t myapp:1.0 .

# 실행
docker run -d -p 3000:3000 --name myapp myapp:1.0

# 확인
curl http://localhost:3000
# {"message":"Hello from Docker!","port":"3000"}

# 로그 확인
docker logs myapp
```

---

## 7.6 Java Spring Boot 앱 Dockerfile 예제

### 단순 버전

```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### 레이어 최적화 버전 (Spring Boot 2.3+)

```dockerfile
FROM openjdk:17-jdk-slim AS builder
WORKDIR /app
COPY target/app.jar app.jar

# Spring Boot LayerTools로 레이어 분리
RUN java -Djarmode=layertools -jar app.jar extract

FROM openjdk:17-jre-slim
WORKDIR /app

# 변경 빈도 낮은 레이어 먼저 복사
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
# 변경 빈도 높은 레이어 나중에
COPY --from=builder /app/application/ ./

EXPOSE 8080

# 비root 사용자
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

### Maven 멀티스테이지 빌드 포함

```dockerfile
# 빌드 스테이지
FROM maven:3.9-openjdk-17 AS build
WORKDIR /app

# 의존성 캐시 최적화
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 소스 빌드
COPY src ./src
RUN mvn clean package -DskipTests

# 실행 스테이지
FROM openjdk:17-jre-slim
WORKDIR /app

COPY --from=build /app/target/app.jar app.jar

EXPOSE 8080
CMD ["java", "-Xmx512m", "-jar", "app.jar"]
```

---

## 핵심 요약

| 명령어 | 용도 |
|--------|------|
| `FROM` | 베이스 이미지 지정 (필수) |
| `RUN` | 빌드 시 명령어 실행 |
| `COPY` | 파일/디렉토리 복사 |
| `ADD` | COPY + URL/tar 지원 |
| `WORKDIR` | 작업 디렉토리 설정 |
| `CMD` | 기본 실행 명령어 (덮어쓰기 가능) |
| `ENTRYPOINT` | 메인 프로세스 (항상 실행) |
| `EXPOSE` | 포트 문서화 |
| `ENV` | 환경변수 (빌드+런타임) |
| `ARG` | 빌드 인수 (빌드 시에만) |
| `LABEL` | 메타데이터 |

---

## 다음 단계

[Chapter 8. Dockerfile 고급 명령어 →](08-dockerfile-고급-명령어.md)

---

## 실습 과제

1. Node.js Express 앱 Dockerfile 작성하고 빌드하기
2. `.dockerignore` 파일 작성 후 빌드 컨텍스트 크기 비교하기
3. Spring Boot 앱 Dockerfile 작성하기
4. `ENV`로 환경변수 설정하고 `docker run -e`로 재정의해보기
5. `WORKDIR` 없이 빌드한 이미지와 비교해보기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자