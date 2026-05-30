# Add Troubleshooting

기존 문서(`$ARGUMENTS`)에 트러블슈팅 항목을 추가합니다.

## 형식

`/add-troubleshooting {file-path} {symptom}`

예시: `/add-troubleshooting docs/operations/index-management.md "인덱스 생성 실패"`

## 작성 형식

트러블슈팅 항목은 다음 형식으로 작성:

```
### {증상}

**원인**: {근본 원인}

**해결책**:
\`\`\`bash
# 명령어
\`\`\`

**예방**: {재발 방지 방법}
```
