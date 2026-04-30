# 문서 작성 규칙

## 언어

- 문서 본문: 한국어
- 기술 용어, API명, 파라미터명: 영어 그대로 사용
- 예: "인덱스(Index)", "샤드(Shard)", "`refresh_interval`"

## 코드 블록

언어 태그 필수:

```bash
# curl: Elasticsearch REST API 호출
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" \
  -k
```

```json
// Elasticsearch API 요청/응답
{
  "index": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

```yaml
# ECK Custom Resource
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
```

```properties
# elasticsearch.yml 또는 jvm.options
-Xms4g
-Xmx4g
```

## 필수 섹션 (5개)

### 1. 개요
- 한 줄 정의
- 사용 목적 / 적용 시나리오
- 관련 개념 링크

### 2. 설명
- 핵심 개념 (표 또는 목록)
- 단계별 코드 예시
- 모범 사례 / 안티패턴

### 3. 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| ... | ... | ... |

또는:

**증상**: `cluster status is RED`
**원인**: 주 샤드(primary shard) 미할당
**해결책**: `POST /_cluster/reroute?retry_failed=true`

### 4. 모니터링 & 검증

필수 포함:
- `_cluster/health`, `_cat/indices`, `_cat/shards` 등 API 명령어
- Kibana Dev Tools 쿼리
- 주요 메트릭 (heap usage, JVM GC, indexing rate 등)

### 5. TIP

- 현장 운영 팁
- 관련 공식 문서 링크
- 주의사항
