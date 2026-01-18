# ELK Stack 바이너리 설치 가이드

공식 바이너리를 직접 다운로드하여 설치하는 방법을 안내합니다.

## 사전 요구사항

### 시스템 요구사항
- **운영체제**: Linux, macOS, 또는 Windows
- **메모리**: 최소 4GB RAM (권장 8GB 이상)
- **디스크**: 최소 10GB 여유 공간

### Java 설치
Elasticsearch 8.x는 번들된 OpenJDK를 포함하므로 별도 설치가 필요하지 않습니다.
Logstash의 경우 Java 11 이상이 필요합니다.

```bash
# Java 버전 확인
java -version

# Ubuntu/Debian에서 OpenJDK 설치
sudo apt update
sudo apt install openjdk-17-jdk

# macOS에서 Homebrew로 설치
brew install openjdk@17
```

## Elasticsearch 설치

### 1. 다운로드 및 설치

```bash
# 버전 설정
ELK_VERSION=8.12.0

# Elasticsearch 다운로드
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-${ELK_VERSION}-linux-x86_64.tar.gz

# 압축 해제
tar -xzf elasticsearch-${ELK_VERSION}-linux-x86_64.tar.gz
cd elasticsearch-${ELK_VERSION}
```

### 2. 기본 설정

`config/elasticsearch.yml` 파일을 편집합니다:

```yaml
# 클러스터 이름
cluster.name: my-elk-cluster

# 노드 이름
node.name: node-1

# 네트워크 설정
network.host: 0.0.0.0
http.port: 9200

# 단일 노드 클러스터 설정 (개발 환경)
discovery.type: single-node

# 보안 설정 (개발 환경에서 비활성화)
xpack.security.enabled: false
```

### 3. JVM 힙 메모리 설정

`config/jvm.options.d/heap.options` 파일을 생성합니다:

```
-Xms2g
-Xmx2g
```

### 4. 시작 및 중지

```bash
# 포그라운드에서 시작
./bin/elasticsearch

# 백그라운드에서 시작
./bin/elasticsearch -d -p pid

# 중지 (백그라운드 실행 시)
pkill -F pid
```

## Logstash 설치

### 1. 다운로드 및 설치

```bash
# Logstash 다운로드
curl -O https://artifacts.elastic.co/downloads/logstash/logstash-${ELK_VERSION}-linux-x86_64.tar.gz

# 압축 해제
tar -xzf logstash-${ELK_VERSION}-linux-x86_64.tar.gz
cd logstash-${ELK_VERSION}
```

### 2. 파이프라인 설정

`config/logstash.yml` 파일을 편집합니다:

```yaml
# 파이프라인 설정
pipeline.workers: 2
pipeline.batch.size: 125

# 모니터링
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["http://localhost:9200"]
```

`config/pipelines.yml` 파일을 편집합니다:

```yaml
- pipeline.id: main
  path.config: "/path/to/logstash/config/conf.d/*.conf"
```

### 3. 샘플 파이프라인 구성

`config/conf.d/sample.conf` 파일을 생성합니다:

```ruby
input {
  beats {
    port => 5044
  }
  stdin { }
}

filter {
  # 로그 파싱 규칙
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

### 4. 시작 및 중지

```bash
# 설정 파일 테스트
./bin/logstash --config.test_and_exit -f config/conf.d/sample.conf

# 시작
./bin/logstash -f config/conf.d/sample.conf

# 백그라운드 실행
nohup ./bin/logstash -f config/conf.d/sample.conf &
```

## Kibana 설치

### 1. 다운로드 및 설치

```bash
# Kibana 다운로드
curl -O https://artifacts.elastic.co/downloads/kibana/kibana-${ELK_VERSION}-linux-x86_64.tar.gz

# 압축 해제
tar -xzf kibana-${ELK_VERSION}-linux-x86_64.tar.gz
cd kibana-${ELK_VERSION}
```

### 2. 기본 설정

`config/kibana.yml` 파일을 편집합니다:

```yaml
# 서버 설정
server.port: 5601
server.host: "0.0.0.0"
server.name: "my-kibana"

# Elasticsearch 연결
elasticsearch.hosts: ["http://localhost:9200"]

# 한국어 설정
i18n.locale: "ko-KR"
```

### 3. 시작 및 중지

```bash
# 포그라운드에서 시작
./bin/kibana

# 백그라운드에서 시작
nohup ./bin/kibana &

