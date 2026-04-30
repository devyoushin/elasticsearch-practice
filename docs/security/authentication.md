# 인증 (Authentication)

## 개요

Elasticsearch 8.x에서는 보안이 기본 활성화되어 있다. ECK 환경에서는 `elastic` 슈퍼유저가 자동 생성되며, 운영에서는 최소 권한 원칙에 따라 전용 사용자를 생성하여 사용한다.

| 항목 | 값 |
|------|-----|
| 관련 API | `POST /_security/user/{username}`, `GET /_security/user` |
| 적용 환경 | Elasticsearch 8.x, ECK v2.x |
| 관련 개념 | [인가](authorization.md), [TLS 암호화](tls-encryption.md) |

---

## 설명

### elastic 슈퍼유저 비밀번호 확인 (ECK)

```bash
# ECK가 자동 생성한 elastic 사용자 비밀번호 확인
kubectl get secret my-cluster-es-elastic-user \
  -n elasticsearch \
  -o go-template='{{.data.elastic | base64decode}}'

# 환경 변수로 저장
export ES_PASSWORD=$(kubectl get secret my-cluster-es-elastic-user \
  -n elasticsearch \
  -o go-template='{{.data.elastic | base64decode}}')

export ES_HOST="https://my-cluster-es-http.elasticsearch.svc.cluster.local:9200"

# 접속 확인
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 사용자 생성 (Native Realm)

```bash
# 읽기 전용 사용자 생성
curl -X POST "${ES_HOST}/_security/user/readonly-user" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "password": "SecurePassword123!",
    "roles": ["readonly-role"],
    "full_name": "Read Only User",
    "email": "readonly@example.com",
    "enabled": true
  }'

# 인덱싱 전용 사용자 생성
curl -X POST "${ES_HOST}/_security/user/indexer-user" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "password": "SecurePassword456!",
    "roles": ["indexer-role"],
    "enabled": true
  }'

# 사용자 목록 확인
curl -X GET "${ES_HOST}/_security/user?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### API Key 생성 (서비스 간 인증 권장)

사용자/비밀번호 대신 API Key를 사용하면 자격증명 노출 위험이 낮고 만료 기간 설정이 가능하다.

```bash
# API Key 생성
curl -X POST "${ES_HOST}/_security/api_key" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "name": "my-service-api-key",
    "expiration": "30d",
    "role_descriptors": {
      "my-service-role": {
        "cluster": ["monitor"],
        "indices": [
          {
            "names": ["logs-app-*"],
            "privileges": ["write", "create_index", "view_index_metadata"]
          }
        ]
      }
    }
  }'

# 응답: id와 api_key를 base64로 인코딩하여 사용
# Authorization: ApiKey {base64(id:api_key)}

# API Key로 요청
API_KEY="base64_encoded_id_and_key"
curl -X GET "${ES_HOST}/_cluster/health" \
  -H "Authorization: ApiKey ${API_KEY}" \
  -k

# API Key 목록 확인
curl -X GET "${ES_HOST}/_security/api_key?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# API Key 만료/무효화
curl -X DELETE "${ES_HOST}/_security/api_key" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "name": "my-service-api-key" }'
```

### 비밀번호 변경

```bash
# elastic 사용자 비밀번호 변경
curl -X POST "${ES_HOST}/_security/user/elastic/_password" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{ "password": "NewSecurePassword789!" }'

# ECK Secret 업데이트 (ECK가 자동 감지하여 적용)
kubectl create secret generic my-cluster-es-elastic-user \
  -n elasticsearch \
  --from-literal=elastic=NewSecurePassword789! \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `401 Unauthorized` | 잘못된 자격증명 | 비밀번호 재확인, Secret 갱신 |
| `403 Forbidden` | 권한 부족 | 사용자 역할 확인, 권한 추가 |
| API Key 사용 불가 | Key 만료 또는 무효화 | 새 API Key 생성 |

### elastic 비밀번호 분실

**증상**: `elastic` 슈퍼유저 비밀번호를 알 수 없음

**원인**: Secret 값 분실 또는 ECK 재설치

**해결책**:
```bash
# ECK Secret에서 다시 확인
kubectl get secret my-cluster-es-elastic-user \
  -n elasticsearch -o yaml

# Secret이 없다면 elasticsearch-reset-password 도구 사용 (pod 내부)
kubectl exec -it -n elasticsearch my-cluster-es-masters-0 \
  -- bin/elasticsearch-reset-password -u elastic -i
```

---

## 모니터링 & 검증

```bash
# 현재 인증 사용자 확인
curl -X GET "${ES_HOST}/_security/_authenticate?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 활성 API Key 목록
curl -X GET "${ES_HOST}/_security/api_key?active_only=true&pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 사용자 상세 정보
curl -X GET "${ES_HOST}/_security/user/readonly-user?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## TIP

- `elastic` 슈퍼유저는 장애 복구용으로만 사용, 일상 운영은 전용 사용자 사용
- API Key는 Kubernetes Secret에 저장하여 pod에서 환경 변수로 주입
- API Key `expiration` 설정 시 만료 알람을 Kibana Alerting으로 설정 권장

**관련 링크**:
- [Security API 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api.html)
- [API Key 인증](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-create-api-key.html)
