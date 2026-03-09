# PolyON Platform Resource Claim (PRC) Specification

> 작성: Jupiter (AI 팀장) | 2026-03-09
> 승인: CMARS (대표) | 2026-03-09
> 대상: Core/Operator 개발자, 모듈 개발자, AI 코딩 에이전트

---

## 1. 개요

### 1.1 동기

PolyON Platform(PP)에 설치되는 모듈은 플랫폼의 공유 인프라 자원을 필요로 합니다.
데이터베이스, 오브젝트 스토리지, LDAP 바인딩, 검색 인덱스, AI 추론 등—이 자원들을
모듈이 직접 관리하면 자격증명 분산, 수동 프로비저닝, 정리 누락 등의 문제가 발생합니다.

**Platform Resource Claim(PRC)**은 Kubernetes의 PVC(PersistentVolumeClaim)에서
영감을 받은 선언적 자원 요청 메커니즘입니다.

```
K8s:   Pod → PVC → StorageClass → PV (자동 프로비저닝)
PP:    Module → Claim → Provider → Resource (자동 프로비저닝)
```

### 1.2 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **선언적** | 모듈은 "무엇이 필요한지"만 선언, "어떻게 만드는지"는 모름 |
| **자동화** | Operator가 프로비저닝, 자격증명 생성, 주입을 전부 자동 처리 |
| **격리** | 모듈별 DB, 버킷, 인덱스, 토큰 — 다른 모듈 자원에 접근 불가 |
| **가역적** | 모듈 삭제 시 프로비저닝된 자원을 역순 정리 (Saga 보상) |
| **이식성** | K8s와 Docker Compose 양쪽에서 동일한 Claims 동작 (Dual Driver) |

---

## 2. Foundation 아키텍처

### 2.1 Foundation 구성

PP의 Foundation은 3개 계층으로 분류됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Platform Layer                          │
│              Traefik · Core · Console · Portal                  │
│          (라우팅, API, 관리 UI, 사용자 UI — Claim 대상 아님)     │
├─────────────────────────────────────────────────────────────────┤
│                       Capability Layer                          │
│                        AI Gateway                               │
│              (무상태, API 기반, 추론 서비스 관문)                 │
├─────────────────────────────────────────────────────────────────┤
│                      Infrastructure Layer                       │
│  PostgreSQL · Redis · OpenSearch · RustFS · DC · Mail · Gitea   │
│            (상태 유지, 데이터 저장, 항상 실행)                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Foundation Provider 8종

| # | 계층 | Service | Claim Type | 프로비저닝 내용 |
|---|------|---------|-----------|---------------|
| 1 | Infrastructure | PostgreSQL | `database` | CREATE DATABASE + USER + GRANT + Extensions |
| 2 | Infrastructure | Redis | `cache` | DB 번호 할당 (SELECT N으로 격리) |
| 3 | Infrastructure | OpenSearch | `search` | 인덱스 생성 + 매핑 템플릿 적용 |
| 4 | Infrastructure | RustFS | `objectStorage` | 버킷 생성 + 전용 접근키 발급 |
| 5 | Infrastructure | Samba AD DC | `directory` | 서비스 계정 OU + 바인드 DN 발급 |
| 6 | Infrastructure | Stalwart Mail | `smtp` | 서비스 발송 계정 + 도메인 설정 |
| 7 | Infrastructure | Gitea | `git` | Organization + Repository + API 토큰 발급 |
| 8 | Capability | AI Gateway | `ai` | 모듈 전용 API 토큰 + 모델 허용 목록 + Rate Limit |

### 2.3 Foundation 자체의 자원 요청

Foundation 모듈도 다른 Foundation 자원이 필요합니다 (예: Keycloak → PG, DC).

| | Foundation | Business Module |
|---|---|---|
| **시점** | 플랫폼 부트스트랩 (Setup Wizard) | 플랫폼 가동 후 (동적 설치) |
| **의존성 해결** | 정적 — Operator 코드에 내장된 설치 순서 | 동적 — claims[] 파싱 + Topological Sort |
| **Claim 선언** | Operator 내부 Bootstrap Manifest | module.yaml `spec.claims[]` |
| **자격증명 주입** | Operator가 직접 Secret/ConfigMap 생성 | PRC Saga가 자동 생성 |
| **삭제 가능** | ❌ (Foundation은 삭제 불가) | ✅ (Deprovision + Saga 보상) |

Foundation 부트스트랩 순서 (Topological Sort 결과, 정적):

```
Layer 0 (독립)     : ① PG  ② Traefik  ③ Redis  ④ OpenSearch  ⑤ RustFS
Layer 1 (L0 의존)  : ⑥ Samba AD DC
Layer 2 (L0+L1)    : ⑦ Keycloak(→PG,DC)  ⑧ Stalwart(→PG,DC,OpenSearch,RustFS)
Layer 3 (L0~L2)    : ⑨ Gitea(→PG,DC,RustFS)  ⑩ AI Gateway(→PG,Redis)
Layer 4 (전부)     : ⑪ Core  ⑫ Console  ⑬ Portal
```

> **핵심:** Foundation과 Module은 **같은 Claim Type**을 사용하지만,
> Foundation은 설치 순서가 코드에 내장되어 있고, Module은 런타임에 동적으로 해결됩니다.

---

## 3. Claims 스키마

### 3.1 module.yaml 내 Claims 선언

