# PP BPMN (Operaton) — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims

```yaml
claims:
  - type: database
    config:
      name: bpmn
  - type: directory
    config:
      ou: bpmn
  - type: auth
    config:
      clientId: bpmn
  - type: smtp
    config:
      domain: bpmn
```

- objectStorage claim 없음 (BPMN 프로세스 데이터는 DB 저장)

## 2. 인증 (제1원칙)

### 듀얼 패턴 — OIDC + LDAP Identity Provider

| 경로 | 방식 | 비고 |
|------|------|------|
| 웹 UI | Keycloak OIDC (Spring Security OAuth2) | 기본 |
| Identity Service | LDAP (Operaton Identity Provider) | 사용자/그룹 조회 |

- Spring Security: `spring.security.oauth2.client.*`
- Operaton: `operaton.bpm.oauth2.*`
- LDAP Identity Provider: AD에서 사용자/그룹 동기화

## 3. 기술 스택

- Java 21 + Spring Boot 3.x
- Operaton (Camunda 7 포크)
- Maven 빌드
- PostgreSQL (JDBC)

## 4. 환경변수

| 변수 | PRC Claim | 용도 |
|------|-----------|------|
| `SPRING_DATASOURCE_URL` | database | JDBC PostgreSQL |
| `SPRING_DATASOURCE_USERNAME/PASSWORD` | database | DB 인증 |
| `LDAP_*` | directory | AD LDAP |
| `OIDC_CLIENT_ID/SECRET` | auth | Keycloak |
| `OIDC_ISSUER_URI` | auth.issuer | OIDC Discovery |
| `SMTP_*` | smtp | 메일 알림 |

## 5. 빌드/배포

```bash
cd PolyON-BPMN
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-bpmn:v{VERSION} --push .

kubectl set image deploy/polyon-bpmn bpmn=jupitertriangles/polyon-bpmn:v{VERSION} -n polyon
```
