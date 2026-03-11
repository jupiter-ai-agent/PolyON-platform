# PP ERP (Odoo)

| 항목 | 값 |
|------|-----|
| **모듈 ID** | `odoo` |
| **엔진** | Odoo 19 (Community) |
| **리포** | [PolyON-AppEngine](https://github.com/jupiter-ai-agent/PolyON-AppEngine) |
| **이미지** | `jupitertriangles/polyon-odoo:v0.4.0` |
| **상태** | ⚠️ SSO 검증 중 |
| **라이선스** | LGPL-3.0 |

## 개요

ERP/HR/회계/비즈니스 관리 플랫폼. Odoo 19 Community Edition을 커스텀 addon으로 PP 통합.

## 커스텀 Addon 4개

| Addon | 역할 |
|-------|------|
| `polyon_oidc` | Keycloak SSO 로그인 (OIDC callback, JWT 검증, 자동 사용자 생성) |
| `polyon_iframe` | X-Frame-Options 제거 (Console iframe 호환) |
| `polyon_ldap` | AD LDAP 연결 자동 설정 |
| `polyon_s3_attachment` | 첨부파일을 RustFS(S3)에 저장 |

## 현재 상태

- K8s 배포 완료 (`odoo.cmars.com`)
- PRC 4개 claim 프로비저닝 완료 (database, objectStorage, smtp, auth)
- SSO callback 500 에러 수정 (v0.4.0) — E2E 미검증
- Console iframe 로드 가능 (skipHandshake)
- Core OdooSync 구현 완료 (AD→res.users) — 미실행

## 알려진 이슈

| 이슈 | 우선순위 |
|------|---------|
| SSO E2E 미검증 | 🔴 |
| Console iframe에서 cross-origin SSO 리다이렉트 | 🔴 |
| GitHub 리포명(AppEngine) ≠ 실제 코드(Odoo) | 🟡 |
| module.yaml에 `requires`+`claims` 혼재 | 🟡 |
| module.yaml ID `appengine` ≠ 실제 배포 ID `odoo` | 🟡 |
| PG DB OWNER 자동화 없음 | 🟡 |
