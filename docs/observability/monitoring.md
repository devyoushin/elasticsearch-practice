# 모니터링

## 개요

Elasticsearch 클러스터의 상태를 지속적으로 파악하기 위한 핵심 API, Kibana Stack Monitoring, Prometheus 연동 방법을 다룬다. 클러스터 헬스, JVM, 인덱싱/검색 성능, 디스크 사용량이 주요 모니터링 대상이다.

| 항목 | 값 |
|------|-----|
| 관련 API | `/_cluster/health`, `/_nodes/stats`, `/_cat/*` |
| 적용 환경 | Elasticsearch 8.x, ECK v2.x |
| 관련 개념 | [리소스 튜닝](../performance/resource-tuning.md) |

---

## 설명

### 핵심 상태 확인 API

```bash
# 1. 클러스터 전체 헬스
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 2. 인덱스별 헬스 상태
curl -X GET "${ES_HOST}/_cluster/health?level=indices&pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 3. 노드 목록 및 리소스 현황
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role,master" \
  -u "elastic:${ES_PASSWORD}" -k

# 4. 인덱스 상태
curl -X GET "${ES_HOST}/_cat/indices?v&s=health,index&h=index,health,status,pri,rep,docs.count,store.size" \
  -u "elastic:${ES_PASSWORD}" -k

# 5. 미할당 샤드 확인
curl -X GET "${ES_HOST}/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason,node&s=state" \
  -u "elastic:${ES_PASSWORD}" -k
```

### Kibana Stack Monitoring 설정

ECK 환경에서 Metricbeat를 통해 Stack Monitoring을 활성화한다.

```yaml
# Elasticsearch CR에 monitoring 설정 추가
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: my-cluster
  namespace: elasticsearch
spec:
  version: 8.15.0
  monitoring:
    metrics:
      elasticsearchRefs:
        - name: my-cluster       # 모니터링 데이터를 저장할 클러스터 (자기 자신 또는 별도 모니터링 클러스터)
    logs:
      elasticsearchRefs:
        - name: my-cluster
```

```bash
# Stack Monitoring 데이터 확인
curl -X GET "${ES_HOST}/.monitoring-es-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "size": 1, "sort": [{"timestamp": "desc"}] }'
```

### Prometheus 연동 (Elasticsearch Exporter)

```yaml
# elasticsearch-exporter Deployment (Helm 없이)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-exporter
  namespace: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch-exporter
  template:
    metadata:
      labels:
        app: elasticsearch-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9114"
    spec:
      containers:
        - name: exporter
          image: quay.io/prometheuscommunity/elasticsearch-exporter:latest
          args:
            - --es.uri=https://my-cluster-es-http.elasticsearch.svc.cluster.local:9200
            - --es.ssl-skip-verify
            - --es.all
            - --es.indices
            - --es.shards
          env:
            - name: ES_USERNAME
              value: elastic
            - name: ES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-cluster-es-elastic-user
                  key: elastic
          ports:
            - containerPort: 9114
```

```bash
# Exporter 메트릭 확인
kubectl port-forward -n elasticsearch svc/elasticsearch-exporter 9114:9114 &
curl http://localhost:9114/metrics | grep elasticsearch_cluster_health
```

### 주요 Prometheus 메트릭

```promql
# 클러스터 상태 (0=green, 1=yellow, 2=red)
elasticsearch_cluster_health_status{color="green"}

# Heap 사용률
elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"} * 100

# 인덱싱 속도 (ops/s)
rate(elasticsearch_indices_indexing_index_total[5m])

# 검색 속도 (ops/s)
rate(elasticsearch_indices_search_query_total[5m])

# Write rejected 건수
elasticsearch_thread_pool_rejected_count{name="write"}

# 미할당 샤드
elasticsearch_cluster_health_unassigned_shards
```

### Kibana Alerting 설정

```bash
# Watcher로 클러스터 RED 알람 (Kibana Alerting 대안)
curl -X PUT "${ES_HOST}/_watcher/watch/cluster-red-alert" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "trigger": {
      "schedule": { "interval": "1m" }
    },
    "input": {
      "http": {
        "request": {
          "url": "https://localhost:9200/_cluster/health",
          "auth": { "basic": { "username": "elastic", "password": "{{ctx.metadata.password}}" } }
        }
      }
    },
    "condition": {
      "compare": { "ctx.payload.status": { "eq": "red" } }
    },
    "actions": {
      "notify_slack": {
        "webhook": {
          "scheme": "https",
          "host": "hooks.slack.com",
          "path": "/services/...",
          "method": "post",
          "body": "{\"text\": \"Elasticsearch cluster is RED!\"}"
        }
      }
    }
  }'
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| Stack Monitoring 데이터 없음 | Metricbeat 미설정 | ECK `monitoring` 블록 확인 |
| Prometheus 메트릭 수집 안 됨 | Exporter 인증 오류 | Secret 마운트 및 TLS 설정 확인 |
| `_cat/nodes`에서 노드 누락 | 노드 이탈 | pod 상태 및 로그 확인 |

---

## 모니터링 & 검증

```bash
# 클러스터 통계 (전체 요약)
curl -X GET "${ES_HOST}/_cluster/stats?human&pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 노드별 세부 통계
curl -X GET "${ES_HOST}/_nodes/stats?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 실시간 thread pool 모니터링
watch -n 5 'curl -s -u "elastic:${ES_PASSWORD}" -k \
  "${ES_HOST}/_cat/thread_pool/write,search?v"'
```

### 주요 메트릭 임계값

| 메트릭 | 경고 | 위험 | 설명 |
|--------|------|------|------|
| `cluster.status` | yellow | red | 클러스터 헬스 |
| `jvm.heap_used_percent` | 75% | 85% | JVM Heap 사용률 |
| `disk.used_percent` | 75% | 85% | 디스크 사용률 |
| `thread_pool.write.rejected` | > 0 | - | write 큐 거부 |
| `unassigned_shards` | > 0 | - | 미할당 샤드 |

---

## TIP

- `_cat/nodes?v&s=heap.percent:desc` 로 Heap 높은 노드 우선 확인
- Kibana의 Stack Monitoring은 별도 모니터링 클러스터를 두는 것이 프로덕션 권장 (모니터링 대상 클러스터 부하 분리)
- Prometheus + Grafana 조합: `elasticsearch-exporter` + Grafana 대시보드 ID `266` (Elasticsearch Overview) 사용

**관련 링크**:
- [ECK Stack Monitoring](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-stack-monitoring.html)
- [Elasticsearch Exporter](https://github.com/prometheus-community/elasticsearch_exporter)
