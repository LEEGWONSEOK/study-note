# Chapter 23. Kubernetes 입문

## 23.1 Kubernetes란?

Kubernetes(K8s)는 컨테이너화된 애플리케이션의 **자동 배포, 스케일링, 관리**를 위한 오케스트레이션 플랫폼입니다.

> **핵심**: Docker가 컨테이너를 실행한다면, Kubernetes는 컨테이너들을 **조율(Orchestrate)** 합니다.

**비유**: Docker는 선박의 **컨테이너 박스**, Kubernetes는 **항구 관제 시스템**입니다.
- 어느 선박(노드)에 어떤 컨테이너를 실을지 결정
- 컨테이너가 손상되면 자동 교체
- 화물량에 따라 컨테이너 수 자동 조정

---

## 23.2 Docker vs Kubernetes

| 항목 | Docker (Compose) | Kubernetes |
|------|-----------------|------------|
| 범위 | 단일 호스트 | 다중 호스트 클러스터 |
| 스케일링 | 수동 | 자동 (HPA) |
| 자가 치유 | 기본 재시작 | Pod 자동 재생성 |
| 로드밸런싱 | Nginx 수동 설정 | 자동 서비스 디스커버리 |
| 롤링 업데이트 | 수동 스크립트 | 자동 Rolling Update |
| 복잡도 | 낮음 | 높음 |
| 학습 곡선 | 완만 | 가파름 |
| 적합한 환경 | 소규모, 개발 | 대규모, 운영 |

---

## 23.3 핵심 개념

### Pod

Kubernetes의 가장 작은 배포 단위입니다. 하나 이상의 컨테이너를 포함합니다.

```yaml
# Pod 예시
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      ports:
        - containerPort: 8080
      env:
        - name: DB_HOST
          value: "mysql-service"
      resources:
        limits:
          memory: "512Mi"
          cpu: "500m"
```

### Node

Pod가 실행되는 **물리적 또는 가상 서버**입니다.

```
Cluster
├── Node 1 (Master)
│   ├── kube-apiserver
│   ├── etcd
│   ├── kube-scheduler
│   └── kube-controller-manager
├── Node 2 (Worker)
│   ├── kubelet
│   ├── kube-proxy
│   └── Pods...
└── Node 3 (Worker)
    ├── kubelet
    ├── kube-proxy
    └── Pods...
```

### Cluster

여러 Node를 묶은 하나의 **Kubernetes 환경**입니다.

### Service

Pod에 접근하기 위한 **네트워크 추상화 계층**입니다. Pod가 재생성되어 IP가 바뀌어도 Service를 통해 안정적으로 접근할 수 있습니다.

```yaml
# Service 예시
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp          # 이 레이블의 Pod들에 트래픽 전달
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP       # 클러스터 내부용 (기본)
```

**Service 타입**:
| 타입 | 설명 |
|------|------|
| ClusterIP | 클러스터 내부에서만 접근 (기본) |
| NodePort | 노드 IP:포트로 외부 접근 |
| LoadBalancer | 클라우드 로드밸런서 생성 |
| ExternalName | 외부 DNS 이름 매핑 |

### Deployment

Pod의 **선언적 업데이트** 및 관리를 담당합니다.

```yaml
# Deployment 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3               # Pod 3개 유지
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # 최대 1개 동시 중지
      maxSurge: 1           # 최대 1개 동시 추가
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
          readinessProbe:   # 준비 완료 확인
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
```

---

## 23.4 minikube로 로컬 실습

### 설치

```bash
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### 클러스터 시작/중지

```bash
# 클러스터 시작
minikube start

# Docker 드라이버로 시작 (Docker Desktop 필요)
minikube start --driver=docker

# 클러스터 상태 확인
minikube status

# 대시보드 실행
minikube dashboard

# 클러스터 중지
minikube stop

# 클러스터 삭제
minikube delete
```

### 로컬 이미지 사용

```bash
# minikube Docker 환경 사용
eval $(minikube docker-env)

