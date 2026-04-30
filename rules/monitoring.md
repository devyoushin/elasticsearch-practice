# 모니터링 표준

## 필수 확인 API

```bash
# 클러스터 헬스
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 노드 상태 (역할, heap, CPU, 디스크)
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role" \
  -u "elastic:${ES_PASSWORD}" -k

# 인덱스 상태
curl -X GET "${ES_HOST}/_cat/indices?v&s=index&h=index,health,status,pri,rep,docs.count,store.size" \
  -u "elastic:${ES_PASSWORD}" -k

# 샤드 상태 (UNASSIGNED 확인)
curl -X GET "${ES_HOST}/_cat/shards?v&s=state&h=index,shard,prirep,state,unassigned.reason,node" \
  -u "elastic:${ES_PASSWORD}" -k

# 샤드 할당 실패 원인 분석
curl -X GET "${ES_HOST}/_cluster/allocation/explain?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

## 주요 메트릭

| 메트릭 | 임계값 | 설명 |
|--------|--------|------|
| `cluster.status` | green | yellow: 복제본 미할당, red: 주샤드 미할당 |
| `jvm.heap_used_percent` | < 75% | 85% 이상 시 OOM 위험 |
| `jvm.gc.collectors.old.collection_time_in_millis` | 증가 추세 없음 | Full GC 빈도 |
| `indices.indexing.index_time_in_millis` | 요청 대비 안정적 | 인덱싱 지연 |
| `thread_pool.write.rejected` | 0 | write queue 포화 |
| `fs.data.available_in_bytes` | > 15% 여유 | 디스크 여유 공간 |
| `indices.search.query_time_in_millis` | 요청 대비 안정적 | 검색 지연 |

## Kibana Dev Tools 필수 쿼리

```json
// 클러스터 통계
GET /_cluster/stats?human&pretty

// 노드별 JVM 메모리 상세
GET /_nodes/stats/jvm?pretty

// Thread pool 상태 (rejected 확인)
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected

// 인덱싱/검색 성능 통계
GET /_nodes/stats/indices?pretty

// 슬로우 로그 임계값 확인
GET /my-index/_settings?filter_path=*.*.index.search.slowlog,*.*.index.indexing.slowlog
```

## 알람 기준 (Kibana Alerting)

| 알람명 | 조건 | 심각도 |
|--------|------|--------|
| Cluster Red | `cluster.status == red` | CRITICAL |
| Cluster Yellow | `cluster.status == yellow` | WARNING |
| Heap > 85% | `jvm.heap_used_percent > 85` | CRITICAL |
| Disk > 85% | `disk.used_percent > 85` | WARNING |
| Rejected Writes | `thread_pool.write.rejected > 0` | WARNING |
| Unassigned Shards | `unassigned_shards > 0` | WARNING |
