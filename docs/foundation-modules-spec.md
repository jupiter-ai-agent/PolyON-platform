# PolyON Foundation Modules Specification

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 승인: CMARS (대표) | 2026-03-09
> 대상: Core/Operator 개발자, 모듈 개발자, AI 코딩 에이전트

---

## 1. 개요

PolyON Platform(PP)의 Foundation은 모든 비즈니스 모듈이 의존하는 **공유 인프라**입니다.
Foundation 없이는 모듈이 동작할 수 없으며, Foundation은 삭제할 수 없습니다.

이 문서는 Foundation 8종 서비스의 역할, 프로비저닝 인터페이스, 모듈에 제공하는
자원의 형태를 **포괄적으로** 정의합니다.

### 1.1 Foundation 3계층

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Platform Layer                             │
│                Traefik · Core · Console · Portal                    │
│            (라우팅, API, 관리 UI, 사용자 UI — Claim 대상 아님)       │
├─────────────────────────────────────────────────────────────────────┤
│                        Capability Layer                             │
│                     ⑧ AI Gateway (polyon-ai)                       │
│          무상태 · API 기반 · AI 자원 통합 관문 · 제어 · 감사         │
├─────────────────────────────────────────────────────────────────────┤
│                       Infrastructure Layer                          │
│   ① PostgreSQL  ② Redis  ③ OpenSearch  ④ RustFS                   │
│   ⑤ Samba AD DC  ⑥ Stalwart Mail  ⑦ Gitea                        │
│              상태 유지 · 데이터 저장 · 항상 실행                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 부트스트랩 순서 (Topological Sort 결과)

```
Layer 0 (독립)     : ① PG  ② Redis  ③ OpenSearch  ④ RustFS  + Traefik
Layer 1 (L0 의존)  : ⑤ Samba AD DC
Layer 2 (L0+L1)    : ⑥ Stalwart Mail(→PG,DC,OpenSearch,RustFS)
                     Keycloak(→PG,DC)
Layer 3 (L0~L2)    : ⑦ Gitea(→PG,DC,RustFS)
                     ⑧ AI Gateway(→PG,Redis)
Layer 4 (전부)     : Core → Console → Portal
```

---

## 2. Foundation #1 — PostgreSQL

### 2.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `database` |
| **기술** | PostgreSQL 18 (pgvector 확장 지원) |
| **K8s 서비스** | `polyon-db` |
| **포트** | 5432 |
| **이미지** | `pgvector/pgvector:pg18` |
| **계층** | Infrastructure |
| **의존** | 없음 (Layer 0) |

### 2.2 역할

PP의 **단일 관계형 데이터 저장소**. 모든 Foundation 서비스와 비즈니스 모듈이
동일한 PG 인스턴스를 공유하되, **모듈별 별도 DATABASE로 격리**합니다.

### 2.3 현재 사용처

| Database | 소유자 | 용도 |
|----------|--------|------|
| `keycloak` | polyon | Keycloak 세션/설정 |
| `stalwart` | polyon | Stalwart Mail 메타데이터 |
| `polyon` | polyon | Core 메타데이터 (modules, claims, apps) |
| `polyon_drive` | mod_drive | Drive 파일 메타/버전/공유/즐겨찾기 |

### 2.4 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: database
    config:
      name: drive                  # → DB명: polyon_drive
      extensions: [pgvector]       # 선택: PG 확장
```

**Operator 실행:**
```sql
-- 1. 전용 유저 생성
CREATE ROLE mod_drive LOGIN PASSWORD '{generated_24char}';

-- 2. 전용 DB 생성
CREATE DATABASE polyon_drive OWNER mod_drive;

-- 3. 확장 설치 (요청 시)
\c polyon_drive
CREATE EXTENSION IF NOT EXISTS pgvector;

-- 4. 권한 제한 — 자기 DB만 접근
REVOKE ALL ON DATABASE postgres FROM mod_drive;
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `url` | `postgres://mod_drive:{password}@polyon-db:5432/polyon_drive` |

### 2.5 Deprovision (삭제 시)

```sql
-- dataPolicy: delete 일 때만
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='polyon_drive';
DROP DATABASE IF EXISTS polyon_drive;
DROP ROLE IF EXISTS mod_drive;
```

### 2.6 운영 규칙

- **polyon 슈퍼유저에 CREATEROLE + CREATEDB 권한 필수** (모듈 DB 자동 생성 전제)
- 비밀번호: 24자 `a-zA-Z0-9-_` (특수문자 `&!^*+#=` 금지)
- 모듈별 유저는 자기 DB만 접근 가능 (GRANT 격리)

---

## 3. Foundation #2 — Redis

