# Review Document

주어진 파일(`$ARGUMENTS`)을 검토하고 품질 피드백을 제공합니다.

## 형식

`/review-doc {file-path}`

예시: `/review-doc docs/guides/operations/index-management.md`

## 검토 항목

1. **완성도**: 5개 섹션 모두 포함 여부
2. **코드 품질**: 복사-붙여넣기 실행 가능 여부
3. **트러블슈팅**: 증상 → 원인 → 해결책 형식 준수
4. **모니터링**: API 확인 명령어 포함 여부
5. **컨벤션**: `docs/rules/elasticsearch-conventions.md` 준수 여부
6. **보안**: `docs/rules/security-checklist.md` 준수 여부
