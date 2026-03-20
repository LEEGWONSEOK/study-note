# Chapter 22. CI/CD 파이프라인

## 22.1 CI/CD에서 Docker의 역할

```
개발자 코드 Push
        │
        ▼
┌───────────────────┐
│   CI (빌드/테스트) │
│                   │
│ 1. 코드 체크아웃  │
│ 2. 테스트 실행    │
│ 3. Docker 빌드    │
│ 4. 이미지 스캔    │
│ 5. 레지스트리 Push │
└───────────────────┘
        │
        ▼
┌───────────────────┐
│   CD (배포)       │
│                   │
│ 1. 이미지 Pull    │
│ 2. 컨테이너 교체  │
│ 3. 헬스체크       │
│ 4. 롤백 (실패 시) │
└───────────────────┘
        │
        ▼
운영 서버
```

**Docker가 CI/CD에서 중요한 이유**:
- 빌드 환경의 일관성 보장
- "한번 빌드, 어디서나 배포"
- 롤백이 이미지 태그 변경으로 간단

---

## 22.2 GitHub Actions + Docker 기본

### 기본 워크플로우

```yaml
# .github/workflows/docker.yml
name: Docker CI/CD

on:
  push:
    branches:
      - main
      - develop
    tags:
      - 'v*.*.*'
  pull_request:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      # 코드 체크아웃
      - name: Checkout
        uses: actions/checkout@v4

      # Docker BuildKit 설정
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Docker Hub 로그인
      - name: Login to Docker Hub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # 메타데이터 (태그, 레이블) 자동 생성
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      # 빌드 및 Push
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha        # GitHub Actions 캐시
          cache-to: type=gha,mode=max
```

---

## 22.3 Dockerfile 최적화 (빌드 캐시)

### GitHub Actions 캐시 활용

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./

# BuildKit 캐시 마운트 (npm 캐시 재사용)
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

FROM node:18-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

### Maven 빌드 캐시

```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-openjdk-17 AS build
WORKDIR /app

COPY pom.xml .
# Maven 로컬 저장소 캐시 마운트
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline -B

COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn clean package -DskipTests
```

---

## 22.4 이미지 태그 전략 (git sha, 브랜치명)

### 태그 패턴

```yaml
# GitHub Actions에서 자동 태그 생성
- name: Extract metadata
  uses: docker/metadata-action@v5
  with:
    images: username/myapp
    tags: |
      # main 브랜치: latest
      type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      # 브랜치명: develop, feature-login 등
      type=ref,event=branch

      # Git SHA: sha-abc1234
      type=sha,format=short

      # 릴리즈 태그: v1.2.3, v1.2, v1
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=semver,pattern={{major}}
```

**생성되는 태그 예시**:
```
main 브랜치 push:
  username/myapp:latest
  username/myapp:main
  username/myapp:sha-abc1234

v1.2.3 태그 생성:
  username/myapp:1.2.3
  username/myapp:1.2
  username/myapp:1
  username/myapp:latest
```

---

## 22.5 레지스트리 push 자동화

### ECR (AWS) 자동화

```yaml
name: Deploy to AWS

on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: AWS 인증
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: ap-northeast-2

      - name: ECR 로그인
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: 빌드 및 ECR Push
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.ref_name }}
        run: |
          docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
          docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
```

---

## 22.6 서버 배포 자동화

### SSH를 통한 서버 배포

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push    # 빌드 완료 후 배포

    steps:
      - name: 서버 배포
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            # 최신 이미지 Pull
            docker pull ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest

            # 무중단 배포
            cd /opt/myapp
            docker compose pull app
            docker compose up -d --no-deps app

            # 헬스체크
            echo "헬스체크 대기..."
            sleep 30
            curl -f http://localhost:8080/actuator/health || exit 1

            # 이전 이미지 정리
            docker image prune -f
```

### Docker Swarm 배포

```yaml
- name: Swarm 배포
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: ${{ secrets.SERVER_USER }}
    key: ${{ secrets.SERVER_SSH_KEY }}
    script: |
      docker service update \
        --image ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }} \
        --update-parallelism 1 \
        --update-delay 10s \
        --update-failure-action rollback \
        myapp_app
```

---

## 22.7 실전 GitHub Actions 예제

### 완전한 CI/CD 파이프라인

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/myapp

jobs:
  # ============================================================
  # Job 1: 테스트
  # ============================================================
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: testpassword
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s

    steps:
      - uses: actions/checkout@v4

      - name: Java 설정
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: 테스트 실행
        run: mvn test -Dspring.profiles.active=test

      - name: 테스트 결과 업로드
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/

  # ============================================================
  # Job 2: 빌드 및 이미지 Push
  # ============================================================
  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name != 'pull_request'

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tags: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Docker Buildx 설정
        uses: docker/setup-buildx-action@v3

      - name: Docker Hub 로그인
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: 이미지 메타데이터 추출
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=ref,event=branch
            type=sha,prefix={{branch}}-,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: 빌드 및 Push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}

  # ============================================================
  # Job 3: 보안 스캔
  # ============================================================
  security-scan:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Trivy 보안 스캔
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          severity: 'CRITICAL'
          exit-code: '1'

  # ============================================================
  # Job 4: 스테이징 배포 (develop 브랜치)
  # ============================================================
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - name: 스테이징 서버 배포
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/myapp
            export APP_VERSION=${{ github.sha }}
            docker compose pull app
            docker compose up -d --no-deps app

  # ============================================================
  # Job 5: 운영 배포 (태그)
  # ============================================================
  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production

    steps:
      - name: 운영 서버 배포
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/myapp
            export APP_VERSION=${{ github.ref_name }}
            docker compose pull app
            docker compose up -d --no-deps app
            sleep 30
            curl -f http://localhost:8080/actuator/health
```

---

## 핵심 요약

1. **CI**: 테스트 → 빌드 → 이미지 스캔 → Push 순서
2. **BuildKit 캐시**: `cache-from: type=gha`로 빌드 속도 향상
3. **태그 전략**: Semver + Git SHA로 추적 용이하게
4. **환경 분리**: develop → staging, tag → production
5. **보안 스캔**: Trivy로 CRITICAL 취약점 배포 차단
6. **롤백**: 이전 이미지 태그로 간단히 롤백

---

## 다음 단계

[Chapter 23. Kubernetes 입문 →](23-kubernetes-입문.md)

---

## 실습 과제

1. GitHub Actions 워크플로우로 자동 빌드 구성하기
2. BuildKit 캐시로 빌드 시간 단축 확인하기
3. Trivy 보안 스캔을 CI 파이프라인에 통합하기
4. Docker Hub에 자동 Push 워크플로우 구성하기
5. SSH를 통한 서버 자동 배포 스크립트 작성하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자