### 3.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `cache` |
| **기술** | Redis 8 Alpine |
| **K8s 서비스** | `polyon-redis` |
| **포트** | 6379 |
| **이미지** | `redis:8-alpine` |
| **계층** | Infrastructure |
| **의존** | 없음 (Layer 0) |

### 3.2 역할

**세션 캐시, 임시 데이터, Pub/Sub 메시지 브로커**. 모듈별 DB 번호(0~15)로 격리합니다.

### 3.3 DB 번호 할당 정책

| DB # | 용도 |
|------|------|
| 0 | 시스템 예약 (Core 세션) |
| 1 | Keycloak 세션 |
| 2 | Stalwart 캐시 |
| 3~15 | 모듈에 동적 할당 |

### 3.4 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: cache
    config:
      db: auto                    # auto = 미사용 번호 자동 할당
      maxMemory: 64mb             # 선택: 메모리 제한
```

**Operator 실행:**
```go
// 1. 미할당 DB 번호 조회
var dbNum int
row := pool.QueryRow(ctx,
    "SELECT db_number FROM redis_db_allocations WHERE module_id IS NULL ORDER BY db_number LIMIT 1")
row.Scan(&dbNum)

// 2. 할당 등록
pool.Exec(ctx,
    "UPDATE redis_db_allocations SET module_id=$1, allocated_at=now() WHERE db_number=$2",
    claim.ModuleID, dbNum)

// 3. maxMemory 설정 (CONFIG SET 또는 Redis ACL)
// → Phase 2에서 Redis ACL로 DB별 접근 제어 추가
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `url` | `redis://polyon-redis:6379/3` |

### 3.5 Deprovision

```go
// DB 번호 반납
pool.Exec(ctx,
    "UPDATE redis_db_allocations SET module_id=NULL, allocated_at=NULL WHERE module_id=$1",
    moduleID)

// 해당 DB의 데이터 FLUSH
redisClient.Do(ctx, "SELECT", dbNum)
redisClient.Do(ctx, "FLUSHDB")
```

---

## 4. Foundation #3 — OpenSearch

### 4.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `search` |
| **기술** | OpenSearch 3.5.0 |
| **K8s 서비스** | `polyon-search` |
| **포트** | 9200 |
| **이미지** | `opensearchproject/opensearch:3.5.0` |
| **계층** | Infrastructure |
| **의존** | 없음 (Layer 0) |

### 4.2 역할

**전문 검색(FTS), 로그 수집, 분석**. Stalwart Mail의 이메일 검색,
모듈의 콘텐츠 검색, 감사 로그 저장에 사용됩니다.

### 4.3 현재 사용처

| 인덱스 패턴 | 소유자 | 용도 |
|-------------|--------|------|
| `st_*` | Stalwart | 이메일 전문 검색 |

### 4.4 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: search
    config:
      index: drive-files          # 인덱스 이름
      shards: 1                   # 선택 (기본: 1)
      replicas: 0                 # 선택 (기본: 0)
```

**Operator 실행:**
```bash
# 1. 인덱스 생성
PUT http://polyon-search:9200/drive-files
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}

# 2. 인덱스 템플릿 (모듈이 추가 매핑을 정의할 수 있도록 열어둠)
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `url` | `http://polyon-search:9200` |
| `index` | `drive-files` |

### 4.5 Deprovision

```bash
DELETE http://polyon-search:9200/drive-files
```

### 4.6 운영 규칙

- **ES 9.x 비호환**: Stalwart v0.15.5가 ES 9.x에서 인덱스 생성 실패 → OpenSearch 3.x 사용
- **메모리**: Mac Mini에서 exit 137 (OOM) 빈발 → 2GB 힙 제한 권장
- **`network.host: 0.0.0.0` 필수**: 없으면 Docker 네트워크 내 접근 불가

---

## 5. Foundation #4 — RustFS

### 5.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `objectStorage` |
| **기술** | RustFS 1.0.0-alpha (S3-compatible) |
| **K8s 서비스** | `polyon-rustfs` |
| **포트** | 9000 (S3 API), 9001 (Console) |
| **이미지** | `rustfs/rustfs:1.0.0-alpha.85` |
| **계층** | Infrastructure |
| **의존** | 없음 (Layer 0) |

### 5.2 역할

**PP의 단일 오브젝트 스토리지**. 모든 파일(메일 첨부, 채팅 파일, 드라이브 문서,
위키 에셋 등)이 RustFS에 저장됩니다. 로컬 PVC에 파일 저장 금지.

### 5.3 현재 버킷 구조

| 버킷 | 소유자 | 용도 |
|------|--------|------|
| `drive` | Drive 모듈 | 사용자 파일 |
| (향후) `chat-files` | Chat 모듈 | 채팅 첨부파일 |
| (향후) `wiki` | Wiki 모듈 | 위키 에셋 |
| (향후) `mail-blobs` | Stalwart | 이메일 본문/첨부 |
| (향후) `backup` | Operator | 통합 백업 |

