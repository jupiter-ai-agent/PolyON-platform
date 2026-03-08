# PolyON Platform Overview

## 1. PolyON이란

PolyON은 **기업 인프라 통합 플랫폼**입니다.

메일, 채팅, 파일 스토리지, 위키, ERP 등 기업에 필요한 서비스들을 개별적으로 운영하는 대신,
하나의 플랫폼 위에서 **계정 통합, 스토리지 통합, 인증 통합**을 제공합니다.

기존 그룹웨어가 "기능 나열"이라면, PolyON은 **"인프라 통합"**입니다.

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                    Traefik Ingress                   │
│              (*.company.com → Services)              │
├─────────────────────────────────────────────────────┤
│                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│   │ Console  │  │  Portal  │  │  Module   │  ...    │
│   │ (Admin)  │  │  (User)  │  │ (Drive)   │         │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│        │              │              │               │
│        └──────────────┼──────────────┘               │
│                       │                              │
│                 ┌─────┴─────┐                        │
│                 │   Core    │  ← Go API Server       │
│                 │  (API)    │                         │
│                 └─────┬─────┘                        │
│                       │                              │
│   ┌───────────────────┼───────────────────┐          │
│   │                   │                   │          │
│   ▼                   ▼                   ▼          │
│ ┌──────┐  ┌────────────────┐  ┌────────────┐        │
│ │Samba │  │  PostgreSQL    │  │   RustFS   │        │
│ │AD DC │  │  (Metadata)    │  │ (S3 Files) │        │
│ └──────┘  └────────────────┘  └────────────┘        │
│                                                     │
│ ┌──────┐  ┌────────────────┐  ┌────────────┐        │
│ │Redis │  │  OpenSearch    │  │  Keycloak  │        │
│ │      │  │  (Full-text)   │  │   (SSO)    │        │
│ └──────┘  └────────────────┘  └────────────┘        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 3. Foundation 인프라 서비스

모든 모듈이 공유하는 인프라 레이어입니다. 모듈은 이 인프라를 직접 사용합니다.

| 서비스 | 기술 | 내부 호스트 | 포트 | 용도 |
|--------|------|-------------|------|------|
| **AD DC** | Samba AD DC | `polyon-dc` | 389 (LDAP) | 계정 원천 (Active Directory) |
| **Database** | PostgreSQL 18 | `polyon-db` | 5432 | 관계형 데이터 저장 |
| **Object Storage** | RustFS | `polyon-rustfs` | 9000 (S3) | 파일/오브젝트 저장 |
| **Cache** | Redis 8 | `polyon-redis` | 6379 | 세션/캐시 |
| **Search** | OpenSearch 3.x | `polyon-search` | 9200 | 전문 검색 |
| **SSO** | Keycloak | `polyon-auth` | 8080 | OIDC/SAML 인증 |
| **Ingress** | Traefik v3 | `traefik` | 443 | 외부 진입점 |

## 4. 핵심 개념

### 4.1 계정 통합 (Identity Unification)

**Samba AD DC가 유일한 계정 원천입니다.**

- 모든 사용자 계정은 AD DC에서 관리
- 서비스는 AD DC에 직접 LDAP bind하여 인증 (Pattern A)
- 또는 Keycloak OIDC를 통해 인증 (Pattern B)
- 계정 생성/삭제/수정은 AD DC에서만 발생 → 모든 서비스에 자동 전파

```
사용자 생성 (AD DC)
    ├──► Mail (Stalwart) — LDAP Federation, 자동 메일함 생성
    ├──► Chat (Mattermost) — OIDC JIT, 자동 계정 생성
    ├──► Drive — LDAP bind, 자동 로그인 + 홈폴더 생성
    └──► 기타 모듈 — 동일 패턴
```

### 4.2 인증 패턴

| 패턴 | 방식 | 사용처 | 설명 |
|------|------|--------|------|
| **Pattern A** | Direct LDAP | Mail, Drive | AD DC에 직접 LDAP bind. 가장 단순하고 안정적 |
| **Pattern B** | Keycloak OIDC | Chat, Wiki | Keycloak이 AD와 Federation. 브라우저 SSO 제공 |

**권장: Pattern A** — 서버 사이드 서비스는 AD DC 직접 연결이 가장 안정적.

### 4.3 오브젝트 스토리지 단일화

**모든 모듈의 파일/오브젝트는 RustFS(S3)에 저장합니다.**

- RustFS는 MinIO 호환 S3 서비스
- 모듈별 버킷 분리 (예: `drive`, `chat-files`, `wiki`)
- 통합 검색, 통합 백업의 전제 조건
- 로컬 PVC에 파일 저장 금지 (앱 코드/config 전용만 허용)

```
RustFS (S3-compatible)
├── drive/          ← Drive 모듈 파일
├── chat-files/     ← Chat 첨부파일
├── wiki/           ← Wiki 에셋
└── backup/         ← 통합 백업
```

### 4.4 데이터베이스

- **공유 PostgreSQL** 인스턴스 (`polyon-db`)
- 모듈별 별도 **database** 분리 (같은 인스턴스, 다른 DB)
- 모듈 설치 시 자동으로 `CREATE DATABASE {module} OWNER {module}` 실행
- 연결 정보는 K8s Secret으로 주입

### 4.5 검색 통합

- **OpenSearch** (Elasticsearch 호환)
- 모듈이 자체 인덱스에 문서를 색인
- 향후 Cross-module 통합 검색 제공

## 5. 모듈 생명주기

```
등록 (Catalog)
    ↓
설치 (Install)  ← K8s 리소스 생성, DB 프로비저닝, Secret 생성
    ↓
프로비저닝 (Provision)  ← LDAP 설정, OIDC 클라이언트 생성
    ↓
활성 (Active)  ← 서비스 운영 중
    ↓
삭제 (Uninstall)  ← K8s 리소스 제거, 데이터 정책에 따라 DB 보존/삭제
```

## 6. 네트워크 구조

### 도메인 패턴

| 패턴 | 형태 | 예시 |
|------|------|------|
| 서브도메인 | `{service}.{domain}` | `drive.company.com` |
| URL 패턴 | `portal.{domain}/{service}` | `portal.company.com/drive` |

모듈 설치 시 사용자가 선택합니다.

### 내부 통신

모듈 간 통신은 **Core API 경유만** 허용합니다 (통제와 감사 원칙).

```
Module A ──► Core API ──► Module B
             │
             └──► Audit Log
```

## 7. UI 구조

| 서비스 | 대상 | URL |
|--------|------|-----|
| **Console** | 관리자 | `console.{domain}` |
| **Portal** | 사원 | `portal.{domain}` |
| **모듈 UI** | 사원 | `{module}.{domain}` 또는 `portal.{domain}/{module}` |

- Console/Portal: **IBM Carbon Design System** (React)
- 모듈 UI: 자유 (단, Portal 내 iframe 또는 독립 페이지)
