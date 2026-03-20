# Chapter 8. Dockerfile 고급 명령어

## 8.1 CMD vs ENTRYPOINT 차이

가장 많이 헷갈리는 부분입니다.

### 핵심 차이

| 항목 | CMD | ENTRYPOINT |
|------|-----|-----------|
| 역할 | 기본 실행 명령어 | 항상 실행되는 명령어 |
| 덮어쓰기 | `docker run` 시 가능 | `--entrypoint` 플래그 필요 |
| 함께 사용 시 | 기본 인수 제공 | 메인 명령어 |
| 사용 시기 | 유연한 실행이 필요할 때 | 특정 프로세스 고정 시 |

### CMD 단독 사용

```dockerfile
FROM ubuntu
CMD ["echo", "Hello World"]
```

```bash
# 기본 실행
docker run myimage
# Hello World

# CMD 덮어쓰기 (다른 명령어 실행)
docker run myimage echo "Goodbye"
# Goodbye

docker run myimage bash
# bash 쉘 접속
```

### ENTRYPOINT 단독 사용

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```

```bash
# 기본 실행 (인수 없음)
docker run myimage
# (빈 줄)

# 인수 추가
docker run myimage "Hello World"
# Hello World

# ENTRYPOINT 자체를 바꾸려면
docker run --entrypoint bash myimage
```

### CMD + ENTRYPOINT 함께 사용 (권장 패턴)

```dockerfile
FROM nginx
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

```bash
# 기본 실행: nginx -g "daemon off;"
docker run myimage

# CMD 부분만 바꾸기
docker run myimage -g "daemon off;" -c /etc/nginx/nginx.conf

# ENTRYPOINT 자체를 바꾸기
docker run --entrypoint bash myimage
```

### 실전 활용 예시

```dockerfile
# 스크립트를 ENTRYPOINT로 (초기화 + 실행)
FROM node:18-alpine
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

```bash
#!/bin/sh
# docker-entrypoint.sh
set -e

# 환경변수로 초기화 작업
echo "Starting with NODE_ENV=${NODE_ENV}"

# DB 연결 대기
while ! nc -z $DB_HOST $DB_PORT; do
  echo "Waiting for database..."
  sleep 1
done

# CMD 실행 ("$@"는 CMD로 전달된 인수)
exec "$@"
```

---

## 8.2 Shell form vs Exec form

### Shell form

```dockerfile
# Shell form: /bin/sh -c로 실행
RUN apt-get install -y curl
CMD echo "Hello"
ENTRYPOINT java -jar app.jar
```

**특징**:
- `/bin/sh -c "명령어"` 형태로 실행
- Shell 변수 치환(`$VAR`) 가능
- PID 1이 sh 프로세스 → SIGTERM 전달 문제

### Exec form

```dockerfile
# Exec form: JSON 배열 형태
RUN ["apt-get", "install", "-y", "curl"]
CMD ["echo", "Hello"]
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**특징**:
- 직접 실행 (shell 없이)
- PID 1이 실제 프로세스 → SIGTERM 올바르게 처리
- Shell 변수 치환 불가

### 권장: ENTRYPOINT/CMD는 Exec form

```dockerfile
# 나쁜 예 - Shell form (SIGTERM 처리 안됨)
ENTRYPOINT java -jar app.jar

# 좋은 예 - Exec form (SIGTERM 올바르게 처리)
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 환경변수가 필요할 때

```dockerfile
# 방법 1: sh -c 사용
CMD ["/bin/sh", "-c", "java -jar app.jar --port=${PORT}"]

# 방법 2: 스크립트 사용 (권장)
COPY start.sh /start.sh
RUN chmod +x /start.sh
ENTRYPOINT ["/start.sh"]
```

---

## 8.3 VOLUME 명령어

컨테이너에서 데이터를 영구 저장할 마운트 포인트를 선언합니다.

```dockerfile
# 단일 볼륨
VOLUME /data

