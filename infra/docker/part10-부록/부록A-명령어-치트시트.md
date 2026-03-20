# 부록 A. Docker 명령어 치트시트

## 이미지 관련 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `docker pull` | 이미지 다운로드 | `docker pull nginx:1.25` |
| `docker push` | 이미지 업로드 | `docker push user/app:1.0` |
| `docker images` | 이미지 목록 | `docker images` |
| `docker image ls` | 이미지 목록 (동일) | `docker image ls` |
| `docker rmi` | 이미지 삭제 | `docker rmi nginx:latest` |
| `docker image rm` | 이미지 삭제 (동일) | `docker image rm nginx` |
| `docker tag` | 이미지 태그 | `docker tag nginx user/nginx:1.0` |
| `docker build` | 이미지 빌드 | `docker build -t myapp:1.0 .` |
| `docker history` | 이미지 레이어 이력 | `docker history nginx` |
| `docker inspect` | 이미지 상세 정보 | `docker inspect nginx` |
| `docker save` | 이미지를 파일로 | `docker save nginx > nginx.tar` |
| `docker load` | 파일에서 이미지 로드 | `docker load < nginx.tar` |
| `docker search` | 이미지 검색 | `docker search --filter is-official=true nginx` |
| `docker image prune` | 미사용 이미지 삭제 | `docker image prune -a` |

---

## 컨테이너 관련 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `docker run` | 컨테이너 생성+실행 | `docker run -d -p 80:80 nginx` |
| `docker create` | 컨테이너 생성만 | `docker create --name myapp nginx` |
| `docker start` | 컨테이너 시작 | `docker start myapp` |
| `docker stop` | 컨테이너 중지 | `docker stop myapp` |
| `docker restart` | 컨테이너 재시작 | `docker restart myapp` |
| `docker kill` | 컨테이너 강제 종료 | `docker kill myapp` |
| `docker pause` | 컨테이너 일시 중지 | `docker pause myapp` |
| `docker unpause` | 컨테이너 재개 | `docker unpause myapp` |
| `docker rm` | 컨테이너 삭제 | `docker rm myapp` |
| `docker ps` | 실행 중인 컨테이너 | `docker ps` |
| `docker ps -a` | 모든 컨테이너 | `docker ps -a` |
| `docker logs` | 컨테이너 로그 | `docker logs -f myapp` |
| `docker exec` | 컨테이너 명령 실행 | `docker exec -it myapp bash` |
| `docker attach` | 컨테이너 연결 | `docker attach myapp` |
| `docker cp` | 파일 복사 | `docker cp myapp:/app/log ./log` |
| `docker diff` | 파일시스템 변경 | `docker diff myapp` |
| `docker commit` | 컨테이너→이미지 | `docker commit myapp myimage:1.0` |
| `docker inspect` | 컨테이너 상세 정보 | `docker inspect myapp` |
| `docker rename` | 컨테이너 이름 변경 | `docker rename old-name new-name` |
| `docker stats` | 리소스 사용량 | `docker stats --no-stream` |
| `docker port` | 포트 매핑 확인 | `docker port myapp` |
| `docker top` | 컨테이너 프로세스 | `docker top myapp` |
| `docker export` | 컨테이너 파일시스템 | `docker export myapp > app.tar` |
| `docker container prune` | 중지된 컨테이너 삭제 | `docker container prune` |

---

## 네트워크 관련 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `docker network ls` | 네트워크 목록 | `docker network ls` |
| `docker network create` | 네트워크 생성 | `docker network create mynet` |
| `docker network rm` | 네트워크 삭제 | `docker network rm mynet` |
| `docker network inspect` | 네트워크 상세 정보 | `docker network inspect mynet` |
| `docker network connect` | 네트워크 연결 | `docker network connect mynet myapp` |
| `docker network disconnect` | 네트워크 연결 해제 | `docker network disconnect mynet myapp` |
| `docker network prune` | 미사용 네트워크 삭제 | `docker network prune` |

---

## 볼륨 관련 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `docker volume ls` | 볼륨 목록 | `docker volume ls` |
| `docker volume create` | 볼륨 생성 | `docker volume create mydata` |
| `docker volume rm` | 볼륨 삭제 | `docker volume rm mydata` |
| `docker volume inspect` | 볼륨 상세 정보 | `docker volume inspect mydata` |
| `docker volume prune` | 미사용 볼륨 삭제 | `docker volume prune` |

