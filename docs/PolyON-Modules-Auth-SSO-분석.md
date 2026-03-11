# PolyON 모듈 Auth / SSO (LDAP + Keycloak OIDC) 비교 분석 리포트

대상 모듈:

- PolyON-AppEngine (Odoo 19)
- PolyON-Auto (n8n)
- PolyON-Canvas (AFFiNE)
- PolyON-Channel (Mattermost)
- PolyON-BPMN (Operaton)
- PolyON-CMS (Strapi)
- PolyON-Drive (Rust Drive)

분석 축:

- AD/LDAP를 어디서 어떻게 사용하는지
- Keycloak(OIDC)를 어떤 방식으로 강제·연동하는지
- PolyON 제1원칙(SSO/Keycloak 우선, LDAP 직접 인증 지양)을 얼마나 잘 지키고 있는지

### 모듈별 요약 표

| 모듈 | LDAP 직접 로그인 사용 | LDAP 사용 위치/역할 | Keycloak(OIDC) 연동 방식 | SSO 강제 여부 | 비고 |
|------|----------------------|----------------------|--------------------------|---------------|------|
| **AppEngine** (Odoo) | **예 (auth_ldap, 동일 AD 계정)** | PRC `directory` → `LDAP_*` env → `res.company.ldap` 레코드 자동 생성(`polyon_ldap`) | env(`OIDC_*`) → Odoo 설정, env 없으면 entrypoint에서 기동 차단 | **강제(OIDC 기본, LDAP는 백업 경로)** | Keycloak OIDC SSO + Odoo LDAP 직접 로그인 **듀얼 패턴**, 두 경로 모두 AD/LDAP 동일 계정 사용 |
| **Auto** (n8n) | **예 — LDAP 로그인 토글 활성화(`N8N_SSO_LDAP_LOGIN_ENABLED=true`)** | n8n 엔터프라이즈 LDAP 로그인 기능 사용, LDAP 서버/바인드 정보는 n8n Admin UI에서 AD로 수동 설정 | `N8N_OIDC_*` + `N8N_SSO_*`, init 스크립트가 Settings DB에 OIDC 설정 UPSERT | 강제(실질적, OIDC 기본 + LDAP 병행 가능) | Keycloak OIDC SSO + n8n LDAP 로그인 듀얼 패턴(동일 AD 계정 전제, LDAP 설정은 UI에서 AD/Keycloak과 동일 소스로 맞춤) |
| **Canvas** (AFFiNE) | 아니오 — AFFiNE 자체에 LDAP 직접 로그인 기능이 없으며, AD/LDAP는 Keycloak 유저 페더레이션으로만 사용 | AD/LDAP는 Keycloak 뒤의 **계정 원장** 역할 (AFFiNE는 직접 LDAP bind를 하지 않음) | `claims.auth`(clientId: canvas) → Admin UI의 OIDC 설정에 수동 반영(또는 향후 초기 주입 스크립트로 자동화 예정) | 미강제(향후 OIDC SSO 강제화 계획) | 전략 1: **AD/LDAP ↔ Keycloak 유저 페더레이션 + AFFiNE ↔ Keycloak OIDC SSO** 구조로, 같은 AD 계정에 대해 SSO만 사용 |
| **Channel** (Mattermost) | **예 — LDAP(AD) + OIDC 듀얼 패턴 활성화** | PRC `directory` → `MM_LDAPSETTINGS_*` env로 AD/LDAP 서버·바인드·기본 DN 설정, 동일 AD 계정으로 LDAP 로그인 허용 | PRC `auth` → `MM_OAUTHSETTINGS_*` env로 Keycloak OIDC 설정(Discovery, clientId/secret) | 강제(SSO + LDAP 병행 가능, 동일 AD 계정 전제) | Mattermost가 원래 제공하는 AD/LDAP + OAuth/OIDC 기능을 PRC env로 활성화, 같은 AD 계정으로 SSO와 LDAP 직접 로그인을 모두 허용 |
| **BPMN** (Operaton) | **예 — LDAP Identity Provider + OIDC 듀얼 패턴 구성 가능(설계 단계)** | PRC `directory` → `LDAP_*` env로 Operaton LDAP Identity Provider에 AD/LDAP 계정/그룹 정보를 공급 | Spring Security OAuth2: `spring.security.oauth2.client.*` + `operaton.bpm.oauth2.*` 로 Keycloak OIDC 연동 | 강제(구성되면, LDAP/SSO 병행 가능) | Operaton이 제공하는 LDAP Identity Service + Spring Security OIDC를 함께 사용하면, 같은 AD 계정으로 LDAP 기반 계정 원장 + Keycloak SSO 듀얼 패턴 구현 가능(PolyON-BPMN 래퍼는 현재 LDAP 부분 설계/구현 단계) |
| **CMS** (Strapi) | **예 — LDAP(AD) + OIDC 듀얼 패턴 기술적으로 가능(플러그인/엔터프라이즈 조합)** | AD/LDAP는 계정 원천, Strapi LDAP/SSO 플러그인으로 직접 연동 가능 (PRC `directory` 로 AD 정보 공급 가능) | 현재 PolyON-CMS는 `oauth2-proxy + Caddy` 로 `/admin` 에 Keycloak OIDC 1경로만 강제, 향후 Strapi 내부 OIDC/LDAP 플러그인으로 듀얼 패턴 전환 계획 | 강제(OIDC 한 경로, 현 구현 기준) | Strapi는 제품 능력상 LDAP+OIDC 듀얼 패턴을 수용할 수 있고, PolyON에서는 PRC(auth/directory)로 이를 지원하면서, 1단계로 프록시 기반 OIDC를 먼저 구현한 상태 |
| **Drive** (Rust Drive) | 예 (Basic Auth + LDAP bind) | 인프라 스토리지 계층에서 AD/LDAP를 1차 인증 소스로 사용 | PRC `auth` → `OIDC_ISSUER`/`OIDC_CLIENT_ID`/`OIDC_JWKS_URI`, WebDAV에서 `Authorization: Bearer` 토큰 수용 | 비적용(의도적 예외) | 인프라 스토리지로서 **LDAP 직접 인증 + Keycloak OIDC 토큰** 모두 수용하는 듀얼 패턴, 예외 규칙 문서화 필요 |

