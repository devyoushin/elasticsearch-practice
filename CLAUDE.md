# Elasticsearch Practice - Project Guide

## 프로젝트 설정

| 항목 | 값 |
|------|-----|
| Elasticsearch | 8.x |
| Operator | ECK v2.x |
| Platform | EKS |
| Cluster name | `my-cluster` |
| Namespace (operator) | `elastic-system` |
| Namespace (cluster) | `elasticsearch` |
| Kibana namespace | `elasticsearch` |

## 파일 명명 규칙

- 위치: `docs/{category}/{topic}.md`
- 형식: kebab-case (소문자, 하이픈)
- 예시: `docs/operations/index-management.md`

## 카테고리

| 카테고리 | 디렉토리 | 내용 |
|----------|----------|------|
| 핵심 개념 | `core-concepts/` | 아키텍처, 인덱스 설계, Query DSL |
| 운영 | `operations/` | 인덱스/클러스터 관리, 스냅샷 |
| 성능 | `performance/` | 인덱싱/검색 최적화, 리소스 튜닝 |
| 보안 | `security/` | 인증, 인가, TLS |
| 관찰 가능성 | `observability/` | 모니터링, 슬로우 로그 |

## 문서 작성 원칙

1. **경험 기반**: 실제 운영에서 검증된 내용만 작성
2. **재현 가능한 코드**: 복사-붙여넣기로 바로 실행 가능
3. **근본 원인 중심 트러블슈팅**: 증상 → 원인 → 해결책
4. **한국어 + 영어 기술 용어**: 문서는 한국어, 코드/API 용어는 영어
5. **모니터링 포함 필수**: API 확인 명령어, 주요 메트릭 반드시 포함

## 필수 섹션 (5개)

1. **개요** - 무엇인지, 왜 중요한지, 언제 사용하는지
2. **설명** - 핵심 개념, 코드 예시, 모범 사례
3. **트러블슈팅** - 증상 → 원인 → 해결책
4. **모니터링 & 검증** - API 명령어, Kibana Dev Tools, 주요 메트릭
5. **TIP** - 현장 팁, 관련 링크

## 커스텀 커맨드

- `/new-doc {category}/{topic}` - 새 문서 생성
- `/new-runbook {topic}` - 런북 생성
- `/review-doc {file}` - 문서 검토
- `/add-troubleshooting {file}` - 트러블슈팅 추가
- `/search-kb {keyword}` - 지식 검색

## Backlog

- [ ] Ingest pipeline (processors, conditionals)
- [ ] Cross-cluster replication (CCR)
- [ ] Machine Learning (anomaly detection)
- [ ] Elasticsearch 클러스터 업그레이드 (ECK rolling)
- [ ] Fleet & Elastic Agent
- [ ] EQL (Event Query Language)
- [ ] TSDS (Time Series Data Stream)
- [ ] Watcher (알람 자동화)
