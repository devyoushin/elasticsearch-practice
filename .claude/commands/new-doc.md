# New Document

주어진 인수(`$ARGUMENTS`)를 기반으로 새 Elasticsearch 문서를 생성합니다.

## 형식

`/new-doc {category}/{topic}`

예시: `/new-doc operations/reindex`

## 작업 순서

1. `docs/{category}/{topic}.md` 경로로 파일 생성
2. `docs/templates/service-doc.md` 템플릿 기반으로 스캐폴딩
3. 섹션 5개 포함: 개요, 설명, 트러블슈팅, 모니터링 & 검증, TIP
4. `CLAUDE.md`의 카테고리 목록 업데이트

## 규칙

- `docs/rules/doc-writing.md` 준수
- `docs/rules/elasticsearch-conventions.md` 준수
- 코드 블록: `bash` (curl/kubectl), `json` (ES API 요청/응답), `yaml` (ECK CR)