```yaml
apiVersion: polyon/v1
kind: Module
metadata:
  id: drive
  name: "PolyON Drive"
  version: "0.1.0"
  category: services

spec:
  image: jupitertriangles/polyon-drive:v0.1.0
  port: 8080
  
  # ═══════════════════════════════════════════
  # Platform Resource Claims
  # ═══════════════════════════════════════════
  claims:
    - type: database
      config:
        name: drive                    # → DB명: polyon_drive
        extensions: []                 # PG 확장 (pgvector, postgis 등)
    
    - type: objectStorage
      config:
        bucket: drive                  # → 버킷명: drive
        quota: 50Gi                    # 선택: 용량 제한
    
    - type: directory
      config:
        access: read                   # read | readWrite
        ou: Services                   # 서비스 계정 OU
    
    - type: search
      config:
        index: drive-files             # → 인덱스명: drive-files
        shards: 1                      # 선택: 샤드 수
        replicas: 0                    # 선택: 레플리카 수
    
    - type: cache
      config:
        db: auto                       # auto = Operator가 미사용 번호 할당
        maxMemory: 64mb                # 선택: 메모리 제한
    
    - type: smtp
      config:
        domain: drive                  # 발송 도메인 (drive@{base_domain})
    
    - type: git
      config:
        org: polyon-modules            # Gitea organization
        repo: drive-data               # 레포지토리명
    
    - type: ai
      config:
        capabilities:                  # 필요한 AI 기능
          - chat                       # 대화형 추론
          - embedding                  # 벡터 임베딩
        rateLimit: 1000/day            # 선택: 일일 요청 제한
  
  # ═══════════════════════════════════════════
  # Claim 결과 → 환경변수 매핑
  # ═══════════════════════════════════════════
  env:
    # database claim 결과
    DATABASE_URL: "{{ claims.database.url }}"
    
    # objectStorage claim 결과
    S3_ENDPOINT: "{{ claims.objectStorage.endpoint }}"
    S3_BUCKET: "{{ claims.objectStorage.bucket }}"
    S3_ACCESS_KEY: "{{ claims.objectStorage.accessKey }}"
    S3_SECRET_KEY: "{{ claims.objectStorage.secretKey }}"
    
    # directory claim 결과
    LDAP_URL: "{{ claims.directory.url }}"
    LDAP_BIND_DN: "{{ claims.directory.bindDN }}"
    LDAP_BIND_PW: "{{ claims.directory.bindPassword }}"
    LDAP_BASE_DN: "{{ claims.directory.baseDN }}"
    
    # search claim 결과
    SEARCH_URL: "{{ claims.search.url }}"
    SEARCH_INDEX: "{{ claims.search.index }}"
    
    # cache claim 결과
    REDIS_URL: "{{ claims.cache.url }}"
    
    # smtp claim 결과
    SMTP_HOST: "{{ claims.smtp.host }}"
    SMTP_PORT: "{{ claims.smtp.port }}"
    SMTP_USER: "{{ claims.smtp.user }}"
    SMTP_PASS: "{{ claims.smtp.password }}"
    
    # git claim 결과
    GIT_URL: "{{ claims.git.url }}"
    GIT_TOKEN: "{{ claims.git.token }}"
    
    # ai claim 결과
    AI_ENDPOINT: "{{ claims.ai.endpoint }}"
    AI_API_KEY: "{{ claims.ai.apiKey }}"
    AI_MODELS: "{{ claims.ai.models }}"
    
    # 정적 환경변수 (Claim 무관)
    DRIVE_LISTEN: "0.0.0.0:8080"
    LOG_LEVEL: "info"
```

### 3.2 Claim 타입별 config 필드

#### `database`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | string | ✅ | DB 이름 (접두어 `polyon_` 자동 부여) |
| `extensions` | string[] | - | PG 확장 목록 (pgvector, postgis 등) |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `url` | `postgres://mod_drive:xxxx@polyon-db:5432/polyon_drive` |

#### `objectStorage`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `bucket` | string | ✅ | 버킷 이름 |
| `quota` | string | - | 용량 제한 (10Gi, 100Gi 등) |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `endpoint` | `http://polyon-rustfs:9000` |
| `bucket` | `drive` |
| `accessKey` | `mod-drive-access` |
| `secretKey` | `xxxx` |

#### `directory`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `access` | string | ✅ | `read` 또는 `readWrite` |
| `ou` | string | - | 서비스 계정 OU (기본: Services) |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `url` | `ldap://polyon-dc:389` |
| `bindDN` | `CN=svc-drive,OU=Services,DC=company,DC=com` |
| `bindPassword` | `xxxx` |
| `baseDN` | `DC=company,DC=com` |

#### `search`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `index` | string | ✅ | 인덱스 이름 |
| `shards` | int | - | 샤드 수 (기본: 1) |
| `replicas` | int | - | 레플리카 수 (기본: 0) |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `url` | `http://polyon-search:9200` |
| `index` | `drive-files` |

#### `cache`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `db` | string | ✅ | `auto` (자동 할당) 또는 숫자 (0~15) |
| `maxMemory` | string | - | 메모리 제한 (64mb, 128mb 등) |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `url` | `redis://polyon-redis:6379/3` |

#### `smtp`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `domain` | string | ✅ | 발송 도메인 prefix (→ `{domain}@{base_domain}`) |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `host` | `polyon-mail` |
| `port` | `587` |
| `user` | `drive@company.com` |
| `password` | `xxxx` |

#### `git`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `org` | string | ✅ | Gitea organization 이름 |
| `repo` | string | ✅ | 레포지토리 이름 |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `url` | `http://polyon-gitea:3000/polyon-modules/drive-data.git` |
| `token` | `gitea_mod_drive_xxxx` |

#### `ai`
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `capabilities` | string[] | ✅ | 필요 기능: `chat`, `embedding`, `vision`, `tts`, `stt` |
| `rateLimit` | string | - | 요청 제한 (`1000/day`, `100/hour` 등) |
| `maxTokens` | int | - | 최대 컨텍스트 윈도우 |

**프로비저닝 결과:**
| 키 | 예시 |
|-----|------|
| `endpoint` | `http://polyon-ai-gateway:8080/v1` |
| `apiKey` | `mod_drive_xxxx` |
| `models` | `gpt-4o,text-embedding-3-small` |

---

## 4. 알고리즘 및 패턴

PRC 엔진은 3개의 검증된 패턴을 결합합니다.

### 4.1 Open Service Broker (OSB) — 생명주기 모델

CloudFoundry와 Kubernetes에서 서비스 프로비저닝 표준으로 사용되는 패턴입니다.

```
┌──────────┐     ┌───────────┐     ┌──────┐     ┌─────┐     ┌────────────┐
│ Catalog  │────►│ Provision │────►│ Bind │────►│ Use │────►│ Deprovision│
│ (목록)   │     │ (생성)    │     │(연결)│     │(사용)│     │ (정리)     │
└──────────┘     └───────────┘     └──────┘     └─────┘     └────────────┘
```

