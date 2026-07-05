# elasticsearch-practice

EKS와 ECK 기준으로 Elasticsearch를 운영하기 위한 개인 학습 공간입니다.

## 어디서 시작할까

- 문서 지도: `docs/README.md`
- 첫 문서: `docs/01-core-concepts/cluster-architecture.md`
- 운영 보조 자료: `ops/README.md`
- AI 작업 지침: `CLAUDE.md`

## 구조

| 경로 | 내용 |
|------|------|
| `docs/` | Elasticsearch 개념, 운영, 성능, 보안, 관측 문서 |
| `docs/90-standards/` | 문서 작성 및 운영 규칙 |
| `docs/91-templates/` | 재사용 문서 템플릿 |
| `docs/99-agents/` | Claude 에이전트 프롬프트 |
| `ops/` | ECK, Kibana, index template 설정 예시 |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Elasticsearch | 8.x |
| Operator | ECK v2.x |
