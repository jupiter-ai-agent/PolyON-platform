# PRC (Platform Resource Claim) 규격서

> PP Foundation 자원에 대한 선언적 프로비저닝 규격
> K8s PVC 모델과 동일한 패러다임: **Claim → Provision → Bind → Inject**

## 1. 개요

PRC는 모듈이 PP Foundation 자원을 **선언적으로 요청**하는 메커니즘이다.

```
모듈 설치 요청
  ↓
module.yaml spec.claims[] 파싱
  ↓
PRC Engine: Topological Sort → Saga Provision
  ↓
각 Provider가 자원 자동 프로비저닝
  ↓
Credentials → K8s Secret 생성 → 모듈 Pod 환경변수 주입
  ↓
모듈 기동 — 환경변수만 읽으면 끝
```

**핵심 원칙:**
- 모듈은 자원을 **직접 생성하지 않는다** — Claim만 선언한다
- PP가 자원을 **자동 프로비저닝**하고, Credentials를 **주입**한다
- 모듈 삭제 시 PP가 자원을 **자동 정리**(Saga Compensation)한다
- 사람의 개입 없이, 설치 → 삭제 → 재설치가 **완전 자동화**된다

## 2. K8s 대응 모델

| K8s | PP PRC | 설명 |
|-----|--------|------|
| **StorageClass** | **Foundation Provider** | 자원 유형 정의 (DB, Cache, Auth 등) |
| **PersistentVolume (PV)** | **Provisioned Resource** | 실제 프로비저닝된 자원 (DB, KC client 등) |
| **PersistentVolumeClaim (PVC)** | **Module Claim** | 모듈의 자원 요청 |
| **CSI Driver** | **Provider Implementation** | 실제 프로비저닝 로직 |
| **Bound/Pending/Lost** | **Claim Status** | 자원 바인딩 상태 |

## 3. Foundation Provider 목록

| # | Provider | Foundation 서비스 | 프로비저닝 대상 |
|---|----------|-----------------|---------------|
| 1 | `database` | PostgreSQL | DB + Role + 권한 |
| 2 | `cache` | Redis | DB 번호 할당 |
| 3 | `search` | OpenSearch | 인덱스 prefix 예약 |
| 4 | `objectStorage` | RustFS (S3) | 버킷 + Access Key |
| 5 | `directory` | Samba AD DC | 서비스 계정 + 그룹 |
| 6 | `smtp` | Stalwart Mail | 도메인 + 계정 |
| 7 | `git` | Gitea | Organization + Repository |
| 8 | `ai` | LiteLLM | API Key + Model 할당 |
| 9 | `auth` | Keycloak | OIDC Client + Secret |

## 4. Claim 선언 규격

### 4.1 module.yaml 내 선언

```yaml
spec:
  claims:
    - type: database
      config:
        name: myapp            # DB 이름
    - type: objectStorage
      config:
        bucket: myapp-files    # S3 버킷 이름
    - type: auth
      config:
        clientId: myapp
        accessType: confidential
        redirectUris:
          - "https://myapp.{{ baseDomain }}/*"
```

### 4.2 Claim 구조

```go
type Claim struct {
    Type     string         `yaml:"type" json:"type"`      // Provider 타입
    Config   map[string]any `yaml:"config" json:"config"`  // Provider별 설정
    ModuleID string         // PRC Engine이 주입
}
```

### 4.3 환경변수 템플릿

```yaml
spec:
  env:
    DB_HOST: "{{ claims.database.host }}"
    DB_PASSWORD: "{{ claims.database.password }}"
    OIDC_CLIENT_ID: "{{ claims.auth.clientId }}"
    OIDC_CLIENT_SECRET: "{{ claims.auth.clientSecret }}"
```

`{{ claims.<type>.<key> }}` — Provider가 반환한 Credentials의 키를 참조한다.

## 5. Auth Provider 상세 규격

### 5.1 Claim 설정

```yaml
- type: auth
  config:
    clientId: odoo                    # KC 클라이언트 ID (필수)
    accessType: confidential          # public | confidential (기본: confidential)
    redirectUris:                     # 허용 리다이렉트 URI (필수)
      - "https://odoo.{{ baseDomain }}/*"
      - "https://portal.{{ baseDomain }}/modules/odoo/*"
    scopes:                           # 요청 스코프 (선택, 기본: openid profile email)
      - openid
      - profile
      - email
    pkce: false                       # PKCE 강제 여부 (선택, 기본: false)
```

### 5.2 accessType 결정 기준

| accessType | 용도 | client_secret | PKCE |
|------------|------|--------------|------|
| `public` | SPA (브라우저 전용) | 없음 | S256 강제 |
| `confidential` | 서버 앱 (백엔드) | 자동 발급 | 선택 |

**기본값: `confidential`** — 서버 앱이 대다수이므로.