| OSB 단계 | PP 대응 | 설명 |
|----------|---------|------|
| **Catalog** | Foundation 8종 목록 | 사용 가능한 Provider 열거 |
| **Provision** | CREATE DB, 버킷 생성 등 | 자원 인스턴스 생성 |
| **Bind** | K8s Secret / .env 파일 생성 | 자격증명을 앱에 연결 |
| **Use** | 모듈 컨테이너 실행 | 환경변수로 자원 접근 |
| **Unbind** | Secret / .env 삭제 | 연결 해제 |
| **Deprovision** | DROP DB, 버킷 삭제 등 | 자원 인스턴스 정리 |

### 4.2 Topological Sort (Kahn's Algorithm) — 의존성 순서 해결

Claims 간 의존성이 있을 경우 (예: OIDC 클라이언트 → KC가 먼저 필요), 설치 순서를
결정하는 알고리즘입니다.

```
입력: Claim들의 의존성 DAG (Directed Acyclic Graph)
출력: 프로비저닝 실행 순서

알고리즘: Kahn's Algorithm (BFS 기반 위상 정렬)
시간 복잡도: O(V + E) — V: claim 수, E: 의존성 수
```

**의존성 그래프 예시:**

```
database ──────────────┐
                       ├──► (독립, 병렬 실행 가능)
objectStorage ─────────┘
                       
directory ─────────────┐
                       ├──► smtp (directory 선행 필수)
                       │
                       └──► git (directory 선행 필수)

cache ─────────────────── (독립)

ai ────────────────────── (독립)
```

**Kahn's Algorithm 의사코드:**

```
function topologicalSort(claims, providers):
    // 1. 진입 차수(in-degree) 계산
    inDegree = {}
    for each claim in claims:
        inDegree[claim.type] = 0
    for each claim in claims:
        for each dep in providers[claim.type].dependsOn():
            inDegree[claim.type] += 1
    
    // 2. 진입 차수 0인 노드를 큐에 삽입
    queue = [claim for claim in claims if inDegree[claim.type] == 0]
    
    // 3. BFS 순회
    sorted = []
    while queue is not empty:
        current = queue.dequeue()
        sorted.append(current)
        for each dependent that depends on current:
            inDegree[dependent.type] -= 1
            if inDegree[dependent.type] == 0:
                queue.enqueue(dependent)
    
    // 4. 순환 감지
    if len(sorted) != len(claims):
        error("순환 의존성 감지")
    
    return sorted
```

### 4.3 Saga Pattern — 분산 트랜잭션 보상

여러 Provider에 걸친 프로비저닝은 분산 트랜잭션입니다. 중간 단계 실패 시
이전 단계를 역순으로 롤백해야 합니다.

**Orchestration Saga** (중앙 코디네이터 방식)를 사용합니다:

```
정상 흐름:
  ① database.Provision()     ✅  → 보상: database.Deprovision()
  ② objectStorage.Provision() ✅  → 보상: objectStorage.Deprovision()
  ③ directory.Provision()     ✅  → 보상: directory.Deprovision()
  ④ search.Provision()        ✅  → 보상: search.Deprovision()
  ⑤ (전부 성공) → Bind → Deploy

실패 흐름:
  ① database.Provision()      ✅
  ② objectStorage.Provision()  ✅
  ③ directory.Provision()      ❌ 실패!
     ↓
  보상 실행 (역순):
  ② objectStorage.Deprovision() ← 버킷 삭제
  ① database.Deprovision()      ← DB 삭제
  → 설치 실패 보고
```

**Saga 상태 머신:**

```
         ┌─────────┐
         │ PENDING  │
         └────┬─────┘
              │ 설치 시작
              ▼
         ┌──────────────┐
    ┌───►│ PROVISIONING │◄──── (재시도 가능)
    │    └──────┬───────┘
    │           │ 전부 성공
    │           ▼
    │    ┌──────────────┐
    │    │   BINDING     │
    │    └──────┬───────┘
    │           │ Secret/env 생성 완료
    │           ▼
    │    ┌──────────────┐
    │    │  DEPLOYING    │
    │    └──────┬───────┘
    │           │ Pod/Container Ready
    │           ▼
    │    ┌──────────────┐
    │    │   ACTIVE      │
    │    └──────┬───────┘
    │           │ 삭제 요청
    │           ▼
    │    ┌──────────────┐
    │    │ DEPROVISIONING│
    │    └──────┬───────┘
    │           │ 전부 정리
    │           ▼
    │    ┌──────────────┐
    │    │  REMOVED      │
    │    └──────────────┘
    │
    │    ┌──────────────┐
    └────│ COMPENSATING │◄──── (프로비저닝 실패 시)
         │  (보상 중)    │
         └──────┬───────┘
                │ 역순 롤백 완료
                ▼
         ┌──────────────┐
         │   FAILED      │
         └──────────────┘
```

---

## 5. PRC 실행 프로세스

### 5.1 모듈 설치 전체 흐름

```
Module Install Request
        │
        ▼
┌─────────────────────────────────────┐
│  Step 1. Parse Claims               │
│  module.yaml → spec.claims[] 추출   │
│  → Claim 목록 + env 매핑 파싱       │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 2. Validate                   │
│  - 미지원 Claim Type 검출           │
│  - 필수 config 필드 누락 검사       │
│  - Foundation 서비스 가용 여부 확인  │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 3. Dependency Sort            │
│  Kahn's Algorithm으로 실행 순서 결정 │
│  → 독립 Claim은 병렬 실행 가능       │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 4. Provision (Saga)           │
│  정렬된 순서대로 각 Provider 호출    │
│  - 실패 시 보상 트랜잭션 (역순)      │
│  - 결과: Credentials map            │
└─────────────┬───────────────────────┘
              │ 전부 성공
              ▼
┌─────────────────────────────────────┐
│  Step 5. Bind                       │
│  Credentials → 환경변수 템플릿 치환  │
│  → K8s Secret 또는 .env 파일 생성    │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 6. Deploy                     │
│  Deployment/Container 생성          │
│  → envFrom: secretRef / env_file    │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 7. Health Check               │
│  Pod Ready / Container healthy      │
│  → 상태: ACTIVE                     │
│  → 설치 완료 이벤트 발행            │
└─────────────────────────────────────┘
```

### 5.2 모듈 삭제 흐름

