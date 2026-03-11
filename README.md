# PolyON Platform (PP)

**PolyON**은 기업 인프라 통합 플랫폼입니다.

AD 기반 계정 통합, S3 오브젝트 스토리지, SSO 인증을 제공하며,  
모든 업무 서비스를 하나의 플랫폼 위에서 운영합니다.

---

## 7원칙

| # | 원칙 | 핵심 |
|---|------|------|
| 1 | **계정 통합** | AD DC가 유일한 계정 원천. Keycloak SSO가 기본 인증 |
| 2 | **유기적 통합** | AD 계정 추가 → 메일/채팅/드라이브 자동 프로비저닝 |
| 3 | **통제와 감사** | 정책 기반 접근 제어 + 감사 로그 1년 보존. 모듈 간 직접 통신 금지 |
| 4 | **제품 품질 UI** | IBM Carbon Design System 필수 |
| 5 | **단일 관문** | Traefik Ingress가 유일한 외부 진입점 |
| 6 | **인프라 단일 책임** | Core/PRC만 인프라 조작. 모듈은 claims로 선언적 요청 |
| 7 | **스토리지 단일화** | 모든 파일은 RustFS(S3). 로컬 PVC 저장 금지 |

→ 상세: [**docs/principles.md**](docs/principles.md)

---

## 모듈 현황

| 모듈 | 엔진 | 상태 | PRC | SSO |
|------|------|------|:---:|:---:|
| [PP Drive](https://github.com/jupiter-ai-agent/PolyON-Drive) | Rust (자체) | ✅ 운영 | ✅ | 병행 |
| [PP Channel](https://github.com/jupiter-ai-agent/PolyON-Channel) | Mattermost | ✅ 운영 | ✅ | 병행 |
| [PP ERP](https://github.com/jupiter-ai-agent/PolyON-AppEngine) | Odoo 19 | ⚠️ SSO 검증 중 | ✅ | SSO 강제 |
| [PP Canvas](https://github.com/jupiter-ai-agent/PolyON-Canvas) | AFFiNE | ⛔ 미배포 | 설계만 | — |
| [PP BPMN](https://github.com/jupiter-ai-agent/PolyON-BPMN) | Operaton | ⛔ 미배포 | 설계만 | — |
| [PP CMS](https://github.com/jupiter-ai-agent/PolyON-CMS) | Strapi | ⛔ 미배포 | 설계만 | — |
| [PP Auto](https://github.com/jupiter-ai-agent/PolyON-Auto) | n8n | ⛔ 빈 리포 | — | — |

→ 상세: [**docs/module-catalog.md**](docs/module-catalog.md)

---

## 문서 체계

### 플랫폼 규격 (모듈 전반)

| 문서 | 설명 |
|------|------|
| [**Principles**](docs/principles.md) | 7원칙 상세 정의 — 준수 기준, 예외 기록 |
| [**Module Catalog**](docs/module-catalog.md) | 모듈별 현황 — 상태, PRC 준수도, 인증 방식, 이슈 |
| [Platform Overview](docs/platform-overview.md) | 아키텍처와 설계 개요 |
| [Module Spec](docs/module-spec.md) | 모듈 기술 규격 — module.yaml, 환경변수, K8s 리소스 |
| [Module Lifecycle](docs/module-lifecycle-spec.md) | 설치/삭제 자동화 규격 |
| [Foundation Modules](docs/foundation-modules-spec.md) | Foundation 서비스 상세 |

### PRC (Platform Resource Claim)

| 문서 | 설명 |
|------|------|
| [PRC Specification](docs/platform-resource-claim-spec.md) | PRC 전체 규격 — K8s PVC 모델, Saga, Dual Driver |
| [PRC Provider Reference](docs/prc-provider-reference.md) | 9개 Provider 상세 — Config, Credentials, 동작 |

### UI · API

| 문서 | 설명 |
|------|------|
| [Module UI Spec](docs/module-ui-spec.md) | UI 통합 — 하이브리드 A/C, ModuleSector, 동적 메뉴 |
| [Module SDK Spec](docs/module-sdk-spec.md) | iframe ↔ 호스트 통신 SDK |
| [PostMessage Protocol](docs/postmessage-protocol-spec.md) | Console ↔ Module iframe 프로토콜 |
| [Admin API Spec](docs/admin-api-spec.md) | 관리자 API — Core 프록시, 인증 |
| [User API Spec](docs/user-api-spec.md) | 사원용 API — 파일/공유/검색/WebDAV |
| [Integration Guide](docs/integration-guide.md) | 인프라 연동 코드 예제 |

### 모듈별 문서 (`docs/modules/{id}/`)

각 모듈 폴더에 **README.md** (개요/상태) + **REQUIREMENTS.md** (PP 원칙 기반 요구사항) 포함.

| 모듈 | 폴더 | 추가 문서 |
|------|------|----------|
| [Drive](docs/modules/drive/) | `drive/` | [RFP](docs/modules/drive/rfp.md) |
| [Odoo](docs/modules/odoo/) | `odoo/` | [RFP](docs/modules/odoo/rfp.md), [Task](docs/modules/odoo/task-addons.md) |
| [Channel](docs/modules/channel/) | `channel/` | — |
| [Canvas](docs/modules/canvas/) | `canvas/` | — |
| [BPMN](docs/modules/bpmn/) | `bpmn/` | — |
| [CMS](docs/modules/cms/) | `cms/` | — |
| [Auto](docs/modules/auto/) | `auto/` | — |

> **Cursor 개발 워크플로:** `REQUIREMENTS.md`를 읽고 구현 → 팀장(Jupiter) 검증 → 빌드/배포

### 분석 보고서

| 문서 | 설명 |
|------|------|
| [규칙·일관성 분석](docs/PolyON-Modules-규칙-일관성-분석.md) | 모듈별 PP 규칙 준수도 분석 |
| [Auth/SSO 분석](docs/PolyON-Modules-Auth-SSO-분석.md) | 모듈별 인증 방식 비교 |
| [Headless 분석](docs/HEADLESS-PolyON-Modules-분석.md) | Headless 구성 가능성 분석 |

### SDK

| SDK | 언어 | 용도 |
|-----|------|------|
| [PolyON-sdk-go](https://github.com/jupiter-ai-agent/PolyON-sdk-go) | Go | 자체 개발 모듈용 PRC config + OIDC 검증 |
| [PolyON-sdk-python](https://github.com/jupiter-ai-agent/PolyON-sdk-python) | Python | 자체 개발 모듈용 PRC config + OIDC 검증 |

> **SDK 사용 현실:** 기존 오픈소스 엔진(Odoo, Mattermost 등)은 SDK 대신 환경변수로 PRC 자원을 주입.  
> SDK는 자체 개발 모듈(Drive 등)에 적합.

---

## 런타임 환경

| 구성요소 | 기술 |
|---------|------|
| Orchestration | Kubernetes (Docker Desktop K8s) |
| Ingress | Traefik v3 |
| Database | PostgreSQL 18 (pgvector) |
| Cache | Redis 8 |
| Object Storage | RustFS (S3-compatible) |
| Search | OpenSearch 3.5 |
| Identity | Samba AD DC + Keycloak OIDC |
| TLS | Self-signed wildcard (production: Let's Encrypt) |

---

## 라이선스

Platform documentation: MIT  
Individual modules may have their own licenses.

---

**Triangle.s** — https://triangles.co.kr
