# Chapter 30: Kubernetes 배포 (Kubernetes Deployment)

## 학습 목표
- Kubernetes 기본 개념과 아키텍처
- FastAPI를 Kubernetes에 배포
- Deployment, Service, Ingress 구성
- Helm 차트로 패키징
- 오토 스케일링과 롤링 업데이트
- 모니터링과 로깅 (Prometheus + Grafana)

## 1. Kubernetes 기초

### 1.1 주요 개념

**Pod:**
- 가장 작은 배포 단위
- 하나 이상의 컨테이너 그룹

**Deployment:**
- Pod의 선언적 업데이트
- 복제본(Replica) 관리
- 롤링 업데이트

**Service:**
- Pod에 대한 네트워크 접근
- 로드 밸런싱
- 서비스 디스커버리

**Ingress:**
- 외부에서 클러스터로 HTTP/HTTPS 라우팅
- SSL 종료
- 도메인 기반 라우팅

**ConfigMap & Secret:**
- 설정 데이터 관리
- 민감한 정보 관리

### 1.2 Kubernetes 설치

**로컬 개발 (Minikube):**
```bash
# Minikube 설치 (macOS)
brew install minikube

# 클러스터 시작
minikube start --cpus=4 --memory=8192

# kubectl 설치
brew install kubectl

# 클러스터 확인
kubectl cluster-info
kubectl get nodes
```

**클라우드 (GKE 예시):**
```bash
# GKE 클러스터 생성
gcloud container clusters create fastapi-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type n1-standard-2 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 5

# kubectl 설정
gcloud container clusters get-credentials fastapi-cluster --zone us-central1-a
```

## 2. FastAPI 애플리케이션 배포

### 2.1 Deployment 작성

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
  labels:
    app: fastapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        image: your-registry/fastapi-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: fastapi-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: fastapi-config
              key: redis-url
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

### 2.2 Service 작성

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
  labels:
    app: fastapi
spec:
  type: ClusterIP
  selector:
    app: fastapi
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

### 2.3 ConfigMap과 Secret

**ConfigMap:**
```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fastapi-config
data:
  redis-url: "redis://redis-service:6379"
  log-level: "INFO"
  cors-origins: '["http://localhost:3000","https://example.com"]'
```

**Secret:**
```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: fastapi-secrets
type: Opaque
stringData:
  database-url: "postgresql://user:password@postgres-service:5432/mydb"
  secret-key: "your-secret-key-here"
  jwt-secret: "your-jwt-secret"
```

**생성 (kubectl):**
```bash
# ConfigMap 생성
kubectl apply -f k8s/configmap.yaml

# Secret 생성 (파일에서)
kubectl apply -f k8s/secret.yaml

# 또는 명령어로 직접 생성
kubectl create secret generic fastapi-secrets \
  --from-literal=database-url="postgresql://..." \
  --from-literal=secret-key="your-secret"

# 확인
kubectl get configmaps
kubectl get secrets
kubectl describe secret fastapi-secrets
```

### 2.4 Ingress 작성

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastapi-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: fastapi-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fastapi-service
            port:
              number: 80
```

**Nginx Ingress Controller 설치:**
```bash
# Helm으로 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux

# 확인
kubectl get pods -n ingress-nginx
```

**Cert-Manager (SSL 인증서 자동화):**
```bash
# Cert-Manager 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# ClusterIssuer 생성
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## 3. 데이터베이스 배포

### 3.1 PostgreSQL StatefulSet

```yaml
# k8s/postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 3.2 Redis 배포

```yaml
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

## 4. 배포 명령어

### 4.1 기본 배포