---

## 1. PolyON-AppEngine (Odoo 19)

### LDAP

- AppEngine 이전(Odoo 시절) 설계에서는 LDAP 모듈을 통한 **직접 LDAP 로그인**이 있었다.
- v3 설계(REQUIREMENTS v3 기준)에서는:
  - PRC `directory` Claim을 **복원**하여 AD/LDAP를 공식 계정 원장으로 사용.
  - `claims.directory.*` → `LDAP_*` env → `polyon_ldap` 애드온이 `res.company.ldap` 레코드를 자동 생성.
  - 결과적으로 **Odoo 내부에서 auth_ldap 기반의 직접 LDAP 로그인**을 다시 허용한다.

### Keycloak OIDC

- `polyon-module/module.yaml`:
  - `claims.auth` 에서 `issuer`, `authEndpoint`, `tokenEndpoint`, `jwksUri` 등 Keycloak 정보를 받음.
  - env: `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_AUTH_ENDPOINT`, `OIDC_TOKEN_ENDPOINT`, `OIDC_JWKS_URI` 로 주입.
- `entrypoint.sh`:
  - OIDC 관련 env가 없으면 **Odoo 기동을 차단**하도록 설계되어 있음.

### 평가

- Keycloak OIDC env가 없으면 **Odoo 기동을 차단**해 SSO를 기본 경로로 강제한다.
- 동시에, 같은 AD 계정에 대해 **auth_ldap 기반 직접 LDAP 로그인도 허용**하는 **듀얼 패턴** 모듈로 진화했다.

---

## 2. PolyON-Auto (n8n)

### LDAP