### 5.4 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: objectStorage
    config:
      bucket: drive               # 버킷 이름
      quota: 50Gi                 # 선택: 용량 제한
```

**Operator 실행:**
```go
// 1. 버킷 생성 (S3 API)
_, err := s3Client.CreateBucket(ctx, &s3.CreateBucketInput{
    Bucket: aws.String("drive"),
})

// 2. 모듈 전용 접근키 발급 (RustFS Admin API)
accessKey := fmt.Sprintf("mod-%s-access", moduleID)
secretKey := generateSecurePassword(32)
// RustFS mc admin user add ...

// 3. 버킷 정책 — 해당 접근키는 자기 버킷만 접근 가능
// mc admin policy set ...
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `endpoint` | `http://polyon-rustfs:9000` |
| `bucket` | `drive` |
| `accessKey` | `mod-drive-access` |
| `secretKey` | `{generated_32char}` |

### 5.5 Deprovision

```go
// dataPolicy: delete 일 때만
// 1. 버킷 내 전체 오브젝트 삭제
// 2. 버킷 삭제
s3Client.DeleteBucket(ctx, &s3.DeleteBucketInput{Bucket: aws.String("drive")})
// 3. 접근키 삭제
```

### 5.6 운영 규칙

- **RustFS ≠ MinIO**: 환경변수 `RUSTFS_ACCESS_KEY`/`RUSTFS_SECRET_KEY` (MINIO_ROOT 아님)
- **실행 유저**: `uid=10001(rustfs)` — PVC 디렉토리 `chown 10001:10001`
- **볼륨 초기화**: `/data` 아래 서브디렉토리 없으면 에러 → `mkdir -p /data/vol1`
- **헬스 엔드포인트 없음**: tcpSocket 체크 사용

---

## 6. Foundation #5 — Samba AD DC

### 6.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `directory` |
| **기술** | Samba AD DC 4.x |
| **K8s 서비스** | `polyon-dc` |
| **포트** | 389 (LDAP), 636 (LDAPS), 88 (Kerberos), 53 (DNS) |
| **이미지** | `jupitertriangles/polyon-dc:202603` |
| **계층** | Infrastructure |
| **의존** | 없음 (Layer 1, 단 PG 불필요) |

### 6.2 역할

**PP의 유일한 계정 원천 (Identity Source of Truth)**.
제1원칙 — "AD DC가 유일한 계정 원천. 모든 서비스는 이 위에."

- 사용자/그룹 계정 관리
- LDAP 디렉토리 서비스 (모듈이 직접 바인딩)
- Kerberos 인증
- DNS 서비스

### 6.3 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: directory
    config:
      access: read                # read | readWrite
      ou: Services                # 서비스 계정 OU (기본: Services)
```

**Operator 실행:**
```bash
# 1. 서비스 계정 OU 확인/생성
samba-tool ou list | grep Services || samba-tool ou create "OU=Services"

# 2. 서비스 계정 생성
samba-tool user create svc-drive '{generated_password}' \
  --given-name="Drive Service" \
  --userou="OU=Services"

# 3. 읽기 전용인 경우 제한된 ACL 설정
# readWrite인 경우 Users OU 쓰기 권한 부여
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `url` | `ldap://polyon-dc:389` |
| `bindDN` | `CN=svc-drive,OU=Services,DC=cmars,DC=com` |
| `bindPassword` | `{generated_24char}` |
| `baseDN` | `DC=cmars,DC=com` |
| `usersDN` | `CN=Users,DC=cmars,DC=com` |

### 6.4 Deprovision

```bash
# 서비스 계정 비활성화 (삭제보다 안전)
samba-tool user disable svc-drive
# 또는 완전 삭제
samba-tool user delete svc-drive
```

### 6.5 운영 규칙

- **DC hostname 고정**: `DC1.cmars.com` (변경 시 AD 깨짐)
- **DC DNS IP 고정**: `172.20.0.11` (절대 변경 금지)
- **plain LDAP**: entrypoint에 `ldap server require strong auth = no` 추가 필수 (K8s)
- **privileged 모드**: NT ACL / xattr 필요 → K8s에서도 privileged 필수
- **`/shared` 볼륨 필수**: setup.json 없으면 무한 대기
- **LDAP filter**: `(&(objectClass=user)(|(sAMAccountName=?)(mail=?)(userPrincipalName=?)))`

---

## 7. Foundation #6 — Stalwart Mail

