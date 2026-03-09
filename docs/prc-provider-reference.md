# PRC Provider Reference

> PolyON Platform Resource Claim — Provider별 상세 명세  
> Version: 1.0.0 | Date: 2026-03-09

---

## 개요

PRC(Platform Resource Claim)는 모듈이 `module.yaml`의 `spec.claims`에 선언적으로 자원을 요청하면, Core가 자동으로 프로비저닝하는 시스템이다.

이 문서는 **9개 Foundation Provider**의 정확한 사양을 정의한다.  
모듈 개발자는 이 문서만 보고 claims를 작성할 수 있어야 한다.

---

## 공통 규칙

### Claim 구조

```yaml
spec:
  claims:
    - type: <provider_type>    # 필수: 아래 8종 중 하나
      config:                  # 선택: Provider별 설정
        key: value
```

### 환경변수 템플릿

```yaml
spec:
  env:
    MY_VAR: "{{ claims.<type>.<credentialKey> }}"
```

- 문법: `{{ claims.<claimType>.<credentialKey> }}`
- 공백 허용: `{{ claims.database.url }}` = `{{claims.database.url}}`
- 미해석 템플릿은 에러 반환 (설치 실패)

### 프로비저닝 순서

Provider 간 의존성은 Kahn's Topological Sort로 자동 해결.  
모듈 개발자는 순서를 신경 쓸 필요 없다.

### 멱등성

모든 Provider는 멱등(idempotent). 이미 존재하는 자원은 재생성하지 않는다.

---

## 1. database — PostgreSQL

**Foundation:** PostgreSQL (polyon-db)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `name` | string | `{moduleId}` | DB 이름 접두사. 실제 DB명: `polyon_{name}` |
| `extensions` | string | `""` | 쉼표 구분 PG extension 목록 (예: `"pgvector,uuid-ossp"`) |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `url` | `postgres://mod_drive:xxx@polyon-db:5432/polyon_drive?sslmode=disable` | 완전한 연결 URL |
| `host` | `polyon-db` | DB 호스트 |
| `port` | `5432` | DB 포트 |
| `database` | `polyon_drive` | DB 이름 |
| `user` | `mod_drive` | 자동 생성된 DB 유저 (형식: `mod_{name}`) |
| `password` | `XyLZwBRxh85NZh9LN0KvKGTi` | 자동 생성된 24자 랜덤 비밀번호 |

### 프로비저닝 동작

1. `mod_{name}` 역할(ROLE) 생성 (LOGIN 권한)
2. `polyon_{name}` 데이터베이스 생성
3. 모듈 유저에게 DB 전체 권한 부여 (GRANT ALL)
4. 요청된 extension 설치

### 보상(Deprovision)

- 모든 연결 종료 → DROP DATABASE → DROP ROLE

### 예시

```yaml
claims:
  - type: database
    config:
      name: drive
      extensions: "uuid-ossp"
env:
  DATABASE_URL: "{{ claims.database.url }}"
```

---

## 2. objectStorage — RustFS (S3 호환)

**Foundation:** RustFS (polyon-rustfs)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `bucket` | string | `{moduleId}` | 버킷 이름 |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `endpoint` | `http://polyon-rustfs:9000` | S3 API 엔드포인트 (scheme 포함) |
| `bucket` | `drive` | 생성된 버킷 이름 |
| `accessKey` | `polyon-admin` | S3 접근 키 |
| `secretKey` | `WPrLeDLLr...` | S3 비밀 키 |

> ⚠️ Phase 1: 관리자 키 공유 방식. Phase 2에서 모듈별 IAM 키 분리 예정.

### 프로비저닝 동작

1. 버킷 존재 여부 확인
2. 미존재 시 `MakeBucket` 생성

### 보상(Deprovision)

- 버킷 내 전체 오브젝트 삭제 → 버킷 삭제

### 예시

```yaml
claims:
  - type: objectStorage
    config:
      bucket: drive
env:
  S3_ENDPOINT: "{{ claims.objectStorage.endpoint }}"
  S3_BUCKET: "{{ claims.objectStorage.bucket }}"
  S3_ACCESS_KEY: "{{ claims.objectStorage.accessKey }}"
  S3_SECRET_KEY: "{{ claims.objectStorage.secretKey }}"
```