# 여러 볼륨
VOLUME ["/data", "/logs", "/tmp"]
```

```bash
# VOLUME 선언된 경로는 자동으로 익명 볼륨으로 마운트
docker run -d mysql:8.0
# /var/lib/mysql이 자동으로 익명 볼륨에 연결됨

# 명명 볼륨으로 재정의
docker run -d -v mydata:/var/lib/mysql mysql:8.0
```

> **주의**: Dockerfile의 VOLUME 이후에 해당 경로를 수정해도 이미지에 반영되지 않습니다.

---

## 8.4 HEALTHCHECK 명령어

컨테이너의 상태를 주기적으로 확인합니다.

```dockerfile
# 기본 사용법
HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1

# 옵션 설정
HEALTHCHECK \
  --interval=30s \    # 체크 간격 (기본 30s)
  --timeout=10s \     # 타임아웃 (기본 30s)
  --start-period=60s \# 시작 대기 시간 (기본 0s)
  --retries=3 \       # 실패 허용 횟수 (기본 3)
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

```bash
# 상태 확인
docker ps
# STATUS: Up 2 minutes (healthy)
# STATUS: Up 2 minutes (unhealthy)
# STATUS: Up 2 minutes (health: starting)

# 상세 확인
docker inspect --format '{{json .State.Health}}' 컨테이너명
```

### Spring Boot Actuator 헬스체크

```dockerfile
FROM openjdk:17-jre-slim
COPY app.jar /app/app.jar

HEALTHCHECK \
  --interval=30s \
  --timeout=10s \
  --start-period=60s \
  --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080
CMD ["java", "-jar", "/app/app.jar"]
```

### 헬스체크 비활성화

```dockerfile
# 부모 이미지의 HEALTHCHECK 비활성화
HEALTHCHECK NONE
```

---

## 8.5 USER 명령어 (보안)

보안을 위해 root 이외의 사용자로 실행합니다.

```dockerfile
# 사용자 생성 및 전환
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser

# UID/GID 직접 지정
USER 1001:1001

# 특정 단계부터만 사용자 전환
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install          # root로 설치 (전역 패키지 등)
COPY . .
USER node               # 실행은 node 사용자로
CMD ["node", "server.js"]
```

### Node.js 앱 보안 설정 예시

```dockerfile
FROM node:18-alpine

# node 그룹/사용자는 기본 포함됨
WORKDIR /app

# root로 파일 복사 및 권한 설정
COPY --chown=node:node package*.json ./
RUN npm ci --only=production

COPY --chown=node:node . .

# node 사용자로 실행
USER node

EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 8.6 ONBUILD 명령어

이 이미지를 베이스로 사용하는 Dockerfile에서 실행될 명령어를 지정합니다.

```dockerfile
# 베이스 이미지 Dockerfile
FROM node:18-alpine
WORKDIR /app
ONBUILD COPY package*.json ./
ONBUILD RUN npm install
ONBUILD COPY . .
```

```dockerfile
# 파생 이미지 Dockerfile
FROM mybase-node:latest
# ONBUILD 명령어들이 자동으로 실행됨!
# COPY package*.json ./ 실행
# RUN npm install 실행
# COPY . . 실행
EXPOSE 3000
CMD ["node", "server.js"]
```

> **사용 시기**: 공통 빌드 패턴을 가진 베이스 이미지 만들 때 유용하지만, 복잡해질 수 있으므로 신중하게 사용.

---

## 8.7 ARG vs ENV 차이

| 항목 | ARG | ENV |
|------|-----|-----|
| 사용 시점 | 빌드 시에만 | 빌드 + 런타임 |
| 전달 방법 | `--build-arg` | `docker run -e` |
| 이미지에 남음 | 아니오 | 예 (주의!) |
| 기본값 | 지원 | 지원 |
| 보안 | 히스토리에서 보임 | 컨테이너 환경에 남음 |

### ARG 사용 예시

```dockerfile
# 기본값 있는 ARG
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

