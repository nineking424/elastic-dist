# ELK Stack Kubernetes 배포 가이드

Kubernetes 클러스터에 ELK Stack을 배포하는 방법을 안내합니다.

## 사전 요구사항

### 필수 도구
- **kubectl**: Kubernetes CLI 도구
- **Kubernetes 클러스터**: 1.24 이상 권장
- **Helm** (선택사항): 패키지 매니저

### 클러스터 요구사항
- **노드 수**: 최소 3개 (프로덕션)
- **메모리**: 노드당 최소 8GB
- **스토리지**: PersistentVolume 지원 필요

### 접속 확인

```bash
# kubectl 버전 확인
kubectl version

# 클러스터 연결 확인
kubectl cluster-info

# 노드 상태 확인
kubectl get nodes
```

## 디렉토리 구조

```
k8s/
├── namespace.yaml
├── elasticsearch-configmap.yaml
├── logstash-configmap.yaml
├── kibana-configmap.yaml
├── elasticsearch-statefulset.yaml
├── logstash-deployment.yaml
├── kibana-deployment.yaml
├── otel-collector-daemonset.yaml
└── elk-ingress.yaml
```

## 리소스 배포

### 배포 순서

```bash
# 1. 네임스페이스 생성
kubectl apply -f namespace.yaml

# 2. ConfigMap 적용
kubectl apply -f elasticsearch-configmap.yaml
kubectl apply -f logstash-configmap.yaml
kubectl apply -f kibana-configmap.yaml

# 3. Elasticsearch 배포 (먼저 시작되어야 함)
kubectl apply -f elasticsearch-statefulset.yaml

# 4. Elasticsearch 준비 대기
kubectl wait --for=condition=ready pod -l app=elasticsearch -n elastic-system --timeout=300s

# 5. Logstash, Kibana 배포
kubectl apply -f logstash-deployment.yaml
kubectl apply -f kibana-deployment.yaml

# 6. OpenTelemetry Collector 배포
kubectl apply -f otel-collector-daemonset.yaml

# 7. Ingress 설정 (선택사항)
kubectl apply -f elk-ingress.yaml
```

### 한 번에 배포

```bash
# 모든 리소스 한 번에 배포
kubectl apply -f .
```

### 배포 상태 확인

```bash
# 전체 리소스 확인
kubectl get all -n elastic-system

# Pod 상태 확인
kubectl get pods -n elastic-system -w

# StatefulSet 상태 확인
kubectl get statefulset -n elastic-system

# PVC 상태 확인
kubectl get pvc -n elastic-system
```

## Secret 설정 (선택사항)

프로덕션 환경에서 보안 설정 시 사용합니다:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elastic-credentials
  namespace: elastic-system
type: Opaque
stringData:
  ELASTIC_PASSWORD: "your_secure_password"
  KIBANA_PASSWORD: "your_kibana_password"
```

```bash
kubectl apply -f elastic-secrets.yaml
```

## StorageClass 설정

### 클라우드 환경

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-storage
provisioner: kubernetes.io/gce-pd  # GKE
# provisioner: kubernetes.io/aws-ebs  # EKS
# provisioner: kubernetes.io/azure-disk  # AKS
parameters:
  type: pd-ssd
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

## 로컬 접속 (Port Forward)

```bash
# Kibana 접속
kubectl port-forward svc/kibana 5601:5601 -n elastic-system

# Elasticsearch 접속
kubectl port-forward svc/elasticsearch 9200:9200 -n elastic-system
```

## 모니터링 및 운영

### 로그 확인

```bash
# Elasticsearch 로그
kubectl logs -f elasticsearch-0 -n elastic-system

# Logstash 로그
kubectl logs -f deployment/logstash -n elastic-system

# Kibana 로그
kubectl logs -f deployment/kibana -n elastic-system

# 모든 Pod 로그 (label 기준)
kubectl logs -l app=elasticsearch -n elastic-system --tail=100
```

### 상태 확인

```bash
# Elasticsearch 클러스터 상태
kubectl exec -it elasticsearch-0 -n elastic-system -- curl -s http://localhost:9200/_cluster/health?pretty

# 노드 확인
kubectl exec -it elasticsearch-0 -n elastic-system -- curl -s http://localhost:9200/_cat/nodes?v

# 인덱스 확인
kubectl exec -it elasticsearch-0 -n elastic-system -- curl -s http://localhost:9200/_cat/indices?v

# 샤드 상태
kubectl exec -it elasticsearch-0 -n elastic-system -- curl -s http://localhost:9200/_cat/shards?v
```

### 리소스 모니터링

```bash
# 리소스 사용량
kubectl top pods -n elastic-system

# Pod 상세 정보
kubectl describe pod elasticsearch-0 -n elastic-system

# 이벤트 확인
kubectl get events -n elastic-system --sort-by='.lastTimestamp'
```

## 트러블슈팅

### 일반적인 문제

1. **Pod CrashLoopBackOff**
   ```bash
   # 로그 확인
   kubectl logs elasticsearch-0 -n elastic-system --previous

   # 이벤트 확인
   kubectl describe pod elasticsearch-0 -n elastic-system
   ```

2. **PVC Pending**
   ```bash
   # StorageClass 확인
   kubectl get storageclass

   # PVC 상태 확인
   kubectl describe pvc -n elastic-system
   ```

3. **Elasticsearch 클러스터 형성 실패**
   ```bash
   # DNS 확인
   kubectl exec -it elasticsearch-0 -n elastic-system -- nslookup elasticsearch-headless

   # 네트워크 연결 확인
   kubectl exec -it elasticsearch-0 -n elastic-system -- curl -s elasticsearch-1.elasticsearch-headless:9200
   ```

4. **메모리 부족 (OOMKilled)**
   - resources.limits.memory 증가
   - ES_JAVA_OPTS 힙 크기 조정

### 재시작 및 복구

```bash
# Pod 재시작
kubectl delete pod elasticsearch-0 -n elastic-system

# Deployment 재시작
kubectl rollout restart deployment/kibana -n elastic-system

# StatefulSet 롤링 재시작
kubectl rollout restart statefulset/elasticsearch -n elastic-system
```

### 데이터 백업

```bash
# 스냅샷 리포지토리 등록 (S3 예시)
kubectl exec -it elasticsearch-0 -n elastic-system -- curl -X PUT "localhost:9200/_snapshot/backup_repo" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "your-backup-bucket",
    "region": "ap-northeast-2"
  }
}'

# 스냅샷 생성
kubectl exec -it elasticsearch-0 -n elastic-system -- curl -X PUT "localhost:9200/_snapshot/backup_repo/snapshot_1?wait_for_completion=true"
```

## 정리 (삭제)

```bash
# 전체 리소스 삭제
kubectl delete -f .

# PVC 삭제 (데이터 손실 주의)
kubectl delete pvc -l app=elasticsearch -n elastic-system

# 네임스페이스 삭제
kubectl delete namespace elastic-system
```

## Helm을 이용한 설치 (대안)

ECK (Elastic Cloud on Kubernetes) Operator를 사용한 설치:

```bash
# Elastic Helm 저장소 추가
helm repo add elastic https://helm.elastic.co
helm repo update

# ECK Operator 설치
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace

# Elasticsearch 배포 (CRD 사용)
kubectl apply -f - <<EOF
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
  namespace: elastic-system
spec:
  version: 8.12.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
EOF
```

자세한 내용은 [ECK 공식 문서](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)를 참조하세요.