```
Module Uninstall Request
        │
        ▼
┌─────────────────────────────────────┐
│  Step 1. 의존성 확인                 │
│  다른 모듈이 이 모듈에 의존하는지    │
│  → 있으면 삭제 차단                  │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 2. 워크로드 제거               │
│  Deployment/Container 삭제          │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 3. Unbind                     │
│  Secret / .env 파일 삭제            │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 4. Deprovision                │
│  dataPolicy에 따라:                 │
│  - "delete": DROP DB, 버킷 삭제 등  │
│  - "keep": 자원 유지 (재설치 대비)   │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Step 5. 상태 갱신                   │
│  → 상태: REMOVED                    │
│  → 삭제 완료 이벤트 발행            │
└─────────────────────────────────────┘
```

---

## 6. Provider 인터페이스

### 6.1 Go 인터페이스 정의

```go
package prc

import "context"

// Credentials는 프로비저닝 결과로 생성되는 자격증명 맵
type Credentials map[string]string

// ResourceStatus는 자원의 현재 상태
type ResourceStatus string

const (
    StatusNotFound    ResourceStatus = "not_found"
    StatusProvisioned ResourceStatus = "provisioned"
    StatusDegraded    ResourceStatus = "degraded"
    StatusError       ResourceStatus = "error"
)

// Claim은 module.yaml에서 파싱된 자원 요청
type Claim struct {
    Type     string            `yaml:"type"`
    Config   map[string]string `yaml:"config"`
    ModuleID string            // Operator가 주입
}

// ResourceProvider는 모든 Foundation Provider가 구현하는 인터페이스
type ResourceProvider interface {
    // Type은 이 Provider의 Claim Type 식별자를 반환
    Type() string // "database", "objectStorage", "directory", ...

    // DependsOn은 이 Provider가 의존하는 다른 Provider 타입 목록을 반환
    // Topological Sort에 사용
    DependsOn() []string

    // Provision은 자원을 생성하고 자격증명을 반환
    Provision(ctx context.Context, claim Claim) (Credentials, error)

    // Deprovision은 자원을 삭제 (Saga 보상 트랜잭션)
    Deprovision(ctx context.Context, claim Claim) error

    // Status는 프로비저닝된 자원의 현재 상태를 조회
    // (Reconciliation Loop에서 사용, Phase 2)
    Status(ctx context.Context, claim Claim) (ResourceStatus, error)
}
```

### 6.2 Provider 구현 예시

#### DatabaseProvider

```go
type DatabaseProvider struct {
    adminPool *pgxpool.Pool // PG superuser 연결
}

func (p *DatabaseProvider) Type() string      { return "database" }
func (p *DatabaseProvider) DependsOn() []string { return nil } // 독립

func (p *DatabaseProvider) Provision(ctx context.Context, claim Claim) (Credentials, error) {
    dbName := "polyon_" + claim.Config["name"]
    user := "mod_" + claim.Config["name"]
    password := generateSecurePassword(24)

    // 1. CREATE USER (IF NOT EXISTS)
    _, err := p.adminPool.Exec(ctx,
        fmt.Sprintf("DO $$ BEGIN IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='%s') THEN CREATE ROLE %s LOGIN PASSWORD '%s'; END IF; END $$",
            user, user, password))
    if err != nil {
        return nil, fmt.Errorf("user 생성 실패: %w", err)
    }

    // 2. CREATE DATABASE
    _, err = p.adminPool.Exec(ctx,
        fmt.Sprintf("CREATE DATABASE %s OWNER %s", dbName, user))
    if err != nil {
        return nil, fmt.Errorf("database 생성 실패: %w", err)
    }

    // 3. Extensions (지정된 경우)
    if exts, ok := claim.Config["extensions"]; ok && exts != "" {
        connStr := fmt.Sprintf("postgres://%s:%s@polyon-db:5432/%s", user, password, dbName)
        modPool, _ := pgxpool.New(ctx, connStr)
        defer modPool.Close()
        for _, ext := range strings.Split(exts, ",") {
            modPool.Exec(ctx, fmt.Sprintf("CREATE EXTENSION IF NOT EXISTS %s", strings.TrimSpace(ext)))
        }
    }

    return Credentials{
        "url": fmt.Sprintf("postgres://%s:%s@polyon-db:5432/%s", user, password, dbName),
    }, nil
}

func (p *DatabaseProvider) Deprovision(ctx context.Context, claim Claim) error {
    dbName := "polyon_" + claim.Config["name"]
    user := "mod_" + claim.Config["name"]

    // 연결 강제 종료 → DROP DATABASE → DROP USER
    p.adminPool.Exec(ctx, fmt.Sprintf(
        "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='%s'", dbName))
    p.adminPool.Exec(ctx, fmt.Sprintf("DROP DATABASE IF EXISTS %s", dbName))
    p.adminPool.Exec(ctx, fmt.Sprintf("DROP ROLE IF EXISTS %s", user))
    return nil
}
```

#### ObjectStorageProvider

```go
type ObjectStorageProvider struct {
    s3Client  *s3.Client
    adminKey  string
    adminSecret string
}

func (p *ObjectStorageProvider) Type() string      { return "objectStorage" }
func (p *ObjectStorageProvider) DependsOn() []string { return nil }

func (p *ObjectStorageProvider) Provision(ctx context.Context, claim Claim) (Credentials, error) {
    bucket := claim.Config["bucket"]
    accessKey := fmt.Sprintf("mod-%s-access", claim.ModuleID)
    secretKey := generateSecurePassword(32)

    // 1. 버킷 생성
    _, err := p.s3Client.CreateBucket(ctx, &s3.CreateBucketInput{
        Bucket: &bucket,
    })
    if err != nil {
        return nil, fmt.Errorf("버킷 생성 실패: %w", err)
    }

    // 2. 접근키 발급 (RustFS Admin API)
    // ...

    return Credentials{
        "endpoint":  "http://polyon-rustfs:9000",
        "bucket":    bucket,
        "accessKey": accessKey,
        "secretKey": secretKey,
    }, nil
}
```

#### AIProvider