# 여러 ARG
ARG APP_VERSION=1.0.0
ARG BUILD_DATE
ARG GIT_COMMIT

LABEL version=${APP_VERSION}
LABEL build-date=${BUILD_DATE}
LABEL git-commit=${GIT_COMMIT}
```

```bash
# 빌드 시 전달
docker build \
  --build-arg NODE_VERSION=20 \
  --build-arg APP_VERSION=2.0.0 \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  -t myapp:2.0.0 .
```

### FROM 이전에 ARG 사용

```dockerfile
# FROM 이전 ARG는 특별 처리됨
ARG BASE_VERSION=22.04
FROM ubuntu:${BASE_VERSION}

# FROM 이후 다시 선언해야 사용 가능
ARG BASE_VERSION
RUN echo "Base: ${BASE_VERSION}"
```

---

## 8.8 빌드 인수(build args) 활용

### 환경별 빌드

```dockerfile
FROM node:18-alpine
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

RUN if [ "$NODE_ENV" = "development" ]; then \
    npm install; \
    else \
    npm ci --only=production; \
    fi
```

```bash
# 개발 빌드
docker build --build-arg NODE_ENV=development -t myapp:dev .

# 운영 빌드
docker build -t myapp:prod .
```

### 버전 관리

```dockerfile
ARG APP_VERSION=latest
ARG GIT_COMMIT=unknown

LABEL app.version=$APP_VERSION
LABEL app.git-commit=$GIT_COMMIT

ENV APP_VERSION=$APP_VERSION
```

```bash
# CI/CD에서 사용
docker build \
  --build-arg APP_VERSION=${CI_COMMIT_TAG} \
  --build-arg GIT_COMMIT=${CI_COMMIT_SHA} \
  -t myapp:${CI_COMMIT_TAG} .
```

---

## 8.9 조건부 설치 패턴

### 환경별 패키지 설치

```dockerfile
FROM ubuntu:22.04
ARG INSTALL_DEV_TOOLS=false

RUN apt-get update && apt-get install -y \
    curl \
    && if [ "$INSTALL_DEV_TOOLS" = "true" ]; then \
        apt-get install -y \
        vim \
        git \
        htop; \
    fi \
    && rm -rf /var/lib/apt/lists/*
```

### 플랫폼별 처리

```dockerfile
FROM --platform=$BUILDPLATFORM node:18-alpine AS builder
ARG TARGETOS
ARG TARGETARCH

RUN echo "Building for ${TARGETOS}/${TARGETARCH}"
```

### 캐시 버스팅 패턴

```dockerfile
# 특정 레이어만 캐시 무효화
ARG CACHEBUST=1
RUN echo "${CACHEBUST}" && apt-get update
```

```bash
# 특정 레이어부터 새로 빌드
docker build --build-arg CACHEBUST=$(date +%s) .
```

---

## 핵심 요약

1. **CMD**: 기본 명령어, `docker run` 시 덮어쓰기 가능
2. **ENTRYPOINT**: 항상 실행, `--entrypoint`로만 변경 가능
3. **Exec form 권장**: SIGTERM 올바른 처리
4. **HEALTHCHECK**: 컨테이너 상태 모니터링
5. **USER**: root 아닌 사용자 사용으로 보안 향상
6. **ARG vs ENV**: ARG는 빌드 시에만, ENV는 런타임까지
7. **VOLUME**: 데이터 영속화를 위한 마운트 포인트 선언

---

## 다음 단계

[Chapter 9. 멀티스테이지 빌드 →](09-멀티스테이지-빌드.md)

---

## 실습 과제

1. CMD와 ENTRYPOINT를 함께 사용하는 Dockerfile 작성하기
2. HEALTHCHECK가 있는 Spring Boot 앱 Dockerfile 작성하기
3. `USER node`를 추가하여 비root 실행 확인하기
4. ARG를 사용하여 개발/운영 환경별 빌드 설정하기
5. `docker run --entrypoint sh myapp`으로 ENTRYPOINT 재정의 해보기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자