```bash
# 네임스페이스 생성
kubectl create namespace fastapi-prod

# Secret 및 ConfigMap 생성
kubectl apply -f k8s/secret.yaml -n fastapi-prod
kubectl apply -f k8s/configmap.yaml -n fastapi-prod

# 데이터베이스 배포
kubectl apply -f k8s/postgres.yaml -n fastapi-prod
kubectl apply -f k8s/redis.yaml -n fastapi-prod

# 애플리케이션 배포
kubectl apply -f k8s/deployment.yaml -n fastapi-prod
kubectl apply -f k8s/service.yaml -n fastapi-prod
kubectl apply -f k8s/ingress.yaml -n fastapi-prod

# 배포 상태 확인
kubectl get all -n fastapi-prod
kubectl get ingress -n fastapi-prod

# Pod 로그 확인
kubectl logs -f deployment/fastapi-deployment -n fastapi-prod

# Pod 접속
kubectl exec -it deployment/fastapi-deployment -n fastapi-prod -- /bin/bash
```

### 4.2 업데이트 및 롤백

```bash
# 이미지 업데이트
kubectl set image deployment/fastapi-deployment \
  fastapi=your-registry/fastapi-app:v1.1 \
  -n fastapi-prod

# 롤아웃 상태 확인
kubectl rollout status deployment/fastapi-deployment -n fastapi-prod

# 롤아웃 히스토리
kubectl rollout history deployment/fastapi-deployment -n fastapi-prod

# 롤백 (이전 버전으로)
kubectl rollout undo deployment/fastapi-deployment -n fastapi-prod

# 특정 리비전으로 롤백
kubectl rollout undo deployment/fastapi-deployment --to-revision=2 -n fastapi-prod
```

## 5. Horizontal Pod Autoscaler (HPA)

### 5.1 HPA 설정

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

**적용 및 확인:**
```bash
# Metrics Server 설치 (필요시)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# HPA 생성
kubectl apply -f k8s/hpa.yaml -n fastapi-prod

# HPA 상태 확인
kubectl get hpa -n fastapi-prod

# 부하 테스트 (HPA 동작 확인)
kubectl run -it --rm load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://fastapi-service; done"

# 실시간 확인
watch kubectl get hpa,deployment -n fastapi-prod
```

## 6. Helm 차트

### 6.1 Helm 차트 구조

```
fastapi-helm/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── values-dev.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    └── hpa.yaml
```

**Chart.yaml:**
```yaml
apiVersion: v2
name: fastapi-app
description: A Helm chart for FastAPI application
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - fastapi
  - python
  - api
maintainers:
  - name: Your Name
    email: your-email@example.com
```

**values.yaml:**
```yaml
# 기본 설정
replicaCount: 3

image:
  repository: your-registry/fastapi-app
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: fastapi-tls-cert
      hosts:
        - api.example.com

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  ENVIRONMENT: production
  LOG_LEVEL: INFO

secrets:
  DATABASE_URL: ""
  SECRET_KEY: ""

postgresql:
  enabled: true
  auth:
    username: user
    password: password
    database: mydb

redis:
  enabled: true
```

