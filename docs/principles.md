# PolyON Platform 원칙 (Principles)

> 모든 PP 모듈은 이 원칙을 준수해야 한다.  
> 예외는 보스(대표) 명시적 승인이 필요하며, 이 문서에 기록한다.

---

## 제1원칙 — 계정 통합 (Identity Unification)

**Samba AD DC가 유일한 계정 원천(Single Source of Truth)이다.**

- 모든 사용자 계정은 AD에서 생성·관리된다.
- 모듈이 자체 사용자 DB를 가질 수 있으나, AD 동기화 미러일 뿐이다.
- 모듈 내부에서 사용자 생성/삭제는 금지 (AD → 모듈 단방향).

### 인증 방식

| 방식 | 용도 | 예시 |
|------|------|------|
| **Keycloak OIDC SSO** | 웹 UI 로그인 (1차) | Console, Portal, 모듈 웹 |
| **LDAP bind** | 서버 간 인증, API 인증 | WebDAV, SMTP AUTH, 모듈 API |

- **SSO가 기본**, LDAP 직접 로그인은 보조 경로.
- SSO 미지원 모듈은 LDAP bind 허용 (예외 기록 필수).

### 준수 기준

- [ ] `module.yaml`에 `claims.auth` 선언
- [ ] 사용자 생성 API 차단 또는 AD 동기화 전용
- [ ] 로그인 페이지에서 SSO 리다이렉트가 기본

---

## 제2원칙 — 유기적 통합 (Organic Integration)

**AD에 계정 추가 → 모든 서비스 자동 프로비저닝.**

- SSO 연결만으로는 부족하다. 계정 추가 시 메일함, 드라이브 할당량, 채팅 계정 등이 자동 생성되어야 한다.
- Core가 AD 이벤트를 감지하여 각 모듈에 프로비저닝 API를 호출한다.

### 준수 기준

- [ ] Core OdooSync / MattermostSync 등 동기화 엔드포인트 존재
- [ ] 사용자 최초 로그인 시 자동 계정 생성 (JIT Provisioning)
- [ ] 비활성화된 AD 계정 → 모듈에서도 비활성화

---

## 제3원칙 — 통제와 감사 (Governance & Audit)

**모든 접근은 정책 기반, 모든 행위는 감사 가능.**

- 접근 제어: Keycloak realm role + Core RBAC
- 감사 로그: 최소 1년 보존
- 모듈 간 직접 통신 금지 — Core API 경유만 허용

### 준수 기준

- [ ] 관리자 API는 Core 프록시 경유 (`/api/v1/modules/{id}/proxy/*`)
- [ ] 감사 대상 행위(로그인, 데이터 변경) 로깅
- [ ] 모듈 간 직접 네트워크 호출 없음

---

## 제4원칙 — 제품 품질 UI (Carbon Design)

**Console/Portal은 IBM Carbon Design System으로 통일.**

- `@carbon/react` + `@carbon/icons-react` 필수
- 아이콘: emoji 절대 금지 → Carbon 공식 SVG path만 사용
- Typography: IBM Plex Sans, Carbon type scale
- Layout: Carbon Grid, 8px 단위 spacing

### 모듈 UI 방식

| 방식 | 대상 | 설명 |
|------|------|------|
| **A (Foundation 내장)** | Foundation 서비스 | Console/Portal React 코드로 직접 구현 |
| **C (Module iframe)** | 엔진 모듈 | ModuleSector iframe으로 모듈 자체 UI 로드 |

- iframe 모듈은 PostMessage 프로토콜 준수 (또는 `skipHandshake`)
- 모듈 자체 UI가 있는 경우 (Odoo, Mattermost 등) iframe으로 표시

### 준수 기준

- [ ] Console/Portal 페이지는 Carbon 컴포넌트 사용
- [ ] 모듈 iframe은 X-Frame-Options 제거
- [ ] 아이콘은 `@carbon/icons-react` 32px variant

---

## 제5원칙 — 단일 관문 (Single Gateway)

**Traefik Ingress가 유일한 외부 진입점.**

- 개별 컨테이너/Pod의 호스트 포트 직접 노출 금지
- 모든 외부 트래픽은 Traefik을 경유
- TLS 종단: Traefik에서 처리 (와일드카드 인증서)

### 도메인 구조

| 패턴 | 예시 | 용도 |
|------|------|------|
| `console.{domain}` | console.cmars.com | 관리자 Console |
| `portal.{domain}` | portal.cmars.com | 사용자 Portal |
| `{module}.{domain}` | odoo.cmars.com | 모듈 서브도메인 |
| `sso.{domain}` | sso.cmars.com | Keycloak SSO |

### 준수 기준

- [ ] K8s Ingress 리소스로 라우팅 (hostPort 금지)
- [ ] TLS: Traefik IngressRoute 또는 Ingress annotation

---

## 제6원칙 — 인프라 단일 책임 (Operator as Infrastructure Authority)

**인프라 관장은 Operator/Core만.**

- 컨테이너/라우팅/OIDC client/credential 조작은 Core(PRC)만 수행
- 모듈이 직접 K8s API, Docker 소켓, Keycloak Admin API를 호출하면 안 됨
- 모듈은 PRC claims로 자원을 선언적으로 요청

### 준수 기준

- [ ] `module.yaml`에 `spec.claims`로 자원 선언
- [ ] 모듈 코드에서 K8s API 직접 호출 없음
- [ ] DB/Secret/Ingress는 Core가 프로비저닝

---

## 제7원칙 — 오브젝트 스토리지 단일화 (Object Storage Unification)

**모든 모듈의 파일/오브젝트는 RustFS(S3)를 사용.**

- 로컬 PVC 파일 저장 금지 (통합 검색/백업의 전제)
- 모듈별 S3 버킷 분리 (PRC objectStorage claim)
- RustFS ≠ MinIO: 환경변수 `RUSTFS_*` 사용

### 준수 기준

- [ ] `module.yaml`에 `claims.objectStorage` 선언
- [ ] 파일 업로드/다운로드는 S3 API 경유
- [ ] 컨테이너 내 `/data` 등에 영구 파일 저장 금지

---

## 예외 기록

| 모듈 | 원칙 | 예외 내용 | 승인 |
|------|------|----------|------|
| Drive | 제1원칙 | LDAP bind 1차 인증 (WebDAV 호환) | 2026-03-09 보스 승인 |
| Odoo | 제4원칙 | 자체 UI iframe 사용 (Carbon 미적용) | 엔진 모듈 기본 정책 |