### 7.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `smtp` |
| **기술** | Stalwart Mail v0.15.5 |
| **K8s 서비스** | `polyon-mail` |
| **포트** | 25 (SMTP), 587 (Submission), 993 (IMAPS), 4190 (Sieve) |
| **이미지** | `jupitertriangles/polyon-mail:202603` |
| **계층** | Infrastructure |
| **의존** | PG, DC, OpenSearch, RustFS (Layer 2) |

### 7.2 역할

**PP의 이메일 서비스**. SMTP 발송, IMAP 수신, 전문 검색을 제공합니다.
모듈에게는 프로그래밍 방식의 **이메일 발송 기능**을 Claim으로 제공합니다.

### 7.3 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: smtp
    config:
      domain: drive               # 발송 주소: drive@{base_domain}
```

**Operator 실행:**
```bash
# 1. AD DC에 발송 전용 서비스 계정 생성 (또는 기존 svc-drive 재사용)
# 2. Stalwart에 발송 권한 설정 (API 또는 config)
# 3. 발송 전용 자격증명 생성
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `host` | `polyon-mail` |
| `port` | `587` |
| `user` | `svc-drive@cmars.com` |
| `password` | `{generated_24char}` |
| `from` | `drive@cmars.com` |

### 7.4 Deprovision

```bash
# 발송 계정 비활성화
```

### 7.5 운영 규칙

- **v0.15.5 PostgreSQL 직접 저장 불가**: `--init`은 RocksDB만 지원. Data/Lookup = RocksDB
- **fallback-admin**: flat key 형식 (`authentication.fallback-admin.secret = "..."`)
- **`session.auth.directory` 문법**: `"ldap"` ✗ → `"'ldap'"` ✓ (single-quoted literal)
- **directory 이름 "ldap" 통일**: `ad-ldap` 사용 금지
- **Traefik TCP passthrough**: 25/587/993/4190 포트는 Traefik에서 TCP 수준 프록시

---

## 8. Foundation #7 — Gitea

### 8.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `git` |
| **기술** | Gitea (최신 stable) |
| **K8s 서비스** | `polyon-gitea` |
| **포트** | 3000 (HTTP), 22 (SSH) |
| **이미지** | `gitea/gitea:latest` (공식 이미지 — 보스 승인) |
| **계층** | Infrastructure |
| **의존** | PG, DC, RustFS (Layer 3) |

### 8.2 역할

**PP의 소스 코드 관리 + Git 저장소**. 두 가지 역할을 수행합니다:

1. **사용자용**: 개발팀의 Git 저장소, 코드 리뷰, 이슈 트래킹
2. **모듈용**: 모듈이 버전 관리가 필요한 데이터(설정, 템플릿, IaC 등)를 저장

### 8.3 부트스트랩 설정

```ini
; Gitea app.ini (Operator가 생성)
[database]
DB_TYPE  = postgres
HOST     = polyon-db:5432
NAME     = polyon_gitea
USER     = mod_gitea
PASSWD   = {generated}

[server]
ROOT_URL = https://git.{base_domain}

[service]
DISABLE_REGISTRATION = true    ; AD DC 계정만 사용

[openid]
ENABLE_OPENID_SIGNIN = true    ; Keycloak OIDC

[storage]
STORAGE_TYPE = minio
MINIO_ENDPOINT = polyon-rustfs:9000
MINIO_BUCKET = gitea
MINIO_ACCESS_KEY_ID = {generated}
MINIO_SECRET_ACCESS_KEY = {generated}
```

### 8.4 Claim → Provision

**모듈 요청:**
```yaml
claims:
  - type: git
    config:
      org: polyon-modules          # Gitea organization
      repo: drive-data             # 레포지토리명
```

**Operator 실행:**
```bash
# Gitea Admin API 사용

# 1. Organization 확인/생성
POST http://polyon-gitea:3000/api/v1/orgs
{
  "username": "polyon-modules",
  "visibility": "private"
}

# 2. Repository 생성
POST http://polyon-gitea:3000/api/v1/orgs/polyon-modules/repos
{
  "name": "drive-data",
  "private": true,
  "auto_init": true
}

# 3. 모듈 전용 API 토큰 발급
POST http://polyon-gitea:3000/api/v1/users/{svc-user}/tokens
{
  "name": "mod-drive-token",
  "scopes": ["write:repository"]
}
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `url` | `http://polyon-gitea:3000/polyon-modules/drive-data.git` |
| `token` | `gitea_mod_drive_{token}` |
| `apiUrl` | `http://polyon-gitea:3000/api/v1` |

### 8.5 Deprovision

```bash
# dataPolicy: delete 일 때
DELETE http://polyon-gitea:3000/api/v1/repos/polyon-modules/drive-data
# 토큰 삭제
DELETE http://polyon-gitea:3000/api/v1/users/{svc-user}/tokens/{tokenId}
```

