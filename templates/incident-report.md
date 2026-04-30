# 인시던트 보고서: {제목}

## 기본 정보

| 항목 | 내용 |
|------|------|
| 발생 일시 | YYYY-MM-DD HH:MM KST |
| 종료 일시 | YYYY-MM-DD HH:MM KST |
| 총 영향 시간 | XX분 |
| 심각도 | P1 / P2 / P3 |
| 영향 범위 | ... |
| 클러스터 상태 | RED / YELLOW |

## 요약

{2-3줄로 인시던트 요약}

## 타임라인

| 시간 | 이벤트 |
|------|--------|
| HH:MM | 알람 발생: cluster status RED |
| HH:MM | ... |
| HH:MM | 서비스 복구 완료 |

## 근본 원인

{5 Whys 분석}

1. **왜?** ...
2. **왜?** ...
3. **왜?** ...

## 즉각 대응

```bash
# 복구에 사용한 명령어
curl -X POST "${ES_HOST}/_cluster/reroute?retry_failed=true" \
  -u "elastic:${ES_PASSWORD}" -k
```

## 재발 방지

| 항목 | 담당 | 기한 |
|------|------|------|
| ... | ... | YYYY-MM-DD |

## 교훈

- ...
