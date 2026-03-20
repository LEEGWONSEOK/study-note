# Chapter 16. Private Registry

## 16.1 Private Registry가 필요한 이유

| 이유 | 설명 |
|------|------|
| **보안** | 소스 코드, 비즈니스 로직이 담긴 이미지를 공개하지 않음 |
| **속도** | 내부 네트워크에서 빠른 pull/push |
| **비용** | Docker Hub 유료 플랜 없이 무제한 비공개 저장소 |
| **컴플라이언스** | 법적 규제로 외부 서비스 사용 불가한 경우 |
| **오프라인 환경** | 인터넷 없는 폐쇄망 환경 |

---

## 16.2 Docker Registry 직접 구축

### 간단한 Registry 실행

```bash
# Docker 공식 Registry 이미지로 실행
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  --restart unless-stopped \
  registry:2

# 확인
curl http://localhost:5000/v2/
# {}
```

### 이미지 push/pull (로컬 레지스트리)

```bash
# 이미지에 로컬 레지스트리 태그 추가
docker tag nginx:latest localhost:5000/nginx:latest

# push
docker push localhost:5000/nginx:latest

# 다른 서버에서 pull
docker pull 192.168.1.100:5000/nginx:latest
```

### HTTPS 설정 (운영 환경)

```bash
# 인증서 생성 (자체 서명)
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key \
  -x509 -days 365 \
  -out certs/domain.crt \
  -subj "/CN=registry.company.com"

# HTTPS Registry 실행
docker run -d \
  --name registry \
  -p 443:443 \
  -v $(pwd)/certs:/certs \
  -v registry-data:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### 인증 설정

```bash
# htpasswd로 사용자 생성
mkdir auth
docker run --rm httpd:2 htpasswd -Bbn admin password > auth/htpasswd

# 인증 포함 Registry 실행
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v $(pwd)/auth:/auth \
  -v registry-data:/var/lib/registry \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2

# 로그인
docker login localhost:5000
```

### Docker Compose로 Registry 구축

```yaml
# registry/docker-compose.yml
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Private Registry"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - ./auth:/auth
      - registry-data:/data
    restart: unless-stopped

volumes:
  registry-data:
```

---

## 16.3 AWS ECR (Elastic Container Registry)

### 개요

AWS가 제공하는 완전 관리형 Private Registry입니다.

**특징**:
- IAM으로 접근 제어
- 자동 이미지 스캔 (보안 취약점)
- 이미지 수명 주기 정책
- ECS, EKS와 네이티브 통합

### 설정

```bash
# AWS CLI 설치 및 설정
aws configure
# AWS Access Key ID: ...
# AWS Secret Access Key: ...
# Default region: ap-northeast-2

# ECR 저장소 생성
aws ecr create-repository \
  --repository-name myapp \
  --region ap-northeast-2

# 출력:
# "repositoryUri": "123456789.dkr.ecr.ap-northeast-2.amazonaws.com/myapp"

# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789.dkr.ecr.ap-northeast-2.amazonaws.com
```

### 이미지 push/pull

```bash
# ECR 태그 형식
ECR_URI="123456789.dkr.ecr.ap-northeast-2.amazonaws.com"
REPO_NAME="myapp"

# 태그
docker tag myapp:latest ${ECR_URI}/${REPO_NAME}:latest
docker tag myapp:1.0.0 ${ECR_URI}/${REPO_NAME}:1.0.0

# Push
docker push ${ECR_URI}/${REPO_NAME}:latest
docker push ${ECR_URI}/${REPO_NAME}:1.0.0

# Pull
docker pull ${ECR_URI}/${REPO_NAME}:latest
```

### ECR 수명 주기 정책

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "최신 10개만 유지",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "태그 없는 이미지 7일 후 삭제",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

## 16.4 GCR (Google Artifact Registry)

```bash
# Google Cloud SDK 설치 및 인증
gcloud auth login
gcloud config set project my-project-id

