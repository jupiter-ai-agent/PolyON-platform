# PolyON Module Catalog — 모듈 현황

> 최종 업데이트: 2026-03-12

---

## Foundation (삭제 불가)

Foundation은 모든 모듈이 의존하는 공유 인프라. PRC Provider로 자원을 제공한다.

| 서비스 | 이미지 | PRC Provider | 상태 |
|--------|--------|-------------|------|
| PostgreSQL 18 | pgvector/pgvector:pg18 | `database` | ✅ 운영 중 |
| Redis 8 | redis:8-alpine | `cache` | ✅ 운영 중 |
| OpenSearch 3.5 | opensearchproject/opensearch:3.5.0 | `search` | ✅ 운영 중 |
| RustFS | rustfs/rustfs:1.0.0-alpha.85 | `objectStorage` | ✅ 운영 중 |
| Samba AD DC | jupitertriangles/polyon-dc | `directory` | ✅ 운영 중 |
| Keycloak | quay.io/keycloak/keycloak:latest | `auth` | ✅ 운영 중 |
| Stalwart Mail | jupitertriangles/polyon-mail | `smtp` | ✅ 운영 중 |
| Gitea | gitea/gitea:latest | — | ✅ 운영 중 |
| Traefik v3 | traefik:v3.4 | — | ✅ 운영 중 |
| Core | jupitertriangles/polyon-core | — | ✅ v1.14.0 |
| Console | jupitertriangles/polyon-console | — | ✅ v0.10.25 |

---

## 엔진 모듈

### 배포됨 (Deployed)

| 모듈 | ID | 리포 | 이미지 | 엔진 | 버전 | 상태 |
|------|-----|------|--------|------|------|------|
| PP Drive | `drive` | [PolyON-Drive](https://github.com/jupiter-ai-agent/PolyON-Drive) | polyon-drive:v0.3.0 | Rust (자체 개발) | v0.3.0 | ✅ 운영 중 |
| PP Channel | `mattermost` | [PolyON-Channel](https://github.com/jupiter-ai-agent/PolyON-Channel) | polyon-chat:v0.2.0 | Mattermost | v0.2.0 | ✅ 운영 중 |
| PP ERP | `odoo` | [PolyON-AppEngine](https://github.com/jupiter-ai-agent/PolyON-AppEngine) | polyon-odoo:v0.4.0 | Odoo 19 | v0.4.0 | ⚠️ SSO 검증 중 |

### 미배포 (Not Deployed)

| 모듈 | ID | 리포 | 엔진 | 상태 |
|------|-----|------|------|------|
| PP Canvas | `canvas` | [PolyON-Canvas](https://github.com/jupiter-ai-agent/PolyON-Canvas) | AFFiNE | ⛔ stub (module.yaml만) |
| PP BPMN | `bpmn` | [PolyON-BPMN](https://github.com/jupiter-ai-agent/PolyON-BPMN) | Operaton (Spring Boot) | ⛔ stub (module.yaml + 빈 앱) |
| PP CMS | `cms` | [PolyON-CMS](https://github.com/jupiter-ai-agent/PolyON-CMS) | Strapi | ⛔ 기본 구현 |
| PP Auto | `auto` | [PolyON-Auto](https://github.com/jupiter-ai-agent/PolyON-Auto) | n8n | ⛔ 빈 리포 |

---

## PP 규격 준수 현황

### PRC Claims 사용

| 모듈 | database | objectStorage | smtp | auth | directory | cache |
|------|:--------:|:------------:|:----:|:----:|:---------:|:-----:|
| Drive | ✅ | ✅ | — | ✅ | — | — |
| Channel | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| ERP (Odoo) | ✅ | ✅ | ✅ | ✅ | — | — |
| Canvas | ✅ | ✅ | ✅ | ✅ | — | — |
| BPMN | ✅ | — | ✅ | ✅ | ✅ | — |
| CMS | ✅ | ✅ | ✅ | ✅ | ✅ | — |

### 인증 방식

| 모듈 | SSO (OIDC) | LDAP 직접 | AD 동기화 | SSO 강제 |
|------|:----------:|:---------:|:---------:|:--------:|
| Drive | ✅ Bearer | ✅ WebDAV bind | — | 병행 |
| Channel | ✅ | ✅ 듀얼 | ✅ Core sync | 병행 |
| ERP (Odoo) | ✅ | ❌ 차단 | ✅ Core OdooSync | SSO 강제 |
| Canvas | 계획 | ❌ | — | 미적용 |
| BPMN | 계획 | 계획 | — | 미적용 |
| CMS | ✅ oauth2-proxy | — | — | SSO 강제 |

### SDK 사용

| 모듈 | 언어 | SDK 사용 | 비고 |
|------|------|:--------:|------|
| Drive | Rust | ❌ | SDK 미제공 (Rust). 자체 구현으로 PRC 환경변수 파싱 |
| Channel | Go | ❌ | Mattermost 내장 설정. 환경변수로 주입 |
| ERP (Odoo) | Python | ❌ | Odoo addon으로 구현. SDK 대신 환경변수 직접 파싱 |
| Canvas | TypeScript | ❌ | 미구현 |
| BPMN | Java | ❌ | 미구현. Spring Boot 설정으로 주입 예정 |
| CMS | JavaScript | ❌ | 미구현 |

> **현실:** SDK를 직접 사용하는 모듈이 0개. 각 엔진의 네이티브 설정 방식(환경변수)으로 PRC 자원을 주입하는 것이 실제 패턴.  
> SDK는 **자체 개발 모듈** (Drive 같은)에서 사용하는 것이 적합.

---

## 알려진 이슈

| 모듈 | 이슈 | 우선순위 |
|------|------|---------|
| ERP (Odoo) | SSO callback 500 에러 (group_ids 수정 후 미검증) | 🔴 |
| ERP (Odoo) | Console iframe SSO 리다이렉트 cross-origin 문제 | 🔴 |
| ERP (Odoo) | PG DB 생성 시 GRANT 자동화 없음 | 🟡 |
| ERP (Odoo) | GitHub 리포(AppEngine)와 실제 코드(PolyON-Odoo) 불일치 | 🟡 |
| Drive | module.yaml이 구형 `requires` 사용 (PRC `claims` 미전환) | 🟡 |
| Channel | module.yaml 이미지 태그 `latest` (고정 버전 아님) | 🟡 |
| Auto | 빈 리포 — 목적/계획 미정의 | ⚪ |
| 전체 | SDK 실사용 모듈 0개 | ⚪ |

---

## 리포 정리 TODO

- [ ] `PolyON-AppEngine` → `PolyON-Odoo`로 리네임 (또는 코드 동기화)
- [ ] `PolyON-Auto` — 목적 정의 또는 삭제
- [ ] `helios` — Archive 처리
- [ ] Drive `module.yaml` — `requires` → `claims` 전환
- [ ] 각 모듈 이미지 태그 고정 (latest 금지)
