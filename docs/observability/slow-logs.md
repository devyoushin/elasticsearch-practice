# 슬로우 로그 (Slow Logs)

## 개요

Elasticsearch의 Slow Log는 지정한 임계값을 초과하는 검색(search)과 인덱싱(indexing) 작업을 로그로 기록한다. 성능 문제의 원인이 되는 쿼리와 문서를 찾아내는 데 필수적이다.

| 항목 | 값 |
|------|-----|
| 관련 API | `PUT /{index}/_settings` (slowlog threshold) |
| 로그 위치 | Elasticsearch pod 로그 (`kubectl logs`) |
| 관련 개념 | [검색 성능](../performance/search-performance.md) |

---

## 설명

### 검색 Slow Log 설정

```bash
# 인덱스 수준 slow log 임계값 설정
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index.search.slowlog.threshold.query.warn":  "5s",
    "index.search.slowlog.threshold.query.info":  "2s",
    "index.search.slowlog.threshold.query.debug": "1s",
    "index.search.slowlog.threshold.query.trace": "500ms",
    "index.search.slowlog.threshold.fetch.warn":  "1s",
    "index.search.slowlog.threshold.fetch.info":  "500ms",
    "index.search.slowlog.level":                 "info"
  }'

# 전체 인덱스에 적용 (인덱스 템플릿 활용)
curl -X PUT "${ES_HOST}/_index_template/slowlog-settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index_patterns": ["*"],
    "priority": 100,
    "template": {
      "settings": {
        "index.search.slowlog.threshold.query.warn": "3s",
        "index.search.slowlog.threshold.fetch.warn": "1s",
        "index.search.slowlog.level": "warn"
      }
    }
  }'
```

### 인덱싱 Slow Log 설정

```bash
# 인덱싱 slow log 임계값 설정
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index.indexing.slowlog.threshold.index.warn":  "10s",
    "index.indexing.slowlog.threshold.index.info":  "5s",
    "index.indexing.slowlog.threshold.index.debug": "2s",
    "index.indexing.slowlog.threshold.index.trace": "500ms",
    "index.indexing.slowlog.level":                 "info",
    "index.indexing.slowlog.source":                "1000"
  }'
```

`index.indexing.slowlog.source`: 로그에 포함할 문서 source의 문자 수 (0이면 미포함)

### Slow Log 확인 (ECK 환경)

ECK 환경에서 slow log는 pod의 stdout에 JSON 형식으로 출력된다.

```bash
# search slow log 확인
kubectl logs -n elasticsearch my-cluster-es-data-hot-0 \
  | grep "index.search.slowlog"

# indexing slow log 확인
kubectl logs -n elasticsearch my-cluster-es-data-hot-0 \
  | grep "index.indexing.slowlog"

# 최근 1시간 slow log (jq 파싱)
kubectl logs -n elasticsearch my-cluster-es-data-hot-0 --since=1h \
  | grep "slowlog" \
  | jq '{timestamp, took: .took, query: .source}'

# 전체 data 노드에서 slow log 수집
for pod in $(kubectl get pods -n elasticsearch \
  -l elasticsearch.k8s.elastic.co/node-data=true \
  -o name | sed 's/pod\///'); do
  echo "=== ${pod} ==="
  kubectl logs -n elasticsearch ${pod} --since=30m | grep "slowlog"
done
```

### Slow Log 임계값 조회 및 삭제

```bash
# 현재 임계값 설정 확인
curl -X GET "${ES_HOST}/my-index/_settings?filter_path=*.*.index.search.slowlog,*.*.index.indexing.slowlog&pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# slow log 임계값 비활성화 (-1로 설정)
curl -X PUT "${ES_HOST}/my-index/_settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index.search.slowlog.threshold.query.warn":  "-1",
    "index.search.slowlog.threshold.query.info":  "-1",
    "index.search.slowlog.threshold.fetch.warn":  "-1",
    "index.indexing.slowlog.threshold.index.warn": "-1"
  }'
```

### Kibana에서 Slow Log 분석

ECK Stack Monitoring 또는 Filebeat로 slow log를 Kibana로 수집하면 시각화가 가능하다.

Kibana Discover에서 필터:
```
log.logger: "index.search.slowlog" AND event.duration > 2000000000
```

(event.duration 단위: 나노초)

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| Slow log가 출력되지 않음 | 임계값 설정 안 됨 or level 불일치 | 설정 확인, `trace` 레벨로 낮춰서 테스트 |
| 너무 많은 slow log | 임계값이 너무 낮음 | `warn` 수준으로 높이기 |
| slow log에 쿼리 내용 없음 | Elasticsearch 8.x 보안 기본값 | `index.search.slowlog.include.user: true` 설정 |

### 특정 쿼리의 slow 원인 분석

**증상**: Slow log에서 특정 쿼리 패턴이 반복

**원인**: 비효율적인 쿼리 (wildcard, script, nested 남용 등)

**해결책**:
```bash
# Profile API로 해당 쿼리 분석
curl -X POST "${ES_HOST}/my-index/_search" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "profile": true,
    "query": {
      "wildcard": { "message": "*timeout*" }
    }
  }'

# Profile 결과에서 시간이 오래 걸리는 단계 확인
# → wildcard를 match_phrase 또는 full-text search로 변경 검토
```

---

## 모니터링 & 검증

```bash
# slow log 설정 일괄 확인
curl -X GET "${ES_HOST}/_settings?filter_path=*.*.index.search.slowlog.*&pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# slow log 발생 여부 실시간 확인
kubectl logs -f -n elasticsearch my-cluster-es-data-hot-0 \
  | grep "slowlog"

# 검색 성능 통계 (총 소요 시간 확인)
curl -X GET "${ES_HOST}/_nodes/stats/indices/search?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## TIP

- 프로덕션 기본값: query `warn: 5s`, fetch `warn: 1s`로 설정
- `trace` 레벨은 성능 분석 시에만 임시 활성화 (로그 량 폭증 주의)
- Slow log의 `took_millis`가 일정하게 높으면 쿼리 최적화 필요, 간헐적으로 높으면 GC/resource 문제

**관련 링크**:
- [Slow Log 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html)
- [Search Profiler](https://www.elastic.co/guide/en/kibana/current/xpack-profiler.html)
