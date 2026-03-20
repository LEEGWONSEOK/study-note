# Chapter 17. Docker 보안 기초

## 17.1 Docker 보안 개요

Docker 보안은 여러 계층에서 관리해야 합니다:

```
┌─────────────────────────────────────┐
│         애플리케이션 보안              │ ← 앱 코드, 의존성
├─────────────────────────────────────┤
│         컨테이너 보안                 │ ← 실행 권한, 리소스 제한
├─────────────────────────────────────┤
│         이미지 보안                   │ ← 취약점, 서명
├─────────────────────────────────────┤
│         Docker 데몬 보안             │ ← API 접근, TLS
├─────────────────────────────────────┤
│         호스트 OS 보안               │ ← 커널, 방화벽
└─────────────────────────────────────┘
```

---

## 17.2 root vs non-root 실행

### 왜 root로 실행하면 안 될까?

```bash
# 기본적으로 컨테이너는 root로 실행됨
docker run ubuntu whoami
# root

# root 컨테이너가 뚫리면...
# bind mount된 호스트 파일에 root 권한으로 접근 가능!
docker run -v /etc:/host/etc ubuntu rm -rf /host/etc
# 호스트 /etc 삭제 위험!
```

### non-root 사용자로 실행

```dockerfile
# Dockerfile에서 비root 사용자 설정
FROM node:18-alpine

# 사용자 생성
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

RUN npm ci --only=production

# 비root 사용자로 전환
USER appuser

CMD ["node", "server.js"]
```

```bash
# 실행 시 사용자 지정
docker run --user 1001:1001 myapp

# 확인
docker exec myapp whoami
# appuser  ← root가 아님
```

### 사용자별 UID 관리

```bash
# 현재 컨테이너 사용자 확인
docker inspect --format '{{.Config.User}}' myapp

# 특정 UID/GID로 실행
docker run --user $(id -u):$(id -g) myapp
```

---

## 17.3 읽기 전용 파일시스템

```bash
# 컨테이너 파일시스템을 읽기 전용으로 설정
docker run --read-only nginx

# 쓰기 필요한 경로만 tmpfs 마운트
docker run \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  nginx
```

```yaml
# Docker Compose에서
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
      - /app/tmp
    volumes:
      - app-logs:/app/logs    # 로그는 named volume으로
```

---

## 17.4 리소스 제한

무제한 리소스 사용은 DoS 공격 등에 취약합니다:

```bash
# 메모리 제한
docker run \
  --memory 512m \           # 최대 512MB
  --memory-swap 512m \      # swap 포함 최대 512MB (swap 비활성화)
  --memory-reservation 256m \ # 메모리 예약 (소프트 제한)
  myapp

# CPU 제한
docker run \
  --cpus 0.5 \              # CPU 0.5개 사용
  --cpu-shares 512 \        # 상대적 가중치 (기본 1024)
  myapp

# PID 제한 (프로세스 폭탄 방지)
docker run \
  --pids-limit 100 \        # 최대 100개 프로세스
  myapp
```

```yaml
# Docker Compose
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
          pids: 100
        reservations:
          memory: 256M
```

---

## 17.5 네트워크 보안

### 내부 네트워크 격리

```yaml
services:
  nginx:
    ports:
      - "80:80"
    networks:
      - public

  app:
    expose:
      - "8080"           # 외부 포트 노출 없음
    networks:
      - public
      - private

  db:
    expose:
      - "5432"
    networks:
      - private          # 내부에서만 접근 가능

networks:
  public:
    driver: bridge
  private:
    driver: bridge
    internal: true       # 외부 인터넷 차단
```

### Linux Capabilities 제한

```bash
# 모든 capabilities 제거 후 필요한 것만 추가
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \  # 포트 80 바인딩
  nginx

# 자주 사용하는 capabilities
# NET_BIND_SERVICE: 1024 이하 포트 바인딩
# CHOWN: 파일 소유권 변경
# SETUID/SETGID: UID/GID 변경
```

```yaml
# Docker Compose
services:
  web:
    image: nginx
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true   # 권한 상승 방지
```