# Docker 인증 설정
gcloud auth configure-docker asia-northeast3-docker.pkg.dev

# 저장소 생성
gcloud artifacts repositories create myapp \
  --repository-format=docker \
  --location=asia-northeast3

# 태그 및 Push
docker tag myapp:latest \
  asia-northeast3-docker.pkg.dev/my-project/myapp:latest

docker push asia-northeast3-docker.pkg.dev/my-project/myapp:latest
```

---

## 16.5 Harbor 소개

Harbor는 **오픈소스 엔터프라이즈 레지스트리**입니다.

**특징**:
- 웹 UI 제공
- 역할 기반 접근 제어 (RBAC)
- 이미지 취약점 스캔 (Trivy 통합)
- 이미지 복제 (Replication)
- Helm Chart 저장소
- 프로젝트 단위 관리

### Harbor 설치 (Docker Compose)

```bash
# Harbor 설치 파일 다운로드
wget https://github.com/goharbor/harbor/releases/download/v2.9.0/harbor-online-installer-v2.9.0.tgz
tar xvf harbor-online-installer-v2.9.0.tgz
cd harbor

# 설정 파일 수정
cp harbor.yml.tmpl harbor.yml
vi harbor.yml
# hostname: registry.company.com
# https:
#   certificate: /path/to/cert
#   private_key: /path/to/key

# 설치
sudo ./install.sh
```

### Harbor 사용

```bash
# Harbor 로그인
docker login registry.company.com

# 이미지 push
docker tag myapp:latest registry.company.com/project/myapp:latest
docker push registry.company.com/project/myapp:latest
```

---

## 16.6 인증 설정

### Docker 클라이언트 인증 구성

```bash
# ~/.docker/config.json에 자격증명 저장
docker login registry.company.com
# Username: admin
# Password: ****

cat ~/.docker/config.json
# {
#   "auths": {
#     "registry.company.com": {
#       "auth": "base64encodedcredentials"
#     }
#   }
# }
```

### Kubernetes Secret으로 관리 (K8s 환경)

```bash
# Registry 인증 시크릿 생성
kubectl create secret docker-registry regcred \
  --docker-server=registry.company.com \
  --docker-username=admin \
  --docker-password=password \
  --docker-email=admin@company.com
```

---

## 16.7 Compose와 Private Registry 연동

```yaml
services:
  app:
    # Private Registry 이미지 사용
    image: registry.company.com/myapp:1.0.0
    # 또는 ECR
    # image: 123456789.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:latest

  db:
    image: registry.company.com/mysql-custom:8.0

  nginx:
    image: registry.company.com/nginx-custom:1.25
```

```bash
# 배포 전 레지스트리 로그인
docker login registry.company.com

# 또는 ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.ap-northeast-2.amazonaws.com

# Compose 실행
docker compose pull    # 이미지 미리 다운로드
docker compose up -d
```

---

## 핵심 요약

1. **필요 이유**: 보안, 속도, 비용, 컴플라이언스
2. **자체 구축**: `registry:2` 이미지로 간단히 구축
3. **AWS ECR**: IAM 통합, ECS/EKS 네이티브 지원
4. **GCR/GAR**: GCP 환경에 최적
5. **Harbor**: 오픈소스, 풍부한 기능, 엔터프라이즈 환경
6. **수명 주기 정책**: 오래된 이미지 자동 정리 필수

---

## 다음 단계

[Chapter 17. Docker 보안 기초 →](../part07-보안/17-docker-보안-기초.md)

---

## 실습 과제

1. 로컬 환경에서 `registry:2`로 Private Registry 구축하기
2. 이미지를 Private Registry에 push하고 pull하기
3. htpasswd로 인증 설정하기
4. AWS ECR 또는 GCR 계정이 있다면 연동 실습하기
5. 수명 주기 정책으로 오래된 이미지 자동 정리 설정하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자