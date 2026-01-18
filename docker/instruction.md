# ELK Stack Docker 설치 가이드

Docker Compose를 사용한 컨테이너 기반 ELK Stack 설치 방법을 안내합니다.

## 사전 요구사항

### Docker 설치

```bash
# Docker 버전 확인
docker --version

# Docker Compose 버전 확인
docker compose version
```

### 시스템 설정

```bash
# 가상 메모리 설정 (Linux)
sudo sysctl -w vm.max_map_count=262144

# 영구 설정
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 최소 시스템 요구사항
- **Docker**: 20.10 이상
- **Docker Compose**: 2.0 이상
- **메모리**: 최소 4GB RAM
- **디스크**: 최소 10GB 여유 공간

## docker-compose.yml 구성

프로젝트 디렉토리에 `docker-compose.yml` 파일을 생성합니다:

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - node.name=es-node-1
      - cluster.name=docker-elk-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
      - elasticsearch_logs:/usr/share/elasticsearch/logs
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk_network
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -q 'green\\|yellow'"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    container_name: logstash
    environment:
      - "LS_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.monitoring.enabled=true
      - xpack.monitoring.elasticsearch.hosts=http://elasticsearch:9200
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - logstash_logs:/usr/share/logstash/logs
    ports:
      - "5044:5044"
      - "9600:9600"
    networks:
      - elk_network
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - SERVER_NAME=kibana
      - I18N_LOCALE=ko-KR
    volumes:
      - kibana_logs:/usr/share/kibana/logs
    ports:
      - "5601:5601"
    networks:
      - elk_network
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:5601/api/status | grep -q 'available'"]
      interval: 30s
      timeout: 10s
      retries: 5

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector/otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log:/var/log:ro
      - ./app-logs:/var/log/app:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "13133:13133" # Health check
    networks:
      - elk_network
    depends_on:
      elasticsearch:
        condition: service_healthy

volumes:
  elasticsearch_data:
    driver: local
  elasticsearch_logs:
    driver: local
  logstash_logs:
    driver: local
  kibana_logs:
    driver: local

networks:
  elk_network:
    driver: bridge
```

## 설정 파일 준비

### 디렉토리 구조

```
docker/
├── docker-compose.yml
├── logstash/
│   ├── config/
│   │   └── logstash.yml
│   └── pipeline/
│       └── main.conf
└── otel-collector/
    └── otel-collector-config.yaml
```

### Logstash 설정

`logstash/config/logstash.yml`:

```yaml
http.host: "0.0.0.0"
pipeline.workers: 2
pipeline.batch.size: 125
```

`logstash/pipeline/main.conf`:

```ruby
input {
  # OpenTelemetry Collector에서 데이터 수신
  http {
    port => 5044
    codec => json
  }
}

filter {
  if [docker][container][name] {
    mutate {
      add_field => { "container_name" => "%{[docker][container][name]}" }
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

### OpenTelemetry Collector 설정

`otel-collector/otel-collector-config.yaml`:

```yaml
receivers:
  # Docker 컨테이너 로그 수집
  filelog:
    include:
      - /var/lib/docker/containers/*/*.log
      - /var/log/app/*.log
    start_at: end
    operators:
      - type: json_parser
        id: docker-parser
      - type: move
        from: attributes.log
        to: body
      - type: move
        from: attributes.stream
        to: attributes["log.iostream"]
      - type: move
        from: attributes.time
        to: attributes["log.time"]

  # OTLP 프로토콜 수신 (애플리케이션에서 직접 전송)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Docker 컨테이너 메트릭 수집
  docker_stats:
    endpoint: unix:///var/run/docker.sock
    collection_interval: 30s

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  resource:
    attributes:
      - key: deployment.environment
        value: docker
        action: upsert

  resourcedetection:
    detectors: [env, system, docker]
    timeout: 5s

exporters:
  # Elasticsearch로 직접 전송
  elasticsearch:
    endpoints:
      - http://elasticsearch:9200
    logs_index: otel-logs
    traces_index: otel-traces
    metrics_index: otel-metrics

  # 디버깅용 콘솔 출력
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
      processors: [batch, resource, resourcedetection]
      exporters: [elasticsearch]
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [elasticsearch]
    metrics:
      receivers: [otlp, docker_stats]
      processors: [batch, resource]
      exporters: [elasticsearch]
```

## 컨테이너 실행

### 시작

```bash
# 디렉토리 생성
mkdir -p logstash/config logstash/pipeline otel-collector

# 설정 파일 생성 후 컨테이너 실행
docker compose up -d

# 로그 확인하며 실행
docker compose up
```

### 상태 확인

```bash
# 컨테이너 상태 확인
docker compose ps

# 전체 서비스 헬스 체크
docker compose ps --format "table {{.Name}}\t{{.Status}}"
```

### 중지

```bash
# 컨테이너 중지
docker compose stop

# 컨테이너 중지 및 삭제
docker compose down

# 볼륨까지 삭제 (데이터 삭제)
docker compose down -v
```

### 재시작

```bash
# 전체 재시작
docker compose restart

# 특정 서비스만 재시작
docker compose restart elasticsearch
```

## 볼륨 및 네트워크 설정

### 볼륨 관리

```bash
# 볼륨 목록 확인
docker volume ls | grep elk

# 볼륨 상세 정보
docker volume inspect docker_elasticsearch_data

# 볼륨 백업
docker run --rm -v docker_elasticsearch_data:/data -v $(pwd):/backup alpine tar czf /backup/es-backup.tar.gz -C /data .

# 볼륨 복원
docker run --rm -v docker_elasticsearch_data:/data -v $(pwd):/backup alpine tar xzf /backup/es-backup.tar.gz -C /data
```

### 네트워크 확인

```bash
# 네트워크 목록
docker network ls | grep elk

# 네트워크 상세 정보
docker network inspect docker_elk_network

# 연결된 컨테이너 확인
docker network inspect docker_elk_network --format '{{range .Containers}}{{.Name}} {{end}}'
```

## 로그 확인

### 개별 서비스 로그

```bash
# Elasticsearch 로그
docker compose logs elasticsearch

# Logstash 로그
docker compose logs logstash

# Kibana 로그
docker compose logs kibana

# 실시간 로그 확인
docker compose logs -f

# 특정 서비스 실시간 로그
docker compose logs -f elasticsearch

# 최근 100줄만 확인
docker compose logs --tail=100 elasticsearch
```

### 컨테이너 내부 로그 파일

```bash
# Elasticsearch 내부 로그
docker exec elasticsearch cat /usr/share/elasticsearch/logs/docker-elk-cluster.log

# Logstash 내부 로그
docker exec logstash cat /usr/share/logstash/logs/logstash-plain.log
```

## 동작 확인

### Elasticsearch 확인

```bash
# 클러스터 상태
curl http://localhost:9200/_cluster/health?pretty

# 노드 정보
curl http://localhost:9200/_cat/nodes?v

# 인덱스 목록
curl http://localhost:9200/_cat/indices?v
```

### Kibana 확인

웹 브라우저에서 `http://localhost:5601`에 접속합니다.

### Logstash 확인

```bash
# Logstash API
curl http://localhost:9600/_node/stats?pretty

# 파이프라인 확인
curl http://localhost:9600/_node/pipelines?pretty
```

## 스케일링

### 수평 확장

```bash
# Logstash 인스턴스 추가
docker compose up -d --scale logstash=3
```

### 리소스 제한 설정

`docker-compose.yml`에 리소스 제한 추가:

```yaml
services:
  elasticsearch:
    # ... 기존 설정 ...
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

## 문제 해결

### 일반적인 문제

1. **Elasticsearch 시작 실패 - vm.max_map_count**
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```

2. **메모리 부족**
   - `ES_JAVA_OPTS`에서 힙 크기 조정
   - 시스템 메모리 확인

3. **권한 문제**
   ```bash
   # 볼륨 디렉토리 권한 설정
   sudo chown -R 1000:1000 ./elasticsearch_data
   ```

4. **네트워크 연결 실패**
   ```bash
   # 네트워크 재생성
   docker compose down
   docker network prune
   docker compose up -d
   ```

### 디버깅 명령어

```bash
# 컨테이너 쉘 접속
docker exec -it elasticsearch bash

# 컨테이너 리소스 사용량
docker stats

# 컨테이너 상세 정보
docker inspect elasticsearch
```

## 보안 설정 (프로덕션)

프로덕션 환경에서는 보안 설정을 활성화해야 합니다:

```yaml
elasticsearch:
  environment:
    - xpack.security.enabled=true
    - ELASTIC_PASSWORD=your_secure_password

kibana:
  environment:
    - ELASTICSEARCH_USERNAME=kibana_system
    - ELASTICSEARCH_PASSWORD=your_kibana_password
```

자세한 보안 설정은 [Elastic 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-minimal-setup.html)를 참조하세요.
