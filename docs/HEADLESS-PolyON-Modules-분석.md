# PolyON 모듈 Headless 구성 가능성 분석

대상 모듈:

- PolyON-AppEngine (Odoo 19)
- PolyON-Auto (n8n)
- PolyON-Canvas (AFFiNE)
- PolyON-Channel (Mattermost)
- PolyON-BPMN (Operaton)
- PolyON-CMS (Strapi)
- PolyON-Drive (Rust Drive)

---

## 1. PolyON-AppEngine (Odoo 19)

- **엔진 특성**: 전형적인 모놀리식 ERP 웹앱. 풍부한 UI(웹 클라이언트)가 1급이고, RPC/JSON API는 있지만 “API-only 헤드리스 엔진”으로 설계된 것은 아님.
- **현재 PolyON 사용 방식**: Console/Portal에서 **iframe으로 Odoo UI를 그대로 로드**하는 구조 (AppEngine 자체가 앱 + UI).
- **Headless 가능성**:
  - 일부 기능은 Odoo JSON-RPC / REST API로 호출 가능하지만, 전체 업무 플로우를 전부 headless API로 재노출하려면 **상당한 커스텀 개발**이 필요.
  - 완전한 headless 모듈로 보기보다는, **“강한 UI를 가진 업무 앱”** 으로 보는 것이 현실적.
- **판단**: **부분적 headless(제한적 API)** 는 가능, **완전 headless 엔진**으로는 부적합.

---

## 2. PolyON-Auto (n8n)

- **엔진 특성**: 워크플로우 자동화 엔진, 자체 에디터 UI + 풍부한 REST API.
- **현재 PolyON 사용 방식**:
  - n8n 공식 이미지를 래핑, PRC(DB/S3/SMTP/Auth)로 설정 주입.
  - OIDC SSO 및 OIDC 설정 DB 주입 스크립트(`polyon-oidc-init.js`) 포함.
- **Headless 관점**:
  - 워크플로우 실행/관리용 REST API가 잘 정의돼 있으며, UI 없이도 플로우 트리거·실행이 가능.
  - n8n 에디터 UI는 설계/운영용으로 필요하지만, “실행기 역할”은 headless로 충분히 동작.
- **판단**: **headless 가능 (엔진+API 중심)** — PolyON-Auto는 headless 워크플로우 엔진으로 사용 가능.

---

## 3. PolyON-Canvas (AFFiNE)

- **엔진 특성**: 노트/화이트보드/문서용 **리치 프론트엔드 앱**. 핵심 가치는 에디터 UI에 있음.
- **현재 PolyON 사용 방식**:
  - 공식 Docker 이미지를 래핑, PRC(DB/RustFS/SMTP/Auth)만 주입.
  - Storage-first(only)를 RustFS로 맞추는 정도의 커스터마이징.
- **Headless 관점**:
  - AFFiNE는 내부적으로 데이터/동기화 엔진이 있지만, 공식적으로 “CMS/BPMN 엔진처럼 API-first”로 설계된 것은 아님.
  - 문서를 외부 시스템에서 headless로 전면 관리하는 시나리오는 **공식 스펙이 부족**.
- **판단**: **사실상 non-headless UI 앱** — API-only 엔진으로 쓰기엔 적합하지 않음.

---

## 4. PolyON-Channel (Mattermost)

- **엔진 특성**: 채팅/협업 플랫폼. Web UI + 모바일/데스크톱 클라이언트를 중심으로 설계, REST/WebSocket API 제공.
- **현재 PolyON 사용 방식**:
  - 공식 Mattermost Team Edition 이미지를 래핑, PRC(DB/RustFS/SMTP/Auth) 주입.
  - FileSettings를 S3(RustFS)로 강제하는 Storage-first(only) 구성이 핵심.
