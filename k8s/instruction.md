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

## 네임스페이스 생성

```bash
# 네임스페이스 생성
kubectl create namespace elastic-system

# 기본 네임스페이스 설정 (선택사항)
kubectl config set-context --current --namespace=elastic-system
```

## ConfigMap 설정

### Elasticsearch ConfigMap

`elasticsearch-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: elastic-system
data:
  elasticsearch.yml: |
    cluster.name: k8s-elk-cluster
    network.host: 0.0.0.0
    discovery.seed_hosts:
      - elasticsearch-0.elasticsearch-headless
      - elasticsearch-1.elasticsearch-headless
      - elasticsearch-2.elasticsearch-headless
    cluster.initial_master_nodes:
      - elasticsearch-0
      - elasticsearch-1
      - elasticsearch-2
    xpack.security.enabled: false
    xpack.monitoring.collection.enabled: true
```

### Logstash ConfigMap

`logstash-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elastic-system
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    pipeline.workers: 2
  logstash.conf: |
    input {
      # OpenTelemetry Collector에서 데이터 수신
      http {
        port => 5044
        codec => json
      }
    }
    filter {
      if [kubernetes] {
        mutate {
          add_field => {
            "namespace" => "%{[kubernetes][namespace]}"
            "pod_name" => "%{[kubernetes][pod][name]}"
          }
        }
      }
    }
    output {
      elasticsearch {
        hosts => ["http://elasticsearch:9200"]
        index => "logstash-%{+YYYY.MM.dd}"
      }
    }
```

### Kibana ConfigMap

`kibana-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elastic-system
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    elasticsearch.hosts: ["http://elasticsearch:9200"]
    i18n.locale: "ko-KR"
    monitoring.ui.container.elasticsearch.enabled: true
```

### ConfigMap 적용

```bash
kubectl apply -f elasticsearch-configmap.yaml
kubectl apply -f logstash-configmap.yaml
kubectl apply -f kibana-configmap.yaml
```

## Secret 설정 (선택사항)

프로덕션 환경에서 보안 설정 시 사용합니다:

`elastic-secrets.yaml`:

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

## StorageClass 및 PersistentVolume

### StorageClass (클라우드 환경)

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

## Elasticsearch StatefulSet 배포

`elasticsearch-statefulset.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elastic-system
  labels:
    app: elasticsearch
spec:
  type: ClusterIP
  ports:
    - port: 9200
      name: http
    - port: 9300
      name: transport
  selector:
    app: elasticsearch
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-headless
  namespace: elastic-system
  labels:
    app: elasticsearch
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - port: 9200
      name: http
    - port: 9300
      name: transport
  selector:
    app: elasticsearch
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elastic-system
spec:
  serviceName: elasticsearch-headless
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
        - name: fix-permissions
          image: busybox
          command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          volumeMounts:
            - name: elasticsearch-data
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: busybox
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
          ports:
            - containerPort: 9200
              name: http
            - containerPort: 9300
              name: transport
          env:
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: ES_JAVA_OPTS
              value: "-Xms1g -Xmx1g"
          volumeMounts:
            - name: elasticsearch-config
              mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
              subPath: elasticsearch.yml
            - name: elasticsearch-data
              mountPath: /usr/share/elasticsearch/data
            - name: elasticsearch-logs
              mountPath: /usr/share/elasticsearch/logs
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
            initialDelaySeconds: 60
            periodSeconds: 30
      volumes:
        - name: elasticsearch-config
          configMap:
            name: elasticsearch-config
        - name: elasticsearch-logs
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: elasticsearch-storage
        resources:
          requests:
            storage: 30Gi
```

## Logstash Deployment 배포

`logstash-deployment.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: elastic-system
spec:
  type: ClusterIP
  ports:
    - port: 5044
      name: beats
    - port: 9600
      name: http
  selector:
    app: logstash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elastic-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
        - name: logstash
          image: docker.elastic.co/logstash/logstash:8.12.0
          ports:
            - containerPort: 5044
            - containerPort: 9600
          env:
            - name: LS_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
          volumeMounts:
            - name: logstash-config
              mountPath: /usr/share/logstash/config/logstash.yml
              subPath: logstash.yml
            - name: logstash-pipeline
              mountPath: /usr/share/logstash/pipeline/logstash.conf
              subPath: logstash.conf
            - name: logstash-logs
              mountPath: /usr/share/logstash/logs
          resources:
            requests:
              memory: "1Gi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /
              port: 9600
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: logstash-config
          configMap:
            name: logstash-config
            items:
              - key: logstash.yml
                path: logstash.yml
        - name: logstash-pipeline
          configMap:
            name: logstash-config
            items:
              - key: logstash.conf
                path: logstash.conf
        - name: logstash-logs
          emptyDir: {}
```

## Kibana Deployment 배포

`kibana-deployment.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elastic-system
spec:
  type: ClusterIP
  ports:
    - port: 5601
      targetPort: 5601
  selector:
    app: kibana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elastic-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:8.12.0
          ports:
            - containerPort: 5601
          volumeMounts:
            - name: kibana-config
              mountPath: /usr/share/kibana/config/kibana.yml
              subPath: kibana.yml
            - name: kibana-logs
              mountPath: /usr/share/kibana/logs
          resources:
            requests:
              memory: "1Gi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /api/status
              port: 5601
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: kibana-config
          configMap:
            name: kibana-config
        - name: kibana-logs
          emptyDir: {}
```

