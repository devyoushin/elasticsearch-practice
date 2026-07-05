# Query DSL

## 개요

Elasticsearch의 Query DSL(Domain Specific Language)은 JSON 기반의 검색 쿼리 언어다. `query` 문맥(점수 계산)과 `filter` 문맥(캐시 활용, 점수 없음)을 구분하는 것이 성능의 핵심이다.

| 항목 | 값 |
|------|-----|
| 관련 API | `GET /{index}/_search`, `POST /{index}/_search` |
| 적용 환경 | Elasticsearch 8.x |
| 관련 개념 | [인덱스 설계](index-design.md), [검색 성능](../03-performance/search-performance.md) |

---

## 설명

### query vs filter 문맥

| 구분 | 점수(score) | 캐시 | 사용 상황 |
|------|------------|------|-----------|
| `query` | 계산함 | 없음 | 관련도 순 정렬이 필요한 전문 검색 |
| `filter` | 계산 안 함 | 있음 (filter cache) | 정확 일치, 범위, 존재 여부 확인 |

**원칙**: 관련도 점수가 필요 없으면 `filter` 사용 → 성능 향상

### 기본 쿼리 구조

```json
GET /my-index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "elasticsearch error" } }
      ],
      "filter": [
        { "term":  { "service.name": "api-server" } },
        { "range": { "@timestamp": { "gte": "now-1h", "lte": "now" } } }
      ],
      "must_not": [
        { "term": { "log.level": "DEBUG" } }
      ],
      "should": [
        { "term": { "log.level": "ERROR" } }
      ],
      "minimum_should_match": 0
    }
  },
  "sort": [
    { "@timestamp": { "order": "desc" } }
  ],
  "_source": ["@timestamp", "message", "log.level", "service.name"],
  "size": 20,
  "from": 0
}
```

### 자주 사용하는 쿼리 타입

#### match (전문 검색)
```json
{ "match": { "message": "connection timeout" } }
```

#### term (정확 일치, keyword 타입)
```json
{ "term": { "status": "active" } }
```

#### terms (다중 값 일치)
```json
{ "terms": { "log.level": ["ERROR", "WARN"] } }
```

#### range (범위)
```json
{
  "range": {
    "response_time_ms": {
      "gte": 1000,
      "lte": 5000
    }
  }
}
```

#### exists (필드 존재 여부)
```json
{ "exists": { "field": "error.message" } }
```

#### wildcard / prefix (패턴 매칭, 성능 주의)
```json
{ "prefix": { "user_id": "user_" } }
```

### Aggregation (집계)

```json
GET /logs-app-*/_search
{
  "size": 0,
  "query": {
    "range": { "@timestamp": { "gte": "now-1d" } }
  },
  "aggs": {
    "by_service": {
      "terms": {
        "field": "service.name",
        "size": 10
      },
      "aggs": {
        "error_count": {
          "filter": { "term": { "log.level": "ERROR" } }
        },
        "avg_response": {
          "avg": { "field": "duration_ms" }
        },
        "p99_response": {
          "percentiles": {
            "field": "duration_ms",
            "percents": [50, 95, 99]
          }
        }
      }
    },
    "requests_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      }
    }
  }
}
```

### 모범 사례

- 점수 계산이 필요 없으면 `bool.filter` 사용 (filter cache 활용)
- `_source` 필드 제한으로 네트워크 비용 감소
- `size: 0`으로 집계만 실행 시 문서 반환 비용 제거
- `from/size` 대신 대량 조회는 `search_after` 사용

### 안티패턴

- `wildcard: "*pattern*"` 남용 → 인덱스 전체 스캔
- `script` 쿼리 남용 → 캐시 불가, 성능 저하
- `query.match_all` + 대량 조회 → `scroll` 또는 `search_after` 사용

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| 검색 결과가 예상과 다름 | text vs keyword 타입 혼동 | `_analyze` API로 분석기 결과 확인 |
| 집계 결과 부정확 | `text` 필드에 terms 집계 | `.keyword` 서브 필드 사용 |
| 검색이 매우 느림 | filter 대신 query 문맥 사용 | `bool.filter`로 이동 |

### 분석기 동작 확인

**증상**: `match` 쿼리가 예상한 문서를 반환하지 않음

**원인**: 분석기가 인덱싱/검색 시 다른 토큰 생성

**해결책**:
```bash
# 인덱싱 시 분석 결과 확인
curl -X POST "${ES_HOST}/my-index/_analyze" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "field": "message",
    "text": "Connection Timeout Error"
  }'

# 검색 시 분석 결과 확인
curl -X POST "${ES_HOST}/_analyze" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "analyzer": "standard",
    "text": "Connection Timeout Error"
  }'
```

### keyword 필드 집계 오류

**증상**: `terms` 집계에서 `fielddata` 오류

**원인**: `text` 타입 필드에 집계 시도

**해결책**:
```json
// 올바른 방법: .keyword 서브 필드 사용
{
  "aggs": {
    "by_level": {
      "terms": { "field": "log.level.keyword" }
    }
  }
}
```

---

## 모니터링 & 검증

```bash
# 쿼리 프로파일링 (성능 분석)
curl -X GET "${ES_HOST}/my-index/_search" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "profile": true,
    "query": {
      "bool": {
        "filter": [
          { "term": { "service.name": "api-server" } }
        ]
      }
    }
  }'

# Explain API (점수 계산 이유)
curl -X GET "${ES_HOST}/my-index/_explain/{doc_id}" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "query": { "match": { "message": "error" } } }'

# 검색 통계
curl -X GET "${ES_HOST}/_nodes/stats/indices/search?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## TIP

- `bool.filter`는 bitset 캐시를 사용하므로, 동일한 filter는 두 번째 요청부터 빠름
- `search_after`로 페이지네이션: `sort` 값을 다음 요청의 `search_after`로 전달
- 날짜 범위는 `now-1h/h` (슬래시 반올림) 사용 시 filter cache 재사용률 높아짐

**관련 링크**:
- [Query DSL 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [Aggregations 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)
