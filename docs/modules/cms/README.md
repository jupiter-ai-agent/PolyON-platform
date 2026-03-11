# PP CMS

| 항목 | 값 |
|------|-----|
| **모듈 ID** | `cms` |
| **엔진** | Strapi |
| **리포** | [PolyON-CMS](https://github.com/jupiter-ai-agent/PolyON-CMS) |
| **이미지** | 미정 |
| **상태** | ⛔ 미배포 (기본 구현) |
| **라이선스** | MIT |

## 개요

Headless CMS. Strapi를 npm 패키지로 사용, oauth2-proxy + Caddy로 OIDC 보호.

## 현재 상태

- 기본 Strapi 앱 + Caddy + supervisord 구성
- oauth2-proxy로 `/admin` Keycloak 보호
- 실제 배포/테스트 안 됨

## TODO

- [ ] 실제 빌드 및 배포 테스트
- [ ] Strapi 내부 OIDC/LDAP 플러그인으로 전환 (oauth2-proxy 제거)
- [ ] S3 upload provider 설정
- [ ] Console/Portal iframe 통합
