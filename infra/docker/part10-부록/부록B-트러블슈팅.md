# 부록 B. 트러블슈팅

## B.1 컨테이너가 바로 종료될 때

### 증상

```bash
docker run myapp
# 컨테이너가 즉시 종료됨

docker ps -a
# STATUS: Exited (0) 2 seconds ago
# STATUS: Exited (1) 3 seconds ago
```

### 원인 및 해결

**원인 1: 포그라운드 프로세스 없음**

```dockerfile
# 나쁜 예 - 스크립트가 완료되면 종료
CMD echo "Hello"

# 좋은 예 - 포그라운드에서 계속 실행
CMD ["nginx", "-g", "daemon off;"]
CMD ["java", "-jar", "app.jar"]
```

**원인 2: 앱이 즉시 에러로 종료**

```bash
# 로그 확인
docker logs 컨테이너명

# 에러 추적
docker run -it myapp sh
# 내부에서 수동으로 명령어 실행하여 에러 확인
```

**원인 3: 환경변수 누락**

```bash
# 필요한 환경변수 확인
docker run -e DB_HOST=localhost -e DB_PASSWORD=secret myapp
```

**원인 4: CMD가 shell form에서 신호 처리 안됨**

```dockerfile
# 나쁜 예 (Exec Form으로 수정)
CMD java -jar app.jar

# 좋은 예
CMD ["java", "-jar", "app.jar"]
```

---

## B.2 포트 충돌 문제

### 증상

```bash
docker run -p 8080:80 nginx
# Error: Bind for 0.0.0.0:8080 failed: port is already allocated
```

### 해결

```bash
# 8080 포트 사용 중인 프로세스 확인
lsof -i :8080              # macOS/Linux
netstat -ano | findstr 8080  # Windows

# 사용 중인 Docker 컨테이너 확인
docker ps | grep 8080

# 해결 방법 1: 다른 포트 사용
docker run -p 8081:80 nginx

# 해결 방법 2: 충돌하는 컨테이너 중지
docker stop 컨테이너명

# 해결 방법 3: 프로세스 종료
kill -9 PID
```

---

## B.3 볼륨 권한 문제

### 증상

```bash
docker run -v /host/path:/app/data myapp
# Permission denied: '/app/data/file.txt'
```

### 원인 및 해결

**원인 1: 사용자 UID 불일치**

```dockerfile
# 컨테이너 사용자 UID 확인
RUN id appuser
# uid=1001(appuser)

# 호스트 디렉토리 권한 설정
chown -R 1001:1001 /host/path
# 또는
chmod 777 /host/path  # 임시 방편 (보안 주의)
```

**원인 2: SELinux/AppArmor 설정**

```bash
# SELinux 레이블 추가
docker run -v /host/path:/app/data:z myapp
# z: 공유 가능, Z: 독점 사용
```

**원인 3: root로 생성된 볼륨**

```bash
# 볼륨 초기화 컨테이너로 권한 설정
docker run --rm \
  -v myvolume:/data \
  busybox chown -R 1001:1001 /data

# 이후 비root 컨테이너 실행
docker run -v myvolume:/data -u 1001 myapp
```

---

## B.4 이미지 빌드 실패

### 증상별 해결

**빌드 컨텍스트가 너무 큼**

```bash
docker build .
# Sending build context to Docker daemon  450.2MB  ← 너무 큼!
```

```
# .dockerignore 파일 생성
node_modules
.git
target/
*.log
```

**패키지 설치 실패**

```dockerfile
# 나쁜 예 - apt 업데이트 없이 설치
RUN apt-get install -y curl

# 좋은 예 - 업데이트 후 설치
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

**캐시 문제로 최신 버전 안됨**

```bash
# 캐시 무효화하여 재빌드
docker build --no-cache -t myapp .
```

**멀티스테이지 COPY 실패**

```dockerfile
# 이전 스테이지 이름 확인
FROM maven:3.9 AS build      # 이름: build
COPY --from=build /app/...   # 정확한 이름 사용
```

**권한 없는 디렉토리 생성**

```dockerfile
# 디렉토리 먼저 생성
RUN mkdir -p /app/data && chown appuser:appuser /app/data
USER appuser
COPY --chown=appuser:appuser . /app/
```

---

## B.5 네트워크 연결 문제

### 컨테이너 간 통신 안 됨

```bash
# 같은 네트워크인지 확인
docker inspect --format '{{json .NetworkSettings.Networks}}' 컨테이너1
docker inspect --format '{{json .NetworkSettings.Networks}}' 컨테이너2

# 같은 네트워크에 추가
docker network connect mynet 컨테이너2

