# 인가 (Authorization, RBAC)

## 개요

Elasticsearch RBAC(Role-Based Access Control)는 역할(Role)을 정의하고 사용자에게 할당하여 클러스터, 인덱스, 필드 수준의 접근을 제어한다. 최소 권한 원칙을 적용하여 각 서비스가 필요한 권한만 갖도록 한다.

| 항목 | 값 |
|------|-----|
| 관련 API | `PUT /_security/role/{name}`, `PUT /_security/user/{username}` |
| 적용 환경 | Elasticsearch 8.x |
| 관련 개념 | [인증](authentication.md) |

---

## 설명

### 역할 (Role) 생성

```bash
# 읽기 전용 역할
curl -X PUT "${ES_HOST}/_security/role/readonly-role" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "cluster": ["monitor"],
    "indices": [
      {
        "names": ["logs-app-*", "metrics-*"],
        "privileges": ["read", "view_index_metadata"]
      }
    ]
  }'

# 인덱싱 전용 역할 (쓰기 + 인덱스 생성)
curl -X PUT "${ES_HOST}/_security/role/indexer-role" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "cluster": [],
    "indices": [
      {
        "names": ["logs-app-*"],
        "privileges": ["write", "create_index", "view_index_metadata"]
      }
    ]
  }'

# 운영자 역할 (클러스터 관리 + 인덱스 관리)
curl -X PUT "${ES_HOST}/_security/role/ops-role" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "cluster": [
      "monitor",
      "manage_ilm",
      "manage_index_templates",
      "manage_slm"
    ],
    "indices": [
      {
        "names": ["*"],
        "privileges": ["manage", "read", "write"]
      }
    ]
  }'
```

### 주요 권한 목록

**Cluster 권한**:
| 권한 | 설명 |
|------|------|
| `monitor` | 클러스터 헬스, 노드 stats 조회 |
| `manage` | 모든 클러스터 관리 작업 |
| `manage_ilm` | ILM 정책 관리 |
| `manage_index_templates` | 인덱스 템플릿 관리 |
| `manage_slm` | SLM 정책 관리 |

**Index 권한**:
| 권한 | 설명 |
|------|------|
| `read` | 검색, get |
| `write` | 인덱싱, 업데이트, 삭제 |
| `create_index` | 인덱스 생성 |
| `manage` | alias, 설정 변경 등 모든 관리 |
| `view_index_metadata` | 매핑, 설정 조회 |
| `delete_index` | 인덱스 삭제 |

### Field-Level Security (필드 수준 보안)

```bash
# 특정 필드만 읽을 수 있는 역할
curl -X PUT "${ES_HOST}/_security/role/restricted-reader-role" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "indices": [
      {
        "names": ["logs-app-*"],
        "privileges": ["read"],
        "field_security": {
          "grant": ["@timestamp", "message", "log.level", "service.name"]
        }
      }
    ]
  }'

# 특정 필드를 제외한 나머지 읽기
curl -X PUT "${ES_HOST}/_security/role/no-pii-role" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "indices": [
      {
        "names": ["user-events-*"],
        "privileges": ["read"],
        "field_security": {
          "grant": ["*"],
          "except": ["user.email", "user.phone", "payment.card_number"]
        }
      }
    ]
  }'
```

### Document-Level Security (문서 수준 보안)

```bash
# 특정 조건의 문서만 읽기 허용
curl -X PUT "${ES_HOST}/_security/role/service-a-only-role" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "indices": [
      {
        "names": ["logs-app-*"],
        "privileges": ["read"],
        "query": "{\"term\": {\"service.name\": \"service-a\"}}"
      }
    ]
  }'
```

### 역할 확인

```bash
# 역할 목록
curl -X GET "${ES_HOST}/_security/role?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 특정 역할 상세
curl -X GET "${ES_HOST}/_security/role/readonly-role?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 사용자의 유효 권한 확인
curl -X GET "${ES_HOST}/_security/user/readonly-user/_privileges?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `403 Forbidden` on specific index | 인덱스 패턴 불일치 | 역할의 `names` 패턴 확인 |
| 집계는 되지만 문서 조회 불가 | `read` 권한만 없음 (이상) | 역할 재확인, field-level security 확인 |
| 필드가 응답에서 누락 | Field-Level Security 적용 | `grant` 목록에 해당 필드 추가 |

### 권한 디버깅

**증상**: `403 Forbidden` 오류 발생

**원인**: 역할 권한 부족 또는 인덱스 패턴 불일치

**해결책**:
```bash
# 1. 사용자 현재 권한 확인
curl -X GET "${ES_HOST}/_security/user/my-user/_privileges?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 2. 특정 작업 권한 여부 확인 (has_privileges API)
curl -X POST "${ES_HOST}/_security/user/my-user/_has_privileges" \
  -H "Content-Type: application/json" \
  -u "elastic:${ES_PASSWORD}" -k \
  -d '{
    "index": [
      {
        "names": ["logs-app-production"],
        "privileges": ["read", "write"]
      }
    ]
  }'
```

---

## 모니터링 & 검증

```bash
# 역할 목록 및 사용자 연결
curl -X GET "${ES_HOST}/_security/role?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 사용자별 역할 확인
curl -X GET "${ES_HOST}/_security/user?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 현재 인증 및 권한 확인
curl -X GET "${ES_HOST}/_security/_authenticate?pretty" \
  -u "readonly-user:SecurePassword123!" -k
```

---

## TIP

- 역할 이름은 `{서비스명}-{권한수준}` 형식으로 명명: `api-server-indexer`, `dashboard-reader`
- 인덱스 패턴에 와일드카드 `*` 남용 금지 → 구체적인 패턴 사용
- Field-Level Security와 Document-Level Security는 성능 비용이 있으므로 꼭 필요한 경우만 사용

**관련 링크**:
- [Security Privileges](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html)
- [Field and Document Level Security](https://www.elastic.co/guide/en/elasticsearch/reference/current/field-and-document-access-control.html)