# 이미지 빌드 (minikube 내부에 저장됨)
docker build -t myapp:local .

# imagePullPolicy: Never로 설정 (레지스트리에서 pull하지 않음)
```

---

## 23.5 kubectl 기본 명령어

```bash
# 클러스터 정보
kubectl cluster-info
kubectl get nodes

# Pod 관련
kubectl get pods
kubectl get pods -o wide        # 상세 정보 (IP, 노드 포함)
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs -f <pod-name>      # 실시간 로그
kubectl exec -it <pod-name> -- bash

# Deployment 관련
kubectl get deployments
kubectl apply -f deployment.yml  # 생성/업데이트
kubectl delete -f deployment.yml
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp    # 롤백

# Service 관련
kubectl get services
kubectl get svc

# 스케일링
kubectl scale deployment myapp --replicas=5

# 이미지 업데이트
kubectl set image deployment/myapp myapp=myapp:2.0

# 모든 리소스 조회
kubectl get all
```

---

## 23.6 Docker Compose → Kubernetes 마이그레이션

### Compose 파일

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:1.0
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - mysql-data:/var/lib/mysql
```

### 변환된 Kubernetes 매니페스트

```yaml
# kubernetes/app-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: mysql-service

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 8080
  type: LoadBalancer

---
# kubernetes/mysql-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
  type: ClusterIP    # 내부에서만 접근
```

### 자동 변환 도구 (Kompose)

```bash
# Kompose 설치
brew install kompose

# Compose → K8s 변환
kompose convert -f docker-compose.yml

# 변환된 파일들 확인
ls
# app-deployment.yaml
# app-service.yaml
# db-deployment.yaml
# db-service.yaml

# K8s에 배포
kubectl apply -f .
```

---

## 23.7 다음 학습 방향

Docker를 마스터한 후 Kubernetes 학습 순서:

```
1. Kubernetes 기초
   ├── Pod, Deployment, Service
   ├── ConfigMap, Secret
   ├── PersistentVolume, PVC
   └── Namespace

2. 워크로드 관리
   ├── StatefulSet (DB 등)
   ├── DaemonSet (노드별 1개)
   ├── Job, CronJob
   └── HorizontalPodAutoscaler

3. 네트워킹
   ├── Ingress
   ├── Network Policy
   └── Service Mesh (Istio)

4. 스토리지
   ├── PersistentVolume
   ├── StorageClass
   └── 동적 프로비저닝

5. 운영 및 모니터링
   ├── Helm (패키지 관리)
   ├── Prometheus + Grafana
   ├── Loki (로그)
   └── ArgoCD (GitOps)

6. 클라우드 관리형 K8s
   ├── AWS EKS
   ├── GKE
   └── Azure AKS
```

**추천 학습 자료**:
- 공식 문서: https://kubernetes.io/docs/
- Katacoda 대화형 실습
- "Kubernetes in Action" (도서)

---

## 핵심 요약

1. **Kubernetes**: 컨테이너 오케스트레이션, 다중 호스트 관리
2. **Pod**: 최소 배포 단위 (컨테이너 묶음)
3. **Deployment**: Pod의 선언적 관리, 롤링 업데이트
4. **Service**: Pod 접근을 위한 안정적인 엔드포인트
5. **minikube**: 로컬 K8s 환경 (학습용)
6. **Kompose**: Compose → K8s 자동 변환 도구

---

## 다음 단계

Docker 마스터 완료! → Kubernetes 심화 학습으로 이동

---

## 실습 과제

1. minikube 설치 및 클러스터 시작하기
2. kubectl로 nginx Deployment 생성하기
3. Service로 외부 접근 설정하기
4. 간단한 Compose 파일을 Kompose로 K8s 매니페스트로 변환하기
5. 롤링 업데이트 후 kubectl rollout history 확인하기

---

**작성일**: 2024년
**Docker 버전**: 24.x+
**Kubernetes 버전**: 1.28+
**대상**: 백엔드 개발자