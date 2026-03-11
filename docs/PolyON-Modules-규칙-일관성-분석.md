# PolyON 모듈 규칙·일관성 분석 리포트

대상 모듈:

- PolyON-AppEngine (Odoo 19)
- PolyON-Auto (n8n)
- PolyON-Canvas (AFFiNE)
- PolyON-Channel (Mattermost)
- PolyON-BPMN (Operaton)
- PolyON-CMS (Strapi)
- PolyON-Drive (Rust Drive)

분석 축:

- 원본 소스 포함 여부 (래퍼-only vs 풀 커스텀)
- PRC 매핑 (database / objectStorage / smtp / auth / directory)
- Storage-first(only) — RustFS 사용 여부
- SSO/OIDC (Keycloak) 적용 정도
- Health/entrypoint 패턴

---

## 1. 원본 소스 포함 여부

- **래퍼-only (원본 미포함)**  
  - **PolyON-Auto**: 공식 n8n 이미지 래핑, 소스는 upstream에서 관리.  
  - **PolyON-Canvas**: `ghcr.io/toeverything/affine:stable` 베이스, 래퍼만 존재.  
  - **PolyON-Channel**: `mattermost/mattermost-team-edition` 베이스, 래퍼만 존재.  
  - **PolyON-BPMN**: Operaton은 Maven 의존성으로만 사용, 우리 쪽은 Spring Boot 앱 코드만 포함.  
  - **PolyON-CMS**: Strapi는 npm 패키지로만 사용, 별도 fork 없음.

- **원본 포함(필요 시 예외 허용)**  
  - **PolyON-AppEngine**: Odoo 19 소스 clone + 커스텀 addons + entrypoint.  
    - README/REQUIREMENTS에 “공식 이미지 금지, 소스 빌드 필수”가 명시되어 있음.  
  - **PolyON-Drive**: 자체 Rust 서비스(Nextcloud 대체), 소스 포함이 자연스럽고 목적에 부합.

➡️ **평가**:  
“가능한 경우에는 원본 미포함 래퍼-only, 소스 레벨 커스텀이 필수인 경우(AppEngine, Drive)는 예외”라는 정책이 잘 지켜지고 있으며, AppEngine/Drive는 문서로 예외성이 설명되어 있어 규칙 위반이라기보다 **정책화된 예외**에 가깝다.

---

## 2. PRC 매핑 (database / objectStorage / smtp / auth / directory)

- **AppEngine**
  - `claims`: `database`, `objectStorage`, `smtp`, `auth`
  - env:  
    - DB: `DB_*`  
    - S3: `AWS_*`  
    - SMTP: `SMTP_*`  
    - OIDC: `OIDC_*` (issuer, endpoints 등)

- **Auto (n8n)**
  - `claims`: `database`, `objectStorage`, `smtp`, `auth`
  - env:  
    - DB: `DB_POSTGRESDB_*`  
    - S3: `N8N_EXTERNAL_STORAGE_S3_*`  
    - SMTP: `N8N_SMTP_*`  
    - OIDC: `N8N_OIDC_*` + SSO 플래그 (`N8N_SSO_*`)

- **Canvas (AFFiNE)**
  - `claims`: `database`, `objectStorage`, `smtp`, `auth`
  - env:  
    - DB: `DATABASE_URL` / `DB_*`  
    - Redis: 고정 호스트(`REDIS_SERVER_HOST`)  
    - Storage: `AFFINE_STORAGE_*` (RustFS용)  
    - Auth: claim은 있지만 OIDC를 실제로 주입하는 흐름은 아직 TODO.

- **Channel (Mattermost)**
  - `claims`: `database`, `objectStorage`, `smtp`, `auth`
  - env:  
    - DB: `MM_SQLSETTINGS_*`  
    - Storage: `MM_FILESETTINGS_*` (S3/RustFS)  
    - SMTP: `MM_EMAILSETTINGS_*`  
    - OIDC: `MM_OAUTHSETTINGS_*` 키는 주석 상태(TODO).

