# ELK Stack 프로덕션 체크리스트

프로덕션 환경에 ELK Stack을 배포하기 전에 확인해야 할 항목들입니다.

---

## 배포 전 확인 사항

### 시스템 요구사항

- [ ] **CPU**: 노드당 최소 4코어 이상
- [ ] **메모리**: 노드당 최소 8GB RAM (Elasticsearch 힙에 절반 할당 권장)
- [ ] **디스크**: SSD 사용, 최소 50GB 이상 여유 공간
- [ ] **네트워크**: 노드 간 저지연 연결 (동일 데이터센터/가용영역 권장)

### OS 설정

```bash
# 가상 메모리 설정
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# 파일 디스크립터 제한
echo "elasticsearch soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "elasticsearch hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# 스왑 비활성화 (권장)
sudo swapoff -a
```

### 버전 호환성

- [ ] Elasticsearch, Logstash, Kibana 버전 일치 확인
- [ ] OpenTelemetry Collector 버전 호환성 확인
- [ ] Kubernetes 버전 1.24 이상 (K8s 배포 시)
- [ ] Docker 20.10 이상, Docker Compose 2.0 이상 (Docker 배포 시)

---

## 보안 설정 체크리스트

### 인증 및 권한

- [ ] **X-Pack Security 활성화**
  ```yaml
  xpack.security.enabled: true
  xpack.security.enrollment.enabled: true
  ```

- [ ] **강력한 비밀번호 설정**
  ```bash
  # 내장 사용자 비밀번호 설정
  bin/elasticsearch-setup-passwords auto
  # 또는 interactive 모드
  bin/elasticsearch-setup-passwords interactive
  ```

- [ ] **역할 기반 접근 제어 (RBAC) 구성**
  - 관리자, 개발자, 읽기 전용 등 역할 분리
  - 최소 권한 원칙 적용

- [ ] **API 키 관리**
  - 서비스 계정용 API 키 발급
  - 키 순환 정책 수립

### TLS/HTTPS 설정

- [ ] **노드 간 통신 암호화 (Transport TLS)**
  ```yaml
  xpack.security.transport.ssl.enabled: true
  xpack.security.transport.ssl.verification_mode: certificate
  xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
  xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
  ```

- [ ] **클라이언트 통신 암호화 (HTTP TLS)**
  ```yaml
  xpack.security.http.ssl.enabled: true
  xpack.security.http.ssl.keystore.path: http.p12
  ```

- [ ] **인증서 생성**
  ```bash
  # CA 생성
  bin/elasticsearch-certutil ca

  # 노드 인증서 생성
  bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

  # HTTP 인증서 생성
  bin/elasticsearch-certutil http
  ```

- [ ] **인증서 만료일 모니터링**

### 네트워크 보안

- [ ] **바인딩 주소 제한**
  - 외부 노출이 불필요한 경우 `127.0.0.1` 또는 내부 IP만 바인딩
  - 프로덕션: 특정 네트워크 인터페이스만 노출

- [ ] **방화벽 규칙 설정**
  - 필요한 포트만 허용 (9200, 9300, 5601, 5044 등)
  - 신뢰할 수 있는 IP 범위만 접근 허용

- [ ] **리버스 프록시 사용**
  - Nginx, HAProxy 등을 통한 접근 제어
  - SSL 종료 처리

### 컨테이너 보안 (Docker/Kubernetes)

- [ ] **Non-root 사용자로 실행**
  ```yaml
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  ```

- [ ] **읽기 전용 파일시스템** (가능한 경우)
  ```yaml
  securityContext:
    readOnlyRootFilesystem: true
  ```

- [ ] **Privileged 모드 비활성화**
  - init container에서 필요시 특정 capability만 부여
  ```yaml
  securityContext:
    capabilities:
      add: ["SYS_RESOURCE"]
  ```

