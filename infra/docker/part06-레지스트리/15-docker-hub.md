# Chapter 15. Docker Hub

## 15.1 Docker Hub란?

Docker Hub는 Docker Inc.가 운영하는 **공식 컨테이너 이미지 레지스트리**입니다.

- **URL**: [https://hub.docker.com](https://hub.docker.com)
- 세계 최대의 컨테이너 이미지 저장소
- 공개(Public) 및 비공개(Private) 저장소 지원
- 자동 빌드(GitHub 연동) 지원

> **비유**: Docker Hub는 앱의 **GitHub** 와 같습니다.
> 소스 코드 대신 컨테이너 이미지를 저장하고 공유합니다.

---

## 15.2 계정 생성 및 로그인

### 계정 생성

1. [https://hub.docker.com](https://hub.docker.com) 접속
2. "Sign Up" 클릭
3. Docker ID, 이메일, 비밀번호 입력

### CLI 로그인

```bash
# Docker Hub 로그인
docker login

# 사용자명/비밀번호 입력
# Username: your-username
# Password: your-password

# 특정 레지스트리 로그인
docker login registry.example.com

# 자격증명 저장 확인
cat ~/.docker/config.json
```

### 보안을 위한 액세스 토큰 사용 (권장)

```bash
# Docker Hub → Account Settings → Security → New Access Token
# 생성한 토큰으로 로그인

docker login
# Username: your-username
# Password: [액세스 토큰 입력]  ← 비밀번호 대신 토큰 사용
```

### 로그아웃

```bash
docker logout
docker logout registry.example.com
```

---

## 15.3 이미지 push/pull

### 이미지 push

```bash
# 1. 이미지 빌드
docker build -t myapp:1.0 .

# 2. Docker Hub 형식으로 태그 지정
docker tag myapp:1.0 username/myapp:1.0
docker tag myapp:1.0 username/myapp:latest

# 3. push
docker push username/myapp:1.0
docker push username/myapp:latest
```

### push 스크립트 예시

```bash
#!/bin/bash
# push.sh

IMAGE_NAME="username/myapp"
VERSION="$1"  # 버전을 인수로 받음

if [ -z "$VERSION" ]; then
  echo "사용법: ./push.sh <버전>"
  exit 1
fi

# 빌드
docker build -t ${IMAGE_NAME}:${VERSION} .

# 태그 추가
docker tag ${IMAGE_NAME}:${VERSION} ${IMAGE_NAME}:latest

# Push
docker push ${IMAGE_NAME}:${VERSION}
docker push ${IMAGE_NAME}:latest

echo "✓ ${IMAGE_NAME}:${VERSION} 업로드 완료"
```

### 이미지 pull

```bash
# 최신 버전 다운로드
docker pull username/myapp

# 특정 버전
docker pull username/myapp:1.0

# 모든 태그
docker pull --all-tags username/myapp
```

---

## 15.4 자동 빌드 (GitHub 연동)

Docker Hub의 Automated Builds 기능으로 GitHub Push 시 자동 빌드가 가능합니다.

### 설정 방법

1. Docker Hub → 저장소 → "Builds" 탭
2. "Link to GitHub" 클릭
3. GitHub 저장소 선택
4. 빌드 규칙 설정:

```
Branch/Tag     Dockerfile   Docker Tag
main         → Dockerfile → latest
v*           → Dockerfile → {sourceref}
release/*    → Dockerfile → release-{sourceref}
```

### GitHub Actions로 직접 구현 (현재 권장)

Docker Hub의 Automated Build보다 GitHub Actions를 직접 사용하는 것이 더 유연합니다:

```yaml
# .github/workflows/docker-publish.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Docker Hub 로그인
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: 메타데이터 추출 (태그, 레이블)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: username/myapp
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=ref,event=branch

      - name: 빌드 및 Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## 15.5 공개/비공개 저장소

### 공개 저장소 (Public)

- 무료 계정에서 무제한 생성 가능
- 누구나 pull 가능 (인증 불필요)
- 오픈소스 프로젝트에 적합

```bash
# 공개 이미지는 로그인 없이 pull 가능
docker pull username/public-image
```

### 비공개 저장소 (Private)

- 무료 계정: 1개 무료
- 인증된 사용자만 pull 가능
- 기업 및 개인 프로젝트에 적합

```bash
# Private 이미지 pull 시 로그인 필요
docker login
docker pull username/private-image
```

### Organizations (팀 관리)

- 팀 단위 이미지 관리
- 역할 기반 접근 제어 (RBAC)
- 기업 환경에 적합

---

## 15.6 Official 이미지 vs 커뮤니티 이미지

### Official 이미지

```
특징:
✅ Docker Inc. 또는 업스트림 프로젝트가 관리
✅ 보안 취약점 정기 검토
✅ 명확한 문서화
✅ 이름에 네임스페이스 없음 (nginx, mysql, node 등)
✅ [Official Image] 배지 표시
```

```bash
# Official 이미지 목록 검색
docker search --filter is-official=true mysql
```

### 커뮤니티 이미지 사용 시 주의사항

```bash
# 이미지 평판 확인
docker search mysql --format "table {{.Name}}\t{{.StarCount}}\t{{.IsOfficial}}"

# 다운로드 수, 별점, 공식 여부 확인
# NAME                    STARS     OFFICIAL
# mysql                   14000     [OK]      ← 공식 이미지
# bitnami/mysql           800                 ← 커뮤니티 (Bitnami는 신뢰할 수 있음)
# random-user/mysql       2                   ← 주의!
```

---

## 15.7 이미지 태그 전략

### Semantic Versioning (권장)

```
v1.0.0  → 메이저.마이너.패치
```

```bash
# 버전별 태그
docker tag myapp:latest username/myapp:1.0.0
docker tag myapp:latest username/myapp:1.0
docker tag myapp:latest username/myapp:1
docker tag myapp:latest username/myapp:latest

docker push username/myapp:1.0.0
docker push username/myapp:1.0
docker push username/myapp:1
docker push username/myapp:latest
```

### Git 커밋 기반 태그

```bash
# CI/CD에서 Git SHA로 태그
GIT_SHA=$(git rev-parse --short HEAD)
docker tag myapp:latest username/myapp:${GIT_SHA}
docker push username/myapp:${GIT_SHA}
```

### 태그 전략 비교

| 전략 | 예시 | 사용 시기 |
|------|------|----------|
| latest | `myapp:latest` | 개발 환경 (운영 비권장) |
| Semver | `myapp:1.2.3` | 운영 배포 (권장) |
| Git SHA | `myapp:abc1234` | CI/CD 추적 |
| 날짜 | `myapp:20240101` | 아카이브 목적 |
| 브랜치 | `myapp:main` | 브랜치별 테스트 |

---

## 15.8 Docker Hub 요금제

| 플랜 | 가격 | 특징 |
|------|------|------|
| Free | 무료 | 공개 저장소 무제한, 비공개 1개 |
| Pro | $5/월 | 비공개 저장소 무제한, 자동 빌드 |
| Team | $7/인/월 | 팀 관리, 역할 기반 접근 |
| Business | $24/인/월 | SSO, 감사 로그, 더 많은 기능 |

---

## 핵심 요약

1. **Docker Hub**: 공식 Docker 이미지 레지스트리
2. **로그인**: 액세스 토큰 사용 권장 (보안)
3. **push 절차**: 빌드 → 태그 → push
4. **자동 빌드**: GitHub Actions로 구현 권장
5. **공식 이미지**: 신뢰도 높은 이미지, 우선 사용
6. **태그 전략**: 운영에서는 Semver 사용 (`latest` 비권장)

---

## 다음 단계

[Chapter 16. Private Registry →](16-private-registry.md)

---

## 실습 과제

1. Docker Hub 계정 생성 및 액세스 토큰 발급하기
2. 간단한 앱을 빌드하여 Docker Hub에 push하기
3. 다른 컴퓨터(또는 컨테이너)에서 push한 이미지 pull하기
4. 비공개 저장소 생성 및 접근 제어 확인하기
5. GitHub Actions로 자동 빌드 파이프라인 구성하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자