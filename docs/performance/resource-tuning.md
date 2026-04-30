# 리소스 튜닝

## 개요

Elasticsearch의 JVM Heap, Thread Pool, Circuit Breaker, OS 레벨 설정을 조정하여 안정적인 운영을 달성하는 방법을 다룬다. ECK 환경에서는 대부분의 설정이 `podTemplate` 또는 `config` 블록으로 관리된다.

| 항목 | 값 |
|------|-----|
| 관련 API | `GET /_nodes/stats`, `GET /_cluster/settings` |
| 관련 메트릭 | `jvm.heap_used_percent`, `thread_pool.*.rejected` |
| 관련 개념 | [클러스터 아키텍처](../core-concepts/cluster-architecture.md) |

---

## 설명

### JVM Heap 설정

```yaml
# ECK podTemplate에서 JVM Heap 설정
spec:
  nodeSets:
    - name: data-hot
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms8g -Xmx8g"
              resources:
                requests:
                  memory: 16Gi   # Heap(8GB) * 2 = 노드 메모리 권장
                  cpu: "4"
                limits:
                  memory: 16Gi
```

| 규칙 | 값 |
|------|-----|
| Heap 크기 | 가용 메모리의 50%, 최대 31GB |
| `-Xms` = `-Xmx` | 반드시 동일 (GC 재조정 방지) |
| 31GB 이하 유지 이유 | JVM Compressed OOPs 활용 (포인터 압축) |

### Circuit Breaker 설정

Circuit Breaker는 OOM을 방지하기 위해 메모리 사용량이 임계값을 초과하면 요청을 거부한다.

```bash
# Circuit Breaker 현재 상태 확인
curl -X GET "${ES_HOST}/_nodes/stats/breaker?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# Circuit Breaker 설정 조정
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "persistent": {
      "indices.breaker.total.limit":     "70%",
      "indices.breaker.request.limit":   "40%",
      "indices.breaker.fielddata.limit": "40%"
    }
  }'
```

| Breaker | 기본값 | 역할 |
|---------|--------|------|
| `total` | 95% heap | 전체 메모리 상한 |
| `request` | 60% heap | 집계, 필드 데이터 로딩 |
| `fielddata` | 40% heap | field data cache |
| `inflight_requests` | 100% heap | 진행 중인 요청 크기 |

### Thread Pool 튜닝

Thread pool은 대부분 자동 계산되므로 직접 변경이 필요한 경우는 드물다.

```bash
# Thread pool 상태 확인
curl -X GET "${ES_HOST}/_cat/thread_pool?v&h=node_name,name,type,active,queue,rejected,completed" \
  -u "elastic:${ES_PASSWORD}" -k

# write pool 상태 (rejected > 0 이면 처리 지연)
curl -X GET "${ES_HOST}/_cat/thread_pool/write,search?v" \
  -u "elastic:${ES_PASSWORD}" -k
```

| Thread Pool | 기본 크기 | 설명 |
|-------------|-----------|------|
| `write` | vCPU + 1 (max 10) | 인덱싱, 삭제, bulk |
| `search` | (vCPU * 3) / 2 + 1 | 검색, 집계 |
| `get` | vCPU + 1 | get, mget |

### OS 수준 설정 (ECK initContainer)

```yaml
spec:
  nodeSets:
    - name: data-hot
      podTemplate:
        spec:
          initContainers:
            # vm.max_map_count 설정 (필수)
            - name: sysctl
              securityContext:
                privileged: true
                runAsUser: 0
              command:
                - sh
                - -c
                - |
                  sysctl -w vm.max_map_count=262144
                  sysctl -w vm.swappiness=1
                  sysctl -w net.core.somaxconn=65535
```

### Memory Swapping 방지

```yaml
# elasticsearch.yml (ECK config 블록)
config:
  bootstrap.memory_lock: true
```

ECK에서는 `podTemplate.spec.containers[*].securityContext`로 설정:
```yaml
securityContext:
  capabilities:
    add:
      - IPC_LOCK
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `OutOfMemoryError` in logs | Heap 부족, GC 압박 | Heap 크기 증가, 데이터 노드 추가 |
| `CircuitBreakingException` | 집계/요청이 메모리 한도 초과 | `size: 0` 집계, breaker limit 조정 |
| `rejected execution` on write | write queue 포화 | 인덱싱 속도 감소, 데이터 노드 증설 |

### GC 압박 진단

**증상**: 응답 지연 증가, `jvm.gc.collectors.old.collection_time_in_millis` 급증

**원인**: Heap 사용량 > 75%, Old GC 빈도 증가

**해결책**:
```bash
# JVM GC 상태 확인
curl -X GET "${ES_HOST}/_nodes/stats/jvm?pretty" \
  -u "elastic:${ES_PASSWORD}" -k \
  | python3 -m json.tool | grep -A 20 '"gc"'

# 노드별 heap 사용률
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,heap.current,heap.max,heap.percent" \
  -u "elastic:${ES_PASSWORD}" -k

# Field data 캐시 강제 비우기 (즉각 heap 확보)
curl -X POST "${ES_HOST}/_cache/clear?fielddata=true" \
  -u "elastic:${ES_PASSWORD}" -k
```

**예방**: Heap 75% 이상 시 알람 설정, `fielddata.cache.size` 제한

### CircuitBreakingException 대응

**증상**: `Data too large, data for [<http_request>] would be [...]`

**원인**: 집계 결과가 request breaker 초과

**해결책**:
```bash
# 1. 임시: request breaker 한도 상향
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "transient": { "indices.breaker.request.limit": "70%" } }'

# 2. 근본: 집계 쿼리 최적화 (size 줄이기, filter 먼저 적용)
```

---

## 모니터링 & 검증

```bash
# 전체 노드 리소스 현황
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role" \
  -u "elastic:${ES_PASSWORD}" -k

# Circuit Breaker 트립 횟수
curl -X GET "${ES_HOST}/_nodes/stats/breaker?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# Thread pool rejected 확인
curl -X GET "${ES_HOST}/_nodes/stats/thread_pool?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 주요 메트릭

| 메트릭 | 임계값 | 설명 |
|--------|--------|------|
| `jvm.heap_used_percent` | < 75% (경고), < 85% (위험) | JVM Heap 사용률 |
| `jvm.gc.collectors.old.collection_count` | 증가율 안정적 | Full GC 횟수 |
| `breaker.*.tripped` | 0 | Circuit Breaker 동작 횟수 |
| `thread_pool.write.rejected` | 0 | write 큐 초과 |

---

## TIP

- 31GB Heap이 30GB Heap보다 느릴 수 있음: Compressed OOPs는 ~31GB 이하에서만 활성화
- `vm.swappiness=1` (0이 아닌 이유: Linux OOM killer가 swap 없으면 프로세스 kill)
- Heap Dump 설정: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof`

**관련 링크**:
- [Important System Configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)
- [Circuit Breaker Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html)
