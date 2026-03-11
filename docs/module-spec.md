# PolyON Module Spec

PP(PolyON Platform) 위에서 동작하는 모듈의 기술 규격입니다.

## 1. 모듈이란

모듈은 PP Foundation 인프라 위에서 동작하는 **독립 서비스**입니다.

- 자체 Docker 이미지로 패키징
- K8s Deployment 또는 StatefulSet으로 배포
- Foundation 인프라(AD, DB, S3, Search)를 공유
- Core API를 통해 다른 모듈과 통신

## 2. 모듈 요구사항

### 2.1 필수 (MUST)

| 항목 | 설명 |
|------|------|
| **Docker 이미지** | OCI 호환 컨테이너 이미지. `linux/amd64` + `linux/arm64` 멀티플랫폼 |
| **module.yaml** | 이미지 내 `/polyon-module/module.yaml` — PP 규격 매니페스트 |
| **PRC Claims** | `spec.claims`로 필요한 자원 선언 (database, objectStorage, auth 등) |
| **Keycloak SSO** | 웹 UI는 Keycloak OIDC SSO가 기본 인증 (제1원칙) |
| **Health Endpoint** | HTTP `GET /health` 또는 `/status` — K8s probe 용 |
| **환경변수 설정** | 모든 외부 연결 정보는 PRC가 주입하는 환경변수로 수신 (하드코딩 금지) |
| **Stateless** | 앱 상태는 DB/S3에 저장. 컨테이너 재시작 시 데이터 손실 없음 |

### 2.2 권장 (SHOULD)

| 항목 | 설명 |
|------|------|
| **SDK 사용** | 자체 개발 모듈은 PolyON SDK (Go/Python)로 PRC config 파싱 + OIDC 검증 |
| **OpenSearch 연동** | 검색 기능이 있으면 PRC `search` claim으로 OpenSearch에 인덱싱 |
| **REST API** | Core가 호출할 수 있는 REST API 제공 |
| **Graceful Shutdown** | SIGTERM 수신 시 연결 정리 후 종료 |
| **PostMessage 프로토콜** | iframe 모듈은 `polyon:ready` → `polyon:init` handshake 구현 |

### 2.3 금지 (MUST NOT)

| 항목 | 설명 |
|------|------|
| 자체 사용자 DB | 사용자 정보는 AD DC에서만 관리. 모듈 내 사용자 생성 금지 |
| 로컬 파일 저장 | PVC에 사용자 파일 저장 금지 → RustFS(S3) 사용 (제7원칙) |
| 다른 모듈 직접 호출 | 모듈 간 통신은 Core API 경유만 (제3원칙) |
| 호스트 포트 바인딩 | 외부 노출은 Traefik Ingress만 (제5원칙) |
| 자체 인증 시스템 | AD DC 또는 Keycloak만 사용. 자체 회원가입/로그인 금지 (제1원칙) |
| K8s API 직접 호출 | 인프라 조작은 Core/PRC만 (제6원칙) |
| 구형 `requires` 사용 | ~~`spec.requires`~~는 폐기됨. `spec.claims`만 사용 |

## 3. 인프라 연동 규격

### 3.1 AD DC (LDAP)

```
Host:     polyon-dc (or polyon-dc.polyon.svc.cluster.local)
Port:     389 (plain LDAP)
Base DN:  DC={domain},DC={tld}  (예: DC=company,DC=com)
Bind DN:  CN=Administrator,CN=Users,DC={domain},DC={tld}
Protocol: LDAP (not LDAPS — K8s 내부 네트워크)
```

**사용자 검색 필터:**
```ldap
(&(objectClass=user)(!(objectClass=computer))(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
```

**로그인 필터:**
```ldap
(&(objectClass=user)(|(sAMAccountName=%uid)(mail=%uid)(userPrincipalName=%uid)))
```

**주요 속성:**
| AD 속성 | 용도 |
|---------|------|
| `sAMAccountName` | 사용자 ID (로그인명) |
| `displayName` | 표시 이름 |
| `mail` | 이메일 주소 |
| `memberOf` | 그룹 소속 |
| `userPrincipalName` | UPN (user@domain.com) |

**시스템 계정 (제외 대상):**
`admin`, `administrator`, `system-bot`, `system`, `guest`, `polyon`, `advisor`, `krbtgt`

### 3.2 PostgreSQL

```
Host:     polyon-db
Port:     5432
SSL:      disable (K8s 내부)
```

- 모듈별 DB와 유저가 자동 생성됩니다
- 연결 정보는 환경변수로 주입:
  - `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`
  - 또는 `DATABASE_URL` (connection string)
- Secret name: `polyon-module-{moduleId}`

### 3.3 RustFS (S3-compatible Object Storage)

```
Endpoint: polyon-rustfs:9000
Protocol: HTTP (K8s 내부)
Style:    Path-style (not virtual-hosted)
API:      S3-compatible (AWS SDK 사용 가능)
```

- 모듈별 전용 버킷 사용 (예: `drive`, `chat-files`)
- 버킷은 모듈 설치 시 자동 생성
- 인증: Access Key / Secret Key (환경변수로 주입)
  - `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`

