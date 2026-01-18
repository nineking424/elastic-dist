# ELK Stack 백업 및 복구 가이드

데이터 백업 및 복구 절차를 안내합니다. 정기적인 백업과 복구 테스트는 데이터 보호의 핵심입니다.

---

## Elasticsearch 스냅샷 설정

Elasticsearch는 스냅샷 API를 통해 인덱스 데이터를 백업합니다.

### 파일 시스템 리포지토리 (로컬/NFS)

#### 1. 공유 저장소 설정

```yaml
# elasticsearch.yml
path.repo: ["/mount/backups", "/mount/long_term_backups"]
```

#### 2. 리포지토리 등록

```bash
# 리포지토리 생성
curl -X PUT "localhost:9200/_snapshot/my_fs_backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/my_fs_backup",
    "compress": true
  }
}'

# 리포지토리 확인
curl -X GET "localhost:9200/_snapshot/my_fs_backup?pretty"
```

### S3 리포지토리 (AWS)

#### 1. S3 플러그인 설치

```bash
bin/elasticsearch-plugin install repository-s3

# AWS 자격 증명 설정
bin/elasticsearch-keystore add s3.client.default.access_key
bin/elasticsearch-keystore add s3.client.default.secret_key
```

#### 2. S3 리포지토리 등록

```bash
curl -X PUT "localhost:9200/_snapshot/s3_backup" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "my-elasticsearch-backups",
    "region": "ap-northeast-2",
    "base_path": "elasticsearch/snapshots",
    "compress": true
  }
}'
```

### GCS 리포지토리 (Google Cloud)

#### 1. GCS 플러그인 설치

```bash
bin/elasticsearch-plugin install repository-gcs

# 서비스 계정 키 설정
bin/elasticsearch-keystore add-file gcs.client.default.credentials_file /path/to/service-account.json
```

#### 2. GCS 리포지토리 등록

```bash
curl -X PUT "localhost:9200/_snapshot/gcs_backup" -H 'Content-Type: application/json' -d'
{
  "type": "gcs",
  "settings": {
    "bucket": "my-elasticsearch-backups",
    "base_path": "elasticsearch/snapshots",
    "compress": true
  }
}'
```

### 수동 스냅샷 생성

```bash
# 전체 클러스터 스냅샷
curl -X PUT "localhost:9200/_snapshot/my_fs_backup/snapshot_$(date +%Y%m%d)?wait_for_completion=true"

# 특정 인덱스만 스냅샷
curl -X PUT "localhost:9200/_snapshot/my_fs_backup/snapshot_logs?wait_for_completion=true" -H 'Content-Type: application/json' -d'
{
  "indices": "logs-*",
  "ignore_unavailable": true,
  "include_global_state": false
}'

# 스냅샷 상태 확인
curl -X GET "localhost:9200/_snapshot/my_fs_backup/snapshot_*?pretty"
```

### SLM (Snapshot Lifecycle Management) 자동화

```bash
# SLM 정책 생성
curl -X PUT "localhost:9200/_slm/policy/daily-snapshots" -H 'Content-Type: application/json' -d'
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "my_fs_backup",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true,
    "include_global_state": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}'

# 정책 즉시 실행
curl -X POST "localhost:9200/_slm/policy/daily-snapshots/_execute"

# SLM 상태 확인
curl -X GET "localhost:9200/_slm/status?pretty"
curl -X GET "localhost:9200/_slm/policy?pretty"
```

---

## Elasticsearch 복구 절차

### 인덱스 복구

```bash
# 스냅샷에서 특정 인덱스 복구
curl -X POST "localhost:9200/_snapshot/my_fs_backup/snapshot_20240115/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "logs-2024.01.*",
  "ignore_unavailable": true,
  "include_global_state": false
}'

# 인덱스 이름 변경하여 복구
curl -X POST "localhost:9200/_snapshot/my_fs_backup/snapshot_20240115/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "logs-*",
  "rename_pattern": "logs-(.+)",
  "rename_replacement": "restored_logs-$1"
}'
```

### 전체 클러스터 복구

```bash
# 클러스터 전체 복구
curl -X POST "localhost:9200/_snapshot/my_fs_backup/snapshot_full/_restore" -H 'Content-Type: application/json' -d'
{
  "include_global_state": true
}'
```

### 복구 모니터링

