# elasticsearch-practice

EKS + ECK 기준으로 Elasticsearch 8.x 운영 지식을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/guides/core-concepts/cluster-architecture.md`
- 전체 흐름: 핵심 개념 -> 운영 -> 성능 -> 보안 -> 관측
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
elasticsearch-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── guides/     # 학습 문서
│   ├── rules/      # 작성/운영 규칙
│   ├── templates/  # 재사용 템플릿
│   └── agents/     # Claude 에이전트 프롬프트
└── ops/
    └── config/     # ECK 설정 예시
```

## 학습 경로

| 단계 | 위치 |
|------|------|
| 핵심 개념 | `docs/guides/core-concepts/` |
| 운영 | `docs/guides/operations/` |
| 성능 | `docs/guides/performance/` |
| 보안 | `docs/guides/security/` |
| 관측 | `docs/guides/observability/` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Elasticsearch | 8.x |
| Operator | ECK v2.x |
| Namespace | `elasticsearch` |
| Cluster | `my-cluster` |