- n8n 자체는 LDAP 기반 로그인 옵션이 있으나, PolyON-Auto 설계에서는:
  - `N8N_SSO_LDAP_LOGIN_ENABLED: "false"` 로 비활성화.
  - LDAP 직접 인증 대신 **SSO 기반(OIDC)** 만 허용하는 방향.

### Keycloak OIDC

- `polyon-module/module.yaml`:
  - `claims.auth` → `N8N_OIDC_DISCOVERY_URL`, `N8N_OIDC_CLIENT_ID`, `N8N_OIDC_CLIENT_SECRET`.
  - SSO 플래그: `N8N_SSO_OIDC_LOGIN_ENABLED: "true"`, `N8N_SSO_REDIRECT_LOGIN_TO_SSO: "true"`.
- `scripts/polyon-oidc-init.js`:
  - `N8N_OIDC_*` + `N8N_ENCRYPTION_KEY` env를 읽어,
  - n8n Settings DB의 `features.oidc` 항목을 암호화하여 UPSERT.
  - 결과적으로 n8n은 **SSO 메뉴 없이 바로 OIDC 로그인**을 사용하도록 설정.

### 평가

- LDAP 기반 로그인은 꺼 두고, **Keycloak OIDC만을 활성화**하는 구조.
- SSO 설정을 DB에 선주입하여, 운영자가 수동 설정하지 않아도 되도록 자동화한 점이 뛰어남.

---

## 3. PolyON-Canvas (AFFiNE)

### LDAP

- AFFiNE는 **LDAP 직접 로그인(AD bind)** 기능을 제공하지 않는다.
- PolyON-Canvas에서는 AD/LDAP를 **Keycloak 유저 페더레이션을 통한 계정 원장**으로만 사용한다.
- 즉, 사용자는 LDAP 계정으로 직접 AFFiNE에 로그인하지 않고, **항상 Keycloak을 통해서만** 진입한다.

### Keycloak OIDC

- `polyon-module/module.yaml`:
  - `claims.auth` 선언 (`clientId: canvas`) 으로 Keycloak OIDC 클라이언트가 할당된다.
  - OIDC 관련 env 매핑은 향후 Admin UI 설정 자동화를 위해 도입 예정(TODO).
- `REPORT-AFFiNE-검토.md` / `DEV-CHECKLIST`:
  - OIDC 설정이 **Admin Panel 기반**이며, 현재는 관리자가 Keycloak 정보를 수동 입력한다.
  - “n8n과 유사한 OIDC 초기 주입 스크립트”를 도입해 Keycloak 설정을 자동 주입하는 것을 목표로 한다.

### 평가

- 전략 1 채택:
  - **AD/LDAP ↔ Keycloak 유저 페더레이션**
  - **AFFiNE ↔ Keycloak OIDC SSO**
- 따라서, 같은 AD 계정에 대해 **SSO(OIDC) 한 경로만 사용**하지만,
  - 계정 원장은 AD/LDAP 하나로 통일되어 있고,
  - PolyON 규칙(SSO 우선, LDAP 직접 로그인 지양)과도 일관된다.

---

## 4. PolyON-Channel (Mattermost)

### LDAP

- Mattermost는 LDAP/AD를 직접 연결하는 기능이 있지만,
  - 현재 PolyON-Channel 설계에서는 LDAP env 매핑을 넣지 않고 있음.
  - 대신, PolyON 차원에서 AD 동기화/비활성화는 **향후 설계 과제**로 남겨둔 상태.

### Keycloak OIDC

- Mattermost는 SAML/OIDC 등 다양한 SSO 옵션을 제공.
- `polyon-module/module.yaml`:
  - `claims.auth` 선언은 있으나,
  - `MM_OAUTHSETTINGS_*` env는 **주석 처리**되어 있어, OIDC 자동 설정은 아직 미도입.
- `REPORT-Mattermost-검토.md`:
  - OIDC 지원과 env 규칙(`MM_` 접두사, `FileSettings`/`SqlSettings` 등)을 정리해 두었지만,
  - 실제 PolyON-Channel은 아직 “SSO를 강제하는 단계”까지 가지 않은 것으로 기록.