- [ ] **NetworkPolicy 적용** (Kubernetes)
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: elasticsearch-network-policy
    namespace: elastic-system
  spec:
    podSelector:
      matchLabels:
        app: elasticsearch
    policyTypes:
      - Ingress
      - Egress
    ingress:
      - from:
          - podSelector:
              matchLabels:
                app: kibana
          - podSelector:
              matchLabels:
                app: logstash
        ports:
          - protocol: TCP
            port: 9200
    egress:
      - to:
          - podSelector:
              matchLabels:
                app: elasticsearch
        ports:
          - protocol: TCP
            port: 9300
  ```

---

## 성능 및 용량 계획

### 클러스터 사이징

- [ ] **노드 역할 분리** (대규모 클러스터)
  - Master-eligible 노드: 3개 이상 (홀수)
  - Data 노드: 워크로드에 따라 조정
  - Coordinating 노드: 쿼리 부하 분산용

- [ ] **샤드 계획**
  - 샤드당 크기: 10GB ~ 50GB 권장
  - 샤드 수 = 예상 데이터 크기 / 샤드당 목표 크기
  - 노드당 샤드 수: 1000개 미만 권장

### JVM 메모리 설정

- [ ] **힙 크기 설정**
  ```yaml
  ES_JAVA_OPTS: "-Xms4g -Xmx4g"
  ```
  - 시스템 메모리의 50% 이하
  - 최대 31GB (Compressed OOPs 활용)
  - Xms와 Xmx 동일하게 설정

- [ ] **GC 로그 활성화**
  ```yaml
  ES_JAVA_OPTS: "-Xlog:gc*:file=/var/log/elasticsearch/gc.log:time,tags:filecount=10,filesize=50m"
  ```

### ILM (Index Lifecycle Management) 정책

- [ ] **인덱스 수명주기 정책 설정**
  ```json
  {
    "policy": {
      "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": {
              "max_age": "1d",
              "max_primary_shard_size": "50gb"
            }
          }
        },
        "warm": {
          "min_age": "7d",
          "actions": {
            "shrink": { "number_of_shards": 1 },
            "forcemerge": { "max_num_segments": 1 }
          }
        },
        "cold": {
          "min_age": "30d",
          "actions": {
            "freeze": {}
          }
        },
        "delete": {
          "min_age": "90d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }
  ```

### 리소스 제한

- [ ] **Kubernetes 리소스 설정**
  ```yaml
  resources:
    requests:
      memory: "4Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"
      cpu: "2000m"
  ```

- [ ] **Docker 리소스 설정**
  ```yaml
  deploy:
    resources:
      limits:
        memory: 4G
        cpus: '2.0'
      reservations:
        memory: 2G
        cpus: '1.0'
  ```

---

## 고가용성 체크리스트

### 클러스터 구성

- [ ] **Master 노드 최소 3개** (split-brain 방지)
  ```yaml
  cluster.initial_master_nodes:
    - elasticsearch-0
    - elasticsearch-1
    - elasticsearch-2
  ```

- [ ] **레플리카 샤드 설정**
  ```json
  {
    "index.number_of_replicas": 1
  }
  ```

- [ ] **클러스터 라우팅 할당**
  ```yaml
  cluster.routing.allocation.enable: all
  cluster.routing.allocation.cluster_concurrent_rebalance: 2
  ```

### 장애 대응

- [ ] **Pod Anti-Affinity** (Kubernetes)
  ```yaml
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - elasticsearch
          topologyKey: "kubernetes.io/hostname"
  ```

- [ ] **PodDisruptionBudget 설정** (Kubernetes)
  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: elasticsearch-pdb
    namespace: elastic-system
  spec:
    minAvailable: 2
    selector:
      matchLabels:
        app: elasticsearch
  ```

- [ ] **재시작 정책 설정** (Docker)
  ```yaml
  restart: unless-stopped
  ```

---

## 모니터링 체크리스트

### 메트릭 수집

- [ ] **클러스터 상태 모니터링**
  - `_cluster/health` API 주기적 확인
  - 클러스터 상태: green/yellow/red 알림

- [ ] **노드 통계 모니터링**
  - CPU, 메모리, 디스크 사용률
  - JVM 힙 사용률, GC 빈도

- [ ] **인덱스 통계**
  - 문서 수, 저장소 크기
  - 인덱싱/검색 성능

### 알림 설정

- [ ] **임계치 기반 알림**
  - 디스크 사용률 > 80%
  - JVM 힙 사용률 > 85%
  - 클러스터 상태 yellow/red
  - 노드 이탈

- [ ] **로그 레벨 설정**
  ```yaml
  logger.level: INFO
  logger.org.elasticsearch.transport: WARN
  logger.org.elasticsearch.cluster: INFO
  ```

---

## 배포 방식별 추가 체크리스트

### Docker 배포

- [ ] **로그 로테이션 설정**
  ```yaml
  logging:
    driver: "json-file"
    options:
      max-size: "100m"
      max-file: "5"
  ```

- [ ] **볼륨 마운트 권한**
  ```bash
  sudo chown -R 1000:1000 ./elasticsearch_data
  ```

- [ ] **네트워크 격리**
  - 전용 Docker 네트워크 사용
  - 불필요한 포트 노출 제거

### Kubernetes 배포

- [ ] **StorageClass 설정**
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: elasticsearch-storage
  provisioner: kubernetes.io/gce-pd
  parameters:
    type: pd-ssd
  reclaimPolicy: Retain
  volumeBindingMode: WaitForFirstConsumer
  ```

- [ ] **Ingress TLS 설정**
  ```yaml
  spec:
    tls:
      - hosts:
          - kibana.example.com
        secretName: kibana-tls-secret
  ```

- [ ] **RBAC 설정**
  - ServiceAccount 생성
  - 최소 권한 Role/ClusterRole 적용

- [ ] **Secret 관리**
  - 외부 Secret 관리 도구 연동 (Vault, AWS Secrets Manager 등)
  - 또는 Kubernetes Secrets 암호화

---

## 배포 후 검증

### 기능 검증

- [ ] 클러스터 상태 green 확인
- [ ] 모든 노드 정상 조인 확인
- [ ] 인덱스 생성/검색 테스트
- [ ] Kibana 접속 및 로그인 확인
- [ ] Logstash 파이프라인 동작 확인

### 보안 검증

- [ ] TLS 연결 테스트
- [ ] 인증 없는 접근 차단 확인
- [ ] RBAC 권한 테스트

### 성능 검증

- [ ] 인덱싱 처리량 측정
- [ ] 쿼리 응답 시간 측정
- [ ] 리소스 사용률 모니터링

---

## 참고 문서

- [Elasticsearch 보안 설정 가이드](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)
- [Elasticsearch 프로덕션 배포 가이드](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)
- [Kibana 프로덕션 설정](https://www.elastic.co/guide/en/kibana/current/production.html)
- [Logstash 프로덕션 설정](https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html)