### 5.3 Provision 흐름

```
1. Keycloak Admin API 토큰 획득 (내부 URL)
2. 기존 클라이언트 확인 → 있으면 재사용
3. 클라이언트 생성:
   - clientId, protocol: openid-connect
   - publicClient: (accessType == "public")
   - standardFlowEnabled: true
   - redirectUris: config에서 추출, {{ baseDomain }} 치환
   - webOrigins: console.{domain}, portal.{domain} 자동 추가
   - PKCE: public이면 S256 강제, confidential이면 해제
4. confidential이면 client_secret 자동 발급
5. Credentials 반환
```

### 5.4 반환 Credentials

| Key | 설명 | 예시 |
|-----|------|------|
| `issuer` | OIDC Issuer (외부 URL) | `https://sso.cmars.com/realms/polyon` |
| `clientId` | 클라이언트 ID | `odoo` |
| `clientSecret` | 클라이언트 Secret (confidential만) | `8Y57hxiU...` |
| `authEndpoint` | Authorization Endpoint (외부 URL) | `https://sso.cmars.com/.../auth` |
| `tokenEndpoint` | Token Endpoint (**내부 URL**) | `http://polyon-auth:8080/.../token` |
| `tokenEndpointExternal` | Token Endpoint (외부 URL) | `https://sso.cmars.com/.../token` |
| `jwksUri` | JWKS Endpoint (**내부 URL**) | `http://polyon-auth:8080/.../certs` |
| `jwksUriExternal` | JWKS Endpoint (외부 URL) | `https://sso.cmars.com/.../certs` |

**⚠️ 내부/외부 URL 분리 (2026-03-10 교훈)**

| 엔드포인트 | 누가 호출하는가 | URL |
|-----------|--------------|-----|
| `authEndpoint` | **브라우저** (사용자 리다이렉트) | **외부** URL |
| `tokenEndpoint` | **서버** (백채널 토큰 교환) | **내부** URL |
| `jwksUri` | **서버** (JWT 검증) | **내부** URL |

모듈 Pod에서 외부 도메인(`sso.cmars.com`)은 `127.0.0.1`로 해석될 수 있다.
서버 간 통신은 반드시 K8s 내부 서비스명(`polyon-auth:8080`)을 사용한다.

### 5.5 Deprovision 흐름

```
1. Keycloak Admin API 토큰 획득
2. 클라이언트 조회 (clientId → internal UUID)
3. DELETE /admin/realms/{realm}/clients/{uuid}
4. module_claims 상태 → removed
```

### 5.6 Status 확인

| 상태 | 의미 |
|------|------|
| `provisioned` | KC 클라이언트 존재 + 정상 |
| `not_found` | KC 클라이언트 없음 |
| `degraded` | KC 클라이언트 있으나 설정 불일치 |
| `error` | KC 접근 실패 |

## 6. 각 Provider 반환 Credentials 요약

### 6.1 database

| Key | 예시 |
|-----|------|
| `host` | `polyon-db` |
| `port` | `5432` |
| `database` | `mod_odoo` |
| `user` | `mod_odoo` |
| `password` | `(자동생성)` |
| `url` | `postgresql://mod_odoo:...@polyon-db:5432/mod_odoo` |

### 6.2 objectStorage

| Key | 예시 |
|-----|------|
| `endpoint` | `http://polyon-rustfs:9000` |
| `bucket` | `odoo` |
| `accessKey` | `(공유 키)` |
| `secretKey` | `(공유 키)` |

### 6.3 cache

| Key | 예시 |
|-----|------|
| `host` | `polyon-redis` |
| `port` | `6379` |
| `db` | `3` (자동 할당) |

### 6.4 search

| Key | 예시 |
|-----|------|
| `endpoint` | `http://polyon-search:9200` |
| `indexPrefix` | `odoo_` |

### 6.5 directory

| Key | 예시 |
|-----|------|
| `host` | `polyon-dc` |
| `port` | `389` |
| `baseDn` | `DC=cmars,DC=com` |
| `bindDn` | `CN=svc-odoo,CN=Users,...` |
| `bindPassword` | `(자동생성)` |

### 6.6 smtp

| Key | 예시 |
|-----|------|
| `host` | `polyon-mail` |
| `port` | `587` |
| `user` | `noreply@cmars.com` |
| `password` | `(자동생성)` |

### 6.7 git

| Key | 예시 |
|-----|------|
| `endpoint` | `http://polyon-gitea:3000` |
| `org` | `odoo` |
| `token` | `(자동생성)` |

### 6.8 ai

| Key | 예시 |
|-----|------|
| `endpoint` | `http://polyon-ai:4000` |
| `apiKey` | `sk-polyon-...` |

### 6.9 auth

→ **5.4절 참조**

## 7. Saga 실행 모델

### 7.1 프로비저닝 순서