### 평가

- 현재 상태:
  - LDAP 직접 인증도, OIDC 강제도 없는 **기본 상태**에 가깝다.
  - PolyON 규칙 관점에서는 “SSO 자동 주입이 미완성인 모듈”로 분류.
- 개선 포인트:
  - Operaton/n8n/Strapi처럼 Keycloak OIDC를 env만으로 구성할 수 있는지,  
    Mattermost 문서/소스를 더 깊이 확인한 뒤,
    - `MM_OAUTHSETTINGS_ENABLED`  
    - `MM_OAUTHSETTINGS_DISCOVERYENDPOINT`  
    - `MM_OAUTHSETTINGS_CLIENTID/CLIENTSECRET`  
    등을 PRC `auth` 에서 자동 주입하는 구조로 올리는 것이 이상적이다.

---

## 5. PolyON-BPMN (Operaton)

### LDAP

- Operaton(=Camunda 7 후속)은 LDAP Identity Provider를 지원하지만,
  - Spring Boot Starter 기반 구성에서는 **주로 DB Identity + OIDC** 조합을 사용.
  - PolyON-BPMN 설계에서도 LDAP 직접 인증은 사용하지 않는다.

### Keycloak OIDC

- `pom.xml`:
  - `operaton-bpm-spring-boot-starter-webapp` / `-rest` / `-security` 의존성 사용.
- `application.yaml`:
  - `spring.security.oauth2.client.registration.keycloak.*`
  - `spring.security.oauth2.client.provider.keycloak.*`  
  - `operaton.bpm.oauth2.identity-provider.*` / `operaton.bpm.oauth2.sso-logout.*`
  등을 설정 가능.
- `polyon-module/module.yaml`:
  - `claims.auth` → `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_ISSUER_URI` 로 주입.
  - Spring Security + Operaton의 OAuth2/OIDC 통합에 맞춰 Webapps를 보호.

### 평가

- LDAP 직접 인증은 사용하지 않고,  
  **Spring Security OIDC를 표준 패턴대로 사용하는 모듈**로, AppEngine/Auto와 함께 SSO 모범 사례에 속한다.

---

## 6. PolyON-CMS (Strapi)

### LDAP

- Strapi는 LDAP/AD 연동을 위한 플러그인·엔터프라이즈 옵션을 제공하며,  
  **같은 AD 계정으로 LDAP bind + SSO(OIDC)를 동시에 허용하는 듀얼 패턴을 기술적으로 수용할 수 있는 모듈**이다.
- PolyON 관점의 이상적인 그림:
  - AD/LDAP는 Keycloak 유저 페데레이션으로 계정을 공급하고,
  - Strapi는 필요 시 LDAP 플러그인으로 직접 bind 로그인도 허용,
  - 동시에 Admin/사용자 SSO는 Keycloak OIDC로 처리하는 구조.

### Keycloak OIDC

- Strapi Admin SSO(OIDC/SAML)는 Enterprise/SSO 애드온으로 공식 제공된다.
- **이상적인 듀얼 패턴(목표)**:
  - Admin/사용자 SSO: Strapi 내부 OIDC 플러그인(또는 공식 SSO 기능)을 사용해 Keycloak과 직접 연동.
  - LDAP 로그인: AD/LDAP 플러그인으로 같은 계정으로 직접 로그인 허용.
- **현 PolyON-CMS 구현(임시 상태)**:
  - Strapi 내부 SSO는 아직 사용하지 않고,
  - 컨테이너 앞단의 `oauth2-proxy + Caddy` 로 `/admin` 경로에 대해 OIDC를 강제하는 “프록시 게이트” 방식으로 1차 보호를 하고 있다.

### 평가

- Strapi 자체 능력으로는 **AD/LDAP + Keycloak OIDC 듀얼 패턴을 수용할 수 있는 모듈**이며,  
  PolyON이 지향하는 “같은 AD 계정에 대해 SSO와 LDAP 로그인 둘 다 허용” 모델에 잘 맞는다.
