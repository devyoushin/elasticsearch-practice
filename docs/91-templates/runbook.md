# Runbook: {작업명}

## 개요

| 항목 | 값 |
|------|-----|
| 목적 | ... |
| 소요 시간 | 예상 XX분 |
| 영향도 | 없음 / 부분 중단 / 전체 중단 |
| 롤백 가능 | 예 / 아니오 |

## 사전 조건

- [ ] Elasticsearch 클러스터 green 상태 확인
- [ ] 스냅샷 백업 완료 확인
- [ ] 충분한 디스크 공간 (15% 이상 여유)

```bash
# 사전 확인 명령어
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,heap.percent,disk.used_percent" \
  -u "elastic:${ES_PASSWORD}" -k
```

## 절차

### Step 1: {첫 번째 단계}

```bash
# 명령어
```

**예상 결과**: ...
**이상 신호**: ...

### Step 2: {두 번째 단계}

```bash
# 명령어
```

**예상 결과**: ...

## 검증

```bash
# 완료 후 확인 명령어
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

curl -X GET "${ES_HOST}/_cat/shards?v&s=state" \
  -u "elastic:${ES_PASSWORD}" -k
```

## 롤백

```bash
# 롤백 명령어
```

## 완료 기준

- [ ] 클러스터 상태 green
- [ ] 모든 샤드 할당 완료
- [ ] 인덱싱 정상 작동 확인
- [ ] Kibana 접근 가능 확인