---

## 3. directory — Samba AD DC (LDAP)

**Foundation:** Samba DC (polyon-dc)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `ou` | string | `Services` | 서비스 계정이 생성될 OU 이름 |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `url` | `ldap://polyon-dc:389` | LDAP 연결 URL |
| `bindDN` | `CN=svc-drive,OU=drive,DC=cmars,DC=com` | 서비스 계정 DN |
| `bindPassword` | `BNq1vFPJW...` | 서비스 계정 비밀번호 (24자 랜덤) |
| `baseDN` | `DC=cmars,DC=com` | 검색 Base DN |
| `host` | `polyon-dc` | LDAP 호스트 |
| `port` | `389` | LDAP 포트 |

### 프로비저닝 동작

1. `OU={ou}` 조직단위 생성 (samba-tool ou create)
2. `svc-{moduleId}` 서비스 계정 생성 (samba-tool user create)
3. 비밀번호 설정

### 보상(Deprovision)

- 서비스 계정 삭제 → OU 삭제

### 예시

```yaml
claims:
  - type: directory
    config:
      ou: drive
env:
  LDAP_URL: "{{ claims.directory.url }}"
  LDAP_BIND_DN: "{{ claims.directory.bindDN }}"
  LDAP_BIND_PASSWORD: "{{ claims.directory.bindPassword }}"
  LDAP_BASE_DN: "{{ claims.directory.baseDN }}"
```

---

## 4. cache — Redis

**Foundation:** Redis (polyon-redis)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `db` | string | `"auto"` | Redis DB 번호. `"auto"` = 3~15에서 자동 할당 |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `url` | `redis://polyon-redis:6379/5` | 완전한 Redis URL |
| `host` | `polyon-redis` | Redis 호스트 |
| `port` | `6379` | Redis 포트 |
| `db` | `5` | 할당된 DB 번호 |

### DB 번호 할당 규칙

| DB | 용도 |
|----|------|
| 0 | system (예약) |
| 1 | core (예약) |
| 2 | litellm (예약) |
| 3~15 | 모듈용 (auto 할당) |

### 프로비저닝 동작

1. `redis_db_allocations` 테이블에서 미할당 DB 번호 탐색
2. 해당 번호를 모듈에 할당 (DB 레코드 UPDATE)

### 보상(Deprovision)

- 할당 해제 (module_id = NULL)
- Redis FLUSHDB는 수행하지 않음 (데이터 보존)

### 예시

```yaml
claims:
  - type: cache
env:
  REDIS_URL: "{{ claims.cache.url }}"
```

---

## 5. search — OpenSearch

**Foundation:** OpenSearch (polyon-search)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `index` | string | `{moduleId}` | 인덱스 이름 |
| `shards` | int | `1` | 프라이머리 샤드 수 |
| `replicas` | int | `0` | 레플리카 수 |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `url` | `http://polyon-search:9200` | OpenSearch 엔드포인트 |
| `index` | `drive` | 생성된 인덱스 이름 |

### 프로비저닝 동작

1. 인덱스 존재 여부 확인 (HEAD)
2. 미존재 시 PUT으로 인덱스 생성 (shards/replicas 설정 포함)

### 보상(Deprovision)

- DELETE로 인덱스 삭제

### 예시

```yaml
claims:
  - type: search
    config:
      index: drive-files
      shards: 1
env:
  SEARCH_URL: "{{ claims.search.url }}"
  SEARCH_INDEX: "{{ claims.search.index }}"
```

---

## 6. smtp — Stalwart Mail

**Foundation:** Stalwart Mail (polyon-mail)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `domain` | string | `{moduleId}` | 발송 도메인 식별자 |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `host` | `polyon-mail` | SMTP 호스트 |
| `port` | `587` | SMTP 포트 (Submission) |
| `user` | `svc-drive@cmars.com` | SMTP 인증 유저 |
| `password` | `abc123...` | SMTP 인증 비밀번호 |

### 프로비저닝 동작

1. Stalwart에 서비스 계정 생성 (Phase 2: Admin API 연동)
2. 발송 권한 부여

### 보상(Deprovision)

