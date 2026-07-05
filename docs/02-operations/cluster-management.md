# 클러스터 관리

## 개요

ECK 환경에서의 Elasticsearch 클러스터 운영 작업을 다룬다. 노드 추가/제거, Rolling Restart, 샤드 재배치, 업그레이드 등 운영 중 발생하는 주요 작업의 절차와 주의사항을 정리한다.

| 항목 | 값 |
|------|-----|
| 관련 API | `/_cluster/settings`, `/_cluster/reroute`, `/_nodes` |
| 적용 환경 | Elasticsearch 8.x, ECK v2.x |
| 관련 개념 | [클러스터 아키텍처](../01-core-concepts/cluster-architecture.md) |

---

## 설명

### 노드 추가 (Scale Out)

ECK 환경에서는 `spec.nodeSets[*].count`를 변경하면 ECK 오퍼레이터가 자동으로 처리한다.

```yaml
# elasticsearch-cluster.yaml 수정
spec:
  nodeSets:
    - name: data-hot
      count: 5  # 3 → 5 로 증설
```

```bash
# 변경 적용
kubectl apply -f ops/config/elasticsearch-cluster.yaml

# 노드 추가 확인
kubectl get pods -n elasticsearch -w
curl -X GET "${ES_HOST}/_cat/nodes?v" -u "elastic:${ES_PASSWORD}" -k
```

### 노드 제거 (Scale In)

샤드 이동이 완료된 후 노드가 제거되므로 데이터 손실 없음. 단, 최소 샤드 복제본보다 노드가 적어지지 않도록 주의.

```bash
# 제거 전: 샤드 재배치 우선 실행
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "transient": {
      "cluster.routing.allocation.exclude._name": "instance-data-hot-2"
    }
  }'

# 샤드 이동 완료 확인 (RELOCATING 없어야 함)
curl -X GET "${ES_HOST}/_cat/shards?v&s=state" \
  -u "elastic:${ES_PASSWORD}" -k

# count 줄이기
kubectl patch elasticsearch my-cluster -n elasticsearch \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/nodeSets/1/count","value":4}]'

# 제거 후 설정 초기화
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "transient": { "cluster.routing.allocation.exclude._name": null } }'
```

### Rolling Restart (ECK 자동)

ECK는 `Elasticsearch` CR 변경 시 자동으로 Rolling Restart를 수행한다.

```bash
# ECK rolling restart 상태 모니터링
kubectl get elasticsearch my-cluster -n elasticsearch -w

# 각 pod의 재시작 상태 확인
kubectl get pods -n elasticsearch -l elasticsearch.k8s.elastic.co/cluster-name=my-cluster

# 재시작 중 샤드 상태 (RELOCATING 정상)
curl -X GET "${ES_HOST}/_cat/shards?v&h=index,shard,prirep,state,node" \
  -u "elastic:${ES_PASSWORD}" -k
```

수동 롤링 재시작이 필요한 경우 (ECK 외부 설정 변경):

```bash
# 재시작 전: 샤드 자동 재배치 비활성화 (불필요한 이동 방지)
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "transient": {
      "cluster.routing.allocation.enable": "primaries"
    }
  }'

# pod 재시작 (한 번에 하나씩)
kubectl rollout restart statefulset/my-cluster-es-masters -n elasticsearch

# 재시작 완료 후 샤드 재배치 활성화
curl -X PUT "${ES_HOST}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "transient": { "cluster.routing.allocation.enable": null } }'

# 샤드 복구 가속
curl -X POST "${ES_HOST}/_cluster/reroute?retry_failed=true" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 샤드 재배치

```bash
# 특정 샤드 강제 이동
curl -X POST "${ES_HOST}/_cluster/reroute" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "commands": [
      {
        "move": {
          "index": "my-index",
          "shard": 0,
          "from_node": "instance-data-hot-0",
          "to_node":   "instance-data-hot-1"
        }
      }
    ]
  }'

# UNASSIGNED 샤드 강제 할당 (데이터 손실 위험! 최후 수단)
curl -X POST "${ES_HOST}/_cluster/reroute" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "commands": [
      {
        "allocate_stale_primary": {
          "index": "my-index",
          "shard": 0,
          "node": "instance-data-hot-0",
          "accept_data_loss": true
        }
      }
    ]
  }'
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| ECK rolling restart 무한 대기 | pod readiness probe 실패 | `kubectl describe pod` 로 원인 확인 |
| 노드 제거 후 샤드 UNASSIGNED | 복제본 노드 부족 | 노드 수 유지 또는 복제본 수 줄이기 |
| 클러스터 Red 상태 지속 | 주 샤드 할당 불가 | `_cluster/allocation/explain` 확인 |

### ECK pod가 Pending 상태

**증상**: `kubectl get pods`에서 pod가 `Pending` 상태

**원인**: 리소스 부족 (CPU/메모리), PVC 생성 실패

**해결책**:
```bash
# Pod 이벤트 확인
kubectl describe pod my-cluster-es-data-hot-0 -n elasticsearch

# Node 리소스 확인
kubectl describe nodes | grep -A 5 "Allocated resources"

# PVC 상태 확인
kubectl get pvc -n elasticsearch
```

---

## 모니터링 & 검증

```bash
# 클러스터 헬스 및 샤드 수
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 노드별 디스크 사용량
curl -X GET "${ES_HOST}/_cat/nodes?v&h=name,disk.used,disk.avail,disk.used_percent" \
  -u "elastic:${ES_PASSWORD}" -k

# 클러스터 설정 확인
curl -X GET "${ES_HOST}/_cluster/settings?pretty&include_defaults=true" \
  -u "elastic:${ES_PASSWORD}" -k

# ECK 상태
kubectl get elasticsearch my-cluster -n elasticsearch -o wide
```

---

## TIP

- ECK 환경에서 `count` 증가는 무중단이지만, `count` 감소 전 반드시 샤드 이동 완료 확인
- `cluster.routing.allocation.enable: "primaries"` 설정은 롤링 재시작 시 임시로만 사용, 완료 후 반드시 `null`로 초기화
- `_cluster/reroute?retry_failed=true`는 일시적 할당 실패를 재시도하는 안전한 명령

**관련 링크**:
- [Cluster Management 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html)
- [ECK Operations](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-node-configuration.html)