---

## 17.6 Secrets 관리

### 환경변수 대신 파일 사용 (권장)

```bash
# 나쁜 예 - 환경변수로 시크릿 전달
docker run -e DB_PASSWORD=mysecret myapp
# docker inspect, ps aux 등에서 노출 위험!

# 좋은 예 - 파일로 전달
echo "mysecret" > /run/secrets/db_password
docker run \
  -v /run/secrets/db_password:/run/secrets/db_password:ro \
  myapp

# 앱에서 파일 읽기
# fs.readFileSync('/run/secrets/db_password', 'utf8').trim()
```

### Docker Secrets (Swarm 모드)

```bash
# Secret 생성
echo "mysecretpassword" | docker secret create db_password -

# 서비스에 Secret 연결
docker service create \
  --name myapp \
  --secret db_password \
  myapp
# /run/secrets/db_password 파일로 마운트됨
```

### Docker Compose Secrets

```yaml
services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt    # 로컬 파일
  api_key:
    external: true                     # 외부(Swarm) 시크릿
```

---

## 17.7 Docker Bench for Security

Docker의 보안 설정을 자동으로 검사합니다:

```bash
# Docker Bench Security 실행
docker run --rm \
  -v /etc:/etc:ro \
  -v /usr/bin/docker:/usr/bin/docker:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --pid host \
  --network host \
  --label docker_bench_security \
  docker/docker-bench-security

# 출력 예시
# [PASS] 1 - Host Configuration
# [WARN] 1.1 - Ensure a separate partition for containers is used
# [PASS] 2 - Docker Daemon Configuration
# [WARN] 2.1 - Ensure network traffic is restricted between containers
```

### 주요 체크 항목

| 분류 | 체크 항목 |
|------|----------|
| 호스트 설정 | 컨테이너 전용 파티션, 커널 업데이트 |
| Docker 데몬 | TLS 설정, API 접근 제어 |
| 컨테이너 이미지 | 최신 베이스 이미지, 비root 사용자 |
| 컨테이너 런타임 | 리소스 제한, 권한 설정 |

---

## 17.8 보안 모범 사례 체크리스트

```
이미지 보안
[ ] 공식 이미지 또는 신뢰할 수 있는 소스 사용
[ ] 이미지 버전 고정 (latest 사용 금지)
[ ] 정기적인 취약점 스캔
[ ] 최소 기반 이미지 사용 (alpine, distroless)

컨테이너 보안
[ ] root가 아닌 사용자로 실행
[ ] 읽기 전용 파일시스템 사용
[ ] 리소스 제한 설정 (CPU, 메모리, PID)
[ ] Linux Capabilities 최소화
[ ] no-new-privileges 설정

네트워크 보안
[ ] 필요한 포트만 노출
[ ] 내부 서비스 격리 (internal network)
[ ] TLS/HTTPS 사용

시크릿 관리
[ ] 환경변수 대신 파일/시크릿 사용
[ ] 이미지에 시크릿 포함 금지
[ ] .dockerignore에 민감 파일 추가
```

---

## 핵심 요약

1. **non-root 실행**: 보안의 기본, `USER` 명령어 항상 사용
2. **읽기 전용 파일시스템**: `--read-only` 설정
3. **리소스 제한**: OOM, DoS 공격 방지
4. **네트워크 격리**: `internal: true` 네트워크 활용
5. **Capabilities 최소화**: `--cap-drop ALL` 후 필요한 것만 추가
6. **Secrets**: 환경변수 대신 파일/볼륨으로 전달

---

## 다음 단계

[Chapter 18. 이미지 보안 →](18-이미지-보안.md)

---

## 실습 과제

1. 기존 Dockerfile에 비root 사용자 설정 추가하기
2. `--read-only` 모드로 컨테이너 실행해보기
3. `--cap-drop ALL --cap-add NET_BIND_SERVICE`로 nginx 실행하기
4. Docker Bench Security 실행하여 보안 점수 확인하기
5. 환경변수 대신 파일로 DB 비밀번호 전달하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**대상**: 백엔드 개발자