## OpenTelemetry Collector DaemonSet 배포

`otel-collector-daemonset.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: elastic-system
data:
  otel-collector-config.yaml: |
    receivers:
      # Kubernetes 컨테이너 로그 수집
      filelog:
        include:
          - /var/log/pods/*/*/*.log
          - /var/log/app/*.log
        start_at: end
        include_file_path: true
        include_file_name: false
        operators:
          - type: router
            id: get-format
            routes:
              - output: parser-docker
                expr: 'body matches "^\\{"'
              - output: parser-crio
                expr: 'body matches "^[^ Z]+ "'
              - output: parser-containerd
                expr: 'body matches "^[^ Z]+Z"'
          - type: regex_parser
            id: parser-crio
            regex: '^(?P<time>[^ Z]+) (?P<stream>stdout|stderr) (?P<logtag>[^ ]*) ?(?P<log>.*)$'
            timestamp:
              parse_from: attributes.time
              layout_type: gotime
              layout: '2006-01-02T15:04:05.999999999Z07:00'
          - type: regex_parser
            id: parser-containerd
            regex: '^(?P<time>[^ ^Z]+Z) (?P<stream>stdout|stderr) (?P<logtag>[^ ]*) ?(?P<log>.*)$'
            timestamp:
              parse_from: attributes.time
              layout: '%Y-%m-%dT%H:%M:%S.%LZ'
          - type: json_parser
            id: parser-docker
            timestamp:
              parse_from: attributes.time
              layout: '%Y-%m-%dT%H:%M:%S.%LZ'
          - type: move
            from: attributes.log
            to: body
          - type: move
            from: attributes.stream
            to: attributes["log.iostream"]

      # OTLP 프로토콜 수신
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      # Kubernetes 클러스터 메트릭
      k8s_cluster:
        collection_interval: 30s

    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024

      k8sattributes:
        auth_type: "serviceAccount"
        passthrough: false
        extract:
          metadata:
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.deployment.name
            - k8s.namespace.name
            - k8s.node.name
            - k8s.container.name
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
          - sources:
              - from: resource_attribute
                name: k8s.pod.uid

      resource:
        attributes:
          - key: deployment.environment
            value: kubernetes
            action: upsert

    exporters:
      elasticsearch:
        endpoints:
          - http://elasticsearch:9200
        logs_index: otel-logs
        traces_index: otel-traces
        metrics_index: otel-metrics

      debug:
        verbosity: basic

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133

    service:
      extensions: [health_check]
      pipelines:
        logs:
          receivers: [filelog, otlp]
          processors: [batch, k8sattributes, resource]
          exporters: [elasticsearch]
        traces:
          receivers: [otlp]
          processors: [batch, k8sattributes, resource]
          exporters: [elasticsearch]
        metrics:
          receivers: [otlp, k8s_cluster]
          processors: [batch, resource]
          exporters: [elasticsearch]
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: elastic-system
spec:
  type: ClusterIP
  ports:
    - port: 4317
      name: otlp-grpc
    - port: 4318
      name: otlp-http
    - port: 13133
      name: health
  selector:
    app: otel-collector
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: elastic-system
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector
      terminationGracePeriodSeconds: 30
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.96.0
          args: ["--config=/etc/otel-collector-config.yaml"]
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
            - containerPort: 13133
              name: health
          env:
            - name: K8S_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: K8S_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: config
              mountPath: /etc/otel-collector-config.yaml
              subPath: otel-collector-config.yaml
              readOnly: true
            - name: varlogpods
              mountPath: /var/log/pods
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: varlogapp
              mountPath: /var/log/app
              readOnly: true
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 15
            periodSeconds: 20
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: varlogapp
          hostPath:
            path: /var/log/app
            type: DirectoryOrCreate
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector
  namespace: elastic-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups: [""]
    resources:
      - namespaces
      - pods
      - nodes
      - nodes/stats
      - services
      - endpoints
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources:
      - replicasets
      - deployments
      - daemonsets
      - statefulsets
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources:
      - jobs
      - cronjobs
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector
    namespace: elastic-system
roleRef:
  kind: ClusterRole
  name: otel-collector
  apiGroup: rbac.authorization.k8s.io
```

## 리소스 배포

### 배포 순서

```bash
# 1. 네임스페이스 생성
kubectl create namespace elastic-system

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

## Ingress 설정

### NGINX Ingress

`elk-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
  namespace: elastic-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: kibana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kibana
                port:
                  number: 5601
    - host: elasticsearch.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: elasticsearch
                port:
                  number: 9200
```

### 로컬 접속 (Port Forward)

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
kubectl delete -f otel-collector-daemonset.yaml
kubectl delete -f kibana-deployment.yaml
kubectl delete -f logstash-deployment.yaml
kubectl delete -f elasticsearch-statefulset.yaml
kubectl delete -f elasticsearch-configmap.yaml
kubectl delete -f logstash-configmap.yaml
kubectl delete -f kibana-configmap.yaml

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