- 다만, 현재 PolyON-CMS 코드는 **프록시 기반 OIDC 단일 경로**를 먼저 구현해 둔 상태이며,  
  향후에는 Strapi 내부 OIDC/LDAP 플러그인을 활용해 **네이티브 듀얼 패턴 구성으로 전환하는 것**이 목표이다.

---

## 7. PolyON-Drive (Rust Drive)

### LDAP

- Drive는 인프라 스토리지 계층으로, AD/LDAP을 **직접 인증 소스로 사용**하는 모듈.
  - WebDAV 및 User API는 `Authorization: Basic` (LDAP) 을 사용.
  - `POLYON_DC_*` / `DRIVE_LDAP_*` env로 AD DC 정보를 주입.

### Keycloak OIDC

- 현재 설계에서는 Drive에 Keycloak OIDC를 붙이지 않는다.
  - Portal/Console UI에서 Drive를 사용할 때는,  
    PolyON Portal이 자체적으로 Keycloak 토큰 → LDAP 사용자 매핑을 처리하고,  
    Drive에는 여전히 LDAP 자격증명을 사용해 접속하거나, Admin API에만 토큰 기반 보호를 검토할 수 있음.

### 평가

- PolyON 전체 중 **유일하게 “LDAP 직접 인증”을 공식적으로 허용하는 모듈**.  
  - 인프라 계층(스토리지) 특성상 예외로 인정할 수 있으나,  
  - 해당 예외 규칙을 PolyON 전역 규칙 문서에 명확히 적어두는 것이 바람직하다.

---

## 8. 종합 평가 및 권장 사항

1. **SSO 모범 사례 (강하게 준수)**
   - **AppEngine**: OIDC env 없으면 기동 차단, 동일 AD 계정 기준으로 OIDC + LDAP 듀얼 패턴 지원.  
   - **Auto**: LDAP 로그인 비활성화, OIDC 설정 DB 선주입.  
   - **BPMN**: Spring Security OIDC 표준 패턴 사용.  
   - **CMS**: oauth2-proxy + Caddy로 `/admin` 전면 OIDC 게이트화.

2. **SSO 구현 미완/TODO**
   - **Canvas**: Auth claim은 있으나, AFFiNE 내부에 OIDC를 자동 주입하는 스크립트는 아직 없음.  
   - **Channel**: Mattermost OIDC env가 주석 상태라, 실질적으로 SSO 미도입 상태.

3. **LDAP 직접 인증 예외/인프라 계층**
   - **Drive**: 인프라 스토리지 계층으로, AD LDAP 직접 인증 + Keycloak OIDC 토큰을 함께 수용하는 듀얼 패턴.  
     - “업무/협업 앱 계층은 OIDC SSO 기본, 인프라 계층(Drive)은 LDAP 직접 인증 + 토큰 듀얼 허용”이라는 예외 규칙을 전역 문서에 명시하는 것이 좋다.

4. **권장 후속 작업**
   - Canvas:  
     - AFFiNE의 인증 설정 방식을 더 조사해, n8n/Operaton/CMS처럼 **Keycloak OIDC를 초기 주입하는 스크립트**를 설계.  
   - Channel:  
     - Mattermost OIDC/SAML 설정을 env로 전부 제어할 수 있는지 확인 후,  
       `MM_OAUTHSETTINGS_*` env를 PRC `auth`와 연결해 **SSO 강제 모드**로 끌어올리기.  
   - Drive:  
     - README/규칙 문서에 “Drive는 인프라 계층으로 LDAP 직접 인증 예외 허용”을 명문화.

종합적으로, PolyON 모듈들은 **“업무/엔진 계층은 Keycloak OIDC, 인프라 계층(Drive)은 LDAP 직접 인증 예외”** 라는 방향성을 이미 갖추고 있으며,  
Canvas/Channel에서 SSO 자동화만 보강하면 전체 스토리가 더욱 깔끔하게 정리될 수 있다.

