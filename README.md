# Elasticsearch Practice

ECK(Elastic Cloud on Kubernetes) 기반 Elasticsearch 8.x 운영 지식 베이스입니다.

## 환경 정보

| 항목 | 값 |
|------|-----|
| Platform | EKS (Kubernetes) |
| Elasticsearch | 8.x |
| Operator | ECK (Elastic Cloud on Kubernetes) v2.x |
| Kibana | 8.x |
| Namespace (operator) | `elastic-system` |
| Namespace (cluster) | `elasticsearch` |
| Cluster name | `my-cluster` |

## 디렉토리 구조

```
elasticsearch-practice/
├── .claude/               # Claude Code 설정 및 커스텀 커맨드
├── agents/                # 전문 에이전트 정의
├── rules/                 # 문서 작성 및 코드 컨벤션
├── templates/             # 문서 템플릿
├── config/                # ECK 클러스터 설정 예시
└── docs/                  # 운영 지식 문서
    ├── core-concepts/     # 핵심 개념
    ├── operations/        # 운영 작업
    ├── performance/       # 성능 최적화
    ├── security/          # 보안 설정
    └── observability/     # 모니터링 및 로그
```

## 학습 경로

### 1단계: 핵심 개념
- [클러스터 아키텍처](docs/core-concepts/cluster-architecture.md)
- [인덱스 설계](docs/core-concepts/index-design.md)
- [Query DSL](docs/core-concepts/query-dsl.md)

### 2단계: 운영
- [인덱스 관리](docs/operations/index-management.md)
- [클러스터 관리](docs/operations/cluster-management.md)
- [스냅샷 & 복구](docs/operations/snapshot-restore.md)

### 3단계: 성능 최적화
- [인덱싱 성능](docs/performance/indexing-performance.md)
- [검색 성능](docs/performance/search-performance.md)
- [리소스 튜닝](docs/performance/resource-tuning.md)

### 4단계: 보안
- [인증](docs/security/authentication.md)
- [인가 (RBAC)](docs/security/authorization.md)
- [TLS 암호화](docs/security/tls-encryption.md)

### 5단계: 관찰 가능성
- [모니터링](docs/observability/monitoring.md)
- [슬로우 로그](docs/observability/slow-logs.md)

## 커스텀 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/new-doc` | 새 문서 스캐폴딩 |
| `/new-runbook` | 운영 런북 생성 |
| `/review-doc` | 문서 품질 검토 |
| `/add-troubleshooting` | 트러블슈팅 섹션 추가 |
| `/search-kb` | 지식 베이스 검색 |