```go
type AIProvider struct {
    gatewayURL string
    adminToken string
}

func (p *AIProvider) Type() string        { return "ai" }
func (p *AIProvider) DependsOn() []string { return nil }

func (p *AIProvider) Provision(ctx context.Context, claim Claim) (Credentials, error) {
    moduleToken := fmt.Sprintf("mod_%s_%s", claim.ModuleID, generateSecurePassword(16))
    capabilities := claim.Config["capabilities"]
    rateLimit := claim.Config["rateLimit"]

    // 1. AI Gateway에 모듈 등록 (Admin API)
    // → 토큰 발급, 허용 모델 설정, Rate Limit 적용
    err := p.registerModule(ctx, claim.ModuleID, moduleToken, capabilities, rateLimit)
    if err != nil {
        return nil, fmt.Errorf("AI Gateway 등록 실패: %w", err)
    }

    // 2. 할당된 모델 목록 조회
    models, _ := p.getAllowedModels(ctx, capabilities)

    return Credentials{
        "endpoint": p.gatewayURL + "/v1",
        "apiKey":   moduleToken,
        "models":   strings.Join(models, ","),
    }, nil
}
```

### 6.3 Saga 실행기

```go
// SagaExecutor는 Claims를 순서대로 프로비저닝하고, 실패 시 역순 보상
type SagaExecutor struct {
    providers map[string]ResourceProvider
}

func (s *SagaExecutor) Execute(ctx context.Context, claims []Claim) (map[string]Credentials, error) {
    // 1. Topological Sort
    sorted, err := s.topologicalSort(claims)
    if err != nil {
        return nil, fmt.Errorf("의존성 해결 실패: %w", err)
    }

    // 2. 순서대로 프로비저닝
    results := make(map[string]Credentials)
    var completed []Claim // 보상용 스택

    for _, claim := range sorted {
        provider, ok := s.providers[claim.Type]
        if !ok {
            // 보상 실행
            s.compensate(ctx, completed)
            return nil, fmt.Errorf("미지원 Claim Type: %s", claim.Type)
        }

        creds, err := provider.Provision(ctx, claim)
        if err != nil {
            log.Error("프로비저닝 실패, 보상 시작",
                "claim", claim.Type, "module", claim.ModuleID, "error", err)
            // 역순 보상
            s.compensate(ctx, completed)
            return nil, fmt.Errorf("claim %s 프로비저닝 실패: %w", claim.Type, err)
        }

        results[claim.Type] = creds
        completed = append(completed, claim)
        log.Info("프로비저닝 성공", "claim", claim.Type, "module", claim.ModuleID)
    }

    return results, nil
}

// compensate는 완료된 프로비저닝을 역순으로 롤백
func (s *SagaExecutor) compensate(ctx context.Context, completed []Claim) {
    for i := len(completed) - 1; i >= 0; i-- {
        claim := completed[i]
        provider := s.providers[claim.Type]
        if err := provider.Deprovision(ctx, claim); err != nil {
            log.Error("보상 실패 (수동 정리 필요)",
                "claim", claim.Type, "module", claim.ModuleID, "error", err)
        } else {
            log.Info("보상 완료", "claim", claim.Type, "module", claim.ModuleID)
        }
    }
}

// topologicalSort는 Kahn's Algorithm으로 Claims 실행 순서를 결정
func (s *SagaExecutor) topologicalSort(claims []Claim) ([]Claim, error) {
    // 활성 타입 집합
    activeTypes := make(map[string]bool)
    for _, c := range claims {
        activeTypes[c.Type] = true
    }

    // 진입 차수 계산
    inDegree := make(map[string]int)
    adj := make(map[string][]string)
    for _, c := range claims {
        if _, ok := inDegree[c.Type]; !ok {
            inDegree[c.Type] = 0
        }
        provider := s.providers[c.Type]
        for _, dep := range provider.DependsOn() {
            if activeTypes[dep] {
                adj[dep] = append(adj[dep], c.Type)
                inDegree[c.Type]++
            }
        }
    }

    // BFS
    var queue []string
    for t, deg := range inDegree {
        if deg == 0 {
            queue = append(queue, t)
        }
    }

    var order []string
    for len(queue) > 0 {
        curr := queue[0]
        queue = queue[1:]
        order = append(order, curr)
        for _, next := range adj[curr] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    if len(order) != len(claims) {
        return nil, fmt.Errorf("순환 의존성")
    }

    // order 순서대로 claims 재배열
    typeToIndex := make(map[string]int)
    for i, t := range order {
        typeToIndex[t] = i
    }
    sorted := make([]Claim, len(claims))
    copy(sorted, claims)
    sort.Slice(sorted, func(i, j int) bool {
        return typeToIndex[sorted[i].Type] < typeToIndex[sorted[j].Type]
    })
    return sorted, nil
}
```

---

## 7. Dual Driver 아키텍처

PRC는 **런타임에 독립적**입니다. 자원 프로비저닝(Claim → Provision → Credentials)은
동일하며, 마지막 마일(자격증명 주입 + 컨테이너 생성)만 런타임에 따라 다릅니다.

### 7.1 DeployDriver 인터페이스

```go
// DeployDriver는 자격증명 주입과 컨테이너 관리를 추상화
type DeployDriver interface {
    // InjectCredentials는 프로비저닝 결과를 모듈에 전달할 수 있는 형태로 저장
    // K8s: Secret 생성, Docker: .env 파일 생성
    InjectCredentials(moduleID string, envMap map[string]string) error

    // Deploy는 모듈 컨테이너를 생성/시작
    Deploy(moduleID string, spec DeploySpec) error

    // Remove는 모듈 컨테이너를 정지/삭제
    Remove(moduleID string) error

    // IsReady는 모듈이 정상 실행 중인지 확인
    IsReady(moduleID string) (bool, error)

    // RemoveCredentials는 주입된 자격증명을 삭제
    RemoveCredentials(moduleID string) error
}

type DeploySpec struct {
    Image     string            // 컨테이너 이미지
    Port      int               // 서비스 포트
    Replicas  int               // 레플리카 수 (Docker에서는 무시)
    Resources ResourceLimits    // CPU/Memory 제한
}
```

### 7.2 Kubernetes Driver

