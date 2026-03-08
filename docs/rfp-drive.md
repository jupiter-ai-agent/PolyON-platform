# PP Drive Module — 요구사항 정의서 (RFP)

## 1. 개요

PolyON Platform(PP)용 **파일 스토리지 서비스**를 Rust로 개발합니다.

기존 Nextcloud(PHP)를 대체하는 **경량, 고성능 파일 서비스**입니다.
Nextcloud의 기능을 복제하는 것이 아니라, PP에 필요한 핵심 기능만 구현합니다.

## 2. 범위

### ✅ 포함 (In Scope)

| 기능 | 설명 | 우선순위 |
|------|------|----------|
| 파일 업/다운로드 | 단일 파일 + 멀티파트 업로드, Range 다운로드 | **P0** |
| 폴더 CRUD | 폴더 생성/이름변경/이동/삭제, 트리 구조 | **P0** |
| AD LDAP 인증 | Samba AD DC 직접 LDAP bind (Pattern A) | **P0** |
| RustFS S3 백엔드 | 모든 파일을 RustFS(S3)에 저장 | **P0** |
| PostgreSQL 메타데이터 | 파일/폴더 메타데이터, 공유 정보, 사용자 설정 | **P0** |
| WebDAV | RFC 4918 기본 구현 (PROPFIND, GET, PUT, MKCOL, DELETE, MOVE, COPY) | **P1** |
| 파일 공유 (링크) | 공유 링크 생성, 만료일, 비밀번호 보호 | **P1** |
| 파일 공유 (사용자/그룹) | AD 사용자/그룹 단위 공유 | **P1** |
| 파일 버전 관리 | S3 오브젝트 버저닝 기반, 버전 목록/복원 | **P2** |
| OpenSearch 검색 | 파일명/내용 전문 검색, 인덱싱 | **P2** |
| 휴지통 | 삭제 파일 보관, 복원, 자동 삭제 정책 | **P2** |
| 썸네일 | 이미지 파일 썸네일 자동 생성 | **P3** |
| 오피스 미리보기 | PDF/문서 미리보기 (향후 OnlyOffice 연동) | **P3** |

### ❌ 제외 (Out of Scope)

| 기능 | 이유 |
|------|------|
| CalDAV / CardDAV | PP에서 Stalwart Mail이 담당 |
| 앱/플러그인 시스템 | 불필요한 복잡도 |
| 앱 스토어 | 불필요 |
| 외부 스토리지 마운트 | PP는 RustFS 단일 스토리지 |
| 영상 통화 | PP에서 Chat 모듈이 담당 |
| 메일 | PP에서 Mail 모듈이 담당 |

## 3. 기술 스택

### 필수

| 항목 | 기술 |
|------|------|
| **언어** | Rust (latest stable) |
| **HTTP** | axum 또는 actix-web |
| **Database** | sqlx (PostgreSQL) |
| **S3 Client** | aws-sdk-s3 또는 rust-s3 |
| **LDAP** | ldap3 crate |
| **WebDAV** | dav-server crate 또는 자체 구현 |

### 권장

| 항목 | 기술 |
|------|------|
| 직렬화 | serde + serde_json |
| 비동기 | tokio |
| 로깅 | tracing |
| 설정 | 환경변수 (dotenv 선택적) |
| 마이그레이션 | sqlx migrate |
| 검색 | opensearch-rs |

## 4. API 설계

### 4.1 인증

```
POST /api/v1/auth/login
Content-Type: application/json

{
    "username": "testuser1",      // sAMAccountName
    "password": "****"
}

Response 200:
{
    "token": "eyJ...",            // JWT
    "user": {
        "username": "testuser1",
        "displayName": "홍길동",
        "email": "testuser1@company.com"
    }
}
```

- JWT 토큰 기반 인증
- LDAP bind으로 비밀번호 검증
- JWT payload: `sub` (sAMAccountName), `name` (displayName), `email`, `groups`