```bash
# 복구 상태 확인
curl -X GET "localhost:9200/_recovery?pretty"

# 특정 인덱스 복구 상태
curl -X GET "localhost:9200/logs-*/_recovery?pretty"

# 복구 취소
curl -X DELETE "localhost:9200/restored_logs-*"
```

---

## Docker 볼륨 백업

### 수동 볼륨 백업

```bash
# 서비스 중지 (데이터 일관성 보장)
docker compose stop elasticsearch

# 볼륨 백업 (tar 압축)
docker run --rm \
  -v docker_elasticsearch_data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/es-data-$(date +%Y%m%d).tar.gz -C /data .

# 서비스 재시작
docker compose start elasticsearch
```

### 볼륨 복원

```bash
# 서비스 중지
docker compose stop elasticsearch

# 기존 데이터 삭제 (주의!)
docker volume rm docker_elasticsearch_data
docker volume create docker_elasticsearch_data

# 볼륨 복원
docker run --rm \
  -v docker_elasticsearch_data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/es-data-20240115.tar.gz -C /data

# 서비스 재시작
docker compose start elasticsearch
```

### 자동화 스크립트

```bash
#!/bin/bash
# backup-docker-volumes.sh

BACKUP_DIR="/path/to/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# 디렉토리 생성
mkdir -p $BACKUP_DIR

# Elasticsearch 볼륨 백업
echo "Backing up Elasticsearch data..."
docker run --rm \
  -v docker_elasticsearch_data:/data:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/es-data-$DATE.tar.gz -C /data .

# Logstash 데이터 백업 (설정 파일)
echo "Backing up Logstash config..."
tar czf $BACKUP_DIR/logstash-config-$DATE.tar.gz ./logstash/

# 오래된 백업 삭제
echo "Cleaning old backups..."
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $DATE"
```

```bash
# crontab 등록 (매일 새벽 2시)
0 2 * * * /path/to/backup-docker-volumes.sh >> /var/log/elk-backup.log 2>&1
```

---

## Kubernetes PVC 백업

### Velero를 사용한 백업

#### 1. Velero 설치

```bash
# Velero CLI 설치
brew install velero  # macOS
# 또는
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz

# Velero 서버 설치 (AWS 예시)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=ap-northeast-2 \
  --snapshot-location-config region=ap-northeast-2 \
  --secret-file ./credentials-velero
```

#### 2. 백업 생성

```bash
# 네임스페이스 전체 백업
velero backup create elk-backup-$(date +%Y%m%d) \
  --include-namespaces elastic-system \
  --wait

# 특정 리소스만 백업
velero backup create elk-pvc-backup \
  --include-namespaces elastic-system \
  --include-resources persistentvolumeclaims,persistentvolumes \
  --wait

# 백업 상태 확인
velero backup describe elk-backup-20240115
velero backup logs elk-backup-20240115
```

#### 3. 스케줄 백업

```bash
# 매일 백업 스케줄 생성
velero schedule create elk-daily \
  --schedule="0 2 * * *" \
  --include-namespaces elastic-system \
  --ttl 720h  # 30일 보관

# 스케줄 확인
velero schedule get
```

#### 4. 복구

```bash
# 백업에서 복구
velero restore create --from-backup elk-backup-20240115

# 복구 상태 확인
velero restore describe elk-backup-20240115-restore
```

### VolumeSnapshot 사용

#### 1. VolumeSnapshotClass 생성

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: elasticsearch-snapshot-class
driver: pd.csi.storage.gke.io  # 클라우드 프로바이더에 따라 변경
deletionPolicy: Retain
```

#### 2. VolumeSnapshot 생성

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: elasticsearch-snapshot-20240115
  namespace: elastic-system
spec:
  volumeSnapshotClassName: elasticsearch-snapshot-class
  source:
    persistentVolumeClaimName: data-elasticsearch-0
```

```bash
kubectl apply -f volume-snapshot.yaml

# 스냅샷 상태 확인
kubectl get volumesnapshot -n elastic-system
```

#### 3. 스냅샷에서 PVC 복구

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-restored
  namespace: elastic-system
