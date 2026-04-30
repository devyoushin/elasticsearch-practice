# 인덱스 관리

## 개요

Elasticsearch 인덱스의 생성, 수정, 삭제, 별칭(alias) 관리와 ILM 정책 운영에 관한 내용이다. 운영 중 인덱스 설정 변경은 제한이 있으므로 alias 기반의 zero-downtime 전환 패턴을 사용한다.

| 항목 | 값 |
|------|-----|
| 관련 API | `PUT /{index}`, `POST /_aliases`, `PUT /_ilm/policy/{name}` |
| 적용 환경 | Elasticsearch 8.x |
| 관련 개념 | [인덱스 설계](../core-concepts/index-design.md) |

---

## 설명

### 인덱스 CRUD

```bash
# 인덱스 생성
curl -X PUT "${ES_HOST}/my-index-v1" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" }
      }
    }
  }'

# 인덱스 목록 확인
curl -X GET "${ES_HOST}/_cat/indices?v&s=index" \
  -u "elastic:${ES_PASSWORD}" -k

# 인덱스 삭제 (주의!)
curl -X DELETE "${ES_HOST}/my-index-v1" \
  -u "elastic:${ES_PASSWORD}" -k
```

### Alias (별칭) 관리

Alias를 통해 인덱스 이름을 추상화하면 reindex 시 무중단 전환 가능하다.

```bash
# Alias 추가
curl -X POST "${ES_HOST}/_aliases" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "actions": [
      { "add": { "index": "my-index-v1", "alias": "my-index" } }
    ]
  }'

# Alias 전환 (원자적 연산 - 다운타임 없음)
curl -X POST "${ES_HOST}/_aliases" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "actions": [
      { "remove": { "index": "my-index-v1", "alias": "my-index" } },
      { "add":    { "index": "my-index-v2", "alias": "my-index" } }
    ]
  }'

# Alias 목록 확인
curl -X GET "${ES_HOST}/_cat/aliases?v" \
  -u "elastic:${ES_PASSWORD}" -k
```

### ILM 정책 생성 및 적용

```bash
# ILM 정책 생성
curl -X PUT "${ES_HOST}/_ilm/policy/logs-ilm-policy" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_primary_shard_size": "30gb",
              "max_age": "1d"
            }
          }
        },
        "warm": {
          "min_age": "3d",
          "actions": {
            "shrink": { "number_of_shards": 1 },
            "forcemerge": { "max_num_segments": 1 }
          }
        },
        "delete": {
          "min_age": "30d",
          "actions": { "delete": {} }
        }
      }
    }
  }'

# 기존 인덱스에 ILM 정책 적용
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index.lifecycle.name": "logs-ilm-policy" }'
```

### Reindex (데이터 마이그레이션)

```bash
# 동일 클러스터 reindex
curl -X POST "${ES_HOST}/_reindex?wait_for_completion=false" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "source": {
      "index": "my-index-v1",
      "size": 1000
    },
    "dest": {
      "index": "my-index-v2",
      "op_type": "create"
    }
  }'

# reindex 진행 상태 확인
curl -X GET "${ES_HOST}/_tasks?actions=*reindex&detailed" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 운영 중 변경 가능한 설정

```bash
# 복제본 수 변경 (운영 중 가능)
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index": { "number_of_replicas": 0 } }'

# refresh interval 변경
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index": { "refresh_interval": "30s" } }'
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| 인덱스 생성 실패 | 샤드 할당 가능 노드 없음 | 노드 상태 및 디스크 확인 |
| `resource_already_exists_exception` | 동일 이름 인덱스 존재 | 인덱스명 변경 또는 기존 삭제 |
| ILM이 동작 안 함 | `index.lifecycle.name` 미설정 | 설정 확인 및 적용 |
| Reindex 속도 느림 | `refresh_interval` 짧음, 복제본 있음 | 임시로 `-1`, `replicas: 0` 설정 |

### Reindex 성능 최적화

**증상**: Reindex가 너무 느림

**원인**: 기본 refresh + replica 오버헤드

**해결책**:
```bash
# 대상 인덱스 임시 설정 변경
curl -X PUT "${ES_HOST}/my-index-v2/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index": {
      "refresh_interval": "-1",
      "number_of_replicas": 0
    }
  }'

# reindex 실행 (병렬 처리)
curl -X POST "${ES_HOST}/_reindex?slices=auto&wait_for_completion=false" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "source": { "index": "my-index-v1" },
    "dest":   { "index": "my-index-v2" }
  }'

# 완료 후 설정 복원
curl -X PUT "${ES_HOST}/my-index-v2/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index": {
      "refresh_interval": "5s",
      "number_of_replicas": 1
    }
  }'
```

---

## 모니터링 & 검증

```bash
# 인덱스 상세 통계
curl -X GET "${ES_HOST}/my-index/_stats?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# ILM 상태 확인
curl -X GET "${ES_HOST}/my-index/_ilm/explain?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# ILM 정책 목록
curl -X GET "${ES_HOST}/_ilm/policy?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# Alias 상태 확인
curl -X GET "${ES_HOST}/_cat/aliases?v&s=alias" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## TIP

- 인덱스 삭제는 복구 불가. 삭제 전 반드시 스냅샷 확인
- ILM rollover는 write alias가 있어야 동작 (Data Stream은 자동 관리)
- `number_of_shards`는 생성 후 변경 불가 → `_shrink` 또는 reindex만 가능

**관련 링크**:
- [Index Management 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices.html)
- [ILM 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
