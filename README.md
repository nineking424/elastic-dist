# ELK Stack Distribution

ELK(Elasticsearch, Logstash, Kibana) 스택과 OpenTelemetry Collector를 다양한 환경에 배포하기 위한 설정 및 가이드 모음입니다.

## 프로젝트 구조

```
elastic-dist/
├── prebuilt/                      # 바이너리 직접 설치
│   └── instruction.md
├── docker/                        # Docker Compose 기반 설치
│   ├── docker-compose.yml
│   ├── logstash/
│   │   ├── config/
│   │   │   └── logstash.yml
│   │   └── pipeline/
│   │       └── main.conf
│   ├── otel-collector/
│   │   └── otel-collector-config.yaml
│   └── instruction.md
├── k8s/                           # Kubernetes 배포
│   ├── namespace.yaml
│   ├── elasticsearch-configmap.yaml
│   ├── elasticsearch-statefulset.yaml
│   ├── logstash-configmap.yaml
│   ├── logstash-deployment.yaml
│   ├── kibana-configmap.yaml
│   ├── kibana-deployment.yaml
│   ├── otel-collector-daemonset.yaml
│   ├── elk-ingress.yaml
│   └── instruction.md
├── REVIEW.md                      # 프로젝트 검토 보고서
└── README.md
```

## 설치 방법

### 1. 바이너리 설치 (prebuilt)

공식 바이너리를 직접 다운로드하여 설치하는 방법입니다. 개발 환경이나 단일 서버에 적합합니다.

```bash
cd prebuilt
# instruction.md 참조
```

**주요 내용:**
- Elasticsearch, Logstash, Kibana 수동 설치
- OpenTelemetry Collector Contrib 설치
- systemd 서비스 등록

### 2. Docker Compose 설치 (docker)

컨테이너 기반으로 빠르게 ELK 스택을 실행할 수 있습니다.

```bash
cd docker

# vm.max_map_count 설정 (Linux)
sudo sysctl -w vm.max_map_count=262144

# 실행
docker compose up -d
```

**주요 기능:**
- 단일 명령어로 전체 스택 실행
- 볼륨을 통한 데이터 영속성
- OpenTelemetry Collector로 컨테이너 로그 수집
- 모든 서비스에 재시작 정책 적용 (`unless-stopped`)
- 전체 서비스 헬스체크 구성

### 3. Kubernetes 배포 (k8s)

프로덕션 환경에 적합한 Kubernetes 기반 배포입니다.

```bash
cd k8s

# 전체 리소스 배포
kubectl apply -f namespace.yaml
kubectl apply -f elasticsearch-configmap.yaml
kubectl apply -f logstash-configmap.yaml
kubectl apply -f kibana-configmap.yaml
kubectl apply -f elasticsearch-statefulset.yaml
kubectl apply -f logstash-deployment.yaml
kubectl apply -f kibana-deployment.yaml
kubectl apply -f otel-collector-daemonset.yaml

# 또는 한 번에 배포
kubectl apply -f .
```

**주요 기능:**
- StatefulSet 기반 Elasticsearch 클러스터 (3 replicas)
- DaemonSet 기반 OpenTelemetry Collector
- ConfigMap을 통한 설정 관리
- RBAC 설정 포함
- Pod Anti-Affinity로 고가용성 보장
- readinessProbe/livenessProbe 완전 구성

## 구성 요소

| 구성 요소 | 버전 | 설명 |
|-----------|------|------|
| Elasticsearch | 8.12.0 | 분산 검색 및 분석 엔진 |
| Logstash | 8.12.0 | 데이터 처리 파이프라인 |
| Kibana | 8.12.0 | 데이터 시각화 대시보드 |
| OpenTelemetry Collector | 0.96.0 | 로그/메트릭/트레이스 수집기 |

## 시스템 요구사항

### 최소 사양
- **CPU**: 2 코어
- **메모리**: 4GB RAM
- **디스크**: 10GB 여유 공간

### 권장 사양 (프로덕션)
- **CPU**: 4 코어 이상
- **메모리**: 8GB RAM 이상
- **디스크**: SSD 50GB 이상

## 포트 정보

| 서비스 | 포트 | 프로토콜 |
|--------|------|----------|
| Elasticsearch HTTP | 9200 | HTTP |
| Elasticsearch Transport | 9300 | TCP |
| Logstash Beats/HTTP | 5044 | TCP |
| Logstash API | 9600 | HTTP |
| Kibana | 5601 | HTTP |
| OTLP gRPC | 4317 | gRPC |
| OTLP HTTP | 4318 | HTTP |
| OTel Health Check | 13133 | HTTP |

## 빠른 시작

### Docker Compose (가장 빠른 방법)

```bash
# 저장소 클론
git clone https://github.com/nineking424/elastic-dist.git
cd elastic-dist/docker

# vm.max_map_count 설정 (Linux)
sudo sysctl -w vm.max_map_count=262144

# 실행
docker compose up -d

# 상태 확인
docker compose ps
```

### Kubernetes

```bash
# 저장소 클론
git clone https://github.com/nineking424/elastic-dist.git
cd elastic-dist/k8s

# 배포
kubectl apply -f .

# 상태 확인
kubectl get pods -n elastic-system
```

### 접속 정보

- **Kibana**: http://localhost:5601
- **Elasticsearch**: http://localhost:9200

## 데이터 흐름

```
┌─────────────┐     ┌──────────────────────┐     ┌───────────────┐
│ Application │────▶│ OpenTelemetry        │────▶│ Elasticsearch │
│ Logs/Traces │     │ Collector            │     │               │
└─────────────┘     └──────────────────────┘     └───────┬───────┘
                              │                          │
                              │                          ▼
                              │                    ┌───────────┐
                              └───────────────────▶│  Kibana   │
                                (optional)         └───────────┘
                                Logstash
```

## 프로덕션 배포 시 주의사항

현재 설정은 개발/테스트 환경에 최적화되어 있습니다. 프로덕션 배포 시 다음 사항을 고려하세요:

- **보안**: `xpack.security.enabled=true` 설정 및 TLS 인증서 적용
- **비밀번호**: 강력한 비밀번호 설정 및 Secret 관리
- **네트워크**: NetworkPolicy 적용 (Kubernetes)
- **리소스**: 적절한 CPU/메모리 제한 설정

자세한 내용은 [REVIEW.md](REVIEW.md)를 참조하세요.

## 참고 문서

- [Elasticsearch 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Logstash 공식 문서](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Kibana 공식 문서](https://www.elastic.co/guide/en/kibana/current/index.html)
- [OpenTelemetry Collector 문서](https://opentelemetry.io/docs/collector/)

## 라이선스

이 프로젝트의 설정 파일들은 자유롭게 사용할 수 있습니다. ELK Stack 및 OpenTelemetry는 각각의 라이선스를 따릅니다.
