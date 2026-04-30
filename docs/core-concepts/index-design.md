# 인덱스 설계

## 개요

Elasticsearch에서 인덱스(Index)는 데이터를 저장하는 논리적 단위다. 샤드 수, 매핑(Mapping), ILM(Index Lifecycle Management) 정책을 잘못 설계하면 이후 수정이 어렵기 때문에 초기 설계가 매우 중요하다.

| 항목 | 값 |
|------|-----|
| 관련 API | `PUT /{index}`, `PUT /_index_template/{name}`, `PUT /_ilm/policy/{name}` |
| 적용 환경 | Elasticsearch 8.x |
| 관련 개념 | [클러스터 아키텍처](cluster-architecture.md) |

---

## 설명

### 샤드 설계 원칙

| 항목 | 권장값 | 이유 |
|------|--------|------|
| 샤드 크기 | 10~50GB | 너무 작으면 오버헤드, 너무 크면 복구 느림 |
| 복제본 수 | 1 (프로덕션) | 노드 장애 시 데이터 보호 |
| 샤드 수 | 데이터 노드 수 이하 | 노드당 샤드 오버헤드 최소화 |

샤드 수는 인덱스 생성 후 변경 불가. 변경이 필요하면 `reindex` 필요.

### 매핑 (Mapping)

```json
PUT /my-index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "refresh_interval": "5s"
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "user_id": {
        "type": "keyword"
      },
      "message": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "response_time_ms": {
        "type": "integer"
      },
      "tags": {
        "type": "keyword"
      }
    }
  }
}
```

### 주요 필드 타입

| 타입 | 사용 용도 | 특징 |
|------|-----------|------|
| `keyword` | 정확 일치, 집계, 정렬 | 분석(analyze) 없음 |
| `text` | 전문 검색 (full-text search) | 형태소 분석기 적용 |
| `date` | 날짜/시간 | ISO 8601, epoch 지원 |
| `integer`/`long` | 숫자 | 범위 쿼리, 집계 |
| `ip` | IP 주소 | CIDR 범위 쿼리 지원 |
| `nested` | 배열 내 객체 독립 쿼리 | 성능 비용 높음 |

### ILM (Index Lifecycle Management)

```json
PUT /_ilm/policy/logs-ilm-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "30gb",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Data Stream (시계열 데이터 권장)

```bash
# 1. 인덱스 템플릿 생성 (data_stream 포함)
curl -X PUT "${ES_HOST}/_index_template/logs-app-template" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index_patterns": ["logs-app-*"],
    "data_stream": {},
    "template": {
      "settings": {
        "index.lifecycle.name": "logs-ilm-policy"
      }
    },
    "priority": 200
  }'

# 2. Data stream 생성
curl -X PUT "${ES_HOST}/_data_stream/logs-app-production" \
  -u "elastic:${ES_PASSWORD}" -k

# 3. 문서 인덱싱 (반드시 @timestamp 포함)
curl -X POST "${ES_HOST}/logs-app-production/_doc" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{"@timestamp": "2024-01-15T10:00:00Z", "message": "test"}'
```

### 모범 사례

- `dynamic: "strict"` 설정으로 의도치 않은 필드 자동 생성 방지
- 검색만 필요한 필드는 `doc_values: false`, 정렬/집계만 필요한 필드는 `index: false`
- 시계열 데이터는 Data Stream + ILM 조합 사용

### 안티패턴

- `dynamic: true` (기본값) 그대로 사용 → mapping explosion 위험
- 너무 많은 샤드 수 설정 → 오버헤드 증가
- `_source: false` 설정 → `update`, `reindex` 불가

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `mapper_parsing_exception` | 필드 타입 불일치 | 매핑 확인 후 타입에 맞는 값 전송 |
| `index_closed_exception` | ILM cold 단계 freeze | `POST /my-index/_unfreeze` |
| 샤드 크기 불균형 | ILM rollover 기준 미도달 | rollover 조건 조정 |

### 매핑 변경 (필드 추가는 가능, 타입 변경은 reindex 필요)

**증상**: 기존 필드 타입 변경 필요

**해결책**:
```bash
# 1. 새 인덱스 생성 (새 매핑)
curl -X PUT "${ES_HOST}/my-index-v2" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "mappings": { ... } }'

# 2. 데이터 마이그레이션
curl -X POST "${ES_HOST}/_reindex" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "source": { "index": "my-index" },
    "dest": { "index": "my-index-v2" }
  }'

# 3. alias 전환
curl -X POST "${ES_HOST}/_aliases" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "actions": [
      { "remove": { "index": "my-index", "alias": "my-alias" } },
      { "add":    { "index": "my-index-v2", "alias": "my-alias" } }
    ]
  }'
```

---

## 모니터링 & 검증

```bash
# 인덱스 매핑 확인
curl -X GET "${ES_HOST}/my-index/_mapping?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 인덱스 설정 확인
curl -X GET "${ES_HOST}/my-index/_settings?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# ILM 정책 적용 상태 확인
curl -X GET "${ES_HOST}/my-index/_ilm/explain?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 샤드 크기 확인
curl -X GET "${ES_HOST}/_cat/shards/my-index?v&h=index,shard,prirep,store,node" \
  -u "elastic:${ES_PASSWORD}" -k

# Data stream 목록
curl -X GET "${ES_HOST}/_data_stream?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## TIP

- 인덱스 생성 전 항상 `_analyze` API로 분석기 동작 검증
- ILM rollover는 `min_age`가 아닌 `max_primary_shard_size` 기준 사용 권장 (데이터 양 예측 어려운 경우)
- 운영 중 `number_of_replicas` 변경은 가능: `PUT /my-index/_settings { "index": { "number_of_replicas": 0 } }`

**관련 링크**:
- [Mapping 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)
- [ILM 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