### 4.2 파일 API

```
# 파일/폴더 목록
GET /api/v1/files/{path}
Authorization: Bearer {token}

Response 200:
{
    "path": "/documents",
    "items": [
        {
            "name": "report.pdf",
            "type": "file",
            "size": 1048576,
            "mimeType": "application/pdf",
            "modified": "2026-03-09T01:00:00Z",
            "etag": "abc123"
        },
        {
            "name": "photos",
            "type": "folder",
            "modified": "2026-03-08T15:00:00Z"
        }
    ]
}

# 파일 업로드
PUT /api/v1/files/{path}
Authorization: Bearer {token}
Content-Type: application/octet-stream

# 파일 다운로드
GET /api/v1/files/{path}?download=true
Authorization: Bearer {token}

# 폴더 생성
MKCOL /api/v1/files/{path}
Authorization: Bearer {token}

# 파일/폴더 삭제
DELETE /api/v1/files/{path}
Authorization: Bearer {token}

# 파일/폴더 이동
MOVE /api/v1/files/{path}
Destination: /api/v1/files/{newpath}
Authorization: Bearer {token}

# 파일/폴더 복사
COPY /api/v1/files/{path}
Destination: /api/v1/files/{newpath}
Authorization: Bearer {token}
```

### 4.3 공유 API

```
# 공유 링크 생성
POST /api/v1/shares
{
    "path": "/documents/report.pdf",
    "type": "link",                    // "link" | "user" | "group"
    "permissions": "read",             // "read" | "write" | "readwrite"
    "expiresAt": "2026-04-01T00:00:00Z",
    "password": "optional"
}

Response 201:
{
    "id": "abc123",
    "url": "https://drive.company.com/s/abc123",
    "expiresAt": "2026-04-01T00:00:00Z"
}

# 공유 목록
GET /api/v1/shares?path=/documents

# 공유 삭제
DELETE /api/v1/shares/{shareId}
```

### 4.4 WebDAV

```
WebDAV Root: /dav/files/{username}/

지원 메서드:
  PROPFIND   — 파일/폴더 속성 조회
  GET        — 파일 다운로드
  PUT        — 파일 업로드
  MKCOL      — 폴더 생성
  DELETE     — 삭제
  MOVE       — 이동
  COPY       — 복사
  HEAD       — 메타데이터
```

## 5. 데이터 모델

### PostgreSQL 스키마

```sql
-- 파일/폴더 메타데이터
CREATE TABLE files (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner       TEXT NOT NULL,                     -- sAMAccountName
    path        TEXT NOT NULL,                     -- /documents/report.pdf
    name        TEXT NOT NULL,                     -- report.pdf
    parent_id   UUID REFERENCES files(id),
    is_folder   BOOLEAN NOT NULL DEFAULT false,
    mime_type   TEXT,
    size        BIGINT DEFAULT 0,
    s3_key      TEXT,                              -- RustFS object key
    etag        TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (owner, path)
);

CREATE INDEX idx_files_owner ON files(owner);
CREATE INDEX idx_files_parent ON files(parent_id);

-- 공유
CREATE TABLE shares (
    id          TEXT PRIMARY KEY,                  -- 공유 링크 ID
    file_id     UUID NOT NULL REFERENCES files(id),
    owner       TEXT NOT NULL,
    share_type  TEXT NOT NULL,                     -- 'link', 'user', 'group'
    share_with  TEXT,                              -- username 또는 group name
    permissions TEXT NOT NULL DEFAULT 'read',
    password    TEXT,                              -- bcrypt hash
    expires_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 파일 버전 (P2)
CREATE TABLE file_versions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id     UUID NOT NULL REFERENCES files(id),
    version     INTEGER NOT NULL,
    s3_key      TEXT NOT NULL,
    size        BIGINT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 휴지통 (P2)
CREATE TABLE trash (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL,
    owner           TEXT NOT NULL,
    original_path   TEXT NOT NULL,
    s3_key          TEXT NOT NULL,
    size            BIGINT NOT NULL,
    deleted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL     -- 자동 삭제 시점
);
```