# 중지
pkill -f kibana
```

## OpenTelemetry Collector 설치 (선택사항)

OpenTelemetry Collector는 로그, 메트릭, 트레이스를 수집하여 Elasticsearch로 전송할 수 있는 벤더 중립적인 수집기입니다.

### 1. 다운로드 및 설치

```bash
# 버전 설정
OTEL_VERSION=0.96.0

# OpenTelemetry Collector Contrib 다운로드
curl -OL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTEL_VERSION}/otelcol-contrib_${OTEL_VERSION}_linux_amd64.tar.gz

# 압축 해제
mkdir otelcol-contrib
tar -xzf otelcol-contrib_${OTEL_VERSION}_linux_amd64.tar.gz -C otelcol-contrib
cd otelcol-contrib
```

### 2. 기본 설정

`otel-config.yaml` 파일을 생성합니다:

```yaml
receivers:
  # 파일 로그 수집
  filelog:
    include:
      - /var/log/*.log
    start_at: end
    operators:
      - type: regex_parser
        regex: '^(?P<time>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d+Z?) (?P<level>\w+) (?P<message>.*)$'
        timestamp:
          parse_from: attributes.time
          layout: '%Y-%m-%dT%H:%M:%S.%LZ'
        severity:
          parse_from: attributes.level

  # OTLP 프로토콜로 데이터 수신
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  resource:
    attributes:
      - key: host.name
        from_attribute: host.name
        action: upsert

exporters:
  # Elasticsearch로 전송
  elasticsearch:
    endpoints:
      - http://localhost:9200
    logs_index: otel-logs
    traces_index: otel-traces
    metrics_index: otel-metrics

  # 디버깅용 콘솔 출력
  debug:
    verbosity: detailed

service:
  pipelines:
    logs:
      receivers: [filelog, otlp]
      processors: [batch, resource]
      exporters: [elasticsearch, debug]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
```

### 3. 시작

```bash
# 설정 파일 검증
./otelcol-contrib validate --config=otel-config.yaml

# 포그라운드에서 시작
./otelcol-contrib --config=otel-config.yaml

# 백그라운드 실행
nohup ./otelcol-contrib --config=otel-config.yaml &
```

### 4. 동작 확인

```bash
# 헬스 체크
curl http://localhost:13133/health

# zpages 확인 (디버깅)
curl http://localhost:55679/debug/tracez
```

## 동작 확인

### Elasticsearch 확인

```bash
# 클러스터 상태 확인
curl -X GET "localhost:9200/_cluster/health?pretty"

# 노드 정보 확인
curl -X GET "localhost:9200/_cat/nodes?v"

# 인덱스 목록 확인
curl -X GET "localhost:9200/_cat/indices?v"
```

### Kibana 확인

웹 브라우저에서 `http://localhost:5601`에 접속합니다.

### Logstash 확인

```bash
# Logstash 프로세스 확인
ps aux | grep logstash

# Logstash API 확인
curl -X GET "localhost:9600/_node/stats?pretty"
```

## 서비스 등록 (systemd)

### Elasticsearch 서비스

`/etc/systemd/system/elasticsearch.service` 파일을 생성합니다:

```ini
[Unit]
Description=Elasticsearch
Documentation=https://www.elastic.co
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
User=elasticsearch
Group=elasticsearch
ExecStart=/path/to/elasticsearch/bin/elasticsearch
Restart=on-failure
LimitNOFILE=65535
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

### 서비스 활성화

```bash
# 데몬 리로드
sudo systemctl daemon-reload

# 서비스 활성화 및 시작
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# 상태 확인
sudo systemctl status elasticsearch
```

## 문제 해결

### 일반적인 문제

1. **메모리 부족 오류**
   - JVM 힙 크기를 시스템 메모리의 50% 이하로 설정

2. **파일 디스크립터 제한**
   ```bash
   ulimit -n 65535
   ```

3. **가상 메모리 제한**
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```

4. **권한 문제**
   - root 사용자로 Elasticsearch를 실행하지 마세요
   - 전용 사용자 계정을 생성하여 실행하세요

### 로그 확인

```bash
# Elasticsearch 로그
tail -f elasticsearch-${ELK_VERSION}/logs/my-elk-cluster.log

# Logstash 로그
tail -f logstash-${ELK_VERSION}/logs/logstash-plain.log

# Kibana 로그
tail -f kibana-${ELK_VERSION}/logs/kibana.log
```
