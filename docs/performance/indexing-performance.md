# 인덱싱 성능

## 개요

Elasticsearch의 인덱싱 처리량을 최대화하기 위한 Bulk API 활용, `refresh_interval` 조정, write thread pool 튜닝 등의 최적화 기법을 다룬다.

| 항목 | 값 |
|------|-----|
| 관련 API | `POST /_bulk`, `PUT /{index}/_settings` |
| 관련 메트릭 | `indices.indexing.index_total`, `thread_pool.write.rejected` |
| 관련 개념 | [리소스 튜닝](resource-tuning.md) |

---

## 설명

### Bulk API

단건 인덱싱 대신 Bulk API를 사용하면 네트워크 오버헤드와 merge 비용을 대폭 줄일 수 있다.

```bash
# Bulk 인덱싱 (한 번에 여러 문서)
curl -X POST "${ES_HOST}/_bulk" \
  -H "Content-Type: application/x-ndjson" \
  -u "elastic:${ES_PASSWORD}" -k \
  --data-binary '
{"index": {"_index": "my-index", "_id": "1"}}
{"@timestamp": "2024-01-15T10:00:00Z", "message": "first document"}
{"index": {"_index": "my-index", "_id": "2"}}
{"@timestamp": "2024-01-15T10:00:01Z", "message": "second document"}
{"create": {"_index": "my-index"}}
{"@timestamp": "2024-01-15T10:00:02Z", "message": "auto-id document"}
'

# Bulk 응답 오류 확인 (errors: false면 전체 성공)
# errors: true면 items 배열에서 오류 항목 확인
```

**Bulk 크기 권장값**: 요청당 5~15MB, 1,000~5,000개 문서

### refresh_interval 조정

`refresh_interval`은 데이터가 검색 가능해지는 주기다. 짧을수록 실시간이지만 merge 오버헤드 증가.

```bash
# 대량 인덱싱 전 임시로 refresh 비활성화
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index": { "refresh_interval": "-1" } }'

# 대량 인덱싱 완료 후 수동 refresh
curl -X POST "${ES_HOST}/my-index/_refresh" \
  -u "elastic:${ES_PASSWORD}" -k

# 정상 주기로 복원
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index": { "refresh_interval": "5s" } }'
```

| refresh_interval | 적합한 시나리오 |
|-----------------|----------------|
| `1s` | 실시간 검색 필요 (기본값) |
| `5s~30s` | 준실시간 로그 수집 (권장) |
| `-1` | 대량 초기 적재(bulk load) |

### 복제본 설정

```bash
# 초기 대량 적재 시 복제본 0으로 설정 (인덱싱 속도 2배↑)
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index": { "number_of_replicas": 0 } }'

# 완료 후 복제본 복원
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "index": { "number_of_replicas": 1 } }'
```

### 인덱스 설정 최적화

```json
PUT /my-high-throughput-index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "10s",
    "index.translog.durability": "async",
    "index.translog.sync_interval": "30s",
    "index.merge.policy.max_merged_segment": "5gb"
  },
  "mappings": {
    "dynamic": "strict",
    "_source": { "enabled": true },
    "properties": {
      "@timestamp": { "type": "date" },
      "message": {
        "type": "text",
        "norms": false
      }
    }
  }
}
```

### Write Thread Pool 확인

```bash
# write queue 상태 확인 (rejected > 0 이면 병목)
curl -X GET "${ES_HOST}/_cat/thread_pool/write?v&h=node_name,active,queue,rejected,completed" \
  -u "elastic:${ES_PASSWORD}" -k

# 노드 thread pool 설정
curl -X GET "${ES_HOST}/_nodes/settings?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

write thread pool size = `min(10, vCPU + 1)` (기본값, 변경 불가)

Queue가 포화되면: 클라이언트에서 `429 Too Many Requests` 수신 → 클라이언트 측에서 retry + backoff 처리 필요

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `429 Too Many Requests` | write queue 포화 | 클라이언트 처리량 감소, 데이터 노드 증설 |
| 인덱싱 속도 점차 감소 | segment merge 부하 | `refresh_interval` 늘리기, merge throttle 조정 |
| Bulk 요청 일부 실패 | 문서 크기 초과, 매핑 오류 | bulk 응답의 `items` 배열에서 오류 항목 확인 |

### 인덱싱 성능 저하 진단

**증상**: 인덱싱 속도가 갑자기 느려짐

**원인**: disk flush 지연, GC 압박, merge 폭주

**해결책**:
```bash
# 인덱싱 통계 확인
curl -X GET "${ES_HOST}/_nodes/stats/indices/indexing?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# Segment 상태 확인 (merge 중인지)
curl -X GET "${ES_HOST}/_cat/segments?v&h=index,shard,segment,size,size.memory,committed,search" \
  -u "elastic:${ES_PASSWORD}" -k

# 노드 JVM GC 확인
curl -X GET "${ES_HOST}/_nodes/stats/jvm?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## 모니터링 & 검증

```bash
# 노드별 인덱싱 속도
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,indexing.index_total,indexing.index_time,indexing.index_failed" \
  -u "elastic:${ES_PASSWORD}" -k

# write thread pool rejected 카운트
curl -X GET "${ES_HOST}/_cat/thread_pool/write?v" \
  -u "elastic:${ES_PASSWORD}" -k

# 인덱스별 인덱싱 통계
curl -X GET "${ES_HOST}/my-index/_stats/indexing?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 주요 메트릭

| 메트릭 | 정상 범위 | 설명 |
|--------|-----------|------|
| `thread_pool.write.rejected` | 0 | write queue 초과 거부 |
| `indexing.index_failed` | 0 | 인덱싱 실패 건수 |
| `segments.count` per shard | < 100 | segment 수 (많을수록 merge 필요) |

---

## TIP

- Bulk 요청 크기는 5~15MB가 최적. 너무 크면 오히려 GC 압박 증가
- `translog.durability: async` + `translog.sync_interval: 30s` 설정 시 성능 향상되지만 노드 크래시 시 30초치 데이터 손실 가능
- 초기 대량 적재 후 `POST /my-index/_forcemerge?max_num_segments=1` 실행하면 검색 성능 개선

**관련 링크**:
- [Tune for Indexing Speed](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-indexing-speed.html)