- **BPMN (Operaton)**
  - `claims`: `database`, `auth`, `smtp`
  - env:  
    - DB: `SPRING_DATASOURCE_*`  
    - OIDC: `OIDC_*` → Spring Security + `operaton.bpm.oauth2.*` 설정에 사용  
    - smtp claim은 아직 env로 직접 매핑되지 않음(향후 확장 여지).

- **CMS (Strapi)**
  - `claims`: `database`, `objectStorage`, `smtp`, `auth`
  - env:  
    - DB: `DATABASE_*`  
    - Storage: `AWS_*` (RustFS/S3)  
    - SMTP: 아직 module.yaml에는 직접 매핑되지 않음(TODO).  
    - OIDC: `OIDC_*` + `OAUTH2_PROXY_COOKIE_SECRET` → oauth2-proxy 설정에 사용.

- **Drive**
  - `claims`: `database`, `objectStorage`, `directory`
  - env:  
    - DB: `DATABASE_URL`  
    - Storage: `S3_*` / `POLYON_RUSTFS_*`  
    - Directory(AD): `LDAP_*` / `POLYON_DC_*`

➡️ **평가**:  
- 비즈니스/엔진 계층(AppEngine/Auto/Canvas/Channel/BPMN/CMS)은 공통적으로 `database`/`objectStorage`/`smtp`/`auth` 조합을 사용하고, Drive는 인프라 특성상 `directory` claim으로 AD를 직접 사용.  
- 다만 Canvas/Channel/BPMN/CMS 일부에서 **smtp/auth가 “선언만 있고 구현은 TODO”** 인 부분이 있어, DEV-CHECKLIST에 명시하고 단계적으로 메우는 것이 좋다.

---

## 3. Storage-first(only) — RustFS 적용

- **강하게 Storage-first 적용된 모듈**
  - **Drive**: 설계 자체가 RustFS(S3) 기반 WebDAV 스토리지, 로컬 디스크는 메타 수준.  
  - **Canvas**: AFFiNE의 local-first 대신 RustFS-only를 목표로 하고, `AFFINE_STORAGE_*` env로 RustFS 정보를 제공.  
  - **Channel**: Mattermost FileSettings를 `amazons3` 로 고정해 첨부파일은 RustFS에만 저장.  
  - **CMS**: Strapi upload provider를 `aws-s3`로 강제, endpoint를 RustFS에 맞춤.

- **부분적 Storage-first**
  - **Auto**: n8n binary data를 S3 모드로, execution data는 DB에 저장 — 워크플로우 엔진 용도에 적합한 절충.  
  - **AppEngine**: Odoo 첨부(ir.attachment)를 S3(RustFS)로 보내는 addon 구조, 임시/로그 등 일부 local fs 사용은 허용.

- **Storage-first 요구가 약한 모듈**
  - **BPMN**: 주로 메타데이터/이력 위주라 objectStorage를 강제할 필요가 크지 않음.

➡️ **평가**:  
파일/콘텐츠가 중요한 모듈(Drive/Canvas/Channel/CMS/Auto/AppEngine)은 RustFS 사용이 설계에 녹아있고, PolyON의 “회사 데이터는 회사 스토리지에” 원칙을 잘 반영하고 있다.

---

## 4. SSO / 인증 (Keycloak OIDC vs LDAP)

- **SSO(Keycloak OIDC)를 강제하는 모듈**
  - **AppEngine**: `OIDC_*` env 없으면 기동을 막는 엔트리포인트, REQUIREMENTS에 “LDAP 직접 인증 금지” 명시.  
  - **Auto**: n8n SSO 플래그 + DB 초기화 스크립트로 OIDC 설정을 DB에 주입.  
  - **BPMN**: Spring Security OAuth2 + `operaton.bpm.oauth2.*`로 Operaton Webapps를 OIDC로 보호.  
  - **CMS**: Strapi Enterprise SSO 대신 **oauth2-proxy + Caddy** 로 `/admin` 전체를 Keycloak OIDC 게이트 뒤에 배치.

- **SSO 설계는 있으나 구현 미완**
  - **Canvas**: auth claim은 있으나 AFFiNE에 OIDC를 자동 설정하는 init 스크립트는 아직 TODO 상태.  
  - **Channel**: auth claim은 선언했지만 Mattermost OIDC env (`MM_OAUTHSETTINGS_*`) 는 주석 처리.