# DNS 해석 테스트
docker exec app1 ping app2
docker exec app1 nslookup app2
```

### 외부 인터넷 접속 안 됨

```bash
# DNS 설정 확인
docker exec myapp cat /etc/resolv.conf

# ping 테스트
docker exec myapp ping 8.8.8.8

# DNS로 해석 테스트
docker exec myapp nslookup google.com

# 해결: Docker 데몬 DNS 설정
# /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

### 포트가 외부에서 접근 안 됨

```bash
# 포트 매핑 확인
docker port 컨테이너명
# 80/tcp -> 0.0.0.0:8080

# 방화벽 확인
ufw status
iptables -L

# 컨테이너 IP 직접 접근 테스트
docker inspect -f '{{.NetworkSettings.IPAddress}}' 컨테이너명
curl http://172.17.0.2:80
```

---

## B.6 디스크 공간 부족

### 증상

```bash
docker build .
# no space left on device
```

### 해결

```bash
# 현재 Docker 디스크 사용량 확인
docker system df

# 단계별 정리
# 1. 중지된 컨테이너 삭제
docker container prune -f

# 2. 사용하지 않는 이미지 삭제
docker image prune -a -f

# 3. 사용하지 않는 볼륨 삭제 (데이터 백업 먼저!)
docker volume prune -f

# 4. 빌드 캐시 삭제
docker builder prune -a -f

# 5. 전체 정리
docker system prune -a --volumes -f

# Docker Desktop (macOS)에서 디스크 이미지 크기 늘리기
# Settings → Resources → Disk image size
```

---

## B.7 컨테이너 OOM (메모리 부족) 문제

### 증상

```bash
docker ps -a
# STATUS: Exited (137) 2 minutes ago  ← 137 = SIGKILL (OOM)

docker inspect myapp
# "OOMKilled": true  ← OOM으로 종료 확인
```

### 해결

```bash
# 메모리 사용량 확인
docker stats myapp

# 메모리 제한 늘리기
docker run --memory 1g myapp

# JVM 메모리 설정 최적화 (Java 앱)
docker run -e JAVA_OPTS="-Xmx512m -XX:+UseContainerSupport" myapp

# 메모리 누수 디버깅
docker stats --no-stream --format "{{.MemUsage}}" myapp
# 시간에 따라 메모리가 계속 증가하면 누수
```

---

## B.8 자주 묻는 질문 (FAQ)

**Q: `docker run`과 `docker start`의 차이는?**
- `docker run`: 이미지에서 새 컨테이너 생성 + 실행
- `docker start`: 이미 생성된(중지된) 컨테이너 재시작

**Q: 이미지를 삭제하려는데 "image is being used" 오류가 납니다.**
```bash
# 해당 이미지를 사용하는 컨테이너 확인
docker ps -a --filter ancestor=이미지명

# 컨테이너 삭제 후 이미지 삭제
docker rm $(docker ps -aq --filter ancestor=이미지명)
docker rmi 이미지명
```

**Q: Docker Desktop이 느립니다.**
- Settings → Resources에서 CPU/메모리 할당 늘리기
- `.dockerignore` 로 빌드 컨텍스트 크기 줄이기
- VirtioFS (파일 공유 구현체) 사용 확인

**Q: Apple Silicon(M1/M2)에서 x86 이미지가 실행 안됩니다.**
```bash
# --platform 옵션 사용
docker run --platform linux/amd64 myimage

# 또는 환경변수 설정
export DOCKER_DEFAULT_PLATFORM=linux/amd64
```

**Q: Compose에서 환경변수가 인식되지 않습니다.**
```bash
# .env 파일 위치 확인 (docker-compose.yml과 같은 디렉토리)
ls -la .env

# .env 파일 내용 확인 (따옴표 없이)
DB_PASSWORD=secret   ← 올바름
DB_PASSWORD="secret" ← 따옴표 주의 (일부 환경에서 문제)

# 명시적으로 env 파일 지정
docker compose --env-file .env.prod up -d
```

**Q: 같은 이미지를 여러 컨테이너로 실행하면 디스크를 많이 차지하나요?**
- 아니요! 이미지 레이어는 공유됩니다.
- 각 컨테이너는 쓰기 레이어(변경 내용)만 추가로 사용합니다.

**Q: `docker exec`와 `docker attach`의 차이는?**
- `docker exec`: 실행 중인 컨테이너에 **새 프로세스** 실행
- `docker attach`: 컨테이너의 **메인 프로세스** 표준 입출력에 연결
- 일반적으로 `docker exec -it 컨테이너 bash` 사용 권장

---

**작성일**: 2024년
**Docker 버전**: 24.x+