```go
type K8sDriver struct {
    clientset kubernetes.Interface
    namespace string // "polyon"
}

func (d *K8sDriver) InjectCredentials(moduleID string, envMap map[string]string) error {
    secretName := fmt.Sprintf("polyon-module-%s", moduleID)
    secret := &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{
            Name:      secretName,
            Namespace: d.namespace,
            Labels: map[string]string{
                "app.kubernetes.io/managed-by": "polyon-operator",
                "polyon.io/module":             moduleID,
            },
        },
        StringData: envMap,
    }
    _, err := d.clientset.CoreV1().Secrets(d.namespace).Create(ctx, secret, metav1.CreateOptions{})
    return err
}

func (d *K8sDriver) Deploy(moduleID string, spec DeploySpec) error {
    // Deployment + Service + Ingress 생성
    deployment := &appsv1.Deployment{
        // ...
        Spec: appsv1.DeploymentSpec{
            Template: corev1.PodTemplateSpec{
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  moduleID,
                        Image: spec.Image,
                        EnvFrom: []corev1.EnvFromSource{{
                            SecretRef: &corev1.SecretEnvSource{
                                LocalObjectReference: corev1.LocalObjectReference{
                                    Name: fmt.Sprintf("polyon-module-%s", moduleID),
                                },
                            },
                        }},
                    }},
                },
            },
        },
    }
    _, err := d.clientset.AppsV1().Deployments(d.namespace).Create(ctx, deployment, metav1.CreateOptions{})
    return err
}
```

### 7.3 Docker Compose Driver

```go
type ComposeDriver struct {
    runtimeDir string // 예: ~/helios-test/runtime
    projectName string // 예: "helios"
}

func (d *ComposeDriver) InjectCredentials(moduleID string, envMap map[string]string) error {
    // .env 파일 생성
    envPath := filepath.Join(d.runtimeDir, "modules", moduleID+".env")
    os.MkdirAll(filepath.Dir(envPath), 0700)

    var lines []string
    for k, v := range envMap {
        lines = append(lines, fmt.Sprintf("%s=%s", k, v))
    }
    return os.WriteFile(envPath, []byte(strings.Join(lines, "\n")), 0600)
}

func (d *ComposeDriver) Deploy(moduleID string, spec DeploySpec) error {
    // compose override 파일 생성
    overridePath := filepath.Join(d.runtimeDir, "modules", moduleID+".compose.yaml")
    
    composeContent := fmt.Sprintf(`services:
  %s-%s:
    image: %s
    env_file:
      - ./modules/%s.env
    networks:
      - %s-net
    restart: unless-stopped
`, d.projectName, moduleID, spec.Image, moduleID, d.projectName)
    
    os.WriteFile(overridePath, []byte(composeContent), 0644)

    // docker compose up
    cmd := exec.Command("docker", "compose",
        "-p", d.projectName,
        "-f", "docker-compose.yaml",
        "-f", overridePath,
        "up", "-d", fmt.Sprintf("%s-%s", d.projectName, moduleID))
    cmd.Dir = d.runtimeDir
    return cmd.Run()
}

func (d *ComposeDriver) Remove(moduleID string) error {
    serviceName := fmt.Sprintf("%s-%s", d.projectName, moduleID)
    // 개별 stop + rm (compose down 금지 원칙)
    exec.Command("docker", "stop", serviceName).Run()
    exec.Command("docker", "rm", "-f", serviceName).Run()
    return nil
}

func (d *ComposeDriver) RemoveCredentials(moduleID string) error {
    envPath := filepath.Join(d.runtimeDir, "modules", moduleID+".env")
    overridePath := filepath.Join(d.runtimeDir, "modules", moduleID+".compose.yaml")
    os.Remove(envPath)
    os.Remove(overridePath)
    return nil
}
```

### 7.4 Driver 선택

```go
func NewDeployDriver(runtime string) DeployDriver {
    switch runtime {
    case "kubernetes":
        return &K8sDriver{
            clientset: kubernetes.NewForConfigOrDie(rest.InClusterConfig()),
            namespace: "polyon",
        }
    case "docker":
        return &ComposeDriver{
            runtimeDir:  os.Getenv("RUNTIME_DIR"),
            projectName: os.Getenv("COMPOSE_PROJECT_NAME"),
        }
    default:
        panic("unsupported runtime: " + runtime)
    }
}
```

### 7.5 비교 요약

| 단계 | K8s Driver | Docker Compose Driver |
|------|-----------|----------------------|
| 자격증명 저장 | K8s Secret | `.env` 파일 (runtime/modules/) |
| 환경변수 주입 | `envFrom: secretRef` | `env_file: ./modules/{id}.env` |
| 컨테이너 생성 | Deployment + Service + Ingress | compose override + `up -d` |
| 헬스체크 | Pod readinessProbe | `docker inspect` healthcheck |
| 컨테이너 삭제 | `kubectl delete deployment` | `docker stop + rm` |
| 자격증명 삭제 | `kubectl delete secret` | `.env` 파일 삭제 |

---

## 8. Claim 상태 관리

### 8.1 DB 스키마

프로비저닝 상태를 DB에 영속화하여, Operator 재시작 후에도 상태를 복원합니다.

```sql
-- 모듈의 Claim 프로비저닝 상태
CREATE TABLE module_claims (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id     VARCHAR(64) NOT NULL,        -- 모듈 식별자
    claim_type    VARCHAR(32) NOT NULL,        -- database, objectStorage, ...
    claim_config  JSONB NOT NULL DEFAULT '{}', -- module.yaml의 config
    credentials   JSONB,                       -- 프로비저닝 결과 (암호화 권장)
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
                  -- pending | provisioning | provisioned | failed | deprovisioning | removed
    error_message TEXT,                        -- 실패 시 에러 메시지
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(module_id, claim_type)
);

-- Saga 실행 이력
CREATE TABLE claim_saga_log (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id  VARCHAR(64) NOT NULL,
    action     VARCHAR(20) NOT NULL,  -- provision | deprovision | compensate
    claim_type VARCHAR(32) NOT NULL,
    status     VARCHAR(20) NOT NULL,  -- success | failed
    duration_ms INT,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Redis DB 번호 할당 추적 (cache claim용)
CREATE TABLE redis_db_allocations (
    db_number   INT PRIMARY KEY,      -- 0~15
    module_id   VARCHAR(64),          -- NULL = 미할당
    allocated_at TIMESTAMPTZ
);

-- 초기화: 0~15 번호 등록, 0은 시스템 예약
INSERT INTO redis_db_allocations (db_number)
    SELECT generate_series(0, 15);
UPDATE redis_db_allocations SET module_id = 'system' WHERE db_number = 0;
```

### 8.2 Reconciliation Loop (Phase 2)

Phase 2에서 추가할 기능으로, 주기적으로 선언 상태와 실제 상태를 비교하여 수렴시킵니다.

