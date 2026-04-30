# Elasticsearch 컨벤션

## REST API 호출

- `curl`에 `\` 줄 이음 사용
- 인증 정보: 환경 변수 사용 (`${ES_PASSWORD}`)
- TLS: `-k` (개발/랩 환경), `--cacert` (프로덕션)
- 출력: `?pretty` 또는 `?v` 파라미터

```bash
# 환경 변수 설정 (ECK 클러스터)
export ES_HOST="https://my-cluster-es-http.elasticsearch.svc.cluster.local:9200"
export ES_PASSWORD=$(kubectl get secret my-cluster-es-elastic-user \
  -n elasticsearch \
  -o go-template='{{.data.elastic | base64decode}}')

# API 호출 패턴
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" \
  -k
```

## kubectl exec 패턴 (ECK 환경)

```bash
# master pod에서 직접 실행
kubectl exec -it -n elasticsearch \
  $(kubectl get pod -n elasticsearch \
    -l elasticsearch.k8s.elastic.co/cluster-name=my-cluster \
    -l elasticsearch.k8s.elastic.co/node-master=true \
    --output=jsonpath='{.items[0].metadata.name}') \
  -- bash
```

## ECK Custom Resource 표준

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: my-cluster
  namespace: elasticsearch
spec:
  version: 8.15.0
  nodeSets:
    - name: masters
      count: 3
      config:
        node.roles: ["master"]
      podTemplate:
        spec:
          initContainers:
            - name: sysctl
              securityContext:
                privileged: true
                runAsUser: 0
              command: ["sh", "-c", "sysctl -w vm.max_map_count=262144"]
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 4Gi
                  cpu: "2"
                limits:
                  memory: 4Gi
              env:
                - name: ES_JAVA_OPTS
                  value: "-Xms2g -Xmx2g"
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 50Gi
            storageClassName: gp3
```

## 인덱스 템플릿 표준

```json
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "@timestamp": { "type": "date" }
      }
    }
  },
  "priority": 200,
  "_meta": {
    "description": "Template description"
  }
}
```

## 네임스페이스 규칙

| 네임스페이스 | 용도 |
|-------------|------|
| `elastic-system` | ECK 오퍼레이터 |
| `elasticsearch` | Elasticsearch 클러스터, Kibana |

## JVM 설정

- Heap 크기: 가용 메모리의 50%, 최대 31GB
- `-Xms`와 `-Xmx` 동일하게 설정 (GC 오버헤드 방지)
- G1GC 사용 (Elasticsearch 8.x 기본값)

## 보안 기본값

- TLS: 항상 활성화 (ECK 자동 설정)
- 인증: `elastic` 슈퍼유저 초기 설정 후, 최소 권한 사용자 생성
- API Key: 서비스 간 통신에 사용자/비밀번호 대신 API Key 사용
- `dynamic: "strict"`: 의도치 않은 필드 자동 생성 방지