- **Headless 관점**:
  - REST/WebSocket API를 사용해 “봇/통합”은 headless하게 구현 가능.
  - 하지만 전체 채팅 UX를 다른 UI에서 완전히 대체하려면 메시지/채널/알림/검색/파일 등 **풀 스택 재구현**이 필요.
- **판단**: **부분 headless(봇/통합용 API)** 는 가능, **채널 엔진만 쓰는 완전 headless 구성**은 비용 대비 이점이 적음.

---

## 5. PolyON-BPMN (Operaton)

- **엔진 특성**: Camunda 7 후속 BPMN/DMN 엔진. **엔진/REST/Webapps 분리가 명확**.
- **현재 PolyON 사용 방식**:
  - Spring Boot 앱으로 Operaton 엔진 + REST + Webapps를 포함.
  - OIDC/Spring Security + `operaton.bpm.oauth2.*` 로 Webapps 보호.
- **Headless 관점**:
  - Spring Boot에서 Webapps starter를 제거하거나 비활성화하면, **엔진+REST만 남기는 headless 구성이 자연스럽게 가능**.
  - 이미 Operaton 자체가 “BPMN 엔진+REST” 구조라 headless 엔진 사용이 1급 시나리오.
- **판단**: **완전 headless 구성에 매우 적합** — 필요 시 Webapps 없이 엔진+REST만 배포 가능.

---

## 6. PolyON-CMS (Strapi)

- **엔진 특성**: **Headless CMS**가 1급 컨셉. Admin UI는 관리용, 컨텐츠 제공은 REST/GraphQL API.
- **현재 PolyON 사용 방식**:
  - Strapi CE를 기반으로, 업로드는 RustFS(S3) provider로 고정.
  - `/admin`은 oauth2-proxy + Caddy를 통한 Keycloak OIDC 게이트 뒤에 배치.
- **Headless 관점**:
  - 프론트엔드는 완전히 분리 가능(Next.js, React 등 어떤 클라이언트든 사용).
  - API는 REST/GraphQL로 제공되며, PolyON Portal/외부 앱이 전부 소비할 수 있음.
- **판단**: **“Headless” 라는 이름 그대로**, PolyON-CMS는 **완전 headless 구성이 기본값**.

---

## 7. PolyON-Drive (Rust Drive)

- **엔진 특성**: Rust 기반 파일 스토리지 서비스. WebDAV + JSON API + 공유/버전/휴지통 등 제공.
- **현재 PolyON 사용 방식**:
  - PolyON-Drive는 **완전히 API/프로토콜 중심**(WebDAV, REST).
  - 웹 UI는 별도로 있을 수 있지만, 핵심은 “파일 시스템 API”.
- **Headless 관점**:
  - 이미 WebDAV/REST를 통해 어떤 클라이언트(Portal, OS 클라이언트, 모바일 등)에서든 사용할 수 있는 구조.
  - UI를 완전히 제거해도 기능적으로 문제 없고, 오히려 **인프라 계층 headless 서비스**에 가깝다.
- **판단**: **Headless 스토리지 엔진**으로 이미 구현되어 있음.

---

## 8. 요약 — “Headless 가능성” 레벨 분류

- **완전 Headless 엔진/서비스로 사용 가능**
  - **PolyON-Auto** (n8n 워크플로우 엔진 + REST)
  - **PolyON-BPMN** (Operaton BPMN/DMN 엔진 + REST, Webapps 분리 가능)
  - **PolyON-CMS** (Strapi Headless CMS)
  - **PolyON-Drive** (WebDAV/REST 파일 스토리지 엔진)

- **부분 Headless (API는 있지만 UI 성격이 강한 앱)**
  - **PolyON-Channel** (Mattermost) — 봇/통합용 headless 가능, 전체 채팅 UX headless 대체는 비용 큼
  - **PolyON-AppEngine** (Odoo) — 일부 JSON-RPC/REST 가능하지만 ERP 전체를 headless로 쓰기엔 구조적으로 UI 의존도가 높음

