# PolyON Module Lifecycle Spec — 모듈 설치/삭제 자동화 규격

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 대상: Core 개발자, Operator 개발자, 모듈 개발자, AI 코딩 에이전트

---

## 1. 개요

모듈의 생명주기: **등록 → 설치 → 프로비저닝 → 활성 → 삭제**

각 단계에서 **누가, 무엇을, 어떤 순서로** 수행하는지 정의합니다.

```
┌──────────────────────────────────────────────────────────┐
│                    Module Lifecycle                       │
│                                                          │
│  Catalog ──► Install ──► Provision ──► Active ──► Uninstall
│  (등록)      (설치)      (프로비저닝)  (운영)     (삭제)   │
│                                                          │
│  Core        Core+K8s   Core+Module   Module    Core+K8s │
│  (DB 등록)   (리소스    (LDAP,OIDC    (서비스   (리소스   │
│              생성)      초기화)        제공)    정리)     │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 역할 분담

| 역할 | 책임 |
|------|------|
| **Core** | 모듈 DB 관리, K8s 리소스 생성/삭제, Secret 생성, 프록시, 메뉴 제공 |
| **Operator** | Foundation 인프라 관리 (Core와 별개, 모듈 생명주기에 관여하지 않음) |
| **Module** | 자체 초기화 (DB 마이그레이션, 인덱스 생성), Health 제공, API 제공 |
| **Console** | 관리자 UI (모듈 설치/삭제 트리거, Admin 페이지) |

---

## 3. Phase 1: 등록 (Catalog)

### 3.1 등록 방식

모듈은 두 가지 방식으로 등록됩니다:

**A. Go embed 내장 (Foundation 모듈)**
```
core/internal/module/catalog/
├── drive/
│   └── module.yaml      # 빌드 시 Go embed
├── chat/
│   └── module.yaml
└── wiki/
    └── module.yaml
```

Core 시작 시 embed된 module.yaml을 읽어 `infra_modules` 테이블에 자동 등록합니다.

**B. 외부 등록 API (향후 Module Store)**
```
POST /api/v1/modules/catalog
Content-Type: application/json