spec:
  storageClassName: elasticsearch-storage
  dataSource:
    name: elasticsearch-snapshot-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```

---

## 백업 검증 및 테스트

### 복구 테스트 절차

1. **테스트 환경 준비**
   - 프로덕션과 격리된 테스트 클러스터 준비
   - 동일한 버전의 ELK Stack 설치

2. **백업 복원**
   ```bash
   # 테스트 환경에서 복원
   curl -X POST "test-cluster:9200/_snapshot/backup_repo/latest/_restore"
   ```

3. **데이터 검증**
   ```bash
   # 문서 수 확인
   curl -X GET "localhost:9200/_cat/count?v"

   # 인덱스 상태 확인
   curl -X GET "localhost:9200/_cat/indices?v"

   # 샘플 데이터 검색
   curl -X GET "localhost:9200/logs-*/_search?size=10&pretty"
   ```

4. **무결성 검사**
   ```bash
   # 샤드 상태 확인
   curl -X GET "localhost:9200/_cat/shards?v"

   # 클러스터 상태 확인
   curl -X GET "localhost:9200/_cluster/health?pretty"
   ```

### 백업 무결성 검사 스크립트

```bash
#!/bin/bash
# verify-backup.sh

SNAPSHOT_REPO="my_fs_backup"
SNAPSHOT_NAME=$1

if [ -z "$SNAPSHOT_NAME" ]; then
  echo "Usage: $0 <snapshot_name>"
  exit 1
fi

echo "Verifying snapshot: $SNAPSHOT_NAME"

# 스냅샷 상태 확인
STATUS=$(curl -s "localhost:9200/_snapshot/$SNAPSHOT_REPO/$SNAPSHOT_NAME" | jq -r '.snapshots[0].state')

if [ "$STATUS" == "SUCCESS" ]; then
  echo "✅ Snapshot status: SUCCESS"

  # 포함된 인덱스 확인
  INDICES=$(curl -s "localhost:9200/_snapshot/$SNAPSHOT_REPO/$SNAPSHOT_NAME" | jq -r '.snapshots[0].indices[]')
  echo "📦 Included indices:"
  echo "$INDICES"

  # 샤드 통계 확인
  SHARDS=$(curl -s "localhost:9200/_snapshot/$SNAPSHOT_REPO/$SNAPSHOT_NAME" | jq '.snapshots[0].shards')
  echo "🔢 Shard statistics:"
  echo "$SHARDS"
else
  echo "❌ Snapshot status: $STATUS"
  exit 1
fi
```

---

## 재해 복구 (DR) 계획

### 시나리오별 대응

#### 시나리오 1: 단일 노드 장애

1. Kubernetes가 자동으로 Pod 재시작
2. StatefulSet이 PVC 재연결
3. 클러스터가 샤드 재할당

```bash
# 클러스터 상태 모니터링
kubectl get pods -n elastic-system -w

# 샤드 재할당 상태 확인
curl -X GET "localhost:9200/_cat/allocation?v"
```

#### 시나리오 2: 전체 클러스터 장애

1. 새 클러스터 프로비저닝
2. 최신 스냅샷에서 복구
3. 애플리케이션 재연결

```bash
# 1. 새 클러스터 배포
kubectl apply -f k8s/

# 2. 스냅샷 리포지토리 등록
curl -X PUT "localhost:9200/_snapshot/disaster_recovery" -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "my-elasticsearch-backups",
    "region": "ap-northeast-2"
  }
}'

# 3. 최신 스냅샷 확인 및 복구
curl -X GET "localhost:9200/_snapshot/disaster_recovery/_all?pretty"
curl -X POST "localhost:9200/_snapshot/disaster_recovery/latest/_restore"
```

#### 시나리오 3: 데이터 센터 장애

1. Cross-Cluster Replication (CCR) 활성화된 보조 클러스터로 전환
2. DNS 변경으로 트래픽 전환
3. 데이터 동기화 확인

### RTO/RPO 목표 설정

| 항목 | 목표 | 달성 방법 |
|------|------|----------|
| **RPO** (복구 시점 목표) | 1시간 | 시간별 스냅샷 |
| **RTO** (복구 시간 목표) | 4시간 | 자동화된 복구 스크립트 |

### 복구 체크리스트

- [ ] 스냅샷 리포지토리 접근 가능 확인
- [ ] 최신 스냅샷 무결성 검증
- [ ] 새 클러스터 프로비저닝
- [ ] 스냅샷 복구 실행
- [ ] 클러스터 상태 green 확인
- [ ] 데이터 무결성 검증
- [ ] 애플리케이션 연결 테스트
- [ ] 모니터링/알림 재구성

---

## 참고 문서

- [Elasticsearch Snapshot and Restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html)
- [Snapshot Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-lifecycle-management.html)
- [Velero Documentation](https://velero.io/docs/)
- [Kubernetes VolumeSnapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