- **사실상 Non-headless UI 앱에 가까움**
  - **PolyON-Canvas** (AFFiNE) — 에디터/화이트보드 UI가 본체, 엔진을 독립적인 headless 서비스로 쓰기 어렵다.

---

## 9. PolyON 규칙 관점 정리

- **PolyON이 “엔진/플랫폼” 성격으로 요구하는 headless 모듈**:  
  - BPMN, CMS, Drive, Auto → **요구사항을 충족하거나 충족 가능**.
- **“업무 앱 / 협업 클라이언트 / 지식편집기” 성격의 모듈**:  
  - AppEngine, Channel, Canvas → **UI와 강하게 결합된 앱**으로, headless는 부차적인 보너스 수준.

따라서, **PolyON 코어 엔진/스토리지 계층은 headless 아키텍처를 잘 따르고 있고**,  
업무/협업 UI 계층은 “가능한 곳부터 점진적으로 headless로 끌어올리는 전략”이 현실적인 접근이다.

---

## 10. Odoo/복잡 UI 모듈을 위한 하이브리드 전략 — Micro-Frontend 샌드박스

완전히 headless로 다시 만들 수 없거나, 이미 잘 만들어진 복잡 화면(Odoo 회계/인사, 일부 Mattermost UI 등)을 그대로 활용해야 하는 경우:

- **문제의 본질**  
  - “모두를 headless로 만들면 이상적이지만, 양·복잡도 때문에 현실적으로 어렵다”  
  - 그래도 PolyON Portal 입장에서는 **톤앤매너·네비게이션·레이아웃 일관성**이 필요하다.

- **전략 1: Micro-Frontend 기반 샌드박스 삽입**
  - Portal(예: Next.js) 전체 레이아웃/내비게이션은 그대로 두고,
  - **복잡한 화면만 Odoo의 뷰(또는 웹앱)를 독립 영역(sandbox)으로 삽입**한다.
  - 구현 형태:
    - iframe + 상단/사이드바는 Portal이 제공
    - 또는 Web Component/Module Federation 등으로 별도 번들을 로드
  - 효과:
    - “프레임은 Portal, 알맹이는 Odoo” 구조로 **개발 비용은 줄이고, 사용자는 한 포탈 안에 있다는 느낌**을 유지.

- **전략 2: CSS 변수 기반 스타일 인젝션(Theming)**
  - 샌드박스 안에서도 **Portal의 디자인 시스템(색/폰트/간격)을 강제로 주입**해 시각적 이질감을 최소화.
  - 방법:
    - Web Components/Shadow DOM 경계를 활용해, Portal에서 노출하는 CSS 변수(`--color-primary`, `--font-family` 등)를 Odoo 플러그인/테마에 반영.
  - 효과:
    - 구조는 기존 Odoo UI를 사용하되, **겉으로 보이는 룩앤필은 Portal과 통일**.

이 두 가지를 조합하면:

- **엔진 레벨**: AppEngine은 지금처럼 PolyON 모듈로 SSO/Storage-first/PRC를 맞추고,
- **Portal 레벨**: 
  - 가능한 부분은 완전 headless(React 컴포넌트 + Odoo API)로,
  - 복잡한 부분은 Micro-Frontend 샌드박스로 묶어,
  - 최종적으로는 사용자가 “Odoo 느낌의 화면을 쓴다”는 인식을 최대한 줄이면서도,  
    Odoo의 방대한 기능을 그대로 활용할 수 있게 된다.

PolyON 관점에서 이 하이브리드 전략은:

- “엔진을 headless로 끌어올릴 수 있는 곳은 최대한 끌어올리고,”  
- “그렇지 못한 부분은 샌드박스 + 스타일 동기화로 포탈 경험에 녹인다”  
라는 **실용적인 중간 지점**으로, AppEngine/Channel/Canvas 같은 복잡 UI 모듈을 장기적으로 통합하는 데 유효한 패턴이다.