{
  "image": "registry.example.com/polyon-erp:v1.0.0",
  "manifest": { ... module.yaml 내용 ... }
}
```

### 3.2 infra_modules 테이블 (Catalog)

```sql
CREATE TABLE infra_modules (
    id          TEXT PRIMARY KEY,       -- module ID (예: "drive")
    name        TEXT NOT NULL,          -- 표시 이름
    version     TEXT NOT NULL,          -- semver
    category    TEXT NOT NULL,          -- engine | tool | integration
    description TEXT,
    icon        TEXT,                   -- @carbon/icons 이름
    vendor      TEXT,
    license     TEXT,
    manifest    JSONB NOT NULL,         -- 전체 module.yaml (JSON 변환)
    status      TEXT NOT NULL DEFAULT 'available',
    -- status: available | installing | active | uninstalling | error
    installed_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.3 Console "시스템 정보" 페이지

Console의 시스템 정보 페이지에서 관리자가 모듈 카탈로그를 조회합니다:

```
GET /api/v1/modules
Response:
[
  {
    "id": "drive",
    "name": "PP Drive",
    "version": "0.1.0",
    "category": "engine",
    "description": "파일 스토리지 서비스",
    "icon": "Document",
    "status": "available",       // 설치 가능
    "requires": ["postgresql", "rustfs", "samba-dc"]
  },
  {
    "id": "chat",
    "name": "PP Chat",
    "status": "active",          // 설치됨
    "installed_at": "2026-03-07T10:00:00Z"
  }
]
```

---

## 4. Phase 2: 설치 (Install)

관리자가 Console에서 "설치" 버튼 클릭 시 실행되는 절차입니다.

### 4.1 설치 API

```
POST /api/v1/modules/{moduleId}/install
Authorization: Bearer {admin-jwt}

Response 202 Accepted:
{
  "id": "drive",
  "status": "installing",
  "progress": { "step": 1, "total": 7, "message": "Secret 생성 중" }
}
```

### 4.2 설치 순서 (7 Steps)

Core가 순차적으로 실행합니다:

```
Step 1: 의존성 확인
Step 2: DB + User 생성 (PostgreSQL)
Step 3: Secret 생성 (K8s)
Step 4: S3 버킷 생성 (RustFS)
Step 5: K8s 워크로드 배포 (Deployment/StatefulSet + Service)
Step 6: Ingress 생성 (선택)
Step 7: Health 확인 + 상태 업데이트
```

### Step 1: 의존성 확인

module.yaml의 `spec.requires` 검사:

```yaml
requires:
  - id: postgresql    # Foundation DB — 항상 존재
  - id: rustfs        # Foundation S3 — 항상 존재
  - id: samba-dc      # Foundation AD — 항상 존재
  - id: chat          # 다른 모듈 의존 (해당 모듈이 active여야 함)
```

- Foundation 인프라: Operator가 설치한 상태이므로 존재 여부만 확인
- 다른 모듈: `infra_modules.status = 'active'` 확인
- 미충족 시: `409 Conflict` 반환

### Step 2: DB + User 생성

Core가 PostgreSQL에 모듈 전용 DB와 유저를 생성합니다:

```sql
-- Core가 admin 연결로 실행
CREATE USER drive WITH PASSWORD '{generated_password}';
CREATE DATABASE drive OWNER drive;
GRANT ALL PRIVILEGES ON DATABASE drive TO drive;
```

- DB 이름: `module.yaml → spec.database.name` (기본: moduleId)
- User 이름: `module.yaml → spec.database.user` (기본: moduleId)
- Password: 랜덤 생성 (a-zA-Z0-9, 24자)
- `spec.database.create: false`이면 이 단계 스킵

### Step 3: Secret 생성

K8s Secret에 연결 정보를 저장합니다:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: polyon-module-drive
  namespace: polyon
  labels:
    polyon.io/module: drive
type: Opaque
stringData:
  DB_HOST: "polyon-db"
  DB_PORT: "5432"
  DB_NAME: "drive"
  DB_USER: "drive"
  DB_PASSWORD: "{generated_password}"
  DATABASE_URL: "postgres://drive:{password}@polyon-db:5432/drive?sslmode=disable"
```

### Step 4: S3 버킷 생성

Core가 RustFS에 모듈 전용 버킷을 생성합니다:

```
PUT http://polyon-rustfs:9000/{bucket_name}
```

- 버킷 이름: module.yaml `spec.engine` 또는 moduleId (예: `drive`)
- 이미 존재하면 스킵

### Step 5: K8s 워크로드 배포

Core가 module.yaml의 `spec.resources`를 기반으로 K8s 리소스를 생성합니다:

```yaml
# Deployment 생성
apiVersion: apps/v1
kind: Deployment
metadata:
  name: polyon-drive
  namespace: polyon
  labels:
    app: polyon-drive
    polyon.io/module: drive
spec:
  replicas: 1     # module.yaml → spec.resources.replicas
  selector:
    matchLabels:
      app: polyon-drive
  template:
    metadata:
      labels:
        app: polyon-drive
        polyon.io/module: drive
    spec:
      containers:
        - name: drive
          image: jupitertriangles/polyon-drive:v0.1.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: polyon-common-config   # POLYON_DC_*, POLYON_AUTH_* 등
            - secretRef:
                name: polyon-common-secret   # POLYON_DC_ADMIN_PASSWORD, RUSTFS keys
            - secretRef:
                name: polyon-module-drive     # DB_*, DATABASE_URL
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

---
# Service 생성
apiVersion: v1
kind: Service
metadata:
  name: polyon-drive
  namespace: polyon
  labels:
    polyon.io/module: drive
spec:
  selector:
    app: polyon-drive
  ports:
    - port: 8080
      targetPort: 8080
```

### Step 6: Ingress 생성 (선택)

module.yaml에 `spec.ingress` 선언이 있으면:

```yaml
# module.yaml
ingress:
  subdomain: drive      # → drive.{domain}
  port: 8080
```

Core가 Ingress 리소스를 생성합니다:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: polyon-drive
  namespace: polyon
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
    - host: drive.${POLYON_DOMAIN}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: polyon-drive
                port:
                  number: 8080
```

> **참고:** Drive 모듈의 WebDAV는 이 Ingress를 통해 외부 접근 가능. 메타데이터 API는 Core 프록시 경유.

### Step 7: Health 확인 + 상태 업데이트

```
1. K8s Deployment Ready 대기 (최대 5분)
2. GET http://polyon-drive:8080/health → 200 OK 확인
3. infra_modules.status = 'active' 업데이트
4. infra_modules.installed_at = now() 기록
```

실패 시: `status = 'error'`, 에러 메시지 기록, 생성된 리소스 롤백(삭제).

### 4.3 설치 진행률 조회

```
GET /api/v1/modules/{moduleId}/progress
Response:
{
  "status": "installing",
  "step": 5,
  "total": 7,
  "message": "K8s 워크로드 배포 중",
  "started_at": "2026-03-09T08:00:00Z"
}
```

Console은 이 API를 폴링하여 설치 진행 상황을 표시합니다.

---

## 5. Phase 3: 프로비저닝 (Provision)

설치 완료 후, 모듈이 자체적으로 수행하는 초기화입니다.

### 5.1 모듈 자체 초기화

모듈 컨테이너가 시작되면 자동으로 수행해야 할 작업:

| 항목 | 주체 | 시점 |
|------|------|------|
| DB 마이그레이션 | 모듈 | 컨테이너 시작 시 (`sqlx migrate`) |
| S3 버킷 확인 | 모듈 | 시작 시 (없으면 생성 시도) |
| 검색 인덱스 생성 | 모듈 | 시작 시 (OpenSearch) |
| LDAP 연결 확인 | 모듈 | 시작 시 (Health에 반영) |

### 5.2 LDAP 프로비저닝 (Core 담당)

module.yaml에 `spec.ldap.bind: true`이면 Core가 수행:

```yaml
ldap:
  bind: true
  engine: drive    # LDAP 서비스 계정 식별자
```

Core가 AD DC에 모듈 전용 서비스 계정을 생성합니다 (향후):
```
CN=svc-drive,CN=Users,DC={domain},DC={tld}
```

> 현재(MVP): Administrator 공용 계정 사용. 서비스 계정 분리는 Phase 2.

### 5.3 OIDC 클라이언트 프로비저닝 (Core 담당, 선택)

모듈이 Keycloak OIDC를 사용하는 경우 (예: Chat, Wiki):

```yaml
# module.yaml
oidc:
  create_client: true
  client_id: polyon-{moduleId}
  client_type: confidential    # confidential | public
  redirect_uris:
    - "https://{moduleId}.{domain}/*"
```

Core가 Keycloak Admin API로 클라이언트를 자동 생성합니다.

> Drive는 Pattern A (Direct LDAP)이므로 OIDC 클라이언트 불필요.

---

## 6. Phase 4: 활성 (Active)

### 6.1 Core 프록시 라우팅

모듈이 `active` 상태가 되면 Core는 프록시 라우팅을 활성화합니다:

```go
// Core 내부
func proxyToModule(moduleId string, path string, r *http.Request) (*http.Response, error) {
    module := getActiveModule(moduleId)
    if module == nil {
        return nil, ErrModuleNotFound
    }

    // 모듈 내부 URL 구성
    port := module.Manifest.Spec.Resources.Ports[0].ContainerPort
    targetURL := fmt.Sprintf("http://polyon-%s:%d%s", moduleId, port, path)

    // 프록시 요청 전달
    return httpProxy(targetURL, r)
}
```

### 6.2 Console/Portal 메뉴 API

```
GET /api/v1/modules/menu
Response:
{
  "console": [
    {
      "moduleId": "drive",
      "name": "PP Drive",
      "icon": "Document",
      "menuGroup": "services",
      "pages": [
        { "id": "overview", "title": "개요", "route": "/modules/drive", "default": true },
        { "id": "storage", "title": "스토리지", "route": "/modules/drive/storage" },
        ...
      ]
    }
  ],
  "portal": [
    {
      "moduleId": "drive",
      "name": "드라이브",
      "icon": "Folder",
      "menuGroup": "apps",
      "pages": [
        { "id": "files", "title": "드라이브", "route": "/drive", "default": true },
        ...
      ]
    }
  ]
}
```

---

## 7. Phase 5: 삭제 (Uninstall)

### 7.1 삭제 API

```
POST /api/v1/modules/{moduleId}/uninstall
Authorization: Bearer {admin-jwt}

Body (선택):
{
  "keep_data": true    // true: DB/S3 보존, false: 전부 삭제
}

Response 202 Accepted
```

### 7.2 삭제 순서 (5 Steps)

```
Step 1: 의존 모듈 확인 (다른 모듈이 이 모듈에 의존하면 거부)
Step 2: K8s 워크로드 삭제 (Deployment + Service + Ingress)
Step 3: Secret 삭제
Step 4: DB/S3 삭제 (keep_data=false인 경우만)
Step 5: 상태 업데이트 (status → 'available')
```

### 7.3 데이터 보존 정책

| `keep_data` | DB | S3 버킷 | Secret |
|-------------|------|---------|--------|
| `true` (기본) | 보존 | 보존 | 삭제 |
| `false` | DROP DATABASE | 버킷 삭제 | 삭제 |

> **재설치 시:** `keep_data=true`로 삭제 후 재설치하면, 기존 DB와 S3 데이터가 그대로 유지됩니다.

---

## 8. 에러 처리

### 8.1 설치 실패 시 롤백

설치 중 어느 단계에서든 실패하면 이전 단계를 역순으로 정리합니다:

```
Step 5 실패 시: Deployment 삭제 → Secret 삭제 → 버킷 삭제 → DB 삭제
Step 3 실패 시: Secret 삭제 → DB 삭제
```

상태: `status = 'error'`, `error_message` 기록

### 8.2 재시도

```
POST /api/v1/modules/{moduleId}/install
```

`status = 'error'`인 모듈에 대해 다시 설치 요청하면:
1. 기존 잔여 리소스 정리
2. Step 1부터 재시작

---

## 9. infra_services 테이블 (런타임 상태)

Core가 모듈 런타임 정보를 관리하는 테이블:

```sql
CREATE TABLE infra_services (
    id            TEXT PRIMARY KEY,           -- moduleId
    display_name  TEXT NOT NULL,
    service_type  TEXT NOT NULL,              -- 'module'
    host          TEXT NOT NULL,              -- 'polyon-drive'
    port          INTEGER NOT NULL,           -- 8080
    health_path   TEXT DEFAULT '/health',
    status        TEXT DEFAULT 'unknown',     -- running | stopped | error | unknown
    health_status TEXT DEFAULT 'unknown',     -- healthy | unhealthy | unknown
    last_health_check TIMESTAMPTZ,
    version       TEXT,
    db_name       TEXT,                       -- 'drive'
    db_user       TEXT,                       -- 'drive'
    s3_bucket     TEXT,                       -- 'drive'
    ingress_host  TEXT,                       -- 'drive.cmars.com'
    created_at    TIMESTAMPTZ DEFAULT now(),
    updated_at    TIMESTAMPTZ DEFAULT now()
);
```

---

## 10. 전체 흐름 요약

```
[관리자가 Console "시스템 정보"에서 "PP Drive 설치" 클릭]
         │
         ▼
  POST /api/v1/modules/drive/install
         │
         ▼
  ┌─── Core 설치 프로세스 ───┐
  │ 1. 의존성 확인 (PG, S3, DC) ✓
  │ 2. CREATE DATABASE drive   ✓
  │ 3. K8s Secret 생성         ✓
  │ 4. S3 버킷 "drive" 생성   ✓
  │ 5. Deployment + Service    ✓
  │ 6. Ingress (drive.domain)  ✓
  │ 7. Health 확인             ✓
  └────────────────────────────┘
         │
         ▼
  [모듈 컨테이너 시작]
  │ - sqlx migrate 실행 (테이블 생성)
  │ - LDAP 연결 확인
  │ - S3 버킷 확인
  │ - /health → 200 OK
         │
         ▼
  [Core: status = 'active']
         │
         ▼
  [Console 사이드바에 "서비스 > 드라이브" 메뉴 자동 표시]
  [Portal 사이드바에 "앱 > 드라이브" 메뉴 자동 표시]
         │
         ▼
  [관리자: Console > 드라이브 > 할당량 관리]
  [사원:   Portal > 드라이브 > 파일 업로드/다운로드]
```

---

## 참조

- [Module Spec](module-spec.md) — module.yaml 규격
- [Module UI Spec](module-ui-spec.md) — UI 통합 규격
- [Admin API Spec](admin-api-spec.md) — 관리자 API