### 8.6 운영 규칙

- **공식 이미지 사용** (보스 승인 — 설정 기반 서비스)
- **AD DC LDAP 연동**: `ENABLE_LDAP = true`, LDAP 소스 추가
- **RustFS 저장**: LFS + 첨부파일을 RustFS에 저장 (로컬 PVC 최소화)

---

## 9. Foundation #8 — AI Gateway

### 9.1 Profile

| 항목 | 값 |
|------|-----|
| **Claim Type** | `ai` |
| **기술** | LiteLLM Proxy (Python) + PP Integration Layer |
| **K8s 서비스** | `polyon-ai` |
| **포트** | 8080 |
| **이미지** | `jupitertriangles/polyon-ai:v{semver}` (LiteLLM 베이스) |
| **계층** | Capability |
| **의존** | PG, Redis (Layer 3) |

### 9.2 역할

AI Gateway는 **PP의 AI 자원 통합 관문**입니다.
단순 API 프록시가 아니라, 조직의 AI 사용을 **통제하고 감사**하는 시스템입니다.

#### 기술 선택: LiteLLM (보스 승인 2026-03-09)

| 항목 | 값 |
|------|-----|
| **베이스** | [LiteLLM Proxy](https://github.com/BerriAI/litellm) (MIT 라이선스) |
| **언어** | Python — PP 내 유일한 Python 컨테이너 (스택 오염 없음) |
| **채택 이유** | 100+ Provider 지원, Virtual Keys, Spend Tracking, Rate Limit, PG 감사 로그 **기 구현** |
| **커버리지** | PP 요구사항의 **~80% 기 구현** → 자체 개발은 PP 통합 레이어만 |
| **대안 검토** | Bifrost (Go, 경량) — Provider/감사 기능 부족으로 자체 개발량 과다 → 불채택 |

**PP 자체 개발 범위 (Integration Layer):**
| 항목 | 설명 |
|------|------|
| LiteLLM config.yaml 자동 생성 | Operator 부트스트랩 시 Provider 설정 주입 |
| PRC → Virtual Key 발급 | LiteLLM `/key/generate` API로 모듈별 토큰 생성 |
| Console 관리 페이지 | Provider 관리, 정책, 사용량 대시보드 (Carbon Design) |
| Kill Switch | LiteLLM `/key/update` (budget=0) 또는 `/key/delete` |
| Dockerfile | LiteLLM 베이스 이미지 + PP 설정 오버레이 |

```
┌───────────────────────────────────────────────────────────────────┐
│                        AI Gateway (polyon-ai)                     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    ① 인증 + 인가                            │ │
│  │  모듈 토큰 검증 → 허용 모델/capability 확인 → Rate Limit   │ │
│  └──────────────────────────┬──────────────────────────────────┘ │
│                             │                                     │
│  ┌──────────────────────────▼──────────────────────────────────┐ │
│  │                    ② 라우팅 + 부하 분산                     │ │
│  │  capability + 정책 + 가용성 → provider 선택                │ │
│  │  fallback: primary 실패 → secondary provider               │ │
│  └──┬──────────┬──────────┬──────────┬────────────────────────┘ │
│     │          │          │          │                           │
│  ┌──▼───┐  ┌──▼────┐  ┌──▼───┐  ┌──▼──────────┐               │
│  │OpenAI│  │Claude │  │Ollama│  │ 기타 Provider│               │
│  │ API  │  │  API  │  │(로컬)│  │  (vLLM 등)  │               │
│  └──────┘  └───────┘  └──────┘  └─────────────┘               │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    ③ 제어 (Control)                         │ │
│  │  • 모듈별 모델 허용/차단 정책                               │ │
│  │  • Rate Limit (분/시/일/월 단위)                            │ │
│  │  • 비용 상한 (월간 $한도)                                   │ │
│  │  • 콘텐츠 필터링 (입출력 정책)                              │ │
│  │  • 긴급 차단 (kill switch)                                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    ④ 감사 (Audit)                           │ │
│  │  • 모든 요청/응답 메타데이터 기록 (PG)                     │ │
│  │  • 토큰 사용량 추적 (입력/출력 토큰 수)                    │ │
│  │  • 비용 계산 (provider별 단가 × 사용량)                    │ │
│  │  • 모듈별 / 사용자별 / provider별 대시보드                 │ │
│  │  • 이상 감지 (급격한 사용량 증가 알림)                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### 9.3 핵심 기능 4가지

#### ① 인증 + 인가 (Authentication & Authorization)

```
모듈 요청:
  POST /v1/chat/completions
  Authorization: Bearer mod_drive_xxxx
  
AI Gateway:
  1. mod_drive_xxxx 토큰 검증 → module_id=drive 확인
  2. drive 모듈의 허용 capability 확인 → chat ✅
  3. drive 모듈의 허용 모델 확인 → gpt-4o ✅
  4. Rate Limit 확인 → 일일 1000회 중 432회 사용 → 통과
  5. → Provider로 포워딩
```

#### ② 라우팅 + 부하 분산 (Routing & Load Balancing)

관리자가 설정한 정책에 따라 Provider를 선택합니다:

| 정책 | 설명 |
|------|------|
| **priority** | 우선순위 기반 (예: Ollama 먼저 → 실패 시 OpenAI) |
| **cost-optimized** | 비용 최적화 (저렴한 provider 우선) |
| **latency-optimized** | 응답 속도 우선 |
| **round-robin** | 균등 분배 |

```go
type RoutingPolicy struct {
    Strategy    string            // priority | cost | latency | round-robin
    Providers   []ProviderConfig  // 우선순위 순
    Fallback    string            // 전부 실패 시 최종 provider
}
```

#### ③ 제어 (Control)

**Console 관리 화면에서 설정:**

```yaml
# 전역 정책
globalPolicy:
  allowedModels:
    - gpt-4o
    - gpt-4o-mini
    - claude-sonnet-4-20250514
    - text-embedding-3-small
    - local/llama3.1       # Ollama 로컬 모델
  
  defaultRateLimit: 1000/day
  defaultCostCap: "$30/month"
  
  contentFilter:
    enabled: true
    blockPatterns: ["비밀번호", "사내비밀"]  # 출력 필터링

# 모듈별 정책 (전역 위에 override)
moduleOverrides:
  chat:
    allowedModels: [gpt-4o, gpt-4o-mini, local/llama3.1]
    rateLimit: 5000/day
    costCap: "$100/month"
  
  drive:
    allowedModels: [gpt-4o-mini, text-embedding-3-small]
    rateLimit: 1000/day
    costCap: "$20/month"
    
  workstream:
    allowedModels: [gpt-4o, claude-sonnet-4-20250514, text-embedding-3-small]
    rateLimit: 10000/day
    costCap: "$200/month"
```

**Kill Switch (긴급 차단):**
```bash
# Console → Core API → AI Gateway
POST /api/v1/admin/ai/kill
{ "scope": "all" }        # 전체 차단
{ "scope": "module", "moduleId": "chat" }  # 특정 모듈만 차단
{ "scope": "provider", "provider": "openai" }  # 특정 provider 차단
```

#### ④ 감사 (Audit)

**모든 AI 요청의 메타데이터를 기록합니다:**

```sql
CREATE TABLE ai_audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id       VARCHAR(64) NOT NULL,      -- 요청한 모듈
    user_id         VARCHAR(256),              -- 요청 발생시킨 사용자 (추적 가능 시)
    provider        VARCHAR(32) NOT NULL,      -- openai | anthropic | ollama
    model           VARCHAR(64) NOT NULL,      -- gpt-4o | llama3.1 등
    capability      VARCHAR(32) NOT NULL,      -- chat | embedding | vision | tts | stt
    
    -- 사용량
    input_tokens    INT NOT NULL DEFAULT 0,
    output_tokens   INT NOT NULL DEFAULT 0,
    total_tokens    INT NOT NULL DEFAULT 0,
    
    -- 비용 (USD cents)
    cost_cents      NUMERIC(10,4) NOT NULL DEFAULT 0,
    
    -- 성능
    latency_ms      INT NOT NULL,
    status          VARCHAR(16) NOT NULL,      -- success | error | rate_limited | blocked
    error_message   TEXT,
    
    -- 메타
    request_id      UUID NOT NULL,             -- 추적용 고유 ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 인덱스
CREATE INDEX idx_ai_audit_module ON ai_audit_log(module_id, created_at);
CREATE INDEX idx_ai_audit_provider ON ai_audit_log(provider, created_at);
CREATE INDEX idx_ai_audit_user ON ai_audit_log(user_id, created_at);

-- 비용 단가 테이블
CREATE TABLE ai_pricing (
    provider    VARCHAR(32) NOT NULL,
    model       VARCHAR(64) NOT NULL,
    input_cost  NUMERIC(10,6) NOT NULL,    -- per 1K tokens (USD)
    output_cost NUMERIC(10,6) NOT NULL,    -- per 1K tokens (USD)
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (provider, model)
);

-- 월간 사용량 집계 뷰
CREATE VIEW ai_monthly_usage AS
SELECT
    module_id,
    provider,
    model,
    date_trunc('month', created_at) AS month,
    COUNT(*) AS request_count,
    SUM(input_tokens) AS total_input_tokens,
    SUM(output_tokens) AS total_output_tokens,
    SUM(cost_cents) AS total_cost_cents
FROM ai_audit_log
WHERE status = 'success'
GROUP BY module_id, provider, model, date_trunc('month', created_at);
```

### 9.4 API 인터페이스

AI Gateway는 **OpenAI-compatible API**를 제공합니다.
모듈은 OpenAI SDK를 그대로 사용할 수 있습니다.

| Method | Path | 설명 |
|--------|------|------|
| POST | `/v1/chat/completions` | 대화형 추론 |
| POST | `/v1/embeddings` | 벡터 임베딩 |
| POST | `/v1/images/generations` | 이미지 생성 |
| POST | `/v1/audio/speech` | TTS |
| POST | `/v1/audio/transcriptions` | STT |
| GET | `/v1/models` | 사용 가능한 모델 목록 (허용된 것만) |

**모듈에서의 사용 예시:**

```python
# Python (OpenAI SDK)
import openai

client = openai.OpenAI(
    base_url=os.environ["AI_ENDPOINT"],   # http://polyon-ai:8080/v1
    api_key=os.environ["AI_API_KEY"],     # mod_drive_xxxx
)

# Chat
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "이 문서를 요약해줘"}]
)