### 3.4 OpenSearch

```
Host:     polyon-search:9200
Protocol: HTTP
API:      Elasticsearch 7.x compatible
```

- 모듈별 인덱스 prefix 사용 (예: `drive_files`, `chat_messages`)
- 인증: 없음 (K8s 내부)

### 3.5 Redis

```
Host:     polyon-redis:6379
Protocol: RESP
Auth:     없음 (K8s 내부)
```

- 세션, 캐시, pub/sub 용도
- 모듈별 key prefix 사용 (예: `drive:session:*`)

### 3.6 Keycloak (OIDC SSO)

```
Internal: http://polyon-auth:8080
External: https://auth.{domain}
Realm:    polyon
```

- OIDC Discovery: `https://auth.{domain}/realms/polyon/.well-known/openid-configuration`
- 클라이언트 생성은 Operator가 수행
- 클라이언트 타입: Confidential (서버 앱) 또는 Public (SPA)

## 4. 환경변수 규격

모듈에 주입되는 환경변수:

### 4.1 자동 주입 (polyon-common-config ConfigMap)

| 변수 | 설명 | 예시 |
|------|------|------|
| `POLYON_DOMAIN` | 기본 도메인 | `company.com` |
| `POLYON_DC_HOST` | AD DC 호스트 | `polyon-dc` |
| `POLYON_DC_PORT` | AD DC 포트 | `389` |
| `POLYON_DC_BASE_DN` | LDAP Base DN | `DC=company,DC=com` |
| `POLYON_DC_ADMIN_DN` | LDAP Admin DN | `CN=Administrator,CN=Users,...` |
| `POLYON_AUTH_URL` | Keycloak 내부 URL | `http://polyon-auth:8080` |
| `POLYON_AUTH_EXTERNAL_URL` | Keycloak 외부 URL | `https://auth.company.com` |

### 4.2 자동 주입 (polyon-common-secret Secret)

| 변수 | 설명 |
|------|------|
| `POLYON_DC_ADMIN_PASSWORD` | AD DC 관리자 비밀번호 |
| `POLYON_RUSTFS_ACCESS_KEY` | RustFS S3 Access Key |
| `POLYON_RUSTFS_SECRET_KEY` | RustFS S3 Secret Key |

### 4.3 모듈별 Secret (polyon-module-{id})

| 변수 | 설명 |
|------|------|
| `DB_HOST` | PostgreSQL 호스트 |
| `DB_PORT` | PostgreSQL 포트 |
| `DB_NAME` | 모듈 DB 이름 |
| `DB_USER` | 모듈 DB 유저 |
| `DB_PASSWORD` | 모듈 DB 비밀번호 |
| `DATABASE_URL` | PostgreSQL 연결 문자열 |

## 5. K8s 리소스 구조

```yaml
# 모듈 배포 시 자동 생성되는 리소스
Namespace: polyon

# 1. Secret
polyon-module-{id}    # DB 연결, 모듈별 시크릿

# 2. Deployment 또는 StatefulSet
polyon-{id}           # 모듈 워크로드

# 3. Service
polyon-{id}           # ClusterIP 서비스

# 4. Ingress (선택)
polyon-{id}           # 외부 접근 시
```

### Labels

```yaml
labels:
  app: polyon-{moduleId}
  polyon.io/module: {moduleId}
```

## 6. Module Manifest (module.yaml)

모듈 이미지의 `/polyon-module/module.yaml`에 위치하는 자기완결적 패키지 명세:

```yaml
apiVersion: polyon.io/v1
kind: Module

metadata:
  id: drive              # 모듈 고유 ID
  name: PP Drive         # 표시 이름
  version: 1.0.0
  category: engine       # engine | tool | integration
  description: "파일 스토리지 서비스"
  vendor: "Your Company"
  license: MIT

spec:
  engine: drive

  requires:              # 의존 인프라
    - id: postgresql
    - id: rustfs
    - id: samba-dc

  resources:
    image: your-registry/pp-drive:v1.0.0
    replicas: 1
    ports:
      - name: http
        containerPort: 8080
    env:
      - name: DB_HOST
        valueFrom:
          secretKeyRef:
            name: polyon-module-drive
            key: DB_HOST
    health:
      path: /health
      port: 8080
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits: { cpu: 500m, memory: 512Mi }

  database:
    create: true
    name: drive
    user: drive

  ingress:
    subdomain: drive
    port: 8080

  ldap:
    bind: true
    engine: drive
```

## 7. 통신 규격

### 7.1 Core → Module (Health Check)

```
GET http://polyon-{id}:{port}/health
Response: 200 OK
```

### 7.2 Core → Module (API)

```
POST http://polyon-{id}:{port}/api/v1/{resource}
Authorization: Bearer {internal-jwt}
Content-Type: application/json
```

### 7.3 Module → Core (Callback)

```
POST http://polyon-core:8000/api/v1/modules/{moduleId}/callback
Authorization: Bearer {module-token}
Content-Type: application/json
```
