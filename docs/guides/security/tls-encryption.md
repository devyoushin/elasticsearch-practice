# TLS 암호화

## 개요

Elasticsearch 8.x는 HTTP 레이어와 Transport 레이어 모두 TLS가 기본 활성화되어 있다. ECK 환경에서는 오퍼레이터가 인증서를 자동 생성·갱신하므로 인증서 관리 부담이 없다.

| 항목 | 값 |
|------|-----|
| 관련 API | `GET /_ssl/certificates` |
| 적용 환경 | Elasticsearch 8.x, ECK v2.x |
| 관련 개념 | [인증](authentication.md), [인가](authorization.md) |

---

## 설명

### ECK 자동 TLS 구조

ECK는 두 레이어의 TLS를 자동으로 관리한다:

| 레이어 | 목적 | 인증서 Secret |
|--------|------|--------------|
| HTTP 레이어 | 클라이언트 ↔ Elasticsearch REST API | `my-cluster-es-http-certs-public` |
| Transport 레이어 | 노드 간 클러스터 통신 | `my-cluster-es-transport-certs-public` |

```bash
# ECK가 생성한 인증서 Secret 확인
kubectl get secret -n elasticsearch | grep my-cluster-es

# HTTP 인증서 확인
kubectl get secret my-cluster-es-http-certs-public \
  -n elasticsearch -o yaml

# CA 인증서 추출 (클라이언트 신뢰 설정용)
kubectl get secret my-cluster-es-http-certs-public \
  -n elasticsearch \
  -o go-template='{{index .data "ca.crt" | base64decode}}' \
  > ca.crt
```

### CA 인증서를 사용한 curl 요청

```bash
# CA 인증서 추출
kubectl get secret my-cluster-es-http-certs-public \
  -n elasticsearch \
  -o go-template='{{index .data "ca.crt" | base64decode}}' \
  > /tmp/es-ca.crt

# CA 인증서로 검증 (프로덕션 권장)
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" \
  --cacert /tmp/es-ca.crt

# 개발 환경: 인증서 검증 생략 (-k)
curl -X GET "${ES_HOST}/_cluster/health?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### 커스텀 인증서 사용 (ECK)

자체 CA 또는 외부 인증서 사용이 필요한 경우:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: my-cluster
  namespace: elasticsearch
spec:
  version: 8.15.0
  http:
    tls:
      certificate:
        secretName: my-custom-tls-secret   # tls.crt, tls.key 포함
      selfSignedCertificate:
        disabled: true
```

```bash
# 커스텀 TLS Secret 생성
kubectl create secret tls my-custom-tls-secret \
  -n elasticsearch \
  --cert=server.crt \
  --key=server.key
```

### Transport TLS 검증

```bash
# Transport 레이어 인증서 확인 (노드 간 통신)
kubectl get secret my-cluster-es-transport-certs-public \
  -n elasticsearch -o yaml

# 현재 TLS 인증서 정보 확인 (Elasticsearch API)
curl -X GET "${ES_HOST}/_ssl/certificates?pretty" \
  -u "elastic:${ES_PASSWORD}" -k
```

### Kibana TLS 설정 (ECK)

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: my-cluster
  namespace: elasticsearch
spec:
  http:
    tls:
      selfSignedCertificate:
        disabled: false
        subjectAltNames:
          - dns: kibana.example.com
          - ip: 10.0.0.100
```

---

## 트러블슈팅

| 증상 | 원인 | 해결책 |
|------|------|--------|
| `SSL handshake failed` | 인증서 불일치, 만료 | 인증서 갱신, CA 신뢰 확인 |
| `certificate verify failed` | CA 인증서 미신뢰 | `--cacert` 또는 시스템 CA 스토어 추가 |
| 노드가 클러스터에 조인 실패 | Transport TLS 오류 | ECK pod 로그 확인, Secret 재생성 |

### 인증서 만료 확인

**증상**: `curl: SSL certificate problem: certificate has expired`

**원인**: TLS 인증서 만료 (ECK 자동 갱신이지만 수동 확인 필요)

**해결책**:
```bash
# 인증서 만료일 확인
kubectl get secret my-cluster-es-http-certs-public \
  -n elasticsearch \
  -o go-template='{{index .data "ca.crt" | base64decode}}' \
  | openssl x509 -noout -dates

# ECK 인증서 강제 갱신 (annotation 추가 → ECK가 감지하여 갱신)
kubectl annotate secret my-cluster-es-http-certs-internal \
  -n elasticsearch \
  update-cert=$(date +%s)
```

### HTTP vs HTTPS 연결 오류

**증상**: `curl: (52) Empty reply from server` 또는 `curl: (35) error:...`

**원인**: HTTP로 HTTPS 포트(9200)에 접속

**해결책**:
```bash
# 반드시 https:// 사용
export ES_HOST="https://my-cluster-es-http.elasticsearch.svc.cluster.local:9200"

# (잘못된 예)
# export ES_HOST="http://..."
```

---

## 모니터링 & 검증

```bash
# TLS 인증서 상세 정보
curl -X GET "${ES_HOST}/_ssl/certificates?pretty" \
  -u "elastic:${ES_PASSWORD}" -k

# 인증서 만료일 확인
openssl s_client -connect my-cluster-es-http.elasticsearch.svc.cluster.local:9200 \
  -servername my-cluster-es-http \
  </dev/null 2>/dev/null | openssl x509 -noout -dates

# ECK TLS 상태 확인
kubectl describe elasticsearch my-cluster -n elasticsearch \
  | grep -A 5 "Tls"
```

---

## TIP

- ECK는 기본적으로 1년 유효 인증서를 자동 갱신. 만료 4주 전 자동 갱신 시작
- 클라이언트 앱에서는 `-k` 대신 CA 인증서를 ConfigMap으로 주입하여 사용 권장
- Transport 레이어 TLS는 ECK 환경에서 비활성화 불가 (보안 설계 원칙)

**관련 링크**:
- [ECK TLS 설정](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-tls-certificates.html)
- [Elasticsearch TLS 설정](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html#ssl-tls-settings)