# Embedding
embedding = client.embeddings.create(
    model="text-embedding-3-small",
    input="검색할 문서 내용"
)
```

```go
// Go (직접 HTTP)
req, _ := http.NewRequest("POST", os.Getenv("AI_ENDPOINT")+"/chat/completions",
    bytes.NewReader(body))
req.Header.Set("Authorization", "Bearer "+os.Getenv("AI_API_KEY"))
req.Header.Set("Content-Type", "application/json")
```

```javascript
// JavaScript (fetch)
const resp = await fetch(`${window.__POLYON__.env.AI_ENDPOINT}/chat/completions`, {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${window.__POLYON__.env.AI_API_KEY}`,
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        model: 'gpt-4o-mini',
        messages: [{ role: 'user', content: 'Hello' }]
    })
});
```

### 9.5 Admin API (Console 관리용)

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/admin/ai/providers` | 등록된 provider 목록 |
| POST | `/api/v1/admin/ai/providers` | provider 추가 |
| PUT | `/api/v1/admin/ai/providers/{id}` | provider 수정 |
| DELETE | `/api/v1/admin/ai/providers/{id}` | provider 삭제 |
| GET | `/api/v1/admin/ai/policy` | 전역/모듈별 정책 조회 |
| PUT | `/api/v1/admin/ai/policy` | 정책 수정 |
| POST | `/api/v1/admin/ai/kill` | 긴급 차단 |
| GET | `/api/v1/admin/ai/usage` | 사용량 대시보드 데이터 |
| GET | `/api/v1/admin/ai/usage/{moduleId}` | 모듈별 사용량 |
| GET | `/api/v1/admin/ai/audit` | 감사 로그 (페이지네이션) |

### 9.6 Provider 구성

```yaml
# AI Gateway 설정 (Operator가 부트스트랩 시 생성)
providers:
  - id: openai
    name: "OpenAI"
    type: openai
    endpoint: "https://api.openai.com/v1"
    apiKey: "sk-..."           # K8s Secret에서 주입
    enabled: true
    models:
      - id: gpt-4o
        capabilities: [chat, vision]
        maxTokens: 128000
      - id: gpt-4o-mini
        capabilities: [chat]
        maxTokens: 128000
      - id: text-embedding-3-small
        capabilities: [embedding]
        dimensions: 1536
    priority: 2                # fallback

  - id: anthropic
    name: "Anthropic"
    type: anthropic
    endpoint: "https://api.anthropic.com/v1"
    apiKey: "sk-ant-..."
    enabled: true
    models:
      - id: claude-sonnet-4-20250514
        capabilities: [chat, vision]
        maxTokens: 200000
    priority: 3

  - id: local-ollama
    name: "Local Ollama"
    type: ollama
    endpoint: "http://ollama.polyon.svc.cluster.local:11434"
    apiKey: ""                 # 불필요
    enabled: true
    models:
      - id: local/llama3.1
        capabilities: [chat]
        maxTokens: 8192
    priority: 1                # 로컬 우선
```

### 9.7 Claim → Provision (LiteLLM Admin API 활용)

**모듈 요청:**
```yaml
claims:
  - type: ai
    config:
      capabilities:
        - chat
        - embedding
      rateLimit: 1000/day       # 선택
      costCap: "$30/month"      # 선택
```

**Operator 실행 (LiteLLM `/key/generate` 호출):**
```go
// LiteLLM Admin API로 Virtual Key 발급
resp, err := http.Post("http://polyon-ai:4000/key/generate", "application/json",
    json.Marshal(map[string]interface{}{
        "models":      getAllowedModels(claim.Config["capabilities"]),
        "max_budget":  30.0,     // $30/month
        "tpm_limit":   100000,   // tokens per minute
        "metadata": map[string]string{
            "module_id": claim.ModuleID,
            "managed_by": "polyon-operator",
        },
    }))
// resp.key = "sk-polyon-drive-xxxx"
```

**주입 Credentials:**
| 키 | 예시 |
|----|------|
| `endpoint` | `http://polyon-ai:4000/v1` |
| `apiKey` | `sk-polyon-drive-xxxx` (LiteLLM Virtual Key) |
| `models` | `gpt-4o-mini,text-embedding-3-small` |

### 9.8 Deprovision

```go
// LiteLLM Virtual Key 삭제
POST http://polyon-ai:4000/key/delete
{ "keys": ["sk-polyon-drive-xxxx"] }
```

### 9.9 부트스트랩 의존성

```
AI Gateway 자체의 Foundation Claim (정적 부트스트랩):
  - database: polyon_ai (감사 로그, 정책, 토큰 저장)
  - cache: Redis DB 2번 (Rate Limit 카운터, 토큰 캐시)
```

### 9.10 구현 계획

| Phase | 내용 |
|-------|------|
| **v0.1** | LiteLLM Proxy 배포 + PG/Redis 연결 + 기본 Provider 설정 |
| **v0.2** | PRC 연동 — Claim→Virtual Key 자동 발급, Operator 통합 |
| **v0.3** | Console 관리 페이지 — Provider 관리, 정책, 사용량 대시보드 |
| **v0.4** | Kill Switch + 콘텐츠 필터링 (LiteLLM guardrails) |
| **v1.0** | Ollama 로컬 모델 통합 + 모델 관리 + 이상 감지 |

---

## 10. Foundation 간 의존성 매트릭스

어떤 Foundation이 어떤 Foundation을 사용하는지 전체 맵입니다.

| Foundation ↓ 사용 → | PG | Redis | OpenSearch | RustFS | DC | Mail | Gitea | AI |
|---|---|---|---|---|---|---|---|---|
| **① PostgreSQL** | — | | | | | | | |
| **② Redis** | | — | | | | | | |
| **③ OpenSearch** | | | — | | | | | |
| **④ RustFS** | | | | — | | | | |
| **⑤ Samba AD DC** | | | | | — | | | |
| **⑥ Stalwart Mail** | ✅ | | ✅ | ✅ | ✅ | — | | |
| **⑦ Gitea** | ✅ | | | ✅ | ✅ | | — | |
| **⑧ AI Gateway** | ✅ | ✅ | | | | | | — |
| **Keycloak** | ✅ | | | | ✅ | | | |
| **Core** | ✅ | ✅ | | | ✅ | | | |

**읽는 법:** 행이 사용자, 열이 제공자. ✅ = 의존.
예: Stalwart Mail은 PG, OpenSearch, RustFS, DC에 의존.

---

## 11. 모듈이 사용 가능한 Claim 조합 예시

### Drive (파일 스토리지)
```yaml
claims: [database, objectStorage, directory, search, cache]
```

### Chat (메시징)
```yaml
claims: [database, objectStorage, directory, ai]
```

### Wiki (지식 관리)
```yaml
claims: [database, objectStorage, directory, search, ai]
```

### Workstream (업무 관리 — 향후)
```yaml
claims: [database, cache, directory, search, ai, smtp, git]
# Workstream은 거의 모든 Foundation을 사용하는 상위 레이어
```

### CI/CD (빌드 자동화 — 향후)
```yaml
claims: [database, git, ai, smtp, cache]
```

---

## 12. 용어 정리

| 용어 | 설명 |
|------|------|
| **Foundation** | 모든 모듈이 의존하는 PP 공유 인프라 (삭제 불가) |
| **Infrastructure Layer** | 상태 유지 Foundation (PG, Redis, OpenSearch, RustFS, DC, Mail, Gitea) |
| **Capability Layer** | 무상태 API Foundation (AI Gateway) |
| **Platform Layer** | 모듈이 직접 Claim하지 않는 플랫폼 구성요소 (Traefik, KC, Core, Console, Portal) |
| **Claim Type** | Foundation이 모듈에 제공하는 자원 유형 식별자 |
| **Provider** | Claim Type에 대응하는 프로비저닝 구현체 |
| **PRC** | Platform Resource Claim — 모듈의 선언적 자원 요청 메커니즘 |
| **부트스트랩** | Foundation 자체의 정적 설치 과정 (Operator 내장 순서) |
| **Kill Switch** | AI Gateway 긴급 차단 기능 |