```go
// ReconcileLoop는 5분마다 모든 활성 모듈의 Claim 상태를 검증
func (o *Operator) ReconcileLoop(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            modules := o.listActiveModules(ctx)
            for _, mod := range modules {
                claims := o.getModuleClaims(ctx, mod.ID)
                for _, claim := range claims {
                    provider := o.providers[claim.Type]
                    status, err := provider.Status(ctx, claim)
                    if err != nil || status != StatusProvisioned {
                        log.Warn("Claim 상태 이상",
                            "module", mod.ID, "claim", claim.Type, "status", status)
                        // 자동 복구 시도 또는 알림
                    }
                }
            }
        }
    }
}
```

---

## 9. AI Gateway 상세 설계

### 9.1 아키텍처

```
┌──────────────────────────────────────────────────┐
│                 AI Gateway                        │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │           Token Validator                   │  │
│  │  mod_drive_xxxx → module=drive, limits=... │  │
│  └───────────────────┬────────────────────────┘  │
│                      │                           │
│  ┌───────────────────▼────────────────────────┐  │
│  │           Router / Load Balancer            │  │
│  │  capability + policy → provider 선택       │  │
│  └───┬───────────┬──────────┬────────────┬────┘  │
│      │           │          │            │        │
│  ┌───▼──┐  ┌────▼────┐  ┌──▼──┐  ┌─────▼────┐  │
│  │OpenAI│  │Anthropic│  │Ollama│  │Google AI │  │
│  └──────┘  └─────────┘  └─────┘  └──────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │         Usage Tracker (→ PG)               │  │
│  │  module별 토큰 사용량, 비용, 요청 수       │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

### 9.2 관리자 정책

Console에서 관리자가 설정하는 AI 정책:

```yaml
# AI Gateway 전역 설정 (Console → Core → AI Gateway)
ai_policy:
  providers:
    - name: openai
      apiKey: "sk-..."
      enabled: true
    - name: ollama
      endpoint: "http://ollama:11434"
      enabled: true
  
  defaults:
    chatModel: "gpt-4o"
    embeddingModel: "text-embedding-3-small"
    maxTokensPerRequest: 128000
    
  moduleLimits:
    daily: 10000          # 모듈당 일일 기본 한도
    monthly: 200000       # 모듈당 월간 기본 한도
    costCap: "$50/month"  # 모듈당 월간 비용 한도
```

### 9.3 모듈에서의 사용

모듈은 AI Gateway를 표준 OpenAI-compatible API로 호출합니다:

```python
# 모듈 코드 (예: Drive 문서 요약 기능)
import openai

client = openai.OpenAI(
    base_url=os.environ["AI_ENDPOINT"],  # http://polyon-ai-gateway:8080/v1
    api_key=os.environ["AI_API_KEY"],    # mod_drive_xxxx
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "이 문서를 요약해줘: ..."}]
)
```

---

## 10. 보안 고려사항

### 10.1 자격증명 보호

| 항목 | 방법 |
|------|------|
| DB 패스워드 | 24자 `a-zA-Z0-9-_` (특수문자 `&!^*+#=` 금지) |
| API 토큰 | 32자 랜덤 + prefix (`mod_{moduleId}_`) |
| Secret 암호화 | K8s: etcd encryption at rest / Docker: 파일 권한 600 |
| 네트워크 격리 | 모듈은 Foundation 서비스에만 접근, 다른 모듈 직접 통신 금지 |
| Claim credentials DB 저장 | `module_claims.credentials` JSONB — 암호화 컬럼 권장 |

### 10.2 모듈 격리

- 모듈별 DB 유저 → 자기 DB만 접근 가능 (PG GRANT)
- 모듈별 RustFS 접근키 → 자기 버킷만 접근 가능
- 모듈별 Redis DB 번호 → 다른 모듈 데이터에 접근 불가
- 모듈별 AI 토큰 → Rate Limit 독립 적용

---

## 11. 에러 처리 및 복구

### 11.1 프로비저닝 실패 시나리오

| 시나리오 | 처리 |
|----------|------|
| PG 연결 불가 | Saga 보상 → 이미 생성된 자원 롤백 → FAILED 상태 |
| 버킷 이름 중복 | 기존 버킷 재사용 (idempotent) 또는 에러 |
| LDAP 서비스 계정 생성 실패 | Saga 보상 → FAILED → 관리자 알림 |
| AI Gateway 미설치 | ai claim skip (optional로 표시 가능) |
| 순환 의존성 | 설치 차단 → 에러 메시지 반환 |

### 11.2 부분 실패 복구

```go
// 수동 재시도 API
// POST /api/v1/modules/{moduleId}/claims/retry
func (h *Handler) RetryClaims(w http.ResponseWriter, r *http.Request) {
    moduleID := chi.URLParam(r, "moduleId")
    
    // failed 상태인 claim만 재프로비저닝
    failedClaims := h.getFailedClaims(r.Context(), moduleID)
    results, err := h.saga.Execute(r.Context(), failedClaims)
    // ...
}
```

---

## 12. API 엔드포인트