Topological Sort로 의존성 순서 결정:

```
database → (의존 없음, 최우선)
cache → (의존 없음)
search → (의존 없음)
objectStorage → (의존 없음)
directory → (의존 없음)
smtp → (의존 없음)
git → (의존 없음)
ai → (의존 없음)
auth → (의존 없음)
```

현재 Provider 간 의존성 없음 → 병렬 실행 가능 (Phase 2).

### 7.2 실패 시 Saga Compensation

```
Claim 1: database  → ✅ provisioned
Claim 2: auth      → ❌ failed
         ↓
Saga rollback:
  auth: skip (이미 실패)
  database: Deprovision (DROP DATABASE, DROP ROLE)
```

**원칙: All-or-Nothing** — 하나라도 실패하면 전체 롤백.

### 7.3 멱등성

모든 Provider는 멱등이어야 한다:
- Provision: 이미 존재하면 재사용 (CREATE IF NOT EXISTS)
- Deprovision: 없으면 무시 (DROP IF EXISTS)

## 8. DB 스키마

### 8.1 module_claims

```sql
CREATE TABLE module_claims (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id     VARCHAR(64) NOT NULL,
    claim_type    VARCHAR(32) NOT NULL,
    claim_config  JSONB NOT NULL DEFAULT '{}',
    credentials   JSONB,                          -- 프로비저닝 결과 (암호화 권장)
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    error_message TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(module_id, claim_type)
);
```

| status | 의미 |
|--------|------|
| `pending` | Claim 접수됨, 프로비저닝 대기 |
| `provisioning` | 프로비저닝 진행 중 |
| `bound` | 자원 바인딩 완료, 모듈 사용 중 |
| `failed` | 프로비저닝 실패 |
| `removing` | Deprovision 진행 중 |
| `removed` | 자원 정리 완료 |

### 8.2 claim_saga_log

```sql
CREATE TABLE claim_saga_log (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id     VARCHAR(64) NOT NULL,
    action        VARCHAR(20) NOT NULL,    -- provision | deprovision
    claim_type    VARCHAR(32) NOT NULL,
    status        VARCHAR(20) NOT NULL,    -- success | failed | compensated
    duration_ms   INT,
    error_message TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 9. Core Admin API

### 9.1 Provider 목록 조회

```
GET /api/v1/admin/prc/providers

Response:
[
  {
    "type": "database",
    "service": "PostgreSQL",
    "status": "healthy",
    "claimCount": 3,
    "resourceCount": 3
  },
  {
    "type": "auth",
    "service": "Keycloak",
    "status": "healthy",
    "claimCount": 2,
    "resourceCount": 2
  }
]
```

### 9.2 전체 Claim 목록 조회

```
GET /api/v1/admin/prc/claims?status=bound&type=auth

Response:
{
  "items": [
    {
      "id": "uuid-...",
      "moduleId": "odoo",
      "moduleName": "PP Odoo",
      "claimType": "auth",
      "config": { "clientId": "odoo", "accessType": "confidential" },
      "status": "bound",
      "createdAt": "2026-03-10T09:00:00Z",
      "updatedAt": "2026-03-10T09:00:05Z"
    }
  ],
  "total": 1
}
```

### 9.3 모듈별 Claim 조회

```
GET /api/v1/admin/modules/{id}/claims

Response:
[
  { "type": "database", "status": "bound", "config": {...} },
  { "type": "auth", "status": "bound", "config": {...} },
  { "type": "objectStorage", "status": "bound", "config": {...} }
]
```

### 9.4 Provider 상태 상세

```
GET /api/v1/admin/prc/providers/{type}

Response:
{
  "type": "auth",
  "service": "Keycloak",
  "endpoint": "http://polyon-auth:8080",
  "externalUrl": "https://sso.cmars.com",
  "realm": "polyon",
  "status": "healthy",
  "resources": [
    {
      "moduleId": "odoo",
      "resourceId": "odoo",         // KC clientId
      "status": "provisioned",
      "createdAt": "2026-03-10T09:00:00Z"
    }
  ]
}
```

### 9.5 Saga 로그 조회

```
GET /api/v1/admin/prc/saga-log?moduleId=odoo&limit=50

Response:
{
  "items": [
    {
      "moduleId": "odoo",
      "action": "provision",
      "claimType": "auth",
      "status": "success",
      "durationMs": 342,
      "createdAt": "2026-03-10T09:00:03Z"
    }
  ]
}
```

## 10. Console PRC Dashboard

### 10.1 페이지 구조

```
Settings → Platform Resources (새 메뉴)
  ├── Overview (대시보드)
  ├── Providers (Provider 목록)
  ├── Claims (Claim 목록)
  └── Saga Log (프로비저닝 이력)
