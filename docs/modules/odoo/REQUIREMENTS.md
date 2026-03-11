# PP ERP (Odoo) — 요구사항

> PP 원칙 기반 구체 요구사항. Cursor 개발 시 이 문서를 따른다.

---

## 1. PRC Claims

```yaml
claims:
  - type: database
    config:
      name: odoo
  - type: objectStorage
    config:
      bucket: odoo
  - type: smtp
    config:
      domain: odoo
  - type: auth
    config:
      clientId: odoo
      accessType: confidential
      redirectUris:
        - "https://odoo.{{ baseDomain }}/polyon/oidc/callback"
```

- ~~`requires`~~ 제거 필수
- module ID: `odoo` (AppEngine 아님)
- auth `accessType`: `confidential` (서버 간 token exchange)

## 2. 인증 (제1원칙)

### SSO 강제 — Keycloak OIDC가 유일한 로그인 경로

| 항목 | 값 |
|------|-----|
| 방식 | Keycloak OIDC Authorization Code Flow |
| client_id | `odoo` |
| realm | `polyon` |
| callback | `/polyon/oidc/callback` |
| 로그인 페이지 | `/web/login` → Keycloak 리다이렉트 |

### 사용자 생성 규칙

- AD → Core OdooSync → Odoo JSON-RPC → `res.users` 자동 생성
- `oauth_provider_id` + `oauth_uid` (= `sAMAccountName`) 설정
- SSO 첫 로그인 시 JIT 생성 (polyon_oidc addon)
- 사용자 직접 생성 차단 (`polyon_sync` context 필수)
- admin (uid ≤ 2) 비밀번호 로그인 예외 (Core JSON-RPC 인증용)

### LDAP

- **직접 로그인 차단** (SSO 강제)
- `polyon_ldap` addon은 AD 연결 설정만 자동화 (로그인용 아님)

## 3. 스토리지 (제7원칙)

- `polyon_s3_attachment` addon: `ir.attachment`을 RustFS(S3)에 저장
- S3 버킷: `odoo`
- 로컬 filestore 금지

## 4. UI 통합 (제4원칙)

### Console (관리자)

- iframe으로 Odoo 관리자 화면 (`/web`) 표시
- `skipHandshake: true` (Odoo는 PostMessage 미지원)
- `polyon_iframe` addon이 X-Frame-Options 제거

### Portal (사원)

- iframe으로 Odoo 사원 화면 (`/web`) 표시
- Keycloak polyon realm SSO로 자동 로그인

### 제약사항

- Odoo 자체 UI 사용 (Carbon Design 미적용 — 엔진 모듈 예외)
- Console(admin realm)과 Odoo(polyon realm)는 다른 Keycloak realm
  → iframe 내부에서 별도 SSO 로그인 필요

## 5. Addon 구현 규칙

### 공통

- Odoo core 수정 금지 — addon만
- Odoo 19 호환 (필드명: `group_ids`, `company_ids` 등)
- `polyon_sync` context 체크로 사용자 생성/수정 차단

### polyon_oidc

- `/polyon/oidc/login` — SSO 시작 (state 생성 + Keycloak 리다이렉트)
- `/polyon/oidc/callback` — code → token exchange → JWT 검증 → 세션 생성
- JWT 검증: JWKS 캐싱 (`OIDC_JWKS_URI`)
- 사용자 매칭: `oauth_uid` = `preferred_username`
- 미존재 시 자동 생성 (JIT)

### polyon_iframe

- `http.Response` 패치: `X-Frame-Options`, `Content-Security-Policy` 제거
- 모든 HTTP 응답에 적용

### polyon_s3_attachment

- `ir.attachment._file_read` / `_file_write` override
- S3 endpoint/bucket/key는 환경변수에서 읽음
- `ir.config_parameter`에 S3 설정 자동 등록

### polyon_ldap

- `post_install_hook`에서 `res.company.ldap` 레코드 자동 생성
- LDAP 연결 정보는 환경변수에서 읽음

## 6. 환경변수

| 변수 | PRC Claim | 용도 |
|------|-----------|------|
| `DATABASE_URL` | database.url | PostgreSQL |
| `DB_HOST/PORT/NAME/USER/PASSWORD` | database.* | 개별 연결 정보 |
| `S3_ENDPOINT` | objectStorage.endpoint | RustFS |
| `S3_BUCKET` | objectStorage.bucket | 버킷 |
| `S3_ACCESS_KEY` | objectStorage.accessKey | 인증 |
| `S3_SECRET_KEY` | objectStorage.secretKey | 인증 |
| `OIDC_ISSUER` | auth.issuer | OIDC 발급자 |
| `OIDC_CLIENT_ID` | auth.clientId | OIDC 클라이언트 |
| `OIDC_CLIENT_SECRET` | auth.clientSecret | OIDC 시크릿 |
| `OIDC_AUTH_ENDPOINT` | auth.authEndpoint | 인가 엔드포인트 (외부) |
| `OIDC_TOKEN_ENDPOINT` | auth.tokenEndpoint | 토큰 엔드포인트 (내부) |
| `OIDC_JWKS_URI` | auth.jwksUri | JWKS 엔드포인트 (내부) |
| `SMTP_HOST/PORT/USER/PASSWORD` | smtp.* | 메일 |
| `LDAP_HOST/PORT/BIND_DN/BIND_PASSWORD/BASE_DN` | directory.* | AD |

## 7. 빌드/배포

```bash
# 빌드 (Mac Mini에서만)
cd PolyON-Odoo  # 또는 PolyON-AppEngine
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jupitertriangles/polyon-odoo:v{VERSION} --push .

# 배포
kubectl set image deploy/polyon-odoo odoo=jupitertriangles/polyon-odoo:v{VERSION} -n polyon
```

## 8. Core 연동

### OdooSync API

```
POST /api/v1/modules/odoo/sync
```

- AD 사용자 목록 → Odoo `res.users` 동기화
- 생성/업데이트/비활성화
- `polyon_sync: true` context 포함

### OAuth Provider 자동 등록

- PRC auth claim 프로비저닝 시 Core가 Odoo JSON-RPC로 `auth.oauth.provider` 레코드 자동 등록
- provider name: `Keycloak (PolyON)`
- client_id: PRC에서 발급한 값
