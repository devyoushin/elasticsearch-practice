# {서비스/기능명}

## 개요

{한 줄 정의}

| 항목 | 값 |
|------|-----|
| 관련 API | `/_cat/indices`, `/_cluster/health` |
| 적용 환경 | Elasticsearch 8.x, ECK |
| 관련 개념 | [인덱스 설계](../core-concepts/index-design.md) |

### 사용 목적
- ...

### 적용 시나리오
- ...

---

## 설명

### 핵심 개념

{개념 표 또는 목록}

### 설정 예시

```yaml
# ECK Elasticsearch CR 예시
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: my-cluster
  namespace: elasticsearch
spec:
  version: 8.15.0
  ...
```

```json
// Index 설정 예시
PUT /my-index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {}
  }
}
```

### 모범 사례

- ...

### 안티패턴

- ...

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| ... | ... | ... |

### {증상}

**원인**: ...

**해결책**:
```bash
curl -X POST "${ES_HOST}/..." \
  -u "elastic:${ES_PASSWORD}" \
  -k
```

**예방**: ...

---

## 모니터링 & 검증

```bash
# 상태 확인
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 상세 확인
curl -X GET "${ES_HOST}/_cat/indices?v" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 주요 메트릭

| 메트릭 | 정상 범위 | 설명 |
|--------|-----------|------|
| ... | ... | ... |

---

## TIP

- ...

**관련 링크**:
- [공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/)
