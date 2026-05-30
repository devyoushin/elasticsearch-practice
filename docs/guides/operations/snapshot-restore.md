# 스냅샷 & 복구

## 개요

Elasticsearch 스냅샷(Snapshot)은 인덱스 데이터를 외부 저장소(S3 등)에 백업하는 기능이다. SLM(Snapshot Lifecycle Management)으로 자동화하고, 장애 시 특정 시점으로 복구한다.

| 항목 | 값 |
|------|-----|
| 관련 API | `PUT /_snapshot/{repo}`, `PUT /_slm/policy/{name}`, `POST /_snapshot/{repo}/{name}/_restore` |
| 적용 환경 | Elasticsearch 8.x, ECK v2.x |
| 저장소 | AWS S3 |

---

## 설명

### 스냅샷 저장소 등록 (S3)

ECK 환경에서는 IRSA(IAM Roles for Service Accounts)를 활용하여 자격증명 없이 S3에 접근한다.

```bash
# S3 스냅샷 저장소 등록
curl -X PUT "${ES_HOST}/_snapshot/my-s3-repo" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "my-elasticsearch-snapshots",
      "region": "ap-northeast-2",
      "base_path": "my-cluster/snapshots",
      "server_side_encryption": true
    }
  }'

# 저장소 검증
curl -X POST "${ES_HOST}/_snapshot/my-s3-repo/_verify" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 수동 스냅샷

```bash
# 전체 스냅샷 생성
curl -X PUT "${ES_HOST}/_snapshot/my-s3-repo/snapshot-$(date +%Y%m%d-%H%M%S)?wait_for_completion=true" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "indices": "*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'

# 특정 인덱스만 스냅샷
curl -X PUT "${ES_HOST}/_snapshot/my-s3-repo/snapshot-my-index-20240115" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "indices": "my-index-v1,my-index-v2",
    "ignore_unavailable": false,
    "include_global_state": false
  }'

# 스냅샷 목록 확인
curl -X GET "${ES_HOST}/_snapshot/my-s3-repo/*?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 스냅샷 상태 확인
curl -X GET "${ES_HOST}/_snapshot/my-s3-repo/snapshot-20240115/_status?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### SLM (Snapshot Lifecycle Management) 자동화

```bash
# SLM 정책 생성
curl -X PUT "${ES_HOST}/_slm/policy/daily-snapshots" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "schedule": "0 30 1 * * ?",
    "name":     "<daily-snap-{now/d}>",
    "repository": "my-s3-repo",
    "config": {
      "indices": "*",
      "ignore_unavailable": true,
      "include_global_state": false
    },
    "retention": {
      "expire_after": "30d",
      "min_count": 5,
      "max_count": 50
    }
  }'

# SLM 정책 확인
curl -X GET "${ES_HOST}/_slm/policy/daily-snapshots?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# SLM 즉시 실행
curl -X POST "${ES_HOST}/_slm/policy/daily-snapshots/_execute" \
  -u "elastic:${ES_PASSWORD}" -k

# SLM 통계 확인
curl -X GET "${ES_HOST}/_slm/stats?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 스냅샷 복구

```bash
# 인덱스 복구 (기존 인덱스가 있으면 삭제 후 복구)
curl -X POST "${ES_HOST}/_snapshot/my-s3-repo/snapshot-20240115/_restore" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "indices": "my-index-v1",
    "ignore_unavailable": false,
    "include_global_state": false,
    "rename_pattern": "my-index-(.+)",
    "rename_replacement": "restored-my-index-$1",
    "include_aliases": false
  }'

# 복구 진행 상태 확인
curl -X GET "${ES_HOST}/_cat/recovery?v&active_only=true" \
  -u "elastic:${ES_PASSWORD}" -k

# 복구 완료 확인
curl -X GET "${ES_HOST}/_cat/indices/restored-*?v" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| 스냅샷 저장소 등록 실패 | S3 권한 없음, 버킷 미존재 | IRSA 역할 정책, 버킷 확인 |
| 스냅샷 생성 `PARTIAL` 상태 | 일부 샤드 UNASSIGNED 상태 | 클러스터 health 먼저 복구 |
| 복구 후 인덱스 RED | 복구 중 노드 부족 | 복구 중 `_cat/recovery` 모니터링 |

### S3 접근 권한 오류

**증상**: `repository verification failed` 또는 `AccessDenied`

**원인**: IRSA 설정 누락 또는 S3 버킷 정책 오류

**해결책**:
```bash
# Elasticsearch pod의 AWS 자격증명 확인
kubectl exec -it -n elasticsearch my-cluster-es-data-hot-0 \
  -- curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# S3 접근 테스트
kubectl exec -it -n elasticsearch my-cluster-es-data-hot-0 \
  -- curl "https://my-elasticsearch-snapshots.s3.ap-northeast-2.amazonaws.com/"

# ECK에서 IRSA 설정 (ServiceAccount에 annotation 추가)
kubectl annotate serviceaccount -n elasticsearch my-cluster \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT_ID:role/elasticsearch-snapshot-role
```

---

## 모니터링 & 검증

```bash
# 스냅샷 목록 및 상태
curl -X GET "${ES_HOST}/_snapshot/my-s3-repo/*?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# SLM 최근 실행 결과
curl -X GET "${ES_HOST}/_slm/policy?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 복구 중인 샤드 확인
curl -X GET "${ES_HOST}/_cat/recovery?v&active_only=true&h=index,shard,stage,percent,source_node,target_node" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## TIP

- 스냅샷은 증분(incremental) 방식 → 첫 번째 스냅샷 이후부터 빠름
- `include_global_state: false` 권장 (인덱스 템플릿, ILM 정책 등 글로벌 설정 제외)
- 복구 시 `rename_pattern`/`rename_replacement`로 이름 변경하면 기존 인덱스와 공존 가능

**관련 링크**:
- [Snapshot/Restore 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html)
- [SLM 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-lifecycle-management.html)