- **LDAP 직접 인증 사용하는 모듈(예외)**
  - **Drive**: WebDAV + Basic Auth + LDAP(AD). 인프라 스토리지 계층의 특성상 AD 직접 인증을 허용한 구조.

➡️ **평가**:  
- AppEngine/Auto/BPMN/CMS는 PolyON 제1원칙(SSO/Keycloak 우선)을 강하게 지키고 있음.  
- Canvas/Channel은 SSO 자동화가 **“설계 중/주석 상태”** 이므로,  
  - 문서에 “현재는 Admin 수동 설정 필요 / 향후 init 스크립트 도입 예정”을 더 명확히 적어두고,  
  - 점진적으로 Auto/BPMN 수준으로 끌어올리는 것이 좋다.  
- Drive는 “인프라 계층은 AD 직접 인증 허용”이라는 예외 규칙을 별도 문서에서 명문화할 필요가 있다.

---

## 5. Health / Entrypoint 패턴

- **Health 엔드포인트**
  - AppEngine: `/web/health`  
  - Auto: `/healthz/readiness`  
  - Canvas: `/`  
  - Channel: `/api/v4/system/ping`  
  - BPMN: `/actuator/health`  
  - CMS: `/admin` (Caddy + oauth2-proxy + Strapi 체인 전체 헬스 확인)  
  - Drive: `/health`

- **Entrypoint 패턴**
  - AppEngine: PRC env → `odoo.conf` 생성 + addon 설치 + DB 대기 후 기동.  
  - Auto/Canvas/Channel: DB/Redis 대기 후 `exec "$@"` 패턴.  
  - BPMN: Spring Boot 앱이므로 간단한 DB 대기만 필요.  
  - CMS: 필수 시크릿(APP_KEYS/JWT 등) 검증 후 `supervisord`로 Strapi + oauth2-proxy + Caddy 기동.  
  - Drive: Rust 바이너리 직접 기동.

➡️ **평가**:  
각 모듈의 기술 스택에 맞춰 구현은 다르지만,  
**“PRC 기반 env → readiness 대기 → 서비스 기동”** 이라는 공통 패턴은 잘 지켜지고 있다.

---

## 6. 종합 평가 및 개선 포인트

1. **원본 소스 포함 정책**  
   - AppEngine/Drive는 설계 상 커스텀 소스가 필수라 예외로 인정 가능.  
   - 나머지 모듈은 래퍼-only 구조를 지키고 있어 정책 일관성이 좋다.

2. **PRC 매핑**  
   - 대부분 모듈에서 DB/ObjectStorage/Auth/Smtp/directory가 PRC 스펙에 맞게 잘 매핑.  
   - Canvas/Channel/BPMN/CMS 의 smtp/auth 일부가 TODO 상태이므로, DEV-CHECKLIST 및 리포트에서 “미구현/향후 작업”으로 명시해 두는 것이 좋다.

3. **Storage-first(only)**  
   - Drive/Canvas/Channel/CMS/Auto/AppEngine이 RustFS를 중심으로 설계되어 있으며, PolyON 데이터 주권 원칙을 준수.

4. **SSO/OIDC**  
   - AppEngine/Auto/BPMN/CMS는 Keycloak OIDC를 사실상 필수로 강제.  
   - Canvas/Channel의 SSO 자동화는 향후 과제로 남아 있고,  
     - 특히 Channel은 OIDC env가 주석 상태이므로, Mattermost 문서 확인 후 실제 env 매핑을 도입하면 좋다.  
   - Drive의 LDAP 직접 인증은 인프라 예외 규칙으로 별도 문서화 필요.

전반적으로, **PolyON 코어 엔진/스토리지 모듈들은 규칙을 잘 따르고 있고**,  
AppEngine/Drive는 “예외지만 명시적으로 설계된 특수 모듈”, Canvas/Channel은 “SSO 자동화 측면에서 추후 보강 대상” 정도로 정리할 수 있다.