```

### 10.2 Overview 탭

| 위젯 | 내용 |
|------|------|
| **Provider Status Cards** | 9개 Provider 상태 (Tile: 아이콘 + 이름 + healthy/degraded + claim 수) |
| **Claim Summary** | 전체 Claim 수, Bound/Pending/Failed 분포 (도넛 차트) |
| **Recent Activity** | 최근 Saga 로그 5건 (타임라인) |

### 10.3 Providers 탭

DataTable:
| Provider | Service | Status | Claims | Actions |
|----------|---------|--------|--------|---------|
| database | PostgreSQL | 🟢 Healthy | 3 | Detail → |
| auth | Keycloak | 🟢 Healthy | 2 | Detail → |
| cache | Redis | 🟢 Healthy | 1 | Detail → |

**Detail 클릭 → Provider 상세 페이지:**
- Provider 정보 (endpoint, version)
- 프로비저닝된 Resource 목록
- 관련 Claim 목록

### 10.4 Claims 탭

DataTable (필터: status, type, moduleId):
| Module | Claim Type | Status | Config | Created | Updated |
|--------|-----------|--------|--------|---------|---------|
| PP Odoo | auth | 🟢 Bound | clientId: odoo | 03-10 09:00 | 03-10 09:00 |
| PP Odoo | database | 🟢 Bound | name: odoo | 03-10 09:00 | 03-10 09:00 |
| PP Chat | database | 🟢 Bound | name: chat | 03-07 14:00 | 03-07 14:00 |

### 10.5 Saga Log 탭

DataTable (필터: moduleId, action, status):
| Time | Module | Action | Type | Status | Duration |
|------|--------|--------|------|--------|----------|
| 09:00:05 | odoo | provision | auth | ✅ success | 342ms |
| 09:00:03 | odoo | provision | database | ✅ success | 128ms |
| 09:00:02 | odoo | provision | objectStorage | ✅ success | 89ms |

## 11. 코드 수정 사항 (auth.go 반영 필요)

### 11.1 내부/외부 URL 분리

```go
// 현재 (문제): 외부 URL만 반환
creds["tokenEndpoint"] = oidcBase + "/token"  // sso.cmars.com

// 수정: 내부 URL 반환 + 외부 URL 별도
internalBase := fmt.Sprintf("%s/realms/%s/protocol/openid-connect", p.AdminURL, p.Realm)
creds["tokenEndpoint"] = internalBase + "/token"          // polyon-auth:8080
creds["tokenEndpointExternal"] = oidcBase + "/token"      // sso.cmars.com
creds["jwksUri"] = internalBase + "/certs"                 // polyon-auth:8080
creds["jwksUriExternal"] = oidcBase + "/certs"             // sso.cmars.com
```

### 11.2 accessType 기본값 변경

```go
// 현재: 기본 public
accessType := claim.ConfigString("accessType", "public")

// 수정: 기본 confidential (서버앱이 대다수)
accessType := claim.ConfigString("accessType", "confidential")
```

### 11.3 PKCE 설정 분리

```go
// public → PKCE S256 강제
// confidential → PKCE 해제 (서버가 client_secret으로 인증)
if accessType == "public" {
    clientBody["attributes"] = map[string]string{
        "pkce.code.challenge.method": "S256",
    }
} else {
    clientBody["attributes"] = map[string]string{
        "pkce.code.challenge.method": "",  // 해제
    }
}
```

### 11.4 confidential client_secret 자동 반환

```go
// confidential이면 항상 client_secret 포함
if accessType == "confidential" {
    secret, err := p.getClientSecret(ctx, token, clientID)
    if err == nil && secret != "" {
        creds["clientSecret"] = secret
    }
}
```

## 12. Odoo module.yaml 수정 사항

```yaml
# 변경 전
- type: auth
  config:
    clientId: odoo
    accessType: public          # ← 잘못됨 (서버앱)

# 변경 후
- type: auth
  config:
    clientId: odoo
    accessType: confidential    # ← 서버앱은 confidential
    redirectUris:
      - "https://odoo.{{ baseDomain }}/*"
      - "https://portal.{{ baseDomain }}/modules/odoo/*"
      - "https://console.{{ baseDomain }}/modules/odoo/*"

# 환경변수에 client_secret + 내부 URL 추가
env:
  OIDC_ISSUER: "{{ claims.auth.issuer }}"
  OIDC_CLIENT_ID: "{{ claims.auth.clientId }}"
  OIDC_CLIENT_SECRET: "{{ claims.auth.clientSecret }}"
  OIDC_AUTH_ENDPOINT: "{{ claims.auth.authEndpoint }}"           # 외부 (브라우저용)
  OIDC_TOKEN_ENDPOINT: "{{ claims.auth.tokenEndpoint }}"         # 내부 (서버용)
  OIDC_JWKS_URI: "{{ claims.auth.jwksUri }}"                     # 내부 (서버용)
```