**templates/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "fastapi-app.fullname" . }}
  labels:
    {{- include "fastapi-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "fastapi-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "fastapi-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        {{- range $key, $value := .Values.secrets }}
        - name: {{ $key }}
          valueFrom:
            secretKeyRef:
              name: {{ include "fastapi-app.fullname" $ }}-secrets
              key: {{ $key }}
        {{- end }}
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### 6.2 Helm 배포

```bash
# Helm 설치
brew install helm

# 차트 검증
helm lint fastapi-helm/

# Dry run (실제 배포 없이 확인)
helm install fastapi-app fastapi-helm/ \
  --dry-run \
  --debug \
  --namespace fastapi-prod

# 설치
helm install fastapi-app fastapi-helm/ \
  --namespace fastapi-prod \
  --create-namespace \
  --values fastapi-helm/values-prod.yaml

# 업그레이드
helm upgrade fastapi-app fastapi-helm/ \
  --namespace fastapi-prod \
  --values fastapi-helm/values-prod.yaml

# 롤백
helm rollback fastapi-app 1 --namespace fastapi-prod

# 삭제
helm uninstall fastapi-app --namespace fastapi-prod

# 상태 확인
helm list --namespace fastapi-prod
helm status fastapi-app --namespace fastapi-prod
```

## 7. 모니터링 (Prometheus + Grafana)

### 7.1 Prometheus Stack 설치

```bash
# Helm으로 Prometheus Operator 설치
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### 7.2 ServiceMonitor 생성

```yaml
# k8s/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fastapi-metrics
  labels:
    app: fastapi
spec:
  selector:
    matchLabels:
      app: fastapi
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

### 7.3 Grafana 대시보드 접근

```bash
# Grafana 비밀번호 확인
kubectl get secret --namespace monitoring prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# 포트 포워딩
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# 브라우저에서 접속: http://localhost:3000
# ID: admin, PW: (위에서 확인한 비밀번호)
```

## 8. 로깅 (EFK Stack)

### 8.1 Elasticsearch + Fluentd + Kibana

```bash
# Elastic Cloud on Kubernetes (ECK) 설치
kubectl create -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml

# Elasticsearch 배포
kubectl apply -f - <<EOF
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: logging
spec:
  version: 8.11.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF

# Kibana 배포
kubectl apply -f - <<EOF
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: logging
spec:
  version: 8.11.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
EOF
```

## 9. CI/CD 파이프라인

### 9.1 GitHub Actions + Kubernetes

```yaml
# .github/workflows/k8s-deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        push: true
        tags: ${{ secrets.DOCKER_REGISTRY }}/fastapi-app:${{ github.sha }}

    - name: Install kubectl
      uses: azure/setup-kubectl@v3

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig

    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/fastapi-deployment \
          fastapi=${{ secrets.DOCKER_REGISTRY }}/fastapi-app:${{ github.sha }} \
          -n fastapi-prod

    - name: Wait for rollout
      run: |
        kubectl rollout status deployment/fastapi-deployment -n fastapi-prod
```

## 10. 운영 체크리스트

### 10.1 배포 전

- [ ] 리소스 제한 설정 (CPU, Memory)
- [ ] Liveness/Readiness Probe 설정
- [ ] Secret과 ConfigMap 분리
- [ ] 이미지 태그 버전 명시 (latest 지양)
- [ ] PersistentVolume 백업 전략

### 10.2 배포 후

- [ ] Pod 상태 확인
- [ ] 로그 확인
- [ ] 메트릭 모니터링
- [ ] Ingress 접근 테스트
- [ ] 오토 스케일링 동작 확인

### 10.3 모니터링

- [ ] Prometheus 메트릭 수집
- [ ] Grafana 대시보드 설정
- [ ] 알림 규칙 설정
- [ ] 로그 수집 (EFK/ELK)

## 11. 요약

### 핵심 포인트

1. **Kubernetes 기본 리소스**
   - Deployment: 애플리케이션 배포
   - Service: 네트워크 접근
   - Ingress: 외부 라우팅
   - ConfigMap/Secret: 설정 관리

2. **고가용성**
   - 여러 개의 Replica
   - Liveness/Readiness Probe
   - 롤링 업데이트

3. **오토 스케일링**
   - HPA로 자동 확장
   - CPU/메모리 기반

4. **Helm**
   - 패키지 관리
   - 재사용 가능한 차트
   - 버전 관리

5. **모니터링**
   - Prometheus + Grafana
   - EFK Stack 로깅
   - 알림 설정

### 다음 챕터 예고

**Chapter 31: 실전 프로젝트 - 블로그 API (Real-world Project - Blog API)**

다음 챕터에서는 실전 프로젝트를 구축합니다:
- 완전한 블로그 API 구현
- 사용자 인증 및 권한
- CRUD 작업
- 검색 및 페이지네이션
- 이미지 업로드
- 댓글 시스템

Kubernetes로 확장 가능하고 안정적인 프로덕션 환경을 구축하세요!