---

## 시스템 관련 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `docker info` | Docker 시스템 정보 | `docker info` |
| `docker version` | Docker 버전 정보 | `docker version` |
| `docker system df` | 디스크 사용량 | `docker system df -v` |
| `docker system prune` | 미사용 리소스 정리 | `docker system prune -a` |
| `docker system events` | Docker 이벤트 스트림 | `docker system events` |

---

## Docker Compose 명령어

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `docker compose up` | 서비스 시작 | `docker compose up -d` |
| `docker compose down` | 서비스 중지+제거 | `docker compose down -v` |
| `docker compose start` | 서비스 시작 (컨테이너 유지) | `docker compose start app` |
| `docker compose stop` | 서비스 중지 (컨테이너 유지) | `docker compose stop app` |
| `docker compose restart` | 서비스 재시작 | `docker compose restart app` |
| `docker compose ps` | 서비스 상태 | `docker compose ps` |
| `docker compose logs` | 서비스 로그 | `docker compose logs -f app` |
| `docker compose exec` | 서비스 명령 실행 | `docker compose exec app bash` |
| `docker compose build` | 이미지 빌드 | `docker compose build --no-cache` |
| `docker compose pull` | 이미지 다운로드 | `docker compose pull` |
| `docker compose push` | 이미지 업로드 | `docker compose push` |
| `docker compose config` | 설정 확인 | `docker compose config` |
| `docker compose top` | 서비스 프로세스 | `docker compose top` |
| `docker compose run` | 일회성 명령 실행 | `docker compose run app npm test` |

---

## 자주 쓰는 옵션 정리

### docker run 옵션

| 옵션 | 설명 | 예시 |
|------|------|------|
| `-d` | 백그라운드 실행 | `docker run -d nginx` |
| `-it` | 대화형 터미널 | `docker run -it ubuntu bash` |
| `-p` | 포트 매핑 | `docker run -p 8080:80 nginx` |
| `-v` | 볼륨 마운트 | `docker run -v data:/app/data nginx` |
| `--name` | 컨테이너 이름 | `docker run --name myapp nginx` |
| `-e` | 환경변수 설정 | `docker run -e KEY=value nginx` |
| `--env-file` | env 파일 사용 | `docker run --env-file .env nginx` |
| `--rm` | 종료 시 자동 삭제 | `docker run --rm nginx` |
| `--network` | 네트워크 지정 | `docker run --network mynet nginx` |
| `--restart` | 재시작 정책 | `docker run --restart unless-stopped nginx` |
| `--memory` | 메모리 제한 | `docker run --memory 512m nginx` |
| `--cpus` | CPU 제한 | `docker run --cpus 0.5 nginx` |
| `-u` | 사용자 지정 | `docker run -u 1001 nginx` |
| `--read-only` | 읽기 전용 파일시스템 | `docker run --read-only nginx` |

### docker build 옵션

| 옵션 | 설명 | 예시 |
|------|------|------|
| `-t` | 이미지 태그 | `docker build -t myapp:1.0 .` |
| `-f` | Dockerfile 지정 | `docker build -f Dockerfile.prod .` |
| `--no-cache` | 캐시 사용 안 함 | `docker build --no-cache .` |
| `--build-arg` | 빌드 인수 | `docker build --build-arg VERSION=1.0 .` |
| `--target` | 특정 스테이지 | `docker build --target build .` |
| `--platform` | 플랫폼 지정 | `docker build --platform linux/amd64 .` |

---

## 유용한 조합 명령어

```bash
# 모든 컨테이너 중지
docker stop $(docker ps -q)

# 중지된 모든 컨테이너 삭제
docker rm $(docker ps -aq)

# 태그 없는 이미지 모두 삭제
docker rmi $(docker images -f "dangling=true" -q)

# 전체 시스템 정리 (볼륨 포함)
docker system prune -a --volumes -f

# 컨테이너 IP 주소 확인
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' 컨테이너명

# 실행 중인 컨테이너의 환경변수 확인
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' 컨테이너명

# 이미지 레이어 크기 상위 10개
docker history nginx --format "{{.Size}}\t{{.CreatedBy}}" | sort -rh | head -10
```

---

**작성일**: 2024년
**Docker 버전**: 24.x+