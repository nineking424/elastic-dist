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

## 디렉토리 구조

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

## 컨테이너 실행

### 시작

```bash
# docker 디렉토리로 이동
cd docker

# 컨테이너 실행 (백그라운드)
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
```

볼륨 백업 및 복원에 대한 자세한 내용은 [백업/복구 가이드](../docs/backup-recovery-guide.md#docker-볼륨-백업)를 참조하세요.

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

---

## 추가 문서

프로덕션 환경 배포 및 운영에 대한 자세한 내용은 다음 문서를 참조하세요:

| 문서 | 설명 |
|------|------|
| [프로덕션 체크리스트](../docs/production-checklist.md) | 프로덕션 배포 전 확인해야 할 모든 항목 |
| [백업/복구 가이드](../docs/backup-recovery-guide.md) | 데이터 백업 및 복구 절차 |