### S3 키 구조

```
{bucket}/{owner}/{path}

예:
  drive/testuser1/documents/report.pdf
  drive/testuser1/photos/vacation.jpg
  drive/__shared__/company-docs/handbook.pdf
```

## 6. S3 ↔ DB 동기화

- **업로드:** DB에 메타데이터 INSERT → S3에 파일 PUT (트랜잭션)
- **다운로드:** DB에서 s3_key 조회 → S3에서 GET (스트리밍)
- **삭제:** DB soft-delete (휴지통) → 만료 시 S3 DELETE
- **파일 크기/etag:** S3 HEAD 응답에서 추출하여 DB 업데이트

## 7. 비기능 요구사항

| 항목 | 요구사항 |
|------|----------|
| **성능** | 1GB 파일 업/다운로드 지원, 멀티파트 |
| **메모리** | 컨테이너 128MB 이하 (idle), 512MB 이하 (peak) |
| **동시성** | 100+ 동시 사용자 |
| **시작 시간** | 5초 이내 |
| **Docker 이미지** | 50MB 이하 (scratch 또는 alpine base) |
| **아키텍처** | `linux/amd64` + `linux/arm64` |
| **로깅** | JSON structured logging (tracing) |
| **설정** | 환경변수만 (설정 파일 없음) |
| **Graceful Shutdown** | SIGTERM 시 진행 중 요청 완료 후 종료 |

## 8. 배포

### Docker 이미지

```dockerfile
FROM rust:1.85-alpine AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM alpine:3.21
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/target/release/pp-drive /usr/local/bin/
EXPOSE 8080
CMD ["pp-drive"]
```

### K8s 배포

Core의 모듈 시스템이 자동으로 처리합니다.
`module.yaml`만 이미지에 포함하면 됩니다.

```yaml
# /polyon-module/module.yaml
apiVersion: polyon.io/v1
kind: Module
metadata:
  id: drive
  name: PP Drive
  version: 1.0.0
spec:
  resources:
    image: your-registry/pp-drive:v1.0.0
    ports:
      - name: http
        containerPort: 8080
  database:
    create: true
    name: drive
  ingress:
    subdomain: drive
    port: 8080
```

## 9. 검증 기준

### Phase 1 (P0) — MVP

- [ ] AD 사용자로 로그인
- [ ] 파일 업로드 (10MB, 100MB, 1GB)
- [ ] 파일 다운로드
- [ ] 폴더 CRUD
- [ ] 파일 목록 조회 (100+ 파일)
- [ ] RustFS에 파일 저장 확인
- [ ] PostgreSQL에 메타데이터 저장 확인
- [ ] Health endpoint 응답
- [ ] Docker 이미지 50MB 이하

### Phase 2 (P1) — 공유 + WebDAV

- [ ] WebDAV PROPFIND, GET, PUT, MKCOL, DELETE, MOVE, COPY
- [ ] macOS Finder WebDAV 마운트 동작
- [ ] Windows 탐색기 WebDAV 마운트 동작
- [ ] 공유 링크 생성/접근
- [ ] 사용자/그룹 공유
- [ ] 공유 링크 비밀번호 보호

### Phase 3 (P2) — 검색 + 버전

- [ ] 파일명 검색 (OpenSearch)
- [ ] 파일 내용 검색 (PDF/txt)
- [ ] 파일 버전 히스토리
- [ ] 버전 복원
- [ ] 휴지통 + 자동 삭제

## 10. 참고

- [Platform Overview](platform-overview.md) — PP 전체 아키텍처
- [Module Spec](module-spec.md) — 모듈 개발 규격
- [Integration Guide](integration-guide.md) — 인프라 연동 코드 예제
