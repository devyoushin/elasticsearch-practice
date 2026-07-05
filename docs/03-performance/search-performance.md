# 검색 성능

## 개요

Elasticsearch 검색 속도를 최적화하기 위한 쿼리 설계, 캐시 활용, Profile API 분석, Slow Log 설정을 다룬다. 검색 성능의 핵심은 filter 문맥 활용과 불필요한 문서 로딩 최소화다.

| 항목 | 값 |
|------|-----|
| 관련 API | `GET /{index}/_search`, `GET /{index}/_search?profile=true` |
| 관련 메트릭 | `indices.search.query_time_in_millis`, `indices.search.fetch_time_in_millis` |
| 관련 개념 | [Query DSL](../01-core-concepts/query-dsl.md) |

---

## 설명

### Filter vs Query 문맥

검색 최적화의 핵심은 관련도 점수(score)가 필요 없는 조건을 `bool.filter`로 이동하는 것이다.

```json
// 나쁜 예: 모든 조건을 query에 넣음 (점수 계산 + 캐시 불가)
GET /my-index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "error" } },
        { "term":  { "service.name": "api" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}

// 좋은 예: 점수가 필요한 full-text만 query, 나머지는 filter
GET /my-index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "error" } }
      ],
      "filter": [
        { "term":  { "service.name": "api" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  }
}
```

### 캐시 종류와 활용

| 캐시 | 적용 대상 | 무효화 시점 |
|------|-----------|-----------|
| Filter (bitset) cache | `filter` 문맥의 쿼리 결과 | segment refresh 시 |
| Shard Request Cache | `size: 0` 집계 결과 | 샤드 refresh 시 |
| Field Data Cache | `text` 필드 집계/정렬 | 메모리 부족 시 eviction |

```bash
# 캐시 상태 확인
curl -X GET "${ES_HOST}/_nodes/stats/indices/query_cache,request_cache,fielddata?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 캐시 초기화 (필요 시)
curl -X POST "${ES_HOST}/my-index/_cache/clear" \
  -u "elastic:${ES_PASSWORD}" -k
```

### Profile API로 느린 쿼리 분석

```bash
curl -X GET "${ES_HOST}/my-index/_search" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "profile": true,
    "query": {
      "bool": {
        "must": [{ "match": { "message": "timeout" } }],
        "filter": [{ "term": { "service.name": "api" } }]
      }
    }
  }'
```

Profile 결과에서 주목할 항목:
- `time_in_nanos`: 각 쿼리 단계별 소요 시간
- `collector`: `SimpleTopScoreDocCollector` (정상), `BucketCollector` (집계)
- `rewrite_time`: 쿼리 재작성 시간 (크면 복잡한 쿼리)

### Slow Log 설정

```bash
# Slow log 임계값 설정 (인덱스 수준)
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index.search.slowlog.threshold.query.warn":  "5s",
    "index.search.slowlog.threshold.query.info":  "2s",
    "index.search.slowlog.threshold.fetch.warn":  "1s",
    "index.search.slowlog.threshold.fetch.info":  "500ms",
    "index.search.slowlog.level": "info"
  }'

# Slow log 확인 (ECK 환경)
kubectl logs -n elasticsearch my-cluster-es-data-hot-0 \
  | grep "index.search.slowlog"
```

### 검색 최적화 기법

```json
// 1. _source 필터링 (불필요한 필드 제외)
GET /my-index/_search
{
  "_source": ["@timestamp", "message", "log.level"],
  "query": { ... }
}

// 2. docvalue_fields (stored field 로딩 없이 집계)
GET /my-index/_search
{
  "size": 0,
  "docvalue_fields": ["@timestamp", "service.name"],
  "query": { ... }
}

// 3. search_after (deep pagination)
GET /my-index/_search
{
  "size": 100,
  "sort": [{ "@timestamp": "desc" }, { "_id": "asc" }],
  "search_after": ["2024-01-15T10:00:00Z", "abc123"]
}
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| 특정 쿼리만 느림 | wildcard/script 쿼리 | filter 문맥으로 변경, 매핑 최적화 |
| Field data OOM | `text` 필드 집계 | `.keyword` 서브필드 사용 |
| 집계가 부정확함 | `terms` agg의 shard 분산 문제 | `shard_size` 늘리기 |

### Field Data OOM

**증상**: `Data too large, data for [field] would be ...`

**원인**: `text` 타입 필드에 집계/정렬 시 field data 캐시 초과

**해결책**:
```bash
# 1. 즉각 대응: field data 캐시 초기화
curl -X POST "${ES_HOST}/_cache/clear?fielddata=true" \
  -u "elastic:${ES_PASSWORD}" -k

# 2. 근본 해결: .keyword 서브필드로 집계
# 매핑에서 "fields": {"keyword": {"type": "keyword"}} 추가 후 reindex

# 3. Field data 캐시 크기 제한 설정
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "persistent": {
      "indices.fielddata.cache.size": "30%"
    }
  }'
```

---

## 모니터링 & 검증

```bash
# 노드별 검색 통계
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,search.query_total,search.query_time,search.fetch_total,search.fetch_time" \
  -u "elastic:${ES_PASSWORD}" -k

# 인덱스별 검색 성능
curl -X GET "${ES_HOST}/my-index/_stats/search?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 캐시 hit rate 확인
curl -X GET "${ES_HOST}/_nodes/stats/indices/query_cache?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 주요 메트릭

| 메트릭 | 정상 범위 | 설명 |
|--------|-----------|------|
| `search.query_time_in_millis` 증가율 | 요청 수 대비 안정적 | 검색 처리 시간 |
| `query_cache.hit_count` / `miss_count` | hit rate > 70% | filter cache 효율 |
| `fielddata.memory_size_in_bytes` | heap의 20% 이하 | fielddata 메모리 |

---

## TIP

- `now-1h` 같은 상대 날짜는 `now-1h/h` (1시간 단위 반올림)로 변환 시 filter cache 재사용
- 집계 전용 요청은 `size: 0` 설정 → shard request cache 활성화
- `_forcemerge?max_num_segments=1` 후 read-only 인덱스는 검색 성능 최대화

**관련 링크**:
- [Tune for Search Speed](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html)
- [Profile API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-profile.html)
