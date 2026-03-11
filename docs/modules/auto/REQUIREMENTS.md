# PP Auto (n8n) — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims (예정)

```yaml
claims:
  - type: database
    config:
      name: auto
  - type: objectStorage
    config:
      bucket: auto
  - type: smtp
    config:
      domain: auto
  - type: auth
    config:
      clientId: auto
```

## 2. 인증 (제1원칙)

### 듀얼 패턴 — OIDC + LDAP 병행

- `N8N_OIDC_*` → Keycloak OIDC (기본)
- `N8N_SSO_LDAP_LOGIN_ENABLED=true` → AD LDAP 보조
- n8n Enterprise LDAP 기능 활성화

## 3. 스토리지 (제7원칙)

- n8n binary data를 S3에 저장
- `N8N_DEFAULT_BINARY_DATA_MODE=s3`

## 4. 래핑 방식

- n8n 공식 이미지 기반 (소스 빌드 불필요)
- entrypoint.sh에서 PRC 환경변수 → n8n 내부 설정 변환
- OIDC/LDAP 설정은 n8n Settings DB에 UPSERT

## 5. 환경변수

| 변수 | PRC Claim | 용도 |
|------|-----------|------|
| `DB_*` | database | PostgreSQL |
| `N8N_*_S3_*` | objectStorage | RustFS |
| `N8N_OIDC_*` | auth | Keycloak |
| `N8N_SSO_LDAP_*` | directory | AD |
| `SMTP_*` | smtp | 메일 |

## 6. 빌드/배포

```bash
cd PolyON-Auto
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-auto:v{VERSION} --push .

kubectl set image deploy/polyon-auto auto=jupitertriangles/polyon-auto:v{VERSION} -n polyon
```