- 서비스 계정 삭제 (Phase 2)

### 예시

```yaml
claims:
  - type: smtp
env:
  SMTP_HOST: "{{ claims.smtp.host }}"
  SMTP_PORT: "{{ claims.smtp.port }}"
  SMTP_USER: "{{ claims.smtp.user }}"
  SMTP_PASSWORD: "{{ claims.smtp.password }}"
```

---

## 7. git — Gitea

**Foundation:** Gitea (polyon-gitea)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `org` | string | `polyon-modules` | Gitea 조직 이름 |
| `repo` | string | `{moduleId}-data` | 저장소 이름 |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `url` | `http://polyon-gitea:3000/polyon-modules/drive-data.git` | Git 저장소 URL |
| `token` | `giteapat_xxxx` | Gitea API 토큰 |
| `org` | `polyon-modules` | 조직 이름 |
| `repo` | `drive-data` | 저장소 이름 |

### 프로비저닝 동작

1. 조직 존재 여부 확인, 미존재 시 생성
2. 저장소 존재 여부 확인, 미존재 시 생성
3. 모듈 전용 Access Token 발급

### 보상(Deprovision)

- 저장소 삭제 (조직은 공유 자원이므로 유지)

### 예시

```yaml
claims:
  - type: git
    config:
      repo: drive-config
env:
  GIT_REPO_URL: "{{ claims.git.url }}"
  GIT_TOKEN: "{{ claims.git.token }}"
```

---

## 8. ai — LiteLLM (AI Gateway)

**Foundation:** LiteLLM (polyon-ai)

### Config

| Key | Type | Default | 설명 |
|-----|------|---------|------|
| `rateLimit` | string | `""` | 분당 요청 제한 (예: `"100/min"`) |

### Credentials (반환값)

| Key | 예시 | 설명 |
|-----|------|------|
| `endpoint` | `http://polyon-ai:4000/v1` | OpenAI 호환 API 엔드포인트 |
| `apiKey` | `sk-mod-drive-xxxx` | 모듈 전용 API 키 |

### 프로비저닝 동작

1. LiteLLM Admin API로 모듈 전용 Virtual Key 생성
2. 키 별칭: `mod-{moduleId}`
3. Rate Limit 설정 (옵션)

### 보상(Deprovision)

- Virtual Key 삭제

### 예시

```yaml
claims:
  - type: ai
    config:
      rateLimit: "60/min"
env:
  AI_ENDPOINT: "{{ claims.ai.endpoint }}"
  AI_API_KEY: "{{ claims.ai.apiKey }}"
```

---

## 전체 예시: PP Drive module.yaml

```yaml
apiVersion: polyon.io/v1
kind: Module

metadata:
  id: drive
  name: PP Drive
  version: "0.1.1"
  category: engine
  description: "파일 스토리지 서비스"
  icon: Document
  vendor: "Triangle.s"

spec:
  engine: drive

  requires:
    - id: postgresql
    - id: rustfs
    - id: samba-dc

  resources:
    image: jupitertriangles/polyon-drive:v0.1.1
    replicas: 1
    ports:
      - name: http
        containerPort: 8080
    health:
      path: /health
      port: 8080
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits:   { cpu: 500m, memory: 512Mi }

  claims:
    - type: database
      config:
        name: drive
    - type: objectStorage
      config:
        bucket: drive
    - type: directory
      config:
        ou: drive

  env:
    DATABASE_URL: "{{ claims.database.url }}"
    S3_ENDPOINT: "{{ claims.objectStorage.endpoint }}"
    S3_BUCKET: "{{ claims.objectStorage.bucket }}"
    S3_ACCESS_KEY: "{{ claims.objectStorage.accessKey }}"
    S3_SECRET_KEY: "{{ claims.objectStorage.secretKey }}"
    LDAP_URL: "{{ claims.directory.url }}"
    LDAP_BIND_DN: "{{ claims.directory.bindDN }}"
    LDAP_BIND_PASSWORD: "{{ claims.directory.bindPassword }}"
    LDAP_BASE_DN: "{{ claims.directory.baseDN }}"
    S3_REGION: "us-east-1"
    RUST_LOG: "info"

  ingress:
    subdomain: drive
    port: 8080
```

