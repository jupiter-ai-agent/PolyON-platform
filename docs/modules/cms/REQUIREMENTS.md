# PP CMS (Strapi) — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims

```yaml
claims:
  - type: database
    config:
      name: cms
  - type: objectStorage
    config:
      bucket: cms
  - type: directory
    config:
      ou: cms
  - type: smtp
    config:
      domain: cms
  - type: auth
    config:
      clientId: cms
```

## 2. 인증 (제1원칙)

### 현재: oauth2-proxy 기반 SSO

- Caddy + oauth2-proxy로 `/admin` 앞에 Keycloak OIDC 게이트
- Strapi 자체 인증은 우회

### 목표: Strapi 내부 OIDC

- Strapi SSO 플러그인 (Enterprise) 또는 커스텀 OIDC 구현
- AD/LDAP는 Keycloak Federation으로만

## 3. 스토리지 (제7원칙)

- Strapi upload provider: `@strapi/provider-upload-aws-s3` → RustFS
- 모든 미디어/에셋 S3 저장

## 4. 환경변수

| 변수 | PRC Claim | 용도 |
|------|-----------|------|
| `DATABASE_*` | database | PostgreSQL |
| `AWS_S3_ENDPOINT/BUCKET/KEY` | objectStorage | RustFS |
| `LDAP_*` | directory | AD |
| `SMTP_*` | smtp | 메일 |
| `OIDC_*` | auth | Keycloak |
| `APP_KEYS`, `JWT_SECRET` 등 | (자동생성) | Strapi 시크릿 |

## 5. 빌드/배포

```bash
cd PolyON-CMS
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-cms:v{VERSION} --push .

kubectl set image deploy/polyon-cms cms=jupitertriangles/polyon-cms:v{VERSION} -n polyon
```
