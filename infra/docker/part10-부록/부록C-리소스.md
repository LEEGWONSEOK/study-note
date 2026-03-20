# 부록 C. 리소스 및 참고 자료

## C.1 공식 문서

### Docker 공식 문서

| 자료 | URL | 설명 |
|------|-----|------|
| Docker 문서 | https://docs.docker.com | 공식 전체 문서 |
| Docker CLI Reference | https://docs.docker.com/engine/reference/commandline/cli/ | CLI 명령어 레퍼런스 |
| Dockerfile Reference | https://docs.docker.com/engine/reference/builder/ | Dockerfile 명령어 |
| Compose Reference | https://docs.docker.com/compose/compose-file/ | Compose 파일 레퍼런스 |
| Docker Hub | https://hub.docker.com | 공식 이미지 저장소 |
| Docker Blog | https://www.docker.com/blog/ | 공식 블로그 |

### 관련 프로젝트 문서

| 자료 | URL | 설명 |
|------|-----|------|
| BuildKit | https://docs.docker.com/build/buildkit/ | 고급 빌드 기능 |
| Docker Scout | https://docs.docker.com/scout/ | 이미지 보안 분석 |
| Docker Swarm | https://docs.docker.com/engine/swarm/ | 컨테이너 오케스트레이션 |
| Kubernetes | https://kubernetes.io/docs/ | K8s 공식 문서 |

---

## C.2 추천 도서

| 도서 | 저자 | 특징 |
|------|------|------|
| **Docker Deep Dive** | Nigel Poulton | 입문서, 읽기 쉬움 |
| **Docker in Action** | Jeff Nickoloff | 실습 중심 |
| **Docker: Up & Running** | Sean P. Kane, Karl Matthias | 중급~고급 |
| **Kubernetes in Action** | Marko Luksa | K8s 심화 |
| **Container Security** | Liz Rice | 컨테이너 보안 전문 |

---

## C.3 추천 강의

### 한국어 강의

| 강의 | 플랫폼 | 특징 |
|------|--------|------|
| 따라하며 배우는 도커 | 인프런 | 한국어, 입문자용 |
| DevOps를 위한 쿠버네티스 마스터 | 인프런 | 실습 중심 |
| Docker & Kubernetes: The Practical Guide | 유데미 (한국어 자막) | 체계적인 커리큘럼 |

### 영어 강의

| 강의 | 플랫폼 | 특징 |
|------|--------|------|
| Docker Mastery | 유데미 (Bret Fisher) | 가장 유명, 정기 업데이트 |
| Learn Docker by Doing | Linux Foundation | 실습 위주 |
| Docker Certified Associate | Linux Foundation | 자격증 준비 |

---

## C.4 유용한 도구

### 이미지 관련

| 도구 | 용도 | URL |
|------|------|-----|
| **Trivy** | 이미지 취약점 스캔 | https://github.com/aquasecurity/trivy |
| **Dive** | 이미지 레이어 탐색 | https://github.com/wagoodman/dive |
| **Hadolint** | Dockerfile 린터 | https://github.com/hadolint/hadolint |
| **Docker Slim** | 이미지 크기 최소화 | https://github.com/docker-slim/docker-slim |

### 관리 UI

| 도구 | 용도 | URL |
|------|------|-----|
| **Portainer** | Docker Web UI | https://portainer.io |
| **Lazydocker** | TUI (터미널 UI) | https://github.com/jesseduffield/lazydocker |
| **ctop** | 컨테이너 top | https://github.com/bcicen/ctop |

### 개발 도구

| 도구 | 용도 | URL |
|------|------|-----|
| **Kompose** | Compose → K8s 변환 | https://kompose.io |
| **Buildpacks** | Dockerfile 없는 빌드 | https://buildpacks.io |
| **Skaffold** | K8s 개발 워크플로우 | https://skaffold.dev |

---

## C.5 레지스트리 참고 사이트

### 공개 레지스트리

| 레지스트리 | URL | 특징 |
|-----------|-----|------|
| **Docker Hub** | https://hub.docker.com | 가장 큰 공개 레지스트리 |
| **GitHub Container Registry** | https://ghcr.io | GitHub 통합 |
| **Quay.io** | https://quay.io | Red Hat 운영 |
| **Google Container Registry** | https://gcr.io | GCP 공식 |

### 공식 이미지 카탈로그

- **Docker Official Images**: https://hub.docker.com/search?image_filter=official
- **Verified Publishers**: https://hub.docker.com/search?image_filter=store
- **Docker Sponsored OSS**: https://hub.docker.com/search?image_filter=open_source

---

## C.6 Docker 커뮤니티

| 커뮤니티 | URL | 특징 |
|---------|-----|------|
| Docker 공식 포럼 | https://forums.docker.com | 공식 Q&A |
| Stack Overflow | https://stackoverflow.com/questions/tagged/docker | 기술 Q&A |
| Reddit r/docker | https://reddit.com/r/docker | 커뮤니티 토론 |
| Docker Discord | https://discord.gg/docker | 실시간 채팅 |

---

## C.7 실습 환경

### 온라인 실습 환경 (설치 없이)

| 환경 | URL | 특징 |
|------|-----|------|
| **Play with Docker** | https://labs.play-with-docker.com | 4시간 무료 실습 환경 |
| **Killercoda** | https://killercoda.com | 인터랙티브 시나리오 |
| **KodeKloud** | https://kodekloud.com | 실습 기반 학습 |

---

## C.8 참고 GitHub 저장소

| 저장소 | 설명 |
|--------|------|
| `docker/awesome-compose` | Compose 예제 모음 |
| `veggiemonk/awesome-docker` | Docker 도구 큐레이션 |
| `wsargent/docker-cheat-sheet` | 치트시트 |
| `moby/buildkit` | BuildKit 소스 |

---

## C.9 자격증

| 자격증 | 발행 기관 | 수준 |
|--------|---------|------|
| **DCA (Docker Certified Associate)** | Docker Inc. | 중급 |
| **CKA (Certified Kubernetes Administrator)** | CNCF | 중급~고급 |
| **CKAD (Certified Kubernetes Application Developer)** | CNCF | 중급 |
| **AWS Certified DevOps Engineer** | AWS | 고급 |

---

**작성일**: 2024년
**Docker 버전**: 24.x+