# Search Knowledge Base

지식 베이스에서 키워드(`$ARGUMENTS`)를 검색합니다.

## 형식

`/search-kb {keyword}`

예시: `/search-kb "shard allocation"`

## 작업 순서

1. `docs/` 디렉토리 전체를 키워드로 검색
2. 관련 파일 목록과 관련 섹션 요약 반환
3. 가장 관련성 높은 문서 3개를 상세 내용 포함하여 출력
