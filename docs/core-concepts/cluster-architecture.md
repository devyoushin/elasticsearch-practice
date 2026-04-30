# 클러스터 아키텍처

## 개요

Elasticsearch 클러스터는 여러 노드(Node)가 협력하여 데이터를 분산 저장하고 검색하는 시스템이다. 노드는 역할(role)에 따라 master, data, ingest, coordinating으로 구분되며, ECK 환경에서는 각 역할을 전용 노드셋(nodeSet)으로 분리하는 것이 권장된다.

| 항목 | 값 |
|------|-----|
| 관련 API | `/_cat/nodes`, `/_cluster/health`, `/_cluster/state` |
| 적용 환경 | Elasticsearch 8.x, ECK v2.x |
| 관련 개념 | [인덱스 설계](index-design.md) |

---

## 설명

### 노드 역할 (Node Roles)

| 역할 | 설명 | 권장 메모리 |
|------|------|------------|
| `master` | 클러스터 상태 관리, 샤드 할당 결정 | 2~4GB |
| `data_hot` | 최신 데이터 인덱싱 및 검색 (고성능 디스크) | 8~32GB |
| `data_warm` | 오래된 데이터 검색 (대용량 디스크) | 4~16GB |
| `data_cold` | 거의 검색되지 않는 데이터 (최대 용량) | 2~8GB |
| `ingest` | 인덱싱 전 데이터 전처리 (Ingest Pipeline) | 4~8GB |
| `coordinating` | 검색/인덱싱 요청 분산, 결과 취합 | 4~8GB |
| `ml` | Machine Learning 작업 | 4~16GB |

### 클러스터 상태 (Cluster State)

| 상태 | 의미 |
|------|------|
| `green` | 모든 주(primary) 샤드와 복제(replica) 샤드 할당 완료 |
| `yellow` | 주 샤드는 할당, 일부 복제 샤드 미할당 (단일 노드 클러스터에서 정상) |
| `red` | 일부 주 샤드 미할당 → 해당 인덱스의 데이터 접근 불가 |

### 마스터 선출 (Master Election)

- 마스터 노드는 최소 `(n/2)+1`개가 필요 (n = master-eligible node 수)
- 3개 마스터 노드 권장: 1개 장애 시에도 quorum(2) 유지
- ECK는 `discovery.seed_hosts`와 `cluster.initial_master_nodes`를 자동 설정

### ECK 노드셋 구성 예시

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: my-cluster
  namespace: elasticsearch
spec:
  version: 8.15.0
  nodeSets:
    # 전용 마스터 노드
    - name: masters
      count: 3
      config:
        node.roles: ["master"]
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 2Gi
                limits:
                  memory: 2Gi
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms1g -Xmx1g"
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 10Gi
            storageClassName: gp3

    # Hot data 노드
    - name: data-hot
      count: 3
      config:
        node.roles: ["data_hot", "data_content", "ingest"]
        node.attr.data: hot
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 8Gi
                limits:
                  memory: 8Gi
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms4g -Xmx4g"
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 500Gi
            storageClassName: gp3
```

### 샤드와 복제본 (Shards & Replicas)

- **Primary Shard**: 데이터 원본 저장, 인덱싱 처리
- **Replica Shard**: Primary의 복사본, 검색 부하 분산 및 장애 복구
- Primary와 Replica는 **반드시 다른 노드**에 배치됨

```bash
# 샤드 배치 확인
curl -X GET "${ES_HOST}/_cat/shards/my-index?v&h=index,shard,prirep,state,node" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 모범 사례

- 프로덕션: 전용 마스터 3개 + 데이터 노드 분리
- `vm.max_map_count=262144` initContainer로 설정 필수
- 노드당 Heap 크기 = 가용 메모리의 50%, 최대 31GB

### 안티패턴

- 단일 노드 클러스터 (SPOF)
- 마스터와 데이터 역할 혼용 (대규모 클러스터)
- Heap `-Xms`와 `-Xmx` 불일치 설정

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `cluster status: RED` | 주 샤드 미할당 | `POST /_cluster/reroute?retry_failed=true` |
| 마스터 선출 실패 | quorum 부족 (2개 이하 마스터) | 마스터 노드 복구 또는 재기동 |
| 노드가 클러스터 조인 실패 | `discovery.seed_hosts` 오류, TLS 불일치 | ECK pod 로그 확인, Secret 재생성 |

### 샤드 할당 실패 진단

**증상**: 인덱스가 yellow/red 상태

**원인**: 디스크 부족, 노드 부족, 할당 제외 설정

**해결책**:
```bash
# 할당 실패 원인 확인
curl -X GET "${ES_HOST}/_cluster/allocation/explain?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 강제 재시도
curl -X POST "${ES_HOST}/_cluster/reroute?retry_failed=true" \
  -u "elastic:${ES_PASSWORD}" -k

# 디스크 워터마크 임시 완화 (긴급 시)
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "transient": {
      "cluster.routing.allocation.disk.watermark.low": "90%",
      "cluster.routing.allocation.disk.watermark.high": "95%"
    }
  }'
```

---

## 모니터링 & 검증

```bash
# 클러스터 헬스
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 노드 목록 및 역할 확인
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,node.role,master" \
  -u "elastic:${ES_PASSWORD}" -k

# 마스터 노드 확인
curl -X GET "${ES_HOST}/_cat/master?v" \
  -u "elastic:${ES_PASSWORD}" -k

# ECK 클러스터 상태 확인
kubectl get elasticsearch -n elasticsearch
kubectl get pods -n elasticsearch -l elasticsearch.k8s.elastic.co/cluster-name=my-cluster
```

### 주요 메트릭

| 메트릭 | 정상 범위 | 설명 |
|--------|-----------|------|
| `cluster.status` | green | 전체 클러스터 상태 |
| `cluster.number_of_nodes` | 설정값 | 활성 노드 수 |
| `cluster.unassigned_shards` | 0 | 미할당 샤드 수 |
| `cluster.active_primary_shards` | 전체 주 샤드 수 | 활성 주 샤드 |

---

## TIP

- ECK 환경에서 노드 수 조정은 `spec.nodeSets[*].count` 변경으로 처리 (rolling restart 자동)
- `_cat/nodes?v&h=name,node.role,master` 출력의 `master` 컬럼: `*`가 현재 마스터 노드
- Heap 75% 이상 지속 시 GC 압박 → 데이터 노드 증설 또는 ILM 정책으로 데이터 이동

**관련 링크**:
- [ECK 공식 문서](https://www.elastic.co/guide/en/cloud-on-k8s/current/)
- [Elasticsearch Node Roles](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)
