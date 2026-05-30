# 보안 체크리스트

## 인증 정보

- [ ] 하드코딩된 비밀번호 없음 (`${ES_PASSWORD}` 환경 변수 사용)
- [ ] API Key, 인증서 파일 경로가 코드에 포함되지 않음
- [ ] Kubernetes Secret 참조만 사용
- [ ] `.gitignore`에 `.env`, `*.key`, `*.crt` 포함

## TLS/암호화

- [ ] HTTP 레이어 TLS 활성화 (ECK 기본값)
- [ ] Transport 레이어 TLS 활성화 (ECK 기본값)
- [ ] `-k` / `--insecure` 플래그는 개발 환경만 허용
- [ ] 인증서 갱신 정책 확인 (ECK 자동 갱신)

## 접근 제어

- [ ] `elastic` 슈퍼유저 직접 사용 최소화
- [ ] 최소 권한 원칙 (인덱스별 권한 분리)
- [ ] API Key 만료 기간 설정 (`expiration` 필드)
- [ ] Field-Level Security로 민감 데이터 보호

## 네트워크

- [ ] Elasticsearch 포트(9200, 9300) 외부 노출 없음
- [ ] Kibana만 Ingress로 노출
- [ ] NetworkPolicy로 Pod 간 트래픽 제한

## 데이터 보호

- [ ] 민감 데이터 인덱스는 `index: false` 또는 field-level 보안 적용
- [ ] PII 데이터 필드에 `doc_values: false` 검토
- [ ] 스냅샷 저장소(S3) 접근 권한 최소화

## 로깅 및 감사

- [ ] Audit log 활성화 (Elasticsearch Security Audit)
- [ ] Slow log 임계값 설정 (search/indexing)
- [ ] 로그에 인증 정보 포함되지 않음