### 12.1 Claim 관리 API

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/modules/{id}/claims` | 모듈의 Claim 목록 + 상태 |
| POST | `/api/v1/modules/{id}/claims/retry` | 실패한 Claim 재프로비저닝 |
| GET | `/api/v1/claims/providers` | 사용 가능한 Provider 목록 (Catalog) |
| GET | `/api/v1/claims/providers/{type}/status` | Provider 건강 상태 |

### 12.2 응답 예시

```json
// GET /api/v1/modules/drive/claims
{
  "moduleId": "drive",
  "status": "active",
  "claims": [
    {
      "type": "database",
      "config": { "name": "drive" },
      "status": "provisioned",
      "provisionedAt": "2026-03-09T13:00:00Z"
    },
    {
      "type": "objectStorage",
      "config": { "bucket": "drive", "quota": "50Gi" },
      "status": "provisioned",
      "provisionedAt": "2026-03-09T13:00:01Z"
    },
    {
      "type": "ai",
      "config": { "capabilities": ["chat", "embedding"] },
      "status": "provisioned",
      "provisionedAt": "2026-03-09T13:00:02Z"
    }
  ]
}
```

---

## 13. 구현 로드맵

| Phase | 내용 | 우선순위 |
|-------|------|---------|
| **Phase 1** | database + objectStorage + directory Provider | 🔴 즉시 |
| **Phase 1** | Saga 실행기 + Dual Driver (K8s + Docker) | 🔴 즉시 |
| **Phase 1** | module.yaml claims 파싱 + env 템플릿 치환 | 🔴 즉시 |
| **Phase 2** | cache + search + smtp + git Provider | 🟡 다음 |
| **Phase 2** | Reconciliation Loop | 🟡 다음 |
| **Phase 2** | Console UI — Claim 상태 대시보드 | 🟡 다음 |
| **Phase 3** | AI Gateway Provider | 🟢 후순위 |
| **Phase 3** | 비용 추적 + Rate Limit 대시보드 | 🟢 후순위 |
| **Phase 3** | Optional Claims (ai claim 없어도 설치 가능) | 🟢 후순위 |

---

## 14. 용어 정리

| 용어 | 설명 |
|------|------|
| **PRC** | Platform Resource Claim — 모듈의 선언적 자원 요청 |
| **Provider** | Foundation 서비스별 프로비저닝 구현체 |
| **Claim** | module.yaml에 선언된 개별 자원 요청 |
| **Credentials** | 프로비저닝 결과로 생성된 자격증명 (URL, 토큰 등) |
| **Saga** | 분산 프로비저닝의 트랜잭션 + 보상 패턴 |
| **Bind** | Credentials를 모듈에 주입 (Secret / .env) |
| **Deprovision** | 프로비저닝의 역연산 (자원 삭제) |
| **Compensate** | Saga 실패 시 이전 단계를 역순 롤백 |
| **DeployDriver** | K8s/Docker 추상화 인터페이스 |
| **Reconciliation** | 선언 상태와 실제 상태의 주기적 수렴 |

---

## 부록 A. module.yaml 전체 예시 (Drive)

```yaml
apiVersion: polyon/v1
kind: Module
metadata:
  id: drive
  name: "PolyON Drive"
  version: "0.1.0"
  description: "파일 스토리지 및 WebDAV 서비스"
  category: services
  icon: Folder

spec:
  image: jupitertriangles/polyon-drive:v0.1.0
  port: 8080
  healthPath: /health
  
  claims:
    - type: database
      config:
        name: drive
    - type: objectStorage
      config:
        bucket: drive
        quota: 50Gi
    - type: directory
      config:
        access: read
    - type: search
      config:
        index: drive-files
    - type: cache
      config:
        db: auto
  
  env:
    DATABASE_URL: "{{ claims.database.url }}"
    S3_ENDPOINT: "{{ claims.objectStorage.endpoint }}"
    S3_BUCKET: "{{ claims.objectStorage.bucket }}"
    S3_ACCESS_KEY: "{{ claims.objectStorage.accessKey }}"
    S3_SECRET_KEY: "{{ claims.objectStorage.secretKey }}"
    LDAP_URL: "{{ claims.directory.url }}"
    LDAP_BIND_DN: "{{ claims.directory.bindDN }}"
    LDAP_BIND_PW: "{{ claims.directory.bindPassword }}"
    LDAP_BASE_DN: "{{ claims.directory.baseDN }}"
    SEARCH_URL: "{{ claims.search.url }}"
    SEARCH_INDEX: "{{ claims.search.index }}"
    REDIS_URL: "{{ claims.cache.url }}"
    DRIVE_LISTEN: "0.0.0.0:8080"
    LOG_LEVEL: "info"
  
  dataPolicy: keep  # 삭제 시 데이터 유지 (재설치 시 복원 가능)

  console:
    adminPath: /admin
    menuGroup: services
    pages:
      - id: overview
        title: "개요"
        icon: Dashboard
        hash: "/overview"
      - id: users
        title: "사용자"
        icon: UserMultiple
        hash: "/users"
      - id: quota
        title: "할당량"
        icon: Meter
        hash: "/quota"
      - id: activity
        title: "활동"
        icon: Activity
        hash: "/activity"
      - id: settings
        title: "설정"
        icon: Settings
        hash: "/settings"

  portal:
    userPath: /
    menuGroup: apps
    pages:
      - id: files
        title: "내 드라이브"
        icon: Folder
        hash: "/"
      - id: shared
        title: "공유"
        icon: Share
        hash: "/shared"
      - id: recent
        title: "최근"
        icon: RecentlyViewed
        hash: "/recent"
      - id: favorites
        title: "즐겨찾기"
        icon: Star
        hash: "/favorites"
      - id: trash
        title: "휴지통"
        icon: TrashCan
        hash: "/trash"
```

## 부록 B. module.yaml 전체 예시 (Chat — AI Claim 포함)

```yaml
apiVersion: polyon/v1
kind: Module
metadata:
  id: chat
  name: "PolyON Chat"
  version: "2.0.3"
  description: "팀 메시징 및 채널 기반 커뮤니케이션"
  category: services

spec:
  image: jupitertriangles/polyon-chat:v2.0.3
  port: 8065
  healthPath: /api/v4/system/ping
  
  claims:
    - type: database
      config:
        name: chat
    - type: objectStorage
      config:
        bucket: chat-files
    - type: directory
      config:
        access: read
    - type: ai
      config:
        capabilities:
          - chat
        rateLimit: 5000/day
  
  env:
    MM_SQLSETTINGS_DATASOURCE: "{{ claims.database.url }}"
    MM_FILESETTINGS_AMAZONS3ENDPOINT: "{{ claims.objectStorage.endpoint }}"
    MM_FILESETTINGS_AMAZONS3BUCKET: "{{ claims.objectStorage.bucket }}"
    MM_FILESETTINGS_AMAZONS3ACCESSKEYID: "{{ claims.objectStorage.accessKey }}"
    MM_FILESETTINGS_AMAZONS3SECRETACCESSKEY: "{{ claims.objectStorage.secretKey }}"
    MM_LDAPSETTINGS_LDAPSERVER: "{{ claims.directory.url }}"
    MM_LDAPSETTINGS_BINDUSERNAME: "{{ claims.directory.bindDN }}"
    MM_LDAPSETTINGS_BINDPASSWORD: "{{ claims.directory.bindPassword }}"
    AI_ENDPOINT: "{{ claims.ai.endpoint }}"
    AI_API_KEY: "{{ claims.ai.apiKey }}"
    
  dataPolicy: keep
```