---

---

## 9. auth — Keycloak OIDC 클라이언트

### 역할
모듈 설치 시 **Keycloak `polyon` realm에 OIDC 클라이언트를 자동 등록**한다.  
PP 제1원칙(모든 서비스는 Keycloak SSO)을 모듈 수준에서 자동화.

### Config

| Key | 설명 | 기본값 |
|-----|------|--------|
| `clientId` | Keycloak 클라이언트 ID | `{moduleId}` |
| `accessType` | `public` (PKCE SPA) 또는 `confidential` | `public` |
| `redirectUris` | 허용 리다이렉트 URI 목록 | `["https://{moduleId}.{baseDomain}/*"]` |
| `webOrigins` | CORS 허용 출처 | `["https://console.{baseDomain}", "https://portal.{baseDomain}"]` |

### Credential (주입되는 환경변수)

| Key | 설명 | 예시 |
|-----|------|------|
| `issuer` | OIDC Issuer URL | `https://auth.cmars.com/realms/polyon` |
| `clientId` | 등록된 클라이언트 ID | `odoo` |
| `clientSecret` | 시크릿 (confidential만) | `a3f8b2c1-...` |
| `authEndpoint` | Authorization URL | `https://auth.cmars.com/realms/polyon/protocol/openid-connect/auth` |
| `tokenEndpoint` | Token URL | `https://auth.cmars.com/realms/polyon/protocol/openid-connect/token` |
| `jwksUri` | JWKS URL | `https://auth.cmars.com/realms/polyon/protocol/openid-connect/certs` |

### env 템플릿 예시

```yaml
env:
  OIDC_ISSUER: "{{ claims.auth.issuer }}"
  OIDC_CLIENT_ID: "{{ claims.auth.clientId }}"
  # confidential 타입인 경우:
  # OIDC_CLIENT_SECRET: "{{ claims.auth.clientSecret }}"
  OIDC_AUTH_ENDPOINT: "{{ claims.auth.authEndpoint }}"
  OIDC_TOKEN_ENDPOINT: "{{ claims.auth.tokenEndpoint }}"
  OIDC_JWKS_URI: "{{ claims.auth.jwksUri }}"
```

### 프로비저닝 동작

1. Keycloak Admin API (`/admin/realms/polyon/clients`) 호출
2. 클라이언트 생성: ID, accessType, redirectUris, webOrigins 설정
3. public 타입: PKCE 활성화 (`pkce.code.challenge.method = S256`)
4. confidential 타입: client secret 생성 후 Secret에 주입
5. 삭제 시: 클라이언트 삭제 (Saga 보상)

### Saga 보상
- **Provision**: Keycloak 클라이언트 생성
- **Compensate**: Keycloak 클라이언트 삭제

### module.yaml 예시

```yaml
claims:
  - type: auth
    config:
      clientId: odoo
      accessType: public
      redirectUris:
        - "https://odoo.{{ baseDomain }}/*"
        - "https://console.{{ baseDomain }}/modules/odoo/*"
        - "https://portal.{{ baseDomain }}/modules/odoo/*"
```

---

## 주의사항

### 1. Config 기본값
모든 config 필드는 선택사항. 생략 시 `moduleId` 기반 기본값 적용.

### 2. Credential Key는 정확히 사용
`{{ claims.database.URL }}`처럼 대소문자를 틀리면 미해석 에러 발생.  
이 문서의 Key 컬럼을 정확히 참조할 것.

### 3. 정적 환경변수
템플릿이 아닌 고정값도 `spec.env`에 선언 가능:
```yaml
env:
  RUST_LOG: "info"          # 정적값 (템플릿 아님)
  DB: "{{ claims.database.url }}"  # 템플릿
```

### 4. Claims 없는 모듈
`spec.claims`가 없으면 PRC를 건너뛰고 기존 방식(수동 Secret)으로 설치.

### 5. Saga 보상
프로비저닝 중 하나라도 실패하면 이미 생성된 자원을 역순으로 롤백(Saga Compensation).

### 6. 멱등 재설치
같은 모듈 재설치 시 이미 존재하는 자원(DB, 버킷 등)은 재생성하지 않음.  
데이터는 보존됨.
